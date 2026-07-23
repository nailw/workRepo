# Assignment 时间计算与循环依赖排查指南

> 适用模型：`ASJW-京卫作业排程`  
> 分析对象：`Assignment`、`EarliestAssignment`、`LatestAssignment`  
> 整理日期：2026-07-23  
> 说明：本文基于模型设计器中当前可见的属性、自动计算方法、调用链和方法实现整理。模型后续变更时，应以当前版本源码为准。

## 1. 先建立统一的时间轴

一个 Assignment 的完整占用区间分为三段：

```text
startTime
  │
  ├── prefixStartTime ── prefixDuration ── prefixEndTime
  │
  ├── processStartTime ─ processDuration ─ processEndTime
  │
  └── suffixStartTime ── suffixDuration ── suffixEndTime
                                                    │
                                                  endTime
```

含义如下：

| 时间 | 含义 | 主要来源 |
|---|---|---|
| `prefixStartTime` | 前处理开始 | ASAP 取 `earliestAssignment.prefixES`；JIT 取 `latestAssignment.prefixLS` |
| `prefixEndTime` | 前处理结束 | 前处理开始加前处理时长，并按设备日历修正 |
| `processStartTime` | 正式生产开始 | 通常接在前处理结束之后，并按设备可用时间修正 |
| `processEndTime` | 正式生产结束 | 生产开始加生产时长，并按设备日历修正 |
| `suffixStartTime` | 后处理开始 | 通常接在生产结束之后，并按设备可用时间修正 |
| `suffixEndTime` | 后处理结束 | 后处理开始加后处理时长，并按设备日历修正 |
| `startTime` | Assignment 总开始 | 当前实现首先取 `prefixStartTime`，部分工序有特殊修正 |
| `endTime` | Assignment 总结束 | 通常取最早/最晚边界的总结束时间，部分工序有特殊修正 |
| `duration` | Assignment 总时长 | `TimeCalc.durationBetween(startTime, endTime)` |

## 2. 模式选择：ASAP 与 JIT

`Assignment.asapMode` 决定采用哪条计算链。

### 2.1 ASAP：从左向右正推

ASAP 的目标是“在所有约束满足的情况下尽早开始”。

```mermaid
flowchart LR
    A["上游工艺结束 / 同设备前任务结束 / 系统时间"] --> B["possibleStartES"]
    B --> C["prefixES"]
    C -->|"+ prefixDuration，按日历修正"| D["prefixEE"]
    D -->|"必要时 nextAvailable"| E["processES"]
    E -->|"+ processDuration，按日历修正"| F["processEE"]
    F -->|"必要时 nextAvailable"| G["suffixES"]
    G -->|"+ suffixDuration，按日历修正"| H["suffixEE"]
    H --> I["endEE"]
    C --> J["Assignment.startTime / startTimeES"]
    I --> K["Assignment.endTime / endTimeEE"]
```

主链的已核对实现如下：

1. `Assignment.startTimeES = earliestAssignment.startES`
2. `EarliestAssignment.startES = prefixES`
3. `EarliestAssignment.prefixES`
   - 如果 `forceDragFlag=true`，直接使用 `forceStartTime`。
   - 否则从 `possibleStartES` 起算。
   - 如果时间落入“订单产能占用”等不可用区间，则移动到占用结束。
   - 根据设备的 `prefix/process/suffix CanBeFit` 和 `CanBeSplit` 配置，调用：
     - `resource.nextAvailable(...)`
     - `resource.calcStartForNotSplitByStartPlusDuration(...)`
   - 特定上下游车间场景还会与上一工序 `endTime` 取最大值。
4. `prefixEE = timePlusDuration(prefixES, prefixDuration)`
   - 前处理不可利用停机但可跨停机时，使用 `resource.nextFit(...)`。
5. `processES = prefixEE`
   - 生产不可利用停机时，使用 `resource.nextAvailable(prefixEE)`。
6. `processEE = timePlusDuration(processES, processDuration)`
   - 生产不可利用停机但可跨停机时，使用 `resource.nextFit(...)`。
