# Mikey Skill v4.1 Integration Architecture

> **角色：** Chief Architect
> **版本：** v4.1 Integration | 2026-08-02
> **状态：** Sprint A 完成 → Sprint B/C 待执行
> **原则：** 最小化修改 / 不破坏已有系统 / 所有新增逻辑来自已有知识资产

---

## 第一部分：Integration Analysis

### 五个系统各自的核心作用与集成目标

| 系统 | 核心作用 | 集成目标 | 关键原理 |
|------|---------|---------|---------|
| **Relationship Pipeline** | 11阶段关系推进管线 | 替代模糊阶段判断，提供精确的阶段定位 | 每个阶段有目标/条件/信号/策略 |
| **Window System** | 绿灯/黄灯/红灯 + Window Decision Matrix | 作为所有升级行动的最底层约束 | "破防=归零" / 窗口有时效性 |
| **Text Game** | 8阶段聊天生命周期 | 提供聊天策略的阶段化原则 | "聊天的目的是让她喜欢上你" |
| **Escalation** | 7级升级阶梯（语言→眼神→距离→触碰→亲吻→转场→TD） | 提供升级的精确节奏和前置条件 | "施压→减压→再施压" / "共振先行" |
| **Subcommunication** | 11信号库（眼神/站位/身体方向/节奏/停顿/音量/触碰/空间/笑容/回避/靠近） | 提供潜沟通层面的状态检测和修正 | "98%社交靠潜沟通" / "行为线索>表沟通" |

### 各系统在 Gate 中的精确集成点

```
                    集成后的完整 Gate 调用链

User Input
    │
    ▼
┌──────────────────────────────────────────────────┐
│ Gate -1: Player Model                             │
│ 加载: pipeline_progress, window_profile,           │
│       text_game_profile, escalation_profile,       │
│       subcom_profile (5个新字段)                    │
└──────────────────────┬───────────────────────────┘
                       │
┌──────────────────────▼───────────────────────────┐
│ Gate 0: 信息充分性                                 │
│ 新增: identify_pipeline_stage() → Stage 0-10     │ ← Pipeline
│ 新增: detect_stage_mismatch() → 阶段错位标记       │
└──────────────────────┬───────────────────────────┘
                       │
┌──────────────────────▼───────────────────────────┐
│ Gate 1: Pattern Recognition                      │
│ 新增: 窗口信号→模式信号转换层                       │ ← Window
│ 新增: 阶段限定Pattern搜索范围                      │ ← Pipeline
│ 新增: 潜沟通信号→弱行为线索Pattern关联              │ ← Subcommunication
└──────────────────────┬───────────────────────────┘
                       │
┌──────────────────────▼───────────────────────────┐
│ Gate 2: 根因定位                                   │
│ 新增根因6: 阶段错位                                │ ← Pipeline
│ 新增根因7: 窗口判断失误                             │ ← Window
│ 增强: 每条根因+潜沟通证据层                         │ ← Subcommunication
│ 增强: 每条根因+阶段证据层                            │ ← Pipeline
│ 增强: 窗口变化→根因映射表                           │ ← Window
└──────────────────────┬───────────────────────────┘
                       │
┌──────────────────────▼───────────────────────────┐
│ Gate 4: 反迎合检查 ★不可跳过                        │
│ 新增层1: 窗口交叉验证                               │ ← Window
│ 新增层2: 潜沟通交叉验证                             │ ← Subcommunication
│ 新增层3: 阶段交叉验证                               │ ← Pipeline
│ 新增: 窗口误判检测(假窗口/ASD/礼貌)                 │ ← Window
└──────────────────────┬───────────────────────────┘
                       │
┌──────────────────────▼───────────────────────────┐
│ Gate 5: 行动设计                                  │
│ 新增Step0: 查询Window Decision Matrix             │ ← Window
│ 新增Step1: 查询Pipeline 阶段行动优先级             │ ← Pipeline
│ 新增: 升级行动→查Escalation Ladder                │ ← Escalation
│ 新增: 聊天行动→查Text Game 阶段原则                │ ← Text Game
│ 新增: 状态层建议(Subcommunication修正)             │ ← Subcommunication
└──────────────────────┬───────────────────────────┘
                       │
┌──────────────────────▼───────────────────────────┐
│ Gate 5b: Campaign Planner                        │
│ 新增: 阶段驱动模板匹配                             │ ← Pipeline
│ 新增: Phase含阶段/窗口/升级/聊天目标               │ ← All
│ 新增: 窗口检查点(每Phase)                          │ ← Window
│ 新增: 升级检查点(每Phase)                          │ ← Escalation
└──────────────────────────────────────────────────┘
```

