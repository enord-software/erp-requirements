# ERP & E-commerce Integration: Business Logic Overview

This document provides a high-level, non-technical explanation of how the Enord e-commerce website and the internal ERP system work together to manage products and inventory.

## 1. The Core Concept: Single Source of Truth
The **ERP (Enterprise Resource Planning)** system is the "Single Source of Truth" for all product data. This means that whatever information is in the ERP (names, prices, stock levels) is what will be shown to the public on the website.

## 2. How Inventory Synchronization Works
The primary goal of this integration is to ensure that the stock count is always accurate, preventing "overselling" (selling an item that is actually out of stock).

### A. Public Sales (Website to ERP)
When a customer visits the Enord website and makes a purchase:
1.  **Real-time Check:** The website asks the ERP if the item is available.
2.  **Reservation:** While the customer is checking out, the ERP "reserves" the item so no one else can buy it.
3.  **Stock Reduction:** Once the payment is successful, the ERP permanently reduces the quantity of that item in its database.

### B. Internal Usage (ERP to Website)
The ERP is also used internally by the Enord team for various business operations:
1.  **Internal Consumption:** If the internal team uses an item (e.g., for testing, demos, or internal projects), they record this action in the ERP.
2.  **Immediate Update:** As soon as the internal team reduces the stock in the ERP, the website is notified automatically.
3.  **Public Visibility:** The website immediately reflects the lower stock count, ensuring that public customers only see what is truly available for sale.

## 3. Benefits of this Integration
*   **No Manual Updates:** The website and ERP talk to each other automatically. No one needs to manually update stock levels on the website.
*   **Customer Trust:** Customers will never be able to buy an item that isn't actually in the warehouse.
*   **Operational Efficiency:** Internal teams can use stock for business needs without worrying about causing discrepancies on the public storefront.

---

*For technical details, please refer to the [ERP API Requirements](REQUIREMENTS.md) document.*

