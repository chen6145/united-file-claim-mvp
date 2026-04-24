---
tags:
  - research
  - voice-ai
  - twilio
  - vapi
  - united
date: 2026-04-23
status: draft
---

# United file claim 电话 agent MVP
*2026-04-23*

## 目标

做一个 `Twilio + Vapi` 的电话 agent MVP，目标是替用户给 `United Airlines` 打电话，完成：

- 自动过 IVR
- 自动等待真人客服
- 向客服说明 claim / reimbursement 问题
- 收集 claim 所需信息
- 必要时把电话转给用户本人
- 保存全部录音和全部转写，便于复盘和继续跟进

## 假设

这个 MVP 不假设：
- AI 能在所有情况下完全替代用户
- United 一定允许全流程纯电话完成 claim
- claim 一定能在一通电话里闭环

这个 MVP 假设：
- 第一版的成功定义是：`AI 成功到达真人客服 + 完成 claim intake / case creation / next-step clarification`
- 如果出现身份验证、OTP、本人确认、支付或敏感争议，系统应该 `transfer to user`
- 如果最终需要网页上传材料，这通电话的价值仍然成立，因为 AI 可以拿到：
  - claim reference number
  - submission instructions
  - next step
  - fax/email/upload path

## 成功定义

### MVP 成功

满足以下任一即可：

1. AI 成功联系到 United 真人客服，并完成 claim intake
2. AI 成功拿到 claim case number / reference number
3. AI 成功确认下一步所需文件与提交路径
4. AI 在需要本人确认时，成功 transfer 给用户

### MVP 失败

- 卡死在 IVR
- 长时间等待但没正确识别真人接通
- 与客服多轮误解，无法推进
- 未在需要本人验证时及时 handoff
- 通话结束后没有完整音频和转写可复盘

## 共用技术栈

不管采用哪种编排方案，底层基础设施相同：

- 电话网络：`Twilio`
- Voice agent runtime：`Vapi`
- 路线：`STT -> text LLM -> TTS`
- 后端：`FastAPI` 或 `Node.js`
- 数据库：`Postgres / Supabase`
- 对象存储：`S3 / Supabase Storage / GCS`
- 结构化日志：`Postgres`

## 共用基础能力

### 1. 电话层

- `Twilio` 发起 outbound call
- `Vapi` 托管 assistant、语音链路、tools、transferCall、dtmf
- 用户的手机号码作为 `transfer destination`

### 2. 录音与转写留存

MVP 明确要求：

- 保存整通电话录音
- 保存完整 transcript
- 保存 partial/final transcript 事件
- 保存 tool 调用时间线
- 保存 transfer/handoff 时间线
- 保存通话元数据

### 3. 最低留存字段

每通电话至少保存：

- `call_id`
- `user_id`
- `twilio_call_sid`
- `vapi_call_id`
- `started_at`
- `ended_at`
- `duration_seconds`
- `call_result`
- `agent_result`
- `full_audio_url`
- `transcript_json`
- `final_summary`
- `tool_timeline_json`
- `transfer_timeline_json`
- `claim_reference_number`
- `next_step`

### 4. 录音与转写实现建议

- 原始录音：优先保存 `Twilio` call recording
- 实时/最终转写：保存 `Vapi` transcript 事件
- 最终结构化纪要：由后端在 post-call 阶段生成

建议把留存拆成三层：

1. 原始层  
   原始录音、原始 transcript 事件、原始 webhook payload

2. 结构化层  
   claim number、agent success/failure、是否 handoff、下一步动作

3. 复盘层  
   摘要、失败原因、可改进点、下次重拨建议

### 5. 合规注意

如果通话涉及录音，MVP 里至少要考虑：

- 是否需要录音告知
- 是否由 United 侧已做录音告知
- 你的系统是否需要额外告知用户

第一版不要跳过这个问题。

## 工具定义

两种方案共用的 tool 集合建议如下：

- `lookup_user_claim_context`
- `lookup_flight_info`
- `lookup_baggage_info`
- `lookup_user_profile`
- `append_claim_notes`
- `mark_transfer_required`
- `transfer_to_user`
- `save_claim_reference`
- `save_next_step`
- `end_call_with_result`

如果第一版做得更轻，可以只保留：

- `lookup_user_claim_context`
- `append_claim_notes`
- `transfer_to_user`
- `save_claim_reference`
- `save_next_step`
- `end_call_with_result`

## 关键坑点与解决策略

下面这些问题，不管你选 `方案 A` 还是 `方案 B`，都会遇到。区别只在于：你是显式解决，还是等它在 demo 时炸出来。

