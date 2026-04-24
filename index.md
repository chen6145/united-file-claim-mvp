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

## Twilio / Vapi 电话 Setup 流程

这一节只写“明天要跑 demo，最短怎么配起来”。

### Step 1：准备 Twilio 账号

建议：

- 直接用 `Twilio paid account`
- 不要赌 `trial`

原因：

- trial 有被叫限制
- trial 会插入提示音
- trial 不适合真实外呼客服

### Step 2：准备 Twilio 电话号码

至少准备两类号码中的一种：

1. `Twilio number`  
   作为系统实际使用的电话入口

2. 你的个人手机号做 `verified caller ID`  
   如果你希望外呼时显示你自己的号码

MVP 里最稳的做法是：

- 系统底层用 `Twilio number`
- 如有需要，对外显示你的个人手机号

### Step 3：在 Vapi 创建基础 assistant

先不要一开始就追求复杂多 agent。

第一版建议：

- 一个主 assistant
- 一个统一 system prompt
- 一个 tool backend
- 一个 transfer policy

assistant 至少要配置：

- model
- voice
- transcriber
- tools
- server URL
- fallback / handoff 规则

### Step 4：把 Twilio 号码接进 Vapi

有两种常见方式：

1. 直接用 Vapi 分配/支持的号码  
2. 导入你自己的 `Twilio number`

对你这个项目，建议优先：

- 用你自己控制的 `Twilio number`
- 再导入 Vapi

这样后面录音、回拨、日志、迁移都更可控。

### Step 5：接好后端 server

你的后端至少要提供：

- tool endpoints
- webhook 接收
- post-call 处理
- transcript / recording 保存

第一版最少 endpoints：

- `POST /calls/start`
- `POST /tools/lookup_user_claim_context`
- `POST /tools/append_claim_notes`
- `POST /tools/transfer_to_user`
- `POST /tools/save_claim_reference`
- `POST /tools/save_next_step`
- `POST /calls/postcall`

### Step 6：先把 recording 和 transcript 存起来

顺序上，这一步应该比“优化 prompt”更早。

至少要确认：

- Twilio recording 能落盘
- Vapi transcript 事件能保存
- call_id 能串起：
  - Twilio call SID
  - Vapi call id
  - user id

如果这一步没做，demo 一旦失败就很难复盘。

### Step 7：配置 transfer 到你的手机

先把 transfer 做成一个明确工具，而不是临时想起来再配。

至少要准备：

- 你的目标手机号
- transfer 触发条件
- transfer 前给用户的 summary
- transfer 失败时的 fallback

MVP 的最低要求是：

- 只要出现本人验证或高风险确认，就可以稳定转到你手机

### Step 7B：如果你希望“你离开后再回到 AI”

这不是普通 `transferCall`，而是 `conference / bridge` 问题。

这里要先明确：

- `AI -> 你，transfer 成功`  
  默认通常意味着 AI 退出

- `AI + 客服 + 你` 三方短暂同在线，之后你离开、AI 继续  
  这需要 `Twilio conference orchestration`

也就是说：

- 纯 `Vapi transferCall` 适合“转给你然后你接手”
- 如果你想“你插一下话，然后还回 AI”，通常要 `Twilio + own server`

#### 这条高级路径要额外准备什么

- 一个 `conference id`
- 当前 call leg 的 `Twilio CallSid`
- 第二条拨给你手机的 call leg
- participant status callbacks
- 你加入 / 你离开后的路由规则

#### 最小流程

1. AI 正在和客服通话
2. 你的 server 把当前 call 更新到一个 conference
3. 你的 server 再拨一条 call 给你手机，并加入同一 conference
4. 你短暂介入
5. 你离开 conference
6. AI 和客服继续

#### 工程判断

如果只是明天 demo：

- 不建议第一版就做这条路径

如果你明确要验证“人类短暂介入后 AI 继续”的体验：

- 这条路可做
- 但它已经更接近 `方案 C` 的工作量，而不是简单的 `Twilio + Vapi setup`

### Step 8：先跑最小 happy path

第一轮测试不要直接打 United。

先测：

1. 外呼能打通
2. transcript 正常
3. tool call 正常
4. transfer 到你手机正常
5. recording 正常保存

只有这些都通了，再上真实客服电话。

### Step 9：再测真实客服电话路径

这里建议分三层验证：

1. `IVR layer`
   - 能不能稳定过菜单

2. `waiting layer`
   - 能不能正确识别真人接通

3. `human layer`
   - 能不能正确描述 claim 背景并拿到 next step

不要第一次测试就把所有能力同时打开。

### Step 10：最后才优化 prompt

很多人会一开始狂改 prompt，但对电话 agent 来说，优先级通常应该是：

1. telephony 通
2. transcript / recording 通
3. transfer 通
4. tool call 通
5. 最后才是 prompt 优化

### 推荐的 setup 顺序

如果你明天要跑 demo，我建议按这个顺序配：

