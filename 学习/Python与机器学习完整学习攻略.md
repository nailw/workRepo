# Python + 机器学习完整学习攻略

> 适合零基础到能够独立完成机器学习项目的人。建议学习周期为 4～6 个月，每周投入 10～15 小时；如果已有编程基础，可压缩到 2～3 个月。

## 一、先看最终目标

学完后，你应该能够独立完成以下闭环：

1. 把业务问题转换成分类、回归、聚类、排序或时间序列问题。
2. 使用 Python、NumPy、Pandas 清洗和分析真实数据。
3. 用可视化和统计方法发现数据规律与异常。
4. 正确划分训练集、验证集和测试集，避免数据泄漏。
5. 使用 scikit-learn 建立基线模型、调参并选择合理指标。
6. 理解常用模型的原理、适用条件和局限。
7. 使用 PyTorch 构建基础神经网络。
8. 保存模型，通过 FastAPI 提供预测接口。
9. 编写可复现的项目代码、测试、说明文档和实验记录。
10. 解释模型输出，监控上线后的数据漂移和效果衰减。

机器学习项目的标准流程：

```text
业务目标 → 数据定义 → 数据检查 → 探索分析 → 特征工程
        → 基线模型 → 交叉验证 → 调参与比较 → 误差分析
        → 最终测试 → 部署 → 监控 → 迭代
```

## 二、学习路线总览

| 阶段 | 建议时间 | 核心内容 | 阶段成果 |
|---|---:|---|---|
| 0. 环境准备 | 1～2 天 | Python、虚拟环境、Jupyter、Git | 可运行项目模板 |
| 1. Python 基础 | 2～3 周 | 语法、函数、类、异常、文件、模块 | 命令行数据处理工具 |
| 2. 数据分析 | 2～3 周 | NumPy、Pandas、Matplotlib、Seaborn | 一份完整 EDA 报告 |
| 3. 数学与统计 | 2～4 周，可并行 | 线代、微积分、概率统计、优化 | 能解释损失函数与评估指标 |
| 4. 经典机器学习 | 4～6 周 | 回归、分类、树模型、聚类、降维 | 结构化数据预测项目 |
| 5. 深度学习 | 4～6 周 | PyTorch、MLP、CNN、序列与注意力 | 图像或文本项目 |
| 6. 工程化 | 2～4 周 | Pipeline、测试、API、容器、监控 | 可部署预测服务 |
| 7. 方向深入 | 长期 | NLP、CV、时序、推荐、LLM 等 | 领域作品集 |

不要等数学全部学完才写模型。推荐“项目驱动 + 按需补数学”：每学一个算法，理解它解决什么问题、损失函数是什么、如何验证、什么时候会失败。

## 三、环境准备

### 3.1 推荐工具

- Python 3.11 或 3.12：兼容性通常较好；具体项目以依赖要求为准。
- VS Code 或 PyCharm：写正式代码。
- JupyterLab：探索数据和展示实验。
- Git：保存版本和实验改动。
- `venv`、Conda 或 uv：三选一管理环境，不要把包全部装进系统 Python。

使用标准 `venv`：

```bash
python -m venv .venv

# Windows PowerShell
.venv\Scripts\Activate.ps1

# macOS / Linux
source .venv/bin/activate

python -m pip install --upgrade pip
pip install numpy pandas matplotlib seaborn scikit-learn jupyterlab pytest ruff
```

深度学习阶段再安装 PyTorch，并根据电脑系统、是否有 NVIDIA GPU 选择对应安装方式。不要在第一天安装几十个暂时用不到的库。

### 3.2 推荐项目结构

```text
ml-project/
├── README.md
├── pyproject.toml
├── .gitignore
├── data/
│   ├── raw/          # 原始数据，只读
│   ├── interim/      # 中间数据
│   └── processed/    # 可建模数据
├── notebooks/        # 探索实验，不承担正式生产逻辑
├── src/
│   ├── data.py
│   ├── features.py
│   ├── train.py
│   ├── evaluate.py
│   └── predict.py
├── models/
├── reports/
└── tests/
```

原则：Notebook 用来探索，确认有效的逻辑要迁移到 `src/`；原始数据不直接覆盖；随机种子、数据版本、参数和指标要记录。

## 四、Python 基础：学到什么程度才够

### 4.1 必学知识

1. 基本类型：整数、浮点数、布尔值、字符串、`None`。
2. 容器：列表、元组、字典、集合。
3. 控制流程：`if`、`for`、`while`、`break`、`continue`。
4. 函数：参数、返回值、默认参数、关键字参数、作用域。
5. 推导式与生成器：列表/字典推导式、`yield`。
6. 文件处理：文本、CSV、JSON、路径。
7. 异常处理：`try/except/finally`，不要用空 `except` 吞掉错误。
8. 模块与包：`import`、项目内导入、`__name__ == "__main__"`。
9. 面向对象：类、实例、继承的基本使用；机器学习初期不必钻研元类。
10. 类型提示、日志、测试、调试器。

### 4.2 数据处理中最常用的 Python 写法

```python
from pathlib import Path
import json

DATA_DIR = Path("data/raw")

def load_records(path: Path) -> list[dict]:
    """读取 JSON 列表并做最基本的结构校验。"""
    with path.open("r", encoding="utf-8") as f:
        records = json.load(f)
    if not isinstance(records, list):
        raise ValueError("JSON 顶层必须是列表")
    return records

def active_user_ids(records: list[dict]) -> set[str]:
    return {
        str(row["user_id"])
        for row in records
        if row.get("status") == "active" and row.get("user_id") is not None
    }

if __name__ == "__main__":
    rows = load_records(DATA_DIR / "users.json")
    print(f"活跃用户数：{len(active_user_ids(rows))}")
```

### 4.3 Python 阶段练习

- 统计日志中的错误类型和出现次数。
- 合并一个目录下的多个 CSV 文件。
- 读取订单 JSON，按客户和月份汇总金额。
- 写一个命令行脚本：输入文件路径，输出数据行数、缺失率和重复率。
- 给核心函数写 5～10 个 pytest 测试。

达标标准：不看教程也能把一份 CSV 清洗、汇总并导出；遇到错误会看 traceback，会打断点，会拆函数。

## 五、NumPy、Pandas 与数据可视化

### 5.1 NumPy

重点掌握：

- `ndarray` 的形状、维度、数据类型。
- 索引、切片、布尔筛选。
- 广播机制。
- 向量化计算，避免对数组逐元素写 Python 循环。
- 聚合：`mean`、`sum`、`min`、`max`、`std`、`quantile`。
- 随机数生成器和随机种子。
- 矩阵乘法、转置、点积。

```python
import numpy as np

rng = np.random.default_rng(42)
x = rng.normal(size=(1000, 5))
weights = np.array([0.8, -0.2, 0.0, 1.5, 0.3])
y = x @ weights + rng.normal(scale=0.2, size=1000)

standardized = (x - x.mean(axis=0)) / x.std(axis=0)
```

### 5.2 Pandas

必须熟练：

- `read_csv`、`read_excel`、`read_parquet`。
- `head`、`info`、`describe`、`dtypes`。
- `loc`、`iloc`、布尔筛选、排序。
- 缺失值：`isna`、`fillna`、`dropna`。
- 去重、字符串清洗、类型转换。
- `groupby`、`agg`、`transform`。
- `merge`、`join`、`concat`。
- `pivot_table`、宽表和长表转换。
- 日期解析、时间窗口和滞后特征。

```python
import pandas as pd

orders = pd.read_csv("data/raw/orders.csv", parse_dates=["order_time"])

quality = pd.DataFrame({
    "dtype": orders.dtypes.astype(str),
    "missing_rate": orders.isna().mean(),
    "n_unique": orders.nunique(dropna=False),
})

monthly = (
    orders.assign(month=orders["order_time"].dt.to_period("M").astype(str))
    .groupby(["customer_id", "month"], as_index=False)
    .agg(order_count=("order_id", "nunique"), revenue=("amount", "sum"))
)
```

