# stock.quants 与 在手数量
stock.quants （库存包）对象是八版新功能的一个核心概念，可以理解为每次库存移动的一个最小单位。
举个例子，一次库存移动，从库存区移动7个产品A到出库区。那么这7个产品A就可以看做是一个库存包1（stock.quants)，它目前在发货区下。如果7个产品全部发给客户，库存包1的库位将会变成客户区。如果7个产品中
todo

有6个产品发货到客户，那么，库存包1将会被拆成2个库存包， 原库存包1只剩1个，还在发货区1，已发货的6个将会被

库存包（quants）对象具体是怎样参与到库存模块中的呢，引入它又有什么好处呢？
它将在分配库存，占用库存时生成（参考代码1），完成调拨时处理（参考代码2），取消调拨时取消（参考代码3）。

它完成处理的时间将会是调拨上的计划时间。

后面我会在具体介绍

主要影响了以下几个流程：

1. 在手数量，预测数量。
    在手数量，预测数量，在7版中基于已经完成的库存调拨计算。咋
fwef
    八版里，所在库位的在手库存为所在库位的所有库存包的数量之和

2. 分配库存


3. 占用库存

4. 归档功能讨论
7版中有了

5.



## 相关代码
1.
```
	def _get_domain_locations(self, cr, uid, ids, context=None):
'''
Parses the context and returns a list of location_ids based on it.
It will return all stock locations when no parameters are given
Possible parameters are shop, warehouse, location, force_company, compute_child
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

2.
```
fwef
fwefw
```
