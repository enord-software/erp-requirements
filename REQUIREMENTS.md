# Comprehensive ERP Integration Technical Specification
**Version:** 2.0.0
**Status:** DRAFT
**Target Audience:** ERP Development Team, System Architects
**Date:** 2024-01-20

---

## **1. Executive Summary**

This document serves as the authoritative technical reference for integrating the organization's legacy Enterprise Resource Planning (ERP) system with the new Elight E-Commerce Platform. 

The integration logic rests on a fundamental principle: **The ERP is the Source of Truth** for catalog (Products, Inventory, Pricing) data, while the **E-Commerce Platform is the Source of Truth** for customer-generated transactional data (Orders, Returns) until internalized by the ERP.

This specification outlines the interface contracts, data models, security protocols, and operational workflows required to achieve a near real-time, bi-directional synchronization.

---

## **2. System Architecture & Protocols**

### 2.1 Communication Pattern
To minimize latency and avoid polling overhead, the integration relies primarily on **Event-Driven Architecture (Webhooks)** for ERP-to-Commerce updates, supplemented by **RESTful APIs** for Commerce-to-ERP transactions and nightly reconciliation.

*   **ERP -> Commerce**: ERP pushes data via HTTP POST Webhooks immediately upon record modification (Create/Update/Delete).
*   **Commerce -> ERP**: Commerce pushes transactional data (Orders) via REST API immediately upon creation.
*   **Reconciliation**: A scheduled batch job (ETL) runs nightly to verify data integrity and correct drift.

### 2.2 Security & Authentication
All communication must be secured over **HTTPS (TLS 1.2+)**.

#### 2.2.1 Webhook Security (ERP -> Commerce)
To ensure that payloads received by the E-commerce system legitimately originate from the ERP, all webhook requests must include an **HMAC-SHA256 Signature**.
*   **Method**: `POST`
*   **Header**: `X-ERP-Signature`
*   **Algorithm**: `HMAC-SHA256(body, secret_key)`
*   **Secret Key**: A 64-character random string shared securely between teams (Do not commit to code).

**Implementation Requirement:** The ERP must generate this signature for every request. The Commerce system will reject any request with an invalid or missing signature.

#### 2.2.2 API Authentication (Commerce -> ERP)
For the E-commerce system to push orders to the ERP, the ERP must provide a secure REST API protected by **OAuth 2.0 (Client Credentials Flow)** or, at minimum, high-entropy **API Keys**.
*   **Header**: `Authorization: Bearer <token>` or `X-API-Key: <key>`

### 2.3 Idempotency & Reliability
Networks are unreliable. The ERP must implement **Retry Logic** with **Exponential Backoff** for all webhooks.
*   **Retry Schedule**: Immediately, 30s, 5m, 1h, 6h, 24h.
*   **Idempotency**: All payloads must include a unique `event_id`. The Commerce system will cache this ID for 72 hours to prevent processing duplicate messages if a retry occurs after a successful processing acknowledgement was lost.

---

## **3. Data Dictionary: Product Catalog**

### 3.1 Concept Mapping
| ERP Entity | Commerce Entity | Notes |
| :--- | :--- | :--- |
| Item/SKU | Product Variant | The sellable unit. Unique SKU is mandatory. |
| Style/Group | Product | The parent container for variants (e.g., "T-Shirt" containing Red/Blue variants). |
| Category | Collection/Category | Categories are hierarchical. |
| Warehouse | Inventory Location | Multi-warehouse stock mapping. |

### 3.2 Field Specifications (ERP -> Commerce)
The ERP **MUST** provide the following fields in the `product.update` webhook.

#### 3.2.1 Core Identity
*   `sku` (String, Max 64, **PK**): The immutable unique identifier. e.g., "NK-AM-BLK-10".
*   `group_id` (String, Max 64, Optional): The ID of the parent style. If null, product is standalone.
*   `barcode` (String, Max 20, Optional): EAN/UPC/GTIN for scanning.

#### 3.2.2 Descriptive Data
*   `name` (String, Max 255): The consumer-facing product title. e.g., "Nike Air Max 90 - Black".
*   `description_html` (Text): Full product description. Support for HTML tags (`<p>`, `<ul>`, `<li>`, `<b>`) is required.
*   `short_description` (String, Max 500): SEO meta description or collection page summary.
*   `attributes` (JSON Object): Flexible key-value pairs for filtering.
    *   Example: `{"Material": "Leather", "Season": "SS24", "Gender": "Unisex"}`
