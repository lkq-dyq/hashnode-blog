---
title: "DataMind：零代码数据分析 Agent 工作站的设计与实现"
slug: datamind-zero-code-data-analysis-agent
subtitle: 秦梓恒的技术笔记
tags: llm,agent,data-analysis,python,sandbox
publishedAt: 2026-09-23T09:00:00Z
saveAsDraft: false
enableToc: true
hideFromCommunity: false
canonical: https://lkq-dyq.github.io/llm/agent/security/2026/09/23/datamind-zero-code-data-analysis-agent.html
---

## 背景

数据分析的典型痛点：业务人员有数据、有问题，但不会写代码；数据分析师会写代码，但沟通成本高、响应慢。能不能让大模型直接"写代码做分析"，同时保证安全性和可靠性？

DataMind 是一个零代码数据分析 Agent 工作站，用户只需用自然语言提问，系统自动生成分析代码、在受控沙箱中执行、校验结果有效性，最终交付结论与图表。整个过程中用户不需要写一行代码。

## 系统架构

### 整体设计

```
用户提问 → 规划Agent → 生成分析计划+代码
               ↓
          执行Agent → 受控沙箱运行
               ↓
          校验Agent → 结果是否成立？
               ↓
         不通过 → 诊断回灌 → 规划Agent重做（最多N次）
         通过   → 交付结论+图表
         仍失败 → 回落到规则模板（保证可用性底线）
```

### 三元 Agent 闭环

这是系统的核心设计。三个 Agent 各司其职：

**规划 Agent**：接收用户问题，生成分析计划（自然语言描述）和可执行代码。支持多候选方案——同一问题从不同分析角度生成多个 `PlanCandidate`，执行后择优。

```python
@dataclass
class PlanCandidate:
    """一个候选分析方案"""
    plan: str           # 分析计划描述
    code: str           # 可执行代码
    intent: str = ""    # 分析角度
    source: str = "rule"  # rule | llm
    score: float = 0.0  # 评分
    ok: bool = False    # 是否执行成功
    chart: bool = False # 是否出图
    rows: int = 0       # 结果行数
    diagnosis: str = "" # 诊断信息
    conclusion: str = ""# 结论
```

**执行 Agent**：将代码放入受控沙箱运行，捕获输出、图表和异常。

**校验 Agent**：判定结果是否成立——是否为空、是否答非所问、该出的图有没有出。不通过则把诊断回灌给规划 Agent 重做。

### 可用性底线：规则模板回落

无论 LLM 出什么问题（API 超时、生成错误代码、校验不通过），系统都保证有可交付的结果。当 LLM 路径失败 N 次后，回落到规则模板路径——基于关键词匹配的预置分析模板，覆盖常见分析场景。

## 受控代码沙箱

这是系统的**核心安全工程点**。让大模型"能写代码"，但不让它"能干任何事"。

### 四道闸门

```python
FORBIDDEN_NODES = (
    ast.Import, ast.ImportFrom,   # 禁 import
    ast.For, ast.While,            # 禁循环
    ast.FunctionDef, ast.ClassDef, # 禁函数与类定义
    ast.Global, ast.Nonlocal,      # 禁全局/非局部
    ast.Try, ast.Raise,            # 禁异常与断言
)
```

| 闸门 | 作用 | 实现 |
|------|------|------|
| AST 静态白名单 | 禁 import / 禁循环 / 禁函数定义 / 禁异常 | `ast.walk()` 遍历，拒绝禁止节点 |
| 名称黑名单 | eval、open、getattr、globals 等拒绝 | `ast.Name` 节点检查 |
| 属性过滤 | 以 `_` 开头的属性（`__class__`、`__globals__`）拒绝 | `ast.Attribute` 节点检查 |
| 受限命名空间 | 只注入 pd / np / plt / df，无 builtins 危险函数 | 自定义 `exec` 的 globals |

### 执行超时与输出捕获

```python
import threading

def run(code, timeout=10):
    result = ExecutionResult()
    def target():
        exec(code, restricted_globals, local_ns)
    t = threading.Thread(target=target)
    t.start()
    t.join(timeout)
    if t.is_alive():
        raise SandboxError("执行超时")
```

沙箱还叠加了执行超时（防死循环，虽然循环本身已被 AST 白名单禁止，但库函数内部可能有长耗时操作）和输出捕获（stdout 重定向、图表对象提取）。

## 自动建模

用户指定目标列后，系统自动完成建模全流程：

```python
def auto_model(df, target_col):
    # 1. 任务类型判定：分类 or 回归
    task = "classification" if df[target_col].nunique() < 20 else "regression"

    # 2. 预处理流水线
    #    - 数值列：中位数填充 + 标准化
    #    - 类别列：众数填充 + OneHot/Label编码
    #    - SafeLabelEncoder：未见过标签返回-1，不抛异常

    # 3. 多模型对比
    models = {
        "逻辑回归": LogisticRegression(),
        "随机森林": RandomForestClassifier(),
        "梯度提升": GradientBoostingClassifier(),
    }

    # 4. 交叉验证选最优
    for name, model in models.items():
        score = cross_val_score(model, X, y, cv=5)
        results[name] = score.mean()

    # 5. 返回最优模型 + 评估指标
    return best_model, metrics
```

`SafeLabelEncoder` 是一个工程细节亮点：对未见过的新标签返回 -1 而非抛异常，保证模型在新数据上不崩溃。

## 一键报告导出

系统支持一键导出 Word 分析报告，包含：
- 数据概览（行列数、数据类型、缺失率）
- 质量体检（异常值、重复行）
- 分析留痕（每步分析的代码、结论、图表）
- 建模结论（最优模型、特征重要性、评估指标）

## 技术栈

- **GUI**：CustomTkinter（现代化深色主题桌面端）
- **LLM 接入**：DeepSeek / OpenAI 兼容接口，urllib 零第三方依赖
- **数据**：pandas / numpy
- **可视化**：matplotlib（中文渲染已处理）
- **持久化**：SQLite（项目与历史记录）
- **报告**：python-docx

## 工程要点总结

1. **三元 Agent 闭环**是可靠性的核心——规划、执行、校验分离，校验不通过自动重做
2. **多候选择优**让分析更全面——同一问题多角度生成方案，执行后择优
3. **规则模板回落**是可用性底线——LLM 失败仍有结果
4. **四道闸门沙箱**是安全核心——AST 白名单 + 名称黑名单 + 属性过滤 + 受限命名空间
5. **SafeLabelEncoder** 处理新标签——返回 -1 而非崩溃

代码已开源：[github.com/lkq-dyq/DataMind](https://github.com/lkq-dyq/DataMind)