1. `Twilio paid account`
2. 买 `Twilio number`
3. Vapi 建 assistant
4. 导入 / 绑定 Twilio 号码
5. 配最小 tool backend
6. 打通 recording + transcript
7. 打通 transfer 到你手机
8. 跑一通 mock 测试电话
9. 再打真实 United 客服

### 最容易踩的 setup 坑

- 一开始就用 trial
- 一开始就打真实客服电话
- 还没存 recording / transcript 就开始调 prompt
- transfer 号码没预先验证就临场测试
- 没先把 `Twilio SID / Vapi call id / user id` 做统一映射

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

## 方案 C：Twilio + 自己的 server

这个方案的核心思想是：

- 不用 `Vapi`
- 直接用 `Twilio` 做 telephony
- 自己在 server 里实现 `STT -> text LLM -> TTS`
- 自己负责 orchestration、logging、recording、transfer、DTMF

### 架构图

```text
Twilio
  |
Media Streams / TwiML / Webhooks
  |
Your Orchestrator Server
  |-- STT
  |-- text LLM
  |-- TTS
  |-- call state machine
  |-- transfer policy
  |-- DTMF policy
  |-- post-call evaluator
  |
Storage / logs / recordings
```

### 你要额外做什么

相比 `Twilio + Vapi`，这条路多出来的工作不是一点点，而是整层 runtime。

#### 1. 自己做实时音频桥

你要自己处理：

- Twilio Media Streams 接入
- 音频 chunk 收发
- STT 输入
- TTS 输出回推
- 中断 / 抢话 / turn-taking

#### 2. 自己做 agent runtime

Vapi 帮你做掉的很多东西，这里都要自己接：

- 对话轮次管理
- transcript 聚合
- partial / final transcript 处理
- tool 调用编排
- prompt 注入
- post-tool response 拼接

#### 3. 自己做 telephony orchestration

你要自己实现：

- `DTMF`
- `hold / wait`
- `transfer`
- `conference / bridge`
- `call update`
- `timeout / retry`
- `hangup / recover`

#### 4. 自己做录音和转写留存

这里不是“可选加分项”，而是基础设施工作：

- 保存 Twilio recording
- 保存 transcript 事件
- 保存 tool timeline
- 保存 transfer timeline
- 做最终结构化摘要

#### 5. 自己做 observability

至少要自己准备：

- 实时日志
- transcript viewer
- tool call logs
- 状态机日志
- error tracing
- 通话后复盘数据

### 如果你想用“自己 Mac 上的 agent”

这条路理论上可以，但会再多一层工程问题：

- 你的 Mac 上必须跑一个可调用的 API server
- 这个 server 必须被 Twilio / 你的 orchestrator 稳定访问
- 如果是本机服务，通常还要做公网暴露
- 睡眠、断网、进程挂掉都会直接影响电话

所以：

- demo 可以这么干
- 正式测试不建议把 Mac 当生产后端

### 方案 C 的优点

- 控制权最高
- 供应商锁定最低
- 最容易把系统往“真正产品”演进
- 可以精细控制 IVR、hold、transfer、tool UX
- 长期成本结构可能更干净

### 方案 C 的缺点

- 开发量最大
- 最容易死在 plumbing 上
- 最难在 hackathon 时间里做稳
- 录音、转写、日志、状态机都要自己兜
- 调试成本高

### 这条路到底有多难

如果只给一个非常直接的判断：

- `方案 A` 难度：`3/10`
- `方案 B` 难度：`6/10`
- `方案 C` 难度：`8.5/10`

这个分数不是指“理论理解难度”，而是指“明天要做出一个稳定 demo 的难度”。

### 为什么会难这么多

因为你不是只多做一个“LLM 接口”，你实际上多做了：

- voice runtime
- telephony runtime
- orchestration runtime
- observability runtime

也就是说，`Twilio + 自己 server` 不是“Vapi 的轻量替代”，而是“你自己成为 Vapi”。

### 这条路什么时候值得走

如果你满足下面任一条件，才值得认真走：

- 你非常在意控制权
- 你明确知道 Vapi 抽象会卡你
- 你想长期做产品，不只是 hackathon
- 你已经有现成的语音/编排后端积累

如果只是为了明天跑通 demo，这条路通常不值。

## 三种方案的 Tradeoff

### 方案 A 更像

- LLM-first agent
- 快速 demo
- 低工程投入

### 方案 B 更像

- Telephony-first workflow
- 稳定生产雏形
- 明确的流程控制

### 方案 C 更像

- 自建 voice stack
- 最大控制权
- 最大工程成本

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
- `方案 C` 只适合你明确要验证“自建 telephony agent stack”这件事本身

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

### 不建议的路线

如果你的目标就是：

- 明天要跑通
- 重点是展示 use case
- 不是展示自建 voice infra

那不建议直接走 `方案 C`。

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
- 更不要一上来自己重做整套 `Twilio + own server` runtime

最实际的 MVP 形态应该是：

- `LLM-first`
- `guardrails + transfer policy`
- `full recording + full transcript retention`
- `post-call structured summary`

这样最符合 hackathon 节奏，也最容易在第二天往状态机版本演进。
