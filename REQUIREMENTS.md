# Comprehensive ERP Integration Specification: Bi-Directional Product & Inventory Sync
**Version:** 4.0.0
**Status:** DRAFT
**Target Audience:** ERP Development Team / System Architects
**Scope:** Product Catalog, Real-Time Inventory, Stock Deduction

---

## **1. Executive Summary & Architecture**

### **1.1 The Objective**
We are building a unified commerce ecosystem where the **Legacy ERP** and the **New Elight E-Commerce Platform** act as synchronized nodes for inventory and product data. 

**The Golden Rules of Sync:**
1.  **Product Master**: The ERP is the source of truth for *what* we sell (SKUs, Names, Prices).
2.  **Shared Inventory**: The ERP manages the *Master Stock*.
    *   **Offline Sale**: If stock moves in ERP (e.g., in-store sale), the E-commerce DB must update immediately.
    *   **Online Sale**: If stock moves in E-commerce (website order), the ERP Master Stock must update immediately to prevent double-selling.

### **1.2 High-Level Architecture**
To achieve this, we require a **Bi-Directional Interface**:

*   **Inbound (ERP -> Commerce)**: The ERP pushes `Product Updates` and `Inventory Changes` via **HTTP Webhooks**. This ensures the website always reflects the warehouse reality.
*   **Outbound (Commerce -> ERP)**: The Commerce system pushes `Sales Signals` (Order/Inventory Deduction) via **REST API** to the ERP. This ensures the ERP reserves stock for online buyers.

---

## **2. Detailed Interface Specifications**

### **2.1 Security & Authentication**
All exchanges must occur over **HTTPS**.

*   **For Webhooks (ERP -> Commerce)**:
    *   **Mechanism**: HMAC Signature.
    *   **Header**: `X-ERP-Signature`
    *   **Algorithm**: `HMAC-SHA256(request_body, shared_secret)`
    *   **Reason**: Prevents replay attacks and ensures payload integrity.

*   **For API Calls (Commerce -> ERP)**:
    *   **Mechanism**: API Key or Bearer Token.
    *   **Header**: `Authorization: Bearer <token>`
    *   **Reason**: Authenticates the E-commerce server as a trusted client.

---

## **3. Inbound Data Requirements (ERP -> Commerce)**

The ERP must implement the following **Webhooks**. These are strictly "Push" events triggered by database changes in the ERP.

### **3.1 Event: `catalog.update`**
**Trigger**: When a Product is Created, Updated, or Deleted in ERP.
**Requirement**: Send the full product object. Partial updates are discouraged to avoid data inconsistency.

**Payload Schema:**
```json
{
  "event_id": "evt_550e8400-e29b",
  "event_type": "catalog.update",
  "timestamp": "2024-01-20T10:00:00Z",
  "data": {
    "product_group_id": "PG-1001",    // Groups variants (e.g., iPhone 15)
    "group_name": "iPhone 15",        // Name of the group
    "description": "<p>Full HTML...</p>",
    "category_hierarchy": ["Electronics", "Smartphones", "Apple"], 
    "tax_hsn": "85171300",            // Global tax code for the group
    "is_active": true,
    "variants": [
      {
        "sku": "IPH15-128-BLK",       // PRIMARY KEY
        "display_name": "iPhone 15 128GB Black",
        "price": 79900.00,            // Base Selling Price (Excl Tax)
        "mrp": 89900.00,              // Max Retail Price
        "stock_quantity": 50,         // Current Global Stock
        "attributes": {               // Custom Filters
          "Color": "Black",
          "Storage": "128GB"
        },
        "media": [                    // Publicly accessible URLs
          { "url": "https://erp-cdn.com/u/iph15-blk.jpg", "is_main": true },
          { "url": "https://erp-cdn.com/u/iph15-blk-back.jpg", "is_main": false }
        ]
      },
      {
        "sku": "IPH15-128-BLU",
        "display_name": "iPhone 15 128GB Blue",
        "price": 79900.00,
        "stock_quantity": 42,
        "attributes": { "Color": "Blue", "Storage": "128GB" },
        "media": [...]
      }
    ]
  }
}
```

#### **Field Dictionary**
| Field | Data Type | Constraint | Description |
| :--- | :--- | :--- | :--- |
| `product_group_id` | String | Max 50 chars | Identifier linking variants. If product has no variants, use SKU. |
| `sku` | String | **UNIQUE** | The immutable ID. Critical for mapping. |
| `price` | Decimal | Precision 2 | Selling price. |
| `attributes` | JSON Object | Dynamic | Used for "Filter By" features on the website. |
| `media` | Array | Min 1 item | URLs must be public. Commerce server will confirm download. |

---

