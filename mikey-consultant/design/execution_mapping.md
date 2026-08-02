# Mikey Skill Execution Mapping

> **文档性质：** 执行映射（Execution Mapping）——基于 Integration Plan，逐 Gate 定义修改内容、原因、系统调用、输入输出。
> **角色：** SKILL 架构师
> **版本：** v1.0 | 2026-08-02
> **约束：** 仅输出设计。不修改任何已有文件。
> **上游：** `design/integration_plan.md` + Skill v4.0 SKILL.md

---

## Gate -1：Player Model（玩家上下文加载）

### 新增

**新增 5 个 profile 子对象的加载逻辑：**

| 新增内容 | 原因 |
|---------|------|
| `pipeline_progress` 加载 | 需要跨会话追踪用户的 Pipeline 阶段位置，实现阶段连续性感知 |
| `window_judgment` 加载 | 需要追踪用户的历史窗口判断准确率，为 Gate 4 反迎合提供基线 |
| `text_game_profile` 加载 | 需要知道用户常卡在哪个聊天阶段，为 Gate 5 聊天策略提供个性化 |
| `escalation_profile` 加载 | 需要知道用户的升级节奏偏好（太快/太慢），为 Gate 5 升级建议校准 |
| `subcommunication_profile` 加载 | 需要知道用户的弱行为线索模式，为 Gate 2 根因和 Gate 4 反迎合提供底层证据 |

**新增"按需加载"逻辑：**
- 只加载最近 3 次咨询涉及的系统 profile
- 超过 90 天未更新的子对象标记 `archived`，不加载

### 调用

| 系统 | 调用内容 |
|------|---------|
| **Relationship Pipeline** | 读取 `pipeline_progress.current_stage` 注入后续 Gate |
| **Window System** | 读取 `window_judgment.accuracy_rate` 和 `common_misjudgments` 注入 Gate 4 |
| **Text Game** | 读取 `text_game_profile.typical_stuck_stage` 注入 Gate 5 |
| **Escalation** | 读取 `escalation_profile.typical_pace` 注入 Gate 5 |
| **Subcommunication** | 读取 `subcommunication_profile.baseline` 注入 Gate 2 和 Gate 4 |

### 输出

```
{
  player_context: {
    pipeline: { current_stage: 0-10, highest_stage: 0-10, stage_stuck_points: [...] },
    window: { accuracy_rate: 0-1, common_misjudgments: [...] },
    text_game: { typical_stuck_stage: 1-8, five_rules_violations: {...} },
    escalation: { typical_pace: "fast"|"balanced"|"slow" },
    subcommunication: { baseline: {...}, weak_cue_patterns: [...] }
  }
}
```

---

## Gate 0：信息充分性

### 新增

**新增"阶段定位"步骤（在最低信息集检查之前执行）：**

| 新增内容 | 原因 |
|---------|------|
| `identify_pipeline_stage(user_input) → { stage_id, confidence }` | 替代当前的模糊"关系阶段"判断（陌生/熟悉/暧昧/亲密/长期），替换为精确的 Pipeline Stage 0-10 |
| `detect_stage_mismatch(stage_id, user_behaviors) → { mismatch, expected, actual }` | 检测用户是否在错误阶段做了错误的事——这是 Gate 2 根因定位的重要输入 |
| 新增最低信息维度：`pipeline_stage` | 关系背景需要更强的结构化——不只是"什么关系"，而是"Pipeline 哪个阶段 + 该阶段的关键行为" |

**新增"阶段错位"紧急标记：**
- 检测到严重阶段错位（如 Stage 2 做了 Stage 8 的事）→ 标记为 `HIGH_PRIORITY`，无论其他信息是否充分，都要进入 Gate 1
- 检测到用户卡在某一阶段超过预期时间 → 标记为 `STUCK`，提示 Gate 5b 可能需要 Campaign

### 调用

| 系统 | 调用内容 |
|------|---------|
| **Relationship Pipeline** | 查询 `Pipeline Stage → 该阶段的典型行为基线`，与用户实际行为对比 |
| **Text Game** | 如果阶段定位在 1-8（聊天相关），查询 Text Game 生命周期辅助定位 |

### 输出

```
{
  info_sufficiency: {
    has_background: bool,
    has_key_event: bool,
    has_behavior_sequence: bool,
    sufficiency_level: "sufficient"|"partial"|"insufficient"
  },
  stage_position: {
    pipeline_stage: { id: 0-10, name: string, confidence: 0-1 },
    text_game_stage: { id: 1-8, name: string, confidence: 0-1 } | null,
    stage_mismatch: { detected: bool, severity: "none"|"mild"|"severe", detail: string }
  },
  flags: {
    HIGH_PRIORITY: bool,    // 严重阶段错位
    STUCK: bool,            // 卡在某阶段超预期
    DEGRADED: bool          // 信息不足→降级输出
  }
}
```

