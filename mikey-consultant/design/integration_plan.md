# Mikey Skill Integration Plan

> **文档性质：** 集成设计方案（Integration Design）——定义 5 个 Knowledge Completion 系统如何与现有 Skill 架构集成。
> **角色：** SKILL 架构师
> **版本：** v1.0 | 2026-08-02
> **约束：** 仅输出设计。不修改任何已有文件。不生成代码。
> **上游：** Skill v4.0（Gate -1 ~ Gate 5b + Player Model + Campaign Planner + Coach Decision Policy）
> **下游：** 5 个 Knowledge Completion 系统

---

## 集成总览图

```
                    Knowledge Systems → Gate Integration Map

    User Input
        │
        ▼
    ┌─────────────────────────────────────────────┐
    │ Gate -1: Player Model (加载玩家上下文)         │
    │   ← Player State                             │
    └──────────────┬──────────────────────────────┘
                   │
    ┌──────────────▼──────────────────────────────┐
    │ Gate 0: 信息充分性                            │
    │   ← Relationship Pipeline (阶段定位)          │
    └──────────────┬──────────────────────────────┘
                   │
    ┌──────────────▼──────────────────────────────┐
    │ Gate 1: Pattern Recognition                 │
    │   ← Window System (窗口信号→模式匹配)         │
    │   ← Subcommunication (潜沟通信号→模式匹配)     │
    │   ← Relationship Pipeline (阶段→模式范围)     │
    └──────────────┬──────────────────────────────┘
                   │
    ┌──────────────▼──────────────────────────────┐
    │ Gate 2: 根因定位                              │
    │   ← Window System (窗口变化→根因推断)         │
    │   ← Subcommunication (潜沟通暴露的真实状态)    │
    │   ← Relationship Pipeline (阶段错位→根因)     │
    │   ← Text Game (聊天阶段错误→根因)              │
    └──────────────┬──────────────────────────────┘
                   │
    ┌──────────────▼──────────────────────────────┐
    │ Gate 3: 理论覆盖检查                           │
    │   ← (现有六维矩阵，不受影响)                    │
    └──────────────┬──────────────────────────────┘
                   │
    ┌──────────────▼──────────────────────────────┐
    │ Gate 4: 反迎合检查 ★不可跳过                   │
    │   ← Window System (窗口误判检测)               │
    │   ← Subcommunication (用户自我呈现 vs 实际)    │
    └──────────────┬──────────────────────────────┘
                   │
    ┌──────────────▼──────────────────────────────┐
    │ Gate 5: 行动设计                              │
    │   ← Window Decision Matrix (窗口→许可/禁止)   │
    │   ← Escalation System (升级阶梯→升级建议)      │
    │   ← Text Game (聊天阶段→聊天策略)              │
    │   ← Relationship Pipeline (阶段→行动优先级)    │
    │   ← Subcommunication (状态建议)                │
    └──────────────┬──────────────────────────────┘
                   │
    ┌──────────────▼──────────────────────────────┐
    │ Gate 5b: Campaign Planner                   │
    │   ← Relationship Pipeline (模板匹配+阶段转换)  │
    │   ← Window System (检查点设置)                │
    │   ← Escalation System (升级节奏控制)           │
    │   ← Text Game (聊天阶段规划)                   │
    └─────────────────────────────────────────────┘
```

---

## 系统 1：Relationship Pipeline

### ① 应该在哪一个 Gate 调用？

| Gate | 调用方式 | 优先级 |
|------|---------|--------|
| **Gate 0** | 作为"关系背景"的结构化框架——用 Pipeline 阶段（0-10）替代当前的模糊"关系阶段"判断 | ★★★ 主调用点 |
| **Gate 1** | 限定 Pattern 搜索范围——当前阶段只匹配该阶段相关的 Pattern | ★★★ |
| **Gate 2** | 阶段错位检测——用户在 Stage X 做了 Stage Y 的事→根因定位 | ★★ |
| **Gate 5** | 行动设计的阶段基线——当前阶段决定行动优先级（止损>状态>策略） | ★★ |
| **Gate 5b** | Campaign 模板匹配——根据当前阶段和目标阶段选择 T1-T6 | ★★★ |