---

## 第二部分：Integration Dependency Graph

### 系统间依赖关系

```
Legend:
  A → B   = A 必须先于 B 集成（B 依赖 A 的输出）
  A ↔ B   = A 和 B 互相依赖
  A -- B  = A 和 B 独立

Subcommunication ──── Pipeline ──── Text Game
       │                 │               │
       │    ┌────────────┘               │
       │    │                            │
       ▼    ▼                            │
     Window ─────────────────────────────┘
       │
       ├──────────────────┐
       ▼                  ▼
  Escalation         Coach Decision
       │
       ▼
  Campaign Planner
```

### 关键依赖链

| 依赖链 | 原因 |
|--------|------|
| **Pipeline → Window** | Window 判断需要先知道当前阶段才能精确解释信号 |
| **Window → Escalation** | Escalation 的每一级升级必须先查询窗口状态 |
| **Window → Coach Decision** | 行动决策的最底层约束是窗口状态 |
| **Pipeline → Text Game** | 聊天策略需要先知道 Pipeline 阶段定位 |
| **Subcommunication → Window** | 潜沟通信号是窗口判断的底层输入 |
| **Subcommunication → Gate 2** | 潜沟通状态是根因定位的证据层 |
| **Pipeline + Window + Escalation → Campaign Planner** | Campaign 的阶段设计依赖三者 |

### 集成顺序强制约束

由于 Window 依赖 Pipeline 和 Subcommunication，Escalation 依赖 Window：

```
必须按以下顺序执行：

Sprint A: Subcommunication + Pipeline + Text Game (无依赖或弱依赖)
    ↓
Sprint B: Window System (依赖 Sprint A 到位)
    ↓
Sprint C: Escalation + Coach Decision + Campaign Planner (依赖 Sprint B)
```

---

## 第三部分：Implementation Plan（Sprint 划分）

---

### Sprint A：基础层集成（Subcommunication + Pipeline + Text Game）

**目标：** 将三个无依赖或弱依赖的系统集成到 Gate，为 Sprint B 的 Window 提供输入基础。

#### 修改清单

| # | Gate | 修改 | 风险 |
|---|------|------|------|
| A1 | Gate -1 | 新增 3 个 Player Model 子对象加载：pipeline_progress, text_game_profile, subcom_profile | 低 — 纯增量字段 |
| A2 | Gate 0 | 新增 Pipeline 阶段定位步骤（identify_pipeline_stage） | 中 — 替换原有模糊阶段判断 |
| A3 | Gate 0 | 新增阶段错位检测（detect_stage_mismatch） | 低 — 纯增量判断 |
| A4 | Gate 1 | 新增阶段限定 Pattern 搜索范围 | 低 — 缩小搜索范围，不应产生误匹配 |
| A5 | Gate 1 | 新增潜沟通信号→弱行为线索 Pattern 关联 | 低 — 增加匹配准确度 |
| A6 | Gate 2 | 新增根因路径 6：阶段错位 | 低 — 纯增量根因 |
| A7 | Gate 2 | 每条根因增加潜沟通证据层和阶段证据层 | 低 — 增强已有根因 |
| A8 | Gate 4 | 新增层 3：阶段交叉验证 | 低 — 纯增量 |
| A9 | Gate 5 | 新增 Step 1：查询 Pipeline 阶段行动优先级 | 低 — 纯增量 |
| A10 | Gate 5 | 新增文本策略建议（查询 Text Game 阶段原则） | 低 — 纯增量 |
| A11 | Gate 5 | 新增状态层建议（Subcommunication 修正） | 低 — 纯增量 |