---

## Gate 1：一眼判断（Pattern Recognition）

### 新增

**新增"一级信号 vs 二级信号"分层提取：**

| 新增内容 | 原因 |
|---------|------|
| 一级信号：Pipeline 阶段、窗口状态、趋势方向（原有） | 所有通道必须提取——这三个信号是 Pattern 匹配的基础维度 |
| 二级信号：潜沟通状态、聊天阶段、女生类型 | 仅在特定 Pattern 触发时提取——防止 Gate 1 复杂度膨胀 |
| 新增匹配维度：阶段匹配度 | 根据当前 Pipeline Stage，限定 Pattern 搜索范围（早期 E/中期 M/长期 L/跨阶段 X），该阶段外的 Pattern 不计入高匹配 |
| 新增匹配维度：窗口匹配度 | 将 Window System 输出的窗口状态映射到 Pattern 的信号权重 |

**新增"窗口信号→模式信号"转换层：**

| 窗口信号 | → | 模式信号 |
|---------|---|---------|
| 🟢 绿灯（持续） | → | 趋势方向=升温 / 回复主动性=高 |
| 🟡 黄灯（持续） | → | 趋势方向=稳定但不确定 / 需要更多信号 |
| 🔴 红灯（持续） | → | 趋势方向=降温 / 吸引力不存在 / 窗口关闭 |
| 绿→黄变化 | → | 趋势方向=降温 + 可能需求感暴露 |
| 黄→红变化 | → | 趋势方向=快速降温 + 可能一致性断裂 |

**新增"潜沟通信号→弱行为线索 Pattern"关联：**
- 用户语速快 + 小动作多 + 眼神飘 → 自动提升 E03（搭讪焦虑）匹配度
- 用户自述"我不焦虑"但潜沟通信号全是弱行为线索 → 自动提升 X01（自欺驱动）匹配度

### 调用

| 系统 | 调用内容 |
|------|---------|
| **Relationship Pipeline** | `query: { current_stage } → { relevant_pattern_range: "E"|"M"|"L"|"X", patterns_in_range: [...] }` |
| **Window System** | `query: { signals } → { window: "green"|"yellow"|"red", subtype, confidence }` |
| **Subcommunication** | `query: { user_behavior_descriptions } → { overall: "strong"|"weak"|"mixed", breakdown }` （仅二级信号触发） |
| **Text Game** | `query: { user_stage } → { stage_position }` （仅当窗口状态为黄灯且涉及聊天时触发） |

### 输出

```
{
  pattern_matches: [
    {
      pattern_id: "string",
      pattern_name: "string",
      match_score: 0-1,
      match_dimensions: {
        stage_match: 0-1,     // 新增：阶段匹配度
        window_match: 0-1,    // 新增：窗口匹配度
        signal_match: 0-1,    // 原有：信号匹配度
      },
      confidence: 0-1,
      mutually_exclusive_with: ["string"]  // 易混淆模式
    }
  ],
  window_state: {              // 新增：窗口状态注入
    current: "green"|"yellow"|"red",
    direction: "opening"|"closing"|"stable",
    misjudgment_risk: "high"|"medium"|"low"
  },
  speed_channel: "fast"|"standard"|"deep"  // 原有
}
```

---

## Gate 2：根因定位

### 新增

**新增"多系统证据链"根因定位方法：**

| 新增内容 | 原因 |
|---------|------|
| 根因路径 6：**阶段错位**（新增） | Pipeline 阶段错位（如 Stage 2 做了 Stage 8 的事）本身就是根因——用户不懂升级节奏 |
| 根因路径 7：**窗口判断失误**（新增） | 用户把假窗口当真窗口、把 ASD 当真拒绝、把礼貌当兴趣——导致在错误窗口下行动 |
| 每条根因路径增加"潜沟通证据层" | 用户的弱行为线索是根因的底层证据——不是"他做了什么"错了，是"他是什么状态"错了 |
| 每条根因路径增加"阶段证据层" | 用户当前的 Pipeline 阶段与他的行为是否匹配——如果不匹配，根因可能是阶段认知错误 |
| 新增"窗口变化→根因映射表" | 绿→黄 = 需求感暴露 / 黄→红 = 一致性断裂 / 持续黄 = 窗口未推进（吸引力不够或不敢测试） |

**新增自欺检测增强：**

| 新增内容 | 原因 |
|---------|------|
| `subcommunication_crosscheck` | 用户自述状态 vs Subcommunication System 检测的行为线索——如果矛盾，自欺标记 confidence+0.3 |
| `window_crosscheck` | 用户声称"她喜欢我" vs Window System 判断的红灯——如果矛盾，自欺标记 confidence+0.3 |