### ② 输入是什么？

- **Gate 0**：用户描述的行为序列
- 输入格式：`{ 当前阶段: Stage X, 互动历史: [...], 最近事件: {...} }`
- 从用户描述中提取：认识方式、当前互动方式、最近一次互动的具体行为

### ③ 输出是什么？

- **阶段定位**：`{ stage_id: 0-10, stage_name: string, confidence: 0-1 }`
- **阶段错位标记**：`{ stage_mismatch: true/false, expected_stage_behaviors: [...], actual_behaviors: [...] }`
- **升级/降级信号**：`{ upgrade_signals: [...], downgrade_signals: [...] }`

### ④ 会影响哪些 Pattern？

| 影响类型 | 具体变更 |
|---------|---------|
| **范围限定** | Gate 1 匹配时，根据当前阶段只检索该阶段相关 Pattern（早期 E01-E11 / 中期 M01-M12 / 长期 L01-L10），减少误匹配 |
| **新增匹配维度** | Pattern 匹配增加"阶段匹配度"权重——阶段越匹配，置信度越高 |
| **具体 Pattern 影响** | E02（聊天降温）现在可以通过阶段定位更精确判断——是正在 Stage 3（聊天升温）的降温还是 Stage 5（约会后）的降温？含义完全不同 |

### ⑤ 会影响哪些 Coach Decision？

| 决策点 | 变更 |
|--------|------|
| **快速/标准/深度通道选择** | 当检测到阶段严重错位（如 Stage 2 就做了 Stage 8 的事），强制至少标准通道 |
| **Gate 5 单步 vs Campaign 分流** | 当检测到用户需要跨越 2+ 个阶段（如 Stage 2→Stage 5），自动倾向 Campaign |
| **信息追问策略** | Gate 0 追问时，根据当前阶段问更有针对性的问题 |

### ⑥ 会影响哪些 Campaign Planner？

| Campaign 模板 | 变更 |
|--------------|------|
| **T1 渐进升温** | Phase 设计基于当前阶段→目标阶段的路径（不再从零设计） |
| **T2 长周期修复** | 修复路径基于阶段回退的距离（回退阶段多→修复周期长） |
| **T3 新手训练** | 训练阶段基于 Pipeline Stage 0→1→2→3 的递进 |
| **T4 决胜局** | 确认当前阶段为 Stage 5 或以上才能激活 |
| **T5 TD后防守** | 确认刚完成 Stage 9，触发 Stage 10 的防守逻辑 |
| **新增 T7** | 阶段加速（用户多阶段并行推进的场景，如 Stage 0 和 Stage 1 同时进行） |

### ⑦ 哪些 Player Model 字段需要新增？

```json
{
  "pipeline_progress": {
    "current_stage": "int (0-10)",
    "highest_stage_reached": "int",
    "stage_history": [
      { "stage": "int", "entered_at": "date", "exited_at": "date", "duration_days": "int" }
    ],
    "stage_stuck_count": {
      "stage_1": "int",  // 在 Stage 1 卡住的次数
      "stage_3": "int",  // 在聊天升温卡住的次数
      // ...
    },
    "common_stage_jumps": ["string"],  // 如 ["跳过Stage3直接Stage4"] 等越阶模式
    "stage_regression_triggers": ["string"]  // 导致阶段回退的常见原因
  }
}
```

### ⑧ 哪些已有逻辑需要删除？

- Gate 0 中当前的"快速判断关系阶段（陌生/熟悉/暧昧/亲密/长期）"→ 替换为 Pipeline 的 11 阶段精确定位
- Gate 1 中"提取关键信号"的 5 个维度保持不变，但需要增加"阶段"维度作为匹配权重

### ⑨ 哪些已有逻辑需要修改？

| 位置 | 当前逻辑 | 修改为 |
|------|---------|--------|
| Gate 0 最低信息集 | "关系背景（什么关系、什么阶段）" | "关系背景（Pipeline Stage 0-10 + 该阶段的进入方式）" |
| Gate 1 步骤 1 | "快速判断关系阶段（陌生/熟悉/暧昧/亲密/长期）" | "精确定位 Pipeline Stage（0-10），记录 confidence" |
| Gate 5 行动优先级 | 止损→状态→策略 | 增加"阶段复位"——如果检测到阶段错位，第一步行动是回到正确阶段 |

