**PART 1**

**1\. Identify Issues**

**Technical Issues**

1.  **Two commits (db.session.commit())**
    *   You commit after creating the product and then again after creating inventory.
2.  **Using product.id before commit**
    *   In SQLAlchemy, primary keys may not be assigned until after a commit() or flush(). Using it immediately could cause issues.
3.  **No input validation**
    *   Directly using data\['field'\] will throw KeyError if the field is missing.
4.  **Price handling**
    *   price=data\['price'\] assumes it’s already a float. If a string is passed (e.g., "12.50"), it may fail depending on the DB model.
5.  **No uniqueness check for SKU**
    *   If two products with the same SKU are inserted, it may break business rules or raise a DB integrity error.
6.  **No transaction rollback on failure**
    *   If creating the product succeeds but inventory fails, you’ll end up with a product without inventory.
7.  **Error handling missing**
    *   No try/except → server could crash and return 500 with no useful message.

**Business Logic Issues**

1.  **Products can exist in multiple warehouses**
    *   Current code links warehouse\_id directly in Product, but business rule suggests **Product should be warehouse-independent**. Inventory should connect product ↔ warehouse.
2.  **Initial quantity optional**
    *   Code assumes initial\_quantity is always provided.
3.  **Price must allow decimals**
    *   Should use Decimal type for precision (avoid float rounding issues in financial systems).
4.  **SKU uniqueness across platform**
    *   Must check if SKU exists before inserting.

**2\. Explain Impact**

1.  **Two commits** → inefficiency, partial data if the second commit fails.
2.  **Product ID issue** → might throw errors or create orphan inventory rows.
3.  **No validation** → app crashes on missing fields → bad UX.
4.  **Price floats** → rounding errors (e.g., 0.1 + 0.2 ≠ 0.3). Financial bugs in production.
5.  **No SKU check** → duplicate products, breaking search/inventory consistency.
6.  **No rollback** → orphan products without inventory.
7.  **Wrong model design (warehouse\_id inside Product)** → duplicates of the same product per warehouse, breaking "central catalog" idea.
8.  **No error handling** → server exposes stack traces or returns 500.

**3\. Corrected Version**

Here’s a **best-practice Flask + SQLAlchemy** rewrite with fixes and explanations inline:

from flask import request, jsonify

from sqlalchemy.exc import IntegrityError

from decimal import Decimal

from app import app, db

from models import Product, Inventory

@app.route('/api/products', methods=\['POST'\])

def create\_product():

data = request.get\_json()

\# 1. Validate input safely

name = data.get("name")

sku = data.get("sku")

price = data.get("price")

warehouse\_id = data.get("warehouse\_id")

initial\_quantity = data.get("initial\_quantity", 0)

if not name or not sku or price is None or not warehouse\_id:

return jsonify({"error": "Missing required fields"}), 400

try:

\# 2. Ensure SKU uniqueness across platform

existing = Product.query.filter\_by(sku=sku).first()

if existing:

return jsonify({"error": "SKU must be unique"}), 400

\# 3. Create product (warehouse-independent)

product = Product(

name=name,

sku=sku,

price=Decimal(str(price)) # handle decimal safely

)

db.session.add(product)

db.session.flush() # assign product.id without committing

\# 4. Create inventory record for this warehouse

inventory = Inventory(

product\_id=product.id,

warehouse\_id=warehouse\_id,

quantity=initial\_quantity

)

db.session.add(inventory)

\# 5. Commit once, as an atomic transaction

db.session.commit()

return jsonify({

"message": "Product created",

"product\_id": product.id,

"warehouse\_id": warehouse\_id,

"initial\_quantity": initial\_quantity

}), 201

except IntegrityError:

db.session.rollback()

return jsonify({"error": "Database integrity error"}), 400

except Exception as e:

db.session.rollback()

return jsonify({"error": str(e)}), 500

**4\. Explanation of Fixes**

1.  **Validation** → prevents missing field crashes.
2.  **SKU uniqueness check** → enforces business rule.
3.  **Decimal for price** → avoids float rounding errors.
4.  **Flush before using product.id** → ensures ID exists before linking inventory.
5.  **Single commit** → both product + inventory saved atomically.
6.  **Rollback on failure** → prevents orphan data.
7.  **Error handling with JSON responses** → better API UX.
8.  **Product model warehouse-independent** → supports multiple warehouses correctly.

**PART 2**

1\. Database Schema Design
==========================

**SQL DDL style notation** (MySQL/Postgres-like).

\-- Companies table

CREATE TABLE companies (

company\_id BIGSERIAL PRIMARY KEY,

name VARCHAR(255) NOT NULL,

created\_at TIMESTAMP DEFAULT CURRENT\_TIMESTAMP

);

