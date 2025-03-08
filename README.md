# SQL SELECTED QUERIES

### 3 Products Missing NetSuite ID

**Business Problem:**  
A product cannot sync to NetSuite unless it has a valid NetSuite ID. The OMS needs a list of all products that still need to be created or updated in NetSuite.

**Fields to Retrieve:**  
- `PRODUCT_ID`  
- `INTERNAL_NAME`  
- `PRODUCT_TYPE_ID`  
- `NETSUITE_ID` (or similar field indicating the NetSuite ID; may be `NULL` or empty if missing)

```
select
	p.PRODUCT_ID,
	p.INTERNAL_NAME ,
	p.PRODUCT_TYPE_ID ,
	gi.ID_VALUE NETSUITE_ID
from
	product p
left join good_identification gi on
	p.PRODUCT_ID = gi.PRODUCT_ID
	and gi.GOOD_IDENTIFICATION_TYPE_ID = 'ERP_ID'
	and gi.ID_VALUE is null;
```
# Cost: 835,541.9
# Explanation: It selects the PRODUCT_ID, INTERNAL_NAME, PRODUCT_TYPE_ID from the product table and the GOOD_IDENTIFICATION_TYPE_ID from the good_identification table where the GOOD_IDENTIFICATION_TYPE_ID is 'ERP_ID' (representing the NetSuite ID) and the ID_VALUE is null, indicating that the product either lacks or has an invalid NetSuite ID and needs to be updated in NetSuite.

### 5.3 Top-Selling Product in New York

**Business Problem:**  
Merchandising teams need to identify the best-selling product(s) in a specific region (New York) for targeted restocking or promotions.

**Fields to Retrieve:**  
- `PRODUCT_ID`  
- `INTERNAL_NAME`
- `TOTAL_QUANTITY_SOLD`  
- `CITY` / `STATE` (within New York region) 
- `REVENUE` (optionally, total sales amount)

```
select
	p.PRODUCT_ID ,
	p.INTERNAL_NAME,
	sum(oi.QUANTITY) TOTAL_QUANTITY_SOLD,
	sum(oi.UNIT_PRICE * oi.QUANTITY) AS REVENUE,
	pa.CITY ,
	pa.STATE_PROVINCE_GEO_ID
from
	order_item oi
join order_contact_mech ocm on
	ocm.ORDER_ID = oi.ORDER_ID and OCM.CONTACT_MECH_PURPOSE_TYPE_ID like '%LOCATION'
join postal_address pa on
	pa.CONTACT_MECH_ID = ocm.CONTACT_MECH_ID
	and pa.STATE_PROVINCE_GEO_ID = 'NY'
join product p on
	p.PRODUCT_ID = oi.PRODUCT_ID
group by
	p.PRODUCT_ID,
	pa.city,
	pa.STATE_PROVINCE_GEO_ID
having
	SUM(oi.QUANTITY) = (
	select
		MAX(total_quantity)
	from
		(
		select
			SUM(oi2.QUANTITY) as total_quantity,
			pa2.CITY
		from
			order_item oi2
		join order_contact_mech ocm2 on
			ocm2.ORDER_ID = oi2.ORDER_ID
		join postal_address pa2 on
			pa2.CONTACT_MECH_ID = ocm2.CONTACT_MECH_ID
		where
			pa2.STATE_PROVINCE_GEO_ID = 'NY'
		group by
			pa2.CITY,
			oi2.PRODUCT_ID
    ) as city_sales
	where
		city_sales.CITY = pa.CITY
)
```
# Cost-56,386.83

# EXPLANATION - The query starts with the order_item table because it contains details about products sold, including PRODUCT_ID, QUANTITY, and UNIT_PRICE.Next, order_contact_mech is joined to link orders to their shipping addresses.The postal_address table is then joined to retrieve location details such as CITY and STATE_PROVINCE_GEO_ID. The query filters for STATE_PROVINCE_GEO_ID = 'NY' to ensure that only orders from New York are considered. Yhe product table is also joined to obtain product-related details, specifically PRODUCT_ID and INTERNAL_NAME. Aggregations are performed to compute TOTAL_QUANTITY_SOLD and REVENUE. A subquery determines the maximum quantity sold in each city, and the HAVING clause ensures only the top-selling product per city is returned.