---

## 系统 2：Window System

### ① 应该在哪一个 Gate 调用？

| Gate | 调用方式 | 优先级 |
|------|---------|--------|
| **Gate 1** | 窗口信号→模式匹配：女生的绿灯/黄灯/红灯信号直接映射到 Pattern 的信号权重 | ★★★ 主调用点 |
| **Gate 2** | 窗口变化→根因推断：绿灯→黄灯（暴露需求感）、黄灯→红灯（一致性断裂） | ★★★ |
| **Gate 4** | 反迎合检查：用户自述的窗口（"她肯定喜欢我"）vs Window System 客观信号 | ★★★ |
| **Gate 5** | Window Decision Matrix → 行动许可：当前窗口状态下什么允许、什么禁止 | ★★★ 主调用点 |
| **Gate 5b** | Campaign 检查点设置基于窗口状态判断 | ★★ |

### ② 输入是什么？

- **Gate 1**：用户描述中的女生行为序列（回复内容/频率/主动性/线下行为）
- 输入格式：`{ signals: [{ type: string, value: string, source: "user_report"|"user_quote" }], context: { stage: int, recent_event: string } }`

### ③ 输出是什么？

- **窗口状态**：`{ window: "green"|"yellow"|"red", confidence: 0-1, subtype: "true_green"|"fake_green"|"ASD_yellow"|"true_yellow"|"hard_red"|"soft_red" }`
- **窗口变化方向**：`{ direction: "opening"|"closing"|"stable", rate: "fast"|"slow" }`
- **窗口误判风险**：`{ misjudgment_risk: "high"|"medium"|"low", likely_misread_as: "green"|"yellow"|"red" }`
- **Decision Matrix 查询结果**：`{ allowed: [...], forbidden: [...], conditional: [...] }`

### ④ 会影响哪些 Pattern？

| Pattern | 影响 |
|---------|------|
| **E01-E11（早期）** | 每个 Pattern 增加"典型窗口状态"字段 |
| **M01-M12（中期）** | 窗口变化作为 Pattern 的"动态属性"——同 Pattern 在不同窗口下处理不同 |
| **X01（自欺驱动）** | 增加窗口误判检测——"用户声称绿灯 + Window System 判断红灯 = 高置信度 X01" |
| **X04（零压力成功）** | 增加窗口确认——持续绿灯 = X04 的核心特征 |

### ⑤ 会影响哪些 Coach Decision？

| 决策点 | 变更 |
|--------|------|
| **信息追问** | 当窗口信号矛盾时→追加追问（"她最近一次主动找你是什么时候？"） |
| **通道升级** | 自欺（用户声称绿灯但 Window System 判断红灯）→强制深度通道 |
| **Gate 5 行动设计** | 行动前必须查询 Window Decision Matrix——红灯下不允许任何升级行动，黄灯下只允许试探 |

### ⑥ 会影响哪些 Campaign Planner？

| Campaign | 变更 |
|----------|------|
| **所有 Campaign** | 每个 Phase 的检查点增加"窗口状态检查" |
| **T1 渐进升温** | Phase 转换条件 = 窗口从黄灯→绿灯 |
| **T2 长周期修复** | 修复目标 = 窗口从红灯→黄灯→绿灯 |
| **T4 决胜局** | 激活前提 = 窗口确认为绿灯 |
| **T5 TD后防守** | 核心监控对象 = 窗口变化（TD后绿灯是否保持） |
| **Abandoned 条件** | 增加：连续 2 次 Phase 检查点检测到窗口关闭 |

### ⑦ 哪些 Player Model 字段需要新增？