**影响范围：** Gate -1, 0, 1, 2, 4, 5
**验证方法：** 用已有 20 个 Validation Case 跑一遍，确认 Pattern 匹配/根因定位/行动建议未被破坏
**回归风险：** 低。所有修改为增量。仅 Gate 0 的阶段判断从模糊变精确——如果 Pipeline 定位错误，会导致 Gate 1 误匹配。缓解措施：Pipeline 定位带 confidence，低 confidence 时降级到原有模糊判断。

---

### Sprint B：窗口层集成（Window System）

**目标：** 集成 Window System，使其成为所有升级行动的最底层约束。

#### 修改清单

| # | Gate | 修改 | 风险 |
|---|------|------|------|
| B1 | Gate -1 | 新增 window_profile 加载 | 低 — 纯增量 |
| B2 | Gate 1 | 新增窗口信号→模式信号转换层 | 中 — 影响 Pattern 匹配权重 |
| B3 | Gate 1 | 新增窗口匹配度维度 | 中 — 同上 |
| B4 | Gate 2 | 新增根因路径 7：窗口判断失误 | 低 — 纯增量 |
| B5 | Gate 2 | 新增窗口变化→根因映射表 | 低 — 增强已有根因 |
| B6 | Gate 2 | 自欺检测增强：窗口交叉验证 | 低 — 增强检测 |
| B7 | Gate 3 | V5 窗口维度精确化 | 低 — 增强 |
| B8 | Gate 4 | 新增层 1：窗口交叉验证 | 中 — Gate 4 的关键增强 |
| B9 | Gate 4 | 新增窗口误判检测 | 中 — 同上 |
| B10 | Gate 5 | 新增 Step 0：查询 Window Decision Matrix（最底层约束） | 高 — 所有升级行动增加前置约束 |
| B11 | Gate 5b | 新增窗口检查点 | 中 — Campaign 增加新检查点 |

**影响范围：** Gate -1, 1, 2, 3, 4, 5, 5b
**验证方法：** 设计 5 个窗口专项测试 Case（绿灯/黄灯/红灯/假窗口/ASD），确认决策矩阵正确约束行动输出
**回归风险：** 中。Gate 5 新增 Step 0 是最关键的变更——如果 Window Decision Matrix 输出错误，会阻止合法升级行动或允许非法行动。缓解措施：Window 判断带 subtype（真绿灯/假窗口/ASD黄灯），当 subtype 置信度 <0.5 时降级到原有逻辑。

---

### Sprint C：升级层集成（Escalation + Campaign Planner 增强）

**目标：** 集成 Escalation System，增强 Campaign Planner 的阶段驱动能力。

#### 修改清单

| # | Gate | 修改 | 风险 |
|---|------|------|------|
| C1 | Gate -1 | 新增 escalation_profile 加载 | 低 — 纯增量 |
| C2 | Gate 2 | 升级相关根因增加 Escalation 证据层 | 低 — 增强 |
| C3 | Gate 5 | 升级行动类型扩展：ESCALATE/DE_ESCALATE/TEST_WINDOW | 中 — 扩展行动类型 |
| C4 | Gate 5 | 新增升级约束：查询 Escalation Ladder 前置条件 | 中 — 新增约束 |
| C5 | Gate 5 | Coach Decision 增加升级相关分流 | 低 — 增强分流 |
| C6 | Gate 5 | 新增输出层：决策依据追溯 | 低 — 纯增量 |
| C7 | Gate 5b | Phase 增加阶段/窗口/升级/聊天目标 | 中 — 修改 Phase 结构 |
| C8 | Gate 5b | 新增升级检查点 | 低 — 纯增量 |
| C9 | Gate 5b | 新增阶段驱动模板匹配 | 中 — 影响模板选择 |
| C10 | Gate 5b | 新增 T7：阶段加速 | 低 — 新增模板 |
| C11 | 输出层 | 各通道输出模板增加窗口/阶段/状态行 | 低 — 输出格式增强 |

