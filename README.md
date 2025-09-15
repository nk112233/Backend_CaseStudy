# 📄 Case Study Submission

This document contains solutions for all three parts of the case study: debugging, database design, and API implementation.
Each section includes reasoning, assumptions, and explanations.

---

## **PART 1 – Debugging & Fixing Flask Code**

### 🔎 1. Identify Issues

#### Technical Issues

1. **Two commits (`db.session.commit()`)**

   * Commits after creating product and again after inventory.

2. **Using `product.id` before commit**

   * Primary keys may not be assigned until `flush()` or `commit()`.

3. **No input validation**

   * `data['field']` will raise `KeyError` if field missing.

4. **Price handling**

   * Assumes `price` is a float. Strings (e.g., `"12.50"`) could break.

5. **No uniqueness check for SKU**

   * Allows duplicate SKUs, violating business rules.

6. **No transaction rollback on failure**

   * Can leave orphan products without inventory.

7. **No error handling**

   * Server may crash and return 500.

#### Business Logic Issues

1. **Products can exist in multiple warehouses**

   * Current design links `warehouse_id` directly in Product.

2. **Initial quantity optional**

   * Code assumes always provided.

3. **Price precision**

   * Floats cause rounding issues. Should use `Decimal`.

4. **SKU uniqueness**

   * Must be enforced across the platform (or at least per company).

---

### ⚠️ 2. Explain Impact

* Inefficient commits → partial saves.
* Missing validations → app crashes.
* Wrong schema → duplicate products per warehouse.
* Floats → financial rounding bugs.
* No rollbacks → inconsistent data.

---

### ✅ 3. Corrected Code (Flask + SQLAlchemy)

```python
from flask import request, jsonify
from sqlalchemy.exc import IntegrityError
from decimal import Decimal
from app import app, db
from models import Product, Inventory

@app.route('/api/products', methods=['POST'])
def create_product():
    data = request.get_json()

    # 1. Validate input
    name = data.get("name")
    sku = data.get("sku")
    price = data.get("price")
    warehouse_id = data.get("warehouse_id")
    initial_quantity = data.get("initial_quantity", 0)

    if not name or not sku or price is None or not warehouse_id:
        return jsonify({"error": "Missing required fields"}), 400

    try:
        # 2. Ensure SKU uniqueness
        existing = Product.query.filter_by(sku=sku).first()
        if existing:
            return jsonify({"error": "SKU must be unique"}), 400

        # 3. Create product (warehouse-independent)
        product = Product(
            name=name,
            sku=sku,
            price=Decimal(str(price))
        )
        db.session.add(product)
        db.session.flush()  # assign product.id without full commit

        # 4. Create inventory record
        inventory = Inventory(
            product_id=product.id,
            warehouse_id=warehouse_id,
            quantity=initial_quantity
        )
        db.session.add(inventory)

        # 5. Commit once (atomic)
        db.session.commit()

        return jsonify({
            "message": "Product created",
            "product_id": product.id,
            "warehouse_id": warehouse_id,
            "initial_quantity": initial_quantity
        }), 201

    except IntegrityError:
        db.session.rollback()
        return jsonify({"error": "Database integrity error"}), 400

    except Exception as e:
        db.session.rollback()
        return jsonify({"error": str(e)}), 500
```

---

### 📝 4. Fixes Summary

* Validation for required fields.
* SKU uniqueness check.
* `Decimal` for price.
* `flush()` before using `product.id`.
* Single atomic commit.
* Rollbacks on failure.
* Error handling with JSON.
* Product not tied to one warehouse.

---

## **PART 2 – Database Schema Design**

### 📊 Schema (Postgres/MySQL DDL Style)

```sql
-- Companies
CREATE TABLE companies (
  company_id BIGSERIAL PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Warehouses
CREATE TABLE warehouses (
  warehouse_id BIGSERIAL PRIMARY KEY,
  company_id BIGINT NOT NULL REFERENCES companies(company_id) ON DELETE CASCADE,
  name VARCHAR(255) NOT NULL,
  location VARCHAR(255),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Products
CREATE TABLE products (
  product_id BIGSERIAL PRIMARY KEY,
  company_id BIGINT NOT NULL REFERENCES companies(company_id) ON DELETE CASCADE,
  name VARCHAR(255) NOT NULL,
  sku VARCHAR(100) NOT NULL,
  price DECIMAL(12,2) NOT NULL,
  is_bundle BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  UNIQUE (company_id, sku)
);

-- Inventory
CREATE TABLE inventory (
  inventory_id BIGSERIAL PRIMARY KEY,
  product_id BIGINT NOT NULL REFERENCES products(product_id) ON DELETE CASCADE,
  warehouse_id BIGINT NOT NULL REFERENCES warehouses(warehouse_id) ON DELETE CASCADE,
  quantity INT NOT NULL DEFAULT 0,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  UNIQUE (product_id, warehouse_id)
);

-- Inventory Movements
CREATE TABLE inventory_movements (
  movement_id BIGSERIAL PRIMARY KEY,
  inventory_id BIGINT NOT NULL REFERENCES inventory(inventory_id) ON DELETE CASCADE,
  change_amount INT NOT NULL,
  reason VARCHAR(255),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Suppliers
CREATE TABLE suppliers (
  supplier_id BIGSERIAL PRIMARY KEY,
  company_id BIGINT NOT NULL REFERENCES companies(company_id) ON DELETE CASCADE,
  name VARCHAR(255) NOT NULL,
  contact_info TEXT
);

-- Supplier ↔ Products
CREATE TABLE supplier_products (
  supplier_id BIGINT NOT NULL REFERENCES suppliers(supplier_id) ON DELETE CASCADE,
  product_id BIGINT NOT NULL REFERENCES products(product_id) ON DELETE CASCADE,
  PRIMARY KEY (supplier_id, product_id)
);

-- Product Bundles
CREATE TABLE product_bundles (
  bundle_id BIGINT NOT NULL REFERENCES products(product_id) ON DELETE CASCADE,
  component_id BIGINT NOT NULL REFERENCES products(product_id) ON DELETE CASCADE,
  quantity INT NOT NULL DEFAULT 1,
  PRIMARY KEY (bundle_id, component_id)
);
```