```json
{
  "window_judgment": {
    "accuracy_rate": "float (0-1)",  // 历史窗口判断正确率
    "common_misjudgments": [
      { "type": "ASD_as_rejection", "count": "int" },
      { "type": "politeness_as_interest", "count": "int" },
      { "type": "fake_window_as_real", "count": "int" }
    ],
    "last_window_state": "green|yellow|red",
    "window_change_events": [
      { "from": "string", "to": "string", "trigger": "string", "date": "date" }
    ]
  },
  "behavior_patterns": {
    "window_anxiety": {  // 新增
      "description": "看到黄灯就恐慌/放弃",
      "severity": "1-5",
      "occurrences": "int"
    },
    "window_aggression": {  // 新增
      "description": "红灯下仍然强推",
      "severity": "1-5",
      "occurrences": "int"
    }
  }
}
```

### ⑧ 哪些已有逻辑需要删除？

无直接删除。Window System 是新增判断层次，不替代任何已有逻辑。

### ⑨ 哪些已有逻辑需要修改？

| 位置 | 当前逻辑 | 修改为 |
|------|---------|--------|
| Gate 1 信号提取 | 提取 5 个关键信号（回复主动性/趋势/邀约成功率/一致性/情绪） | 增加第 6 个信号维度："窗口状态（基于 Window System）" |
| Gate 2 根因路径 | 5 条固定路径 | 增加窗口变化作为根因定位的辅助证据（如"趋势降温 + 窗口从绿变黄 = 需求感暴露"） |
| Gate 5 止损 | "停止当前的错误行为" | 增加"红灯下：停止一切升级行为（参考 Window Decision Matrix 禁止列）" |
| Gate 5b Phase 检查点 | 当前无明确的窗口检查 | 每个 phase 结束时检查窗口状态变化 |

---

## 系统 3：Text Game Methodology

### ① 应该在哪一个 Gate 调用？

| Gate | 调用方式 | 优先级 |
|------|---------|--------|
| **Gate 0** | 判断用户在 Text Game 生命周期中的位置（加微信/破冰/舒适/吸引/窗口确认/邀约/约会前/约会后） | ★★★ |
| **Gate 2** | 根因定位——用户在错误的聊天阶段做了错误的事（如 Stage 2 破冰就猛攻） | ★★★ |
| **Gate 5** | 聊天策略——根据当前聊天阶段给出对应的行为原则/频率/节奏 | ★★★ 主调用点 |

### ② 输入是什么？

- 用户提供的聊天内容/频率/对方回复模式
- 输入格式：`{ chat_stage: int (1-8), user_behavior: { reply_freq: string, initiative_freq: string, content_type: string }, her_behavior: { reply_freq: string, initiative: bool, content_quality: string } }`

### ③ 输出是什么？

- **聊天阶段定位**：`{ stage: 1-8, stage_name: string, previous_stage: int }`
- **阶段错误检测**：`{ wrong_stage_behaviors: [...], correct_stage_behaviors: [...] }`
- **聊天策略建议**：`{ principles: [...], reply_freq: string, initiative_freq: string, upgrade_condition: string, downgrade_condition: string, forbidden: [...] }`

### ④ 会影响哪些 Pattern？

| Pattern | 影响 |
|---------|------|
| **E02（聊天降温）** | 现在可以精确判断是在 Text Game 的哪个阶段降温——Stage 2→3 的降温 vs Stage 4→5 的降温，根因和策略完全不同 |
| **E05（对方不主动）** | 可以判断是 Stage 2 的正常破冰期还是在 Stage 4 的吸引建立失败 |
| **E06（备胎化）** | 可以检测用户是否卡在 Stage 3（建立舒适）永远不进入 Stage 4（建立吸引） |

### ⑤ 会影响哪些 Coach Decision？

| 决策点 | 变更 |
|--------|------|
| **Gate 5 单步 vs Campaign** | 如果在 Text Game Stage 3 卡住超过 2 周→倾向激活 Campaign T3（新手训练） |
| **行动建议粒度** | 从之前的"聊什么"→升级为"当前阶段的行为原则 + 频率 + 禁止行为" |

### ⑥ 会影响哪些 Campaign Planner？

| Campaign | 变更 |
|----------|------|
| **T1 渐进升温** | Phase 设计基于 Text Game 生命周期——Stage 2→3→4→5→6→7 |
| **T3 新手训练** | 训练内容按 Text Game 阶段组织（先学破冰→再学吸引→再学邀约） |
| **新增 Phase 类型** | "聊天阶段复位"——当检测到用户在 Text Game 中越阶操作时 |

