# Cursor 多 Agent 头脑风暴系统

[English Version](README_en.md)

---

一套运行在 [Cursor IDE](https://cursor.sh) 中的**多 AI Agent 头脑风暴系统**。通过共享文件通信 + grep-wait 同步机制，让 4 个 AI Agent（1 Lead + 3 Participants）围绕任意话题进行结构化、多轮、有对抗性的深度讨论。

> **⚠️ 费用 & 模型警告**
>
> 本系统每次运行会并行启动 **4 个 Agent（subagent）**，每个 Agent 在整个讨论过程中会进行多轮 tool call（读写文件、grep-wait、搜索证据等）。
>
> **关于模型选择（旧计费模式用户必读）**：
> - 本系统设计时利用了 Claude Opus / GPT / Gemini 三个模型各自的认知特性。但在 Cursor 的**旧计费模式**下，只有开启 **Max 模式**才会使用 agent 文件中指定的模型（如 Claude Opus、GPT 5.2 等）；**非 Max 模式下所有 subagent 统一使用 Composer 模型**，无法体现不同模型的差异化风格。
> - 因此，如果你使用旧计费模式，**强烈建议在 Max 模式下运行**以获得最佳效果。
>
> **关于费用**：
> - **Max 模式（按量计费）**：一次完整的 brainstorm 大约消耗 **1 个父 Agent 请求 + 4 个子 Agent 请求**，每个子 Agent 内部有 10-30+ 次 tool call。根据话题复杂度和辩论轮数，单次会话的 token 消耗大约在 **50k-150k tokens**（总计，含所有 Agent）。请确保你的用量预算充足。
> - **Ultra 模式（无限请求）**：无需担心，尽情使用。
>
> 建议首次使用时先用一个简单话题测试，观察消耗后再决定是否调整辩论深度。

## 为什么做这个

现有的 AI 对话是 1v1 的——你问它答。但复杂问题需要**多视角碰撞**。

这套系统的核心思路是**利用"御三家"（Claude / GPT / Gemini）各自的认知特性**来形成互补：

- **Claude (Opus)**：擅长深层因果推理和系统性批判，能挖出其他模型忽略的结构性风险
- **GPT**：擅长实证分析和工程判断，倾向于用数据和先例说话，天然的"证伪者"
- **Gemini**：擅长跨界联想和创意重构，经常给出反直觉的框架——"你们在讨论 A，但真正的问题其实是 B"

三个模型的思维盲区几乎不重叠。让它们在同一个"聊天室"里实时辩论、互相攻击对方的论点，由第四个 Agent 主持和裁判，最终收敛出的分析质量远超任何单一模型。

## 仓库结构

```
multiagent-brainstorm/
├── README.md                          ← 你在这里（中文）
├── README_en.md                       ← English version
├── LICENSE
├── agents/                            ← 4 个 Agent 定义文件（复制到 .cursor/agents/）
│   ├── brainstorm-lead.md             ── 主持人 & 裁判
│   ├── brainstorm-opus.md             ── 深度分析者
│   ├── brainstorm-gpt.md              ── 实证评估者
│   └── brainstorm-gemini.md           ── 创意发散者
└── examples/                          ← 实际讨论记录
    └── openclaw-future/
        ├── chatroom.md                ── "如何看待 OpenClaw 的未来"（中文，~550 行）
        └── chatroom_en.md             ── English translation（~560 lines）
```

## 系统架构

```
User
  │
  ▼
Parent Agent ──── 触发词: "brainstorm" / "bs" / "讨论一下" / "头脑风暴"
  │
  │  并行启动 4 个 Task（subagent）
  │
  ├──▶ Lead (brainstorm-lead)      ── 主持人 & 裁判
  ├──▶ Opus (brainstorm-opus)      ── 深度分析者
  ├──▶ GPT  (brainstorm-gpt)       ── 实证评估者
  └──▶ Gemini (brainstorm-gemini)  ── 创意发散者
         │
         ▼
    ┌─────────────────┐
    │  chatroom.md    │  ◀── 共享文件，所有通信通过 Read/Append
    └─────────────────┘
```

### 通信机制

Agent 之间**不直接调用**。所有通信通过一个共享的 `chatroom.md` 文件：

- **写入**：先写临时文件 → `cat tmp >> chatroom.md`（原子追加）
- **等待**：`grep -c` 循环检测特定签名，每 2 秒扫描一次
- **签名**：每条消息末尾带 `---SIG:{Name}:{Tag}---`，用于精确同步

```
Agent 写完消息 ──▶ 追加到 chatroom.md（带签名）
                      │
Agent 进入 grep-wait ──▶ 每 2s grep 扫描文件
                      │
目标签名出现 ──▶ grep 退出循环 ──▶ Agent 读取全文并继续
```

### 讨论流程（5 个阶段）

```
DIVERGE ──▶ CHALLENGE ──▶ FREE DEBATE ──▶ CONVERGE ──▶ SYNTHESIS
  发散         挑战          自由辩论         收敛          综合
```

| 阶段 | Lead 做什么 | 参与者做什么 |
|------|-----------|-----------|
| **DIVERGE** | 提出议题 + 3-4 个子问题 | 各自独立给出观点 |
| **CHALLENGE** | 提炼冲突点，点名挑战 | 逐点 AGREE/DISAGREE |
| **FREE DEBATE** | 开启辩论 + 当裁判 | 互相 @对方 直接辩论，可搜索证据 |
| **CONVERGE** | 给出共识草案 | 补充风险/机会/建议 |
| **SYNTHESIS** | 综合全部讨论，写出最终结论 | — |

## 安装

### 前置条件

- [Cursor IDE](https://cursor.sh)（需要支持 Agent / Task 功能）
- 使用 Cursor 的 **Agent 模式**（非 Ask 模式）

### 安装步骤

**最简方式**：直接把本仓库链接 `https://github.com/hyleepp/multiagent-brainstorm` 发给 Cursor Agent，它会自动识别并完成安装。

**手动安装**：将 `agents/` 目录下的 4 个文件复制到你的项目或全局的 `.cursor/agents/` 目录：

```bash
# 方式一：放到当前项目（仅当前项目可用）
mkdir -p .cursor/agents
cp agents/brainstorm-*.md .cursor/agents/

# 方式二：放到全局目录（所有项目可用）
mkdir -p ~/.cursor/agents
cp agents/brainstorm-*.md ~/.cursor/agents/
```

放好后 Cursor 会自动识别这些 Agent，无需重启。

## 使用方法

在 Cursor 的 Agent 模式下，直接输入触发词 + 话题：

```
bs 讨论一下黄金的未来行情
```

```
头脑风暴 如何看待 OpenClaw 的未来？
```

系统会自动：
1. 创建 `.brainstorm/{session}/` 会话目录和空的 `chatroom.md`
2. 并行启动 4 个 Agent
3. Lead 分析你的话题，拟定议程和子问题
4. 三个参与者发散 → 挑战 → 辩论 → 收敛
5. Lead 综合所有观点，输出最终结论

讨论全程记录在 `chatroom.md` 中，你可以随时打开查看实时进展。

## 实际输出示例

`examples/` 目录包含完整的讨论记录，以下是摘要。

### OpenClaw 的未来（[查看全文](examples/openclaw-future/chatroom.md) | [English](examples/openclaw-future/chatroom_en.md)）

**DIVERGE 阶段**——三个 Agent 分别从不同角度切入：
- **GPT** 将 OpenClaw 定位为"末端执行器互操作标准"，强调 conformance 徽章和供应链可复制
- **Gemini** 将其比作"物理世界的 Android"，核心价值是统一数据采集格式
- **Opus** 聚焦安全危机（CVE、恶意技能比例），认为增长速度远超安全治理

**FREE DEBATE 阶段**——Agent 互相 @对方 直接攻击：
- Gemini @GPT：硬件接口标准会陷入 xkcd 927 困境
- GPT @Opus：翻译层会被大模型厂商 SDK 吞噬
- Opus @Gemini："先能力后治理"在机器人领域行不通——Web 漏洞泄露数据，机器人漏洞伤害人

**最终结论**——收敛出 12 个月 MVP 三件套：
1. 硬件级安全网关（物理沙箱）
2. Conformance 测试套件 + 兼容徽章
3. 遥操作/数据录制工具

注意观察三个模型的风格差异——这正是系统设计的核心：**不同模型的认知盲区不重叠，互相攻击后留下的结论更经得起推敲。**

## 核心设计决策

**为什么用文件通信而不是直接 Agent 调用？**
Cursor 的 subagent（Task tool）不支持 Agent 之间直接互调。共享文件是在平台限制下最可靠的通信方式。

**为什么用 grep-wait 而不是轮询？**
传统轮询（`sleep → Read → parse`）每次都要把整个 chatroom 发回 Agent，浪费大量 token。`grep -c` 在 shell 内部循环，接近零 token 消耗。

**为什么参与者是状态机而不是顺序脚本？**
LLM Agent 在长序列中容易"忘记"自己在第几步。状态机模式让 Agent 每次醒来都从头判断当前状态，更鲁棒。

**为什么 Lead 和参与者的签名格式不同？**
Lead 用 `SIG:Lead:PHASE:DIVERGE`（指令），参与者用 `SIG:Opus:DIVERGE`（回应）。分开后 grep 匹配更精确。

## 自定义

### 更换参与者角色

编辑 `agents/brainstorm-*.md` 中的 `Debate Style` 部分。当前角色分配基于各模型的认知特长：

| Agent | 模型 | 角色 | 为什么选它 |
|-------|------|------|-----------|
| **Opus** | Claude Opus | 深度分析者 | 擅长长链推理和系统性批判，适合挖因果链和结构性风险 |
| **GPT** | GPT 系列 | 实证评估者 | 擅长基于数据和先例做判断，天然适合当"证伪者" |
| **Gemini** | Gemini Pro | 创意发散者 | 擅长跨领域联想和框架重构，常能给出意料之外的切入角度 |

核心原则是**让不同模型的认知盲区互相覆盖**。

### 更换语言

所有 Agent 默认输出中文。修改每个文件的 `Rules` 部分中的 `Chinese for all responses` 即可。

### 调整辩论深度

- `Max 8 debate messages`：每个参与者在 FREE DEBATE 阶段最多发 8 条消息
- Lead 的 `Max 5 loops on Step 10`：裁判最多等 5 轮再强制收口
- 超时时间：Lead 等待 300s，参与者等待 120s

## 已知限制 & Troubleshooting

| 问题 | 原因 | 应对 |
|------|------|------|
| Agent 提前退出 | LLM 倾向于"完成任务后返回" | prompt 中有多层 "DO NOT RETURN" 规则 |
| 签名格式不一致 | LLM 模仿 Lead 格式 | 模板中有 ✅/❌ 对比示例 |
| grep-wait 被 backgrounded | `block_until_ms` 设置太低 | Lead 310000ms，参与者 130000ms |
| 辩论阶段卡住 | grep 未监听 REDIRECT | wait 命令已包含 `DEBATE:REDIRECT` |
| 某个 Agent 未响应 | 网络/超时 | Lead 超时兜底，2/3 响应即可推进 |

## 开发历程

经过约 20 轮迭代测试，解决了以下核心工程挑战：

1. **Agent 间通信**：从尝试直接调用 → 共享文件 chatroom 架构
2. **同步机制**：从 sleep 轮询 → grep-wait 阻塞等待
3. **签名协议**：从 HTML 注释 → 纯文本 `---SIG:..---` 分隔符
4. **Agent 持久性**：从顺序脚本 → 状态机循环 + 多层防早退规则
5. **辩论深度**：从单轮 Challenge → FREE DEBATE + @Receiver 定向消息 + Lead 裁判
6. **证据能力**：集成 WebSearch/WebFetch，辩论中可实时搜索数据

## License

MIT