### 调用

| 系统 | 调用内容 |
|------|---------|
| **Relationship Pipeline** | `query: { user_stage, user_behaviors } → { stage_mismatch, expected_behaviors }` |
| **Window System** | `query: { window_history } → { direction, possible_root_cause }` |
| **Subcommunication** | `query: { user_state_descriptions } → { weak_cue_patterns, consistency_breaks }` |
| **Text Game** | `query: { user_chat_behaviors } → { stage_error: "在StageX做了StageY的事" }` |
| **Escalation** | `query: { recent_escalation_attempt } → { violated_precondition }` （仅当涉及升级问题时） |

### 输出

```
{
  root_causes: [
    {
      cause_id: 1-7,
      cause_name: "string",
      confidence: 0-1,
      evidence_chain: {
        pipeline_evidence: {...},       // 新增
        window_evidence: {...},         // 新增
        subcommunication_evidence: {...}, // 新增
        text_game_evidence: {...},      // 新增
        escalation_evidence: {...}      // 新增
      },
      primary_system_source: "Pipeline"|"Window"|"Subcommunication"|"TextGame"|"Escalation"|"MentalModel"
    }
  ],
  self_deception: {                     // 增强
    detected: bool,
    confidence: 0-1,
    subcommunication_gap: {...},        // 新增
    window_gap: {...}                   // 新增
  }
}
```

---

## Gate 3：理论覆盖检查

### 新增

**无结构新增。** Gate 3 的六维覆盖矩阵（V1-V6）已经足够全面。Pipeline / Window / Escalation 等系统的存在强化了每个维度的判断精度，但不改变 Gate 3 的结构。

| 维度 | 精度提升来源 |
|------|------------|
| V5 窗口 | Window System 提供精确的窗口状态替代模糊的"窗口开/关" |
| V6 一致性 | Subcommunication System 提供一致性断裂的底层检测 |
| V7（隐含）阶段 | Pipeline 提供"用户当前在关系推进的哪个位置"作为新维度考虑 |

### 调用

| 系统 | 调用内容 |
|------|---------|
| **Window System** | V5 窗口维度的精确值 |
| **Subcommunication** | V6 一致性维度的底层证据 |

### 输出

- 同现有 Gate 3 输出格式，不新增字段。仅各维度的判断依据更精确。

---

## Gate 4：反迎合检查 ★不可跳过

### 新增

**新增三层交叉验证：**

| 新增内容 | 原因 |
|---------|------|
| **层 1：窗口交叉验证** `window_reality_check` | 用户声称"她有窗口" vs Window System 的客观信号判断——这是最高频的迎合场景 |
| **层 2：潜沟通交叉验证** `subcommunication_reality_check` | 用户自我描述（"我很自信""我很放松"） vs Subcommunication System 检测的行为线索——自述与实况的矛盾 |
| **层 3：阶段交叉验证** `stage_reality_check` | 用户声称"我们在暧昧期" vs Pipeline 阶段定位（可能他们还在 Stage 2）——阶段认知偏差 |

**新增窗口误判检测：**

| 检测类型 | 触发条件 | 输出 |
|---------|---------|------|
| 假窗口当真窗口 | 用户认为绿灯 + Window System 判断黄灯/红灯 | `misjudgment: fake_window_as_real` |
| ASD 当真拒绝 | 用户认为红灯（她说"太快了"）+ Window System 判断黄灯（身体没走） | `misjudgment: ASD_as_rejection` |
| 礼貌当兴趣 | 用户认为绿灯（她回复了）+ Window System 判断红灯（敷衍/无注意力） | `misjudgment: politeness_as_interest` |

### 调用

| 系统 | 调用内容 |
|------|---------|
| **Window System** | `query: { user_claim_about_window, window_system_judgment } → { gap, misjudgment_type }` |
| **Subcommunication** | `query: { user_self_description, detected_behavioral_cues } → { gap, inconsistency_points }` |
| **Relationship Pipeline** | `query: { user_claimed_stage, pipeline_stage_detection } → { gap, stage_deviation }` |

### 输出

```
{
  compliance_report: {
    passed: bool,
    concerns: [
      {
        layer: "window"|"subcommunication"|"stage",
        user_claim: "string",
        system_judgment: "string",
        gap_severity: "none"|"mild"|"severe",
        action: "accept_user_view"|"flag_for_correction"|"override_user_view"
      }
    ]
  },
  misjudgment_detection: {
    detected: bool,
    type: "fake_window_as_real"|"ASD_as_rejection"|"politeness_as_interest"|null,
    evidence: "string"
  }
}
```

---

## Gate 5：行动设计

### 新增

**新增"系统查询→行动生成"管线（替代当前的模糊策略生成）：**

