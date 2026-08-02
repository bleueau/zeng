# Campaign Planner Reference v1

> 将 Gate 5 单步行动扩展为多步战役。不等玩家问"然后呢"。

---

## 一、激活条件

### 不激活（Gate 5 单步足够）
止损类（"停发消息"）、即时纠正类（"现在就做X"）、简单二选一类。

### 激活（任一满足）

| 条件 | 示例 |
|------|------|
| 窗口存在且需要渐进推进 | "她给了 IOI，需要 2-3 步从聊天推进到邀约" |
| 长周期恢复计划 | "框架丢了 8 个月，需要 2 个月三阶段修复" |
| 多关系并行策略 | "A 要降温、B 要推进、C 要开始" |
| 新手分阶段训练 | "先外形改造→再网聊→再实战" |
| 决胜局环境控制 | "第三次约会，需要设计完整场景链" |
| TD 后防需求感链 | "TD 后 5 天内的消息节奏规划" |

**与 Gate 5 的分工：**
```
Gate 5 输出：          "停发消息 7 天"
Campaign Planner 输出： "Day 0-7: 停发一切 → Day 7: 发轻松消息 → 如果回：对齐热度(≤5轮) → Day 10: 提邀约 → 如果不回：再等7天 → Day 14不回：放弃"
```

---

## 二、Campaign 结构

### 主结构
```json
{
  "campaign_id": "CAMP-001",
  "type": "escalation | recovery | training | multi_rel | decisive_date | post_td",
  "target_outcome": "一句话：这次战役要达成的具体目标",
  "context": {
    "relationship_tag": "关联的关系",
    "current_stage": "关系阶段",
    "root_cause": "Gate 2 诊断结果",
    "player_experience_level": "引用 Player Model",
    "urgency": "immediate | this_week | this_month"
  },
  "phases": [...],
  "global_exit_conditions": [...],
  "status": "draft | active | paused | completed | abandoned",
  "confidence": 0.0-1.0
}
```

### Phase 子结构
```json
{
  "phase_id": 1,
  "name": "中止当前错误行为",
  "order": 1,
  "timing": {
    "type": "immediate | delay_days | after_signal",
    "value": "立即" 或 "3" 或 "她主动发消息后"
  },
  "action": {
    "instruction": "具体行动指令（由 Gate 5 生成）",
    "dosage": "做多少",
    "coach_pattern_used": "CP-C1"
  },
  "success_condition": "客观判断标准",
  "branch": [
    {"condition": "如果 X 发生", "next_phase": 2},
    {"condition": "如果 Y 发生", "next_phase": "EXIT", "exit_reason": "..."}
  ],
  "checkpoint": {
    "evaluate_after": "执行后 N 天或等玩家反馈",
    "metrics": ["对方是否主动联系", "玩家是否控制住没加量发消息"]
  }
}
```

### 全局退出条件
- 连续2个phase的success_condition未达成 → abandon
- 目标达成 → complete
- 对方明确拒绝/拉黑 → abandon
- 玩家放弃执行 → abandon（标记到 Player Model）

---

## 三、6 种 Campaign 类型模板

### T1：渐进升温（Escalation）
```
适用：窗口存在，需要从当前阶段推进到下一阶段
Phase 1 试探信号 → Phase 2 确认窗口 → Phase 3 行动升级
节奏：3-7天，每 phase 间隔 1-3 天
Evidence：CB-002, CB-111, CP-C1
```

### T2：长周期修复（Recovery）
```
适用：框架丢失/吸引力长期流失
Phase 1 止损+边界重建 → Phase 2 降低可得性 → Phase 3 评估变化
节奏：4-8周，每 phase 间隔 2-4 周
Evidence：CB-012, CP-C2
```

### T3：新手分阶段训练（Training）
```
适用：基础差的玩家需要系统提升
Phase 1 外形/基础改造 → Phase 2 低压力练习（网聊/对视）→ Phase 3 实战
节奏：4-12周，Phase 1完成→Phase 2开放；Phase 2≥3次成功→Phase 3开放
Evidence：CB-012, CB-013, CB-108
```

