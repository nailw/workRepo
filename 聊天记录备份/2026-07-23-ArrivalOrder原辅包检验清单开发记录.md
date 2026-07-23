# ArrivalOrder 原辅包检验清单开发记录

- 整理日期：2026-07-23
- 项目节点：`ArrivalOrder`
- 目标节点：`InspectList`
- 需求范围：根据已到货数据生成原辅包检验清单，不处理采购订单在途检验清单，也不调整后续核料逻辑。

## 一、最终需求

将 `ArrivalOrder` 的到货记录直接追加到现有 `InspectList`，使 `InspectList` 同时包含：

1. `InspectDemand` 生成的半成品检验清单；
2. `ArrivalOrder` 生成的原辅包检验清单。

同一到货单的不同明细、批次分别生成检验清单，不合并数量。到货单记录使用 `ArrivalOrder.joint` 作为来源唯一键，并增加 `ARRIVAL_ORDER` 前缀，防止与半成品检验清单联合键冲突。

## 二、关键业务结论

`ArrivalOrder` 中两个到货日期字段含义不同：

- `arrivalDateInit`：初始到货日期，作为最早检验开始时间；
- `arrivalDate`：已经加上质检周期的到货日期，直接作为最晚检验完成时间。

因此不能再执行 `arrivalDate.plus(inspectionDuration)`，否则会重复增加一次质检周期。

最终时间关系：

```text
最早检验开始时间 = ArrivalOrder.arrivalDateInit
检验周期 = Duration.ofDays(MaterialDictionary.inspectDays)
最晚检验完成时间 = ArrivalOrder.arrivalDate
```

`InspectList` 新增字段：

| 字段 | 类型 | 描述 |
|---|---|---|
| `inspectEarliestEndTime` | `DateTime` | 最晚检验完成时间 |

说明：字段名称包含 `Earliest`，但当前业务描述和赋值口径为“最晚检验完成时间”。

## 三、字段映射

| InspectList字段 | 半成品来源 | 到货单来源 |
|---|---|---|
| `categoryName` | `InspectDemand.categoryName` | `MaterialDictionary.categoryName` |
| `jointId` | 生产单、物料、车间、工艺、工序联合键 | `ARRIVAL_ORDER + ArrivalOrder.joint` |
| `pieceNr` | `InspectDemand.pieceNr` | `ArrivalOrder.pieceNr` |
| `productId` | `InspectDemand.productId` | `ArrivalOrder.materialId` |
| `productName` | `InspectDemand.productName` | `ArrivalOrder.materialName` |
| `productionNr` | 生产通知单号 | 暂存 `ArrivalOrder.arrivalOrderNr` |
| `quantity` | `InspectDemand.quantity` | `ArrivalOrder.arrivalQuantity` |
| `specification` | `InspectDemand.specification` | `ArrivalOrder.specificationGrade` |
| `inspectEarliestStartTime` | 当前工序生产结束时间 | `ArrivalOrder.arrivalDateInit` |
| `inspectionDuration` | 当前工序检验周期 | `Duration.ofDays(MaterialDictionary.inspectDays)` |
| `inspectEarliestEndTime` | 开始时间＋检验周期 | `ArrivalOrder.arrivalDate` |
| `dueDate` | 下一工序开始日期 | `ArrivalOrder.arrivalDate.toDate()` |

`ArrivalOrder` 已有关联属性 `materialDictionary`，关联至 `MaterialDictionary`；物料主数据另有索引 `indexByPlantIdAndMaterialId(plantId, materialId)`。

## 四、生成规则

到货检验清单仅处理以下记录：

```text
joint不为空
materialId不为空
arrivalQuantity大于0
arrivalDateInit有效
arrivalDate有效
materialDictionary关联不为空
```

当前不根据 `arrivalClosed` 或 `inboundClosed` 过滤。只要存在有效到货数量及日期，即生成检验清单。

当前程序采用 `Group<InspectList>().clear()` 后全量重建，因此不再执行清空后的旧数据索引查询。

## 五、最终 InspectDemand 生成代码

