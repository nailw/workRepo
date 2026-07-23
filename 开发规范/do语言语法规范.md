# DO语言语法规范

- 文档性质：长期维护
- 首次整理日期：2026-07-15
- 适用范围：DO方法、模型方法及业务脚本
- 维护原则：用户指出语法错误、平台出现特殊语法、或发现与Java/Groovy/Kotlin等语言不同的行为时，更新本文件并记录变更。

## 1. 文档说明

本文根据goodo平台实际代码和方法编辑器整理，不把DO语言直接等同于Java、Groovy、Kotlin或C#。

文档中的规则分为三种状态：

| 状态 | 含义 |
|---|---|
| 已确认 | 已在用户提供的现有业务代码中多次出现，可以按现有写法使用 |
| 推定 | 根据现有代码可以合理推断，但还没有独立测试语义 |
| 待验证 | 需要通过平台编译、运行结果或平台文档进一步确认 |

编写新代码时，优先模仿同一平台、同一模块中已经成功运行的DO代码，不直接套用其他语言的写法。

## 2. 方法定义规则

### 2.1 方法签名由平台界面配置

状态：已确认。

DO方法编辑器右侧配置：

- 方法名称；
- 所属类；
- 返回值；
- 参数名称；
- 参数类型；
- 方法描述。

代码编辑区通常只填写方法体，不重复写完整方法声明。

例如平台界面显示：

```text
static void checkInputMaterialForClient(Context context)
```

编辑区只写：

```java
var materialManager = MaterialManager.getInstance(context)
materialManager.checkInputMaterial()
```

### 2.2 实例方法和静态方法

状态：已确认。

实例方法中使用：

```java
this.synMaterialSupply()
this.synMaterialDemand()
```

静态方法通常通过类名调用：

```java
MaterialDemand.assignSuppliesToFirstDemands(context, firstDemands, supplyNode, demandNode)
MaterialSupply.queryAvailableFromInputSupplies(supplyNode, demand, Constant.EPS, context)
```

是否为静态方法由平台方法配置决定，不在方法体中再次写`static`。

## 3. 变量与类型

### 3.1 var类型推断

状态：已确认。

使用`var`声明局部变量，由平台推断类型：

```java
var time1 = DateTime.now()
var pattern = this.owner.sceneParameter?.orderAndProcessPattern ?? ""
var toAdd = new List<MaterialSupply>()
```

### 3.2 显式类型

状态：已确认。

局部变量也可以显式声明类型：

```java
QMaterialSupply qSupply = QMaterialSupply.qmaterialsupply
```

### 3.3 泛型集合

状态：已确认。

```java
var supplies = new List<MaterialSupply>()
var values = new List<String>()
var data = new List<Map<String, Any>>()
var valueMap = new Map()
```

`Any`表示任意类型，常用于DM接口返回的动态字段值。

### 3.4 默认值

状态：已确认。

根据现有代码，模型的字符串字段通常使用空字符串而不是`null`表示未配置：

```java
if (productId == "") {
    return
}
```

但外部数据、关联对象和日期仍可能为`null`，不能假设所有字段都有默认值。

## 4. 空值处理

### 4.1 安全访问运算符

状态：已确认。

使用`?.`安全访问可能为空的对象：

```java
var detectDuration = product?.detectDuration
var workShopId = pieceStep.resourceGroups.firstOrNull()?.workShopId
```

当左侧对象为空时，不继续访问后面的属性。

### 4.2 空值默认运算符

状态：已确认。

使用`??`提供默认值：

```java
var detectDuration = product?.detectDuration ?? Duration.ZERO
var pattern = this.owner.sceneParameter?.orderAndProcessPattern ?? ""
var value = routing?.pieceSizeQuantity1 ?? ""
```

DO语言这里使用`??`，不能直接套用Java的`Optional`或Groovy的`?:`。

### 4.3 空对象和空集合判断

状态：已确认。

```java
if (obj == null) {
    return
}

if (list.isEmpty()) {
    return
}

if (list.isNotEmpty()) {
    // 继续处理
}
```

现有代码中也出现字符串的：

```java
if (!value.isEmpty()) {
    // 继续处理
}
```

## 5. 条件与运算符

### 5.1 条件语句

状态：已确认。

```java
if (!this.enabledCheckInputMaterial()) {
    return
}
```

```java
if (pattern == ModelConstants.Pattern23) {
    this.synInputAndOutOnStepByOperationBOM()
} else {
    this.synInputAndOutOnStepByBOM()
}
```

### 5.2 逻辑运算符

状态：已确认。

```java
conditionA && conditionB
conditionA || conditionB
!condition
```

### 5.3 三元表达式

状态：已确认。

```java
var totalRemainRatio = hasUserFulfillRatio ? userFulfillRatio : firstDemand.leftRatio
var ratio = demandQty > 0.0 ? fulfill.quantity / demandQty : 0.0
```

### 5.4 数字比较

状态：已确认。

浮点数量通常不直接和零比较，而是使用平台误差：

```java
if (quantity <= Constant.EPS) {
    return
}
```

### 5.5 C风格for循环

状态：已确认。

DO支持带显式`Int`类型、判断条件和累加表达式的C风格循环：

```java
var numberList = new List<String>()
for (Int i = 1; i < 13; i += 1) {
    numberList.add(i.toString())
}
```

当前代码已确认支持`i += 1`。新增循环时仍应保持循环上限清晰，避免在DO方法中产生不可控的大循环。

## 6. 集合闭包语法

### 6.1 单参数闭包

状态：已确认。

```java
orders.forEach{ order ->
    // 使用order
}
```

### 6.2 it隐式参数

状态：已确认。

闭包只有一个参数时可以使用`it`：

```java
var effectiveBoms = boms.filter{ it.isEffective }
var materialIds = demands.map{ it.materialId }
```

### 6.3 双参数Map遍历

状态：已确认。

```java
data.groupBy{ it.productId }.forEach((productId, values) -> {
    // 使用productId和values
})
```

### 6.4 闭包中的return

状态：推定，待专项验证。

现有代码大量使用：

```java
list.forEach{ item ->
    if (item == null) {
        return
    }
    // 处理当前item
}
```

从业务写法推断，闭包内的`return`用于结束当前闭包调用，效果接近循环中的`continue`。但它是否在所有闭包中都不会退出外层方法，仍应通过最小测试确认。

在确认前，新代码优先沿用现有同类方法的写法，不自行改成其他语言的`continue`语法。

## 7. 常用集合方法

状态：已确认。

### 7.1 遍历和筛选

```java
list.forEach{ it.isDirty = true }
var valid = list.filter{ it.quantity > Constant.EPS }
list.removeIf{ it.isDirty }
```

### 7.2 映射和去重

```java
var ids = list.map{ it.productId }
var unique = list.distinctBy{ it.pieceStepId + "|" + it.consumeSet }
```

### 7.3 查找

```java
var first = list.firstOrNull()
var last = list.lastOrNull()
var found = list.find{ it.productId == productId }
```

### 7.4 排序

```java
var sorted = list.sortedBy{ it.replacePriority }
var lastStep = steps.sortedBy{ it.sequenceNr }.lastOrNull()
```

### 7.5 分组

```java
var supplyMap = supplies.groupBy{ it.productId }
var productSupplies = supplyMap.getOrDefault(productId, new List<MaterialSupply>())
```

### 7.6 集合修改

```java
toAdd.add(obj)
toAdd.addAll(values)
list.remove(item)
list.removeIf{ condition }
```

### 7.7 contain

状态：已确认。

现有DO代码使用`contain`，不是Java集合的`contains`：

```java
if (productIds.contain(productId)) {
    // 命中
}
```

新增DO代码应优先使用平台现有写法`contain`。

## 8. Map语法

状态：已确认。

```java
var valueMap = new Map()
valueMap.putIfAbsent(key, value)
var value = valueMap.getOrDefault(key, defaultValue)
var values = valueMap.values()
```

通过`groupBy`得到的结果可以按Map使用。

## 9. Tuple元组

状态：已确认。

DO方法可以返回元组：

```java
return (totalFulfillRatio, totalToAddFulfill)
```

调用方通过下标读取：

```java
var tuple = MaterialDemand.assignSuppliesToFirstReplaceDemand(...)
var ratio = tuple[0]
var fulfills = tuple[1]
```

下标从`0`开始。

## 10. 场景对象Group

### 10.1 获取当前场景对象

状态：已确认。

```java
Group<WorkOrder>()
Group<Piece>()
Group<PieceStep>()
Group<InputOnStep>()
```

在持有Context参数的静态方法中，也出现：

```java
context.Group<PieceStep>()
```

具体使用`Group<T>()`还是`context.Group<T>()`，应参考当前方法所在类和已有上下文变量。

### 10.2 Group增删

状态：已确认。

```java
Group<InputOnStep>().removeIf{ it.isDirty }
Group<InputOnStep>().addAll(toAddInput)
```

## 11. indexing索引查询

状态：已确认。

平台通过预定义索引查询场景对象：

```java
var inputs = indexing(InputOnStep.indexByOwnerAndMaterialIdAndLineNr, workOrderStep, materialId, lineNr)
var input = inputs.firstOrNull()
```

索引参数顺序必须与索引定义一致，不能因为字段名称相同而随意调整。

新创建但尚未加入`Group`的对象，通常不能被`indexing`查到。因此现有代码会同时查询临时待新增集合：

```java
if (input == null) {
    input = toAddInput.find{ it.owner == workOrderStep && it.materialId == materialId && it.lineNr == lineNr }
}
```

## 12. KT基础数据查询

### 12.1 通过Context查询

状态：已确认。

```java
var routings = context.KT<KTProductRouting>().filter{ it.productId == productId }
```

如果当前变量实际命名为`conetxt`，代码必须使用现有变量名：

```java
conetxt.KT<KTProductRouting>()
```

不要在修改局部代码时只改引用而不改方法参数，否则会出现变量不存在。

### 12.2 select表达式

状态：已确认。

现有代码也使用：

```java
var rows = KT<KTInputConstraint>().select(it.productId == productId, it.stepSequenceNr != "")
```

`filter`和`select`是否具有完全相同的执行位置和性能特征，当前未确认。新增代码优先参考相邻业务代码的写法。

### 12.3 字段名大小写

状态：已确认。

模型字段名称区分大小写，必须与模型定义一致：

```java
routing.NoLocationId
inventory.sapStockingPointId
```

不能因为Java通常使用小驼峰，就自行把`NoLocationId`改成`noLocationId`。

字段的历史拼写也必须完全照模型使用，即使英文单词看起来拼写不规范。例如MaterialSupply模型的真实字段为：

```java
leftQuantityVritual
```

这里模型使用的是`Vritual`，业务代码必须同样写`leftQuantityVritual`，不能根据英文习惯自行改成`leftQuantityVirtual`。界面表头、截图显示或人的口头描述不能替代模型属性面板中的真实字段名。

## 13. 数据库访问

### 13.1 DbHelper

状态：已确认。

```java
var dh = context.getDbHelper()
```

在实例方法中可能使用：

```java
var dh = context().getDbHelper()
```

### 13.2 Q查询模型

状态：已确认。

```java
var qSupply = QMaterialSupply.qmaterialsupply
var data = dh.dslSelectFrom(qSupply).findAll()
```

### 13.3 批量写入

状态：已确认。

```java
dh.insertBatch(toAdd)
dh.updateBatch(toUpdate)
dh.deleteBatch(toDelete)
```

### 13.4 条件表达式

状态：已确认。

```java
dh.dslSelectFrom(qFulfill).where(qFulfill.frozenFlag.eq(true)).findAll()
```

查询条件中使用的`&&`、`||`可能由平台转换为数据库表达式，不应随意替换成Java QueryDSL的`.and()`或`.or()`。

## 14. DM接口动态数据

状态：已确认。

DM返回值通常转换为：

```java
var data = (List<Map<String, Any>>) Utils.getDataFromDM(context(), "DataSenderManager", "接口名称", null)
```

读取动态字段时使用：

```java
var productId = Utils.anyToString(row["productId"])
var quantity = Utils.anyToDouble(row["quantity"])
```

不要直接假设Map中的值已经是目标类型。

## 15. 日期、时间和时长

状态：已确认。

```java
var now = DateTime.now()
var zeroDuration = Duration.ZERO
var minTime = DateTime.MIN
var maxDate = Constant.DATE_MAX
```

日期时间加时长：

```java
obj.availableReleaseTime = TimeCalc.timePlusDuration(obj.availableTime, detectDuration)
obj.availableReleaseDate = obj.availableReleaseTime.toDate()
```

现有代码还确认了以下日期方法：

```java
date.month()
date.year()
date.dayOfMonth()
date.minusMonths(monthCount)
date.minusDays(dayCount)
dateTime.toDate()
```

例如计算交期向前N个月所在月份的第一天：

```java
var day = workOrder.dueDate.dayOfMonth()
var firstDay = workOrder.dueDate.minusMonths(monthCount).minusDays(day - 1)
```