7. `suffixES = processEE`
   - 后处理时长为零时直接返回。
   - 后处理不可利用停机时，使用 `resource.nextAvailable(processEE)`。
8. `suffixEE = timePlusDuration(suffixES, suffixDuration)`
   - 后处理不可利用停机但可跨停机时，使用 `resource.nextFit(...)`。
9. `endEE = suffixEE`。

`possibleStartES` 是正推链的关键入口。其候选值一般来自：

- 同设备上一 Assignment 的结束时间；
- 工艺上游 PieceStep/Assignment 的结束时间；
- 设备最早可用时间；
- 当前 `systemTime`；
- 设备关联、副资源、车间互斥等其他最早边界。

最终应理解为：

```text
possibleStartES = max(
  工艺上游约束,
  同设备前任务约束,
  设备可用约束,
  副资源约束,
  车间互斥约束,
  系统时间或强制时间
)
```

### 2.2 JIT：从右向左反推

JIT 的目标是“在不违反交期及后续约束的情况下尽量晚开始”。

```mermaid
flowchart RL
    A["交期 / 下游最晚开始 / 同设备后任务开始"] --> B["possibleEndLE"]
    B --> C["suffixLE"]
    C -->|"- suffixDuration，按日历反推"| D["suffixLS"]
    D --> E["processLE"]
    E -->|"- processDuration，按日历反推"| F["processLS"]
    F --> G["prefixLE"]
    G -->|"- prefixDuration，按日历反推"| H["prefixLS"]
    C --> I["endLE"]
    H --> J["startLS"]
    J --> K["Assignment.startTimeLS"]
```

已核对的主要反推规则：

- `Assignment.startTimeLS`
  - 若 `asapMode=true`，直接返回 `startTimeES`；
  - 否则返回 `latestAssignment.startLS`。
- `suffixLE` 从 `possibleEndLE` 出发，并根据设备是否允许利用或跨越停机向前修正。
- `suffixLS = suffixLE - suffixDuration`；需要跨日历时调用 `resource.previousFit(...)`。
- `processLS = processLE - processDuration`；需要跨日历时调用 `resource.previousFit(...)`。
- `prefixLE` 通常接在 `processLS`；前处理不可利用停机时调用 `resource.previousAvailable(...)`。
- `endLE = suffixLE`。

概念上：

```text
possibleEndLE = min(
  订单交期,
  工艺下游最晚边界,
  同设备后任务开始边界,
  副资源最晚边界,
  其他最晚约束
)
```

## 3. 时长是如何得到的

### 3.1 总时长

当前实现：

```text
duration = TimeCalc.durationBetween(startTime, endTime)
```

它不是简单的三个 Duration 相加结果，因为开始、结束之间可能跨停机、班次、不可用区间或特殊日历。

### 3.2 前处理时长 `prefixDuration`

主要来源：

1. 同设备上一 Assignment 与当前 Assignment 产品不同：
   - 按前产品、后产品、资源组、资源查询 `KTPrefixDuration`。
2. 当前 Assignment 是设备首任务：
   - 与设备最后一条反馈任务比较产品，再查询 `KTPrefixDuration`。
3. 人工指定超期清场：
   - 使用 `KTChangeOverCalculator.changeOverDuration`。
4. `isOverdueClean=true`：
   - 使用 `overCalculatorPrefixDuration` 覆盖前面的值。

### 3.3 生产时长 `processDuration`

当前模型中包含按工序定制的规则。例如：

- `Zhongchu` 根据前工序、资源名称，从 `KTIntermediateStorage` 或 `KTWeightCheckConfig` 取时长；
- `Jianlou` 且工序号大于 5 时，从 `KTSingleResourceSpeedExtend.durationForSinglePiece` 取时长；
- 其他场景可能保持 `Duration.ZERO` 或由子类/其他规则覆盖。

因此，排查生产时长时必须同时检查 Assignment 的实际子类（如 `SingleAssignment`、`BatchAssignment`）以及项目定制规则。

