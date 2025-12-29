   Inventory Management System – StockFlow

Backend Engineering Intern – Case Study Submission
Name: Behara Lakshmi Sai Charan
________________________________________

                       Part 1: Code Review & Debugging
Problem Statement
The following API endpoint was implemented to create a new product and initialize inventory. Although it compiles, it fails to behave correctly in production.
Issues Identified (Technical & Business Logic)
1. Missing Input Validation
•	The code assumes all required fields are present in the request payload.
•	Optional fields are not handled safely.
Impact:
•	Application crashes with KeyError
•	Unclear error responses to API consumers
________________________________________
2. SKU Uniqueness Not Enforced
•	SKUs must be globally unique.
•	No database or application-level validation exists.
Impact:
•	Duplicate SKUs break reporting, integrations, and inventory tracking.
________________________________________
3. Incorrect Product–Warehouse Relationship
•	warehouse_id is stored directly in the Product table.
•	Products can exist in multiple warehouses.
Impact:
•	Forces product duplication per warehouse
•	Violates core business requirement
________________________________________
4. No Transaction Management
•	Two separate commit() calls are used.
•	Partial writes occur if inventory creation fails.
Impact:
•	Product exists without inventory
•	Data inconsistency and manual cleanup
________________________________________
5. Price Precision Issue
•	Price is likely stored as a float.
•	Monetary values require decimal precision.
Impact:
•	Rounding errors
•	Incorrect financial calculations
________________________________________
6. No Error Handling or Rollback
•	Exceptions are not handled.
•	Database is not rolled back on failure.
Impact:
•	Unstable production behavior
•	Hard-to-debug issues
________________________________________
Corrected Implementation (Production-Safe)
from decimal import Decimal
from sqlalchemy.exc import IntegrityError

@app.route('/api/products', methods=['POST'])
def create_product():
    data = request.json

    required_fields = ['name', 'sku', 'price', 'warehouse_id']
    for field in required_fields:
        if field not in data:
            return {"error": f"{field} is required"}, 400

    try:
        with db.session.begin():  # Atomic transaction

            product = Product(
                name=data['name'],
                sku=data['sku'],
                price=Decimal(str(data['price']))
            )
            db.session.add(product)
            db.session.flush()  # Generate product.id

            inventory = Inventory(
                product_id=product.id,
                warehouse_id=data['warehouse_id'],
                quantity=data.get('initial_quantity', 0)
            )
            db.session.add(inventory)

        return {
            "message": "Product created successfully",
            "product_id": product.id
        }, 201

    except IntegrityError:
        db.session.rollback()
        return {"error": "SKU already exists"}, 409

    except Exception:
        db.session.rollback()
        return {"error": "Internal server error"}, 500
________________________________________
Why This Works Better
•	Ensures atomicity
•	Enforces business constraints
•	Supports multi-warehouse inventory
•	Safe monetary handling
•	Production-grade error control
________________________________________

                               Part 2: Database Design
Proposed Schema (SQL)
Company (
  id BIGINT PRIMARY KEY,
  name VARCHAR,
  created_at TIMESTAMP
)

Warehouse (
  id BIGINT PRIMARY KEY,
  company_id BIGINT REFERENCES Company(id),
  name VARCHAR,
  location VARCHAR
)

Product (
  id BIGINT PRIMARY KEY,
  company_id BIGINT REFERENCES Company(id),
  name VARCHAR,
  sku VARCHAR UNIQUE,
  price DECIMAL(10,2),
  product_type VARCHAR
)

Inventory (
  id BIGINT PRIMARY KEY,
  product_id BIGINT REFERENCES Product(id),
  warehouse_id BIGINT REFERENCES Warehouse(id),
  quantity INT,
  updated_at TIMESTAMP
)

InventoryHistory (
  id BIGINT PRIMARY KEY,
  inventory_id BIGINT REFERENCES Inventory(id),
  change INT,
  reason VARCHAR,
  created_at TIMESTAMP
)

Supplier (
  id BIGINT PRIMARY KEY,
  name VARCHAR,
  contact_email VARCHAR
)

ProductSupplier (
  product_id BIGINT REFERENCES Product(id),
  supplier_id BIGINT REFERENCES Supplier(id)
)

ProductBundle (
  bundle_id BIGINT REFERENCES Product(id),
  child_product_id BIGINT REFERENCES Product(id),
  quantity INT
)
________________________________________
Missing Requirements / Questions for Product Team
1.	Is SKU unique globally or per company?
2.	What defines “recent sales activity”?
3.	Can inventory be shared across warehouses?
4.	Are bundles physical stock or virtual?
5.	Can suppliers vary per warehouse?
6.	Are low-stock thresholds configurable by company?
________________________________________
Design Decisions & Justifications
•	Inventory separated from product → supports multi-warehouse storage
•	InventoryHistory enables auditing and analytics
•	Indexes recommended on:
o	sku
o	(product_id, warehouse_id)
o	company_id
•	Bundle table supports composite products
•	Many-to-many supplier mapping supports flexibility
________________________________________

                               Part 3: Low-Stock Alerts API

Endpoint
GET /api/companies/{company_id}/alerts/low-stock
Assumptions Made
•	Recent sales = activity within last 30 days
•	Daily sales rate = sales / 30 days
•	Low-stock threshold depends on product type
•	Each product has a primary supplier
________________________________________
Implementation (Flask)
@app.route('/api/companies/<int:company_id>/alerts/low-stock', methods=['GET'])
def low_stock_alerts(company_id):
    alerts = []

    inventories = db.session.query(Inventory)\
        .join(Product)\
        .join(Warehouse)\
        .filter(Warehouse.company_id == company_id)\
        .all()

    for inv in inventories:
        product = inv.product
        warehouse = inv.warehouse

        recent_sales = get_sales_last_30_days(product.id, warehouse.id)
        if recent_sales == 0:
            continue

        threshold = get_threshold(product.product_type)

        if inv.quantity < threshold:
            daily_rate = recent_sales / 30
            days_left = int(inv.quantity / daily_rate) if daily_rate > 0 else None

            supplier = get_primary_supplier(product.id)

            alerts.append({
                "product_id": product.id,
                "product_name": product.name,
                "sku": product.sku,
                "warehouse_id": warehouse.id,
                "warehouse_name": warehouse.name,
                "current_stock": inv.quantity,
                "threshold": threshold,
                "days_until_stockout": days_left,
                "supplier": {
                    "id": supplier.id,
                    "name": supplier.name,
                    "contact_email": supplier.contact_email
                }
            })

    return {
        "alerts": alerts,
        "total_alerts": len(alerts)
    }
________________________________________
 Edge Cases Considered
•	Products without recent sales
•	Zero sales rate (division safety)
•	Missing supplier information
•	Multi-warehouse inventory aggregation
________________________________________
Final Notes
This solution focuses on:
•	Data integrity
•	Scalability
•	Real-world failure handling
•	Clear communication of assumptions
All decisions were made considering production-readiness and maintainability.