注意：

- `apply` 很方便，但通常比向量化、`groupby` 或内置字符串函数慢。
- 合并前检查连接键是否唯一，避免多对多连接导致数据量意外膨胀。
- 金额、日期、类别编码和缺失值要显式检查，不能只看前五行。

### 5.3 探索性数据分析（EDA）清单

每份数据至少回答：

1. 一行代表什么？目标变量是什么？预测发生在什么时点？
2. 数据量、字段类型、时间范围、主键是什么？
3. 缺失、重复、异常值分别有多少？
4. 目标变量分布如何？分类是否失衡？回归是否长尾？
5. 数值特征的分布、相关性和极端值如何？
6. 类别特征是否存在大量罕见类别或拼写不一致？
7. 数据是否随时间、地区、设备或人群发生漂移？
8. 是否存在只有结果发生后才知道的字段，即目标泄漏？

可视化选择：

- 单变量数值：直方图、箱线图。
- 单变量类别：条形图。
- 数值与数值：散点图、相关矩阵。
- 类别与目标：分组均值、比例图、箱线图。
- 时间变化：折线图，必要时加移动平均。

## 六、机器学习需要的数学

数学目标不是手工推导所有公式，而是能够理解数据、模型、损失函数和优化过程。

### 6.1 线性代数

需要理解：向量、矩阵、形状、点积、矩阵乘法、转置、逆矩阵的含义、秩、特征值与特征向量、正交、范数。

关键联系：

- 一个样本可表示为特征向量 $x$。
- 数据集常表示为矩阵 $X \in \mathbb{R}^{n \times p}$。
- 线性模型为 $\hat y = Xw + b$。
- L2 正则使用 $\|w\|_2^2$，L1 正则使用 $\|w\|_1$。
- PCA 使用新的正交方向保留尽可能多的方差。

### 6.2 微积分与优化

需要理解：函数、导数、偏导数、梯度、链式法则、凸函数、学习率。

均方误差：

$$
L(w,b)=\frac{1}{n}\sum_{i=1}^{n}(y_i-(w^Tx_i+b))^2
$$

梯度下降：

$$
\theta_{t+1}=\theta_t-\eta\nabla_\theta L(\theta_t)
$$

学习率过大可能震荡或发散，过小则收敛很慢。深度学习中的反向传播，本质上是利用链式法则高效计算每层参数的梯度。

### 6.3 概率与统计

重点：

- 均值、中位数、方差、标准差、分位数。
- 条件概率、独立性、贝叶斯公式。
- 常见分布：伯努利、二项、正态、泊松。
- 抽样、估计、置信区间、假设检验。
- 相关不等于因果。
- 最大似然估计。
- 偏差—方差权衡。

贝叶斯公式：

$$
P(A|B)=\frac{P(B|A)P(A)}{P(B)}
$$

不必一开始沉迷证明。学线性回归时补矩阵和最小二乘，学逻辑回归时补概率、对数似然和梯度，学 PCA 时补特征值和方差。

## 七、理解机器学习问题

### 7.1 常见任务

| 任务 | 输出 | 示例 | 常用指标 |
|---|---|---|---|
| 二分类 | 两种类别或概率 | 是否违约 | Precision、Recall、F1、ROC-AUC、PR-AUC |
| 多分类 | 多个互斥类别 | 故障类型 | Macro-F1、加权 F1、准确率 |
| 回归 | 连续数值 | 销量、价格 | MAE、RMSE、$R^2$ |
| 聚类 | 无标签分组 | 客户分群 | 轮廓系数 + 业务解释 |
| 排序 | 候选顺序 | 推荐结果 | NDCG、MAP、Recall@K |
| 时间序列 | 未来值 | 未来库存需求 | MAE、RMSE、MAPE/WAPE |
| 异常检测 | 异常分数 | 设备异常 | Precision@K、Recall、人工核验 |

### 7.2 先写“问题定义卡”

建模前写清：

- 样本单位：一行是一名客户、一次订单还是一个设备时间窗口？
- 预测目标：具体预测什么？标签如何生成？
- 预测时点：什么时候作出预测？当时能获得哪些字段？
- 预测窗口：预测未来 1 天、7 天还是 30 天？
- 决策动作：预测后谁采取什么动作？
- 错误成本：误报和漏报哪个更贵？
- 基线：不使用机器学习时怎么做？
- 成功标准：离线指标提升多少，线上业务指标提升多少？

如果这些问题说不清楚，调再多模型也不会形成可靠结果。

## 八、数据集划分与防止泄漏

### 8.1 三类数据集

- 训练集：拟合模型参数。
- 验证集：选择模型、特征和超参数。
- 测试集：只在方案基本确定后使用，估计最终泛化能力。

普通独立同分布数据可以随机划分；分类任务通常使用分层抽样。时间序列必须按时间先后划分，不能随机打乱未来数据到训练集。一个用户、病人、设备产生多行数据时，通常要按实体分组切分，避免同一实体同时出现在训练集和验证集。

### 8.2 常见数据泄漏

- 用全量数据计算均值后再切分。
- 先在全量数据做特征选择、过采样或 PCA，再交叉验证。
- 使用预测时点之后才生成的字段。
- 同一用户或同一订单的关联行跨训练集和验证集。
- 时间序列随机切分。
- 标签本身或标签的代理字段进入特征。

解决方法：把缺失值填补、标准化、编码、特征选择和模型放进同一个 `Pipeline`，让每个交叉验证折只使用自己的训练部分拟合预处理器。

## 九、经典机器学习算法地图

### 9.1 监督学习

#### 线性回归

- 用途：连续值预测、建立可解释基线。
- 优点：快、可解释、容易诊断。
- 局限：默认关系近似线性；对异常值和共线性敏感。
- 延伸：Ridge、Lasso、Elastic Net。

#### 逻辑回归

- 用途：分类概率预测。
- 优点：强基线、速度快、概率和系数较易解释。
- 注意：数值特征通常要标准化；类别变量需要编码；非线性关系需要构造特征或换模型。

#### 决策树

- 优点：能表达非线性和特征交互，不要求标准化。
- 缺点：单棵树容易过拟合且不稳定。
- 关键参数：`max_depth`、`min_samples_leaf`、`min_samples_split`。

#### 随机森林

- 多棵随机树进行集成，通常比单棵树稳定。
- 对结构化数据是可靠基线，但模型可能较大，外推能力有限。

#### 梯度提升树

- 代表：scikit-learn 的 HistGradientBoosting，以及 XGBoost、LightGBM、CatBoost。
- 结构化表格数据中通常非常强。
- 重点参数：树深/叶子数、学习率、树数量、采样比例、正则化。
- 先用默认参数建立基线，再围绕过拟合情况小范围调参。

#### 支持向量机（SVM）

- 中小规模数据、较高维特征可能表现良好。
- 对特征尺度敏感；大数据训练成本较高；概率输出需要额外校准。

#### K 近邻（KNN）

- 简单直观，适合作为教学和小数据基线。
- 对尺度、噪声和维数敏感；预测阶段成本较高。

#### 朴素贝叶斯

- 文本词袋特征和小数据分类中常用。
- 条件独立假设很强，但速度快、基线价值高。

### 9.2 无监督学习

#### K-Means

- 适合近似球状、尺度相近的簇。
- 必须先处理缺失值并通常需要标准化。
- 聚类结果需要业务解释，不能只看轮廓系数。

#### DBSCAN

- 能发现不规则形状簇和噪声点。
- 对距离尺度和参数敏感，高维空间效果容易下降。

#### 层次聚类

- 能用树状结构展示样本之间的合并关系。
- 适合样本量不太大、需要解释分层结构的场景。

#### PCA

- 用于降维、去相关和可视化。
- 对尺度敏感；主成分是线性组合，可解释性会下降。
- PCA 也必须只在训练数据上拟合。

## 十、完整的 scikit-learn 建模模板

下面模板覆盖数值和类别字段、缺失值处理、编码、交叉验证、评估和保存模型。