| 新增步骤 | 原因 |
|---------|------|
| Step 0：查询 Window Decision Matrix | 最底层约束——当前窗口状态允许什么、禁止什么（这决定了行动的上限） |
| Step 1：查询 Pipeline 阶段行动优先级 | 当前阶段该做什么、不该做什么——这是行动的范围 |
| Step 2：查询 Escalation Ladder | 如果行动涉及升级——当前在哪个 Rung？是否可以升级？怎么升级？ |
| Step 3：查询 Text Game 阶段策略 | 如果行动涉及聊天——当前在哪个聊天阶段？该阶段的行为原则是什么？ |
| Step 4：查询 Subcommunication 状态修正 | 行动中"用什么状态做"——语速、眼神、站姿的修正建议 |

**新增行动类型：**

| 行动类型 | 说明 | 何时触发 |
|---------|------|---------|
| `STOP` | 停止当前所有主动行为 | Window 红灯 + Pipeline 阶段错位 |
| `HOLD` | 保持当前状态不变 | Window 黄灯，等待更多信号 |
| `TEST_WINDOW` | 施压测试窗口（进一步退两步） | Window 黄灯，需要确认窗口 |
| `ESCALATE` | 推进升级（查询 Escalation Ladder） | Window 绿灯，前置条件满足 |
| `DE_ESCALATE` | 降级减压 | 升级被拒，需要退回前一级 |
| `STAGE_RESET` | 回到正确阶段 | 检测到 Pipeline 阶段错位 |
| `STATE_FIX` | 修正潜沟通状态 | Subcommunication 检测到弱行为线索需要修正 |

**新增"行动+状态"双层输出：**

- 不仅输出"做什么"，也输出"用什么状态做"
- 例："停发消息" + "（状态：放松，不焦虑，去做别的事）"

### 调用

| 系统 | 调用内容 | 优先级 |
|------|---------|--------|
| **Window System** | `query: DecisionMatrix(window) → { allowed, forbidden, conditional }` | ★★★ 最先调用 |
| **Relationship Pipeline** | `query: { stage } → { stage_priorities, stage_forbidden }` | ★★ |
| **Escalation System** | `query: { rung, girl_type, window } → { can_escalate, strategy, de_escalation_plan }` | ★★ |
| **Text Game** | `query: { chat_stage } → { principles, freq, forbidden }` | ★★ |
| **Subcommunication** | `query: { weak_cues } → { state_fix_suggestions }` | ★ |

### 输出

```
{
  action_plan: {
    primary_action: {
      type: "STOP"|"HOLD"|"TEST_WINDOW"|"ESCALATE"|"DE_ESCALATE"|"STAGE_RESET"|"STATE_FIX",
      description: "string",
      timing: "immediate"|"within_24h"|"within_3d"|"within_1w",
      exit_condition: "string"
    },
    state_layer: {                      // 新增
      subcommunication_fix: "string",   // 如："语速放慢三倍"
      mindset_fix: "string"             // 如："不破防——她不回不是因为你不配"
    },
    decision_basis: {                   // 新增：决策依据追溯
      window_constraint: "string",       // 来自 Window Decision Matrix
      stage_constraint: "string",        // 来自 Pipeline
      escalation_advice: "string",       // 来自 Escalation Ladder
      text_game_advice: "string",        // 来自 Text Game
      state_advice: "string"             // 来自 Subcommunication
    }
  },
  coach_decision: "single_step"|"campaign"  // 原有
}
```

---

## Gate 5b：Campaign Planner

### 新增

**新增"阶段驱动"模板匹配：**

| 新增内容 | 原因 |
|---------|------|
| `match_campaign_by_stage(current_stage, target_stage) → template` | Pipeline 阶段差决定 Campaign 模板——不再仅靠场景判断 |
| `generate_phase_plan(template, current_stage, target_stage, escalation_rungs) → phases` | Phase 设计基于阶段路径和升级阶梯 |
| `set_window_checkpoints(phases) → checkpoints` | 每个 Phase 结束时检查窗口状态变化 |
| `set_escalation_checkpoints(phases) → checkpoints` | 每个 Phase 的升级步骤验证 |

**新增阶段→模板映射：**

| Pipeline 阶段路径 | 推荐 Campaign 模板 |
|------------------|-------------------|
| Stage 0→1 | T3 新手训练 |
| Stage 2→3→4→5 | T1 渐进升温 |
| Stage 5→6→7→8→9 | T1 渐进升温 + T4 决胜局（最后阶段） |
| Stage X→X-2（回退 2+ 阶段） | T2 长周期修复 |
| Stage 9 刚完成 | T5 TD后防守 |
| 多关系并行 | T6 多关系并行 |
| 阶段加速（Stage 0+1 同时） | **新增 T7：阶段加速** |

