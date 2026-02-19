---
title: "Agent Design Review"
summary: "Comprehensive review of OpenClaw's agent architecture, execution patterns, and decision-making loop"
read_when:
  - You want to understand how OpenClaw's agent loop works
  - You're comparing OpenClaw with other agent frameworks
  - You need to understand the decision-making and action selection patterns
---

# Agent Design Review 🧠

This document provides a comprehensive review of how OpenClaw designs and implements its agent full loop, including the patterns it uses, how it differs from traditional decision-making frameworks like OODA, and how it provides suggested next actions.

## Executive Summary

OpenClaw implements a **context-driven agentic loop** pattern that emphasizes:

- **Single serialized execution per session** with queued concurrency control
- **Rich context assembly** from workspace files, tools, skills, and conversation history
- **Tool-mediated action execution** with streaming feedback
- **Automatic memory management** with pre-compaction flushes
- **Model-driven decision making** without explicit observe-orient-decide-act phases

The architecture prioritizes **flexibility, extensibility, and human-in-the-loop control** over rigid autonomy, making it fundamentally different from traditional autonomous agent frameworks.

## Architecture Overview

### High-Level Components

```
┌─────────────────────────────────────────────────────────────┐
│                     Message Intake                          │
│  (Telegram, Discord, WhatsApp, Signal, CLI, Web UI, etc.)  │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│                  Session Management                         │
│  • Session key resolution                                   │
│  • Agent ID mapping                                         │
│  • Queue mode (collect/steer/followup)                      │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│                    Agent Command                            │
│  • Resolve model + thinking/verbose defaults                │
│  • Load skills snapshot                                     │
│  • Call runEmbeddedPiAgent                                  │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│               runEmbeddedPiAgent Core                       │
│  • Serialize runs via session + global queues               │
│  • Resolve model + auth profile                             │
│  • Build pi-agent-core session                              │
│  • Subscribe to events & stream deltas                      │
│  • Enforce timeout & abort handling                         │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│            Context Assembly & Prompt Build                  │
│  • System prompt (tools, skills, safety, runtime)           │
│  • Bootstrap files (AGENTS.md, SOUL.md, TOOLS.md, etc.)    │
│  • Conversation history                                     │
│  • Tool schemas (JSON)                                      │
│  • Model-specific limits & compaction reserve              │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│              Model Inference (LLM Provider)                 │
│  • Anthropic, OpenAI, Google, Ollama, etc.                 │
│  • Streaming responses                                      │
│  • Tool call generation                                     │
│  • Reasoning/thinking blocks (when supported)               │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│                  Tool Execution Loop                        │
│  • Parse tool calls from model                              │
│  • Execute tools (read, edit, exec, browser, etc.)         │
│  • Stream tool output                                       │
│  • Handle errors & retries                                  │
│  • Check for queued steering messages                       │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│            Response Assembly & Delivery                     │
│  • Assistant text (with optional reasoning)                 │
│  • Inline tool summaries (when verbose)                     │
│  • Error text (when model errors)                           │
│  • Suppress NO_REPLY tokens                                 │
│  • Deduplicate messaging tool sends                         │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│              Session Persistence & Hooks                    │
│  • Write session transcript (JSONL)                         │
│  • Trigger agent_end hooks                                  │
│  • Update session metadata                                  │
│  • Emit lifecycle events                                    │
└─────────────────────────────────────────────────────────────┘
```

## The OpenClaw Agent Loop Pattern

### Core Execution Model

OpenClaw uses a **single-threaded, serialized execution loop** per session with the following characteristics:

1. **Intake Phase**
   - Message arrives via messaging channel (Telegram, Discord, etc.) or CLI
   - Session key is resolved or created
   - Message is queued according to queue mode (collect/steer/followup)

2. **Context Assembly Phase**
   - **System prompt** is built from:
     - Tool list + descriptions
     - Skills list (metadata with file paths)
     - Safety guardrails
     - Workspace location
     - Current date/time and timezone
     - Runtime metadata (host, OS, model, thinking level)
   - **Bootstrap files** are injected from workspace:
     - `AGENTS.md` — operating instructions + memory
     - `SOUL.md` — persona, boundaries, tone
     - `TOOLS.md` — tool usage conventions
     - `IDENTITY.md` — agent name/vibe/emoji
     - `USER.md` — user profile
     - `MEMORY.md` — curated long-term memory (optional)
   - **Conversation history** is loaded from session transcript
   - **Tool schemas** (JSON) are included for model understanding