```python
from pathlib import Path
import joblib
import pandas as pd

from sklearn.compose import ColumnTransformer
from sklearn.impute import SimpleImputer
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import (
    classification_report,
    confusion_matrix,
    precision_recall_curve,
    roc_auc_score,
)
from sklearn.model_selection import StratifiedKFold, cross_validate, train_test_split
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import OneHotEncoder, StandardScaler

RANDOM_STATE = 42
TARGET = "is_churn"

df = pd.read_csv("data/processed/customers.csv")
X = df.drop(columns=[TARGET, "customer_id"])
y = df[TARGET]

X_train, X_test, y_train, y_test = train_test_split(
    X,
    y,
    test_size=0.2,
    stratify=y,
    random_state=RANDOM_STATE,
)

numeric_features = X.select_dtypes(include="number").columns.tolist()
categorical_features = X.select_dtypes(exclude="number").columns.tolist()

numeric_pipeline = Pipeline([
    ("imputer", SimpleImputer(strategy="median")),
    ("scaler", StandardScaler()),
])

categorical_pipeline = Pipeline([
    ("imputer", SimpleImputer(strategy="most_frequent")),
    ("onehot", OneHotEncoder(handle_unknown="ignore")),
])

preprocessor = ColumnTransformer([
    ("num", numeric_pipeline, numeric_features),
    ("cat", categorical_pipeline, categorical_features),
])

model = Pipeline([
    ("preprocess", preprocessor),
    ("classifier", LogisticRegression(max_iter=2000, class_weight="balanced")),
])

cv = StratifiedKFold(n_splits=5, shuffle=True, random_state=RANDOM_STATE)
scores = cross_validate(
    model,
    X_train,
    y_train,
    cv=cv,
    scoring=["roc_auc", "average_precision", "f1"],
    return_train_score=True,
)

print("CV ROC-AUC:", scores["test_roc_auc"].mean())
print("CV PR-AUC:", scores["test_average_precision"].mean())

model.fit(X_train, y_train)
prob = model.predict_proba(X_test)[:, 1]

# 阈值应根据业务成本和验证集选择；这里只做演示。
precision, recall, thresholds = precision_recall_curve(y_test, prob)
threshold = 0.4
pred = (prob >= threshold).astype(int)

print("Test ROC-AUC:", roc_auc_score(y_test, prob))
print(confusion_matrix(y_test, pred))
print(classification_report(y_test, pred, digits=4))

Path("models").mkdir(exist_ok=True)
joblib.dump(model, "models/churn_pipeline.joblib")
```

重要提醒：正式项目中不要使用测试集选择阈值。应使用验证集或交叉验证的折外预测选择阈值，测试集只做最后一次评估。

## 十一、评估指标怎么选

### 11.1 分类

混淆矩阵包含 TP、FP、TN、FN。

$$
Precision=\frac{TP}{TP+FP},\qquad Recall=\frac{TP}{TP+FN}
$$

$$
F1=2\cdot\frac{Precision\cdot Recall}{Precision+Recall}
$$

- Accuracy：类别比较均衡、错误成本接近时使用。
- Precision：误报代价高时重视，例如错误拦截正常交易。
- Recall：漏报代价高时重视，例如漏掉高风险故障。
- F1：Precision 和 Recall 的折中。
- ROC-AUC：评价整体排序能力，但极度不平衡时可能显得过于乐观。
- PR-AUC：正类稀少时通常更有解释力。
- Log Loss：关心概率质量时使用。
- 概率校准：预测 0.8 的样本中是否约 80% 真为正类。

分类阈值不是固定的 0.5。应该基于业务成本、处理能力和验证集曲线选择。

### 11.2 回归

- MAE：平均绝对误差，直观且对极端值相对稳健。
- RMSE：更重罚大误差。
- $R^2$：相对均值基线解释了多少方差，可能为负。
- MAPE：真实值接近 0 时会失真；此时考虑 WAPE、MAE 或其他业务指标。

始终把模型指标与简单基线比较，例如“预测为训练集均值”“预测为上周同一天”“沿用当前业务规则”。

## 十二、特征工程

### 12.1 常用方法

- 数值：对数变换、分箱、比例、差值、交互项、异常值标记。
- 类别：One-Hot、频次编码、目标编码（必须防泄漏）。
- 时间：小时、星期、月份、是否节假日、距某事件时长。
- 历史行为：过去 7/30/90 天次数、金额、均值、最大值、趋势。
- 文本：词袋、TF-IDF、预训练向量或语言模型表示。
- 图像：尺寸统一、归一化、数据增强。

### 12.2 时间窗口特征的关键原则

假设在时间 $t$ 预测未来 7 天，所有特征只能使用 $t$ 及之前的信息。例如“未来 7 天累计订单量”绝不能成为特征。计算滚动统计时，要确认窗口边界和 `shift` 是否正确。

### 12.3 特征选择

- 先删除明显无意义、泄漏、常量和重复字段。
- 用业务知识和交叉验证判断特征价值。
- 线性模型可使用 L1 正则。
- 树模型可参考置换重要性；不要把树的内置重要性当作唯一证据。
- 强相关不代表应该全部删除，是否删除取决于模型、稳定性和解释需求。

## 十三、调参与实验管理

正确顺序：

1. 定义指标和数据切分。
2. 建立最简单的业务/统计基线。
3. 训练 2～4 类模型的默认或轻量参数版本。
4. 做误差分析，检查数据和特征。
5. 只对有希望的模型调参。
6. 最终方案在测试集评估一次。

不要盲目网格搜索所有参数。优先使用随机搜索或贝叶斯优化，并限制搜索空间。每次实验至少记录：代码版本、数据版本、特征集合、随机种子、参数、指标、训练时间和结论。

过拟合诊断：

- 训练好、验证差：降低复杂度、加强正则、补数据、改切分、排查泄漏或分布差异。
- 训练和验证都差：模型偏弱、特征信息不足、标签噪声大或问题本身不可预测。
- 交叉验证波动大：样本少、数据分组不均、时间/人群分布差异明显。

## 十四、不平衡数据

处理顺序：

1. 先确认业务中真实正类比例，不要因为“不平衡”就自动采样。
2. 使用分层或分组切分。
3. 采用 PR-AUC、Recall、Precision、F1、成本函数等指标。
4. 调整分类阈值。
5. 尝试类别权重。
6. 必要时只在训练折内部欠采样或过采样。

采样必须在交叉验证的训练折内部进行，否则会造成泄漏。测试集应保持真实分布。

## 十五、误差分析与解释

模型训练完成后，把错误样本按以下维度切片：

- 时间、地区、渠道、客户类型、产品类型。
- 高值/低值、常见/罕见类别。
- 新客户与老客户。
- 缺失字段多少。
- 置信度最高的错误样本。

解释工具：

- 线性模型系数。
- 置换重要性。
- Partial Dependence / ICE。
- SHAP（复杂模型常用，但需理解其背景数据和相关特征限制）。
- 反事实分析。

解释是对模型行为的近似描述，不自动代表因果关系。模型利用某字段，不等于改变该字段就一定改变现实结果。

## 十六、深度学习与 PyTorch

### 16.1 什么时候需要深度学习

优先考虑：图像、音频、自然语言、大规模序列、复杂非结构化数据。对中小规模表格数据，梯度提升树通常是更值得先试的基线。

### 16.2 必学概念

- 张量、形状、设备 CPU/GPU。
- 自动求导。
- 神经网络层、激活函数。
- 前向传播、损失、反向传播、优化器。
- batch、epoch、学习率。
- 训练/验证模式：`model.train()` 与 `model.eval()`。
- Dropout、Batch Normalization、权重衰减。
- 早停、学习率调度、检查点。

### 16.3 PyTorch 最小训练模板