**新增 Phase 结构：**

每个 Phase 现在包含：
```
Phase N:
  stage_target: int          # 目标 Pipeline Stage
  text_game_target: int      # 目标 Text Game Stage（如适用）
  escalation_target: int     # 目标 Escalation Rung（如适用）
  window_requirement: string  # 进入下一 Phase 的窗口要求
  actions: [...]             # 基于上述 target 的行动列表
  checkpoints: [
    { type: "window", query: "window状态是否达到要求？" },
    { type: "escalation", query: "前置Rung是否完成？" },
    { type: "stage", query: "Pipeline阶段是否推进？" }
  ]
```

### 调用

| 系统 | 调用内容 |
|------|---------|
| **Relationship Pipeline** | `query: { current_stage, target_stage } → { stage_path, recommended_template, stage_specific_risks }` |
| **Window System** | `query: { current_window } → { window_gate_for_next_phase }` |
| **Escalation System** | `query: { current_rung, target_rung, girl_type } → { rung_path, phase_milestones }` |
| **Text Game** | `query: { current_chat_stage, target_chat_stage } → { chat_phase_plan }` |

### 输出

```
{
  campaign: {
    template: "T1"|"T2"|...|"T7",
    stage_path: { from: int, to: int, distance: int },
    confidence: 0-1,
    phases: [
      {
        id: 1,
        stage_target: int,
        text_game_target: int | null,
        escalation_target: int | null,
        window_requirement: "green"|"yellow_improving"|"any",
        actions: [
          {
            type: "STOP"|"HOLD"|"TEST_WINDOW"|...,
            description: "string",
            system_source: "Pipeline"|"Window"|"Escalation"|"TextGame",
            timing: "string"
          }
        ],
        checkpoints: [
          { type: "window", check: "string", next_if_pass: int, next_if_fail: "string" },
          { type: "escalation", check: "string", next_if_pass: int, next_if_fail: "string" },
          { type: "stage", check: "string", next_if_pass: int, next_if_fail: "string" }
        ]
      }
    ],
    abandon_conditions: [
      "连续2次phase检查点窗口关闭",
      "玩家连续2次不执行phase行动",
      "对方明确终止"
    ]
  }
}
```

---

## 附录 A：Skill Modification Checklist

> **格式：** `[位置] → [修改类型] → [具体修改] → [涉及系统]`
> **修改类型：** ADD（新增）/ MODIFY（修改）/ DELETE（删除）/ REPLACE（替换）

### Gate -1：玩家状态管理

| # | 位置（SKILL.md 章节/行） | 类型 | 修改内容 | 系统 |
|---|------------------------|------|---------|------|
| G-1.1 | 零·Gate -1·步骤 3·表格 | ADD | 新增读取行：pipeline_progress（用途：Gate 0 阶段定位） | Pipeline |
| G-1.2 | 零·Gate -1·步骤 3·表格 | ADD | 新增读取行：window_judgment（用途：Gate 4 反迎合基线） | Window |
| G-1.3 | 零·Gate -1·步骤 3·表格 | ADD | 新增读取行：escalation_profile（用途：Gate 5 升级节奏校准） | Escalation |
| G-1.4 | 零·Gate -1·步骤 3·表格 | ADD | 新增读取行：text_game_profile（用途：Gate 5 聊天策略个性化） | Text Game |
| G-1.5 | 零·Gate -1·步骤 3·表格 | ADD | 新增读取行：subcommunication_profile（用途：Gate 2/4 行为线索基线） | Subcomm |
| G-1.6 | 零·会话结束·7步更新 | ADD | 步骤 8：更新 pipeline_progress（阶段变化/阶段停留时间） | Pipeline |
| G-1.7 | 零·会话结束·7步更新 | ADD | 步骤 9：更新 window_judgment（本次窗口判断结果 vs 实际） | Window |
| G-1.8 | 零·会话结束·7步更新 | ADD | 步骤 10：更新 escalation_profile（本次升级结果） | Escalation |
| G-1.9 | 零·会话结束·7步更新 | ADD | 步骤 11：更新 text_game_profile（本次聊天阶段+违规检测） | Text Game |
| G-1.10 | 零·会话结束·7步更新 | ADD | 步骤 12：更新 subcommunication_profile（本次咨询中的弱行为线索模式） | Subcomm |
| G-1.11 | 零·定期 Review·表格 | ADD | 新增触发条件："pipeline_progress 中某阶段停留超过 90 天" → 专项 Review | Pipeline |
| G-1.12 | 零·定期 Review·表格 | ADD | 新增触发条件："window_judgment accuracy_rate < 0.4 持续 5 次" → 专项 Review | Window |

### Gate 0：信息充分性