日期和DateTime不能在不了解类型的情况下直接混用。

## 16. 字符串与转换工具

状态：已确认。

```java
Utils.anyToString(value)
Utils.anyToDouble(value)
Utils.splitStringToStringList(value)
value.toDouble()
count.toString()
```

多个业务代码通常使用平台工具拆分：

```java
var supplierIds = Utils.splitStringToStringList(materialDemand.supplierIds)
```

新增多值字段优先复用该工具，不自行假设分隔符解析规则。

## 17. 对象创建和临时集合

状态：已确认。

```java
var obj = new MaterialSupply()
var input = new InputOnStep(workOrderStep)
var output = new OutputOnStep(lastWorkOrderStep)
```

常见批量处理模式：

```java
var toAdd = new List<MaterialSupply>()
data.forEach{ row ->
    var obj = new MaterialSupply()
    // 赋值
    toAdd.add(obj)
}
context.getDbHelper().insertBatch(toAdd)
```

## 18. 格式与代码风格

### 18.1 不强制分号

状态：已确认。

现有DO代码每行末尾通常不写分号：

```java
var productId = demand.productId
var quantity = demand.demandQuantity
```

### 18.2 控制无意义换行

状态：用户明确要求。

简单调用和短条件尽量保持紧凑：

```java
var outputSupplies = supplyMap.getOrDefault(materialDemand.materialId, new List<MaterialSupply>())
```

长条件可以换行，但应围绕完整逻辑块换行，避免一个参数占一行导致DO编辑器代码过长。

### 18.3 保持相邻代码风格

状态：已确认。

修改已有方法时，变量名、Context获取方式、闭包格式和集合工具应尽量与相邻代码一致，避免在同一方法混用不同风格。

### 18.4 日志调试

状态：用户明确确认。

DO平台存在日志调用的作用域差异，不能假设所有方法都统一使用`log.info(String)`：

- 在`Utils`等部分方法中，现有代码确认可使用`log.info("...")`；
- 在`MaterialSupply`静态模型方法中，平台编译器提示`Cannot find name 'info'`和`Unexpected Argument: String`，此处应使用平台提供的`log("...")`；
- 编写日志时先参考当前所属类的已有成功代码，不能只参考其他类的日志写法。

```java
// MaterialSupply静态模型方法中的已确认写法
log("核料供应过滤：productId="+materialDemand.productId+",materialId="+materialDemand.materialId)

// Utils等已存在日志对象的上下文
log.info("运行结果："+result.toString())
```

排查集合过滤、分支判断和数量变化时，应优先一次性打印关键链路，减少逐点断点调试。日志内容应带产品、物料、需求或批次标识；高频循环中只对目标数据打印，问题确认后删除临时日志，避免产生大量日志和影响性能。

## 19. 与常见语言不同的重点

| DO写法 | 不应直接套用的其他语言习惯 |
|---|---|
| `??` | 不直接改成Groovy的`?:`或Java Optional |
| `?.` | Java本身没有该运算符 |
| `contain` | 不直接改成Java的`contains` |
| `new List<T>()` | 不按标准Java改成`new ArrayList<>()` |
| `Group<T>()` | 不是普通Java集合构造 |
| `context.KT<T>()` | 不是标准Java泛型方法调用场景 |
| `(value1, value2)` | 用于DO元组返回，不是Java返回语法 |
| `tuple[0]` | 用下标读取元组 |
| `{ it.field }` | DO闭包写法，不是标准Java Lambda |
| 方法体不写签名 | 签名由平台界面配置 |

## 20. 编写DO代码的检查清单

提交代码前检查：

1. 方法名、返回类型和参数是否已在界面正确配置；
2. 方法体中是否误写了完整方法签名；
3. Context变量实际叫`context`、`conetxt`还是通过`context()`获取；
4. 模型字段大小写是否完全一致；
5. 集合使用的是`contain`还是误写成`contains`；
6. 新对象是否需要加入临时`toAdd`集合和最终`Group`/数据库；
7. 新建但未加入Group的对象是否错误地只通过`indexing`查找；
8. 可空对象是否使用`?.`和`??`；
9. 浮点数量是否使用`Constant.EPS`；
10. 闭包内`return`的作用范围是否符合现有同类代码；
11. KT查询是否放在高频循环中造成性能问题；
12. 批量更新前是否存在重复对象或已删除对象；
13. 代码换行是否过多，是否便于在DO编辑器中复制和维护。

## 21. 待验证规则

| 编号 | 规则 | 当前判断 | 验证方式 |
|---:|---|---|---|
| 1 | `forEach`闭包中的`return`是否只结束当前元素 | 推定是 | 编写3元素最小测试并观察后续元素是否执行 |
| 2 | `roundHalfDown(7)`是否原地修改数值 | 未确认 | 比较调用前后变量，不赋值和赋值分别测试 |
| 3 | `KT.filter`与`KT.select`执行位置和性能是否相同 | 未确认 | 查看平台文档或运行性能测试 |
| 4 | 多行短代码能否使用分号合并 | 不建议假设 | 优先保持一条语句一行，不依赖分号 |
| 5 | `context()`与`context`的适用范围 | 取决于方法类型 | 对照实例方法和静态方法配置验证 |

## 22. 纠错与变更记录

| 日期 | 来源 | 修正内容 | 影响规则 |
|---|---|---|---|
| 2026-07-15 | 初始整理 | 根据现有核料代码建立第一版DO语言语法规范 | 全文 |
| 2026-07-15 | 用户要求 | 代码示例减少不必要换行，保持DO编辑器中易复制 | 18.2 |
| 2026-07-15 | 用户提供模型字段截图 | 确认MaterialSupply真实字段为`leftQuantityVritual`；历史字段拼写必须按模型原样使用，不能自行纠正为`Virtual` | 12.3 |
| 2026-07-15 | 用户提供生产月份过滤代码 | 确认DO支持`for(Int i=...;...;i+=1)`循环，以及`month/year/dayOfMonth/minusMonths/minusDays`日期运算 | 5.5、15 |

| 2026-07-15 | 用户纠正调试方式 | 复杂过滤链应优先一次性打印关键数量和条件，减少逐点断点排查 | 18.4 |
| 2026-07-15 | 平台编译结果纠错 | 日志调用具有作用域差异：`Utils`等上下文可用`log.info(String)`，当前`MaterialSupply`静态模型方法应使用`log(String)`；不能把一种写法推广到所有DO方法 | 18.4 |

后续每次发生以下情况时，在本表追加一条记录：

- 用户明确指出生成的DO代码语法不正确；
- 平台编译结果证明某项推定错误；
- 新代码显示DO语言存在尚未记录的特殊语法；
- 原有写法虽然类似Java/Groovy/Kotlin，但DO实际语义不同；
- 平台升级导致语法或API发生变化。


## 23. 24039/24040开发补充规范

本节记录2026-07-17在24039、24040核料规则开发中进一步确认的语法和代码约束。

### 23.1 sortedBy是当前确认的排序方法

状态：已确认。

~~~java
var result = context.KT<KTProcessingTradeHandBook>().filter{
    (it.semiFinishedId ?? "") == targetSemiFinishedId &&
    (it.manualNumber ?? "") != ""
}.sortedBy{ it.createDate }
~~~

当前DO代码使用 sortedBy。新增排序代码优先沿用该写法。

### 23.2 普通if允许多个条件

状态：已确认。

普通 if 可以写多个逻辑条件：

~~~java
if (!hasAvailableSupply &&
    MaterialSupply.isInventorySupply(supply) &&
    (supply.productId ?? "") == targetMaterialId &&
    (supply.sapStockingPointId ?? "") == targetLocationId &&
    supply.leftQuantity > Constant.EPS) {
    hasAvailableSupply = true
}
~~~

此前把“普通if允许多个条件”和“firstOrNull闭包不适合复杂多步逻辑”混为一谈，现已纠正。

### 23.3 firstOrNull闭包不写多步复杂逻辑

状态：已确认的代码约束。

以下写法不作为24039/24040的实现方式：

~~~java
var selectedHandBook = handBooks.firstOrNull{ handBook ->
    var targetMaterialId = handBook.materialId ?? ""
    var targetLocationId = handBook.warehouseLocation ?? ""
    ...
}
~~~

原因是当前DO编辑器中的 firstOrNull 闭包适合返回简单判断表达式，不适合在闭包内声明多个临时变量、遍历另一个集合并修改状态。

推荐使用：

~~~java
var hasSelectedHandBook = false
var selectedMaterialId = ""
var selectedLocationId = ""

handBooks.forEach{ handBook ->
    if (!hasSelectedHandBook) {
        // 计算当前手册是否有可用供应
        if (hasAvailableSupply) {
            selectedMaterialId = targetMaterialId
            selectedLocationId = targetLocationId
            hasSelectedHandBook = true
        }
    }
}
~~~

### 23.4 不使用复杂的filter后否定判断

状态：已确认的代码约束。

不使用：

~~~java
if (!supplies.filter{ ... }.isEmpty()) {
}
~~~

改用显式标记变量：

~~~java
var hasAvailableSupply = false

supplies.forEach{ supply ->
    if (condition) {
        hasAvailableSupply = true
    }
}

if (hasAvailableSupply) {
}
~~~

### 23.5 KT对象不作为空占位对象

状态：已确认的实现约束。

不建议：

~~~java
var selectedHandBook = new KTProcessingTradeHandBook()
~~~

这里的 new 表示创建一个空的KT对象，不是查询数据。由于24039/24040只需要保存选中的物料代码和库位，使用字符串和布尔变量更安全：

~~~java
var hasSelectedHandBook = false
var selectedMaterialId = ""
var selectedLocationId = ""
~~~

### 23.6 if的大括号和代码格式

状态：用户明确要求。

所有 if 都必须有大括号，包括只有一行的条件：

~~~java
if (handBooks.isEmpty()) {
    return
}
~~~

代码减少无意义换行，但逻辑块保持完整，不使用无大括号单行 if。

### 23.7 本次纠错记录

| 日期 | 纠错内容 | 最终规则 |
|---|---|---|
| 2026-07-17 | 将 sortBy 纠正为 sortedBy | 排序使用 sortedBy{} |
| 2026-07-17 | 将普通 if 多条件误认为不允许 | 普通 if 允许多个 &&、|| 条件 |
| 2026-07-17 | firstOrNull 中声明变量并执行多步逻辑 | 改用 forEach 和标记变量 |
| 2026-07-17 | 使用 !filter.isEmpty() 判断集合 | 改用显式遍历和布尔变量 |
| 2026-07-17 | 使用 new KTProcessingTradeHandBook() 作为占位对象 | 改为保存实际字段和选择状态 |

## 24. 官方模型开发与 SDK 文档补充（3.1.15 LTS）

- 整理日期：2026-07-23
- 文档来源：谷斗智能体平台“模型开发”实用技巧及 3.1.15（LTS）SDK 文档
- 适用范围：DO 模型代码、数据同步、基础工具、集成服务、Cplex、commons.lang、设备日历和数据服务
- 状态说明：本节来自平台官方文档。官方示例中存在拼写、类型和返回值错误时，以 API 签名、平台编译结果及实际运行结果为准，错误示例不得直接复制。

### 24.1 使用原则

1. 优先遵守本文件前文中已由实际业务代码或平台编译结果确认的规则。
2. SDK 文档中的方法签名可以作为能力依据，但示例代码仍需检查类型、变量名、返回值和拼写。
3. 涉及版本差异、废弃 API 或官方文档未完整说明的语义时，标记为“待验证”，不得自行套用 Java 行为。
4. 新代码只引入当前任务需要的 API，不为展示 SDK 能力而增加无关依赖。

## 25. 模型节点数据同步规范

状态：官方文档已确认。

### 25.1 禁止全量脏标记加嵌套查找

以下模式不再作为大数据量同步的推荐实现：

~~~java
Group<A>().forEach{ it.isDirty = true }
var toAdd = new List<A>()

Group<KTA>().forEach{ kt ->
    var a = Group<A>().find{ it.id == kt.id }
    if (a == null) {
        a = new A()
        toAdd.add(a)
    }
    a.updateFromKT(kt)
    a.isDirty = false
}

Group<A>().removeIf{ it.isDirty }
Group<A>().addAll(toAdd)
~~~

原因：

- 在知识表循环内反复扫描节点集合，时间复杂度约为 O(n²)；
- 同步专用的 `isDirty` 没有业务含义，会增加内存、持久化和变更收集成本；
- 节点属性作为临时状态会产生并发竞争；
- 多次全量扫描节点，数据利用率低。

### 25.2 使用 Map 做一次性匹配

推荐先按业务唯一键建立 Map，再通过 `remove` 同时完成匹配与剩余数据识别：

~~~java
var oldMap = new Map<String, A>()
Group<A>().forEach{ oldMap.put(it.id, it) }

var synchronized = new List<A>()
Group<KTA>().forEach{ kt ->
    var a = oldMap.remove(kt.id)
    if (a == null) {
        a = new A()
    }
    a.updateFromKT(kt)
    synchronized.add(a)
}

