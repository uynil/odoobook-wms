# 库存包（stock.quants）与在手数量
stock.quants （库存包）对象是八版新功能的一个核心概念，可以理解为每次库存移动的一个最小单位。

举个例子:
* 一次库存移动，从库存区移动7个产品A到出库区。那么这7个产品A就可以看做是一个库存包1（stock.quants)，它目前在发货区下。

* 如果7个产品全部发给客户，库存包1的库位将会变成客户区。如果7个产品
6个产品发货到客户，那么，库存包1将会被拆成2个库存包， 原库存包1只剩1个，还在发货区1，已发货的6个将会被

概念上，库存调拨是一个库存区到另一个库存区的移动，库存包则是库存产品在一个库位上的存储单位。

库存包（quants）对象具体是怎样参与到库存模块中的呢，引入它又有什么好处呢？

## 在手数量
* 七版中，
	+ 在手数量 = 查询时间为止的查询库位调拨数量之和
	+ 预测数量 = 在手数量 + 已经确定的入库调拨 - 已经确定的出库调拨
* 八版中，
	+ 在手数量 = 查询时间为止的查询库位库存包库存量之和之和
	+ 预测数量 = 在手数量 + 已经确定的入库调拨 - 已经确定的出库调拨 

### 三个因素
在手数量跟以下三个因素相关：

* 查询时间
* 库存区
* 库存产品

### 类似案例
* 产品界面当前在手数量
在手数量 = 查询时间之前的所有库存包库存数量之和， 产品的在手数量 没有指定库存区，缺省值为所有仓库的库存库位。

* 库存库位区间在手数量
在手数量 = 查询区间之内的所选库位的库存包库存数量之和，产品没有指定，为所有产品。


注： 八版方法可参考代码1.

##相关流程
库存调拨，odoo操作会有四个步骤， 1.新建。 2.确定。 3.检查可用（分配库存） 4.完成调拨。 5. （如果有）作废调拨。实际操作上，我们可以通过处理移库单，处理库存调拨；也可以直接处理库存调拨。

八版新增了库存包，操作对象依然在移库单和库存调拨, 处理库存调拨时，会自动生成，完结，作废库存包对象。

操作界面大幅改动，根据新增加的库存单分类。操作界面增加了手持设备界面支持。界面改动很大，也很直接。这里不详细介绍，用户可以自己体验。 

库存包对象，将在分配库存，分配库存调拨时生成（参考代码2），完成调拨调拨处理（参考代码3），作废调拨调拨作废（参考代码4）。

### 0. 新建、确认库存调拨
无相关库存包操作

### 1. 分配库存调拨
分配库存调拨上，如何产生库存调拨，代码中这样注释：
1. 先根据相关的操作（包装，批号）找到对应的库存包.
	这一步中会用到quants_get_prefered_domain方法，这一方法会应用removal strategy。
2. 然后数量不够的话，分配没有特殊限制的其他库存包.

### 2. 完成库存调拨
1. 先移动相关操作（包装，批号）的库存包
2. 再移动剩余数量的库存包

### 3. 作废库存调拨
见代码


## 遗留问题
### 自动生成无法手工更正的库存包
目前库存包仍是系统在分配库存时自动生成，不能手动修改。

### 归档功能讨论
当库存量很大的时候，比如说一天30000行，那么目前照7版方法，势必会影响系统。在这样的前提下，我们提出设想说，在手数量计算不必计算全部历史，秩序计算上次盘点，以及盘掉后累计出入货数量。
	1. Use standard Inventory Line 
	2. Customerize another warehouse solution.