### 8.4 Items Where QOH and ATP Differ

**Business Problem:**  
Sometimes the **Quantity on Hand (QOH)** doesn’t match the **Available to Promise (ATP)** due to pending orders, reservations, or data discrepancies. This needs review for accurate fulfillment planning.

**Fields to Retrieve:**  
- `PRODUCT_ID`
- `FACILITY_ID`
- `QOH` (Quantity on Hand)  
- `ATP` (Available to Promise)  
- `DIFFERENCE` (QOH - ATP)

```
select
	ii.PRODUCT_ID ,
	ii.FACILITY_ID ,
	ii.QUANTITY_ON_HAND_TOTAL ,
	ii.AVAILABLE_TO_PROMISE_TOTAL,
	(ii.QUANTITY_ON_HAND_TOTAL -ii.AVAILABLE_TO_PROMISE_TOTAL) as DIFFERENCE
from
	inventory_item ii ;
```
# Cost - 217,246.4
# Explanation - This SQL query selects inventory details from the inventory_item table. It retrieves the PRODUCT_ID, FACILITY_ID, total quantity on hand, available-to-promise quantity, and calculates the difference between these two quantities as DIFFERENCE (i.e., the quantity on hand that is not available to promise).



### 3 Single-Return Orders (Last Month)

**Business Problem:**  
The mechandising team needs a list of orders that only have one return.

**Fields to Retrieve:**  
- `PARTY_ID`  
- `FIRST_NAME`


```
select
	person.PARTY_ID ,
	ri.ORDER_ID ,
	person.FIRST_NAME
from
	return_item ri
join return_header rh on
	rh.RETURN_ID = ri.RETURN_ID and 
    MONTH(rh.ENTRY_DATE ) = MONTH(CURRENT_DATE - INTERVAL 1 MONTH)
    AND YEAR(rh.ENTRY_DATE) = YEAR(CURRENT_DATE)
join person on
	rh.FROM_PARTY_ID = person.PARTY_ID
group by
	ri.ORDER_ID ,
	person.PARTY_ID
having
	COUNT(ri.RETURN_ID) = 1;
```
# Cost - 6,214.17
# Explanation - This SQL query retrieves information about returned items from the previous month by joining the return_item, return_header, and person tables. It selects the PARTY_ID, ORDER_ID, and FIRST_NAME of the person associated with the return. The query filters for returns that occurred in the previous month and groups the results by ORDER_ID and PARTY_ID. It only includes orders that have exactly one return, using the HAVING COUNT(ri.RETURN_ID) = 1 condition.

### 4 Returns and Appeasements 

**Business Problem:**  
The retailer needs the total amount of items, were returned as well as how many appeasements were issued.

**Fields to Retrieve:**  
- `TOTAL RETURNS`
- `RETURN $ TOTAL`
- `TOTAL APPEASEMENTS`
- `APPEASEMENTS $ TOTAL`

```
select
	count(ri.RETURN_ID),
	sum(ri.RETURN_PRICE * ri.RETURN_QUANTITY) as RETURN_TOTAL,
	count(ra.RETURN_ADJUSTMENT_ID) as TOTAL_APPEASEMENT,
	sum(ra.AMOUNT) as APPEASEMENT_AMOUNT_TOTAL
from
	return_header rh
join return_item ri on
	rh.RETURN_ID = ri.RETURN_ID
left join return_adjustment ra on
	ra.RETURN_ID = rh.RETURN_ID
	and ra.RETURN_ADJUSTMENT_TYPE_ID = 'APPEASEMENT';
```
# Cost - 4,221.67
# Explanation - This SQL query calculates key return details by joining the `return_header`, `return_item`, and `return_adjustment` tables. It counts the total number of returned items, calculates the total return value by multiplying the return price by the return quantity, and sums the results. Additionally, it counts the number of appeasements made for returns and calculates the total appeasement amount. The appeasements are identified by the condition `RETURN_ADJUSTMENT_TYPE_ID = 'APPEASEMENT'`, and all returns, including those without appeasements, are considered using a left join.
