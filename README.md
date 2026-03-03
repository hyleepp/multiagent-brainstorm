# Cursor Multi-Agent Brainstorm System

[中文版](README_zh.md)

---

A **multi-AI-agent brainstorming system** running inside [Cursor IDE](https://cursor.sh). Four AI agents (1 Lead + 3 Participants) conduct structured, multi-round, adversarial discussions on any topic via shared-file communication and grep-wait synchronization.

> **⚠️ Cost & Model Warning**
>
> Each brainstorm session launches **4 parallel agents (subagents)**, each performing 10-30+ tool calls throughout the discussion (file I/O, grep-wait, web search, etc.).
>
> **Model selection (important for legacy billing users)**:
> - This system is designed to leverage the distinct cognitive strengths of Claude Opus / GPT / Gemini. However, under Cursor's **legacy billing model**, only **Max Mode** uses the models specified in the agent files (e.g., Claude Opus, GPT 5.2). **Without Max Mode, all subagents run on the Composer model**, losing the differentiated cognitive styles across models.
> - Therefore, if you are on the legacy billing plan, **we strongly recommend running in Max Mode** for the best experience.
>
> **Cost**:
> - **Max Mode (usage-based billing)**: A full session consumes roughly **1 parent + 4 child agent requests**, totaling approximately **50k-150k tokens** depending on topic complexity and debate depth. Make sure your usage budget can accommodate this.
> - **Ultra Mode (unlimited requests)**: No concerns — use freely.
>
> We recommend running a simple test topic first to observe consumption before adjusting debate depth.

## Why This Exists

Current AI conversations are 1-on-1 — you ask, it answers. But complex problems demand **multi-perspective collision**.

The core idea is to **leverage the distinct cognitive strengths of the "Big Three" (Claude / GPT / Gemini)** for complementary analysis:

- **Claude (Opus)**: Excels at deep causal reasoning and systematic critique — uncovers structural risks others miss
- **GPT**: Excels at empirical analysis and engineering judgment — naturally the "falsifier" who demands data and precedent
- **Gemini**: Excels at cross-domain analogies and creative reframing — often delivers counter-intuitive perspectives: "You're debating A, but the real question is B"

Their cognitive blind spots barely overlap. By having them debate in a shared "chatroom" — attacking each other's arguments in real time, moderated by a fourth agent — the resulting analysis far exceeds what any single model produces.

## Repository Structure

```
multiagent-brainstorm/
├── README.md                          ← You are here (English)
├── README_zh.md                       ← 中文版
├── LICENSE
├── v1/agents/                         ← V1: Original version
│   ├── brainstorm-lead.md
│   ├── brainstorm-opus.md
│   ├── brainstorm-gpt.md
│   └── brainstorm-gemini.md
├── v2/agents/                         ← V2: Enhanced version (recommended)
│   ├── brainstorm-lead.md
│   ├── brainstorm-opus.md
│   ├── brainstorm-gpt.md
│   └── brainstorm-gemini.md
└── examples/                          ← Real discussion transcripts
    └── openclaw-future/
        ├── chatroom.md                ── "Future of OpenClaw" (Chinese, ~550 lines)
        └── chatroom_en.md             ── English translation (~560 lines)
```

## Versions

This project ships two versions of the agent definitions. Choose based on your needs:

### V1 — Original (`v1/agents/`)

The original brainstorm system with the core 5-phase flow. Simple, battle-tested, reliable.

- 5 phases: DIVERGE → CHALLENGE → FREE DEBATE → CONVERGE → SYNTHESIS
- Lead unilaterally controls debate closing (`DEBATE:CLOSE`)
- Max 8 debate messages per participant
- No minority report mechanism

**Best for**: Quick brainstorms, simple topics, minimal overhead.

### V2 — Enhanced (`v2/agents/`) ⭐ Recommended

Built on V1 with three high-leverage improvements inspired by [Robert's Rules of Order](https://en.wikipedia.org/wiki/Robert%27s_Rules_of_Order) (1876). Core philosophy: *"The great purpose of all rules and forms, is to subserve the will of the assembly, rather than to restrain it."*

**New in V2:**

1. **Procedural Rights + Lead RULING**: Participants can challenge the discussion direction or suggest missed topics in natural language at any phase. Lead MUST scan for these and issue a formal RULING (accept/reject + reasoning). This creates a "bottom-up feedback channel" — participants are no longer locked into Lead's agenda.

2. **Silent Consent Window**: Lead no longer unilaterally closes debate. Instead: broadcast `INTENT_CLOSE` → wait for objections (must include new evidence) → only then `CLOSE`. Prevents premature truncation of valuable discussions.

3. **DISSENT (Minority Report)**: In CONVERGE, participants can attach an optional `DISSENT:` block with evidence/reasoning. Lead MUST preserve unrefuted dissent in the final synthesis. Prevents false consensus — AI disagreements often represent valuable alternative framings.

**Also in V2:**
- Max 50 debate messages per participant (let them debate freely)
- Duplicate submission prevention (fixes a state machine bug)
- Chair neutrality rule (Lead stays procedural during DIVERGE→DEBATE, saves analysis for SYNTHESIS)

**Best for**: Complex topics, deep analysis, high-stakes decisions where you want thorough adversarial examination.

## Architecture

```
User
  │
  ▼
Parent Agent ──── Triggers: "brainstorm" / "bs" / "debate"
  │
  │  Launches 4 Tasks in parallel
  │
  ├──▶ Lead (brainstorm-lead)      ── Moderator & referee
  ├──▶ Opus (brainstorm-opus)      ── Deep analyst
  ├──▶ GPT  (brainstorm-gpt)       ── Empirical evaluator
  └──▶ Gemini (brainstorm-gemini)  ── Creative synthesizer
         │
         ▼
    ┌─────────────────┐
    │  chatroom.md    │  ◀── Shared file, all communication via Read/Append
    └─────────────────┘
```

### Communication

Agents **never call each other directly**. All communication flows through a shared `chatroom.md` file:

- **Write**: Write to temp file → `cat tmp >> chatroom.md` (atomic append)
- **Wait**: `grep -c` loop checking for specific signatures every 2 seconds
- **Signatures**: Each message ends with `---SIG:{Name}:{Tag}---` for precise synchronization

```
Agent finishes message ──▶ Append to chatroom.md (with signature)
                               │
Agent enters grep-wait ──▶ grep scans file every 2s
                               │
Target signature found ──▶ grep exits loop ──▶ Agent reads full file and continues
```

### Discussion Flow

**V1 — 5 Phases:**
```
DIVERGE ──▶ CHALLENGE ──▶ FREE DEBATE ──▶ CONVERGE ──▶ SYNTHESIS
```

**V2 — 5 Phases + Procedural Mechanisms:**
```
DIVERGE ──▶ CHALLENGE ──▶ FREE DEBATE ──▶ CONVERGE ──▶ SYNTHESIS
                                │
                    V2: INTENT_CLOSE → Silent Consent Window → CLOSE
```

| Phase | Lead | Participants |
|-------|------|-------------|
| **DIVERGE** | Pose topic + 3-4 sub-questions | Give independent perspectives; may suggest missed topics (V2) |
| **CHALLENGE** | Extract conflicts, challenge directly | State AGREE/DISAGREE per point; may challenge direction (V2) |
| **FREE DEBATE** | Open debate + referee | @mention each other, search for evidence |
| **Silent Consent** | Broadcast INTENT_CLOSE, wait for objections (V2) | OBJECTION with new evidence, or DONE |
| **CONVERGE** | Draft consensus | Confirm + optional DISSENT block with evidence (V2) |
| **SYNTHESIS** | Write final analysis (preserving unrefuted DISSENT in V2) | — |

## Installation

### Prerequisites

- [Cursor IDE](https://cursor.sh) (with Agent / Task support)
- Use Cursor's **Agent mode** (not Ask mode)

### Steps

**Easiest way**: Just paste this repo link `https://github.com/hyleepp/multiagent-brainstorm` to Cursor Agent — it will automatically recognize and install the agents (defaults to V2).

**Manual install**: Choose your version and copy the 4 agent files:

```bash
# V2 (recommended) — with procedural rights, silent consent, DISSENT
mkdir -p .cursor/agents
cp v2/agents/brainstorm-*.md .cursor/agents/

# V1 (original) — simpler, fewer mechanisms
mkdir -p .cursor/agents
cp v1/agents/brainstorm-*.md .cursor/agents/

# Global install (available in all projects) — add ~/.cursor/agents/ instead
mkdir -p ~/.cursor/agents
cp v2/agents/brainstorm-*.md ~/.cursor/agents/   # or v1/agents/
```

Cursor auto-detects the agents — no restart needed.

## Usage

In Cursor's Agent mode, type a trigger word + topic:

```
bs discuss the future of gold prices
```

```
brainstorm What's the future of OpenClaw?
```

The system will automatically:
1. Create `.brainstorm/{session}/` directory with empty `chatroom.md`
2. Launch 4 agents in parallel
3. Lead analyzes the topic, frames the agenda
4. Participants diverge → challenge → debate → converge
5. Lead synthesizes all perspectives into a final conclusion

The full discussion is recorded in `chatroom.md` — open it anytime to watch the live progress.

## Example Output

See `examples/` for full transcripts. Summary below.

### Future of OpenClaw ([English](examples/openclaw-future/chatroom_en.md) | [中文原文](examples/openclaw-future/chatroom.md))

**DIVERGE** — Three agents approach from different angles:
- **GPT** frames OpenClaw as an "end-effector interop standard" — conformance badges, supply chain
- **Gemini** calls it "the Android of the physical world" — data format standardization
- **Opus** focuses on security crisis — CVEs, malicious skill ratio, growth outpacing governance

**FREE DEBATE** — Agents directly @attack each other:
- Gemini @GPT: Hardware interface standards will fall into the xkcd 927 trap
- GPT @Opus: The translation layer will be absorbed by LLM provider SDKs
- Opus @Gemini: "Move fast, govern later" doesn't work in robotics — web bugs leak data, robot bugs cause physical harm

**Final Conclusion** — Converged on a 12-month MVP:
1. Hardware-level safety gateway (physical sandbox)
2. Conformance test suite + compatibility badge
3. Teleoperation / data recording tools

Notice the distinct style of each model — this is the core design principle: **different models have non-overlapping blind spots; conclusions that survive their cross-examination are more robust.**

## Key Design Decisions

**Why file-based communication instead of direct agent calls?**
Cursor's subagents (Task tool) don't support inter-agent invocation. Shared files are the most reliable communication method within platform constraints.

**Why grep-wait instead of polling?**
Traditional polling (`sleep → Read → parse`) sends the entire chatroom back to the agent each cycle, wasting tokens. `grep -c` loops inside the shell with near-zero token cost.

**Why state machines instead of sequential scripts?**
LLM agents tend to "forget" their position in long sequences. State machines let agents re-evaluate the current state from scratch each time they wake up — more robust than remembering "I'm on step N."

**Why different signature formats for Lead vs participants?**
Lead uses `SIG:Lead:PHASE:DIVERGE` (command), participants use `SIG:Opus:DIVERGE` (response). Separation makes grep matching more precise.

## Customization

### Swap Participant Roles

Edit `Debate Style` in each `v1/agents/brainstorm-*.md` or `v2/agents/brainstorm-*.md`. Current assignments are based on each model's cognitive strengths:

| Agent | Model | Role | Why |
|-------|-------|------|-----|
| **Opus** | Claude Opus | Deep analyst | Strong at long-chain reasoning and systematic critique |
| **GPT** | GPT series | Empirical evaluator | Data-driven judgment, natural "falsifier" |
| **Gemini** | Gemini Pro | Creative synthesizer | Cross-domain analogies, counter-intuitive reframing |

The core principle: **make sure different models' cognitive blind spots cover each other**.

### Change Language

Agents default to responding in the same language as the user's topic. To force a specific language, change `Respond in the same language as the user's topic` in each file's `Rules` section.

### Adjust Debate Depth

**V1:**
- `Max 8 debate messages`: Per participant in FREE DEBATE
- `Max 5 loops on Step 10`: Lead's referee patience limit

**V2:**
- `Max 50 debate messages`: Per participant in FREE DEBATE
- `Max 50 loops on Step 10`: Lead's referee patience limit
- Silent consent window: 60s (adjustable in Lead prompt)

**Both versions:**
- Timeouts: Lead waits 300s, participants wait 120s

## Known Limitations & Troubleshooting

| Issue | Cause | Mitigation |
|-------|-------|-----------|
| Agent exits early | LLMs tend to "return" after completing a task | Multi-layer "DO NOT RETURN" rules in prompts |
| Inconsistent signatures | LLM mimics Lead's format | Templates include ✅/❌ examples |
| grep-wait backgrounded | `block_until_ms` too low | Lead: 310000ms, participants: 130000ms |
| Debate phase stuck | grep doesn't watch for REDIRECT | Wait commands include `DEBATE:REDIRECT` (V2 also includes `INTENT_CLOSE`) |
| Duplicate phase responses | Agent re-submits CHALLENGE during DEBATE | V2 adds DUPLICATE PREVENTION rule in state machine |
| Agent jumps to CONVERGE on INTENT_CLOSE | Agent treats INTENT_CLOSE as PHASE:CONVERGE | V2 adds explicit warning in Action Table |
| An agent doesn't respond | Network/timeout | Lead has timeout fallback, proceeds with 2/3 |

## Development History

Built through ~27 iterations, solving these core engineering challenges:

1. **Inter-agent communication**: Direct invocation → shared-file chatroom
2. **Synchronization**: Sleep polling → grep-wait blocking
3. **Signature protocol**: HTML comments → plain-text `---SIG:..---` delimiters
4. **Agent persistence**: Sequential scripts → state machine loops + anti-early-exit rules
5. **Debate depth**: Single-round challenge → FREE DEBATE + @Receiver targeting + Lead as referee
6. **Evidence gathering**: Integrated WebSearch/WebFetch for real-time evidence during debates
7. **Procedural rights (V2)**: Inspired by Robert's Rules of Order — RULING mechanism, Silent Consent Window, DISSENT minority reports, chair neutrality

## License

MIT