### ⑦ 哪些 Player Model 字段需要新增？

```json
{
  "text_game_profile": {
    "typical_stuck_stage": "int (1-8)",  // 最常卡住的聊天阶段
    "chat_initiative_pattern": "over_active|balanced|under_active|erratic",
    "reply_rhythm_pattern": "too_fast|too_slow|matched|erratic",
    "common_chat_mistakes": [
      { "stage": "int", "mistake": "string", "count": "int" }
    ],
    "five_iron_rules_violations": {
      "broken_defense": "int",       // 破防次数
      "heat_mismatch": "int",        // 热度不对齐次数
      "no_flirt_and_run": "int",     // 没有撩一下就跑
      "consistency_break": "int",    // 一致性断裂
      "display_issues": "int"        // 展示面问题
    }
  }
}
```

### ⑧ 哪些已有逻辑需要删除？

无直接删除。

### ⑨ 哪些已有逻辑需要修改？

| 位置 | 当前逻辑 | 修改为 |
|------|---------|--------|
| Gate 5 聊天建议 | 原则性建议（"不要破防""热度对齐"） | 增加"首先定位你的 Text Game 阶段→然后查询该阶段的行为原则" |
| Gate 2 根因 | "聊天方式错误"作为泛化根因 | 细化为"在 Text Game Stage X 做了 Stage Y 的事"作为具体根因 |
| Gate 5b T3 | "低压力练习→实战" | 按 Text Game 8 阶段组织（破冰练习→吸引练习→邀约练习→约会后维护） |

---

## 系统 4：Escalation System

### ① 应该在哪一个 Gate 调用？

| Gate | 调用方式 | 优先级 |
|------|---------|--------|
| **Gate 5** | 升级决策——查询当前 Rung + 前置条件 + 许可信号来决定是否可以升级 | ★★★ 主调用点 |
| **Gate 5b** | 升级节奏控制——Campaign 中每个 Phase 的升级步骤基于 Escalation Ladder | ★★★ |
| **Gate 2** | 升级错误→根因定位（施压过度=跳过共振；升级过慢=不敢测试窗口） | ★★ |

### ② 输入是什么？

- 用户当前互动场景 + 女生类型 + 窗口状态
- 输入格式：`{ current_rung: 1-7, girl_type: "introvert"|"extrovert"|"high_window"|"low_window", window: "green"|"yellow"|"red", recent_escalation_attempt: { rung: int, result: "accepted"|"rejected"|"ASD" } }`

### ③ 输出是什么？

- **当前 Rung 定位**：`{ rung: 1-7, rung_name: string }`
- **升级许可判断**：`{ can_escalate: bool, to_rung: int, confidence: 0-1 }`
- **升级策略**：`{ method: "test_window"|"escalate"|"de-escalate"|"hold", pressure_level: "light"|"medium"|"full", de-escalation_plan: {...} }`
- **女生类型差异应用**：`{ adjusted_strategy: string, girl_type: string, specific_notes: [...] }`

### ④ 会影响哪些 Pattern？

| Pattern | 影响 |
|---------|------|
| **M07（升级过快）** | Escalation System 提供精确的"过快"判断标准——跳过哪个 Rung 的前置条件 |
| **M08（升级过慢）** | 检测：在某个 Rung 停留过久（窗口存在但不测试） |
| **M09（ASD反应）** | 精确区分每个 Rung 的 ASD 表现和处理方式 |
| **M10（框架测试）** | 亲吻/转场 Rung 的框架测试与 Escalation System 对齐 |
| **M11（转场失败）** | 转场五大原则中的哪一条被违反 |

### ⑤ 会影响哪些 Coach Decision？

| 决策点 | 变更 |
|--------|------|
| **行动类型判断** | 当前不再是"止损/状态/策略"三选一，增加"升级测试/升级推进/降级减压"三选一 |
| **女生类型分流** | 升级策略根据女生四象限自动调整（内向=施压减半+眼神通道；高窗口=加速） |

### ⑥ 会影响哪些 Campaign Planner？