| # | 位置（SKILL.md 章节/行） | 类型 | 修改内容 | 系统 |
|---|------------------------|------|---------|------|
| G0.1 | 三·Gate 0·最低信息集 | MODIFY | "关系背景（什么关系、什么阶段）" → "关系背景（Pipeline Stage 0-10 + 该阶段的进入方式）" | Pipeline |
| G0.2 | 三·Gate 0·判断逻辑 | ADD | 在"三项都有→进入 Gate 1"之前，增加"调用 Pipeline 阶段定位"步骤 | Pipeline |
| G0.3 | 三·Gate 0 | ADD | 新增："阶段错位检测：如果用户在 Stage X 做了 Stage Y 的事，标记 HIGH_PRIORITY" | Pipeline |
| G0.4 | 三·Gate 0 | ADD | 新增："如果用户卡在某一阶段超过该阶段典型时长，标记 STUCK" | Pipeline |
| G0.5 | 三·Gate 0·判断逻辑 | ADD | 新增追问维度：如果阶段定位不明确→追问"你们认识多久了？最近一次见面/聊天是什么时候？" | Pipeline |

### Gate 1：一眼判断

| # | 位置（SKILL.md 章节/行） | 类型 | 修改内容 | 系统 |
|---|------------------------|------|---------|------|
| G1.1 | 三·Gate 1·执行步骤 1 | REPLACE | "快速判断关系阶段（陌生/熟悉/暧昧/亲密/长期）" → "使用 Pipeline 精确定位阶段（Stage 0-10，带 confidence）" | Pipeline |
| G1.2 | 三·Gate 1·执行步骤 2 | MODIFY | 信号提取从 5 个维度增加到 6 个：增加"窗口状态（基于 Window System）" | Window |
| G1.3 | 三·Gate 1·执行步骤 2 | ADD | 新增信号维度 7："用户潜沟通状态（基于 Subcommunication System）" | Subcomm |
| G1.4 | 三·Gate 1·执行步骤 3 | MODIFY | "与 40 个 Pattern 做相似度匹配" → "根据当前阶段限定搜索范围（E/M/L/X），在该范围内匹配" | Pipeline |
| G1.5 | 三·Gate 1·执行步骤 3 | ADD | 新增匹配维度：窗口匹配度——将 Window System 的窗口状态映射到 Pattern 信号权重 | Window |
| G1.6 | 三·Gate 1·匹配度评估 | ADD | 新增：窗口信号与 Pattern 窗口特征高度一致→match_score+0.2 | Window |
| G1.7 | 三·Gate 1·匹配度评估 | ADD | 新增：Subcommunication 检测到严重弱行为线索 + 用户声称无问题→自动提升 X01（自欺驱动）优先级 | Subcomm |

### Gate 2：根因定位

| # | 位置（SKILL.md 章节/行） | 类型 | 修改内容 | 系统 |
|---|------------------------|------|---------|------|
| G2.1 | 三·Gate 2·5条根因路径 | ADD | 新增根因路径 6："阶段错位——用户在 Pipeline Stage X 做了 Stage Y 的事" | Pipeline |
| G2.2 | 三·Gate 2·5条根因路径 | ADD | 新增根因路径 7："窗口判断失误——用户将假窗口当真窗口/ASD当真拒绝/礼貌当兴趣" | Window |
| G2.3 | 三·Gate 2·根因路径 1（吸引力不存在） | MODIFY | 增加证据层："Pipeline 阶段未推进的直接证据"（如：Stage 3 停留超过 4 周） | Pipeline |
| G2.4 | 三·Gate 2·根因路径 2（需求感暴露） | MODIFY | 增加证据层："窗口变化证据——绿→黄的窗口下降方向""潜沟通证据——用户的弱行为线索模式" | Window / Subcomm |
| G2.5 | 三·Gate 2·根因路径 3（一致性断裂） | MODIFY | 增加证据层："Subcommunication 检测到的前后行为线索不一致" | Subcomm |
| G2.6 | 三·Gate 2·根因路径 4（框架沦陷） | MODIFY | 增加证据层："Pipeline 阶段回退（如 Stage 5→Stage 3）" | Pipeline |
| G2.7 | 三·Gate 2·自欺检测 D1c | ADD | 新增交叉验证："用户自述状态 vs Subcommunication 行为线索检测" | Subcomm |
| G2.8 | 三·Gate 2·自欺检测 D1c | ADD | 新增交叉验证："用户声称她有窗口 vs Window System 判断的红灯" | Window |

### Gate 3：理论覆盖检查

| # | 位置（SKILL.md 章节/行） | 类型 | 修改内容 | 系统 |
|---|------------------------|------|---------|------|
| G3.1 | 三·Gate 3·V5 窗口检查 | MODIFY | "窗口开还是关？" → "窗口状态（Window System 输出：绿灯/黄灯/红灯 + 变化方向）" | Window |
| G3.2 | 三·Gate 3·V6 一致性检查 | MODIFY | "前后一致吗？" → "前后一致吗？（Subcommunication 一致性断裂检测）" | Subcomm |