3. **Model Inference Phase**
   - Complete context is sent to the LLM provider
   - Model streams responses in real-time
   - Assistant text and tool calls are emitted as events
   - Reasoning/thinking blocks are captured (when supported)

4. **Tool Execution Phase**
   - Tool calls are parsed and validated
   - Tools execute in sequence (not parallel for safety)
   - Tool output is streamed back to the model
   - **Steering check**: after each tool call, queue is checked for new user messages
   - If steering message present, remaining tools are skipped and user message is injected

5. **Response Assembly Phase**
   - Assistant text is assembled
   - Tool summaries are included (when verbose mode enabled)
   - `NO_REPLY` tokens are filtered out
   - Duplicate messaging tool sends are deduplicated

6. **Persistence Phase**
   - Session transcript is written to disk (JSONL format)
   - Session metadata is updated
   - Lifecycle hooks are triggered (`agent_end`)

### Key Design Principles

1. **Serialization Over Parallelism**
   - One run at a time per session (prevents tool/session races)
   - Optional global lane for cross-session serialization
   - Tools execute sequentially, not in parallel

2. **Context-Driven Decision Making**
   - Model has full visibility into:
     - All available tools and their schemas
     - Workspace bootstrap files (instructions, persona, memory)
     - Complete conversation history
     - Current runtime state
   - Decisions are made by the model based on rich context, not predetermined logic

3. **Tool-Mediated Actions**
   - All actions (read, write, exec, browser, etc.) go through tools
   - Tool execution is logged and traceable
   - Tool policy can restrict/approve actions
   - Sandboxing can isolate dangerous operations

4. **Streaming-First**
   - Assistant text streams in real-time
   - Tool execution output streams incrementally
   - Lifecycle events (start/end/error) stream as they occur
   - Enables responsive UX and real-time feedback

5. **Memory Externalization**
   - Memory lives in workspace Markdown files (`MEMORY.md`, `memory/YYYY-MM-DD.md`)
   - Memory is not "in RAM" — must be written to disk to persist
   - Automatic pre-compaction memory flush reminds model to save important context

## Comparison with OODA Loop

The **OODA loop** (Observe-Orient-Decide-Act) is a decision-making framework developed by military strategist John Boyd. It consists of four phases:

1. **Observe** — Gather information about the environment
2. **Orient** — Analyze and synthesize the observations in context
3. **Decide** — Select a course of action based on the analysis
4. **Act** — Execute the decision and observe the results

### How OpenClaw Differs from OODA

| Aspect | OODA Loop | OpenClaw Agent Loop |
|--------|-----------|---------------------|
| **Phase Structure** | Explicit 4-phase cycle | Implicit, context-driven inference |
| **Observation** | Discrete observation phase | Continuous context assembly + streaming feedback |
| **Orientation** | Separate analysis step | Implicit in model inference with full context |
| **Decision** | Explicit decision point | Model-driven tool call generation |
| **Action** | Discrete execution phase | Tool execution loop with streaming output |
| **Iteration** | Loop back to Observe | Automatic continuation until task completion |
| **Control Flow** | Rigid phase boundaries | Flexible, model-driven progression |
| **Feedback** | After full loop completion | Real-time streaming during execution |
| **Concurrency** | Can parallelize observations | Serialized execution per session |
| **Memory** | Implicit state management | Explicit file-based persistence |

### Key Differences Explained

#### 1. No Explicit Phase Boundaries

OpenClaw does not have explicit "observe", "orient", "decide", and "act" phases. Instead:

- **Observation** happens continuously through:
  - Initial context assembly (bootstrap files, conversation history)
  - Tool execution results (streamed back to model in real-time)
  - Queued user messages (can interrupt/steer ongoing execution)