| Campaign | 变更 |
|----------|------|
| **T1 渐进升温** | Phase 结构基于 Escalation Ladder：Phase 1=Rung 1-3, Phase 2=Rung 4-5, Phase 3=Rung 6-7 |
| **T4 决胜局** | 当天流程设计基于 Rung 1→7 的加速通道 |
| **所有 Campaign** | Phase 行动增加"升级许可检查"——每个 Phase 开始前确认前置 Rung 是否完成 |

### ⑦ 哪些 Player Model 字段需要新增？

```json
{
  "escalation_profile": {
    "typical_pace": "too_fast|balanced|too_slow",
    "rung_stuck_points": [
      { "rung": "int", "count": "int" }
    ],
    "escalation_mistakes": {
      "pressure_overdose": "int",      // 施压过度次数
      "no_resonance_escalation": "int", // 跳过共振直接升级
      "missed_windows": "int",          // 错过窗口未测试
      "consistency_break_during_esc": "int"  // 升级过程中一致性断裂
    },
    "girl_type_adaptation": {
      "introvert_handled": "int",
      "extrovert_handled": "int",
      "high_window_handled": "int",
      "low_window_handled": "int"
    },
    "last_escalation_result": {
      "rung": "int",
      "result": "string",
      "girl_type": "string",
      "date": "date"
    }
  }
}
```

### ⑧ 哪些已有逻辑需要删除？

无直接删除。

### ⑨ 哪些已有逻辑需要修改？

| 位置 | 当前逻辑 | 修改为 |
|------|---------|--------|
| Gate 5 单步行动 | "策略行动（状态调整后）→框架工程/具体行动" | 细化："升级策略=Escalation System 查询当前 Rung→判断是否可升级→选择测试/推进/降级" |
| Gate 5b Campaign Phase | 每个 Phase 只有一个通用 action | 每个 Phase 增加"升级步骤"——基于 Escalation Ladder 的 Rung 推进计划 |
| Gate 2 根因"施压过度" | 模糊判断 | 精确判断：在哪个 Rung 施压过度？跳过了哪个前置条件？ |
| Coach Decision Policy | "止损类/即时纠正/简单二选一"→Gate 5 单步 | 增加一条："升级决策类（该不该亲/该不该转场）→Gate 5 单步 + Escalation System 查询" |

---

## 系统 5：Sub-communication System

### ① 应该在哪一个 Gate 调用？

| Gate | 调用方式 | 优先级 |
|------|---------|--------|
| **Gate 1** | 潜沟通信号→模式匹配——用户描述的"他的状态"映射到弱行为线索 Pattern | ★★ |
| **Gate 2** | 根因定位——用户的弱行为线索（语速快/小动作/眼神飘）作为根因的底层证据 | ★★★ |
| **Gate 4** | 反迎合——用户自述"我很自信"vs 行为中暴露的弱行为线索 | ★★★ 主调用点 |
| **Gate 5** | 行动建议——不只给"做什么"，也给"用什么状态做" | ★★★ |

### ② 输入是什么？

- 用户对自身状态的描述 + 可观察的行为线索
- 输入格式：`{ user_self_report: "string", observed_cues: [{ signal_type: "eye_contact"|"stance"|"rhythm"|..., cue_value: "strong"|"weak", source: "user_stated"|"inferred" }] }`

### ③ 输出是什么？

- **行为线索评估**：`{ overall: "strong"|"weak"|"mixed", breakdown: { eye_contact: "strong"|"weak", stance: "strong"|"weak", ... } }`
- **一致性检查**：`{ consistent: bool, inconsistency_points: [{ front_behavior: string, back_behavior: string, severity: "high"|"medium" }] }`
- **状态修正建议**：`{ cue_to_fix: string, how_to_fix: string, why: string }`

### ④ 会影响哪些 Pattern？

| Pattern | 影响 |
|---------|------|
| **E03（搭讪焦虑）** | 底层原因 = 弱行为线索（眼神飘/语速快/身体前倾） |
| **X01（自欺驱动）** | 增强检测——"用户声称自信 + Subcommunication 检测弱行为线索 = 最高置信度 X01" |
| **X03（表演型吸引）** | 核心 = 潜沟通层面的"装"→一致性断裂 |
| **X07（自我合理化）** | 用户把自己的弱行为线索解释为"正常"→Subcommunication System 标记为正确识别 |