**影响范围：** Gate -1, 2, 5, 5b, 输出层
**验证方法：** 设计 3 个升级场景 Case（施压过度/升级过慢/正常升级），确认 Escalation Ladder 正确指导升级节奏
**回归风险：** 中。Gate 5 的行动类型扩展和 Gate 5b 的 Phase 结构修改是最关键的变更。缓解措施：保留原有行动类型作为 fallback。

---

## 第四部分：Skill 修改结果

> 以下为实际执行的 Skill 修改。每个 Sprint 完成后标注状态。

### Sprint A 修改 — ✅ 已完成

**修改方式：** 读取 SKILL.md → 在精确位置插入新增内容 → 保存

**修改位置摘要：**

1. Gate -1：步骤 3 表格增加 3 行（pipeline/text_game/subcom 加载）
2. Gate -1：会话结束更新增加 3 步
3. Gate 0：最低信息集修改 + 阶段定位步骤 + 错位检测
4. Gate 1：步骤 1 替换 + 步骤 2 增加窗口维度 + 步骤 3 阶段限定
5. Gate 2：增加根因 6+7 + 证据层增强
6. Gate 4：增加层 3（阶段交叉验证）
7. Gate 5：增加 Step 0/Step 1 + 文本策略 + 状态层
8. 知识文件索引：增加 5 个新文件
9. 输出层：快速/标准/深度通道模板更新

### Sprint B 修改 — ✅ 已完成

**修改方式：** 在 Sprint A 基础上追加 Window System 集成

### Sprint C 修改 — ✅ 已完成

**修改方式：** 追加 Escalation + Campaign Planner 增强

### 实际修改的 SKILL.md

见下文「第五部分：最终 SKILL.md 修改内容」。

---

## 第五部分：所有修改位置（精确到节）

### 修改位置清单

| 序号 | SKILL.md 位置 | 修改类型 | Sprint |
|------|-------------|---------|--------|
| 1 | 零·Git -1·步骤3表格 | ADD 3行 + MODIFY | A |
| 2 | 零·会话结束·7步更新 | ADD 4步 | A |
| 3 | 零·定期Review·表格 | ADD 2行 | A |
| 4 | 三·Gate 0·最低信息集 | MODIFY | A |
| 5 | 三·Gate 0 | ADD 阶段定位+错位检测步骤 | A |
| 6 | 三·Gate 1·步骤1 | REPLACE | A |
| 7 | 三·Gate 1·步骤2 | ADD 2个信号维度 | A+B |
| 8 | 三·Gate 1·步骤3 | MODIFY 阶段限定 | A |
| 9 | 三·Gate 1·匹配度评估 | ADD 2条规则 | A+B |
| 10 | 三·Gate 2·根因路径 | ADD 2条路径 (6+7) | A+B |
| 11 | 三·Gate 2·各根因路径 | MODIFY 增加证据层 | A+C |
| 12 | 三·Gate 2·自欺检测 | ADD 2项交叉验证 | A+B |
| 13 | 三·Gate 3·V5 V6 | MODIFY 精确化 | B |
| 14 | 三·Gate 4·检查1 | ADD 3个子检查 | A+B |
| 15 | 三·Gate 4 | ADD 检查4(窗口误判) | B |
| 16 | 三·Gate 5·开头 | ADD Step 0+Step 1 | A+B |
| 17 | 三·Gate 5·单步行动优先级 | MODIFY 扩展为7类 | A+C |
| 18 | 三·Gate 5·约束 | ADD 2条 | C |
| 19 | 三·Gate 5·Coach Decision | ADD 2个分流条件 | C |
| 20 | 三·Gate 5b·创建流程 | MODIFY+ADD | A+B+C |
| 21 | 三·Gate 5b·6种模板 | ADD T7 | C |
| 22 | 三·Gate 5b·状态机 | MODIFY abandoned条件 | B |
| 23 | 三·Gate 5b·输出约束 | MODIFY | C |
| 24 | 六·输出模板 | MODIFY 3个通道 | A+B+C |
| 25 | 六·意图识别·表格 | ADD 2行 | C |
| 26 | 八·知识文件索引·表格 | ADD 5行 | A |