### 坑 1：LLM 擅长“说什么”，但不擅长“系统现在处于什么状态”

电话里真正关键的是：

- 现在在 `IVR` 还是真人客服
- 是在排队还是已经接通
- 刚刚是不是转部门了
- 现在该按键还是该说话
- 是否已经进入必须本人确认的阶段

这些不是自然对话问题，而是系统状态问题。

#### 解决方式

第一版不要让 LLM 自己“脑补状态”，而要显式维护最小状态：

- `ivr`
- `waiting`
- `human_connected`
- `claim_intake`
- `needs_user`
- `done`
- `failed`

后端需要保存一个 `call_state`，并把它作为每轮推理上下文的一部分传给 LLM。

#### 最小实现

每次 LLM 调用时，额外注入：

- `current_state`
- `goal`
- `known_facts`
- `allowed_actions`
- `transfer_rules`

这样 LLM 不是自由发挥，而是在当前状态里做局部决策。

#### 工程规则

- LLM 可以提出状态建议
- 但真正的状态转移由 orchestration layer 执行
- 状态变化必须写日志

### 坑 2：纯 LLM 在边界条件上很容易飘

常见表现：

- 把 hold music 当成人声
- 在不该说话的时候抢话
- 不知道该继续等还是该重试
- 没意识到已经到了该 transfer 的时机
- 对方问到 `DOB / OTP / policy number` 时还硬接着聊

#### 解决方式

不要只靠 prompt，至少要补三层 guardrails：

1. 事件层 guardrails  
   用 telephony / transcript 事件判断：
   - 是否长时间无人声
   - 是否刚进入 hold
   - 是否检测到真人问候
   - 是否发生 transfer / redirect

2. 内容层 guardrails  
   用关键字和规则检测：
   - `date of birth`
   - `verification code`
   - `policy number`
   - `account holder`
   - `can I speak with ...`

3. 次数层 guardrails  
   用计数器限制：
   - 连续误解轮数
   - 工具失败次数
   - 长时间无进展次数

#### 最小实现

后端维护这些 counters：

- `misunderstanding_count`
- `tool_failure_count`
- `no_progress_turn_count`
- `hold_duration_seconds`

然后在每轮决策前，先跑规则，再决定：

- 继续让 LLM 回复
- 转为等待
- 触发 transfer
- 结束通话

### 坑 3：“搞不定就 transfer” 这句话本身太模糊

“搞不定”不是一个可执行条件。

如果没有规则，LLM 常见的两个极端是：

- 明明该 transfer 了却还在硬聊
- 明明可以自己处理却太早 transfer

#### 解决方式

把 transfer 触发条件显式化。

### 必须触发 transfer

出现以下任一情况，直接进入 `needs_user`：

- 对方要求本人确认
- 出现 `OTP / DOB / SSN / policy verification`
- 对方明确要求和 `account holder / passenger` 通话
- 连续 `2` 轮沟通失败
- 工具调用失败超过 `2` 次
- LLM 输出 `uncertain` 或显式建议升级

### 不应触发 transfer

以下情况默认由 AI 继续处理：

- 普通 FAQ
- claim 背景说明
- case number / reference number 记录
- 所需材料确认
- 回复时间 / 下一步说明确认
- 简单 claim intake

### 建议的 transfer policy

定义一个后端函数：

`should_transfer(call_state, transcript_window, counters, tool_status) -> bool`

而不是让 LLM 自己凭感觉决定。

#### 最小伪代码

```text
if asks_for_user_identity:
    transfer
elif contains_sensitive_verification:
    transfer
elif misunderstanding_count >= 2:
    transfer
elif tool_failure_count >= 2:
    transfer
else:
    continue
```

### 坑 4：让 LLM 管对话，让规则管编排

这是第一版最重要的切分原则。

#### LLM 负责

- 听懂意图
- 自然对话
- 收集信息
- 决定调用哪个 tool
- 生成对客服的话术

#### 规则 / 状态机负责

- 什么时候按 `DTMF`
- 什么时候静音等待
- 什么时候开始 transfer
- 哪些信息绝不能由 AI 自己回答
- 连续失败几次后必须 handoff
- `tool timeout / retry / fallback`

#### 为什么这样切

因为：

- LLM 负责局部智能
- 规则层负责系统稳定性

如果全交给规则，系统会很僵。  
如果全交给 LLM，系统会很飘。

### 坑 5：具体到这个 MVP，第一版建议怎么落地

如果你明天就要做 demo，我建议的最小组合是：