- **Orientation** is implicit in the model's inference process:
  - The model receives the full context (system prompt + history + tool schemas)
  - Internal transformer attention mechanisms "orient" based on all available information
  - No separate orientation step — it's baked into the model's forward pass

- **Decision** is expressed as tool calls:
  - Model generates tool calls (or decides to reply without tools)
  - Tool selection is probabilistic, not deterministic
  - Multiple tools can be called in sequence (model-driven)

- **Action** is tool execution:
  - Tools execute and return results
  - Results are appended to conversation history
  - Model automatically sees results in next inference (if more tools are needed)

#### 2. Context-Driven vs. State-Machine Driven

OODA is often implemented as a **state machine** with explicit state transitions between phases. OpenClaw is **context-driven**:

- The model doesn't "know" it's in an "observe" or "decide" phase
- All information is presented as context
- The model's training (instruction-following, tool use) drives behavior
- No hardcoded phase logic — flexibility over rigidity

#### 3. Real-Time Feedback vs. Batch Processing

OODA typically completes a full loop before feedback:
- Observe → Orient → Decide → Act → (observe results)

OpenClaw provides **streaming feedback**:
- Tool execution output streams to the model in real-time
- Assistant text streams to the user immediately
- User can interrupt/steer with new messages (queue mode: `steer`)
- More responsive, less batch-oriented

#### 4. Memory Management

OODA frameworks often have implicit memory (state in the system). OpenClaw makes memory **explicit**:

- Memory must be written to disk (`MEMORY.md`, `memory/YYYY-MM-DD.md`)
- Automatic pre-compaction flush prompts memory saving
- Memory search tools enable semantic retrieval
- No invisible "agent state" — everything is in files or session transcript

#### 5. Human-in-the-Loop Control

OpenClaw emphasizes human control:

- Tool policy can require human approval for dangerous operations
- Sandboxing can isolate risky actions
- Channel allowlists gate access
- Queue steering allows real-time interruption
- OODA frameworks are often more autonomous

## How OpenClaw Provides Suggested Next Actions

OpenClaw does **not** have a built-in "suggested next actions" system in the traditional sense. Instead, it relies on the **model's agency** and **context-driven prompting** to guide behavior.

### Mechanisms for Action Guidance

#### 1. System Prompt Instructions

The system prompt (see [`docs/concepts/system-prompt.md`](/concepts/system-prompt)) includes:

- **Tool descriptions** — Each tool has a short description explaining its purpose
- **Safety guardrails** — Advisory text guiding model behavior (e.g., avoid power-seeking, respect oversight)
- **Skills list** — Metadata about available skills (with file paths for on-demand loading)
- **Documentation pointers** — Directs model to consult local OpenClaw docs when needed

These instructions implicitly guide the model toward appropriate next actions.

#### 2. Bootstrap Files (Workspace Context)

Bootstrap files injected into every run provide explicit guidance:

- **`AGENTS.md`** — Operating instructions, conventions, memory
  - User can write explicit "next steps" or "workflow" instructions here
  - Example: "When asked to analyze code, first run tests, then review results"

- **`TOOLS.md`** — Tool usage conventions
  - Example: "Always use `grep` before editing files to understand context"
  - Example: "Use `bash` with `mode="sync"` for long-running commands"

- **`SOUL.md`** — Persona and boundaries
  - Example: "Be proactive: suggest next steps when a task completes"

These files act as a **persistent instruction layer** that shapes behavior across all sessions.

#### 3. Conversation History

The model sees the entire conversation history, including:

- User's original request
- Previous assistant responses
- Tool execution results
- Errors and retries

This history provides **implicit context** for what has been tried and what remains to be done.

#### 4. Tool Execution Results

Tool results often contain **natural next steps**:

- Running tests might reveal failures → fix the failing tests
- Reading a file might show a TODO comment → address the TODO
- `grep` might find multiple matches → decide which to edit

The model uses tool results to **dynamically plan next actions**.

#### 5. Model Training (Instruction-Following)

Modern LLMs (like Claude, GPT-4, etc.) are trained to:

- Break down complex tasks into steps
- Suggest next actions when a step completes
- Ask clarifying questions when stuck
- Chain tool calls to accomplish goals

