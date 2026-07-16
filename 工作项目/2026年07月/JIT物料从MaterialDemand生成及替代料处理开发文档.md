# JIT物料从MaterialDemand生成及替代料处理开发文档

## 1. 改造背景

原实现从 `MaterialFulfill` 反推物料未满足量。当某个需求完全没有供应时，没有 `MaterialFulfill` 记录，缺料会被遗漏。因此改为直接读取最终核料后的 `MaterialDemand`。

`MaterialDemand` 的普通需求未满足量为：

```text
max(demandQuantity - fulfillQuantity, 0)
```

其中 `inventoryFulfillQuantity` 仅表示库存满足量，不能用于计算最终缺料，因为采购订单、采购计划、其他场景生产计划和本车间产出也可能满足需求。

## 2. 替代料处理规则

替代料的 `MaterialDemand` 分别记录在不同需求行中，不能直接按 `materialId` 分别计算后相加。

替代组按照 `pieceStepId + replaceGroupId` 作为一个逻辑需求组：

1. 找到 `isFirstReplace=true` 的第一顺位需求；
2. 使用第一顺位需求的需求数量作为替代组基准数量；
3. 使用第一顺位需求的 `leftRatio` 作为整个替代组剩余未满足比例；
4. 替代组最终缺料量为 `第一顺位需求数量 × leftRatio`；
5. 生成 `RawMaterialOnPiece` 时使用第一顺位主物料代码，而不是已经使用的替代供应物料代码。

例如主物料A需求100，A满足40，替代物料B满足30，第一顺位需求 `leftRatio=0.3`，最终缺料为30，而不是主物料单行计算出的60。

## 3. 执行时点

本方法必须在最终核料完成后执行：

```text
生成MaterialDemand
→ 删除旧分配
→ 冻结分配重占用
→ 普通物料和替代料重新分配
→ 更新MaterialDemand.fulfillQuantity和第一顺位leftRatio
→ 生成RawMaterialOnPiece
→ 汇总RawMaterialSummary
→ 同步DM
```

## 4. 参考实现代码