#### 状态

- `ivr`
- `waiting`
- `human_connected`
- `claim_intake`
- `needs_user`

#### 允许 LLM 自由发挥的范围

- 客服接通后的开场说明
- claim 背景描述
- 跟客服确认 case number / next step
- 一般性澄清对话

#### 不允许 LLM 自由决定的范围

- 何时按键
- 何时结束等待
- 何时进入 `needs_user`
- 何时结束通话
- 是否回答高风险身份验证问题

#### 推荐实现顺序

1. 先实现 transcript + recording 全量留存
2. 再实现 transfer policy
3. 再实现最小状态字段
4. 最后才优化 prompt

如果顺序反过来，通常会在 demo 前夜被边界情况拖死。

## 方案 A：纯 LLM + Prompt Engineering + Tool Call

这个方案的核心思想是：

- 不显式建复杂状态机
- 让 LLM 尽量自主处理电话流程
- 靠 prompt、tool schema、guardrails 和 transfer policy 控制风险

### 架构图

```text
Twilio
  |
Vapi assistant
  |
STT -> text LLM -> TTS
  |
Tool calls -> backend
  |
Storage / logs / recordings
```

### 控制方式

控制主要来自：

- system prompt
- tool definitions
- transfer policy
- post-tool instructions
- refusal / escalation rules

### Prompt 核心要求

prompt 至少要写清楚：

- 你的角色：代表用户致电 United file claim
- 首要目标：到达真人客服并推进 claim intake
- 不可做事项：
  - 不得捏造信息
  - 不得猜测 policy/account details
  - 遇到 OTP / 本人确认必须 transfer
- 可做事项：
  - 解释 claim 背景
  - 提供用户已授权的信息
  - 请求 case/reference number
  - 请求下一步说明
- Transfer 规则：
  - 客服要求本人确认
  - 需要 DOB / OTP / payment / legal acknowledgment
  - 连续两次沟通失败
  - LLM 不确定是否应继续时

### LLM 行为策略

LLM 的默认策略应该是：

1. 先快速识别当前是不是 IVR
2. 如果是 IVR，优先少说、多听、谨慎按键
3. 一旦到真人，先用一句简洁的话描述 claim 目的
4. 尽量收集：
   - case number
   - claim path
   - 所需材料
   - SLA / 回复时间
5. 只要进入高风险验证阶段，立即 transfer

### 方案 A 的优点

- 开发最快
- Prompt 改起来快
- 很适合 hackathon
- 对自然对话比较友好
- 工具接入简单

### 方案 A 的缺点

- 对 IVR、hold、转接、边界情况不够稳
- 行为更难预测
- 失败时不容易知道是 prompt 问题还是流程问题
- 纯靠“搞不定就 transfer”容易太晚触发

### 适合的 MVP 边界

如果选方案 A，建议把目标压低到：

- `到达真人客服`
- `完成问题描述`
- `拿到 claim reference / next step`

不要把第一版目标定成：
- 无人干预完整结案

### 方案 A 的推荐实现

- 一个主 assistant
- 一套强 system prompt
- 一个很薄的 tool backend
- 一个明确的 `transfer_to_user` 规则集
- 一个 post-call evaluator

## 方案 B：状态机编排方案

这个方案的核心思想是：

- 大流程由状态机控制
- LLM 只负责每个状态内的语言理解和回复
- 电话编排不完全交给模型自由发挥

### 架构图

```text
Twilio
  |
Vapi assistant
  |
STT -> text LLM -> TTS
  |
Orchestrator / State Machine
  |-- call state
  |-- tool policy
  |-- transfer policy
  |-- timeout / retry rules
  |
Tool calls -> backend
  |
Storage / logs / recordings
```

### 最小状态机

第一版建议只用这几个状态：

- `ivr`
- `waiting`
- `human_connected`
- `claim_intake`
- `needs_user`
- `done`
- `failed`

### 状态定义

#### `ivr`

目标：
- 识别菜单层级
- 发送必要 DTMF
- 尽快进入真人队列

允许动作：
- `press_digits`
- `wait`
- `speak_short`
- `transfer_to_user`

#### `waiting`

目标：
- 保持在线
- 识别真人接通

允许动作：
- `wait`
- `speak_short`
- `transfer_to_user`

#### `human_connected`

目标：
- 用一句简洁的话解释来电目的
- 确认当前客服是否能处理 claim / reimbursement

允许动作：
- `speak`
- `call_tool`
- `transfer_to_user`
- `end_call`

#### `claim_intake`

