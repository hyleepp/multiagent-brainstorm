# Cursor Multi-Agent Brainstorm System

[中文版](README.md)

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
├── README.md                          ← Chinese version
├── README_en.md                       ← You are here (English)
├── LICENSE
├── agents/                            ← 4 agent definition files (copy to .cursor/agents/)
│   ├── brainstorm-lead.md             ── Moderator & referee
│   ├── brainstorm-opus.md             ── Deep analyst
│   ├── brainstorm-gpt.md              ── Empirical evaluator
│   └── brainstorm-gemini.md           ── Creative synthesizer
└── examples/                          ← Real discussion transcripts
    └── openclaw-future/
        ├── chatroom.md                ── "Future of OpenClaw" (Chinese, ~550 lines)
        └── chatroom_en.md             ── English translation (~560 lines)
```

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

### Discussion Flow (5 Phases)

```
DIVERGE ──▶ CHALLENGE ──▶ FREE DEBATE ──▶ CONVERGE ──▶ SYNTHESIS
```

| Phase | Lead | Participants |
|-------|------|-------------|
| **DIVERGE** | Pose topic + 3-4 sub-questions | Give independent perspectives |
| **CHALLENGE** | Extract conflicts, challenge directly | State AGREE/DISAGREE per point |
| **FREE DEBATE** | Open debate + referee | @mention each other, search for evidence |
| **CONVERGE** | Draft consensus | Add risks/opportunities/recommendations |
| **SYNTHESIS** | Write final comprehensive analysis | — |

## Installation

### Prerequisites

- [Cursor IDE](https://cursor.sh) (with Agent / Task support)
- Use Cursor's **Agent mode** (not Ask mode)

### Steps

Copy the 4 files from `agents/` to your project or global `.cursor/agents/` directory:

```bash
# Option 1: Project-level (current project only)
mkdir -p .cursor/agents
cp agents/brainstorm-*.md .cursor/agents/

# Option 2: Global (available in all projects)
mkdir -p ~/.cursor/agents
cp agents/brainstorm-*.md ~/.cursor/agents/
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

Edit `Debate Style` in each `agents/brainstorm-*.md`. Current assignments are based on each model's cognitive strengths:

| Agent | Model | Role | Why |
|-------|-------|------|-----|
| **Opus** | Claude Opus | Deep analyst | Strong at long-chain reasoning and systematic critique |
| **GPT** | GPT series | Empirical evaluator | Data-driven judgment, natural "falsifier" |
| **Gemini** | Gemini Pro | Creative synthesizer | Cross-domain analogies, counter-intuitive reframing |

The core principle: **make sure different models' cognitive blind spots cover each other**.

### Change Language

All agents default to Chinese output. Change `Chinese for all responses` in each file's `Rules` section.

### Adjust Debate Depth

- `Max 8 debate messages`: Per participant in FREE DEBATE
- `Max 5 loops on Step 10`: Lead's referee patience limit
- Timeouts: Lead waits 300s, participants wait 120s

## Known Limitations & Troubleshooting

| Issue | Cause | Mitigation |
|-------|-------|-----------|
| Agent exits early | LLMs tend to "return" after completing a task | Multi-layer "DO NOT RETURN" rules in prompts |
| Inconsistent signatures | LLM mimics Lead's format | Templates include ✅/❌ examples |
| grep-wait backgrounded | `block_until_ms` too low | Lead: 310000ms, participants: 130000ms |
| Debate phase stuck | grep doesn't watch for REDIRECT | Wait commands include `DEBATE:REDIRECT` |
| An agent doesn't respond | Network/timeout | Lead has timeout fallback, proceeds with 2/3 |

## Development History

Built through ~20 iterations, solving these core engineering challenges:

1. **Inter-agent communication**: Direct invocation → shared-file chatroom
2. **Synchronization**: Sleep polling → grep-wait blocking
3. **Signature protocol**: HTML comments → plain-text `---SIG:..---` delimiters
4. **Agent persistence**: Sequential scripts → state machine loops + anti-early-exit rules
5. **Debate depth**: Single-round challenge → FREE DEBATE + @Receiver targeting + Lead as referee
6. **Evidence gathering**: Integrated WebSearch/WebFetch for real-time evidence during debates

## License

MIT