### Gate 4：反迎合检查

| # | 位置（SKILL.md 章节/行） | 类型 | 修改内容 | 系统 |
|---|------------------------|------|---------|------|
| G4.1 | 三·Gate 4·检查 1·迎合检测 | ADD | 新增子检查 1a："窗口交叉验证——用户声称的窗口 vs Window System 客观判断" | Window |
| G4.2 | 三·Gate 4·检查 1·迎合检测 | ADD | 新增子检查 1b："潜沟通交叉验证——用户自我描述 vs Subcommunication 行为线索检测" | Subcomm |
| G4.3 | 三·Gate 4·检查 1·迎合检测 | ADD | 新增子检查 1c："阶段交叉验证——用户声称的关系阶段 vs Pipeline 阶段定位" | Pipeline |
| G4.4 | 三·Gate 4 | ADD | 新增检查 4："窗口误判检测——是否存在假窗口当真窗口、ASD当真拒绝、礼貌当兴趣" | Window |

### Gate 5：行动设计

| # | 位置（SKILL.md 章节/行） | 类型 | 修改内容 | 系统 |
|---|------------------------|------|---------|------|
| G5.1 | 三·Gate 5 | ADD | 新增 Step 0（在所有步骤之前）："查询 Window Decision Matrix——当前窗口允许什么、禁止什么" | Window |
| G5.2 | 三·Gate 5 | ADD | 新增 Step 1（原 Step 1 之前）："查询 Pipeline 阶段行动优先级——当前阶段该做什么" | Pipeline |
| G5.3 | 三·Gate 5·单步行动优先级 | MODIFY | 从 3 类（止损/状态/策略）扩展到 7 类（+ 阶段复位/窗口测试/升级推进/降级减压） | All |
| G5.4 | 三·Gate 5·单步行动优先级 | ADD | 新增：在"策略行动"中增加子类"升级测试"和"升级推进"（查询 Escalation Ladder） | Escalation |
| G5.5 | 三·Gate 5·单步行动优先级 | ADD | 新增：在"策略行动"中增加子类"聊天策略"（查询 Text Game 阶段原则） | Text Game |
| G5.6 | 三·Gate 5·约束 | ADD | 新增约束："任何涉及升级的行动（邀约/肢体/亲吻/转场/TD），必须查询 Escalation System 的当前 Rung 前置条件" | Escalation |
| G5.7 | 三·Gate 5·约束 | ADD | 新增约束："任何涉及聊天的行动，必须查询 Text Game 当前阶段的行为原则" | Text Game |
| G5.8 | 三·Gate 5 | ADD | 新增输出层："状态层——不只要告诉用户做什么，还要告诉用什么状态做（基于 Subcommunication）" | Subcomm |
| G5.9 | 三·Gate 5 | ADD | 新增输出层："决策依据追溯——这个行动建议来自哪个系统、基于什么判断" | All |
| G5.10 | 三·Gate 5·Coach Decision Policy | MODIFY | 增加分流条件："当前→目标阶段差 >= 2 → 自动倾向 Campaign" | Pipeline |
| G5.11 | 三·Gate 5·Coach Decision Policy | ADD | 新增分流条件："涉及升级推进（亲吻/转场/TD）→ Gate 5 + Escalation System 联合决策" | Escalation |

### Gate 5b：Campaign Planner

| # | 位置（SKILL.md 章节/行） | 类型 | 修改内容 | 系统 |
|---|------------------------|------|---------|------|
| G5b.1 | 三·Gate 5b·创建流程·步骤 2 | MODIFY | "选择模板（T1-T6）" → "选择模板（T1-T7）：增加 Pipeline 阶段差匹配逻辑" | Pipeline |
| G5b.2 | 三·Gate 5b·创建流程 | ADD | 新增步骤 2a："根据当前阶段和目标阶段计算阶段路径→匹配 Campaign 模板（查询阶段→模板映射）" | Pipeline |
| G5b.3 | 三·Gate 5b·创建流程·步骤 3 | MODIFY | "调用 Gate 5 生成每 phase 的具体 action" → "调用 Gate 5 + Escalation System + Text Game 生成每 phase 的具体 action" | All |
| G5b.4 | 三·Gate 5b·创建流程·步骤 4 | ADD | 新增："设置窗口检查点——每个 phase 结束时查询 Window System 检查窗口状态" | Window |
| G5b.5 | 三·Gate 5b·创建流程·步骤 4 | ADD | 新增："设置升级检查点——每个 phase 的升级步骤验证（查询 Escalation System）" | Escalation |
| G5b.6 | 三·Gate 5b·6种战役模板 | ADD | 新增 T7："阶段加速——当用户需要多阶段并行推进（如 Stage 0 内在建设和 Stage 1 搭讪同时进行）" | Pipeline |
| G5b.7 | 三·Gate 5b·输出约束 | ADD | 新增："初次只给 2-3 个 phase，每个 phase 标注：阶段目标 + 窗口要求 + 升级目标 + 聊天阶段目标" | All |
| G5b.8 | 三·Gate 5b·状态机 | MODIFY | abandoned 条件增加："连续 2 次 phase 检查点窗口状态未达到要求" | Window |
| G5b.9 | 三·Gate 5b·Campaign 执行期间的 Player Model 更新 | ADD | 新增更新项："每次 phase 完成后更新 pipeline_progress.stage_history" | Pipeline |
| G5b.10 | 三·Gate 5b·Campaign 执行期间的 Player Model 更新 | ADD | 新增更新项："每次 phase 完成后更新 escalation_profile（升级结果）" | Escalation |