## 相关代码
### 在手数量
```
	def _get_domain_locations(self, cr, uid, ids, context=None):
	''
	Prses the context and returns a list of location_ids based on it.
	I will return all stock locations when no parameters are given
	Pssible parameters are shop, warehouse, location, f rce_company, compute_child
	''
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

### 分配库存调拨


```
    def quants_get_prefered_domain(self, cr, uid, location, product, qty, domain=None, prefered_domain_list=[], restrict_lot_id=False, restrict_partner_id=False, context=None):
        ''' This function tries to find quants in the given location for the given domain, by trying to first limit
            the choice on the quants that match the first item of prefered_domain_list as well. But if the qty requested is not reached
            it tries to find the remaining quantity by looping on the prefered_domain_list (tries with the second item and so on).
            Make sure the quants aren't found twice => all the domains of prefered_domain_list should be orthogonal
        '''
        if domain is None:
            domain = []
        quants = [(None, qty)]
        #don't look for quants in location that are of type production, supplier or inventory.
        if location.usage in ['inventory', 'production', 'supplier']:
            return quants
        res_qty = qty
        if not prefered_domain_list:
            return self.quants_get(cr, uid, location, product, qty, domain=domain, restrict_lot_id=restrict_lot_id, restrict_partner_id=restrict_partner_id, context=context)
        for prefered_domain in prefered_domain_list:
            if res_qty > 0:
                #try to replace the last tuple (None, res_qty) with something that wasn't chosen at first because of the prefered order
                quants.pop()
                tmp_quants = self.quants_get(cr, uid, location, product, res_qty, domain=domain + prefered_domain, restrict_lot_id=restrict_lot_id, restrict_partner_id=restrict_partner_id, context=context)
                for quant in tmp_quants:
                    if quant[0]:
                        res_qty -= quant[1]
                quants += tmp_quants
        return quants
```

```
quant_create
def quants_reserve(self, cr, uid, quants, move, link=False, context=None):
        '''This function reserves quants for the given move (and optionally given link). If the total of quantity reserved is enough, the move's state
        is also set to 'assigned'

        :param quants: list of tuple(quant browse record or None, qty to reserve). If None is given as first tuple element, the item will be ignored. Negative quants should not be received as argument
        :param move: browse record
        :param link: browse record (stock.move.operation.link)
        '''
        toreserve = []
        reserved_availability = move.reserved_availability
        #split quants if needed
        for quant, qty in quants:
            if qty <= 0.0 or (quant and quant.qty <= 0.0):
                raise osv.except_osv(_('Error!'), _('You can not reserve a negative quantity or a negative quant.'))
            if not quant:
                continue
            self._quant_split(cr, uid, quant, qty, context=context)
            toreserve.append(quant.id)
            reserved_availability += quant.qty
        #reserve quants
        if toreserve:
            self.write(cr, SUPERUSER_ID, toreserve, {'reservation_id': move.id}, context=context)
            #if move has a picking_id, write on that picking that pack_operation might have changed and need to be recomputed
            if move.picking_id:
                self.pool.get('stock.picking').write(cr, uid, [move.picking_id.id], {'recompute_pack_op': True}, context=context)
        #check if move'state needs to be set as 'assigned'
        if reserved_availability == move.product_qty and move.state in ('confirmed', 'waiting'):
            self.pool.get('stock.move').write(cr, uid, [move.id], {'state': 'assigned'}, context=context)
        elif reserved_availability > 0 and not move.partially_available:
            self.pool.get('stock.move').write(cr, uid, [move.id], {'partially_available': True}, context=context)
stock.quant.package -- “physycal packkage"
stock.pack.operation -" Packing Operation"
    used by barcode interface    