```python
import copy
import torch
from torch import nn
from torch.utils.data import DataLoader, TensorDataset

torch.manual_seed(42)
device = torch.device("cuda" if torch.cuda.is_available() else "cpu")

# 示例张量；真实项目应先划分数据，再只用训练集统计量做标准化。
X_train = torch.randn(2000, 20)
y_train = (X_train[:, :3].sum(dim=1) > 0).float().unsqueeze(1)
X_val = torch.randn(400, 20)
y_val = (X_val[:, :3].sum(dim=1) > 0).float().unsqueeze(1)

train_loader = DataLoader(
    TensorDataset(X_train, y_train), batch_size=64, shuffle=True
)
val_loader = DataLoader(
    TensorDataset(X_val, y_val), batch_size=256, shuffle=False
)

model = nn.Sequential(
    nn.Linear(20, 64),
    nn.ReLU(),
    nn.Dropout(0.2),
    nn.Linear(64, 1),
).to(device)

criterion = nn.BCEWithLogitsLoss()
optimizer = torch.optim.AdamW(model.parameters(), lr=1e-3, weight_decay=1e-4)

best_loss = float("inf")
best_state = None

for epoch in range(30):
    model.train()
    for xb, yb in train_loader:
        xb, yb = xb.to(device), yb.to(device)
        optimizer.zero_grad()
        logits = model(xb)
        loss = criterion(logits, yb)
        loss.backward()
        optimizer.step()

    model.eval()
    total_loss = 0.0
    with torch.no_grad():
        for xb, yb in val_loader:
            xb, yb = xb.to(device), yb.to(device)
            total_loss += criterion(model(xb), yb).item() * len(xb)
    val_loss = total_loss / len(val_loader.dataset)

    if val_loss < best_loss:
        best_loss = val_loss
        best_state = copy.deepcopy(model.state_dict())

model.load_state_dict(best_state)
torch.save(model.state_dict(), "models/mlp_state.pt")
```

二分类使用 `BCEWithLogitsLoss` 时，模型最后一层不要手工添加 Sigmoid；推理阶段再用 `torch.sigmoid(logits)` 转换成概率。

### 16.4 深度学习方向路线

- 计算机视觉：MLP → CNN → 数据增强 → 迁移学习 → 检测/分割。
- NLP：分词和 TF-IDF → 词向量 → RNN（理解即可）→ Attention → Transformer → 微调与检索增强。
- 时间序列：滞后特征 + 树模型基线 → 统计模型 → 深度序列模型。

先学会使用预训练模型和迁移学习，通常比从零训练大模型更符合真实工作。

## 十七、时间序列特别注意

时间序列不是把日期字段直接丢进随机切分模型。

必须处理：

- 按时间回测：滚动窗口或扩展窗口。
- 趋势、周期、节假日和促销。
- 滞后特征和滚动统计。
- 预测跨度：一步预测还是多步预测。
- 冷启动、缺货导致的销量截断。
- 不同序列之间的层级关系。

基线至少包括：上一期、去年同期、移动平均。复杂模型如果不能稳定超过朴素基线，就不应上线。

## 十八、模型部署

### 18.1 推理接口示例

```python
from contextlib import asynccontextmanager
import joblib
import pandas as pd
from fastapi import FastAPI
from pydantic import BaseModel, Field

MODEL = None

@asynccontextmanager
async def lifespan(app: FastAPI):
    global MODEL
    MODEL = joblib.load("models/churn_pipeline.joblib")
    yield

app = FastAPI(lifespan=lifespan)

class CustomerFeatures(BaseModel):
    age: float = Field(ge=0, le=120)
    monthly_fee: float = Field(ge=0)
    contract_type: str

@app.post("/predict")
def predict(features: CustomerFeatures):
    frame = pd.DataFrame([features.model_dump()])
    probability = float(MODEL.predict_proba(frame)[0, 1])
    return {
        "churn_probability": probability,
        "prediction": int(probability >= 0.4),
        "model_version": "1.0.0",
    }
```

部署时不只是“接口能返回结果”，还要考虑：

- 输入字段、类型、范围和枚举校验。
- 训练与推理使用同一个预处理流程。
- 模型版本和特征版本。
- 延迟、吞吐、批量预测还是在线预测。
- 日志中避免记录敏感数据。
- 回滚方案和灰度发布。
- 单元测试、集成测试和固定样例测试。

### 18.2 上线监控

监控四层：

1. 服务：可用率、错误率、延迟、资源使用。
2. 数据：缺失率、取值范围、类别变化、数据漂移。
3. 模型：预测分布、置信度、校准、实际指标。
4. 业务：收益、成本、转化、人工处理量、公平性或安全约束。

真实标签经常延迟到达，因此需要同时监控输入分布和预测分布，并在标签到达后计算实际性能。

## 十九、三个递进项目

### 项目一：客户流失预测

目标：完成结构化数据二分类闭环。

- 明确预测时点和流失定义。
- EDA、缺失和类别检查。
- 逻辑回归基线。
- 随机森林或梯度提升树比较。
- 交叉验证、PR-AUC、阈值选择。
- 按客户类型做误差分析。
- 保存 Pipeline，提供 FastAPI 接口。

### 项目二：销量预测或设备故障预测

目标：掌握时间切分和历史窗口特征。

- 设计滚动回测。
- 建立上一期/移动平均基线。
- 构造滞后、滚动均值、节假日特征。
- 比较线性模型和树模型。
- 检查不同时间段、产品或设备的误差。

### 项目三：文本分类或图像分类

目标：掌握深度学习和迁移学习。

- 先做简单基线：文本用 TF-IDF + 逻辑回归，图像用小型 CNN。
- 再使用预训练模型。
- 分析混淆类别和高置信错误。
- 导出模型并制作批量/在线推理脚本。

### 高质量项目 README 应包含

- 问题背景和价值。
- 数据来源、字段含义和合规说明。
- 数据切分和防泄漏设计。
- 基线、模型和指标对比表。
- 最终模型、阈值和误差分析。
- 运行命令和环境说明。
- 已知限制与后续计划。

作品集看重完整决策过程，不只是 Notebook 最后一行的高分。

## 二十、24 周学习计划

### 第 1～4 周：Python 与数据处理

- 第 1 周：语法、容器、控制流程、函数。
- 第 2 周：文件、异常、模块、类、类型提示、pytest。
- 第 3 周：NumPy 与 Pandas。
- 第 4 周：数据清洗、可视化、完成 EDA 小项目。

### 第 5～8 周：统计和基础模型

- 第 5 周：描述统计、概率、抽样、置信区间。
- 第 6 周：线性代数、导数、梯度下降。
- 第 7 周：线性回归、正则化、回归指标。
- 第 8 周：逻辑回归、分类指标、阈值。

### 第 9～12 周：树模型与可靠评估

- 第 9 周：决策树、随机森林。
- 第 10 周：梯度提升树。
- 第 11 周：Pipeline、交叉验证、调参、防泄漏。
- 第 12 周：完成客户流失或风控项目。

### 第 13～16 周：扩展算法和工程基础

- 第 13 周：聚类、PCA、异常检测。
- 第 14 周：特征工程、解释、误差分析。
- 第 15 周：代码结构、配置、日志、测试。
- 第 16 周：FastAPI、模型保存、容器基础。

### 第 17～20 周：深度学习

- 第 17 周：张量、自动求导、MLP。
- 第 18 周：训练循环、正则化、优化器、检查点。
- 第 19 周：CNN 或文本表示。
- 第 20 周：迁移学习项目。

### 第 21～24 周：综合项目与作品集

- 第 21 周：选题、问题定义、数据审计。
- 第 22 周：基线、特征、模型比较。
- 第 23 周：误差分析、解释、部署。
- 第 24 周：测试、README、演示和复盘。

## 二十一、每周学习方式

推荐时间分配：

- 20%：看课程、书或文档。
- 30%：跟着实现小例子，但不能只复制。
- 40%：独立项目和调试。
- 10%：复盘、整理笔记、补测试。

每个知识点用四个问题验收：

1. 它解决什么问题？
2. 它依赖什么假设？
3. 关键参数怎样影响欠拟合和过拟合？
4. 什么情况下不应该使用它？