Group<A>().removeAll(oldMap.values())
Group<A>().addAll(synchronized)
~~~

规则：

1. Map 的键必须是真正的业务唯一键；复合键需使用无歧义的组合方式。
2. `oldMap.remove(key)` 返回已存在对象，同时将其从“待删除”集合中移除。
3. 同步结束后，`oldMap.values()` 只包含源数据中已不存在的旧对象。
4. 优先使用 `removeAll` 删除明确对象集合，不使用全量 `removeIf{ isDirty }`。
5. 临时状态只保存在方法局部变量中，不向模型增加同步专用字段。
6. 若源数据可能包含重复键，必须先校验或定义确定的冲突处理规则。

## 26. 基础工具 SDK 约束

### 26.1 通用函数和基础类型

- `maxOf(...)`、`minOf(...)` 当前最多支持 5 个参数。
- `Any.toString()` 用于字符串转换，`Any.equals(Any)` 用于值比较。
- `Number.intValue()`、`Number.doubleValue()` 用于数字类型转换。
- `Double.valueOf(Int/String)` 可从整数或字符串构造 Double。
- Double 支持 `abs`、`ceil`、`floor`、`round`、`roundUp`、`roundDown`、`roundHalfUp`、`roundHalfDown`、`pow` 等方法。
- 舍入方法返回新值。调用时必须接收返回值，不假设原变量被原地修改。

~~~java
var rounded = value.roundHalfDown(2)
~~~

### 26.2 查询专用加速 API

`inValues`、`subset`、`likeValues` 只能用于构建查询条件，禁止在普通模型代码中调用。官方文档明确指出，错误使用可能造成内存泄漏。

- `value.inValues(values, splitter)`：判断当前值是否存在于给定范围；
- `collection.subset(values, splitter)`：判断给定数据是否存在于当前范围；
- `value.likeValues(values, splitter)`：类似 SQL `LIKE '%value%'`；
- 当前支持 String、Int、Double、Boolean；
- splitter 为空时，默认分隔符为逗号。

### 26.3 控制语句

官方 SDK 确认支持：

- `while`；
- C 风格 `for`；
- `if / else if / else`；
- `break` 和 `continue`；
- `try / catch`；
- `throw`。

`try` 与 `catch(e)` 必须同时出现，且只能有一个 `catch`。当前异常变量 `e` 的类型为 Any，异常体系尚不完整。

~~~java
try {
    riskyOperation()
} catch (e) {
    log.error("执行失败")
    throw e
}
~~~

### 26.4 transient 属性

`transient` 写在属性类型之前。该属性：

1. 不在页面数据中展示；
2. 不保存到全量数据库；
3. 不通过集成服务传输，但场景间调用除外。

~~~java
transient String runtimeOnlyValue
~~~

### 26.5 定长字符串处理

- `TextFormat(N)`、`VarCharFormat(N)`、`VarChar2Format(N)` 限制最大长度；`-1` 表示不限制。
- `NumberFormat(M, N)` 中 M 为总数字位数，N 为小数位数，整数位数为 M-N。
- `StringRangeTemplate(charset)` 支持按指定编码定义定长片段。
- 使用 `addSegment` 时必须确认 start/end 的边界语义、编码和必填标志。
- GBK 与 UTF-8 的字节长度不同，不能按字符数量替代编码后的长度。

## 27. 集成服务规范

### 27.1 入口与任务链

`integrationDispatcher()` 返回 IntegrationDispatcher，是协议、节点和任务链的入口。

常用能力：

- `createTaskChains(code, name[, nodes])`；
- `getTaskChainsByCode(code)`；
- `execute(code)` 或 `execute(nodes)`；
- 创建方法、SOAP、数据库等协议；
- 创建并执行 HTTP、REST WebService、方法、数据库等节点。

任务链和协议的 code 是唯一标识，必须稳定且不可重复。

### 27.2 跨场景调用优先级

官方文档不建议通过任务链在场景方法之间传输数据，优先使用：

- `callSceneStaticMethod`：同步调用静态方法；
- `asyncSceneStaticMethod`：异步调用静态方法；
- `asyncCallSceneStaticMethod`：异步调用并处理回调。

跨场景参数目前只能使用 List、Map 或简单类型。传入前必须转换为这些类型，且参数对象不可复用，避免场景之间共享引用、相互修改。

### 27.3 对象传输规则

- compute 对象传输时转换为 Map；
- compute 内嵌套的 compute 属性不会继续传输，其值为 null；
- 集合转换为 List；
- Map 保持为 Map；
- Group 不属于支持的返回集合；
- 通过集成任务调用的方法，除 Context 外最多只能声明一个业务参数。

### 27.4 SOAP、HTTP 和 REST

- SOAP/REST 请求参数必须是 Map，或可以解析为 Map 的 JSON 字符串；
- 其他参数类型会报错；
- HTTP 和 REST 节点的 URL、请求方式、参数、请求头、媒体类型必须显式设置；
- 不得在代码中硬编码真实密码、令牌或生产连接信息，应从安全配置中读取。

### 27.5 数据库节点

- SELECT 参数：`List<Any>`；
- INSERT/UPDATE/DELETE 批量参数：`List<List<Any>>`；
- 无参数 SQL 按 API 要求传 null；
- 不同数据库 SQL 方言不同，不能直接复制其他数据库示例；
- 返回值可能是 JSON 字符串或动态 Map，使用前必须按实际类型解析。

### 27.6 废弃 API

官方 3.1.15 文档仍列出但已标记废弃的 API，例如旧 `createMQAgreement` 和 `nodeMappingHelper`。新代码禁止继续引入；维护旧代码时先确认平台当前版本是否仍保留兼容实现。

## 28. Cplex SDK 规范

### 28.1 推荐建模顺序

每个计算类通过 `cplex()` 获取 CplexExecutor。推荐流程：

1. 创建决策变量；
2. 构造线性或二次约束；
3. 设置目标函数；
4. 设置求解参数；
5. 调用 `solve()`；
6. 检查求解状态；
7. 读取目标值和变量值。

### 28.2 变量、表达式和约束

- 变量类型：`IloNumVarType.Float`、`Int`、`Bool`；
- 变量通过名称和实例列表共同定位；
- 线性表达式使用 `linearNumExpr()` 与 `addTerm()`；
- 约束方向使用 `leSense()`、`eqSense()`、`geSense()`；
- 二次约束使用 `addQConstr()`；
- 目标函数可使用 `addMaximize()` 等平台已提供的方法。

### 28.3 求解结果检查

禁止无条件调用 `getObjValue()` 或 `getValue()`。应先检查 `solve()` 的返回值和 `getCplexStatus()`。

~~~java
cplex().function(model -> {
    // 创建变量、约束和目标函数
}).function(model -> {
    model.setParam(IloCplex.Param.TimeLimit, 10)
}).function(model -> {
    var solved = model.solve()
    if (!solved) {
        log.info("Cplex 未得到可用解，状态="+model.getCplexStatus())
        return
    }
    var objective = model.getObjValue()
    var value = model.getValue(model.getVarByName("x", instances))
}).exec()
~~~

Cplex SDK 是 Java API 的 DO 映射。平台未列出的参数、状态或过期能力，必须结合平台集成的 Cplex 版本验证。

## 29. commons.lang 辅助工具规范

### 29.1 多字段排序

`ComparatorBuilder<T>` 按选择器加入顺序比较，默认升序。支持 Int、Double、String、Boolean、Date、DateTime、Duration、Decimal 选择器。

~~~java
var comparator = new ComparatorBuilder<Task>()
    .dateTimeSelector(it -> it.startTime)
    .intSelector(it -> it.priority)
    .desc()

var sorted = CollectionUtils.sorted(tasks, comparator)
~~~

`CollectionUtils.sorted` 返回新的 List，不应假设原集合被原地修改。

### 29.2 UUID 和对象转换

- `UUIDUtil.generateUUID()`：生成带连字符 UUID；
- `UUIDUtil.generateCompactUUID()`：生成不带连字符的紧凑 UUID；
- `AnyUtil.anyToMap(any)`：将对象属性拆解为 `Map<String, Any>`，适合集成边界转换。

### 29.3 29s 电文解析

`IXCom29sUtil.transformBody(body, args[, charset])` 按字段描述拆解电文：

- `字段名@长度` 定义字段；
- 逗号后的数字表示小数位；
- charset 当前文档明确支持 GBK；
- GBK 模式必须按编码字节长度计算，不能按 Java/DO 字符串长度直接推断。

## 30. 设备日历规范

### 30.1 禁止直接修改日历属性

官方文档明确说明日历属性只读。禁止直接编辑 Calendar、CalendarRule、TimeInterval 等对象的受管属性，必须通过 CalendarManager 和相应实例方法修改。

### 30.2 规则变更批量生效

绑定、解绑、修改、删除规则只记录变更；调用 `consumerRuleChangedEvents()` 后，规则才会对日历产生或更新停机时间。

推荐：

1. 批量创建或修改日历和规则；
2. 完成 bind/unbind；
3. 最后调用一次 `consumerRuleChangedEvents()`。

### 30.3 局部更新

`update...IgnoreNull()` 会忽略 null 参数。调用重载方法时应为 null 显式指定类型，避免匹配到错误重载。

~~~java
manager.updateRuleByIdIgnoreNull(
    rule.id,
    (String)null,
    (String)null,
    (DateTime)null,
    (DateTime)null,
    "week",
    (Int)null
)
~~~

### 30.4 时间区间生成

- Calendar 与 CalendarRule 的时间范围共同限制有效区间；
- `repeatCycle` 和 `repeatFrequency` 定义重复周期；
- TimeIntervalTemplate 在每个有效周期中生成 TimeInterval；
- 模板时间不在当前周期时，会按规则顺延；
- 修改周期、频率、模板或绑定关系后必须重新消费变更事件。

### 30.5 场景与持久化

- 场景默认 CalendarManager：`context().calendarManager()`；
- 只有 Group 中的数据会持久化；
- CalendarManager 接入响应式上下文后，其管理的数据会同步至 Group。

## 31. 数据服务规范

### 31.1 data 数据类

使用 `data` 声明数据库实体。数据类默认包含创建者、创建时间、修改者、修改时间等审计字段。

支持的属性类型包括 Boolean、Int、Double、String、Decimal、Date、DateTime、Duration。Serial 在当前文档中标记为尚未支持。

Decimal 默认总长度 38、精度 6。Duration 在数据库中按数值存储，使用原始 SQL 时需通过 `toMillis()` 转为毫秒。

### 31.2 主键、唯一约束和索引

- 主键不可省略；
- 未显式声明主键时，平台默认尝试使用名为 id 的字段；
- 主键定义后不可修改；
- 使用 `unique(...)` 声明唯一约束；
- 使用 `index(...)` 声明索引；
- 复合约束和索引中的字段顺序属于定义的一部分。

~~~java
data Example {
    Int id
    String name
    Int age

    primary(id)
    unique(name)
    index(age, name)
}
~~~

### 31.3 DbHelper 增删改

DbHelper 支持：

- `insert`、`update`、`delete`、`insertOrUpdate`；
- 对应的 Batch 批量方法；
- `transactional` 事务代码块；
- DSL 查询和原始 SQL。

优先批量操作，避免在循环中逐条访问数据库。

### 31.4 事务和缓存上下文

事务开启时创建持久化上下文，提交或回滚时销毁。事务内 DSL 查询出的数据实例由上下文托管；先查询再批量更新或删除可以减少重复查询。

禁止把网络调用、求解、大循环等长耗时业务放入数据库事务代码块，避免长事务和锁占用。

### 31.5 原始 SQL

原始 SQL 不自动处理以下事项：

- 数据类名与真实表名映射；
- 属性名驼峰转数据库下划线；
- 审计字段；
- Duration 毫秒转换；
- 参数安全。

使用 `actualTableName(Class)` 获取真实表名。包含业务输入的 SQL 必须使用平台支持的参数化方式，禁止字符串拼接造成注入风险。

### 31.6 QueryDSL

优先使用编辑器生成的 Q 辅助类：

~~~java
var qUser = QUser.quser
var users = context.getDbHelper()
    .dslSelectFrom(qUser)
    .where(qUser.age.eq(18))
    .orderBy(List.of(qUser.name.asc()))
    .findAll()
~~~

支持投影、join、groupBy、having、offset、limit、orderBy、update 和 delete。

仅用于构造 DSL 语句的 SelectStatement 不支持 `findOne` 和 `findAll`；需要真实访问数据库时必须从 DbHelper 创建可执行查询。

## 32. 官方文档示例纠错原则

本次整理发现官方示例存在多处明显问题，包括但不限于：

- `minOf` 示例误写为 `maxOf`；
- `roundDown` 示例误调用 `roundUp`；
- `DateTime.parse` 多处误写为 `DateTime.pase`；
- 示例日期包含不存在的月份或日期；
- 部分变量、Q 类、集合添加对象和返回类型前后不一致；
- 个别描述把降序写成升序；
- 示例包含真实样式的数据库口令或地址，不得复制到业务代码。