*   `variant_options` (JSON Array): Defining attributes for the variant.
    *   Example: `[{"name": "Size", "value": "10"}, {"name": "Color", "value": "Black"}]`

#### 3.2.3 Commercial Data
*   `price` (Decimal, 10,2): The selling price excluding tax. e.g., `2499.00`.
*   `mrp` (Decimal, 10,2): The "Strike-through" price (MRP/RRP). e.g., `2999.00`.
*   `cost_price` (Decimal, 10,2, **Internal Only**): For margin calculation (Do not enable if sensitive).
*   `tax_hsn` (String, Max 10): Harmonized System Nomenclature (HSN) code for GST.
*   `tax_rate` (Decimal, 4,2): The applicable tax percentage (e.g., `18.00`).

#### 3.2.4 Media Requirements
*   `images` (Array of Strings): An ordered list of PUBLICLY ACCESSIBLE URLs.
    *   **Constraint**: The ERP must host images or push them to a CDN. The Commerce system will download and cache them. Sending Base64 streams is **NOT** supported due to payload size limits.

#### 3.2.5 Logistics
*   `weight_kg` (Decimal, 6,3): e.g., `0.450`.
*   `dimensions_cm` (Object): `{ "length": 30, "width": 20, "height": 10 }`.

---

## **4. Data Dictionary: Inventory**

Inventory is the most time-sensitive data point. The ERP must push "delta" updates immediately.

### 4.1 Multi-Location Support
If the ERP manages multiple warehouses (e.g., "Main Warehouse", "Retail Store 01"), the inventory payload must specify the `location_id`.

### 4.2 Payload Specification
**Event Name**: `inventory.update`
```json
{
  "event_id": "evt_123456789",
  "occurred_at": "2024-01-20T14:30:00Z",
  "sku": "NK-AM-BLK-10",
  "total_available": 45,
  "breakdown": [
    {
      "location_id": "WH-001",
      "quantity": 40,
      "reserved": 0
    },
    {
      "location_id": "STR-005",
      "quantity": 5,
      "reserved": 2
    }
  ]
}
```
*   **Logic**: The E-commerce system will sum the `quantity` from all mappable locations to determine "Sellable Online Stock".

---

## **5. Webhook Specifications (ERP -> Commerce)**

The following Webhook Events are mandatorily required.

### 5.1 `product.created` / `product.updated`
Triggered when a product is created or ANY field listed in Section 3.2 is modified.
**Endpoint**: `/api/webhooks/erp/products`
**Payload Schema**:
```json
{
  "event": "product.update",
  "data": {
    "sku": "SHOE-001",
    "active": true,
    "identity": {
      "name": "Leather Shoe",
      "group_id": "GRP-SHOE-A",
      "barcode": "8901234567890"
    },
    "commerical": {
      "price": 2500.00,
      "mrp": 3000.00,
      "tax_code": "6403"
    },
    "attributes": {
      "Color": "Brown",
      "Material": "Leather"
    }
  }
}
```

### 5.2 `price.updated`
Triggered when ONLY price changes (optimization for high-frequency updates).
**Endpoint**: `/api/webhooks/erp/prices`
**Payload Schema**:
```json
{
  "event": "price.update",
  "data": {
    "sku": "SHOE-001",
    "price": 2400.00,
    "sale_price": null,
    "currency": "INR",
    "effective_date": "2024-01-20T00:00:00Z"
  }
}
```

### 5.3 `inventory.updated`
(See Section 4.2)

---

## **6. API Specifications (Commerce -> ERP)**

The ERP must expose these endpoints.

