# Mint Classics Inventory Analysis and Recommendations

## Business Problem
Mint Classics Company, a retailer of classic model cars and other vehicles, is looking to optimize its inventory storage. The company's goal is to reduce overall inventory and explore the possibility of closing one of its storage facilities while still maintaining its commitment to a 24-hour shipping turnaround time. The objective of this project is to analyze the inventory data, identify slow-moving and dead stock, and provide data-driven recommendations on warehouse consolidation and inventory reduction.

## Analysis Conducted
The provided relational database consists of 9 tables: `customers`, `employees`, `offices`, `orderdetails`, `orders`, `payments`, `productlines`, `products`, and `warehouses`. The core of the analysis focused on the `products`, `warehouses`, and `orderdetails` tables.

The analysis was conducted in the following steps:
1. **Warehouse Distribution:** Evaluated the number of unique products and total units in stock stored in each warehouse.
2. **Sales-to-Stock Ratio:** Joined the `products` table with historical sales data from the `orderdetails` table to calculate the sales-to-stock ratio for each product. 
3. **Slow-Moving Products Identification:** Isolated products that had exceptionally low sales compared to their on-hand inventory levels (defined as a ratio of < 0.3).
4. **Dead Stock Identification:** Checked for any products that had zero recorded sales.

### Observations per Warehouse
An initial evaluation of the inventory distribution revealed the following total units in stock per warehouse:
*   **Warehouse A (North):** 25 products — 131,688 total units
*   **Warehouse B (East):** 38 products — 219,183 total units
*   **Warehouse C (West):** 24 products — 124,880 total units
*   **Warehouse D (South):** 23 products — 79,380 total units

Warehouse D clearly stood out as the facility holding the lowest volume of total inventory with the fewest unique products.

When investigating the sales-to-stock ratios for products within Warehouse D, significant discrepancies were found. Many products have ratios below 0.2. Some notable examples include:
*   **1950's Chicago Surface Lines Streetcar:** 8,601 units in stock vs. 934 sold (0.11 ratio)
*   **1964 Mercedes Tour Bus:** 8,258 units in stock vs. 1,053 sold (0.13 ratio)
*   **Collectable Wooden Train:** 6,450 units in stock vs. 918 sold (0.14 ratio)

However, a few fast-moving products do exist within Warehouse D, such as the *Pont Yacht* (2.31 ratio) and *The Mayflower* (1.22 ratio), which must be accounted for if the warehouse is to be closed.

## Key Findings
*   **Warehouse Storage Discrepancies:** Inventory is highly skewed towards Warehouse B (over 219,000 units), while Warehouse D is substantially underutilized with under 80,000 units. Warehouse D holds less than 15% of the company's total product volume.
*   **Excessive Stock Relative to Demand:** Multiple products are heavily overstocked. Certain items have inventory quantities that are roughly 9 times the historical demand (e.g., 8,601 units in stock vs. 934 units sold). 
*   **Slow-Moving Stock Distribution:** Low-demand, high-stock products (ratio < 0.3) are present in all warehouses, but they make up the majority of Warehouse D's inventory.


## Recommendations
Based on the data derived from the SQL queries, I recommend the following actionable steps:

**1. Close Warehouse D (South Facility)**
Warehouse D is the clear candidate for closure. It holds the lowest amount of stock (79,380 units) and houses only 23 product lines. Since the majority of these 23 lines are significantly overstocked and slow-moving, it will be feasible to close this facility. Fast-selling products currently stored in Warehouse D, such as the *Pont Yacht* and *The Mayflower*, can be relocated to Warehouses A, B, or C to ensure the 24-hour shipping capability is not disrupted.

**2. Implement an Inventory Reduction Strategy**
Across all warehouses, the company should reduce the levels of overstocked, slow-moving items (sales-to-stock ratio < 0.3) and completely remove any identified dead stock. Specifically for Warehouse D items like the *1950's Chicago Streetcar* and *1964 Mercedes Tour Bus*, inventory levels are unjustifiably high relative to sales. Mint Classics should consider halting procurement on these items, running discount promotions to liquidate the excessive stock, or discontinuing the lowest performers altogether. This will successfully free up space across the remaining three warehouses to compensate for the closure of Warehouse D.

## Approach / Methodology
The approach started with a top-down view of the business problem: finding the optimal way to reduce inventory and close a warehouse without disrupting operations.

1.  **Exploration:** I began by querying the total product count and unit capacity of every warehouse to find the weakest link. Warehouse D immediately became the primary target for closure.
2.  **Validation:** It wasn't enough to just look at raw capacity; I needed to determine if the products inside Warehouse D were actually valuable to day-to-day operations. By joining the `products` table with the `orderdetails` table, I established a "sales-to-stock ratio" to measure product demand against current inventory constraints.
3.  **Synthesis:** The aggregate query grouping (via `GROUP BY`) and aggregate evaluation (via `HAVING`) confirmed that the vast majority of Warehouse D's items are simply sitting on shelves. Consequently, a logical plan was formulated to distribute the successful products while eliminating the warehouse and the bloated stock holding back operations.

## SQL Scripts

### 1. Identify Product Counts per Warehouse
This query reveals the number of unique product lines stored in each warehouse.
```sql
SELECT 
    p.warehouseCode,
    w.warehouseName,
    COUNT(*) AS total_products
FROM products p
JOIN warehouses w
    ON p.warehouseCode = w.warehouseCode
GROUP BY p.warehouseCode, w.warehouseName;
```

### 2. Identify Total Units in Stock per Warehouse
This query sums the total physical items remaining in each warehouse to identify the facility storing the least amount of inventory.
```sql
SELECT 
    warehouseCode,
    SUM(quantityInStock) AS total_units
FROM products
GROUP BY warehouseCode
ORDER BY total_units ASC;
```

### 3. Calculate Sales-to-Stock Ratio
This query compares the number of units physically in stock with the total number of units sold to determine how quickly items are moving off the shelves.
```sql
SELECT 
    p.warehouseCode,
    p.productCode,
    p.productName,
    p.quantityInStock,
    SUM(od.quantityOrdered) AS total_sold,
    ROUND(SUM(od.quantityOrdered)/quantityInStock, 2) AS sales_to_stock_ratio
FROM products p
LEFT JOIN orderdetails od 
    ON p.productCode = od.productCode
GROUP BY p.productCode, p.warehouseCode
ORDER BY sales_to_stock_ratio ASC;
```

### 4. Isolate Slow-Moving Products
Building on the previous query, this statement utilizes `HAVING` to filter and expose only the products with drastically low demand compared to stock availability (ratios below 0.3).
```sql
SELECT 
    p.warehouseCode,
    p.productCode,
    p.productName,
    p.quantityInStock,
    SUM(od.quantityOrdered) AS total_sold,
    ROUND(SUM(od.quantityOrdered)/quantityInStock, 2) AS sales_to_stock_ratio
FROM products p
LEFT JOIN orderdetails od 
    ON p.productCode = od.productCode
GROUP BY warehouseCode, productCode
HAVING sales_to_stock_ratio < 0.3
ORDER BY sales_to_stock_ratio ASC;
```

### 5. Identify Dead Stock (Zero Sales)
This query performs a `LEFT JOIN` and filters for null matches indicating products that exist in the system but have zero recorded orders.
```sql
SELECT 
    p.warehouseCode,
    p.productCode,
    p.productName,
    p.quantityInStock
FROM products p
LEFT JOIN orderdetails od 
    ON p.productCode = od.productCode
WHERE od.productCode IS NULL
ORDER BY warehouseCode;
```