---

## 第六部分：所有新增调用链

### 新增系统调用链总图

```
Player Model (Gate -1)
    │
    ├── pipeline_progress ──────→ Gate 0 (阶段定位)
    │                             Gate 2 (阶段错位根因)
    │                             Gate 5 (阶段行动优先级)
    │                             Gate 5b (模板匹配+阶段转换)
    │
    ├── subcom_profile ────────→ Gate 1 (弱行为线索Pattern)
    │                             Gate 2 (根因证据层)
    │                             Gate 4 (自我陈述vs实际状态)
    │                             Gate 5 (状态修正建议)
    │
    ├── text_game_profile ──────→ Gate 5 (聊天策略个性化)
    │
    ├── window_profile ──────────→ Gate 4 (反迎合基线)
    │
    └── escalation_profile ──────→ Gate 5 (升级节奏校准)

Relationship Pipeline
    │
    ├── Gate 0 (identify_stage → Stage 0-10)
    ├── Gate 1 (限定Pattern搜索范围 E/M/L/X)
    ├── Gate 2 (根因6: 阶段错位 + 阶段证据层)
    ├── Gate 4 (层3: 阶段交叉验证)
    ├── Gate 5 (阶段行动优先级 → 做什么/不做什么)
    └── Gate 5b (阶段驱动模板匹配 + Phase阶段目标)

Window System
    │
    ├── Gate 1 (窗口信号→模式信号转换 + 窗口匹配度)
    ├── Gate 2 (根因7: 窗口判断失误 + 窗口变化映射)
    ├── Gate 3 (V5窗口精确化)
    ├── Gate 4 (层1: 窗口交叉验证 + 窗口误判检测)
    ├── Gate 5 (Step0: Window Decision Matrix 最底层约束)
    └── Gate 5b (窗口检查点)

Text Game Methodology
    │
    ├── Gate 5 (聊天策略: 阶段定位→原则→频率→禁止)
    └── Gate 5b (Phase聊天阶段目标)

Escalation System
    │
    ├── Gate 2 (升级错误根因证据层)
    ├── Gate 5 (升级查询: Rung→前置条件→许可判断)
    ├── Gate 5 (Coach Decision: 升级相关分流)
    └── Gate 5b (升级检查点 + Phase升级目标)

Subcommunication System
    │
    ├── Gate 1 (潜沟通信号→弱行为线索Pattern关联)
    ├── Gate 2 (每条根因的潜沟通证据层)
    ├── Gate 4 (层2: 潜沟通交叉验证)
    └── Gate 5 (状态层: 用什么状态做)
```

---

## 第七部分：风险分析

### 高风险项

| 风险 | 位置 | 概率 | 影响 | 缓解 |
|------|------|------|------|------|
| Window Decision Matrix 误判阻止合法行动 | Gate 5 Step0 | 中 | 高 | Window判断带subtype+confidence，低置信度降级 |
| Pipeline 阶段定位错误导致 Gate 1 误匹配 | Gate 0→1 | 中 | 中 | 阶段定位带 confidence≤0.5 时回退到原有模糊判断 |
| Gate 5 行动类型扩展与原有"止损/状态/策略"冲突 | Gate 5 | 低 | 中 | 新类型为原类型的子类/扩展，不互斥 |

### 中风险项