### 6.1 Order Insertion
**Method**: `POST`
**Endpoint**: `/api/orders`
**Description**: Called when a customer completes checkout successfully.
**Payload Schema**:
```json
{
  "commerce_order_id": "ORD-2401-001",
  "order_date": "2024-01-20T15:00:00Z",
  "customer": {
    "commerce_id": "CUST-999",
    "email": "customer@example.com",
    "phone": "+919876543210",
    "first_name": "Rahul",
    "last_name": "Sharma"
  },
  "billing_address": {
    "line1": "Flat 101, Galaxy Apts",
    "city": "Mumbai",
    "state": "Maharashtra",
    "pincode": "400001",
    "country": "IN",
    "gstin": "27ABCDE1234F1Z5" // Optional B2B
  },
  "shipping_address": { ... },
  "items": [
    {
      "sku": "NK-AM-BLK-10",
      "quantity": 1,
      "unit_price": 2500.00,
      "tax_amount": 450.00,
      "total": 2950.00
    }
  ],
  "financials": {
    "subtotal": 2500.00,
    "tax_total": 450.00,
    "shipping_fee": 100.00,
    "discount_total": 0.00,
    "grand_total": 3050.00,
    "payment_method": "Razorpay",
    "payment_ref": "pay_O123456789"
  }
}
```
**Expected Response**:
*   `201 Created`: `{ "erp_order_id": "SO-230055", "status": "Booked" }`
*   `400 Bad Request`: `{ "error": "Invalid SKU NK-AM-BLK-10" }` (Commerce will log this as a critical error requiring human intervention).

### 6.2 Order Cancellation
**Method**: `POST`
**Endpoint**: `/api/orders/{erp_order_id}/cancel`
**Description**: Called if customer cancels before shipping.

---

## **7. Error Handling & Recovery**

### 7.1 Webhook Failure Scenarios (ERP -> Commerce)
If the Commerce system returns `4xx` or `5xx`:
1.  **ERP Action**: Log the error, Mark event for Retry.
2.  **Retry Policy**: Execute Exponential Backoff (Section 2.3).
3.  **Dead Letter Queue**: After 5 failed attempts, move to DLQ and alert ERP admin.

### 7.2 API Failure Scenarios (Commerce -> ERP)
If the Order Push fails:
1.  **Commerce Action**: Mark local order status as `Sync Failed`.
2.  **Retry**: Commerce cron job will retry syncing `Sync Failed` orders every 5 minutes for 1 hour.
3.  **Alert**: If fail persists > 1 hour, email alert sent to SysAdmin.

---

## **8. Reconciliation (Nightly Batch)**

A "Safety Net" process is required to handle missed webhooks or race conditions.

### 8.1 Full Inventory Sync
**Direction**: Commerce pulls from ERP.
**Schedule**: 02:00 AM IST.
**Endpoint**: `GET /api/inventory/snapshot` (ERP Endpoint)
**Logic**: Commerce downloads a CSV/JSON dump of ALL SKU stock levels. Overwrites local database values with ERP values.

### 8.2 Full Price Sync
**Direction**: Commerce pulls from ERP.
**Schedule**: 03:00 AM IST.
**Endpoint**: `GET /api/prices/snapshot`
**Logic**: Updates base price and tax info for all SKUs.

---

## **9. Implementation Roadmap**

### Phase 1: Preparation (Active)
*   [ ] Agree on security keys (API Secrets).
*   [ ] ERP Team provides static list of Categories and Tax Codes for mapping.
*   [ ] Network: Allowlisting IPs between ERP server and Commerce server.

### Phase 2: Development
*   [ ] ERP implements `product` and `inventory` webhooks.
*   [ ] Commerce implements Webhook Receiver endpoints.
*   [ ] Commerce implements Order Push client.
*   [ ] ERP implements Order Receive endpoint.

### Phase 3: Testing (UAT)
*   [ ] **Test Case 1**: Create Product in ERP -> Verify appearing on Website within 5 seconds.
*   [ ] **Test Case 2**: Change Price in ERP -> Verify updated implementation on Website.
*   [ ] **Test Case 3**: Place Order on Website -> Verify Sales Order created in ERP.
*   [ ] **Test Case 4**: Reduce Stock to 0 in ERP -> Verify Product "Out of Stock" on Website.
*   [ ] **Test Case 5**: Simulate Network Failure -> Verify Retry mechanism delivers payload after recovery.

---

## **10. Appendix: Data Mapping Tables**

### 10.1 Tax Codes
| ERP Tax Code | GST Rate | Description |
| :--- | :--- | :--- |
| `GST0` | 0% | Exempt Goods |
| `GST5` | 5% | Essentials |
| `GST12` | 12% | Standard Tier 1 |
| `GST18` | 18% | Standard Tier 2 |
| `GST28` | 28% | Luxury Goods |

*(This table must be populated by ERP Finance Team)*