目标：
- 收集并确认：
  - case number
  - required documents
  - next step
  - expected turnaround time

允许动作：
- `speak`
- `call_tool`
- `transfer_to_user`
- `end_call`

#### `needs_user`

目标：
- 把电话安全转给用户
- 在转前给用户一段简短摘要

允许动作：
- `play_summary`
- `transfer_to_user`
- `wait`

### 转移规则

#### `ivr -> waiting`

当：
- 完成 DTMF 菜单选择
- 或检测到 hold / queue

#### `waiting -> human_connected`

当：
- 检测到真人客服问候语

#### `human_connected -> claim_intake`

当：
- 客服确认能处理该 claim

#### `human_connected -> needs_user`

当：
- 客服要求本人确认
- 需要 DOB / OTP / 敏感个人信息

#### `claim_intake -> needs_user`

当：
- 客服需要用户本人接手完成验证或确认

#### `claim_intake -> done`

当：
- 已拿到 case/reference number
- 或已拿到明确 next step

#### `* -> failed`

当：
- 多轮沟通失败
- 长时间无进展
- transfer 失败且无法恢复

### 方案 B 的优点

- 对 IVR / hold / transfer / handoff 更稳
- 更适合客服电话场景
- debug 更容易
- 可以更明确地控制何时 transfer 给用户
- 更适合做 post-call 评估

### 方案 B 的缺点

- 开发更慢
- 需要自己写 orchestrator
- 需要维护状态转移规则
- 灵活性略低于纯 LLM 方案

### 适合的 MVP 边界

如果选方案 B，可以把目标抬高到：

- `AI 自主过 IVR`
- `AI 自主等待真人`
- `AI 自主完成 claim intake`
- `只在验证或高风险节点 transfer`

## 两种方案的 Tradeoff

### 方案 A 更像

- LLM-first agent
- 快速 demo
- 低工程投入

### 方案 B 更像

- Telephony-first workflow
- 稳定生产雏形
- 明确的流程控制

### 对这个具体 use case 的判断

`United file claim` 这个用例不是一个轻任务，因为它天然带有：

- IVR
- 等待真人
- 高概率身份确认
- claim number / next step 收集
- 可能的转部门

所以如果只问一句“哪个更适合这个用例”：

- `方案 A` 更适合 hackathon demo
- `方案 B` 更适合做真正能反复跑的系统

## 我建议的 MVP 路线

如果你是为了明天 hackathon：

### Day 1 目标

先做一个偏 `方案 A` 的版本，但加入最薄的一层显式规则：

- 明确 transfer 条件
- 明确录音/转写留存
- 明确 post-call evaluation

也就是：

- `LLM-first`
- 但不是完全无状态

### Day 2 / 下一阶段

再往 `方案 B` 升级，把：

- `ivr`
- `waiting`
- `human_connected`
- `needs_user`

这几个状态显式化。

## MVP 接口设计

### 1. `POST /calls/start`

用途：
- 发起一次 United claim 电话

输入：
- `user_id`
- `claim_type`
- `flight_info`
- `claim_context`
- `transfer_phone_number`

输出：
- `call_id`
- `vapi_call_id`

### 2. `POST /tools/lookup_user_claim_context`

返回：
- 用户 claim 摘要
- 航班信息
- 已知材料

### 3. `POST /tools/append_claim_notes`

返回：
- notes 已保存

### 4. `POST /tools/transfer_to_user`

输入：
- `call_id`
- `reason`
- `summary_for_user`

返回：
- transfer initiated / failed

### 5. `POST /tools/save_claim_reference`

输入：
- `claim_reference_number`
- `agent_notes`

### 6. `POST /calls/postcall`

用途：
- 生成结构化总结
- 保存最终结果
- 关联录音和 transcript

## Post-call 输出

每通电话结束后都应生成：

- 是否接到真人
- 是否完成 claim intake
- 是否已 transfer 给用户
- claim reference number
- next step
- follow-up needed
- 失败原因
- 原始录音链接
- transcript 链接

## 最终建议

如果只选一个版本明天做：

- 用 `Twilio + Vapi`
- 采用 `STT -> text LLM -> TTS`
- 编排上选择：
  - 表面上是 `方案 A`
  - 实际上加一个最薄的状态层

也就是：

- 不要一上来做重状态机
- 但也不要把所有编排全交给 prompt

最实际的 MVP 形态应该是：

- `LLM-first`
- `guardrails + transfer policy`
- `full recording + full transcript retention`
- `post-call structured summary`

这样最符合 hackathon 节奏，也最容易在第二天往状态机版本演进。