```

```
stock.move
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
```

### 完成库存调拨

```
class stock_quant
    def quants_move(self, cr, uid, quants, move, location_to, location_from=False, lot_id=False, owner_id=False, src_package_id=False, dest_package_id=False, context=None):
        """Moves all given stock.quant in the given destination location.
        :param quants: list of tuple(browse record(stock.quant) or None, quantity to move)
        :param move: browse record (stock.move)
        :param location_to: browse record (stock.location) depicting where the quants have to be moved
        :param location_from: optional browse record (stock.location) explaining where the quant has to be taken (may differ from the move source location in case a removal strategy applied). This parameter is only used to pass to _quant_create if a negative quant must be created
        :param lot_id: ID of the lot that must be set on the quants to move
        :param owner_id: ID of the partner that must own the quants to move
        :param src_package_id: ID of the package that contains the quants to move
        :param dest_package_id: ID of the package that must be set on the moved quant
        """
        quants_reconcile = []
        to_move_quants = []
        self._check_location(cr, uid, location_to, context=context)
        for quant, qty in quants:
            if not quant:
                #If quant is None, we will create a quant to move (and potentially a negative counterpart too)
                quant = self._quant_create(cr, uid, qty, move, lot_id=lot_id, owner_id=owner_id, src_package_id=src_package_id, dest_package_id=dest_package_id, force_location_from=location_from, force_location_to=location_to, context=context)
            else:
                self._quant_split(cr, uid, quant, qty, context=context)
                quant.refresh()
                to_move_quants.append(quant)
            quants_reconcile.append(quant)
        if to_move_quants:
            to_recompute_move_ids = [x.reservation_id.id for x in to_move_quants if x.reservation_id and x.reservation_id.id != move.id]
            self.move_quants_write(cr, uid, to_move_quants, move, location_to, dest_package_id, context=context)
            self.pool.get('stock.move').recalculate_move_state(cr, uid, to_recompute_move_ids, context=context)
        if location_to.usage == 'internal':
            if self.search(cr, uid, [('product_id', '=', move.product_id.id), ('qty','<', 0)], limit=1, context=context):
                for quant in quants_reconcile:
                    quant.refresh()
                    self._quant_reconcile_negative(cr, uid, quant, move, context=context)
```

```
class stock_move:
 line 2397
    def action_done:
        """ Makes the move done and if all moves are done, it will finish the picking.
        @return:
        """
        picking_ids = []
        move_ids = []
        wf_service = netsvc.LocalService("workflow")
        if context is None:
            context = {}

        todo = []
        for move in self.browse(cr, uid, ids, context=context):
            if move.state=="draft":
                todo.append(move.id)
        if todo:
            self.action_confirm(cr, uid, todo, context=context)
            todo = []

        for move in self.browse(cr, uid, ids, context=context):
            if move.state in ['done','cancel']:
                continue
            move_ids.append(move.id)

            if move.picking_id:
                picking_ids.append(move.picking_id.id)
            if move.move_dest_id.id and (move.state != 'done'):
                # Downstream move should only be triggered if this move is the last pending upstream move
                other_upstream_move_ids = self.search(cr, uid, [('id','not in',move_ids),('state','not in',['done','cancel']),
                                            ('move_dest_id','=',move.move_dest_id.id)], context=context)
                if not other_upstream_move_ids:
                    self.write(cr, uid, [move.id], {'move_history_ids': [(4, move.move_dest_id.id)]})
                    if move.move_dest_id.state in ('waiting', 'confirmed'):
                        self.force_assign(cr, uid, [move.move_dest_id.id], context=context)
                        if move.move_dest_id.picking_id:
                            wf_service.trg_write(uid, 'stock.picking', move.move_dest_id.picking_id.id, cr)
                        if move.move_dest_id.auto_validate:
                            self.action_done(cr, uid, [move.move_dest_id.id], context=context)

            self._create_product_valuation_moves(cr, uid, move, context=context)
            if move.state not in ('confirmed','done','assigned'):
                todo.append(move.id)

        if todo:
            self.action_confirm(cr, uid, todo, context=context)

        self.write(cr, uid, move_ids, {'state': 'done', 'date': time.strftime(DEFAULT_SERVER_DATETIME_FORMAT)}, context=context)
        for id in move_ids:
             wf_service.trg_trigger(uid, 'stock.move', id, cr)

        for pick_id in picking_ids:
            wf_service.trg_write(uid, 'stock.picking', pick_id, cr)

        return True
```

### 取消库存调拨
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

### 参考 LPN 对象
库存包 对比 Oracle LPN 对象

Oracle LPN (license plate number)
http://docs.oracle.com/cd/E18727_01/doc.121/e13433/T211976T321834.htm