```java
Group<InspectDemand>().clear()
var addAll = new List<InspectDemand>()

context().Group<PieceStep>().sortedBy{ it.owner.productionNr }.forEach{
    var productionNr = it.owner.productionNr
    var productId = it.productId
    var productRouting = indexing(ProductRouting.OnlyIndex,it.productId,it.workshopId).firstOrNull()
    var routingId = ""
    var routingName = ""
    if(productRouting != null){
        routingId = productRouting.routingId
        routingName = productRouting.routingName
    }
    var productRoutingStep = it.productRoutingStep
    var operationTypeId = productRoutingStep?.operationTypeId
    var operationTypeName = productRoutingStep?.operationTypeName
    var workShopId = productRoutingStep?.workShopId
    var jointId = CommonUtils.joint(List.of(productionNr,productId,workShopId,routingId,operationTypeId))
    var obj = new InspectDemand(this.owner)
    obj.categoryName = it.owner.owner.categoryName
    obj.jointId = jointId
    obj.operationTypeId = operationTypeId
    obj.operationTypeName = operationTypeName
    obj.pieceNr = it.owner.pieceNr
    obj.productId = productId
    obj.productName = it.productName
    obj.productionNr = productionNr
    obj.routingId = routingId
    obj.routingName = routingName
    obj.workShopId = workShopId
    obj.quantity = it.quantity
    obj.specification = it.product.specification ?? ""
    obj.dueDate = Date.MIN
    obj.inspectEarliestStartTime = it.assignment?.processEndTime ?? DateTime.MIN
    obj.inspectionDuration = it.algorithmOfInspectionTime ?? Duration.ZERO
    var nextPieceStep = it.nextStep
    if(nextPieceStep == null){
        nextPieceStep = indexing(PieceStep.indexByPreviousStep,it).firstOrNull()
    }
    if(nextPieceStep != null){
        obj.dueDate = nextPieceStep.startTime?.toDate() ?? Date.MIN
    }
    obj.isDirty = false
    addAll.add(obj)
}

Group<InspectDemand>().addAll(addAll)
```

## 六、最终 InspectList 生成代码

```java
Group<InspectList>().clear()
var addAll = new List<InspectList>()

context().Group<InspectDemand>().filter{ it.isShow }.sortedBy{ it.productionNr }.forEach{
    var obj = new InspectList(this.owner)
    obj.categoryName = it.categoryName
    obj.jointId = CommonUtils.joint(List.of(it.productionNr,it.productId,it.workShopId,it.routingId,it.operationTypeId))
    obj.operationTypeId = it.operationTypeId
    obj.operationTypeName = it.operationTypeName
    obj.pieceNr = it.pieceNr
    obj.productId = it.productId
    obj.productName = it.productName
    obj.workShopId = it.workShopId
    obj.productionNr = it.productionNr
    obj.routingId = it.routingId
    obj.routingName = it.routingName
    obj.quantity = it.quantity
    obj.specification = it.specification ?? ""
    obj.dueDate = it.dueDate
    obj.inspectEarliestStartTime = it.inspectEarliestStartTime
    obj.inspectionDuration = it.inspectionDuration
    obj.inspectEarliestEndTime = it.inspectEarliestStartTime.plus(it.inspectionDuration)
    obj.isDirty = false
    addAll.add(obj)
}

context().Group<ArrivalOrder>().filter{
    it.joint != "" && it.materialId != "" && it.arrivalQuantity > 0.0 &&
    it.arrivalDateInit != DateTime.MIN && it.arrivalDate != DateTime.MIN &&
    it.materialDictionary != null
}.sortedBy{ it.arrivalOrderNr }.forEach{
    var material = it.materialDictionary
    var obj = new InspectList(this.owner)
    obj.categoryName = material.categoryName
    obj.jointId = CommonUtils.joint(List.of("ARRIVAL_ORDER",it.joint))
    obj.pieceNr = it.pieceNr
    obj.productId = it.materialId
    obj.productName = it.materialName
    obj.productionNr = it.arrivalOrderNr
    obj.quantity = it.arrivalQuantity
    obj.specification = it.specificationGrade ?? ""
    obj.inspectEarliestStartTime = it.arrivalDateInit
    obj.inspectionDuration = Duration.ofDays(material.inspectDays)
    obj.inspectEarliestEndTime = it.arrivalDate
    obj.dueDate = it.arrivalDate.toDate()
    obj.isDirty = false
    addAll.add(obj)
}

Group<InspectList>().addAll(addAll)
```

## 七、开发过程中的修正

1. 初版曾将 `arrivalDate` 当作原始到货时间，再加一次检验周期；结合字段说明及数据确认后发现 `arrivalDate` 已包含质检周期，因此该方案会重复计算。
2. 正确方案改为以 `arrivalDateInit` 作为检验开始时间，以 `arrivalDate` 作为检验完成时间。
3. 原辅包检验清单最终不新建独立节点，统一写入已有的 `InspectList`。
4. 后续核料读取物料到货或供应数据的代码不在本次开发范围内。
