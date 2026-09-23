---
title: "小区宠物智能管理监控系统：从品种识别到风险预测的全链路实践"
slug: pet-recognition-monitoring-system
subtitle: 秦梓恒的技术笔记
tags: deep-learning,computer-vision
publishedAt: 2026-09-21T09:00:00Z
saveAsDraft: false
enableToc: true
hideFromCommunity: false
canonical: https://lkq-dyq.github.io/deep-learning/computer-vision/2026/09/21/pet-recognition-monitoring-system.html
---

## 背景

随着城市社区宠物数量增长，物业管理面临三大痛点：
1. **品种识别难** — 巡查人员无法快速判断宠物品种，难以执行品种管理规定
2. **行为监控难** — 散养、扰民、伤人等事件缺乏实时记录与追溯
3. **主人管理难** — 不文明养宠行为（不牵绳、不清理粪便）难以取证

本文介绍"小区宠物智能管理监控系统"的设计与实现，覆盖品种识别、行为识别、管理监控、风险预测、主人行为细观分析五大功能模块。

## 系统架构

### 功能矩阵

```
┌─────────────────────────────────────────┐
│        小区宠物智能管理监控系统          │
├──────────┬──────────┬───────────────────┤
│  宠物识别  │  行为识别  │  管理监控         │
│  品种分类  │  散养检测  │  实时事件流       │
│  特征提取  │  姿态识别  │  风险态势         │
├──────────┴──────────┴───────────────────┤
│        风险预测 + 主人行为细观分析         │
└─────────────────────────────────────────┘
```

### 技术栈

- 深度学习（宠物品种分类模型）
- SQLite（数据持久化）
- PIL（界面渲染与截图生成）
- 实时事件流（WebSocket 推送）

## 宠物品种识别

### 品种覆盖

系统支持 12 种常见猫品种的自动识别：

```python
CAT_BREEDS = {
    "abyssinian",           # 阿比西尼亚猫
    "bengal",               # 孟加拉猫
    "birman",               # 伯曼猫
    "bombay",               # 孟买猫
    "british_shorthair",    # 英国短毛猫
    "egyptian_mau",         # 埃及猫
    "maine_coon",           # 缅因猫
    "persian",              # 波斯猫
    "ragdoll",              # 布偶猫
    "russian_blue",         # 俄罗斯蓝猫
    "siamese",              # 暹罗猫
    "sphynx",               # 斯芬克斯猫
}
```

### 识别流程

```
摄像头抓帧 → 预处理（缩放/归一化）→ 深度学习模型推理 → 品种分类 + 置信度
                                                          ↓
                                                    置信度 > 阈值 → 入库
                                                    置信度 < 阈值 → 标记"待人工确认"
```

### 数据存储

识别结果与原始数据均存入 SQLite，支持追溯：

```python
import sqlite3

conn = sqlite3.connect("data/pet_monitor.db")
# pets 表：品种、置信度、首次识别时间、最近出现位置
# events 表：事件类型、时间戳、关联宠物ID、关联主人ID
# owners 表：主人信息、风险评分、行为记录
```

## 行为识别

### 姿态识别

通过分析宠物的姿态关键点，判断当前行为状态：
- **行走/奔跑** — 正常运动
- **静止/卧伏** — 休息
- **异常姿态** — 可能受伤或生病

### 散养检测

当识别到宠物在公共区域无牵引绳时，触发散养事件：

```python
def detect_free_roaming(pet_detection, leash_detection):
    """散养检测：有宠物无牵引绳"""
    if pet_detection and not leash_detection:
        trigger_event(
            event_type="free_roaming",
            severity="medium",
            pet_id=pet_detection.id,
            location=pet_detection.location,
        )
```

## 管理监控

### 实时事件流

系统维护一个实时事件流，按时间顺序展示所有检测事件：

```
[09:32:15] 品种识别  布偶猫  置信度 0.94  3号楼东侧
[09:35:08] 散养检测  布偶猫  无牵引绳  3号楼东侧
[09:35:12] 风险升级  布偶猫  中风险  主人: 张某某
[09:38:30] 主人行为  不清理粪便  3号楼花坛
```

### 风险态势

综合宠物风险与主人行为，给出小区宠物管理的整体风险态势：

```python
def assess_risk(pet_records, owner_records):
    """风险态势评估"""
    risk_score = 0
    for pet in pet_records:
        if pet.free_roaming_count > 3:
            risk_score += 20  # 多次散养
        if pet.aggression_flag:
            risk_score += 30  # 攻击性标记
    for owner in owner_records:
        if owner.feces_not_cleaned > 2:
            risk_score += 15  # 不清理粪便
        if owner.no_leash_count > 3:
            risk_score += 15  # 多次不牵绳
    return min(risk_score, 100)
```

## 主人行为细观分析

系统不仅监控宠物，还监控主人行为模式：

| 行为类型 | 检测方式 | 风险权重 |
|----------|----------|----------|
| 不牵绳 | 宠物检测 + 牵引绳检测 | 中 |
| 不清理粪便 | 行为识别 + 区域检测 | 中 |
| 高峰时段遛宠 | 时间 + 位置分析 | 低 |
| 违规养禁养品种 | 品种识别 + 法规比对 | 高 |

### 主人风险画像

```python
def build_owner_profile(owner_id):
    """构建主人风险画像"""
    events = get_owner_events(owner_id)
    profile = {
        "total_events": len(events),
        "risk_level": "low",  # low / medium / high
        "behavior_tags": [],
        "trend": "stable",   # improving / stable / worsening
    }
    # 统计各类型行为
    behavior_counts = defaultdict(int)
    for e in events:
        behavior_counts[e.type] += 1
    # 风险等级判定
    if behavior_counts["no_leash"] > 3 or behavior_counts["aggression"] > 0:
        profile["risk_level"] = "high"
    elif behavior_counts["no_leash"] > 1 or behavior_counts["feces"] > 2:
        profile["risk_level"] = "medium"
    # 趋势分析
    recent = events[-10:]
    earlier = events[-20:-10] if len(events) > 20 else events[:10]
    if len(recent) < len(earlier) * 0.5:
        profile["trend"] = "improving"
    return profile
```

## 界面设计

系统采用三栏式布局，配色对齐前端设计变量：

```python
# 设计变量
BG = (244, 246, 251)       # 背景灰
CARD = (255, 255, 255)      # 卡片白
INK = (31, 39, 51)          # 主文字
PRIMARY = (47, 109, 240)    # 主题蓝
LOW = (31, 170, 89)         # 低风险绿
MID = (232, 133, 26)        # 中风险橙
HIGH = (226, 59, 59)        # 高风险红
PET = (59, 125, 216)        # 宠物识别蓝
POSE = (10, 156, 156)       # 姿态识别青
```

- **左栏**：实时识别分析 + 识别结果（品种、置信度、位置）
- **右栏上**：风险态势 + 实时事件流
- **右栏下**：主人行为细观分析

## 总结

小区宠物智能管理监控系统从三个层面解决社区宠物管理问题：

1. **识别层** — 品种识别 + 行为识别 + 姿态识别，构建感知能力
2. **管理层** — 实时事件流 + 风险态势 + 主人画像，构建决策能力
3. **预测层** — 风险预测 + 趋势分析，构建预警能力

核心技术亮点：品种识别覆盖 12 种常见猫品种，主人行为细观分析从"管宠物"延伸到"管主人"，风险态势综合评估提供小区整体视图。

代码已开源：[github.com/lkq-dyq/PetRecognition](https://github.com/lkq-dyq/PetRecognition)