### 知识文件索引

| # | 位置（SKILL.md 章节/行） | 类型 | 修改内容 | 系统 |
|---|------------------------|------|---------|------|
| K.1 | 八·知识文件索引·表格 | ADD | 新增文件行：`references/relationship_pipeline.md`，用途：Gate 0/1/2/5/5b 阶段定位与行动优先级 | Pipeline |
| K.2 | 八·知识文件索引·表格 | ADD | 新增文件行：`references/window_system.md`，用途：Gate 1/2/4/5/5b 窗口判断与决策矩阵 | Window |
| K.3 | 八·知识文件索引·表格 | ADD | 新增文件行：`references/text_game_methodology.md`，用途：Gate 2/5/5b 聊天阶段定位与策略 | Text Game |
| K.4 | 八·知识文件索引·表格 | ADD | 新增文件行：`references/escalation_system.md`，用途：Gate 2/5/5b 升级阶梯与节奏控制 | Escalation |
| K.5 | 八·知识文件索引·表格 | ADD | 新增文件行：`references/subcommunication_system.md`，用途：Gate 1/2/4/5 潜沟通信号库与状态修正 | Subcomm |
| K.6 | 八·知识文件索引·调用时机列 | ADD | Pipeline 调用时机：Gate 0（阶段定位）/ Gate 1（模式范围限定）/ Gate 2（阶段错位根因）/ Gate 5（行动优先级）/ Gate 5b（模板匹配） | Pipeline |
| K.7 | 八·知识文件索引·调用时机列 | ADD | Window 调用时机：Gate 1（信号→模式匹配）/ Gate 2（根因推断）/ Gate 4（反迎合交叉验证）/ Gate 5（决策矩阵）/ Gate 5b（检查点） | Window |

### 输出层

| # | 位置（SKILL.md 章节/行） | 类型 | 修改内容 | 系统 |
|---|------------------------|------|---------|------|
| O.1 | 六·表达输出铁律·各通道输出模板·快速通道 | MODIFY | "判断+变量+下一步" → "判断+窗口状态+阶段+下一步+状态提示" | Window / Pipeline |
| O.2 | 六·表达输出铁律·各通道输出模板·标准通道 | MODIFY | 增加"窗口判断"行（如："当前窗口：🟡黄灯，ASD可能"） | Window |
| O.3 | 六·表达输出铁律·各通道输出模板·深度通道 | MODIFY | 增加"潜沟通分析"行（如："你的弱行为线索：语速过快、眼神回避"） | Subcomm |
| O.4 | 六·表达输出铁律·意图识别·表格 | ADD | 新增意图行："我该亲她吗/该转场吗" → 处理方式："快速通道+EscalationSystem查询当前Rung" | Escalation |
| O.5 | 六·表达输出铁律·意图识别·表格 | ADD | 新增意图行："我现在是什么阶段" → 处理方式："快速通道+Pipeline阶段定位+下一步建议" | Pipeline |

---

## 附录 B：修改统计

| 类型 | 数量 | 说明 |
|------|------|------|
| **ADD（新增）** | 44 | 新逻辑/新步骤/新字段 |
| **MODIFY（修改）** | 14 | 已有逻辑的增强/细化 |
| **REPLACE（替换）** | 1 | 已有逻辑的完整替换 |
| **DELETE（删除）** | 0 | 无删除——所有修改为增量 |

| 涉及系统 | 修改数量 |
|---------|---------|
| Pipeline | 21 |
| Window | 17 |
| Subcommunication | 12 |
| Escalation | 11 |
| Text Game | 7 |
| All（跨系统） | 3 |

---

*设计完成。不修改任何已有文件。v1.0 | 2026-08-02。*
