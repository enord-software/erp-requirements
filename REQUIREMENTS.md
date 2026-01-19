# ERP Integration Specification: Digital Sales Interface Synchronization
**Version:** 5.0.0
**Target Audience:** ERP Development Team
**Scope:** Bi-Directional Product & Inventory Synchronization

---

## **1. Executive Summary**

### **1.1 The Objective**
We are integrating the legacy **ERP System** with a new **Digital Sales Interface** (e-commerce platform). The goal is to establish a unified system where product data flows from the ERP to the Sales Interface, and sales data flows back to the ERP to maintain accurate inventory across all channels.

**Core Synchronization Logic:**
1.  **Product Master**: The ERP remains the definitive source for Product details, Pricing, and Master Inventory.
2.  **Inventory Unity**:
    *   **Offline/ERP Activity**: If stock is sold or moved within the ERP (e.g., physical store sale), the Sales Interface must be updated immediately to prevent overselling online.
    *   **Online Activity**: If stock is sold via the Sales Interface, the ERP must be notified immediately to reserve/deduct the stock from the master pool.

---

## **2. Communication Architecture**

To achieve near real-time accuracy, we require a **Bi-Directional API Strategy**:

*   **ERPs Push (Inbound)**: The ERP system uses **Webhooks (HTTP POST)** to push data to the Sales Interface whenever a record is created or modified.
*   **Sales Interface Push (Outbound)**: The Sales Interface uses **REST APIs** exposed by the ERP to report sales and sales orders.

---

## **3. Security Protocol**

All data exchange must occur over **HTTPS**.

*   **Webhooks (ERP -> Sales Interface)**: 
    *   Must accept a **Secret Key** configuration.
    *   Requests must include an `X-Signature` header containing an HMAC-SHA256 hash of the payload using this secret key.
*   **API Access (Sales Interface -> ERP)**:
    *   The ERP must provide an **API Key** or **Bearer Token** to authenticate requests from the Sales Interface.

---

## **4. Data Requirements: ERP to Sales Interface (Webhooks)**

The ERP mechanism must trigger triggers on specific database events.

### **4.1 Event: Catalog Update**
**Trigger**: Creation or Modification of a Product Item (SKU), Price, or details.
**Requirement**: Push the full product record to ensure data consistency.

**Payload Specification:**
```json
{
  "event_type": "catalog.update",
  "timestamp": "2024-01-20T10:00:00Z",
  "data": {
    "group_id": "GRP-101",          // Parent Style ID (e.g. Shirt Style A)
    "group_name": "Premium Cotton Shirt",
    "description": "Full product description...",
    "category_path": ["Men", "Apparel", "Formal"], 
    "tax_code": "HSN-8899",         // Tax classification code
    "is_active": true,
    "variants": [                   // List of sellable SKUs
      {
        "sku": "SHIRT-A-WHT-L",     // UNIQUE Identifier
        "variant_title": "White - Large",
        "selling_price": 2500.00,
        "mrp": 2999.00,
        "current_stock": 100,       // Absolute stock level
        "specifications": {         // Dynamic attributes
          "Color": "White",
          "Size": "L",
          "Material": "Cotton"
        },
        "media_assets": [           // Public URLs for images
          "https://storage.erp-system.com/img/shirt-wht-1.jpg"
        ]
      }
    ]
  }
}
```

### **4.2 Event: Inventory Change**
**Trigger**: Any stock movement in the ERP (POS Sale, Stock Receipt, Return, Adjustment).
**Performance**: Must fire within 5 seconds of the transaction.

**Payload Specification:**
```json
{
  "event_type": "inventory.change",
  "timestamp": "2024-01-20T10:05:00Z",
  "data": {
    "sku": "SHIRT-A-WHT-L",
    "location_code": "WH-MAIN",
    "new_quantity": 99,             // The updated absolute quantity
    "change_reason": "POS_SALE"
  }
}
```

---

## **5. Data Requirements: Sales Interface to ERP (API)**

The ERP must provide endpoints to accept transactional data from online sales.

### **5.1 Action: Record Online Sale**
**Trigger**: Customer completes a purchase on the Sales Interface.
**Requirement**: ERP must accept this payload to generate an Invoice and reserve stock.

**Target Endpoint**: `POST /api/sales/orders`
**Payload Specification:**
```json
{
  "transaction_id": "ORD-WEB-2024-555",
  "transaction_date": "2024-01-20T10:15:00Z",
  "customer_details": {
    "name": "Jane Doe",
    "tax_id": "GSTIN12345"
  },
  "line_items": [
    {
      "sku": "SHIRT-A-WHT-L",
      "quantity": 2,
      "unit_price": 2500.00
    }
  ]
}
```

### **5.2 Backup: Simple Inventory Reservation**
If full order ingestion is complex, a minimal "Stock Deduction" endpoint is acceptable.

**Target Endpoint**: `POST /api/inventory/adjust`
**Payload Specification:**
```json
{
  "reference_id": "ORD-WEB-2024-555",
  "adjustments": [
    {
      "sku": "SHIRT-A-WHT-L",
      "quantity_change": -2,
      "location_code": "WH-MAIN"
    }
  ]
}
```

---

## **6. Data Integrity Protocol**

To handle network outages or missed webhooks, we require a **Reconciliation Endpoint**.

**Endpoint**: `GET /api/inventory/snapshot`
**Function**: Returns the current stock levels for ALL SKUs.
**Schedule**: The Sales Interface will call this nightly (e.g., 03:00 AM) to correct any drift.

**Expected Response:**
```json
[
  { "sku": "SHIRT-A-WHT-L", "qty": 99 },
  { "sku": "SHIRT-A-WHT-M", "qty": 45 },
  ...
]
```

---

## **7. Implementation Requirements Checklist**

For the successful integration of this POC (Proof of Concept), the ERP team must provide:

1.  [ ] **Webhook Capability**: Confirm the ERP can trigger HTTP POST calls on database events.
2.  **API Credentials**: Provision an API Key for the Sales Interface to authenticate.
3.  **Media Access**: Ensure product images stored in ERP are accessible via public HTTP links.
4.  **Endpoint Documentation**: Provide the actual URLs for the ERP's API (Staging/Production).