处理规则：

1. API 是否存在以标题签名和平台编译结果为准；
2. 示例与签名冲突时，不把示例作为语法证据；
3. 生产代码不得复制文档中的地址、账号、密码和令牌；
4. 发现新错误时，在本文件变更记录中追加，不静默修正后假装已确认。

## 33. 本次整理变更记录

| 日期 | 来源 | 修正内容 | 影响规则 |
|---|---|---|---|
| 2026-07-23 | 模型开发文档 | 增加 Map 匹配、removeAll 删除和禁止 isDirty 全量同步规范 | 25 |
| 2026-07-23 | 3.1.15 基础工具 SDK | 增加查询专用 API、控制语句、异常、transient 和定长文本规则 | 26 |
| 2026-07-23 | 3.1.15 集成服务 SDK | 增加跨场景参数、对象传输、协议节点、数据库节点及废弃 API 规则 | 27 |
| 2026-07-23 | 3.1.15 Cplex SDK | 增加建模顺序、变量约束和求解状态检查规范 | 28 |
| 2026-07-23 | 3.1.15 commons.lang SDK | 增加比较器、集合排序、UUID、Any 转 Map 和 29s 电文规则 | 29 |
| 2026-07-23 | 3.1.15 设备日历 SDK | 增加只读属性、规则批量生效、时间区间和持久化规范 | 30 |
| 2026-07-23 | 3.1.15 数据服务 SDK | 增加 data、约束索引、DbHelper、事务、原始 SQL 和 QueryDSL 规范 | 31 |
| 2026-07-23 | 官方示例核对 | 记录示例笔误、类型错误和敏感配置复制风险 | 32 |

## 附录 A：官方文档完整目录与 API 索引

- 对应平台版本：3.1.15（LTS）
- 整理日期：2026-07-23
- 收录范围：模型开发、基础工具、集成服务、Cplex、commons.lang、设备日历、数据服务。
- 本附录按官方页面顺序保留全部章节标题、类、属性和方法标题；对于未作为标题展示的方法，补充其正文签名。
- “参数”“返回值”“示例”等重复条目仍保留，以保证与原文目录逐项对应。
- 具体语义、限制和推荐写法以前文规范为准；官方示例中的已知错误不得直接复制。
- 内网来源地址仅用于平台内部定位。

### 数据服务

来源：`http://172.16.9.88/docs/plat3/3.1.15/SDKDoc/%E6%95%B0%E6%8D%AE%E6%9C%8D%E5%8A%A1`

- 1. 快速开始
- 2. 数据定义
- 2.1 类型定义
- 2.2 属性定义
- 2.3 约束定义
- 2.4 索引定义
- 3. 数据操作
- 3.1 DbHelper
- 3.1.1 实例方法
- 3.1.1.1 void insert(DoAny any)
- 3.1.1.2 void update(DoAny any)
- 3.1.1.3 void delete(DoAny any)
- 3.1.1.4 void insertOrUpdate(DoAny any)
- 3.1.1.5 void insertBatch(Collection<?> c)
- 3.1.1.6 void updateBatch(Collection<?> c)
- 3.1.1.7 void deleteBatch(Collection<?> c)
- 3.1.1.8 void insertOrUpdateBatch(Collection<?> c)
- 3.1.1.8 void transactional(func()void fun)
- 3.1.1.9 原始 sql 操作，注意事项
- 3.1.1.10 String actualTableName(Class clazz)
- 3.1.1.11 List<Map<String, Any>> nativeSelect(String sql)
- 3.1.1.12 <T> List<T> nativeSelect(String sql, Clazz<T> resultClazz)
- 3.1.1.13 int nativeInsert(String sql)
- 3.1.1.14 int nativeUpdate(String sql)
- 3.1.1.15 int nativeDelete(String sql)
- 3.2 dsl
- 3.2.1 dsl 增删改查方法
- DBhelper 增删改查
- 3.2.2 dsl 查询语法
- 3.2.3 dsl 插入语法
- 3.2.4 dsl 修改语法
- 3.2.5 dsl 删除语法
- 3.2.6 dsl 表达式 api
- 3.2.6.1 简单表达式
- 3.2.6.1.1 父类
- 3.2.6.1.2 实例方法
- 3.2.6.2 可比较的基本表达式~~~sql
- 3.2.6.2.1 父类
- 3.2.6.2.2 实例方法
- 3.2.6.3 可比较的表达式
- 3.2.6.3.1 父类
- 3.2.6.3.2 实例方法
- 3.2.6.4 布尔类型表达式
- 3.2.6.4.1 父类
- 3.2.6.4.2 实例方法
- 3.2.6.5 字符串表达式
- 3.2.6.5.1 父类
- 3.2.6.5.2 实例方法
- 3.2.6.6 整型表达式
- 3.2.6.6.1 父类
- 3.2.6.6.2 实例方法
- 3.2.6.7 浮点型表达式
- 3.2.6.7.1 父类
- 3.2.6.7.2 实例方法
- 3.2.6.8 日期表达式
- 3.2.6.8.1 父类
- 3.2.6.8.2 实例方法
- 3.2.6.9 日期表达式
- 3.2.6.9.1 父类
- 3.2.6.9.2 实例方法
- 3.2.6.10 时间间隔表达式
- 3.2.6.10.1 父类
- 3.2.6.10.2 实例方法
- 3.2.6.11 排序说明符
- 3.2.6.11.1 父类
- 3.2.6.11.2 实例方法
- 3.2.7 com.goodo.querydsl.SelectStatement 查询
- 3.2.8 QueryDSL 示例
- 3.2.8.1 前言
- 3.2.8.2 类型定义
- 3.2.8.3 创建初始数据
- 3.2.8.4 基本查询
- 3.2.8.5 投影查询
- 3.2.8.6 join 查询
- 3.2.8.7 group 查询
- 3.2.8.8 分页和排序
- 3.2.8.9 更新数据
- 3.2.8.10 删除数据
- 3.3 缓存上下文
- 3.3.1 生命周期
- 3.3.2 作用
- 3.3.3 用法
- 3.3.3.1 先查询再修改
- 3.3.3.2 先查询再删除

补充签名（原文以正文代码列出，未全部设为标题）：

- `<T> com.goodo.querydsl.Query<T> dslSelect(com.goodo.querydsl.EntityPathBase<T> epb)`
- `com.goodo.querydsl.Query<Map<com.goodo.querydsl.SimpleExpression, Any>> dslSelect(List<com.goodo.querydsl.SimpleExpression> columns)`
- `<T> com.goodo.querydsl.Query<T> dslSelectFrom(com.goodo.querydsl.EntityPathBase<T> epb)`
- `com.goodo.querydsl.Insert dslInsert(com.goodo.querydsl.EntityPathBase epb)`
- `com.goodo.querydsl.Update dslUpdate(com.goodo.querydsl.EntityPathBase epb)`
- `com.goodo.querydsl.Delete dslDelete(com.goodo.querydsl.EntityPathBase epb)`
- `com.goaodo.querydsl.Query<T> from(com.goodo.querydsl.EntityPathBase epb)`
- `com.goodo.querydsl.Query<T> from(List<com.goodo.querydsl.EntityPathBase> epbs)`
- `com.goodo.querydsl.Query<T> where(com.goodo.querydsl.BooleanExpression predicate)`
- `com.goodo.querydsl.Query<T> innerJoin(com.goodo.querydsl.EntityPathBase epb)`
- `com.goodo.querydsl.Query<T> leftJoin(com.goodo.querydsl.EntityPathBase epb)`
- `com.goodo.querydsl.Query<T> rightJoin(com.goodo.querydsl.EntityPathBase epb)`
- `com.goodo.querydsl.Query<T> on(com.goodo.querydsl.BooleanExpression predicate)`
- `com.goodo.querydsl.Query<T> orderBy(com.goodo.querydsl.OrderSpecifier os)`
- `com.goodo.querydsl.Query<T> orderBy(List<com.goodo.querydsl.OrderSpecifier> oss)`
- `com.goodo.querydsl.Query<T> groupBy(com.goodo.querydsl.SimpleExpression se)`
- `com.goodo.querydsl.Query<T> groupBy(List<com.goodo.querydsl.SimpleExpression> ses)`
- `com.goodo.querydsl.Query<T> having(com.goodo.querydsl.BooleanExpression be)`
- `com.goodo.querydsl.Query<T> having(List<com.goodo.querydsl.BooleanExpression> bes)`
- `com.goodo.querydsl.Query<T> limit(Int i)`
- `com.goodo.querydsl.Query<T> offset(Int i)`
- `T findOne()`
- `List<T> findAll()`
- `com.goodo.querydsl.Insert columns(List<com.goodo.querydsl.SimpleExpression> cs)`
- `com.goodo.querydsl.Insert values(List<Any> vs)`
- `com.goodo.querydsl.Insert execute()`
- `com.goodo.querydsl.Update set(com.goodo.querydsl.SimpleExpression column, Any value)`
- `com.goodo.querydsl.Update where(com.goodo.querydsl.BooleanExpression predicate)`
- `com.goodo.querydsl.Update execute()`
- `com.goodo.querydsl.Delete set(com.goodo.querydsl.SimpleExpression column, Any value)`
- `com.goodo.querydsl.Delete where(com.goodo.querydsl.BooleanExpression predicate)`
- `com.goodo.querydsl.Delete execute()`
- `count(column)`
- `count(distinct column)`
- `column in (t1, t2)`
- `column not in (t1, t2)`
- `min(column)`
- `max(column)`
- `sum(column)`
- `avg(column)`

### 设备日历

来源：`http://172.16.9.88/docs/plat3/3.1.15/SDKDoc/%E8%AE%BE%E5%A4%87%E6%97%A5%E5%8E%86`

- 1. 日历SDK文档
- 1.1 核心类
- 1.1.1 CalendarManager
- 1.1.1.1 构造器
- 1.1.1.1.1 无参构造
- 1.1.1.2 实例方法
- 1.1.1.2.1 void addCalendar(Calendar)
- 1.1.1.2.2 void removeCalendar(String)
- 1.1.1.2.3 Calendar getCalendarById(String)
- 1.1.1.2.4 List<Calendar> getCalendars()
- 1.1.1.2.5 CalendarRule insertRule(String, String, DateTime, DateTime, String, Int)
- 1.1.1.2.6 CalendarRule insertRule(String, String, String, String, String, Int)
- 1.1.1.2.7 List<CalendarRule> getRules()
- 1.1.1.2.8 CalendarRule updateRuleByIdIgnoreNull(String, String, String, DateTime, DateTime, String, Int)
- 1.1.1.2.9 CalendarRule updateRuleByIdIgnoreNull(String, String, String, String, String, String, Int)
- 1.1.1.2.10 void deleteRuleById(String)
- 1.1.1.2.11 CalendarRule getRuleById(String)
- 1.1.1.2.12 List<CalendarRule> getCalendarBindingRules(String)
- 1.1.1.2.13 void bind(String, String)
- 1.1.1.2.14 void unbind(String, String)
- 1.1.1.2.15 void consumerRuleChangedEvents()
- 1.1.2 Calendar
- 1.1.2.1 构造器
- 1.1.2.1.1 无参构造
- 1.1.2.2 属性
- 1.1.2.2.1 id : String
- 1.1.2.2.2 range : CalendarRange 日历范围
- 1.1.2.2.3 timeIntervals : Map<String, TimeInterval> 停机事件集合
- 1.1.2.2.4 createByRulesTiMap : Map<String, List<String>> 由日历规则创建的停机时间记录
- 1.1.2.3 实例方法
- 1.1.2.3.1 TimeInterval insertTimeInterval(DateTime, DateTime, Int, String)
- 1.1.2.3.2 TimeInterval insertTimeInterval(DateTime, DateTime, Double, String)
- 1.1.2.3.3 void addTimeInterval(TimeInterval)
- 1.1.2.3.4 void addTimeIntervals(List<TimeInterval>)
- 1.1.2.3.5 void deleteTimeIntervalById(String)
- 1.1.2.3.6 void deleteTimeIntervalByIds(List<String>)
- 1.1.2.3.7 TimeInterval updateTimeIntervalByIdIgnoreNull(String, DateTime, DateTime, Double, String)
- 1.1.2.3.8 TimeInterval updateTimeIntervalByIdIgnoreNull(String, String, String, Double, String)
- 1.1.2.3.9 Calendar updateRangeIgnoreNull(String, Int, Int)
- 1.1.2.3.10 DateTime previousAvailable(DateTime)
- 1.1.2.3.11 DateTime nextAvailable(DateTime)
- 1.1.2.3.12 DateTime previousUnavailable(DateTime)
- 1.1.2.3.13 DateTime nextUnavailable(DateTime)
- 1.1.2.3.14 Duration availableTime(DateTime, DateTime)
- 1.1.2.3.15 DateTime previousFit(DateTime, Duration)
- 1.1.2.3.16 DateTime nextFit(DateTime, Duration)
- 1.1.3 CalendarRange
- 1.1.3.1 构造器
- 1.1.3.1.1 无参构造
- 1.1.3.2 属性
- 1.1.3.2.1 activationTime : DateTime 启用时间
- 1.1.3.2.2 timeWindow : Int 时间窗口
- 1.1.3.2.3 timeOffset : Int 偏移量
- 1.1.4 CalendarRule
- 1.1.4.1 构造器
- 1.1.4.1.1 无参构造
- 1.1.4.2 属性
- 1.1.4.2.1 id
- 1.1.4.2.2 name : String 名称
- 1.1.4.2.3 description : String 描述
- 1.1.4.2.4 startTime : DateTime 开始时间
- 1.1.4.2.5 endTime : DateTime 结束时间
- 1.1.4.2.6 repeatCycle : String 周期
- 1.1.4.2.7 repeatFrequency : Int 频率
- 1.1.4.2.8 templates : Map<String, TimeIntervalTemplate> 停机时间模板id与停机时间模板映射
- 1.1.4.3 实例方法
- 1.1.4.3.1 TimeIntervalTemplate insertTemplate(DateTime， DateTime，Double，String)
- 1.1.4.3.2 TimeIntervalTemplate insertTemplate(DateTime， DateTime，Double，String)
- 1.1.4.3.3 TimeIntervalTemplate updateTemplateByIdIgnoreNull(String, DateTime， DateTime，Double，String)
- 1.1.4.3.4 void deleteTemplateById(String)
- 1.1.5 TimeInterval
- 1.1.5.1 构造器
- 1.1.5.1.1 无参构造
- 1.1.5.1.2 有参构造
- 1.1.5.2 属性
- 1.1.5.2.1 id : String 名称
- 1.1.5.2.2 startTime : DateTime 开始时间
- 1.1.5.2.3 endTime : DateTime 结束时间
- 1.1.5.2.4 capacity : Double产能
- 1.1.5.2.5 description : String 描述
- 1.1.5.2.6 modifyTimestamp : String 修改时间
- 1.1.6 TimeIntervalTemplate
- 1.1.6.1 构造器
- 1.1.6.2 属性
- 1.1.6.3 实例方法
- 1.2 使用帮助
- 1.2.1 快速开始
- 1.2.1.1 管理日历与日历规则 case
- 1.2.2 进阶
- 1.2.2.1 TimeInterval 生成业务逻辑
- 1.2.2.2 样例解读
- 1.2.2.3 平台交互
- 1.2.2.4 持久化