\-- Warehouses belong to companies

CREATE TABLE warehouses (

warehouse\_id BIGSERIAL PRIMARY KEY,

company\_id BIGINT NOT NULL REFERENCES companies(company\_id) ON DELETE CASCADE,

name VARCHAR(255) NOT NULL,

location VARCHAR(255),

created\_at TIMESTAMP DEFAULT CURRENT\_TIMESTAMP

);

\-- Products (SKU unique across platform, company-specific)

CREATE TABLE products (

product\_id BIGSERIAL PRIMARY KEY,

company\_id BIGINT NOT NULL REFERENCES companies(company\_id) ON DELETE CASCADE,

name VARCHAR(255) NOT NULL,

sku VARCHAR(100) NOT NULL,

price DECIMAL(12,2) NOT NULL,

is\_bundle BOOLEAN DEFAULT FALSE,

created\_at TIMESTAMP DEFAULT CURRENT\_TIMESTAMP,

UNIQUE (company\_id, sku) -- enforce SKU uniqueness per company

);

\-- Inventory: product stored in multiple warehouses

CREATE TABLE inventory (

inventory\_id BIGSERIAL PRIMARY KEY,

product\_id BIGINT NOT NULL REFERENCES products(product\_id) ON DELETE CASCADE,

warehouse\_id BIGINT NOT NULL REFERENCES warehouses(warehouse\_id) ON DELETE CASCADE,

quantity INT NOT NULL DEFAULT 0,

updated\_at TIMESTAMP DEFAULT CURRENT\_TIMESTAMP,

UNIQUE (product\_id, warehouse\_id) -- one row per product per warehouse

);

\-- Inventory history: track stock changes

CREATE TABLE inventory\_movements (

movement\_id BIGSERIAL PRIMARY KEY,

inventory\_id BIGINT NOT NULL REFERENCES inventory(inventory\_id) ON DELETE CASCADE,

change\_amount INT NOT NULL, -- positive for add, negative for remove

reason VARCHAR(255), -- e.g., "purchase order", "sale", "manual adjustment"

created\_at TIMESTAMP DEFAULT CURRENT\_TIMESTAMP

);

\-- Suppliers

CREATE TABLE suppliers (

supplier\_id BIGSERIAL PRIMARY KEY,

company\_id BIGINT NOT NULL REFERENCES companies(company\_id) ON DELETE CASCADE,

name VARCHAR(255) NOT NULL,

contact\_info TEXT

);

\-- Supplier supplies products

CREATE TABLE supplier\_products (

supplier\_id BIGINT NOT NULL REFERENCES suppliers(supplier\_id) ON DELETE CASCADE,

product\_id BIGINT NOT NULL REFERENCES products(product\_id) ON DELETE CASCADE,

PRIMARY KEY (supplier\_id, product\_id)

);

\-- Bundles: a product can contain other products

CREATE TABLE product\_bundles (

bundle\_id BIGINT NOT NULL REFERENCES products(product\_id) ON DELETE CASCADE,

component\_id BIGINT NOT NULL REFERENCES products(product\_id) ON DELETE CASCADE,

quantity INT NOT NULL DEFAULT 1,

PRIMARY KEY (bundle\_id, component\_id)

);

2\. Identify Gaps
=================

1.  **Multi-company SKU uniqueness**
    *   Should SKUs be **globally unique** (across all companies) or just unique **per company**?
    *   Right now, I assumed **per company**.
2.  **Inventory movements**
    *   Do we need to track **who** made the change (user\_id)?
    *   Do we need **transaction references** (e.g., sale order ID, purchase order ID)?
3.  **Bundles**
    *   Can bundles contain other bundles (nested bundles)?
    *   Should inventory of bundles be tracked directly, or derived from their components?
4.  **Suppliers**
    *   Can the same supplier supply products to **multiple companies**?
    *   Do we track **supplier pricing** per product?
5.  **Warehouses**
    *   Do warehouses need capacity/space constraints?
    *   Can a warehouse belong to multiple companies (shared logistics)?
6.  **Products**
    *   Do products have categories (electronics, apparel)?
    *   Do they need descriptions, images, metadata?
7.  **Inventory precision**
    *   Is quantity always an integer (e.g., units), or do we need decimals (e.g., weight, liters)?

⚖️ 3. Design Decisions & Justifications
=======================================

