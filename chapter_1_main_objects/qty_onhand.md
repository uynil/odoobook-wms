
首先看 wms的核心概念之一 库存数量

1. Qty Avaiable / 在手数量
2.  stock quant

1. Qty Virtual / 在手数量， 预测数量


Code


```
	def _get_domain_locations(self, cr, uid, ids, context=None):
'''
Parses the context and returns a list of location_ids based on it.
It will return all stock locations when no parameters are given
Possible parameters are shop, warehouse, location,  force_company, compute_child
'''
	context = context or {}

	location_obj = self.pool.get('stock.location')
	warehouse_obj = self.pool.get('stock.warehouse')

	location_ids = []
	if context.get('location', False):
	if type(context['location']) == type(1):
		location_ids = [context['location']]
	elif type(context['location']) in (type(''), type(u'')):
	domain = [('complete_name','ilike',context['location'])]
	if context.get('force_company', False):
		domain += [('company_id', '=', context['force_company'])]
		location_ids = location_obj.search(cr, uid, domain, context=context)
	else:
		location_ids = context['location']
	else:
		if context.get('warehouse', False):
			wids = [context['warehouse']]
		else:
			wids = warehouse_obj.search(cr, uid, [], context=context)

	for w in warehouse_obj.browse(cr, uid, wids, context=context):
		location_ids.append(w.view_location_id.id)

	operator = context.get('compute_child', True) and 'child_of' or 'in'
	domain = context.get('force_company', False) and ['&', ('company_id', '=', context['force_company'])] or []
	return (
		domain + [('location_id', operator, location_ids)],
		domain + ['&', ('location_dest_id', operator, location_ids), '!', ('location_id', operator, location_ids)],
		domain + ['&', ('location_id', operator, location_ids), '!', ('location_dest_id', operator, location_ids)]
		)
```
可以看到当没有给任何库位的时候，缺省库位取的是，可选公司里的所有仓库的库存库位 

2. 那么 stock quant 是怎么生成， 又怎样计算数量呢
class stock_quant:

    def quant_move:
    def quant_create:

class stock_move:
    def action_done:



    def action_reserve:
  for ops in operations:
#first try to find quants based on specific domains given by linked operations
for record in ops.linked_move_operation_ids:
move = record.move_id
if move.id in main_domain:
domain = main_domain[move.id] + self.pool.get('stock.move.operation.link').get_specific_domain(cr, uid, record, context=context)
qty = record.qty
if qty:
quants = quant_obj.quants_get_prefered_domain(cr, uid, ops.location_id, move.product_id, qty, domain=domain, prefered_domain_list=[], restrict_lot_id=move.restrict_lot_id.id, restrict_partner_id=move.restrict_partner_id.id, context=context)
quant_obj.quants_reserve(cr, uid, quants, move, record, context=context)
for move in todo_moves:
move.refresh()
#then if the move isn't totally assigned, try to find quants without any specific domain
if move.state != 'assigned':
qty_already_assigned = move.reserved_availability
qty = move.product_qty - qty_already_assigned
quants = quant_obj.quants_get_prefered_domain(cr, uid, move.location_id, move.product_id, qty, domain=main_domain[move.id], prefered_domain_list=[], restrict_lot_id=move.restrict_lot_id.id, restrict_partner_id=move.restrict_partner_id.id, context=context)
quant_obj.quants_reserve(cr, uid, quants, move, context=context)
<pre>
<code>
def action_cancel(self, cr, uid, ids, context=None):
""" Cancels the moves and if all moves are cancelled it cancels the picking.
@return: True
"""
procurement_obj = self.pool.get('procurement.order')
context = context or {}
procs_to_check = []
for move in self.browse(cr, uid, ids, context=context):
if move.state == 'done':
raise osv.except_osv(_('Operation Forbidden!'),
_('You cannot cancel a stock move that has been set to \'Done\'.'))
if move.reserved_quant_ids:
self.pool.get("stock.quant").quants_unreserve(cr, uid, move, context=context)
</code>
</pre>

....

也就是在所有会处理库存调拨的时候 我们会处理stock quant， 然后基于stock quant计算库存数量
那么在move quants的时候


quant_create
stock.quant.package -- “physycal packkage"

stock.pack.operation -" Packing Operation"
    used by barcode interface
    
    
遗留问题：
1. stock.quant不能修改，自动生成。就是说当自动生成的stock.quant 出现问题，用户不能手动修正错误。
