# Player Model Reference v1

> 玩家持久化成长追踪模型。每次咨询开始前加载，结束后更新。

---

## 一、状态文件

所有玩家数据存储在 `mikey-consultant/player_state.json`。

**会话开始：** 
1. 验证 `_meta.user_id`。如非 null 且与当前用户不匹配 → 不加载，视为新玩家。
2. 读取 `player_state.json`（如不存在则玩家为新用户）。

**会话结束（原子写入）：**
0. 读入内存 → A. 执行 7 步更新 → B. 写入 `.tmp` → C. 验证 JSON → D. 替换原文件 → 失败时保留原文件不变。

---

## 二、数据结构

### 1. Player Core（玩家核心）

```json
{
  "user_id": "当前用户的唯一标识",
  "experience_level": "beginner | intermediate | advanced",
  "experience_level_confidence": 0.0-1.0,
  "experience_level_evidence": "Gate0:初诊评估",
  "onboarding_date": "日期",
  "primary_context": "与coaching相关的背景",
  "notable_constraints": ["时间少只能周末", "小城市社交圈有限"],
  "consultation_count": 0,
  "last_reviewed": "日期"
}
```

**首次创建：** experience_level_confidence ≤ 0.5，必须附 evidence_ref。

### 2. Skill Level（技能评估）

8 个维度，每个 1-10 分：`approach`, `chat_maintain`, `date_execution`, `escalation`, `frame_strength`, `neediness_control`, `self_awareness`

```json
{
  "approach": {"score": 3, "confidence": 0.3, "last_changed": "日期", "evidence_ref": "..."},
  ...
}
```

**评级标准：** 1-3=活跃弱点 / 4-6=不稳定 / 7-8=可靠 / 9-10=已掌握
**Confidence 规则：** 首次 ≤0.5，单次观察 ≤0.4，连续3次一致 → ≥0.7
**衰减：** 60天不练习 → score-1, confidence-0.2

### 3. Behavior Patterns（行为模式）

```json
{
  "pattern_id": "BP-001",
  "name": "冷淡加量追逐",
  "type": "neediness_spike",
  "status": "active | dormant | resolved | reactivated | tentative",
  "first_observed": "日期",
  "last_observed": "日期",
  "occurrence_count": 3,
  "severity": "mild | moderate | severe",
  "confidence": 0.7,
  "evidence_ref": "CB-001, CB-014",
  "trigger": "对方回复频率下降超过50%",
  "description": "对方一冷淡，玩家就把消息频率翻倍",
  "verified_interventions": ["停发5天"],
  "related_cases": ["case_003"]
}
```

**生命周期状态机：**
```
occ=1 → active, severity=mild, confidence≤0.3
occ=2 → active, severity≥moderate, confidence≥0.5
occ≥3 → active, severity=severe, confidence≥0.7
90天不复发 → dormant
再次出现(dormant) → reactivated
玩家确认克服 → resolved
```

**创建规则：** occ=1→confidence≤0.3；occ=2→≥0.5；occ≥3→≥0.7

### 4. Blind Spots（盲点）

```json
{
  "blind_spot_id": "BS-001",
  "description": "把礼貌回应误读为兴趣指标",
  "confidence": 0.6,
  "evidence_ref": "CB-002, Gate4:反迎合检测",
  "occurrence_count": 2,
  "status": "active | improving | likely_resolved"
}
```

**创建规则：** 首次 confidence≤0.4；occ≥2→≥0.6

### 5. Growth Baseline（成长基线）

```json
{
  "recorded_date": "日期",
  "initial_state": "纯小白，看到女生走在路上会焦虑",
  "initial_skill_snapshot": {"approach": 1, ...},
  "baseline_anecdote": "第一次搭讪手抖到手机差点掉了"
}
```

**只存一份，永不覆盖。**

### 6. Milestones（里程碑）

```json
{
  "milestone_id": "MS-001",
  "date": "日期",
  "type": "first_approach | first_number | first_date | first_td | pattern_breakthrough | skill_jump",
  "description": "第一次成功搭讪并聊了5分钟以上",
  "confidence": 1.0,
  "player_confirmed": true
}
```

**pattern_breakthrough / skill_jump 需玩家确认。**