```java
context.Group<RawMaterialOnPiece>().clear()
context.Group<RawMaterialSummary>().clear()

var pieceGroup = context.Group<Piece>()
var orderGroup = context.Group<WorkOrder>()
var productGroup = context.Group<Product>()
var demands = MaterialDemand.queryAllData(context)

var rawMaterialOnPieces = new List<RawMaterialOnPiece>()
var rawMaterialSummarys = new List<RawMaterialSummary>()
var dataList = new List<Map<String, Any>>()

var orderIndex = orderGroup.filter{ it.workOrderNr != null && it.workOrderNr != "" }.groupBy{ it.workOrderNr }
var productIndex = productGroup.filter{ it.productId != null && it.productId != "" }.groupBy{ it.productId }
var demandIndex = demands.filter{
    it.pieceId != null &&
    it.pieceId != "" &&
    it.materialId != null &&
    it.materialId != ""
}.groupBy{ it.pieceId }

// 1. 根据最终MaterialDemand生成RawMaterialOnPiece
pieceGroup.forEach(piece -> {
    if (piece.pieceId == null || piece.pieceId == "" || piece.planStartTime == null || piece.planStartTime == DateTime.MAX) {
        return
    }
    if (piece.hasStarted == true || piece.hasFinished == true) {
        return
    }

    var order = orderIndex.get(piece.workOrderNr)?.firstOrNull()
    if (order == null) {
        return
    }

    var pieceDemands = demandIndex.get(piece.pieceId)
    if (pieceDemands == null || pieceDemands.isEmpty()) {
        return
    }

    // 普通需求按uniqueId，替代需求按工序和替代组形成逻辑需求组
    var logicalDemandGroup = pieceDemands.groupBy{
        var replaceGroupId = it.replaceGroupId ?? ""
        if (replaceGroupId != "") {
            return "REPLACE|" + (it.pieceStepId ?? "") + "|" + replaceGroupId
        }
        return "SINGLE|" + (it.uniqueId ?? "")
    }

    var materialNeedQuantityMap = new Map<String, Double>()
    var materialDemandMap = new Map<String, MaterialDemand>()

    logicalDemandGroup.forEach((logicalKey, demandRows) -> {
        var firstRow = demandRows.firstOrNull()
        if (firstRow == null) {
            return
        }

        var replaceGroupId = firstRow.replaceGroupId ?? ""
        var representativeDemand = firstRow
        var remainQuantity = 0.0

        if (replaceGroupId == "") {
            // 普通需求：需求总量减去当前需求行的满足量
            demandRows.forEach(demand -> {
                var demandQuantity = demand.demandQuantity ?? 0.0
                var fulfillQuantity = demand.fulfillQuantity ?? 0.0
                var rowRemainQuantity = maxOf(demandQuantity - fulfillQuantity, 0.0)
                if (rowRemainQuantity > Constant.EPS) {
                    remainQuantity += rowRemainQuantity
                }
            })
        } else {
            // 替代组：以第一顺位主物料为代表
            var firstReplaceDemand = demandRows.filter{ it.isFirstReplace == true }.firstOrNull()
            if (firstReplaceDemand == null) {
                firstReplaceDemand = demandRows.filter{ (it.demandQuantity ?? 0.0) > Constant.EPS }.firstOrNull()
            }
            if (firstReplaceDemand == null) {
                return
            }

            representativeDemand = firstReplaceDemand
            var demandQuantity = firstReplaceDemand.demandQuantity ?? 0.0

            // 与核料分配时保持一致，兼容需求总量未落库的情况
            if (demandQuantity <= Constant.EPS) {
                var scrapRate = firstReplaceDemand.scrapRate ?? 1.0
                if (scrapRate <= Constant.EPS) {
                    scrapRate = 1.0
                }
                demandQuantity = firstReplaceDemand.inputFactor * firstReplaceDemand.pieceStepQuantity / scrapRate
            }

            var leftRatio = firstReplaceDemand.leftRatio ?? 1.0
            leftRatio = maxOf(minOf(leftRatio, 1.0), 0.0)
            remainQuantity = demandQuantity * leftRatio
        }

        if (remainQuantity <= Constant.EPS) {
            return
        }

        var rawMaterialId = representativeDemand.materialId ?? ""
        if (rawMaterialId == "") {
            return
        }

        var currentQuantity = materialNeedQuantityMap.getOrDefault(rawMaterialId, 0.0)
        materialNeedQuantityMap.put(rawMaterialId, currentQuantity + remainQuantity)
        if (!materialDemandMap.containsKey(rawMaterialId)) {
            materialDemandMap.put(rawMaterialId, representativeDemand)
        }
    })

    // 2. 生成批次物料缺料明细
    materialNeedQuantityMap.forEach((rawMaterialId, needQuantity) -> {
        if (needQuantity <= Constant.EPS) {
            return
        }

        var representativeDemand = materialDemandMap.get(rawMaterialId)
        if (representativeDemand == null) {
            return
        }

        var rawMaterial = productIndex.get(rawMaterialId)?.firstOrNull()
        var onPiece = new RawMaterialOnPiece()
        onPiece.orderNr = order.workOrderNr
        onPiece.productId = piece.productId
        onPiece.productName = piece.productName
        onPiece.rawMaterialId = rawMaterialId
        onPiece.rawMaterialName = representativeDemand.materialName ?? rawMaterial?.productName ?? ""
        onPiece.plannedStartTime = piece.planStartTime
        onPiece.needQuantity = needQuantity.toString()
        onPiece.isJitMaterial = rawMaterial?.isJIT ?? false
        onPiece.pieceNr = piece.editPieceNr
        onPiece.plantId = order.plantId
        onPiece.workShopId = order.workShopId
        onPiece.orderDueDate = order.dueDate
        onPiece.givenBatch = order.givenBatch
        onPiece.piece = piece
        rawMaterialOnPieces.add(onPiece)
    })
})

// 3. 汇总JIT物料
var summaryGroup = rawMaterialOnPieces.filter{
    it.isJitMaterial == true &&
    it.plannedStartTime != null &&
    it.needQuantity != null &&
    it.needQuantity != "" &&
    it.needQuantity.toDouble() > Constant.EPS
}.groupBy{
    var order = orderIndex.get(it.orderNr)?.firstOrNull()
    var isInternationalOrder = order?.customType == "国际订单" || order?.customType == "国外订单"
    var hasGivenBatch = it.givenBatch != null && it.givenBatch != ""
    var is1003Material = it.rawMaterialId != null && it.rawMaterialId.startWith("1003")

    if (isInternationalOrder && hasGivenBatch && !is1003Material) {
        return "BATCH|" + it.plantId + "|" + it.workShopId + "|" + it.orderNr + "|" + it.productId + "|" + it.rawMaterialId + "|" + it.pieceNr + "|" + it.givenBatch
    }

    return "DAY|" + it.plantId + "|" + it.workShopId + "|" + it.productId + "|" + it.rawMaterialId + "|" + it.plannedStartTime.format("yyyy-MM-dd")
}

summaryGroup.forEach((key, rows) -> {
    var first = rows.firstOrNull()
    var earliestRow = rows.minByOrNull{ it.plannedStartTime }
    if (first == null || earliestRow == null) {
        return
    }

    var needQuantity = 0.0
    rows.forEach(row -> {
        if (row.needQuantity != null && row.needQuantity != "") {
            needQuantity += row.needQuantity.toDouble()
        }
    })
    if (needQuantity <= Constant.EPS) {
        return
    }

    var isBatchSummary = key.toString().startWith("BATCH|")
    var orderNr = ""
    var displayPieceNr = ""
    var givenBatch = ""

    if (isBatchSummary) {
        orderNr = first.orderNr ?? ""
        displayPieceNr = first.pieceNr ?? ""
        givenBatch = first.givenBatch ?? ""
    } else {
        var orderNrs = new List<String>()
        var pieceNrs = new List<String>()
        var givenBatches = new List<String>()

        rows.forEach(row -> {
            var rowOrderNr = row.orderNr ?? ""
            if (rowOrderNr != "" && !orderNrs.contain(rowOrderNr)) {
                orderNrs.add(rowOrderNr)
            }

            var rowPieceNr = row.pieceNr ?? ""
            if (rowPieceNr != "" && !pieceNrs.contain(rowPieceNr)) {
                pieceNrs.add(rowPieceNr)
            }

            var rowGivenBatch = row.givenBatch ?? ""
            if (rowGivenBatch != "" && !givenBatches.contain(rowGivenBatch)) {
                givenBatches.add(rowGivenBatch)
            }
        })

        orderNrs.forEach(value -> {
            if (orderNr == "") {
                orderNr = value
            } else {
                orderNr = orderNr + "," + value
            }
        })

        pieceNrs.forEach(value -> {
            if (displayPieceNr == "") {
                displayPieceNr = value
            } else {
                displayPieceNr = displayPieceNr + "," + value
            }
        })

        givenBatches.forEach(value -> {
            if (givenBatch == "") {
                givenBatch = value
            } else {
                givenBatch = givenBatch + "," + value
            }
        })
    }

    var summary = new RawMaterialSummary()
    summary.orderNr = orderNr
    summary.productId = first.productId
    summary.productName = first.productName
    summary.rawMaterialId = first.rawMaterialId
    summary.rawMaterialName = first.rawMaterialName
    summary.plannedStartTime = earliestRow.plannedStartTime
    summary.arrivalTime = TimeCalc.timeMinusDuration(earliestRow.plannedStartTime, Duration.ofDays(1))
    summary.needQuantity = needQuantity
    summary.uniqueId = key.toString() + "|" + displayPieceNr
    summary.displayPieceNr = displayPieceNr
    summary.givenBatch = givenBatch
    summary.plantId = first.plantId
    summary.workShopId = first.workShopId

    var earliestDueDateRow = rows.filter{ it.orderDueDate != null && it.orderDueDate != Date.MIN }.minByOrNull{ it.orderDueDate }
    summary.orderDueDate = earliestDueDateRow?.orderDueDate ?? Date.MIN
    rawMaterialSummarys.add(summary)
})

// 4. 组装发送至DM的数据
rawMaterialSummarys.forEach(record -> {
    var map = new Map<String, Any>()
    map["uniqueId"] = record.uniqueId
    map["orderNr"] = record.orderNr
    map["productId"] = record.productId
    map["productName"] = record.productName
    map["rawMaterialId"] = record.rawMaterialId
    map["rawMaterialName"] = record.rawMaterialName
    map["plannedStartTime"] = record.plannedStartTime
    map["arrivalTime"] = record.arrivalTime
    map["needQuantity"] = record.needQuantity
    map["displayPieceNr"] = record.displayPieceNr
    map["givenBatch"] = record.givenBatch
    map["plantId"] = record.plantId
    map["workShopId"] = record.workShopId
    map["orderDueDate"] = record.orderDueDate
    map["ppSceneId"] = context.sceneCode()
    map["ppProjectId"] = context.projectCode()
    dataList.add(map)
})

// 5. 写入节点并发送DM，空Data也发送以清理DM历史数据
context.Group<RawMaterialOnPiece>().addAll(rawMaterialOnPieces)
context.Group<RawMaterialSummary>().addAll(rawMaterialSummarys)

var args = new Map<String, Any>()
args["PPSceneId"] = context.sceneCode()
args["PPProjectId"] = context.projectCode()
args["Data"] = dataList
Utils.sendDataToDM(context, "DataReceiverManager", "syncMaterialSummary", args)
```

## 5. 验证重点

1. 普通需求：`needQuantity = demandQuantity - fulfillQuantity`。
2. 替代组：同一个 `pieceStepId + replaceGroupId` 只生成一条逻辑缺料，数量使用第一顺位 `leftRatio`。
3. 替代组输出第一顺位主物料代码，不输出已经用于满足的替代供应物料代码。
4. 没有任何供应时，MaterialDemand仍然存在，`leftRatio=1`，应生成完整需求数量。
5. `inventoryFulfillQuantity`不参与最终未满足量计算。
6. `RawMaterialSummary`只汇总 `isJitMaterial=true` 的批次物料。
7. 执行前确认最终核料已经更新 `fulfillQuantity` 和 `leftRatio`。

