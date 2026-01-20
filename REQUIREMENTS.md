# ERP API Requirements — Product & Inventory Integration
 
**Document Type:** Integration Requirements  

 

## 1. Executive Summary
This document defines the technical specifications for the **Product, Pricing, and Inventory API** integration between the ERP system and the e-commerce frontend/middleware.

**Note:** The API endpoints and schemas defined in this document serve as a **reference requirement specification**. The ERP team is expected to implement and provide these APIs (or functionally equivalent interfaces) to support the integration objectives.

The API provides full **Read and Write (CRUD)** capabilities to ensure seamless data management and synchronization across all systems. The **ERP Team** is responsible for the implementation and maintenance of these services.

### Core Objectives
*   **Full CRUD Support:** Enable creating, reading, updating, and deleting product and pricing data.
*   **Accuracy:** Ensure product data (titles, descriptions, images) is consistent and up-to-date.
*   **Real-time Tracking:** Provide low-latency inventory lookups to prevent overselling.
*   **Safety:** Implement reservation logic to lock stock during the checkout process.
*   **Synchronization:** Utilize webhooks for immediate updates on pricing and stock level changes.

 

## 2. Scope & Technical Assumptions

### Scope
The scope includes all RESTful API endpoints, data schemas, authentication mechanisms, and webhook notifications required for the ERP system to communicate with Enord’s e-commerce ecosystem.

### Assumptions & Constraints
*   **SSOT:** The SKU (Stock Keeping Unit) is the Single Source of Truth for product identification.
*   **Connectivity:** All communication must occur over **HTTPS** using **TLS 1.2** or higher.
*   **Data Format:** All requests and responses must use **JSON**.
*   **Timezone:** All timestamps must follow **ISO 8601** format in UTC.
*   **Versioning:** This API is version-less to ensure continuous integration and deployment.

 

## 3. Glossary

| Term | Description |
| :--- | :--- |
| **SKU** | Stock Keeping Unit, unique identifier across systems. |
| **product_id** | Internal ERP database identifier. |
| **variant** | A specific version of a product (e.g., different battery capacity for a drone). |
| **reserved_quantity** | Stock temporarily held for a pending order (e.g., during payment processing). |
| **Webhook** | A user-defined HTTP callback triggered by specific events in the ERP system. |


## 4. Security & Authentication

### 4.1 Authentication Mechanism
All API requests must be authenticated using **Bearer Tokens (JWT)** or **API Keys** passed in the `Authorization` header.

```http
Authorization: Bearer <YOUR_ACCESS_TOKEN>
```

### 4.2 Rate Limiting
To ensure system stability, the following rate limits apply:
*   **Standard Endpoints:** 100 requests per minute per API key.
*   **Inventory Check:** 500 requests per minute per API key.
*   **Exceeding Limits:** Returns `429 Too Many Requests`.

 

## 5. Data Models

### 5.1 Product Model
| Field | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `product_id` | string | Yes | Internal unique ID. |
| `sku` | string | Yes | Merchant-facing unique identifier. |
| `name` | string | Yes | Product Title. |
| `slug` | string | Yes | URL-friendly version of the name. |
| `short_description`| string | Rec. | Brief summary for listing pages. |
| `long_description` | string | No | Detailed HTML/Markdown description. |
| `main_image_url` | string | Yes | Primary high-resolution image URL. |
| `gallery_images` | array | No | List of additional image URLs. |
| `category` | string | Yes | Primary product category. |
| `attributes` | object | No | Key-value pairs for specs (e.g., `{"weight": "500g"}`). |

### 5.2 Pricing Model
| Field | Type | Required | Format |
| :--- | :--- | :--- | :--- |
| `price` | number | Yes | Base selling price. |
| `currency` | string | Yes | ISO 4217 (e.g., "INR"). |
| `discount_price` | number | No | Promotional price if applicable. |
| `tax_percentage` | number | Yes | Applicable GST/Tax rate. |
| `is_tax_inclusive` | boolean | Yes | Whether the price includes tax. |

### 5.3 Inventory Model
| Field | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `total_quantity` | integer | Yes | Total physical stock in warehouse. |
| `reserved_quantity`| integer | Yes | Stock locked for pending orders. |
| `available_quantity`| integer | Yes | Calculated as `total` - `reserved`. |
| `stock_status` | enum | Yes | `IN_STOCK`, `OUT_OF_STOCK`, `LIMITED_STOCK`. |
| `min_stock_level` | integer | No | Threshold for `LIMITED_STOCK` status. |

 