This is **baked into the model's weights**, not hardcoded in OpenClaw.

### Example: How Next Actions Emerge

**User request:**
```
"Fix the failing tests in the codebase"
```

**Agent execution flow:**

1. **Context assembly**
   - System prompt: includes `bash`, `read`, `edit` tool descriptions
   - `AGENTS.md`: might say "Always run tests before and after fixing code"
   - Conversation history: empty (first message)

2. **Model inference**
   - Model reasons: "I should first run the tests to see what's failing"
   - Generates tool call: `bash("npm test")`

3. **Tool execution**
   - `bash` runs tests
   - Output: `3 tests failed: test-user.ts, test-auth.ts, test-db.ts`

4. **Model inference (next turn)**
   - Model sees test output in context
   - Reasons: "I should read the first failing test to understand the issue"
   - Generates tool call: `read("/path/to/test-user.ts")`

5. **Tool execution**
   - `read` returns file contents

6. **Model inference (next turn)**
   - Model sees file contents
   - Reasons: "The test expects `userId` but the code uses `user_id`"
   - Generates tool call: `edit("/path/to/user.ts", old_str="user_id", new_str="userId")`

7. **Tool execution**
   - `edit` modifies file

8. **Model inference (next turn)**
   - Model reasons: "I should re-run tests to verify the fix"
   - Generates tool call: `bash("npm test")`

9. **Repeat until all tests pass**

**Suggested next actions emerge from:**
- Model's instruction-following training
- Context of previous tool results
- Guidance in system prompt and bootstrap files

### Comparison with Explicit Next Action Systems

Some agent frameworks (e.g., LangChain, AutoGPT) have **explicit planning steps** where the agent generates a plan upfront:

```
Plan:
1. Run tests
2. Read failing test files
3. Fix code
4. Re-run tests
5. Commit changes
```

OpenClaw does **not** require explicit planning. Instead:

- Plans emerge **dynamically** from model inference
- Each tool call is a "micro-decision"
- The model can adapt mid-execution if new information arrives
- More flexible, less brittle

However, users can **encourage planning** by adding instructions to `AGENTS.md`:

```markdown
When given a complex task:
1. Think through the steps required
2. List the steps explicitly
3. Execute each step in order
4. Report progress after each step
```

This leverages the model's instruction-following to simulate explicit planning.

## Hook Points for Customization

OpenClaw provides multiple **hook systems** for intercepting and modifying agent behavior:

### Plugin Hooks (Agent Lifecycle)