### 3.4 后处理时长 `suffixDuration`

主要来源：

1. 同设备下一 Assignment 存在，且前后产品不同；
2. 按前产品、后产品、资源组、资源查询 `KTSuffixDuration`；
3. 再使用 `earliestAssignment.suffixDuration` 覆盖或补充。

## 4. 影响时间的关系

时间计算不是只依赖当前 Assignment。至少需要关注以下关系：

| 关系 | 影响 |
|---|---|
| `previous` / `next` | 同一资源上的前后 Assignment |
| `previousPieceSteps` / `nextPieceSteps` | 工艺路线的前后工序 |
| `pieceStep` | 当前任务对应的工艺步骤 |
| `resource` | 日历、停机区间、最早可用时间、可利用/可跨越配置 |
| `earliestAssignment` | ASAP 的 ES/EE 时间边界 |
| `latestAssignment` | JIT 的 LS/LE 时间边界 |
| `viceAssignments` / `vice` | 副资源占用 |
| `batch` / `batchFlag` | 批任务共用开始、结束时间 |
| `previousDeviceAssociationAss` | 设备联动约束 |
| `forceDragFlag` / `forceStartTime` | 人工强制开始时间 |
| `dueTime` | JIT 反推的交期边界 |

## 5. 为什么容易出现循环

自动计算属性之间形成了一个依赖图。只要图中存在从某属性出发又回到它自身的路径，就可能触发循环计算。

典型闭环一：同设备前后关系成环。

```mermaid
flowchart LR
    A["A.endTime"] --> B["B.previous = A"]
    B --> C["B.possibleStartES"]
    C --> D["B.startTime / endTime"]
    D --> E["A.next = B"]
    E --> F["A.suffixDuration 或最晚边界"]
    F --> A
```

典型闭环二：工艺关系和资源顺序交叉。

```text
A 的工艺前序是 B
B 在设备上的 previous 又是 A

A.start 依赖 B.end
B.start 依赖 A.end
=> A.start -> B.end -> B.start -> A.end -> A.start
```

典型闭环三：时长与时间互相依赖。

```text
startTime -> endTime -> duration -> prefix/process/suffixDuration -> startTime
```

典型闭环四：ASAP 与 JIT 边界串线。

```text
earliestAssignment.* -> Assignment.*
Assignment.* -> latestAssignment.*
latestAssignment.* -> Assignment.*
```

典型闭环五：批任务。

```text
Assignment.startTime -> batchStartTime
batchStartTime -> 批内成员 startTime 的聚合
聚合又包含当前 Assignment.startTime
```

## 6. 循环出现后的标准排查流程

### 第一步：固定故障对象和入口属性

记录以下信息，不要只记录“Assignment 循环”：

- `assignmentId`
- 实际类：`SingleAssignment`、`BatchAssignment` 或其他子类
- `pieceId`、`sequenceNr`、`operationId`
- `resourceId`
- `asapMode`
- 首个报循环的属性或方法，例如 `calcPrefixES`
- 完整循环栈或调用链

首个报错属性不一定是根因，但它是构建闭环的入口。

### 第二步：在调用链中找“第一个重复节点”

在模型设计器底部“调用链”中，从报错属性向上展开。

例如：

```text
Assignment.startTime
→ Assignment.prefixStartTime
→ EarliestAssignment.prefixES
→ EarliestAssignment.possibleStartES
→ Assignment.previous.endTime
→ Assignment.suffixEndTime
→ ...
→ Assignment.startTime   ← 第一次重复
```

截取“第一次出现”到“第二次出现”之间的节点，这一段就是最小循环。

不要一开始就分析整张调用图。先得到最小闭环，再向外检查触发条件。

### 第三步：把依赖分成四类

给最小闭环中的每条边标记类型：

1. 时间段内部：`prefix → process → suffix`
2. 同设备顺序：`previous / next`
3. 工艺顺序：`previousPieceSteps / nextPieceSteps`
4. 外部约束：设备日历、副资源、批次、交期、强制时间