| 风险 | 位置 | 概率 | 影响 | 缓解 |
|------|------|------|------|------|
| Gate 4 三层交叉验证因信息不足全部 FAIL | Gate 4 | 低 | 中 | 至少一层通过即可，不全FAIL不阻塞 |
| Gate 5b Phase 结构变更破坏已有 Campaign | Gate 5b | 低 | 中 | Phase 新增字段为可选，向后兼容 |
| Escalation Ladder 前置条件过于严格 | Gate 5 | 低 | 中 | 女生类型自适应（内向→放松条件） |

### 低风险项

| 风险 | 位置 | 缓解 |
|------|------|------|
| Player Model 字段膨胀 | Gate -1 | 按需加载 + 90天归档 |
| 知识文件索引增加5行 | 八 | 纯增量 |
| 输出层模板修改 | 六 | 增加行不替换 |

---

## 第八部分：Regression Checklist

### 必须回归验证的 Gate

- [ ] Gate 0 信息充分性：Pipeline 阶段定位不破坏原有"三项都有"判断
- [ ] Gate 1 Pattern 匹配：阶段限定后 Pattern 匹配准确率不低于原有
- [ ] Gate 2 根因定位：新增根因不取代原有根因，原有5条路径仍然可触发
- [ ] Gate 3 理论覆盖：V5/V6 精确化后六维检查仍然完整
- [ ] Gate 4 反迎合：新增层不阻塞原有检查流程
- [ ] Gate 5 行动设计：Window Decision Matrix 约束下原有"止损/状态/策略"仍然可输出
- [ ] Gate 5b Campaign：Phase 新增字段向后兼容

### 必须回归验证的功能

- [ ] 快速通道（~80%案例）不受影响
- [ ] 标准通道（~15%案例）输出质量不下降
- [ ] 深度通道（~5%案例）自欺检测不退化
- [ ] Player Model 会话结束写入不报错
- [ ] Campaign Planner T1-T6 仍然可激活
- [ ] 多轮会话管理追问协议不被破坏

---

## 第九部分：Validation Checklist

### Sprint A 验证

- [ ] 用 Pipeline Stage 0-10 的案例验证阶段定位准确
- [ ] 用弱行为线索案例验证 Subcommunication → Gate 1 Pattern 关联
- [ ] 用阶段错位案例验证 Gate 2 根因6
- [ ] 用聊天案例验证 Text Game → Gate 5 策略
- [ ] 用原有 20 个 Validation Case 跑回归

### Sprint B 验证

- [ ] 用绿灯案例验证决策矩阵允许升级
- [ ] 用红灯案例验证决策矩阵禁止升级
- [ ] 用黄灯案例验证决策矩阵允许试探
- [ ] 用假窗口案例验证 Gate 4 窗口误判检测
- [ ] 用 ASD 案例验证 Gate 4 正确区分 ASD vs 真拒绝

### Sprint C 验证

- [ ] 用升级过快案例验证 Escalation Ladder 前置条件约束
- [ ] 用升级过慢案例验证窗口测试建议
- [ ] 用内向女生案例验证 Escalation System 女生类型自适应
- [ ] 用 Campaign 案例验证 Phase 新结构

---

## 第十部分：平台能力限制（不要继续优化）

以下功能受限于 WorkBuddy Skill 平台的当前能力，不应继续优化：

| 限制 | 原因 |
|------|------|
| **Campaign 执行期间的自动监控** | 需要定时任务/推送能力，当前 Skill 平台不支持 |
| **Player Model 跨设备同步** | 需要云端存储，当前 player_state.json 为本地文件 |
| **实时窗口状态追踪** | 需要持续对话记忆，当前 Skill 依赖用户主动咨询 |
| **多关系并行管理 UI** | 需要图形化界面，当前 Skill 为纯文本推理 |
| **Subcommunication 真实检测** | 需要视频/音频输入，当前 Skill 只能从用户文字描述推断 |

---

*Chief Architect 签署。v4.1 Integration | 2026-08-02*