---

### ❓ Gaps / Questions for Product Team

1. Should SKUs be unique **globally** or **per company**?
2. Should inventory movements track **who** made the change?
3. Can bundles contain bundles (nested)?
4. Do suppliers supply multiple companies? Should we track supplier pricing?
5. Do warehouses need capacity constraints?
6. Do products need categories, descriptions, images?
7. Should inventory support fractional quantities (e.g., weight)?

---

### ⚖️ Design Decisions

* **BIGSERIAL IDs** for scalability.
* **UNIQUE (company\_id, sku)** to prevent duplicates.
* **inventory\_movements table** for audit trail.
* **product\_bundles self-join** for flexibility.
* **Indexes** on SKUs, inventory lookups, and movements history.
* **ON DELETE CASCADE** to avoid orphan rows.

---

## **PART 3 – API Implementation (Express.js)**

### 📌 Endpoint

```js
// routes/alerts.js
const express = require("express");
const router = express.Router();
const db = require("../db"); // Sequelize or query wrapper

/**
 * GET /api/companies/:company_id/alerts/low-stock
 * Returns low-stock alerts for a company
 */
router.get("/companies/:company_id/alerts/low-stock", async (req, res) => {
  const { company_id } = req.params;

  try {
    // 1. Get products with sales in last 30 days
    const recentSalesProducts = await db.query(
      `
      SELECT DISTINCT product_id
      FROM sales
      WHERE company_id = $1
      AND sold_at >= NOW() - INTERVAL '30 days'
      `,
      { bind: [company_id], type: db.QueryTypes.SELECT }
    );
    const activeProductIds = recentSalesProducts.map(r => r.product_id);

    if (activeProductIds.length === 0) {
      return res.json({ alerts: [], total_alerts: 0 });
    }

    // 2. Get stock per product per warehouse
    const stockData = await db.query(
      `
      SELECT 
        p.id AS product_id, p.name AS product_name, p.sku, p.threshold,
        i.quantity AS current_stock,
        w.id AS warehouse_id, w.name AS warehouse_name,
        s.id AS supplier_id, s.name AS supplier_name, s.contact_info
      FROM inventory i
      JOIN products p ON i.product_id = p.id
      JOIN warehouses w ON i.warehouse_id = w.id
      LEFT JOIN supplier_products sp ON sp.product_id = p.id
      LEFT JOIN suppliers s ON sp.supplier_id = s.id
      WHERE p.company_id = $1
      AND p.id = ANY($2)
      `,
      { bind: [company_id, activeProductIds], type: db.QueryTypes.SELECT }
    );

    // 3. Filter & format alerts
    const alerts = stockData
      .filter(item => item.current_stock < item.threshold)
      .map(item => {
        const daysUntilStockout = item.current_stock > 0 ? item.current_stock : 0;

        return {
          product_id: item.product_id,
          product_name: item.product_name,
          sku: item.sku,
          warehouse_id: item.warehouse_id,
          warehouse_name: item.warehouse_name,
          current_stock: item.current_stock,
          threshold: item.threshold,
          days_until_stockout: daysUntilStockout,
          supplier: item.supplier_id
            ? {
                id: item.supplier_id,
                name: item.supplier_name,
                contact_email: item.contact_info || null
              }
            : null
        };
      });

    res.json({ alerts, total_alerts: alerts.length });
  } catch (err) {
    console.error("Error fetching low-stock alerts:", err);
    res.status(500).json({ error: "Internal Server Error" });
  }
});

module.exports = router;
```

---

### ⚠️ Edge Cases Considered

1. No recent sales → empty result.
2. No inventory row → skip product.
3. Missing supplier info → supplier = `null`.
4. Negative stock → show `0`.
5. Missing threshold → assumed mandatory.
6. Bundles ignored (only components tracked).

---

### 📝 Approach Explanation

1. Query recent sales → only alert for active products.
2. Join products, inventory, warehouses, suppliers.
3. Filter `quantity < threshold`.
4. Calculate `days_until_stockout` (simplified).
5. Return JSON in required format.

---

✅ **Submission complete.**
This document contains debugging, schema design, and API implementation with reasoning, assumptions, and trade-offs.

---
