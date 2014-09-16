# Stock Planning
This artile will explain how the pickings are prepared and created by the system, especially with new v8 objects.
To be metioned, of course, pickings can be created manually by users.


## New Objects: 
* Type of operation
	* Define default source location and destination location  
	* define sequence code
	* Define Returning Type (only type, return locaiton is not taken from returning )

* Routes
	* A set of procurment rules 
	* A set of push rules (but not used)

* Warehouse configuration
	* Warehouse configuration will create automaically several routes based on drop shipping, resupply, incoming policy, outgoing policy.

* Other new objects: quants, pickings.

## Pikcing generation
* sale order confirmation -> create procument -> create outgoing shipment
* purchase order confirmation -> create incoming shipment.
* push flow
* runing scheudler to calculate procument
	* -> create direct stock picking
	* -> create another procurement


Here we have another important odoo object `procurment`

## Push Rule
After confirming a sale order,
odoo will check all the related push flow and generate related picking if needed.

Eg.
Push Rule: Location A -> Location B, type: manual
When: after a picking to location A is confirmed
What: another move from locatiion A to location B will be created automatically.


## Procurment
procurment is used to generate
* Purchase order when procurment action is 'buy'
* Manufacturing order when action is 'mrp'
* stock picking when procurment action is 'move' (resupply)
* create another procurment if method is 'procurment'

### procurment creation
How is a procurment created:
* sale order confirmation
* procurment run
* scheduler running (check minimum stock rules `order point`)

No procurment, procurment rule will do nothing.

### procurment rule
procurment rule is the rule to define the action, method, location, seqence, type and other options to generate the 

procurment rule is assigned to a procurment by
* manually set up in procurment
* manually set up in sale order line
* picked by by system when assign procurement with function _find_procument_rule

### find procurment rule
#### 'move', stock
find procurment rule (1st procurment rule in 1st route), procurment rule of procurment route, product route, warehouse.

### run procurment rule

#### 'move' stock
	create move or create procument based on move supply method
#### 'purchase' purchase
	make purchase order
		do nothing if no supplier defined
		create new purchase order if no exsting PO with same location, partner , picking_type, etc.
		add new line to exiting purchase order if old order no product
		merge to old line to old PO if old order old line.
		(check make_po() of  purchase/purchase.py for more details)
#### 'mrp' purchase
	make mo
		do nothing if no bom defined
		create new mo if have bom in product
#### example
EG 1

Procurment Rule: Location A -> Location B, type: manual
When: after a picking to location A is confirmed
What: another move from locatiion A to location B will be created automatically.

### Run Scheduler
#### Stock
1. confirm and run procurment to create new purchase orders, manufacturig orders, stock pickings.
2. check minimum stock rule, create procurment

## Set Up Warehouse
In V8, we have a lot of new options, choose resupply, set up inocming and outgoing policy, 
all these actions will create new stock routes, and stock  routes will affect procurment rule of procurment.
(see article 'find procurment rule')


## Summary
Although, in v8, there are a lot new objects to set up warehouse, the objects to create pickings is still the old objects as v7: procurment, procument rules and push rules.


#code related
* purchase/ purchase.py
* stock/stock.py
* sale/stock.py
* mrp/stock.py
* procurment/procurment.py