若闭环同时包含第 2 类和第 3 类，优先检查“工艺顺序与设备顺序互相反向”。

### 第四步：检查关系数据是否成环

对当前 Assignment 沿以下关系分别遍历，并记录访问过的 `assignmentId`：

```text
previous 链：current -> previous -> previous -> ...
next 链：current -> next -> next -> ...
工艺前序链：current.pieceStep -> previousPieceSteps -> ...
工艺后序链：current.pieceStep -> nextPieceSteps -> ...
```

应满足：

- `previous.next == current`
- `next.previous == current`
- `previous != current`
- `next != current`
- 一条链中同一 `assignmentId` 不能重复
- 设备顺序不能和工艺硬约束形成反向闭环

建议临时输出：

```text
assignmentId,
previous?.assignmentId,
next?.assignmentId,
pieceStep.sequenceNr,
previousPieceSteps.ids,
nextPieceSteps.ids,
resourceId,
sequenceOnResource
```

### 第五步：检查计算属性是否“读回自己”

重点搜索：

- `calcXxx()` 直接读取 `this.xxx`
- `calcXxx()` 读取另一个属性，而另一个属性最终又读取 `xxx`
- 时长计算读取开始/结束时间，同时开始/结束时间又读取该时长
- `EarliestAssignment` 读取 `owner` 的 ASAP 时间，而 `owner` 又读取 `EarliestAssignment`
- `LatestAssignment` 与 `EarliestAssignment` 互相引用

最危险的模式：

```text
calcA() -> this.b
calcB() -> this.a
```

以及跨对象版本：

```text
Assignment.a -> earliestAssignment.b
EarliestAssignment.b -> owner.a
```

### 第六步：检查空值兜底是否掩盖了循环

`DateTime.MAX`、`DateTime.MIN`、`systemTime`、`Duration.ZERO` 只能处理“没有值”，不能切断已经建立的依赖边。

例如：

```text
return previous?.endTime ?? systemTime
```

只要 `previous` 不为空，就仍会进入 `previous.endTime` 的完整计算链。

### 第七步：临时切断一条边验证根因

选择最可疑且业务优先级最低的一条依赖，临时改成固定输入或已缓存的基础属性。例如：

- 暂时不读 `next`，只用交期；
- 暂时不读工艺上游，使用 `systemTime`；
- 暂时把换型时长设为 `Duration.ZERO`；
- 暂时关闭副资源约束；
- 暂时将批任务拆成单任务验证。

如果循环消失，说明被切断的边位于闭环上。之后应重新设计依赖方向，而不是保留固定值。

### 第八步：确认正确的单向依赖方向

建议保持：

```text
ASAP：上游/前任务/资源日历 -> 当前 Assignment
JIT ：下游/后任务/交期     -> 当前 Assignment
```

避免：

```text
ASAP 当前任务同时反向读取后任务
JIT 当前任务同时正向读取前任务
同一个时间属性同时参与 ES 正推和 LS 反推
```

## 7. 建议增加的诊断日志

在每个关键计算方法入口输出一个结构化日志：

```text
calc=<方法名>
assignmentId=<任务>
mode=<ASAP/JIT>
resourceId=<设备>
previous=<前任务>
next=<后任务>
piecePrevious=<工艺前序>
pieceNext=<工艺后序>
input=<所有输入时间>
result=<结果>
```

为防止日志自身过多，建议增加一次计算上下文 ID，并维护访问栈：

```text
contextId=...
stack=[Assignment:A.startTime, Earliest:A.prefixES, Assignment:B.endTime]
```

进入计算前：

1. 若当前节点已在栈中，立即输出完整栈；
2. 抛出包含最小闭环的明确异常；
3. 不继续递归到栈溢出。

节点键建议使用：

```text
<class>:<assignmentId>:<property>
```

## 8. 当前版本应优先复核的实现

以下内容来自本次对模型设计器中当前方法实现的核对，建议代码评审时优先确认。

### 8.1 恒真条件