*   **BIGSERIAL / BIGINT IDs** → future scalability.
*   **UNIQUE (company\_id, sku)** → enforces SKU uniqueness per company (safe for multi-tenant).
*   **Separate inventory\_movements table** → avoids losing history; audit trail is critical in production.
*   **product\_bundles self-referencing join table** → flexible, allows bundles of multiple products.
*   **Indexes**
    *   Index on products(sku, company\_id) → fast lookups.
    *   Index on inventory(product\_id, warehouse\_id) → ensures quick queries for stock per warehouse.
    *   Index on inventory\_movements(inventory\_id, created\_at) → optimize reporting on stock changes.
*   **Foreign keys with ON DELETE CASCADE** → prevents orphan records when company/warehouse/product is deleted.

**PART 3**

📌 1. Implementation (Express.js)
=================================

// routes/alerts.js

const express = require("express");

const router = express.Router();

const db = require("../db"); // assume Sequelize or a query wrapper

/\*\*

\* GET /api/companies/:company\_id/alerts/low-stock

\* Returns low-stock alerts for a given company.

\*/

router.get("/companies/:company\_id/alerts/low-stock", async (req, res) => {

const { company\_id } = req.params;

try {

// ------------------------

// ASSUMPTIONS:

// - products table: { id, name, sku, company\_id, threshold, is\_bundle }

// - inventory table: { id, product\_id, warehouse\_id, quantity }

// - warehouses table: { id, name, company\_id }

// - suppliers + supplier\_products join table

// - sales table: { id, product\_id, sold\_at, quantity }

// ------------------------

// 1. Find products with recent sales (last 30 days)

const recentSalesProducts = await db.query(

\`

SELECT DISTINCT product\_id

FROM sales

WHERE company\_id = $1

AND sold\_at >= NOW() - INTERVAL '30 days'

\`,

{ bind: \[company\_id\], type: db.QueryTypes.SELECT }

);

const activeProductIds = recentSalesProducts.map(r => r.product\_id);

if (activeProductIds.length === 0) {

return res.json({ alerts: \[\], total\_alerts: 0 });

}

// 2. Get current stock per product per warehouse

const stockData = await db.query(

\`

SELECT

p.id AS product\_id, p.name AS product\_name, p.sku, p.threshold,

i.quantity AS current\_stock,

w.id AS warehouse\_id, w.name AS warehouse\_name,

s.id AS supplier\_id, s.name AS supplier\_name, s.contact\_info

FROM inventory i

JOIN products p ON i.product\_id = p.id

JOIN warehouses w ON i.warehouse\_id = w.id

LEFT JOIN supplier\_products sp ON sp.product\_id = p.id

LEFT JOIN suppliers s ON sp.supplier\_id = s.id

WHERE p.company\_id = $1

AND p.id = ANY($2)

\`,

{ bind: \[company\_id, activeProductIds\], type: db.QueryTypes.SELECT }

);

// 3. Filter to only low-stock items

const alerts = stockData

.filter(item => item.current\_stock < item.threshold)

.map(item => {

// Simplified: assume average daily sales = 1, so stockout = current\_stock days

// In real-world, would calculate based on sales velocity

const daysUntilStockout = item.current\_stock > 0 ? item.current\_stock : 0;

return {

product\_id: item.product\_id,

product\_name: item.product\_name,

sku: item.sku,

warehouse\_id: item.warehouse\_id,

warehouse\_name: item.warehouse\_name,

current\_stock: item.current\_stock,

threshold: item.threshold,

days\_until\_stockout: daysUntilStockout,

supplier: item.supplier\_id

? {

id: item.supplier\_id,

name: item.supplier\_name,

contact\_email: item.contact\_info || null

}

: null

};

});

res.json({ alerts, total\_alerts: alerts.length });

} catch (err) {

console.error("Error fetching low-stock alerts:", err);

res.status(500).json({ error: "Internal Server Error" });

}

});

module.exports = router;

⚠️ 2. Edge Cases Considered
===========================

1.  **No recent sales** → return empty alerts.
2.  **No inventory record** → product not in warehouses, skip.
3.  **No supplier info** → still return product, but supplier = null.
4.  **Zero/negative stock** → handle gracefully (show 0, not negative days).
5.  **Missing threshold** → could default to a system-wide value (but here assumed mandatory).
6.  **Bundles** → skipped in this implementation (assumption: bundles aren’t tracked directly, only components).

📝 3. Approach Explanation
==========================

*   **Step 1:** Query recent sales to identify “active” products (business rule: only show alerts for products with recent sales).
*   **Step 2:** Join products + inventory + warehouses + suppliers to get all needed info.
*   **Step 3:** Filter where quantity < threshold.
*   **Step 4:** Calculate days\_until\_stockout. Here simplified (stock ÷ avg daily sales would be real approach).
*   **Step 5:** Return JSON in required format.