See [`docs/tools/plugin.md`](/tools/plugin#plugin-hooks) for full API details.

- **`before_model_resolve`** — Intercept provider/model selection before session load
- **`before_prompt_build`** — Inject `prependContext`/`systemPrompt` after session load
- **`before_agent_start`** — Legacy compatibility hook (prefer explicit hooks above)
- **`agent_end`** — Inspect final message list and run metadata after completion
- **`before_compaction` / `after_compaction`** — Observe/annotate compaction cycles
- **`before_tool_call` / `after_tool_call`** — Intercept tool params/results
- **`tool_result_persist`** — Transform tool results before persisting to transcript

### Internal Hooks (Gateway Lifecycle)

See [`docs/automation/hooks.md`](/automation/hooks) for setup and examples.

- **`agent:bootstrap`** — Runs while building bootstrap files before system prompt is finalized
- **Command hooks** — `/new`, `/reset`, `/stop`, and other slash commands

### Use Cases for Hooks

- **Custom planning** — Inject planning instructions via `before_prompt_build`
- **Tool restrictions** — Block/modify tool calls via `before_tool_call`
- **Logging/analytics** — Track tool usage, model performance via `after_tool_call`, `agent_end`
- **Memory management** — Auto-save summaries to `MEMORY.md` via `agent_end`
- **Multi-agent coordination** — Trigger sub-agents via `agent_end`

## Performance & Scalability Characteristics

### Concurrency Model

- **Per-session serialization** — One run at a time per session key
- **Optional global lane** — Cross-session serialization (when enabled)
- **Queue modes**:
  - `collect` — Queue messages until current run ends
  - `steer` — Inject queued messages mid-execution (after each tool call)
  - `followup` — Start new run immediately after current run ends

### Context Management

- **Automatic compaction** — When session approaches context window limit
- **Pre-compaction memory flush** — Silent turn to save memory before compaction
- **Bootstrap file truncation** — Per-file and total size limits to prevent context bloat
- **Tool result sanitization** — Large tool outputs are truncated/summarized

### Timeout Handling

- **Agent runtime timeout** — Default 600s (configurable via `agents.defaults.timeoutSeconds`)
- **`agent.wait` timeout** — Default 30s (wait-only, does not stop agent)
- **Abort signals** — Can cancel ongoing runs

### Model Failover

- **Automatic retry** with fallback models (see [`docs/concepts/model-failover.md`](/concepts/model-failover))
- **Auth profile cooldown** — Prevent hammering failing API keys
- **Error classification** — Distinguishes auth errors, billing errors, rate limits, context overflows

## Strengths of the OpenClaw Design

1. **Flexibility** — No rigid phase boundaries; model adapts to context
2. **Extensibility** — Rich plugin/hook system for customization
3. **Transparency** — All actions logged to session transcript (JSONL)
4. **Human-in-the-loop** — Tool policy, sandboxing, queue steering enable control
5. **Streaming-first** — Real-time feedback for responsive UX
6. **Memory externalization** — No hidden state; everything in files or transcript
7. **Multi-channel support** — Works across Telegram, Discord, WhatsApp, Signal, CLI, Web UI
8. **Model-agnostic** — Supports Anthropic, OpenAI, Google, Ollama, etc.

## Limitations & Trade-offs

1. **No explicit planning** — Plans emerge dynamically, which can be less predictable
2. **Serialized execution** — One run at a time per session (no parallel tool execution)
3. **Context window constraints** — Large workspaces/histories require compaction
4. **Model-dependent behavior** — Performance varies by model quality
5. **No built-in task prioritization** — User must guide multi-step tasks
6. **Memory management burden** — Model must remember to write important context to files

## Future Directions

Potential enhancements (not yet implemented):

- **Explicit planning mode** — Optional "think → plan → execute" workflow
- **Parallel tool execution** — For independent tools (requires safety analysis)
- **Task queue management** — Built-in multi-task scheduling/prioritization
- **Proactive suggestions** — Automatic "what's next?" prompts after task completion
- **Collaborative multi-agent** — Coordinated execution across multiple agent instances

## Related Documentation

- [Agent Loop](/concepts/agent-loop) — Detailed walkthrough of lifecycle events
- [Agent Runtime](/concepts/agent) — Workspace, bootstrap files, sessions
- [System Prompt](/concepts/system-prompt) — What the model sees
- [Context](/concepts/context) — How context is assembled
- [Memory](/concepts/memory) — Memory files and automatic flush
- [Streaming](/concepts/streaming) — Real-time output behavior
- [Model Failover](/concepts/model-failover) — Automatic retry logic
- [Compaction](/concepts/compaction) — Context window management
- [Plugins](/tools/plugin) — Extension and hook API

## Conclusion

OpenClaw's agent design emphasizes **context-driven flexibility** over rigid autonomy. By providing the model with rich context (tools, skills, workspace files, conversation history), it enables **emergent decision-making** without hardcoded phase logic.

The lack of explicit OODA-style phases is a **design choice**, not a limitation. It allows the model to adapt dynamically to changing context and user interruptions, while still maintaining traceability and human control through tool policy, sandboxing, and queue steering.

Suggested next actions emerge **naturally** from model inference, guided by system prompt instructions, bootstrap files, and tool execution results. This approach leverages the model's instruction-following capabilities rather than implementing a separate planning system.

For users who want more explicit planning, the system is flexible: add planning instructions to `AGENTS.md`, use plugin hooks to inject planning prompts, or rely on the model's native reasoning capabilities (e.g., Claude's extended thinking).

---

_For questions or feedback on this design review, see the [OpenClaw Discord](https://discord.gg/openclaw) or open an issue on [GitHub](https://github.com/openclaw/openclaw)._
