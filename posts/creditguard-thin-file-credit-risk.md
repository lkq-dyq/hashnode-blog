---
title: "基于替代数据的信用白户信贷风险评估工具设计与实现"
slug: creditguard-thin-file-credit-risk
subtitle: 秦梓恒的技术笔记
tags: machine-learning,fintech,credit-risk,python
publishedAt: 2026-09-23T10:00:00Z
saveAsDraft: false
enableToc: true
hideFromCommunity: false
canonical: https://lkq-dyq.github.io/machine-learning/fintech/2026/09/23/creditguard-thin-file-credit-risk.html
---

## 背景

传统信贷风控依赖央行征信报告，但有两类人群被排除在外：
1. **信用白户** — 无信贷记录或记录极其稀薄的人群（应届毕业生、自由职业者、农村人口）
2. **新市民** — 刚迁入城市的流动人口

如何评估这些人的信用风险？答案是**替代数据驱动评估范式**——从运营商消费、公共缴费、线上消费、政务履约、小额经营等非传统维度提取行为信号，用机器学习模型量化信用风险。

本文介绍 CreditGuard V1.0 的设计与实现，一个面向信用白户的桌面端信贷风险辅助评估工具。

## 技术架构

### 核心流程

```
历史数据(Excel/CSV) → 特征工程 → WOE编码 → IV值筛选 → L1正则化 → GridSearchCV超参数搜索 → 逻辑回归模型
                                                                                    ↓
单用户评估：9项特征填写 → 模型预测 → 信用评分(0-100) → 风险等级(低/中/高)
批量评估：Excel上传 → 批量预测 → 结果导出
```

### 技术栈

- Python 3.8
- CustomTkinter（GUI）
- scikit-learn（模型）
- pandas（数据）
- matplotlib（可视化）

## 特征工程

### 9 维替代数据特征

覆盖 5 个非传统维度：

| 维度 | 特征 | 说明 |
|------|------|------|
| 运营商消费 | 月均话费 | 消费稳定性信号 |
| 公共缴费 | 水电燃气缴费及时率 | 履约意愿信号 |
| 线上消费 | 电商月均消费额、消费品类多样性 | 消费能力与稳定性 |
| 政务履约 | 社保连续缴纳月数、纳税记录 | 职业稳定性 |
| 小额经营 | 微店/摆摊月均收入 | 经营能力 |

### WOE 编码

WOE（Weight of Evidence）是金融风控中标准的类别变量编码方法，将每个分箱的"好样本占比 vs 坏样本占比"转化为对数几率：

```python
def woe_encode(df, col, target):
    """WOE编码：将类别变量转化为对数几率"""
    woe_map = {}
    for category in df[col].unique():
        good = len(df[(df[col] == category) & (target == 0)])
        bad = len(df[(df[col] == category) & (target == 1)])
        # 拉普拉斯平滑避免除零
        woe = np.log((good + 0.5) / (bad + 0.5))
        woe_map[category] = woe
    return df[col].map(woe_map)
```

### IV 值筛选

IV（Information Value）衡量特征的预测能力，金融风控中的标准筛选指标：

| IV 值范围 | 预测能力 |
|-----------|----------|
| < 0.02 | 几乎无 |
| 0.02-0.1 | 弱 |
| 0.1-0.3 | 中等 |
| > 0.3 | 强 |

```python
def information_value(df, col, target):
    """计算特征的IV值"""
    iv = 0
    for category in df[col].unique():
        good = len(df[(df[col] == category) & (target == 0)]) / good_total
        bad = len(df[(df[col] == category) & (target == 1)]) / bad_total
        iv += (good - bad) * np.log((good + 0.5) / (bad + 0.5))
    return iv
```

只保留 IV > 0.1 的特征进入模型，减少噪声。

## 模型训练

### L1 正则化 + GridSearchCV

逻辑回归是金融风控的标配——完全可解释，权重系数直接反映特征重要性。L1 正则化自动做特征选择（稀疏化），GridSearchCV 搜索最优超参数：

```python
from sklearn.linear_model import LogisticRegression
from sklearn.model_selection import GridSearchCV

param_grid = {
    "C": [0.01, 0.1, 1, 10, 100],
    "penalty": ["l1"],
    "solver": ["liblinear"],
}

grid = GridSearchCV(
    LogisticRegression(),
    param_grid,
    cv=5,
    scoring="roc_auc",
)
grid.fit(X_train, y_train)

best_model = grid.best_estimator_
print(f"最优 C = {grid.best_params_['C']}")
print(f"AUC = {grid.best_score_:.4f}")
```

### SafeLabelEncoder

处理未见过的新标签是部署时的常见问题。标准 `LabelEncoder` 遇到新标签会抛异常，导致整个预测崩溃：

```python
class SafeLabelEncoder:
    """对未见过标签返回-1，不抛异常"""
    def __init__(self):
        self.classes_ = []

    def fit(self, labels):
        self.classes_ = list(set(labels))
        return self

    def transform(self, labels):
        result = []
        for label in labels:
            if label in self.classes_:
                result.append(self.classes_.index(label))
            else:
                result.append(-1)  # 未见标签返回-1
        return np.array(result)
```

## 信用评分

### 标准化评分公式

将模型输出的概率转化为 0-100 的标准化信用评分：

```python
def to_score(prob_good):
    """概率 → 标准化评分
    Score = 60 + 20 × ln(odds_good)
    odds_good = prob_good / (1 - prob_good)
    """
    prob_good = np.clip(prob_good, 0.001, 0.999)
    odds_good = prob_good / (1 - prob_good)
    score = 60 + 20 * np.log(odds_good)
    return np.clip(score, 0, 100)
```

### 三级风险等级

| 评分 | 等级 | 建议 |
|------|------|------|
| ≥ 75 | 低风险 | 建议通过 |
| 50-74 | 中风险 | 建议人工复审 |
| < 50 | 高风险 | 建议拒绝 |

## GUI 设计

基于 CustomTkinter 实现现代化深色主题界面：

- **模型训练页**：上传数据 → 训练进度条 → AUC/混淆矩阵展示
- **单用户评估页**：9 项特征表单 → 实时评分 → 风险等级
- **批量评估页**：Excel 上传 → 批量预测 → 结果导出
- **数据看板页**：训练数据分布可视化 + 模型性能展示

## 技术特点

1. **可解释性优先** — 逻辑回归权重系数可直接解读，满足金融风控合规审计要求
2. **替代数据驱动** — 覆盖传统征信体系之外的信用白户人群
3. **自动化特征工程** — WOE 编码 + IV 值筛选 + L1 正则化 + GridSearchCV 四步组合优化
4. **标准化评分输出** — 0-100 分对应三级风险，业务侧直接可用
5. **输入鲁棒性** — SafeLabelEncoder 对新标签返回 -1 不崩溃

## 局限与改进

V1.0 是基于传统机器学习的方案。后续 V1.1 转向 LLM 实验能力，探索大模型在信贷风控场景的评估能力边界，与逻辑回归基线对比。

## 总结

CreditGuard V1.0 实现了从替代数据到信用评分的完整链路：WOE 编码 + IV 筛选做特征工程，L1 正则化逻辑回归做模型，标准化评分公式做输出。核心价值在于**可解释性**——每一步都可以向审计人员解释"为什么这个人的评分是 65 分"。

代码已开源：[github.com/lkq-dyq/CreditGuard-V1.0](https://github.com/lkq-dyq/CreditGuard-V1.0)