`calcPrefixStartTime` 中存在：

```text
if (operationId != "Jianlou" || operationId != "Baocaidayin")
```

该条件对任何单一 `operationId` 都为真，因为一个值不可能同时等于两个不同字符串。若原意是“两者都不是”，应为：

```text
operationId != "Jianlou" && operationId != "Baocaidayin"
```

这会导致后面的 `else if` 特殊分支无法进入，也可能把依赖引向本不应执行的通用链路。

### 8.2 `calcSuffixEndTime` 返回值疑似不匹配

当前可见实现为：

```text
if (asapMode) return startTimeES
return latestAssignment?.startLS
```

但属性名和描述是“后处理结束时间”。从正常时间轴看，更符合语义的候选应是 `suffixEE/suffixLE` 或 `endEE/endLE`。需要确认这是粘贴错误、历史遗留，还是特殊业务定义。

此处尤其容易制造：

```text
suffixEndTime -> startTimeES -> prefixES -> ... -> suffixEndTime
```

### 8.3 JIT 方法应逐一复核“读自己”

反推方法的正确方向通常是：

```text
processLE -> processLS
suffixLE -> suffixLS
processLS -> prefixLE
prefixLE -> prefixLS
```

若 `calcProcessLE` 中读取 `this.processLE`，会直接自循环。应在代码评审中逐个确认：

- `calcPrefixLS`
- `calcPrefixLE`
- `calcProcessLS`
- `calcProcessLE`
- `calcSuffixLS`
- `calcSuffixLE`
- `calcStartLS`
- `calcEndLE`

### 8.4 特殊工序代码的空值和关系访问

`Zhongchu`、`Jianlou`、`Baocaidayin` 等分支中存在较深的关系访问，例如：

```text
previousPieceSteps.firstOrNull()
  .assignments.firstOrNull()
  .resource.resourceName
```

这些访问既可能空指针，也可能跨入另一条 Assignment 时间链。建议先保存中间对象并记录 ID，再读取其时间属性。

## 9. 快速排查清单

- [ ] 确认 `assignmentId`、子类、资源、工序和 ASAP/JIT 模式
- [ ] 从调用链截取首个重复节点形成的最小闭环
- [ ] 检查 `previous/next` 是否自指、互指或重复
- [ ] 检查工艺前后序是否成环
- [ ] 检查设备顺序是否与工艺顺序反向
- [ ] 检查批次聚合是否包含自身
- [ ] 检查 ES 正推是否读取后任务
- [ ] 检查 LS 反推是否读取前任务
- [ ] 检查时长是否反向读取开始/结束时间
- [ ] 检查 `EarliestAssignment.owner` 与 `Assignment.earliestAssignment` 的回边
- [ ] 检查 `LatestAssignment.owner` 与 `Assignment.latestAssignment` 的回边
- [ ] 检查 `calcXxx()` 是否直接读取 `this.xxx`
- [ ] 检查设备日历的 `nextAvailable/nextFit/previousFit` 输入是否有限
- [ ] 临时切断一条边验证闭环
- [ ] 修复后重新检查完整调用图，而不仅是原报错属性

## 10. 推荐的长期治理

1. 将 ES 正推和 LS 反推拆成两个明确阶段，避免同时自动触发。
2. 对关系图在排程前做拓扑检查，提前拒绝环。
3. 区分基础输入、派生时间和展示时间，禁止展示属性反向参与核心计算。
4. 为自动计算引擎增加访问栈和最小闭环报告。
5. 为每个时间方法建立依赖声明和单元测试。
6. 对关键不变量做自动校验：

```text
prefixStart <= prefixEnd
prefixEnd <= processStart
processStart <= processEnd
processEnd <= suffixStart
suffixStart <= suffixEnd
startTime <= endTime
startTimeES <= startTimeLS（存在可行窗口时）
endTimeEE <= endTimeLE（存在可行窗口时）
```

7. 对特殊工序分支进行独立测试，不要让项目定制逻辑散落在通用时间主链中。