### ⑤ 会影响哪些 Coach Decision？

| 决策点 | 变更 |
|--------|------|
| **深度通道触发** | 当 Subcommunication 检测到严重的弱行为线索 + 用户不自知→强制深度通道 |
| **行动建议类型** | 增加"状态层建议"——不改变行为，先改变潜沟通状态（慢下来、放松、眼神坚定） |

### ⑥ 会影响哪些 Campaign Planner？

| Campaign | 变更 |
|----------|------|
| **T2 长周期修复** | 修复内容增加"潜沟通重建"——从生活中培养强行为线索 |
| **T3 新手训练** | Phase 0 增加"潜沟通基线检测"——评估用户的弱行为线索模式 |
| **所有 Campaign** | 每个 Phase 的"成功标准"增加"潜沟通状态变化"维度 |

### ⑦ 哪些 Player Model 字段需要新增？

```json
{
  "subcommunication_profile": {
    "baseline": {
      "eye_contact": "strong|weak|mixed",
      "stance": "strong|weak|mixed",
      "speech_rhythm": "strong|weak|mixed",
      "volume": "strong|weak|mixed",
      "smile_type": "natural|forced|mixed",
      "touch_decisiveness": "decisive|hesitant|mixed",
      "space_occupation": "expansive|contracted|mixed"
    },
    "weak_cue_patterns": [
      { "cue": "string", "trigger_context": "string", "frequency": "high|medium|low" }
    ],
    "consistency_breaks": [
      { "front": "string", "back": "string", "context": "string", "date": "date" }
    ],
    "improvement_trajectory": {
      "weeks_tracked": "int",
      "cue_changes": [
        { "cue": "string", "from": "string", "to": "string", "weeks": "int" }
      ]
    }
  }
}
```

### ⑧ 哪些已有逻辑需要删除？

无直接删除。

### ⑨ 哪些已有逻辑需要修改？

| 位置 | 当前逻辑 | 修改为 |
|------|---------|--------|
| Gate 1 信号提取 | 提取 5 个行为信号 | 增加第 7 个信号维度："用户潜沟通状态（基于 Subcommunication System）" |
| Gate 2 根因 | 5 条固定路径 | 每条根因路径增加"潜沟通证据层"——弱行为线索是根因的底层证据 |
| Gate 4 反迎合检查 1 | "用行为数据验证" | 增加"用 Subcommunication System 的强/弱行为线索框架验证用户自我陈述" |
| Gate 5 行动建议 | 只给"做什么" | 增加"用什么状态做"——同步修改潜沟通层面的行为线索 |
| 输出层——快速通道 | "判断+变量+下一步" | "判断+变量+下一步+状态提示（如：语速放慢三倍）" |

---

## 跨系统集成冲突与协调

### 冲突点 1：Gate 1 的复杂度膨胀

**问题：** 当前 Gate 1 提取 5 个信号→匹配 Pattern。集成后信号维度增加到 9+（Pipeline阶段/窗口/潜沟通/聊天阶段等），匹配复杂度翻倍。

**协调方案：**
- **信号分层**：将信号分为"一级信号"（必须提取）和"二级信号"（条件提取）
  - 一级：关系阶段（Pipeline）+ 窗口状态（Window）+ 趋势方向（原有）
  - 二级：潜沟通状态（Subcommunication）+ 聊天阶段（Text Game）——仅在特定 Pattern 触发时提取
- **快速通道降维**：快速通道只使用一级信号

### 冲突点 2：Gate 5 行动建议的决策树膨胀

**问题：** 当前 Gate 5 输出"止损/状态/策略/退出"。集成后需要查询 4 个系统（Window Decision Matrix + Escalation + Text Game + Stage），输出项可能冲突。