### 辅助工具 commons.lang

来源：`http://172.16.9.88/docs/plat3/3.1.15/SDKDoc/%E8%BE%85%E5%8A%A9%E5%B7%A5%E5%85%B7%20commons.lang`

- 1. 比较器构造器
- 1.1 实例方法
- 1.1.1 Int 选择器
- 1.1.2 Double 选择器
- 1.1.3 String 选择器
- 1.1.4 Boolean 选择器
- 1.1.5 Date 选择器
- 1.1.6 DateTime 选择器
- 1.1.7 Duration 选择器
- 1.1.8 Decimal 选择器
- 1.1.9 升序
- 1.1.10 降序
- 2. 集合工具
- 2.1 静态方法
- 2.2.1 排序
- 3. uuid 生成工具包
- 3.1 String generateUUID()
- 3.2 String generateCompactUUID()
- 4. AnyUtil 工具类
- 4.1 静态属性
- 4.2 静态方法
- 4.2.1 Map<String,Any> anyToMap(Any any) 将any实例分解为一个Map
- 4.2.2 实例方法
- 5. IXCom29sUtil工具类
- 5.1 静态属性
- 5.2 静态方法
- 5.2.1 Map<String,Any> transformBody(String string , List<String> args) 拆解29s电文数据
- 5.2.2 Map<String,Any> transformBody(String string , List<String> args, String charset)
- 5.3 实例方法

### Cplex工具

来源：`http://172.16.9.88/docs/plat3/3.1.15/SDKDoc/Cplex%E5%B7%A5%E5%85%B7`

- 1. 核心类
- 1.1 IloCplex
- 1.1.1 实例方法
- 1.2 IloNumVarType
- 1.2.1 静态属性
- 1.2.2 实例方法
- 1.3 IloNumVar extends IloNumExpr implement IloAddable
- 1.3.1 实例方法
- 1.4 IloLinearNumExpr extends IloNumExpr
- 1.4.1 实例方法
- 1.5 IloObjective implement IloAddable
- 1.6 Param
- 1.7 CplexStatus
- 1.8 CplexExecutor
- 1.8.1 实例方法
- 2. 快速开始

补充签名（原文以正文代码列出，未全部设为标题）：

- `IloNumVar addIloNumVar( Double lb, Double ub, IloNumVarType type, String name, List<Any> instances )`
- `IloNumVar getVarByName(String name, List<Any> instances)`
- `Double getVarOptimalValueByName(String name, List<Any> instances)`
- `IloConstraInt addConstr( IloNumExpr expr, String sense, Double rhs, String name, List<Any> instances )`
- `String leSense()`
- `String eqSense()`
- `String geSense()`
- `// 增加 Q 静态变量 IloConstraInt addQConstr( IloNumExpr expr, String sense, Double rhs, String name, List<Any> instances )`
- `IloLinearNumExpr linearNumExpr()`
- `void setParam(Param param, Number i)`
- `Double tuneParam()`
- `IloObjective addMaximize(IloNumExpr iloNumExpr)`
- `void writeParam(String s)`
- `Boolean solve()`
- `CplexStatus getCplexStatus()`
- `Double getValue(IloNumVar iloNumVar)`
- `Double getObjValue()`
- `void remove(IloObjective iloObjective)`
- `void exportModel(String name)`
- `Int getTypeValue()`
- `IloNumVarType getType()`
- `Double getLB()`
- `Double getUB()`
- `void setLB(Double v)`
- `void setUB(Double v)`
- `String getName()`
- `void setName(String s)`
- `void addTerm(Double v, IloNumVar iloNumVar)`
- `void addTerm(IloNumVar iloNumVar, Double v)`
- `void add(IloLinearNumExpr iloLinearNumExpr)`
- `void clear()`
- `Double getConstant()`
- `void setConstant(Double v)`
- `void remove(IloNumVar iloNumVar)`

### 集成服务

来源：`http://172.16.9.88/docs/plat3/3.1.15/SDKDoc/%E9%9B%86%E6%88%90%E6%9C%8D%E5%8A%A1`

- 1. IntegrationDispatcher
- 1.1 createTaskChains(string code, string name)
- 1.1.1 参数
- 1.1.2 返回值
- 1.1.3 示例
- 1.2 createTaskChains(string code, string name, List<Node> nodes)
- 1.2.1 参数
- 1.2.2 返回值
- 1.2.3 示例
- 1.3 getTaskChainsByCode(string code)
- 1.3.1 参数
- 1.3.2 返回值
- 1.3.3 示例
- 1.4 createSoapAgreement(string code, string name, string wsdlUrl)
- 1.4.1 参数
- 1.4.2 返回值
- 1.4.3 示例
- 1.5 createMethodAgreement(string code, string name, string projectId, string sceneId)
- 1.5.1 参数
- 1.5.2 返回值
- 1.5.3 示例
- 1.6 createMethodAgreementByProjectCodeAndSceneCode(string code, string name, string projectCode, string sceneCode)
- 1.6.1 参数
- 1.6.2 返回值
- 1.6.3 示例
- 1.6.4 注意事项
- 1.7 createDatabaseAgreement(string code, string name, string url, string userName, string password)
- 1.7.1 参数
- 1.7.2 返回值
- 1.7.3 示例
- 1.8 createMQAgreement(string code, string name) (已废弃，计划在3.1.3版本中删除)
- 1.8.1 参数
- 1.8.2 返回值
- 1.8.3 示例
- 1.9 nodeMappingHelper() (已废弃，计划在3.1.3版本中删除)
- 1.9.1 返回值
- 1.9.2 示例
- 1.10 execute(string code)
- 1.10.1 参数
- 1.10.2 示例
- 1.11 execute(List<Node> nodes)
- 1.11.1 参数
- 1.11.2 示例
- 1.12 callSceneStaticMethod(String project , String scene ,String class ,String method , Any args)
- 1.12.1 参数
- 1.13 asyncSceneStaticMethod(String project , String scene ,String class ,String method, Any args)
- 1.13.1 参数
- 1.14 asyncCallSceneStaticMethod(Context context, String project, String scene, String class, String method, Any args, Either rs)
- 1.14.1 参数
- 2. Either
- 2.1 of(success: func(Any)Void , fail:func(Void)Void))
- 2.1.1 参数
- 2.1.2 返回
- 2.2 of(success: func(Any)Void)
- 2.2.1 参数
- 2.2.2 返回
- 3. TaskChains
- 3.1 addLast(Node node)
- 3.1.1 参数
- 3.1.2 示例
- 3.2 addBefore(Node node, string preNodeCode)
- 3.2.1 参数
- 3.2.2 示例
- 3.3 addBefore(Node node, Node preNode)
- 3.3.1 参数
- 3.3.2 示例
- 3.4 remove(string code)
- 3.4.1 参数
- 3.4.2 示例
- 3.5 remove(Node node)
- 3.5.1 参数
- 3.5.2 示例
- 3.6 save()
- 3.6.1 示例
- 3.7 getNode(string code)
- 3.7.1 参数
- 3.7.2 示例
- 4. Agreement
- 4.1 save()
- 4.1.1 示例
- 4.2 saveIfNotExist()
- 4.2.1 示例
- 4.3 getNodes()
- 4.3.1 返回值
- 4.3.2 示例
- 4.4 getNode(string code)
- 4.4.1 返回值
- 4.4.2 示例
- 5. DatabaseAgreement
- 5.1 createNode(string code, string name, string operator, string sql)
- 5.1.1 参数
- 5.1.2 返回值
- 6. MethodAgreement
- 6.1 createNode(string code, string name, String className, string methodName)
- 6.1.1 参数
- 6.1.2 返回值
- 6.1.3 示例
- 6.2 createNode(string code, string name,string className, string methodName, boolean startTransaction)
- 6.2.1 参数
- 6.2.2 返回值
- 6.2.3 示例
- 6.3 void setStatic()
- 7. MQAgreement (已废弃，计划在3.1.3版本中删除)
- 7.1 setHost(string host)
- 7.1.1 参数
- 7.1.2 示例
- 7.2 setPort(number port)
- 7.2.1 参数
- 7.2.2 示例
- 7.3 setUsername(string username)
- 7.3.1 参数
- 7.3.2 示例
- 7.4 setPassword(string password)
- 7.4.1 参数
- 7.4.2 示例
- 7.5 setVirtualHost(string virtualHost)
- 7.5.1 参数
- 7.5.2 示例
- 8. SoapAgreement
- 8.1 createNode(string code, string name, string namespace, string methodName, boolean soapAction)
- 8.1.1 参数
- 8.1.2 返回值
- 8.1.3 示例
- 8.2 setHeader(string key, string value)
- 8.2.1 示例
- 8.3 getHeaders()
- 8.3.1 返回值
- 8.3.2 示例
- 8.4 setHeaders(Map <string, string> headers)
- 8.4.1 参数
- 8.4.2 示例
- 9. Node
- 9.1 getIXCom29sNode(string code)
- 9.1.1 参数
- 9.1.2 返回值
- 9.1.3 示例
- 9.2 createHttp(string code, string name, string url)
- 9.2.1 参数
- 9.2.2 返回值
- 9.2.3 示例
- 9.3 createHttp(string code, string name, string url,number readTimeout)
- 9.3.1 参数
- 9.3.2 返回值
- 9.3.3 示例
- 9.4 createRestWebservice(string code, string name, string url)
- 9.4.1 参数
- 9.4.2 返回值
- 9.4.3 示例
- 9.5 createRestWebservice(string code, string name, string url,number readTimeout)
- 9.5.1 参数
- 9.5.2 返回值
- 9.5.3 示例
- 9.6 getUrl()
- 9.6.1 返回值
- 9.6.2 示例
- 9.7 getName()
- 9.7.1 返回值
- 9.7.2 示例
- 9.8 getCode()
- 9.8.1 返回值
- 9.8.2 示例
- 9.9 getType()
- 9.9.1 返回值
- 9.9.2 示例
- 9.10 save(string agreementId)
- 9.10.1 参数
- 9.10.2 示例
- 9.11 execute(any param)
- 9.11.1 参数
- 9.11.2 返回值
- 9.11.3 示例
- 9.12 execute(any param ，Class t)
- 9.12.1 参数
- 9.12.2 返回值
- 9.12.3 示例
- 9.13 equalsCode(string code)
- 9.13.1 参数
- 9.13.2 返回值
- 9.13.3 示例
- 10. DatabaseNode
- 11. IXCom29sNode
- 11.1 sendMes(String mesNo,String mesBody) 发送IXCom29s协议消息
- 11.1.1 参数
- 11.1.2 示例
- 11.2 sendMes(String mesNo,String mesBody, String charset) 发送IXCom29s协议消息
- 11.2.1 参数
- 11.2.2 示例
- 12. HttpNode
- 12.1 setParameters(Map<string, string> parameters)
- 12.1.1 参数
- 12.1.2 示例
- 12.2 addHeader(Map<string, string> headers)
- 12.2.1 参数
- 12.2.2 示例
- 12.3 addBasic(string username, string password)
- 12.3.1 参数
- 12.3.2 示例
- 12.4 setContentType(string contentType)
- 12.4.1 参数
- 12.4.2 示例
- 12.5 setMethod(string method)
- 12.5.1 参数
- 12.5.2 示例
- 12.6 setReadTimeout(number readTimeout)
- 12.6.1 参数
- 12.6.2 示例
- 13. RestWebserviceNode
- 13.1 setPrintLog(Boolean)
- 13.1.1 示例
- 13.2 addHeader(Map<string, string> headers)
- 13.2.1 参数
- 13.2.2 示例
- 13.3 addBasic(string username, string password)
- 13.3.1 参数
- 13.3.2 示例
- 13.4 setContentType(string contentType)
- 13.4.1 参数
- 13.4.2 示例
- 13.5 setMethod(string method)
- 13.5.1 示例
- 13.6 addSuffix(string suffix)
- 13.6.1 示例
- 13.7 addPrefix(string perfix)
- 13.7.1 示例
- 13.8 setReadTimeout(number readTimeout)
- 13.8.1 参数
- 13.8.2 示例
- 14. SoapNode
- 14.1 setPrintLog(Boolean)
- 14.1.1 示例
- 15. MethodNode
- 16. NodeMapping (已废弃，计划在3.1.3版本中删除)
- 16.1 getCode()
- 16.1.1 返回值
- 16.1.2 示例
- 16.2 setCode(string code)
- 16.2.1 参数
- 16.2.2 示例
- 16.3 getName()
- 16.3.1 返回值
- 16.3.2 示例
- 16.4 setName(string name)
- 16.4.1 参数
- 16.4.2 示例
- 16.5 getModel()
- 16.5.1 返回值
- 16.5.2 示例
- 16.6 setModel(string model)
- 16.6.1 参数
- 16.6.2 示例
- 16.7 getScene()
- 16.7.1 返回值
- 16.7.2 示例
- 16.8 setScene(string scene)
- 16.8.1 参数
- 16.8.2 示例
- 16.9 getClassName()
- 16.9.1 返回值
- 16.9.2 示例
- 16.10 setClassName(string className)
- 16.10.1 参数
- 16.10.2 示例
- 16.11 getMethodName()
- 16.11.1 返回值
- 16.11.2 示例
- 16.12 setMethodName(string MethodName)
- 16.12.1 参数
- 16.12.2 示例
- 16.13 getAgreementCode()
- 16.13.1 返回值
- 16.13.2 示例
- 16.14 setAgreementCode(string agreementCode)
- 16.14.1 参数
- 16.14.2 示例
- 16.15 getRemark()
- 16.15.1 返回值
- 16.15.2 示例
- 16.16 setRemark(string remark)
- 16.16.1 参数
- 16.16.2 示例
- 16.17 etAgreementType()
- 16.17.1 返回值
- 16.17.2 示例
- 16.18 setAgreementType(AgreementType agreementType)
- 16.18.1 参数
- 16.18.2 示例
- 17. NodeMappingHelper (已废弃，计划在3.1.3版本中删除)
- 17.1 getNodeMappingList(NodeMapping nodeMapping)
- 17.1.1 参数
- 17.1.2 返回值
- 17.1.3 示例
- 17.2 createNodeMapping(NodeMapping nodeMapping)
- 17.2.1 参数
- 17.2.2 返回值
- 17.2.3 示例
- 17.3 deleteNodeMapping(string code)
- 17.3.1 参数
- 17.3.2 返回值
- 17.3.3 示例
- 18. MQSender
- 18.1 boolean send(Object o, Map<String,Object> mapParams)
- 19. MQSendHelper
- 19.1 MQSender getSenderByConfigCode(String configCode)
- 19.1.1 示例
- 20. OpenPlantConfig
- 21. OpenPlantSqlExecutor
- 21.1 示例
- 22. 使用帮助
- 22.1 快速开始
- 22.1.1 跨场景通信case
- 22.2 进阶
- 22.2.1 模型方法节点
- 22.2.2 Soap节点
- 22.2.3 Http节点
- 22.2.4 RestWebService节点
- 22.2.5 数据库节点
- 22.2.5.1 基本使用
- 22.2.5.2 查询参数 ： oracle 数据库
- 22.2.5.3 查询参数：mysql 数据库
- 22.2.5.4 查询参数： hana 数据库
- 23. 注意事项

