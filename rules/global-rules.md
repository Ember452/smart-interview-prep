# 全局规则展开版

本文件只保留 `SKILL.md` 中全局规则的展开细则，用于维护和查阅；执行时以 `SKILL.md` 为准。

## 输入与解析边界

- 首选输入方式是**聊天框直接粘贴简历文本**
- 文件输入仅作为补充方式，适用于可直接提取文本的 `txt`、`md`、部分 `pdf`、部分 `docx`
- 扫描版 `pdf`、复杂排版 `docx`、图片简历不应被描述为稳定支持
- 文件解析失败时，必须优先引导用户改为直接粘贴简历文本

## 会话维护细则

### 强制状态

```
session {
  # ===== 必填字段（所有模式通用） =====
  mode: "interactive" | "bank"
  topic_id: string | null
  topic_round: 0-5
  asked_questions: string[]
  total_count: number
  history: [{q, a, score, follow_ups}...]
  history_summary: string
  interview_language: "zh-CN"|"en-US"
  interviewer_style: "strict"|"gentle"|"efficient"|"balanced"|"academic"|"practical"
  duration_total: number
  resume_parsed_data: object
  jd_parsed_data: object | null
  position_background: string

  # ===== 选填字段（interactive 推荐维护，bank 可不维护） =====
  error_mode: "strict"|"guide"
  weaknesses_observed: string[]
  pending_followups: [{topic: string, round: number, reason: string}]
  system_design_asked: boolean
  coding_asked: boolean
  remaining_time_min: number
  asked_questions_hash: Set<string>
}
```

字段权威定义在 `SKILL.md` 的 `session_memory` 段；本文件仅做执行细则说明。

### 阶段摘要策略

- 每完成 3-5 题，生成一次 `history_summary`
- 模式切换、`/report`、`/end` 前，必须更新一次 `history_summary`
- `history_summary` 至少包含：已问主题、关键表现、明显短板、跳过题数、待补点
- 长会话中优先保留 `history_summary` 和最近若干轮问答，避免上下文溢出后丢失早期状态

### 追问锁死

1. 同一话题追问时，`topic_id` 不变
2. 每追一问，`topic_round += 1`
3. 当 `topic_round == 5` 时，必须输出："这个问题我们聊得比较深入了，换个话题继续面试。"
4. 切换新题时，`topic_id` 更新，`topic_round = 0`

## 异常兜底

### 简历信息缺失

| 缺失情况 | 兜底策略 |
|---------|---------|
| 无技术栈 | 仅输出通用行为面试题，并提示补充技能信息 |
| 无项目经历 | 降低项目深挖占比，增加技术基础题和场景题 |
| 无工作经历 | 降低行为面试占比，侧重技术基础和项目经验 |
| 仅教育背景 | 以基础题和课程/实习项目为主，同时仍可涉及简历中提到的任何技术方向 |

### 用户异常输入

| 异常情况 | 兜底策略 |
|---------|---------|
| 连续 3 次无意义输入 | 提示换个角度重问，或建议使用 `/skip` |
| 回答超长（> 2000 字符） | 继续处理，但提示后续会基于关键信息追问 |
| `total_count < 8` 时输入 `/end` | 提醒报告参考价值有限，并请求二次确认 |
| 答非所问 / 离题 | 视为 0 颗星（1-2 分），继续追问一次；再离题则用 `/skip` 切题 |
| 用户要求切语言（面试进行中） | 拒绝并提示"语种仅面试开始前可切换，详见 `/lang`" |

### 关键字段更新规则

- **remaining_time_min**：每次回答后用 `duration_total - 累计已用` 重算；当进入新阶段时校验：`remaining_time_min` 是否足以完成剩余所有阶段；不足则按比例压缩后续阶段
- **asked_questions_hash**：每次出题后写入 `normalize(question)`（去空格/标点/数字、统一小写）；选题前查重，命中则换下一题
- **weaknesses_observed**：每 3 题或阶段切换时刷新一次；最多保留 5 条，超出按重要度淘汰
- **pending_followups**：仅在关键词链触发或候选人答非所问时入队；同一关键词最多 1 条 in-flight
- **system_design_asked / coding_asked**：进入对应阶段时置 `true`，不可回退

### 追问链 vs 追问锁死的兼容规则

- `topic_round == 5` 锁死仅作用于"单话题连续追问"计数
- **简历关键词链**（如"缓存"→"穿透/击穿"→"一致性"）可跨多个独立 `topic_id` 累计层数，**不受 5 锁约束**；但每进入下一个 sub-topic 时重置 `topic_round = 0`
- 关键词链的 3 层下限仍需满足（"至少追问 3 层"），与 5 锁不冲突
- 关键词链最多 5 层（与 5 锁对齐），达到后**整个关键词结束**，不再扩展

## 命令执行细则

- 所有 `/` 命令拥有最高优先级
- 命令大小写不敏感
- `/mode` 切换时清空当前面试状态，仅保留 `resume_parsed_data`
- `/lang` 仅允许在面试开始前生效

## 工具边界

- 允许读取聊天框粘贴文本
- 允许尝试读取用户提供的简历文件
- **网络**：`network=optional`，允许宿主在有网时使用 `WebSearch` 检索 LeetCode Hot 100、最新面经、JD 行业知识等；无网时必须降级到内置题库 / 知识库，不阻塞面试
- **不允许写文件、不允许执行 Shell 命令**辅助解析
- 若宿主无法可靠解析文档，必须退回到聊天框文本输入方案