### **3.2 Event: `inventory.change`**
**Trigger**: ANY change in stock level in the ERP (e.g., POS sale, Stock Adjustment, Return, Goods Receipt).
**Requirement**: Must be fired within **5 seconds** of the transaction to strictly minimize "Out of Stock" order risk.
**Payload**: Can be a "Delta" (Change) or "Absolute" (Total). **Absolute is preferred** to fix drift.

**Payload Schema:**
```json
{
  "event_id": "evt_inventory_8899",
  "event_type": "inventory.change",
  "timestamp": "2024-01-20T10:05:00Z",
  "data": {
    "sku": "IPH15-128-BLK",
    "location_id": "WH-MAIN",       // Optional: Context on where stock moved
    "new_quantity": 49,             // The NEW total available stock
    "reason": "POS_SALE"            // Optional: Why it changed
  }
}
```

**Operational Scenario:**
1.  Customer walks into store, buys 1x `IPH15-128-BLK`.
2.  ERP Stock goes 50 -> 49.
3.  ERP fires `inventory.change` webhook.
4.  Commerce receives payload -> Updates DB -> Website now shows "49 Left".

---

## **4. Outbound Data Requirements (Commerce -> ERP)**

The E-commerce system needs to notify the ERP when it sells an item so the ERP can reserve/deduct that stock from the global pool.

**Requirement**: The ERP **MUST** expose a REST Endpoint for this.

### **Option A: Full Order Sync (Recommended)**
Standard approach. Allows ERP to generate Invoice and Shipping Labels.

**Endpoint**: `POST /api/v1/orders/import`
**Payload**:
```json
{
  "order_id": "ORD-WEB-1001",
  "order_date": "2024-01-20T10:15:00Z",
  "customer_ref": "CUST-555",
  "line_items": [
    {
      "sku": "IPH15-128-BLK",
      "quantity": 1,
      "unit_price": 79900.00
    }
  ]
}
```

### **Option B: Inventory Deduction Only (Minimum Viable)**
If the ERP team cannot ingest full orders yet, we need a simple "Reserve Stock" endpoint.

**Endpoint**: `POST /api/v1/inventory/adjust`
**Side Effect**: ERP decreases stock by `quantity`.

**Payload**:
```json
{
  "reference_id": "ORD-WEB-1001",
  "adjustments": [
    {
      "sku": "IPH15-128-BLK",
      "quantity_change": -1,   // Negative to deduct
      "warehouse_id": "WH-MAIN"
    }
  ]
}
```

---

## **5. Data Integrity & Reconcilliation**

Since webhooks can fail, we need a fallback mechanism.

### **5.1 The "Nightly Sweeper"**
**Requirement**: ERP must expose an endpoint to fetch ALL stock levels.
**Endpoint**: `GET /api/v1/inventory/snapshot`
**Response**: JSON Stream or CSV file URL.
```json
[
  { "sku": "A", "qty": 10 },
  { "sku": "B", "qty": 0 },
  ...
]
```
**Process**: Every night at 3 AM, Commerce calls this, compares with its DB, and overwrites any mismatches.

---

## **6. Technical Checklist for ERP Team**

Please mark availability for the following:

1.  **Webhooks Engine**: Can you trigger HTTP POST on database events? [ ] Yes [ ] No
2.  **API Access**: Can you provide an API Key for us to POST orders/stock? [ ] Yes [ ] No
3.  **Image Hosting**: Are product images accessible via public URL? [ ] Yes [ ] No
4.  **SKU Stability**: Is the SKU field static? (It must not change over the product's life). [ ] Yes [ ] No
5.  **Environment**: 
    *   **Staging/Test ERP URL**: ________________________
    *   **Production ERP URL**: ________________________

---

## **7. Error Handling Specification**

### **7.1 Retry Logic (ERP Side)**
If the Commerce Webhook Endpoint returns `500`, `502`, `503`, or `504` (Server Error):
*   **Action**: The ERP must Queue the event and Retry.
*   **Strategy**: Exponential Backoff (1m, 5m, 15m, 1h, 6h).
*   **Alert**: Failure after 24h sends email to Admin.

If the Commerce Endpoint returns `400` (Bad Request):
*   **Action**: Do NOT retry. Log "Data Error" (e.g., Invalid JSON, Missing SKU).

### **7.2 Order Push Failure (Commerce Side)**
If the `POST /orders` call to ERP fails:
*   **Action**: Commerce marks order as "Pending Sync".
*   **Retry**: Retries every 15 mins for 2 hours.
*   **Fallback**: If ERP is down > 2 hours, stock is reserved locally only. Synchronization happens once ERP is back online.

---

**End of Specification**
**Prepared By:** Elight Commerce Team