**协调方案：**
- **决策优先级**：Window（窗口状态→最底层约束） > Pipeline（阶段→行动范围） > Escalation（升级建议） > Text Game（聊天策略）
- **冲突解决规则**：
  - Window 说红灯禁止→其他所有系统建议的升级行动全部禁止
  - Pipeline 说需要在 Stage 3 → Escalation 的 Rung 限制在 1-3
  - Text Game 的聊天原则与其他系统不矛盾（它们是不同层面的行动）

### 冲突点 3：Player Model 字段数量

**问题：** 5 个系统各新增 5-10 个 Player Model 字段，`player_state.json` 膨胀。

**协调方案：**
- **分层存储**：新增字段单独存储在各系统的 profile 子对象中，不扁平化
- **按需加载**：Gate -1 不加载全部子对象，只加载最近 3 次咨询涉及的系统 profile
- **归档策略**：超过 90 天未更新的子对象标记为 archived，不加载到上下文

---

## 实施优先级建议

| 优先级 | 系统 | 理由 |
|--------|------|------|
| **P0** | Window System → Gate 5 | 行动设计的最底层约束，任何升级决策必须查询窗口 |
| **P0** | Pipeline → Gate 0 & Gate 1 | 阶段定位是所有判断的基础，替代当前的模糊阶段判断 |
| **P1** | Escalation → Gate 5 | 升级行动的具体建议层，依赖 Window 和 Pipeline 先到位 |
| **P1** | Text Game → Gate 5 | 聊天策略建议，依赖 Pipeline 的阶段定位 |
| **P2** | Subcommunication → Gate 4 | 反迎合增强，是锦上添花（Gate 4 本身不可跳过，Subcommunication 让它更精确） |
| **P2** | 所有系统 → Player Model | 数据字段新增，依赖各系统在主逻辑中先落地 |

**建议实施顺序：** Pipeline + Window（第一轮）→ Escalation + Text Game（第二轮）→ Subcommunication + Player Model 字段（第三轮）

---

## 总结：5 系统集成后的 Gate 调用图

```
Gate -1: Player Model
    ├── 加载: pipeline_progress, window_judgment, text_game_profile, 
    │         escalation_profile, subcommunication_profile
    └── 输出: 上下文注入后续 Gate

Gate 0: 信息充分性
    ├── 调用: Relationship Pipeline（阶段定位）
    └── 输出: stage_id + stage_mismatch

Gate 1: Pattern Recognition
    ├── 调用: Window System（窗口信号→模式匹配权重）
    ├── 调用: Subcommunication（潜沟通信号→模式匹配）
    ├── 调用: Relationship Pipeline（阶段→模式范围限定）
    └── 输出: matched_patterns + window_state + stage

Gate 2: 根因定位
    ├── 调用: Window System（窗口变化→根因推断）
    ├── 调用: Subcommunication（弱行为线索→根因证据）
    ├── 调用: Relationship Pipeline（阶段错位→根因）
    ├── 调用: Text Game（聊天阶段错误→根因）
    └── 输出: root_cause(s) + evidence_chain

Gate 3: 理论覆盖检查
    └── 不受影响（六维矩阵 + Knowledge Graph）

Gate 4: 反迎合检查
    ├── 调用: Window System（窗口误判检测）
    ├── 调用: Subcommunication（用户自我呈现 vs 实际行为线索）
    └── 输出: compliance_report + misjudgment_flags

Gate 5: 行动设计
    ├── 调用: Window Decision Matrix（许可/禁止矩阵查询）
    ├── 调用: Escalation System（升级阶梯→升级策略）
    ├── 调用: Text Game（聊天阶段→聊天策略）
    ├── 调用: Relationship Pipeline（阶段→行动优先级）
    ├── 调用: Subcommunication（状态层建议）
    └── 输出: action_plan + state_adjustment + escalation_strategy

Gate 5b: Campaign Planner
    ├── 调用: Relationship Pipeline（模板匹配+阶段转换）
    ├── 调用: Window System（检查点设置）
    ├── 调用: Escalation System（升级节奏）
    ├── 调用: Text Game（聊天阶段规划）
    └── 输出: campaign_plan (phases + checkpoints + exit_conditions)
```

---

*设计完成。不修改任何已有文件。不生成代码。v1.0 | 2026-08-02。*