### 7. Active Weaknesses（活跃弱点）

```json
{
  "weakness_id": "AW-001",
  "description": "TD后立刻暴露需求感",
  "related_pattern": "BP-001",
  "coaching_strategy": "CP-C1：停发3天",
  "confidence": 0.7,
  "evidence_ref": "CB-008, Gate2:M2",
  "status": "new | in_progress | improving | near_resolved | resolved | strategy_ineffective",
  "fix_attempts": 2,
  "occurrence_after_last_fix": 0
}
```

**生命周期：**
```
new → in_progress（首次给干预方案）
    → improving（fix_attempts≥1 且 复现=0）
    → near_resolved（连续2次未复现）
    → resolved（连续5次未复现）
fix_attempts≥5 且 复现≥3 → strategy_ineffective
```

### 8. Case History Index（案例历史）

```json
{
  "case_id": "case_015",
  "date": "日期",
  "summary": "第三次因女生冷淡而加量追",
  "patterns_manifested": ["BP-001"],
  "coach_patterns_used": ["CP-A1", "CP-C1"],
  "outcome": "已给止损指令，待验证",
  "campaign_id": null
}
```

**每次咨询后追加，仅追加不修改。最多保留30条 + 均匀采样。**

### 9. Relationship References（关系索引）

```json
{
  "relationship_tag": "女生A-成都舞蹈老师",
  "stage": "暧昧",
  "patterns_in_this_rel": ["BP-001"],
  "status": "active | ended | archived"
}
```

---

## 三、读写流程

### 会话开始：读取（必读项）

**0. 身份验证：** 检查 `_meta.user_id`。非 null 且与当前用户不匹配 → 不加载，视为新玩家。

1. Player Core（experience_level + constraints）
2. Active Weaknesses（全部 active，按 status 排序）
3. Behavior Patterns[status=active]（Top 3 by severity）
4. Growth Baseline
5. Last Case Summary

**选读：** Blind Spots（Gate 2 检测到自欺时）、Milestones（输出成长肯定时）、Relationship References（提到已知关系时）

### 推理中使用

| Gate | 读取 Player Model |
|------|------------------|
| Gate 0 | experience_level → 调整信息阈值 |
| Gate 1 | Behavior Patterns → 重复/新模式判断 |
| Gate 2 | Blind Spots + Active Weaknesses → 根因方向 |
| Gate 4 | Behavior Patterns 历史 → 自述一贯性验证 |
| Gate 5 | verified_interventions → 复用方案 |
| 输出层 | Growth Baseline + Milestones → 成长对比 |

### 会话结束：原子写入

**写入流程：**
0. 读取当前 `player_state.json` 到内存
A. 在内存中执行以下 7 步更新
B. 将更新后内容写入 `player_state.json.tmp`
C. 验证 `.tmp` 文件 JSON 可解析
D. 用 `.tmp` 替换 `player_state.json`
E. 任何步骤失败 → 保留原文件不变，不丢失数据

7 步内存更新：
1. 追加 Case History Index
2. 扫描 Behavior Pattern（更新 occ + confidence）
3. 检查新 Blind Spot（附 evidence_ref + confidence）
4. 评估 Skill Level（单次≤0.4）
5. 更新 Active Weaknesses progress
6. 检查 Milestone 触发
7. 更新 Relationship References

**防污染检查：** 单次表现→不修 severity / 置信度<0.5→标记 tentative / 情绪波动期→延后 Skill Level 更新 / evidence_ref 缺失→不写入

---

## 四、定期 Review

**触发条件：** 累计10次咨询 / 距上次 Review 超过90天 / 单个 BP confidence<0.3 持续5次 / 单个 AW fix≥5但复现≥3

**执行内容：**
- BP confidence<0.3 且 occ=1 → 降级 tentative
- BP confidence≥0.7 且 30天内复发 → confidence+0.1
- AW fix≥5且无改善 → strategy_ineffective
- AW fix≥3且 improving → 提升 confidence
- BS 最后出现≥5次咨询 → likely_resolved

**防膨胀：** BP>10→保留 Top10 / AW>5→保留 Top5 / BS>8→保留 Top8 / Case History>50→保留最近30+均匀采样