## 6. API Endpoints (Reference Specification)

The following endpoints represent the required interface for the integration. The ERP team should provide these (or equivalent) endpoints.

### 6.1 Product Management

#### List Products
`GET /api/products`
*   **Query Params:** `page`, `limit`, `category`, `q`.
*   **Response:** `200 OK` with Product list.

#### Get Product by SKU
`GET /api/products/{sku}`
*   **Response:** `200 OK` with Product object.

#### Create Product
`POST /api/products`
*   **Request Body:** Product object (excluding `product_id`).
*   **Response:** `201 Created` with new Product object.

#### Update Product
`PUT /api/products/{sku}`
*   **Request Body:** Partial or full Product object.
*   **Response:** `200 OK` with updated Product object.

#### Delete Product
`DELETE /api/products/{sku}`
*   **Response:** `204 No Content`.

### 6.2 Pricing Management

#### Get Pricing
`GET /api/pricing/{sku}`
*   **Response:** `200 OK` with Pricing object.

#### Update Pricing
`PUT /api/pricing/{sku}`
*   **Request Body:** Pricing object.
*   **Response:** `200 OK` with updated Pricing object.

### 6.3 Inventory Management

#### Check Inventory Level
`GET /api/inventory/{sku}`
*   **Response:** `200 OK` with Inventory object.

#### Update Inventory (Manual Adjustment)
`PUT /api/inventory/{sku}`
*   **Request Body:** `{"total_quantity": 100}`
*   **Response:** `200 OK` with updated Inventory object.

#### Reserve Inventory
`POST /api/inventory/reserve`
*   **Request Body:** `{"order_id": "O-98765", "sku": "DRN-001", "quantity": 2, "expires_in_seconds": 600}`
*   **Response:** `201 Created`.

#### Release Reservation
`POST /api/inventory/release`
*   **Request Body:** `{"order_id": "O-98765"}`
*   **Response:** `200 OK`.

 

## 7. Webhooks (Real-time Sync)

The ERP system must push notifications to the e-commerce middleware for the following events:

| Event | Trigger | Payload Includes |
| :--- | :--- | :--- |
| `inventory.updated` | Any change in stock level | `sku`, `new_available_qty` |
| `price.updated` | Change in base or discount price | `sku`, `new_price`, `currency` |
| `product.status_changed`| Product enabled/disabled | `sku`, `is_active` |

 

## 8. Error Handling

All error responses must follow this structure:
```json
{
  "error": {
    "code": "INSUFFICIENT_STOCK",
    "message": "Requested quantity exceeds available stock.",
    "request_id": "req-abc-123"
  }
}
```

| Status Code | Meaning |
| :--- | :--- |
| `400 Bad Request` | Invalid parameters or malformed JSON. |
| `401 Unauthorized` | Missing or invalid authentication token. |
| `403 Forbidden` | Authenticated but lacks permission for the resource. |
| `404 Not Found` | Resource (SKU/Product) does not exist. |
| `409 Conflict` | Business logic conflict (e.g., stock unavailable). |
| `500 Internal Error` | Unexpected server-side failure. |

 

## 9. Performance Requirements
*   **Latency:** 95% of GET requests must resolve within **150ms**.
*   **Availability:** The API must maintain **99.9% uptime**.
*   **Concurrency:** Support at least **50 concurrent inventory reservations** per second.

 

## 10. ERP Team Deliverables

To facilitate a successful integration, the ERP team is required to provide the following:

### 10.1 Sandbox Environment
A dedicated **Staging/UAT environment** that mirrors the production system. This environment must be accessible for integration testing without affecting live data.

### 10.2 API Credentials
A set of **API Keys or Bearer Tokens** for the Sandbox environment to enable authenticated requests during development.

### 10.3 Sample Test Data
A comprehensive set of sample data in the Sandbox environment, including:
*   At least 10 products with different categories and attributes.
*   Varying stock levels (In Stock, Out of Stock, Limited Stock).
*   Active and inactive product statuses.

### 10.4 Documentation & Collections
*   **Postman Collection:** A pre-configured Postman collection containing all endpoints with sample request bodies and environment variables.
*   **OpenAPI Specification:** A Swagger/OpenAPI (JSON/YAML) file defining the API schema, endpoints, and security requirements.