### T4：决胜局环境设计（Decisive Date）
```
适用：前几次约会未推进，需要设计完整场景链
Phase 1 场地+物资 → Phase 2 当天流程（各阶段升级节点）→ Phase 3 收尾
节奏：1-3天准备 + 当天执行
Evidence：CB-121, CB-114
```

### T5：TD 后防守（Post-TD）
```
适用：TD后防止需求感暴露
Phase 1 立即停止一切主动联系 → Phase 2 等3-5天 → Phase 3 轻触 → Phase 4 观察
节奏：5-7天
Evidence：CB-008, M2+M8
```

### T6：多关系并行策略（Multi-Rel）
```
适用：玩家同时多条关系
结构：每条关系独立一条 phase 链，共享全局检查点
输出格式：关系矩阵，每行一条关系 + 对应 action + timing
```

---

## 四、状态机

```
draft（AI 生成，待玩家确认）
    ↓
active（玩家接受，开始执行）
    ├── 每完成一个 phase → 玩家反馈 → 检查 success_condition
    │       ├── 达成 → 进入下一个 phase
    │       └── 未达成 →
    │               ├── 首次未达成 → 调整 phase 参数，重试
    │               └── 连续 2 次未达成 → 触发全局退出，abandoned
    ├── 所有 phases 完成 + 目标达成 → completed
    ├── 玩家主动放弃或失联 → abandoned
    └── 外部事件导致无意义（对方拉黑/关系结束）→ abandoned
```

---

## 五、输出约束

**输出给玩家时必须遵守：**
1. 不超过 2-3 个 phase（初次输出不给完整链路，防止信息过载）
2. 只给当前 phase 的详细指令 + 下一个 phase 的预览
3. 退出条件必须明确告知（"如果连续两次 X，就停止，回来找我"）
4. 后续 phases 在玩家反馈后逐步展开（非一次性输出全部）

**规则：Campaign 输出 ≠ 完整剧本一次性给。** Mikey 不会说"接下来 30 天你每天做什么"——他会说"你先做这个，做完告诉我结果，我们再走下一步。"

---

## 六、与 Player Model 集成

### 读取（创建 Campaign 时）
- experience_level → T3 训练节奏调整；T1/T4 激进策略判断
- Active Weaknesses → phase 中预设防护（如 T5 对慢性需求感玩家自动激活）
- verified_interventions → 复用已验证方案
- Case History → 查询上次同类 Campaign 结果

### 写入（Campaign 执行期间）
- 每次咨询后更新 Campaign status 和 phases
- 追加 Case History（type=campaign_phase_completed）
- Campaign 完成 → 更新 Active Weaknesses progress，检查 Milestone
- Campaign abandoned → 记录 abandon_reason；如果因玩家放弃执行，标记到 Behavior Patterns

---

## 七、Confidence 规则

| 场景 | confidence |
|------|-----------|
| 使用成熟模板（T1-T6）匹配度高 | ≥ 0.7 |
| 混合模板/调整了模板参数 | 0.5-0.7 |
| 完全自定义 campaign（无模板匹配）| ≤ 0.5 |
| Player Model 中 experience_level 置信度低 | -0.2 |
| 首次为该玩家创建此类 Campaign | -0.1 |

---

## 八、验证——此设计解决的核心失败

| 案例 | 失败原因 | Campaign Planner 如何解决 |
|------|---------|--------------------------|
| Case 2 | 邀约后无多步规划 | T1：Phase 1 试探 → Phase 2 确认窗口 → Phase 3 邀约 + 退出条件 |
| Case 5 | 窗口判断后有判断无规划 | T1：确认窗口→3步序列，每步有信号判断 |
| Case 17 | 跨关系对比后无法输出复合策略 | T6：关系矩阵 + 每条独立 phase 链 |
| Case 13 | 长周期修复无阶段计划 | T2：3 phase 跨 4-8 周 |
| Case 10 | TD后无防守序列 | T5：从单步"停发"扩展为 4 phase 节奏控制 |
