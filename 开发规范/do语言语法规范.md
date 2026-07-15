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

后续每次发生以下情况时，在本表追加一条记录：

- 用户明确指出生成的DO代码语法不正确；
- 平台编译结果证明某项推定错误；
- 新代码显示DO语言存在尚未记录的特殊语法；
- 原有写法虽然类似Java/Groovy/Kotlin，但DO实际语义不同；
- 平台升级导致语法或API发生变化。
