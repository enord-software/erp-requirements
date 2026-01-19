# ERP Integration Specification for Elight E-Commerce

## 1. Context for ERP Team
We are connecting the existing ERP system to a modern E-commerce website (Elight). Since the E-commerce platform faces customers directly, it requires specific data structures that might differ from internal ERP records.

**Key Goals:**
1.  **Product Sync**: When a product is added/updated in ERP, it must appear on the website.
2.  **Inventory Sync**: Stock levels must stay accurate in real-time to prevent overselling.
3.  **Pricing Sync**: Price changes in ERP must reflect immediately on the website.

## 2. Core Concepts (E-Commerce vs ERP)

### "Product" vs "Variant"
In E-commerce, we group items.
*   **Product**: "Nike Air Max" (The generic style).
*   **Variant**: "Nike Air Max - Size 10 - Black" (The specific sellable item with a unique SKU).

**Requirement:** Does the ERP treat every SKU as a standalone item, or does it have a concept of "Parent Item"?
*   *If independent:* We will need a `group_id` or `style_code` field to group them on the website.
*   *If grouped:* Please provide the `Parent ID` and `Variant ID`.

## 3. Data Requirements (The "Ask")

We need the ERP system to provide the following data fields for every sellable item.

| Field Name | Type | Importance | Description |
| :--- | :--- | :--- | :--- |
| `sku` | String | **Critical** | Unique ID for the specific variant (e.g., `SHOE-BLK-10`). Used as the primary key. |
| `name` | String | **Critical** | Customer-facing name (e.g., "Running Shoe - Black"). |
| `description` | Text | High | Detailed product description (HTML preferred, plain text ok). |
| `category_tree` | String | High | Hierarchy for navigation (e.g., "Men > Shoes > Running"). |
| `price` | Decimal | **Critical** | The selling price (before tax). |
| `mrp` | Decimal | Optional | Maximum Retail Price (to show discounts). |
| `stock_quantity`| Integer| **Critical** | Current available inventory count. |
| `tax_code` | String | High | HSN/SAC code for GST calculation. |
| `images` | List[URL]| High | Direct URLs to product images. If ERP holds images, they must be accessible via public URL. |
| `weight_kg` | Decimal | Medium | Shipping weight calculation. |
| `attributes` | JSON | Medium | E.g., `{"Color": "Black", "Size": "10"}`. |

## 4. Operational Requirements (Webhooks)

Since we need **Real-Time** updates, we request that the ERP system pushes changes to us via **Webhooks**.

**Target URL:** `https://backend.elightspm.com/api/integrations/erp/events/` (Example)
**Method:** `POST`
**Security:** Please verify payloads using a shared Secret Key in the header (e.g., `X-ERP-Signature`).

### Scenario A: Inventory Change
**Trigger:** Stock moves in warehouse, Order created in ERP, etc.
**Expected JSON Payload:**
```json
{
  "event_type": "inventory_update",
  "timestamp": "2024-01-20T10:00:00Z",
  "data": {
    "sku": "SHOE-BLK-10",
    "new_quantity": 45,
    "warehouse_id": "WH-MUM-01"
  }
}
```

### Scenario B: Product Update (Price/Details)
**Trigger:** Price change, Description update.
**Expected JSON Payload:**
```json
{
  "event_type": "product_update",
  "timestamp": "2024-01-20T10:05:00Z",
  "data": {
    "sku": "SHOE-BLK-10",
    "price": 2499.00,
    "mrp": 2999.00,
    "active": true
  }
}
```

## 5. Order Synchronization (E-Commerce -> ERP)

When a customer places an order on the Website, we will push it to the ERP so you can reserve stock and generate the invoice.

**We need an Endpoint from ERP:**
*   `POST /api/orders`

**Payload We Will Send:**
```json
{
  "order_id": "ORD-998877",
  "customer": {
    "name": "John Doe",
    "gstin": "27ABCDE1234F1Z5"
  },
  "items": [
    {
      "sku": "SHOE-BLK-10",
      "quantity": 1,
      "unit_price": 2499.00
    }
  ]
}
```

## 6. Action Items for ERP Team
1.  **Map Data**: Can you map your internal fields to the list in Section 3?
2.  **Webhooks**: Can your system trigger HTTP POST requests on events?
3.  **Images**: How are product images stored? Can you provide URLs?