每周至少产出一个可以运行的结果：脚本、Notebook、图表、实验对比、API 或项目说明。只看视频没有可验证产出，通常会产生“听懂了但写不出”的错觉。

## 二十二、常见误区

1. 一上来就学大模型，Python、数据切分和评估却不扎实。
2. 只追求指标，不先确认标签和业务目标。
3. 先预处理全量数据再切分，造成泄漏。
4. 在测试集反复调参，测试集实际上变成了验证集。
5. 类别失衡时只看准确率。
6. 时间数据随机切分。
7. 只会调用 `fit`，不会分析错误样本。
8. 认为更复杂模型必然更好。
9. Notebook 从头到尾依赖隐藏执行状态，无法复现。
10. 训练和线上各写一份预处理逻辑，导致训练—服务偏差。
11. 不记录数据和模型版本。
12. 把相关性、特征重要性误解为因果关系。

## 二十三、面试与能力验收

### Python

- 列表、元组、字典、集合的区别。
- 可变对象、浅拷贝和深拷贝。
- 迭代器、生成器、上下文管理器。
- 异常、日志、测试和复杂度。

### 数据与统计

- 均值和中位数何时差异很大？
- 置信区间和假设检验表示什么？
- 如何处理缺失值和异常值？
- 如何判断样本偏差和数据漂移？

### 机器学习

- 为什么需要训练/验证/测试集？
- L1 与 L2 正则有什么区别？
- 树模型为什么不要求标准化？
- Bagging 和 Boosting 有什么区别？
- ROC-AUC 与 PR-AUC 如何选择？
- 如何发现并防止数据泄漏？
- 训练好验证差、训练验证都差分别如何处理？
- 预测概率为什么需要校准？

### 项目表达模板

用 3～5 分钟说清：背景和决策 → 样本与标签 → 数据切分 → 基线 → 模型与指标 → 最大困难 → 误差分析 → 部署与监控 → 局限。

## 二十四、最终检查清单

### 数据

- [ ] 一行数据的含义明确。
- [ ] 主键、重复、缺失、异常和时间范围已检查。
- [ ] 标签定义与预测时点一致。
- [ ] 训练、验证、测试划分符合业务结构。
- [ ] 不存在未来信息、实体交叉或预处理泄漏。

### 模型

- [ ] 有简单基线。
- [ ] 指标符合业务成本。
- [ ] 预处理和模型在同一 Pipeline 中。
- [ ] 交叉验证策略正确。
- [ ] 做了误差分析、概率/阈值分析和稳定性检查。
- [ ] 测试集没有参与调参。

### 工程

- [ ] 环境和随机种子可复现。
- [ ] 数据、特征、模型有版本。
- [ ] 关键逻辑有测试。
- [ ] 接口有输入校验、版本号和错误处理。
- [ ] 有监控、告警和回滚方案。
- [ ] README 能让别人独立运行项目。

## 二十五、最短实践路径

如果想马上开始，按下面做：

1. 用 7 天学完 Python 核心语法、函数、文件和 Pandas 基础。
2. 找一份结构化二分类数据，用 3 天完成 EDA。
3. 用 Pipeline 训练逻辑回归，得到第一个可信基线。
4. 加入树模型，用交叉验证比较 ROC-AUC、PR-AUC 和 F1。
5. 做阈值和错误样本分析，不要只调参。
6. 把最好模型保存并做成 FastAPI 接口。
7. 补齐 README、测试、运行命令和局限说明。
8. 完成一个项目后，再进入 PyTorch；不要同时开五门课。

真正的进步标志不是“看完多少内容”，而是能否面对一份陌生数据，独立说清问题、建立可靠基线、避免泄漏、解释误差，并把模型交付给别人使用。

## 二十六、学习资料与使用顺序

优先使用官方教程，因为示例通常与当前 API 保持一致；遇到概念不理解，再用教材或视频补充解释。