### 基础工具

来源：`http://172.16.9.88/docs/plat3/3.1.15/SDKDoc/%E5%9F%BA%E7%A1%80%E5%B7%A5%E5%85%B7`

- 1. DO语言SDK
- 1.1 基础类型
- 1.1.1 Any 类型
- 1.1.1.1 静态属性
- 1.1.1.2 静态方法
- 1.1.1.2.1 static <T> maxOf(...T)
- 1.1.1.2.2 static Any minOf(...Any)
- 1.1.1.2.3 static Logger log(Class<?> clazz)
- 1.1.1.3 实例方法
- 1.1.1.3.1 Any toString()
- 1.1.1.3.2 Boolean equals(Any)
- 1.1.1.3.3 Boolean inValues(String values ,String splitter)
- 1.1.1.3.4 Boolean subset(String values ,String splitter)
- 1.1.1.3.5 Boolean likeValues(String values ,String splitter)
- 1.1.2 Number 数字类型
- 1.1.2.1 静态属性
- 1.1.2.2 静态方法
- 1.1.2.3 实例方法
- 1.1.2.3.1 整形值 : Int intValue()
- 1.1.2.3.2 浮点型值 :Double doubleValue()
- 1.1.3 浮点型
- 1.1.3.1 父类
- 1.1.3.2 静态属性
- 1.1.3.3 静态方法
- 1.1.3.3.1 Int转换 Double：static Double valueOf(Int)
- 1.1.3.3.2 String转换Number：static Number valueOf(String)
- 1.1.3.4 实例方法
- 1.1.3.4.1 取反 : Double negate()
- 1.1.3.4.2 乘 : Double multi(Double)
- 1.1.3.4.3 除 : Double divide(Double)
- 1.1.3.4.4 取模 : Double mod(Double)
- 1.1.3.4.5 加 : Double plus(Double)
- 1.1.3.4.6 减 : Double minus(Double)
- 1.1.3.4.7 加等 : Double plusAssign(Double)
- 1.1.3.4.8 减等 : Double plusAssign(Double)
- 1.1.3.4.9 比较 : Int compare(Double)
- 1.1.3.4.10 绝对值 : Double abs()
- 1.1.3.4.11 向上取整 : Double ceil()
- 1.1.3.4.12 向下取整 : Double floor()
- 1.1.3.4.13 向“最接近的”整数舍入 : Double roundHalfDown(Int)
- 1.1.3.4.14 幂次方 : Double pow(Double)
- 1.1.3.4.15 随机数 : Double randomNext(Double)
- 1.1.3.4.16 随机数 : Double random(Int,Int)
- 1.1.3.4.17 是否是正无穷 Boolean isPositiveInfinite()
- 1.1.3.4.18 是否是负无穷 Boolean isNegativeInfinite()
- 1.1.3.4.19 是否是非数字 Boolean isNaN()
- 1.1.3.4.20 Number转换String：String toString()
- 1.1.4 Int 类型
- 1.1.4.1 父类
- 1.1.4.2 静态属性
- 1.1.4.3 静态方法
- 1.1.4.3.1 static Int valueOf(String)
- 1.1.4.4 实例方法
- 1.1.4.4.1 取反 : Int negate()
- 1.1.4.4.2 乘 : Int multi(Int)
- 1.1.4.4.3 除 : Int divide(Int)
- 1.1.4.4.4 取模 : Int mod(Int)
- 1.1.4.4.5 加 : Int plus(Int)
- 1.1.4.4.6 减 : Int minus(Int)
- 1.1.4.4.7 幂次方 : Double pow(Int)
- 1.1.4.4.8 比较 : Int compare(Int)
- 1.1.5 Boolean 类型
- 1.1.5.1 静态属性
- 1.1.5.2 静态方法
- 1.1.5.3 实例方法
- 1.1.5.3.1 取反 : Boolean not()
- 1.1.5.3.2 与 : Boolean and(Boolean)
- 1.1.5.3.3 或 : Boolean or(Boolean)
- 1.1.6 Class 类型
- 1.1.6.1 静态属性
- 1.1.6.2 静态方法
- 1.1.6.3 实例方法
- 1.1.6.3.1 判断对象是不是该类的实例 : Boolean isInstance(Any)
- 1.1.7 Compute 类型
- 1.1.7.1 静态属性
- 1.1.7.2 静态方法
- 1.1.7.2.1 Group 数据同步: <E, T, K> void sync(List<E>, Group<T>, func<E,K>(E)K, func<T,K>(T)K, func<E,T>(E)T, func<T,E>(E, T)void)
- 1.1.7.3 实例方法
- 1.1.7.3.1 获取知识表 : List<T> KT<T>()
- 1.1.7.3.2 获取数据组 : Group<T> Group<T>()
- 1.1.7.3.3 获取上下文 : Context context()
- 1.1.7.3.4 相等 : Boolean equals(compute)
- 1.1.7.3.5 获取集成服务入口 : IntegrationDispatcher IntegrationDispatcher()
- 1.1.7.3.6 获取 cplex 入口 : CplexExecutor cplex()
- 1.1.7.3.7 算法入口 : BasicJSPSolver createFJSPSolver()
- 1.1.7.3.8 通过属性字面量名称直接获取值 : Any getPropertyValue(String)
- 1.1.7.3.9 通过属性字面量名称直接获取值 没有则使用默认值 : Any getPropertyOrDefault(String,Any)
- 1.1.7.3.11 获取 Group<T> : 获取数据组 Group<T>
- 1.1.7.3.12 建立某一组值的索引 : RSet<T> indexing(IndexingValue)
- 1.1.7.3.13 构建两个对象之间的联系 : void o2m(Groupt<R>,Groupt<RET>,(RE) -> E,(RE) -> R,(R) -> RE)
- 1.1.7.3.14 获取常量 : String getMessage(String)
- 1.1.7.3.15 获取常量 : String getMessage(String, List)
- 1.1.7.3.15 判断当前对象是否是占位对象 : boolean isPlaceholder() since 3.1.15
- 1.1.8 Context 类型
- 1.1.8.1 寻找 identity 标识符对应的对象 : Any findObj(Int)
- 1.1.8.2 获取知识表 : List<T> KT<T>()
- 1.1.8.3 发送系统消息到外界 : void sendMes(String, String, Int(级别), list(用户code集合))
- 1.1.8.4 发送系统消息到外界 : void sendMes(String, String, Int(级别), list(用户code集合), Boolean(是否进行语音播报))
- 1.1.8.5 获取项目code : String projectCode()
- 1.1.8.6 获取场景code : String sceneCode()
- 1.1.8.7 获取场景Info对象 : SceneInfo getSceneInfo()
- 1.1.8.8 获取场景 info 列表：List<SceneInfo> getSceneInfoList(String projectCode)
- 1.1.8.9 获取当前事务信息：TransactionInformation currentTrans()
- 1.1.8.10 获取 Group<T> : 获取数据组 Group<T> 详情见[模型建模使用文档]
- 1.1.8.11 暂停响应式自动计算 : void pause()
- 1.1.8.12 恢复响应式自动计算 : void resume()
- 1.1.8.13 获取日历管理器 CalendarManager calendarManager()
- 1.1.8.14 获取数据帮助类 DbHelper getDbHelper()
- 1.1.8.15 发送SSE消息 void sendSseMes(String str)
- 1.1.8.16 抽取数据到影子上下文 : ShadowContext shadow(expression)
- 1.1.8.17 通过影子对象查找源对象 : ReactiveObject context.findShadowSource(ShadowObject)
- 1.1.9 数据组（Group<T>）
- 1.1.9.1 静态属性
- 1.1.9.2 静态方法
- 1.1.9.3 实例方法
- 1.1.9.3.1 T add(T t)
- 1.1.9.3.2 List<T> addAll(List<T> list)
- 1.1.9.3.3 void remove(T t)
- 1.1.9.3.4 void remveAll(List<T>)
- 1.1.9.3.5 Boolean removeIf(func(T)boolean predicate)
- 1.1.9.3.6 RSet<T> toRSet()
- 1.1.9.3.7 Int size()
- 1.1.9.3.8 Boolean isEmpty()
- 1.1.9.3.9 Boolean isNotEmpty()
- 1.1.9.3.10 Booealn contain(Any)
- 1.1.9.3.11 T firstOrNull()
- 1.1.9.3.12 T lastOrNull()
- 1.1.9.3.13 Int count(func(T)Boolean predicate)
- 1.1.9.3.14 RSet<T> filter(func(T)Boolean predicate)
- 1.1.9.3.15 void forEach(func(T)void action)
- 1.1.9.3.16 <E> List<E> map(func(T)E mapper)
- 1.1.9.3.17 T find(func(T)Boolean predicate)
- 1.1.9.3.18 计数 : Int count((E) -> void)
- 1.1.9.3.19 取值最小或不存在 : E? minByOrNull((E) -> (R))
- 1.1.9.3.20 取值最大或不存在 : E? maxByOrNull((E) -> (R))
- 1.1.9.3.21 任意匹配 : Boolean any((E) -> (Boolean))
- 1.1.9.3.22 全部匹配 : Boolean all((E) -> (Boolean))
- 1.1.9.3.23 累加 Duration 值 : Duration sumByDuration((E) -> Duration)
- 1.1.9.3.24 累加 Int 值 : Int sumByInt((E) -> Int)
- 1.1.9.3.25 累加 Int 值 : Int sumByDouble((E) -> Double)
- 1.1.9.3.26 表达式排序 : RList<E> sortedBy((E) -> Boolean)
- 1.1.9.3.27 通过表达式去重 : RSet<E> distinctBy(E -> Any)
- 1.1.10 Table 类型
- 1.1.10.1 静态属性
- 1.1.10.2 静态方法
- 1.1.10.3 成员属性
- 1.1.10.4 实例方法
- 1.1.11 Date 类型
- 1.1.11.1 静态属性
- 1.1.11.2 静态方法
- 1.1.11.2.1 取当前时间值 : static Date now()
- 1.1.11.2.2 转换时间 : static String parse(String)
- 1.1.11.2.3 转换时间 : static String parse( String, String)
- 1.1.11.2.4 生成时间 : static Date of(Int,Int,Int,Int,Int)
- 1.1.11.3 实例方法
- 1.1.11.3.1 获取年 : Int year()
- 1.1.11.3.2 获取月 : Int month()
- 1.1.11.3.3 获取指定时间的当天日期 : DateTime atStartOfDay()
- 1.1.11.3.4 是否是有限的 : Boolean isFinite()
- 1.1.11.3.5 加年 : Date plusYears()
- 1.1.11.3.6 加月 : Date plusMonths()
- 1.1.11.3.7 加天 : Date plusDays()
- 1.1.11.3.8 减年 : Date minusYears()
- 1.1.11.3.9 减月 : Date minusMonths()
- 1.1.11.3.10 减天 : Date minusDays()
- 1.1.11.3.11 返回该日期是月份中的第几天 : Int dayOfMonth()
- 1.1.11.3.12 返回该日期是年份中的第几天 : Int dayOfYear()
- 1.1.11.3.13 比较 : Int compare(Date)
- 1.1.11.3.14 转换字符串 : String toString()
- 1.1.11.3.15 格式化为字符串 String format()
- 1.1.11.3.16 格式化为字符串 String format(String)
- 1.1.12 DateTime 类型
- 1.1.12.1 静态属性
- 1.1.12.2 静态方法
- 1.1.12.2.1 取当前时间值 : static DateTime now()
- 1.1.12.2.2 转换时间 : static String parse(String)
- 1.1.12.2.3 转换时间 : static String parse(String,String)
- 1.1.12.2.4 生成时间 : static DateTime of(Int,Int,Int,Int,Int)
- 1.1.12.2.5 生成时间 : static DateTime of(Int,Int,Int,Int,Int,Int)
- 1.1.12.2.6 当前纳秒时间 : static Double nanoTime()
- 1.1.12.3 实例方法
- 1.1.12.3.1 加 : DateTime plus(Duration)
- 1.1.12.3.2 减 : DateTime minus(Duration)
- 1.1.12.3.3 减 : Duration minus(DateTime)
- 1.1.12.3.4 获取年 : Int year()
- 1.1.12.3.5 获取月 : Int month()
- 1.1.12.3.6 获取时 : Int hour()
- 1.1.12.3.7 获取分 : Int minute()
- 1.1.12.3.8 获取秒 : Int second()
- 1.1.12.3.9 获取纳秒 : Int nano()
- 1.1.12.3.10 转换日期Date : Date toDate()
- 1.1.12.3.11 获取指定时间的当天日期 : DateTime atStartOfDay()
- 1.1.12.3.12 获取指定时间的当月第一天 : Date firstDayOfMonth()
- 1.1.12.3.13 获取指定时间的下月第一天 : Date firstDayOfNextMonth()
- 1.1.12.3.14 获取指定时间的当月的最后一天 : Date lastDayOfMonth()
- 1.1.12.3.15 获取当前时间周的具体周几 : Date withWeek(Int)
- 1.1.12.3.16 是否是有限的 : Boolean isFinite()
- 1.1.12.3.17 返回该日期是星期几 : DateTime dayOfWeek()
- 1.1.12.3.18 返回该日期是月份中的第几天 : DateTime dayOfMonth()
- 1.1.12.3.19 返回该日期是年份中的第几天 : DateTime dayOfYear()
- 1.1.12.3.20 加年 : DateTime plusYears()
- 1.1.12.3.21 加月 : DateTime plusMonths()
- 1.1.12.3.22 加周 : DateTime plusWeeks()
- 1.1.12.3.23 加天 : DateTime plusDays()
- 1.1.12.3.24 加小时 : DateTime plusHours()
- 1.1.12.3.25 加分钟 : DateTime plusMinutes()
- 1.1.12.3.26 加秒 : DateTime plusSeconds()
- 1.1.12.3.27 减年 : DateTime minusYears()
- 1.1.12.3.28 减月 : DateTime minusMonths()
- 1.1.12.3.29 减周 : DateTime minusWeeks()
- 1.1.12.3.30 减天 : DateTime minusDays()
- 1.1.12.3.31 减分钟 : DateTime minusMinutes()
- 1.1.12.3.32 减秒 : DateTime minusSeconds()
- 1.1.12.3.33 比较 : Int compare(DateTime)
- 1.1.12.3.34 转换字符串 : String toString()
- 1.1.12.3.35 格式化为字符串 String format()
- 1.1.12.3.36 格式化为字符串 String format(String)
- 1.1.13 Decimal 类型
- 1.1.13.1 静态属性
- 1.1.13.2 静态方法
- 1.1.13.3 实例方法
- 1.1.13.3.1 乘 : Decimal multi(Decimal)
- 1.1.13.3.2 除 : Decimal divide(Decimal)
- 1.1.13.3.3 加 : Decimal plus(Decimal)
- 1.1.13.3.4 减 : Decimal minus(Decimal)
- 1.1.14 Duration 类型
- 1.1.14.1 静态属性
- 1.1.14.2 静态方法
- 1.1.14.2.1 转换为天数 : static duration ofDays()
- 1.1.14.2.2 转换为小时数 : static duration ofHours()
- 1.1.14.2.3 转换为分钟数 : static duration ofMinutes()
- 1.1.14.2.4 转换为秒数 : static duration ofSeconds()
- 1.1.14.3 实例方法
- 1.1.14.3.1 加 : duration plus(duration)
- 1.1.14.3.2 减 : duration minus(duration)
- 1.1.14.3.3 加等 : duration plusAssign(duration)
- 1.1.14.3.4 减等 : duration minusAssign(duration)
- 1.1.14.3.5 乘 : duration multi(Double)
- 1.1.14.3.6 除 : duration divide(Double)
- 1.1.14.3.7 除 : Double divide(duration)
- 1.1.14.3.8 比较 : Int compare(duration)
- 1.1.14.3.9 加天数 : duration plusDays(Double)
- 1.1.14.3.10 加小时 : duration plusHours(Double)
- 1.1.14.3.11 加分钟 : duration plusMinutes(Double)
- 1.1.14.3.12 加秒数 : duration plusSeconds(Double)
- 1.1.14.3.13 加毫秒数 : uration plusMillis(Double)
- 1.1.14.3.14 加纳秒数 : duration plusNanos(Double)
- 1.1.14.3.15 转换至毫秒数（Double类型） : Double toMillis()
- 1.1.14.3.16 转换至秒（Double类型） : Double toSeconds()
- 1.1.14.3.17 转换至分钟（Double类型） : Double toMinutes()
- 1.1.14.3.18 转换至小时（Double类型） : Double toHours()
- 1.1.14.3.19 转换至天（Double类型） : Double toDays()
- 1.1.15 List 类型
- 1.1.15.1 构造器
- 1.1.15.2 静态方法
- 1.1.15.2.1 快速构建并添加 : List<E> of(E) 最多支持5个元素
- 1.1.15.3 实例方法
- 1.1.15.3.1 添加 : void add(E)
- 1.1.15.3.2 获取 : E get(Int)
- 1.1.15.3.3 获取list前n个 : List<E> take(Int)
- 1.1.15.3.4 集合大小 : Int size()
- 1.1.15.3.5 添加所有 : List<E> addAll(List<E>)
- 1.1.15.3.6 最后一个或空 : E lastOrNull()
- 1.1.15.3.7 第一个或空 : E firstOrNull()
- 1.1.15.3.8 获取下标 : Int indexOf(E)
- 1.1.15.3.9 前一个 : E previous(E)
- 1.1.15.3.10 后一个 : E next(E)
- 1.1.15.3.11 去重 : List<E> distinct()
- 1.1.15.3.12 是否为空 : Boolean isEmpty()
- 1.1.15.3.13 移除所有 : Boolean removeAll(List<E>)
- 1.1.15.3.14 移除 : Boolean remove(E)
- 1.1.15.3.15 是否存在 : Boolean contain(E)
- 1.1.15.3.16 清空 : Boolean clear()
- 1.1.15.3.17 反转List : List<E> reverse()
- 1.1.15.3.18 移动 : List<E> move(Int idx,E e)
- 1.1.15.3.19 新建并添加 : List<E> addAndRenew(E e)
- 1.1.15.3.20 获取集合中最大元素或空 : E maxByOrNull()
- 1.1.15.3.21 获取集合中最小元素或空 : E minByOrNull()
- 1.1.15.3.22 表达式匹配任何一个 : Boolean any((E) -> Boolean)
- 1.1.15.3.23 表达式匹配所有 :Boolean all((E) -> Boolean)
- 1.1.15.3.24 通过表达式去重 : List<E> distinctBy(E -> Any)
- 1.1.15.3.25 过滤 : List<E> filter((E) -> Boolean)
- 1.1.15.3.26 查找 : E find((E) -> Boolean)
- 1.1.15.3.27 循环 : void forEach((E) -> void)
- 1.1.15.3.28 分组 : Map<K,List<E>> groupBy((E)->Any)
- 1.1.15.3.29 转换函数 : Any map((E) -> Any)
- 1.1.15.3.30 通过表达式获取最大元素或空 : E maxByOrNull((E) -> R)
- 1.1.15.3.31 通过表达式获取最小元素或空 : E minByOrNull((E) -> R)
- 1.1.15.3.32 满足表达式则移除 : List<E> removeIf((E) -> Boolean)
- 1.1.15.3.33 表达式排序 : List<E> sortedBy((E) -> Boolean)
- 1.1.15.3.34 累加 Duration 值 : Duration sumByDuration((E) -> Duration)
- 1.1.15.3.35 累加 Int 值 : Int sumByInt((E) -> Int)
- 1.1.15.3.36 累加 Double 值 : Double sumByDouble((E) -> Double)
- 1.1.15.3.37 flatMap铺平List : List flatMap()
- 1.1.15.3.38 joinToString : String joinToString(String)
- 1.1.16 Map 类型
- 1.1.16.1 构造器
- 1.1.16.2.1 静态方法
- 1.1.16.2.1 快速生成 : Map of(K,V) 支持最多5对键值对
- 1.1.16.2.2 实例方法
- 1.1.16.2.2.1 放入 : V put(K,V);
- 1.1.16.2.2.2 获取 : V get(K)
- 1.1.16.2.2.3 数量 : Int size()
- 1.1.16.2.2.4 移除 : V remove(K)
- 1.1.16.2.2.5 清除所有 : void clear()
- 1.1.16.2.2.6 是否为空 : Boolean isEmpty()
- 1.1.16.2.2.7 是否有此key : Boolean containsKey(K)
- 1.1.16.2.2.8 替换 : V replace(K,V)
- 1.1.16.2.2.9 替换 : Boolean replace(K,oldV,newV)
- 1.1.16.2.2.10 获取为空时使用默认值 : V getOrDefault(K,V)
- 1.1.16.2.2.11 如果没有则放入 : V putIfAbsent(K,V)
- 1.1.16.2.2.12 key集合 : List<K> keys()
- 1.1.16.2.2.13 value集合 : List<V> values()
- 1.1.16.2.2.14 新建并放入新值 : Map<K,V> putAndRenew()
- 1.1.16.2.2.15 V computeIfAbsent(K, (K)->V)
- 1.1.16.2.2.16 V computeIfPresent(K, (K,V)->R)
- 1.1.16.2.2.17 循环 : V forEach((K,V)->void)
- 1.1.16.2.2.18 最大键值对 : Map maxByOrNull((K,V)->R)
- 1.1.16.2.2.19 最小键值对 : Map minByOrNull((K,V)->R)
- 1.1.16.2.2.20 过滤 : Map filter((K,V)-> Boolean)
- 1.1.17 RSet 类型
- 1.1.17.1 构造器
- 1.1.17.2 实例方法
- 1.1.17.2.1 添加 : void add(E)
- 1.1.17.2.2 添加所有 : void addAll(Collection<E>)
- 1.1.17.2.3 第一个或空 : E firstOrNull()
- 1.1.17.2.4 移除 : Boolean remove(E)
- 1.1.17.2.5 移除特定集合 : Boolean removeAll(List<E>)
- 1.1.17.2.6 满足表达式则移除 : Boolean removeIf((E) -> Boolean)
- 1.1.17.2.7 清空 : Boolean clear()
- 1.1.17.2.8 获取当前序列的大小 : Int size()
- 1.1.17.2.9 是否此集合为空 : Boolean isEmpty()
- 1.1.17.2.10 是否此集合不为空 : Boolean isNotEmpty()
- 1.1.17.2.11 是否存在 : Boolean contain(E)
- 1.1.17.2.12 是否存在 : Boolean uncontain(E)
- 1.1.17.2.13 转换成List : List<E> toList()
- 1.1.17.2.14 循环 : void forEach((E) -> void)
- 1.1.17.2.15 查找 : E find((E) -> Boolean)
- 1.1.17.2.16 过滤 : RSet<E> filter((E) -> Boolean)
- 1.1.17.2.17 计数 : Int count((E) -> void)
- 1.1.17.2.18 转换函数 : List<R> map((E) -> (R))
- 1.1.17.2.19 取值最小或不存在 : E? minByOrNull((E) -> (R))
- 1.1.17.2.20 取值最大或不存在 : E? maxByOrNull((E) -> (R))
- 1.1.17.2.21 任意匹配 : Boolean any((E) -> (Boolean))
- 1.1.17.2.22 全部匹配 : Boolean all((E) -> (Boolean))
- 1.1.17.2.23 满足表达式则移除 : Boolean removeIf((E) -> Boolean)
- 1.1.17.2.24 累加 Duration 值 : Duration sumByDuration((E) -> Duration)
- 1.1.17.2.25 累加 Int 值 : Int sumByInt((E) -> Int)
- 1.1.17.2.26 累加 Int 值 : Int sumByDouble((E) -> Double)
- 1.1.17.2.27 分组 : Map<R,List<E>> groupBy((E)->R)
- 1.1.17.2.28 表达式排序 : RList<E> sortedBy((E) -> Boolean)
- 1.1.17.2.29 通过表达式去重 : RSet<E> distinctBy(E -> Any)
- 1.1.18 RList 类型
- 1.1.18.1 在List方法基础上，删除distinct、addAndRenew方法
- 1.1.18.2 通过表达式去重 : RList<E> distinctBy(E -> Any)
- 1.1.18.3 过滤 : RList<E> filter((E) -> Boolean)
- 1.1.18.4 转成 Rset: RSet toRSet()
- 1.1.19 RSequence 类型
- 1.1.19.1 构造器
- 1.1.19.2 实例方法
- 1.1.19.2.1 添加 : Boolean add(E)
- 1.1.19.2.2 添加所有 : void addAll(Collection<E>)
- 1.1.19.2.3 第一个或空 : E firstOrNull()
- 1.1.19.2.4 最后一个或空 : E lastOrNull()
- 1.1.19.2.5 获取下标 : Int indexOf(E)
- 1.1.19.2.6 移除 : Boolean remove(E)
- 1.1.19.2.7 移除特定集合 : Boolean removeAll(List<E>)
- 1.1.19.2.8 满足表达式则移除 : Boolean removeIf((E) -> Boolean)
- 1.1.19.2.9 前一个 : E previous(E)
- 1.1.19.2.10 后一个 : E next(E)
- 1.1.19.2.11 移动到E2之前（如果E1不存在默认实现添加） : Boolean moveBefore(E1,E2)
- 1.1.19.2.12 移动到E2之后（如果E1不存在默认实现添加） : Boolean moveAfter(E1,E2)
- 1.1.19.2.13 将元素移动到头部位置（默认实现添加） : Boolean moveFirst(E1)
- 1.1.19.2.14 将元素移动到尾部位置（默认实现添加） : Boolean moveLast(E1,E2)
- 1.1.19.2.15 获取当前序列的大小 : Int size()
- 1.1.19.2.16 是否为空 : Boolean isEmpty()
- 1.1.19.2.17 是否不为空 : Boolean isNotEmpty()
- 1.1.19.2.18 根据下标获取元素 : E get(Int)
- 1.1.19.2.19 子序列： RSequence<E> subRSequence(Int, Int)
- 1.1.19.2.20 循环 : void forEach((E) -> void)
- 1.1.19.2.21 查找 : E find((E) -> Boolean)
- 1.1.19.2.22 过滤 : RSequence<E> filter((E) -> Boolean)
- 1.1.19.2.23 转成 Rset: RSet toRSet()
- 1.1.19.2.24 取值最小或不存在 : E? minByOrNull((E) -> (R))
- 1.1.19.2.25 取值最大或不存在 : E? maxByOrNull((E) -> (R))java
- 1.1.19.2.26 任意匹配 : Boolean any((E) -> (Boolean))
- 1.1.19.2.27 全部匹配 : Boolean all((E) -> (Boolean))
- 1.1.19.2.28 满足表达式则移除 : List<E> removeIf((E) -> Boolean)
- 1.1.19.2.29 累加 Duration 值 : Duration sumByDuration((E) -> Duration)
- 1.1.19.2.30 累加 Int 值 : Int sumByInt((E) -> Int)
- 1.1.19.2.31 累加 Int 值 : Int sumByDouble((E) -> Double)
- 1.1.19.2.32 分组 : Map<R,List<E>> groupBy((E) -> R)
- 1.1.19.2.33 表达式排序 : RSet<E> sortedBy((E) -> Boolean)
- 1.1.19.2.34 通过表达式去重 : RSequence<E> distinctBy(E -> Any)
- 1.1.20 String 类型
- 1.1.20.1 静态属性
- 1.1.20.2 静态方法
- 1.1.20.3 实例方法
- 1.1.20.4 加 : String plus(String)
- 1.1.20.5 加 : String plus(Any)
- 1.1.20.6 加等 : String plusAssign(String)
- 1.1.20.7 包含 : Boolean contains(String)
- 1.1.20.8 分割 : List<String> split(String regex)
- 1.1.20.9 转换小写字符串 : String toLowerCase ()
- 1.1.20.10 转换大写字符串 : String toUpperCase ()
- 1.1.20.11 是否为空 : Boolean isEmpty()
- 1.1.20.12 长度 : Number length()
- 1.1.20.13 截取字符串 : String substring(Number)
- 1.1.20.14 截取字符串 : String substring(Number,Number)
- 1.1.20.15 替换字符串 : String replace(String)
- 1.1.20.16 转换整型数字 : Int toInt()
- 1.1.20.17 转换浮点数字 : Double toDouble()
- 1.1.20.18 转换Boolean : Boolean toBoolean()
- 1.1.20.19 转换格式 : String format(String)
- 1.1.20.20 转换duration : duration toDuration()
- 1.1.20.21 移除后缀 : String removeSuffix()
- 1.1.20.22 移除前缀 : String removePrefix()
- 1.1.21 ShowContext 影子上下文
- 1.2 工具类
- 1.2.1 Cell
- 1.2.1.1 静态属性
- 1.2.1.2 公开属性
- 1.2.1.2.1 Number rowIndex
- 1.2.1.2.2 Any value
- 1.2.1.2.3 Int column
- 1.2.1.2.4 String column
- 1.2.1.2.5 String originType
- 1.2.1.2.6 class type
- 1.2.1.3 静态方法
- 1.2.1.4 实例方法
- 1.2.2 ExcelReader
- 1.2.2.1 静态方法
- 1.2.2.2 构造函数
- 1.2.2.3 实例方法
- 1.2.2.3.1 List readSheet(String)
- 1.2.2.3.2 Map read(String)
- 1.2.3 InputStream
- 1.2.3.1 静态属性
- 1.2.3.2 静态方法
- 1.2.3.3 实例方法
- 1.2.4 EncryptUtils
- 1.2.4.1 静态属性
- 1.2.4.2 静态方法
- 1.2.4.2.1 String toMd5Hex(String msg)
- 1.2.4.2.2 String base64Encode(String msg)
- 1.2.4.2.3 String base64Decode(String base64)
- 1.2.4.3 实例方法
- 1.2.5 JsonUtils
- 1.2.5.1 静态属性
- 1.2.5.2 静态方法
- 1.2.5.2.1 Map JsonUtils.toMap(String json)
- 1.2.5.2.2 List JsonUtils.toList(String json)
- 1.2.5.2.3 String JsonUtils.toJsonString(Any obj)
- 1.2.5.2.4 List JsonUtils.xmlToList(String xml)
- 1.2.5.2.5 Map JsonUtils.xmlToMap(String json)
- 1.2.5.2.6 实例方法
- 1.2.6 Logger
- 1.2.6.1 静态属性
- 1.2.6.2 静态方法
- 1.2.6.2.1 void info(String);
- 1.2.6.2.2 debug日志 : void debug(String);
- 1.2.6.2.3 warn日志 : void warn(String)
- 1.2.6.2.4 error日志 : void error(String)
- 1.2.6.3 实例方法
- 1.2.7 UUID 工具类
- 1.2.7.1 静态属性
- 1.2.7.2 静态方法
- 1.2.7.2.1 String generateUUID() 生成随机 UUID
- 1.2.7.2.2 String generateUUID() 生成随机 UUID
- 1.2.7.3 实例方法
- 1.2.8 Platforms
- 1.2.8.1 静态方法
- 1.2.8.2 构造函数
- 1.2.8.3 实例方法
- 1.2.8.3.1 UserService getUserService()
- 1.2.8.3.2 OrganizationService getOrganizationService()
- 1.2.9 User
- 1.2.9.1 静态方法
- 1.2.9.2 构造函数
- 1.2.9.3 实例方法
- 1.2.9.3.1 String getNo()
- 1.2.9.3.1 String getUsername()
- 1.2.9.3.1 String getOrganization()
- 1.2.10 Organization
- 1.2.10.1 静态方法
- 1.2.10.2 构造函数
- 1.2.10.3 实例方法
- 1.2.10.3.1 String getCode()
- 1.2.10.3.1 String getName()
- 1.2.10.3.1 String getParent()
- 1.2.11 UserService
- 1.2.11.1 静态方法
- 1.2.11.1.1 List getAll() 获取平台所有用户
- 1.2.11.2 构造函数
- 1.2.11.3 实例方法
- 1.2.12 OrganizationService
- 1.2.12.1 静态方法
- 1.2.12.1.1 List getAll() 获取平台所有组织架构
- 1.2.12.2 构造函数
- 1.2.12.3 实例方法
- 1.3 控制语句
- 1.3.1 while 循环
- 1.3.2 for 循环
- 1.3.3 if else
- 1.3.4 break
- 1.3.5 continue
- 1.3.6 try catch
- 1.3.6.1 用法
- 1.3.6.2 案例
- 1.3.7 throw
- 1.3.7.1 用法
- 1.3.7.2 案例
- 1.3.8 transient
- 1.3.8.1 用法
- 1.3.8.2 案例
- 1.3.9 TextFormat 字符串格式
- 1.3.9.1 构造函数
- 1.3.10 NumberFormat 数字格式
- 1.3.10.1 构造函数
- 1.3.11 VarChar2Format 字符串格式
- 1.3.11.1 构造函数
- 1.3.12 VarCharFormat 字符串格式
- 1.3.12.1 构造函数
- 1.3.13 tringRangeTemplate 字符处理
- 1.3.13.1 构造函数
- 1.3.13.2 实例方法
- 1.3.13.2.1 添加字符处理片段 addSegment(String key, int start, int end, Format format, boolean required)
- 1.3.13.2.2 添加字符处理片段 addSegment(String key, int start, int end, boolean required)
- 1.3.13.2.3 序列化成字符串 String encode(Map<String, Object> dataMap)
- 1.3.13.2.4 反序列化字符串 Map<String, Object> decode(String encodedStr)

### 节点数据同步的推荐写法

来源：`http://172.16.9.88/docs/plat3/skills/ModelDevelopment/NodesAsyncBestWay`

- 1. 背景说明
- 2. 原理与解释
- 3. 推荐方案
- 4. 总结