1. Python：先完成 [Python 官方教程](https://docs.python.org/3/tutorial/index.html) 中的基础语法、数据结构、函数、模块、输入输出、异常和类。
2. Pandas：先做 [Getting started](https://pandas.pydata.org/docs/getting_started/index.html)，然后把真实数据清洗中的问题带到 User Guide 查阅。
3. scikit-learn：先看 [Getting Started](https://scikit-learn.org/stable/getting_started.html)，再按算法进入 [User Guide](https://scikit-learn.org/stable/user_guide.html)；重点阅读预处理、Pipeline、交叉验证、指标、阈值和概率校准。
4. PyTorch：按 [Learn the Basics](https://docs.pytorch.org/tutorials/beginner/basics/intro.html) 顺序完成张量、数据集、模型、自动求导、优化和模型保存。

资料选择原则：

- 一门主课或一本主教材足够，不要同时跟五套教程。
- 官方文档负责确认 API，教材负责建立系统理解，项目负责形成能力。
- 视频中能听懂不代表会用；关闭视频后从空文件重新实现一次。
- 遇到报错先读 traceback 最底部，再检查输入类型、形状、列名和版本。
- 搜索代码时，必须理解数据切分和评估方式，不能只复制最终模型。

## 二十七、第一周每日行动表

### 第 1 天：环境与基本语法

- 创建虚拟环境和 Git 项目。
- 学变量、字符串、数字、布尔值和输入输出。
- 写一个订单金额计算脚本。

### 第 2 天：容器和循环

- 学列表、元组、字典、集合和切片。
- 统计一组订单中每种产品的数量和金额。

### 第 3 天：函数

- 学参数、返回值、作用域、类型提示。
- 把前两天脚本拆成 4～6 个小函数。

### 第 4 天：文件与异常

- 读写 CSV、JSON 和文本。
- 对文件不存在、字段缺失、类型错误提供明确异常信息。

### 第 5 天：Pandas 入门

- 读取 CSV，使用 `info`、`describe`、筛选、排序和 `groupby`。
- 输出一份客户月度汇总表。

### 第 6 天：数据质量

- 检查重复、缺失、非法日期、负金额和异常类别。
- 写一个通用数据质量报告函数。

### 第 7 天：复盘项目

- 从空目录重新实现“读取订单—质量检查—清洗—汇总—导出”。
- 写 README 和至少 5 个测试。
- 记录本周最常见的 3 个错误及解决方式。

完成第一周后，继续按 24 周路线推进。若每天只有 1 小时，不减少练习内容，而是把“一天”延长为两天；稳定持续比短期堆时长更重要。

## 二十八、模块快速上手详解

下面每个模块都按照“目标 → 学习顺序 → 动手任务 → 最小代码 → 练习 → 过关标准”展开。建议一次只攻克一个模块，完成过关标准后再进入下一个。

### 模块 1：Python 开发环境

#### 目标

能创建独立环境、运行脚本、安装依赖、使用 Git 保存代码，并且知道项目为什么在自己的电脑上能运行、换一台电脑却失败。

#### 学习顺序

1. 安装 Python，并确认 `python --version`。
2. 创建 `.venv`，激活后安装一个包。
3. 创建 `src/hello.py`，从终端运行它。
4. 创建 `requirements.txt` 或 `pyproject.toml`。
5. 初始化 Git，提交一次可运行版本。

#### 动手任务

```bash
mkdir ml-start
cd ml-start
python -m venv .venv

# Windows PowerShell
.venv\Scripts\Activate.ps1

# macOS / Linux
source .venv/bin/activate

python -m pip install --upgrade pip
python -c "import sys; print(sys.executable)"
mkdir src data tests
git init
```

创建 `src/hello.py`：

```python
import sys

def main() -> None:
    print(f"Python: {sys.version.split()[0]}")
    print("机器学习项目环境已启动")

if __name__ == "__main__":
    main()
```

#### 练习

- 把 Python、操作系统、当前工作目录打印出来。
- 安装 `numpy`，在脚本中打印版本号。
- 删除并重新创建虚拟环境，确认项目仍能运行。

#### 过关标准

你能从一个空目录创建环境、运行脚本、安装依赖，并说明“系统 Python”和“项目虚拟环境”的区别。不要在未解决环境问题前继续安装复杂框架。

### 模块 2：Python 语法与数据结构

#### 目标

能读懂和编写普通数据处理代码，尤其是列表、字典、循环、函数和异常处理。机器学习库并不能代替这些基础能力。

#### 学习顺序

变量和类型 → 条件与循环 → 列表/字典 → 函数 → 推导式 → 文件 → 异常 → 模块。

#### 最小任务：订单汇总

```python
orders = [
    {"customer": "A", "amount": 120.0, "status": "paid"},
    {"customer": "B", "amount": 80.0, "status": "cancelled"},
    {"customer": "A", "amount": 50.0, "status": "paid"},
]

def paid_total(rows: list[dict]) -> dict[str, float]:
    totals: dict[str, float] = {}
    for row in rows:
        if row.get("status") != "paid":
            continue
        customer = str(row["customer"])
        totals[customer] = totals.get(customer, 0.0) + float(row["amount"])
    return totals

print(paid_total(orders))  # {'A': 170.0}
```

#### 必须理解

- `list` 有顺序，`dict` 是键值映射，`set` 用于去重。
- 函数应该接收输入并返回结果，不要把所有逻辑写在全局区域。
- `row.get("x")` 在键不存在时返回 `None`；`row["x"]` 会直接报错。
- `for` 循环中的变量作用域、可变对象引用和默认参数容易造成隐蔽问题。
- 看到异常时先看最后一行错误信息，再回到 traceback 中定位自己的代码行。

#### 练习

1. 统计日志中每种 `level` 的数量。
2. 找出金额最大的 10 笔订单。
3. 对缺失金额、负金额和未知状态抛出明确的 `ValueError`。
4. 把代码拆成 `load`、`validate`、`summarize` 三个函数。

#### 过关标准

不看教程也能写一个“读取 JSON—校验字段—汇总—导出 JSON”的脚本，并且能为异常输入写测试。

### 模块 3：NumPy 与向量化

#### 目标

理解数组的形状和批量计算，为 Pandas、机器学习模型和深度学习张量打基础。

#### 学习顺序

数组创建 → `shape`/`ndim` → 切片和布尔索引 → 广播 → 聚合 → 矩阵乘法 → 随机数。

```python
import numpy as np

rng = np.random.default_rng(42)
scores = rng.normal(loc=70, scale=10, size=(100, 3))

print(scores.shape)                 # (100, 3)
print(scores.mean(axis=0))          # 每个科目的平均分
passed = scores[(scores >= 60).all(axis=1)]
normalized = (scores - scores.mean(0)) / scores.std(0)

weights = np.array([0.3, 0.3, 0.4])
final_score = scores @ weights
```

#### 常见错误

- 把 `(n,)`、`(n, 1)` 和 `(1, n)` 当成同一种形状。
- 使用 Python 循环逐个计算，忽略 NumPy 的向量化。
- 在标准差为 0 的列上直接标准化。
- 没有固定随机种子，导致每次实验结果都变。

#### 练习

- 写一个不使用显式 `for` 的标准化函数。
- 计算每一行的最大值、最小值和极差。
- 用矩阵乘法实现一个线性模型的预测。

#### 过关标准

看到 `X.shape` 就能判断样本数、特征数和运算方向；能用布尔索引和广播解决大部分简单数组计算。

### 模块 4：Pandas 数据处理

#### 目标

能把真实 CSV/Excel/数据库查询结果处理成可分析、可建模的数据表。

#### 学习顺序

读取 → 检查 → 选择 → 清洗 → 合并 → 分组聚合 → 时间特征 → 导出。

```python
import pandas as pd

df = pd.read_csv("data/raw/orders.csv", parse_dates=["order_time"])
print(df.shape)
print(df.info())

df = df.drop_duplicates(subset=["order_id"])
df["amount"] = pd.to_numeric(df["amount"], errors="coerce")
df["status"] = df["status"].fillna("unknown").str.strip().str.lower()
df["month"] = df["order_time"].dt.to_period("M").astype(str)

summary = (
    df.groupby(["customer_id", "month"], as_index=False)
      .agg(order_count=("order_id", "nunique"),
           total_amount=("amount", "sum"),
           avg_amount=("amount", "mean"))
)
summary.to_csv("data/processed/customer_month.csv", index=False)
```

#### 合并数据前必须检查

```python
print(left["customer_id"].duplicated().mean())
print(right["customer_id"].duplicated().mean())
merged = left.merge(right, on="customer_id", how="left", validate="many_to_one")
```

`validate` 能在连接关系不符合预期时立即报错，避免多对多连接让数据行数突然膨胀。

#### 练习

1. 写数据质量报告：行数、列数、类型、缺失率、重复数、唯一值数。
2. 合并三个月订单 CSV，并增加月份字段。
3. 按客户计算最近一次订单距今天的天数。
4. 找出金额为负、日期为空、客户不存在的记录，分别导出异常文件。

#### 过关标准

面对陌生表格，能在 30 分钟内回答：数据有多少行、主键是什么、缺失在哪里、目标如何分布、哪些字段能用于预测。

### 模块 5：可视化与探索性分析

#### 目标

不是“画漂亮图”，而是通过图表决定接下来该清洗什么、构造什么特征、使用什么指标。

#### 推荐分析顺序

1. 先看数据量、目标分布和时间范围。
2. 再看数值字段分布和极端值。
3. 再看类别字段的频数和长尾。
4. 最后分析特征与目标的关系。

```python
import matplotlib.pyplot as plt
import seaborn as sns

sns.set_theme(style="whitegrid")
fig, axes = plt.subplots(1, 3, figsize=(16, 4))
sns.histplot(data=df, x="amount", bins=40, ax=axes[0])
sns.boxplot(data=df, x="status", y="amount", ax=axes[1])
sns.countplot(data=df, x="is_churn", ax=axes[2])
plt.tight_layout()
plt.savefig("reports/eda.png", dpi=150)
```

#### 每张图都要写结论

例如：不要只放“金额分布图”，而要写“金额右偏明显，少数大额订单影响均值；建模时尝试 `log1p(amount)`，评估使用 MAE 而不是只看 RMSE”。

#### 过关标准

你能从一份图表中提出至少 3 个可验证假设，并用分组统计或下一轮实验验证，而不是把相关性直接当成因果。

### 模块 6：数学与统计的实用学法

#### 目标

理解模型为什么这样计算，不要求一开始手推所有复杂证明。

#### 对照学习表

| 机器学习内容 | 需要补的数学 | 动手验证 |
|---|---|---|
| 线性回归 | 向量、矩阵、最小二乘 | 手写一元线性回归梯度下降 |
| 逻辑回归 | 概率、Sigmoid、对数损失 | 画 Sigmoid 和损失曲线 |
| 正则化 | 范数、偏差—方差 | 比较不同 `alpha` 的系数 |
| PCA | 方差、协方差、特征向量 | 降到二维后画散点图 |
| 神经网络 | 偏导、链式法则、梯度 | 用自动求导核对手算梯度 |
| 评估指标 | 条件概率、抽样、估计 | 用混淆矩阵计算 Precision/Recall |
| 时间序列 | 滞后、平稳性、滚动统计 | 与上一期基线做回测 |

#### 最小梯度下降示例

```python
import numpy as np

rng = np.random.default_rng(0)
x = np.linspace(0, 10, 100)
y = 3 * x + 2 + rng.normal(0, 1, len(x))
w, b = 0.0, 0.0
lr = 0.01

for _ in range(2000):
    pred = w * x + b
    error = pred - y
    dw = 2 * np.mean(error * x)
    db = 2 * np.mean(error)
    w -= lr * dw
    b -= lr * db

print(round(w, 2), round(b, 2))
```

#### 过关标准

你能用自己的话解释：损失函数在衡量什么、梯度指向哪里、学习率过大或过小会怎样、为什么训练集指标不能代表泛化能力。

### 模块 7：监督学习与第一个分类项目

#### 目标

完成一次可靠的“定义标签—划分数据—预处理—训练—评估—分析”闭环。

#### 实战步骤

1. 写问题定义卡：样本、标签、预测时点、错误成本。
2. 检查目标比例和重复实体。
3. 先做多数类或业务规则基线。
4. 使用逻辑回归作为可解释基线。
5. 再比较随机森林或梯度提升树。
6. 用交叉验证比较模型，不要用测试集反复调参。
7. 根据业务成本选择阈值。
8. 对高置信错误做切片分析。

```python
from sklearn.model_selection import train_test_split
from sklearn.metrics import average_precision_score, classification_report
from sklearn.pipeline import make_pipeline
from sklearn.preprocessing import StandardScaler
from sklearn.linear_model import LogisticRegression

X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, stratify=y, random_state=42
)
model = make_pipeline(
    StandardScaler(),
    LogisticRegression(max_iter=2000, class_weight="balanced")
)
model.fit(X_train, y_train)
prob = model.predict_proba(X_test)[:, 1]
pred = (prob >= 0.4).astype(int)
print("PR-AUC:", average_precision_score(y_test, prob))
print(classification_report(y_test, pred, digits=4))
```

#### 必须回答的问题

- 为什么这里用分层划分？
- 正类比例很低时，为什么不能只看 Accuracy？
- 为什么标准化必须放入 Pipeline？
- 0.4 阈值是谁决定的？它是否应该由测试集决定？

#### 过关标准

你能提交一份报告，包含基线、至少两个模型、交叉验证结果、测试结果、混淆矩阵、阈值理由和错误样本分析。

### 模块 8：回归、树模型与调参

#### 目标

理解结构化数据中最常用的模型家族，并能识别欠拟合、过拟合和数据泄漏。

#### 推荐顺序

线性回归 → Ridge/Lasso → 决策树 → 随机森林 → 梯度提升树。

#### 调参只改有意义的参数

- 线性模型：正则强度 `alpha`。
- 决策树：`max_depth`、`min_samples_leaf`。
- 随机森林：树数、最大特征数、叶节点最小样本。
- Boosting：学习率、树数、树深或叶子数、正则化。

```python
from sklearn.model_selection import RandomizedSearchCV
from sklearn.ensemble import HistGradientBoostingRegressor
from sklearn.metrics import mean_absolute_error

search = RandomizedSearchCV(
    HistGradientBoostingRegressor(random_state=42),
    param_distributions={
        "learning_rate": [0.03, 0.05, 0.1],
        "max_iter": [100, 200, 400],
        "max_leaf_nodes": [15, 31, 63],
        "l2_regularization": [0.0, 0.1, 1.0],
    },
    n_iter=15,
    scoring="neg_mean_absolute_error",
    cv=5,
    random_state=42,
    n_jobs=-1,
)
search.fit(X_train, y_train)
print(search.best_params_)
print(mean_absolute_error(y_test, search.predict(X_test)))
```

#### 过关标准

你能画出训练/验证误差随模型复杂度变化的趋势，并说明最终选择是因为泛化能力、速度、可解释性还是业务成本，而不是因为某次随机划分分数最高。

### 模块 9：无监督学习与异常检测

#### 目标

在没有标签时发现结构，但不把算法输出直接当成“真实类别”。

#### K-Means 快速流程

```python
from sklearn.cluster import KMeans
from sklearn.preprocessing import StandardScaler
from sklearn.pipeline import make_pipeline

cluster_model = make_pipeline(
    StandardScaler(),
    KMeans(n_clusters=4, n_init="auto", random_state=42),
)
df["cluster"] = cluster_model.fit_predict(df[["frequency", "amount", "recency"]])
profile = df.groupby("cluster")[["frequency", "amount", "recency"]].mean()
print(profile)
```

#### 正确理解聚类

- `cluster=0` 不代表天然存在某种客户类型，必须查看各簇的特征画像。
- `n_clusters` 不是只能靠肘部图决定，要结合业务可执行性。
- 高维数据中的距离可能失去意义，必要时先做特征选择或降维。
- 异常检测需要人工抽样核验，并且要明确误报成本。

#### 过关标准

你能为每个簇写出可读的名称、典型特征、样本比例、可采取的动作和不能得出的结论。

### 模块 10：时间序列

#### 目标

掌握“只能用过去预测未来”的建模逻辑。

#### 快速上手步骤

1. 按时间排序，确认一个时间点是否有多条记录。
2. 建立上一期、移动平均和去年同期基线。
3. 生成滞后特征时先 `shift`，再计算滚动值。
4. 使用滚动回测，不随机打乱。
5. 分别分析不同预测跨度和不同产品/设备。

```python
df = df.sort_values(["product_id", "date"])
g = df.groupby("product_id", group_keys=False)
df["lag_1"] = g["demand"].shift(1)
df["lag_7"] = g["demand"].shift(7)
df["rolling_7"] = g["demand"].transform(
    lambda s: s.shift(1).rolling(7, min_periods=3).mean()
)
```

`shift(1)` 很重要：如果先计算当天滚动平均，再用它预测当天，就把目标信息混进特征了。

#### 过关标准

你能画出每次回测的训练窗口和验证窗口，说明每个特征在预测时点是否真的可获得，并能和朴素基线比较。

### 模块 11：特征工程、Pipeline 与防泄漏

#### 目标

把所有会“从数据学习参数”的步骤封装进训练流程，确保交叉验证和线上推理一致。

#### 规则

- 缺失值填补器只能在训练折拟合。
- 标准化器只能在训练折拟合。
- One-Hot 的类别集合只能从训练折获得。
- 目标编码、过采样、特征选择和 PCA 也必须放在交叉验证内部。
- 线上输入列名、类型和单位必须与训练一致。

```python
from sklearn.compose import ColumnTransformer
from sklearn.impute import SimpleImputer
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import OneHotEncoder, StandardScaler

preprocess = ColumnTransformer([
    ("num", Pipeline([
        ("impute", SimpleImputer(strategy="median")),
        ("scale", StandardScaler()),
    ]), numeric_features),
    ("cat", Pipeline([
        ("impute", SimpleImputer(strategy="most_frequent")),
        ("onehot", OneHotEncoder(handle_unknown="ignore")),
    ]), categorical_features),
])
```

#### 排查泄漏的提问方式

对每个字段问：“在真实预测发生的那一刻，这个值是否已经存在？它是否由标签、未来事件或人工复核结果生成？”只要答案不确定，就暂时移除并做对照实验。

#### 过关标准

你能人为构造一个泄漏字段，观察它让验证分数异常升高，再删除它并解释分数为什么回落。

### 模块 12：PyTorch 深度学习

#### 目标

能从数据张量开始，写出训练、验证、保存检查点和推理流程；不追求第一天训练大模型。

#### 学习顺序

张量和形状 → Dataset/DataLoader → `nn.Module` → 损失函数 → 优化器 → 训练/验证模式 → 早停 → GPU。

#### 每轮训练必须有的结构

```python
for epoch in range(epochs):
    model.train()
    for xb, yb in train_loader:
        optimizer.zero_grad()
        loss = criterion(model(xb), yb)
        loss.backward()
        optimizer.step()

    model.eval()
    with torch.no_grad():
        # 只计算验证损失和指标，不更新参数
        val_loss = evaluate(model, val_loader)
```

#### 训练失败排查顺序

1. 先用 10 个样本尝试过拟合，确认模型、标签和损失函数连接正确。
2. 打印一个 batch 的形状、类型、最小值和最大值。
3. 检查标签编码与损失函数是否匹配。
4. 检查学习率、归一化、`train/eval` 模式。
5. 再考虑网络结构和数据增强。

#### 过关标准

你能解释一个 batch 从 DataLoader 到 loss 再到参数更新的完整路径，并保存最佳验证模型，而不是只保存最后一轮。

### 模块 13：模型解释与误差分析

#### 目标

知道模型在哪些样本上可靠、在哪些场景会失败，以及如何把结果转成业务动作。

#### 固定分析模板

1. 按置信度分组：高置信正确、高置信错误、低置信样本。
2. 按人群/产品/地区/时间切片。
3. 统计各切片的样本量、正例率、Precision、Recall、MAE。
4. 查看特征缺失率和输入分布。
5. 对错误样本做人工归因：标签错、数据错、特征不足、模型不足。

```python
result = test_df.assign(prob=prob, pred=pred)
result["correct"] = result["pred"] == result["label"]
slice_report = (
    result.groupby("customer_type")
          .agg(n=("label", "size"),
               actual_rate=("label", "mean"),
               predicted_rate=("pred", "mean"),
               error_rate=("correct", lambda s: 1 - s.mean()))
)
print(slice_report)
```

#### 过关标准

你能回答“模型整体分数不错，为什么业务仍然不满意”，并能指出至少一个可改进的数据或流程问题。

### 模块 14：部署与工程化

#### 目标

把训练结果变成别人可以稳定调用的程序，而不是只在自己的 Notebook 里运行。

#### 快速上手顺序

1. 用 `joblib`/PyTorch 保存完整模型和配置。
2. 写一个批量预测脚本，先验证输入输出。
3. 使用 Pydantic 校验接口输入。
4. 用 FastAPI 暴露 `/health` 和 `/predict`。
5. 增加模型版本、日志、异常响应和固定样例测试。
6. 再考虑 Docker、CI/CD、灰度和监控。

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/health")
def health() -> dict[str, str]:
    return {"status": "ok", "model_version": "1.0.0"}
```

#### 上线前自测

- 正常请求是否返回正确字段？
- 缺字段、类型错误、超范围值是否被拒绝？
- 未知类别是否会导致编码器崩溃？
- 训练时的单位、时区和缺失处理是否与线上一致？
- 服务重启后是否能加载模型？
- 是否记录输入摘要而不是敏感原始数据？

#### 过关标准

别人只按 README 的命令，就能启动服务、发送一条样例请求、得到预测结果，并知道模型版本和限制。

### 模块 15：综合项目执行法

#### 目标

把知识串成可展示的作品，不在数据、模型、部署之间反复迷路。

#### 项目分阶段交付

**第 1 阶段：问题和数据（1～2 天）**

- 写一页问题定义卡。
- 列出字段、标签、预测时点和数据来源。
- 做数据质量报告。

**第 2 阶段：基线（1～2 天）**

- 实现业务规则或均值/上一期基线。
- 用一个简单模型跑通端到端流程。

**第 3 阶段：模型迭代（3～5 天）**

- 增加特征和第二种模型。
- 使用正确的交叉验证。
- 记录每次实验和结论。

**第 4 阶段：误差与交付（2～4 天）**

- 做切片和高置信错误分析。
- 固定测试集一次性评估。
- 保存模型，写接口和 README。

#### 项目目录最低要求

```text
project/
├── README.md
├── data/README.md
├── notebooks/01_eda.ipynb
├── src/train.py
├── src/predict.py
├── src/api.py
├── models/
├── reports/metrics.csv
└── tests/test_predict.py
```

#### 过关标准

项目不依赖你的口头解释也能运行；README 能说明为什么这样切分、为什么选这个指标、模型在哪些场景可能失败。

## 二十九、30 天快速上手计划

如果你想尽快获得“能做项目”的能力，可以按下面执行。每天 60～90 分钟即可；已有开发经验时，不必把语法阶段拖长。

| 天数 | 任务 | 当天产出 |
|---:|---|---|
| 1 | 建虚拟环境、Git、项目目录 | 可运行脚本 |
| 2 | Python 容器与循环 | 订单汇总 |
| 3 | 函数、类型提示、异常 | 可复用函数 |
| 4 | CSV/JSON 读写 | 数据加载脚本 |
| 5 | Pandas 基础 | 清洗后的表 |
| 6 | 缺失、重复、异常检查 | 质量报告 |
| 7 | 周复盘与 5 个测试 | 第一个小项目 |
| 8 | NumPy 数组和形状 | 数值处理脚本 |
| 9 | Pandas 分组、合并 | 汇总表 |
| 10 | 可视化 | EDA 图表 |
| 11 | 目标定义和数据切分 | 问题定义卡 |
| 12 | 业务/统计基线 | 基线指标 |
| 13 | 逻辑回归 | 分类基线 |
| 14 | 分类指标和混淆矩阵 | 评估报告 |
| 15 | Pipeline 与缺失编码 | 无泄漏流程 |
| 16 | 决策树与随机森林 | 模型对比 |
| 17 | 交叉验证 | 稳定指标 |
| 18 | 阈值选择 | 业务阈值 |
| 19 | 误差切片 | 错误分析 |
| 20 | 特征工程 | 特征版本 |
| 21 | 最终测试集评估 | 最终结果 |
| 22 | 模型保存和批量预测 | `predict.py` |
| 23 | FastAPI `/health` | 健康检查 |
| 24 | FastAPI `/predict` | 预测接口 |
| 25 | 输入校验和异常处理 | 接口测试 |
| 26 | README 和运行命令 | 可复现项目 |
| 27 | 学习 PyTorch 张量和 DataLoader | 第一个 batch |
| 28 | 写 MLP 训练循环 | 训练曲线 |
| 29 | 复盘模型失败案例 | 限制清单 |
| 30 | 整理作品集和下一阶段计划 | 可展示项目 |

每天结束时只写三行复盘：今天学会了什么、哪里报错、明天要验证什么。这样比堆积大量没有运行过的笔记更容易形成长期能力。

## 三十、遇到问题时的排查顺序

### 代码报错

1. 读 traceback 的最后一行。
2. 打印变量类型、形状、列名和一条真实样本。
3. 缩小为 5～10 行最小复现代码。
4. 查官方文档和函数签名。
5. 修复后为这个错误增加一个测试。

### 模型分数异常高

1. 检查标签是否进入特征。
2. 检查是否使用了预测时点之后的字段。
3. 检查同一用户/订单/设备是否跨数据集。
4. 检查预处理是否在全量数据上拟合。
5. 与简单基线和时间外推结果比较。

### 模型分数很低

1. 先确认标签和样本定义没有错误。
2. 查看目标比例、缺失率和特征常量。
3. 检查训练集指标是否也很低。
4. 做少量样本过拟合测试，确认代码链路正确。
5. 重新做误差分析，不要直接盲目调参。

### 训练很慢或内存不够

1. 先减少数据和特征，确认逻辑正确。
2. 使用合适的数据类型和批处理。
3. 避免重复复制大型 DataFrame。
4. 统计每一步耗时和内存，而不是凭感觉优化。
5. 只有在基线正确后才做并行、缓存或 GPU 优化。

## 三十一、每个模块的最终验收表

| 模块 | 你应该能独立完成的事情 |
|---|---|
| 环境 | 创建环境、安装包、运行脚本、提交 Git |
| Python | 读取文件、写函数、校验数据、处理异常 |
| NumPy | 判断形状、向量化计算、矩阵乘法 |
| Pandas | 清洗、合并、聚合、导出数据 |
| EDA | 发现分布、异常、关系并提出假设 |
| 数学 | 解释损失、梯度、概率和指标 |
| 分类/回归 | 建基线、切分、训练、评估、选阈值 |
| 树模型 | 识别过拟合、调关键参数、分析重要性 |
| 无监督 | 聚类画像、异常抽样、避免过度解读 |
| 时间序列 | 滚动回测、构造滞后特征、防未来泄漏 |
| Pipeline | 让预处理和模型可复现且不泄漏 |
| PyTorch | 写 Dataset、训练循环、验证和检查点 |
| 解释分析 | 找到高置信错误和薄弱切片 |
| 部署 | 提供健康检查、预测接口、版本和校验 |
| 综合项目 | 从问题定义一直交付到 README 和测试 |

如果某个模块只能“看懂代码”而不能完成右侧任务，就先不要进入下一模块；回到最小任务，删掉复杂部分，重新从 10 行代码跑通。
