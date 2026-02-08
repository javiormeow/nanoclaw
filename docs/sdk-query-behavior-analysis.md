# Claude Agent SDK: `query()` Turn Behavior Analysis

Deep dive into when/why the Claude agent stops (ends turn immediately) vs continues (makes another API call), and how V1 vs V2 differ.

**SDK version analyzed:** `@anthropic-ai/claude-agent-sdk@0.2.34` (wrapping `@anthropic-ai/claude-code@2.1.34`)

---

## Architecture Overview

The SDK spawns `cli.js` as a child process with `--output-format stream-json --input-format stream-json --print --verbose` flags. Communication happens via JSON-lines on stdin/stdout. Inside the CLI process, the agentic loop is a **recursive async generator called `EZ()`**.

Both V1 (`query()`) and V2 (`createSession`/`send`/`stream`) use the exact same three-layer architecture:

```
SDK (sdk.mjs)           CLI Process (cli.js)
--------------          --------------------
XX Transport  ------>   stdin reader (bd1)
  (spawn cli.js)           |
$X Query      <------   stdout writer
  (JSON-lines)             |
                        EZ() recursive generator
                           |
                        Anthropic Messages API
```

---

## The Core Agentic Loop (`EZ`)

`EZ` is an async generator called with:

```
EZ({ messages, systemPrompt, canUseTool, maxTurns, turnCount=1, ... })
```

Each invocation = one API call to Claude (one "turn").

### Flow per turn:

1. **Prepare messages** -- trim context, run compaction if needed
2. **Call the Anthropic API** (via `mW1` streaming function)
3. **Extract tool_use blocks** from the response
4. **Branch:**
   - If **no tool_use blocks** -> stop (run stop hooks, return)
   - If **tool_use blocks present** -> execute tools, increment turnCount, recurse

---

## When the Agent STOPS (no more API calls)

### 1. No tool_use blocks in response (THE PRIMARY CASE)

```js
let r = k.flatMap(w1 => w1.message.content.filter(V1 => V1.type === "tool_use"));
if (!k.length || !r.length) {
    yield* _t4(P, k, ...);  // run stop hooks, then return
    return;
}
```

This is the **normal termination**: Claude responded with text only, meaning it decided it has completed the task. The API's `stop_reason` will be `"end_turn"`. The SDK does NOT make this decision -- it's entirely driven by Claude's model output.

**In practical terms, Claude ends the turn immediately when:**
- It has answered a question and doesn't need tools
- It has finished a multi-step task (last tool result was sufficient)
- It determines no further tool use is needed
- It encounters an error it can't recover from and reports it as text
- It's in `plan` mode and produces a plan (no tool execution)

### 2. Max turns exceeded

```js
let j1 = M + 1;
if (D && j1 > D) {
    yield Zq({type: "max_turns_reached", maxTurns: D, turnCount: j1});
    return;
}
```

Results in `SDKResultError` with `subtype: "error_max_turns"`.

### 3. Abort signal (user interruption)

```js
if (w.abortController.signal.aborted) {
    yield "[Request interrupted by user]";
    return;
}
```

### 4. Budget exceeded (SDK wrapper level)

```js
if (maxBudgetUsd !== void 0 && totalCost() >= maxBudgetUsd) {
    yield {type: "result", subtype: "error_max_budget_usd", ...};
    return;
}
```

### 5. Stop hook prevents continuation

When Claude has no tool_use blocks, stop hooks run via `_t4()`. If a hook returns `{preventContinuation: true}`, the loop terminates.

---

## When the Agent CONTINUES (makes another API call)

### 1. Response contains tool_use blocks (THE PRIMARY CASE)

```js
// Execute tools (concurrently where safe via PU1 or TP6)
// Collect tool results as user messages
// RECURSE:
yield* EZ({
    messages: [...original, ...assistantMsgs, ...toolResults],
    turnCount: j1,  // incremented
    ...
});
```

### 2. max_output_tokens recovery (up to 3 retries)

```js
if (lastMsg?.apiError === "max_output_tokens" && recoveryCount < 3) {
    yield* EZ({...params, maxOutputTokensRecoveryCount: recoveryCount + 1});
}
```

When Claude's response was cut off by output token limits, it retries with a "break your work into smaller pieces" context message.

### 3. Stop hook blocking errors

If a stop hook returns errors, they're fed back as context messages and the loop continues:

```js
if (blockingErrors.length > 0) {
    yield* EZ({messages: [...msgs, ...errors], ...});
}
```

---

## Complete Decision Table

| Condition | Action | Result Type |
|-----------|--------|-------------|
| Response has `tool_use` blocks | Execute tools, recurse into `EZ` | continues |
| Response has NO `tool_use` blocks | Run stop hooks, return | `success` |
| `turnCount > maxTurns` | Yield max_turns_reached | `error_max_turns` |
| `totalCost >= maxBudgetUsd` | Yield budget error | `error_max_budget_usd` |
| `abortController.signal.aborted` | Yield interrupted msg | depends on context |
| `stop_reason === "max_tokens"` (output) | Retry up to 3x with recovery prompt | continues |
| Stop hook `preventContinuation` | Return immediately | `success` |
| Stop hook blocking error | Feed error back, recurse | continues |
| Model fallback error | Retry with fallback model (one-time) | continues |

---

## V1 vs V2: API Differences

### V1: `query()` -- One-shot async generator

```ts
import { query } from "@anthropic-ai/claude-agent-sdk";

const q = query({ prompt: "...", options: {...} });
for await (const msg of q) { /* process events */ }
```

- Returns a `$X` (Query) directly as an async iterable
- When `prompt` is a string: `isSingleUserTurn = true` -> stdin auto-closes after first result
- For multi-turn: must pass an `AsyncIterable<SDKUserMessage>` and manage coordination yourself

### V2: `createSession()` + `send()` / `stream()` -- Persistent session

```ts
import { unstable_v2_createSession } from "@anthropic-ai/claude-agent-sdk";

await using session = unstable_v2_createSession({ model: "..." });
await session.send("first message");
for await (const msg of session.stream()) { /* events */ }
await session.send("follow-up");
for await (const msg of session.stream()) { /* events */ }
```

- Returns a `U9` wrapper around the same `$X` (Query)
- `isSingleUserTurn = false` always -> stdin stays open for multiple sends
- `send()` enqueues into an async queue (`QX`)
- `stream()` yields from the same message generator, stopping on `result` type
- Multi-turn is natural -- just alternate `send()` / `stream()`
- V2 does NOT call V1 `query()` internally -- both independently create Transport + Query

### Comparison Table

| Aspect | V1 | V2 |
|--------|----|----|
| `isSingleUserTurn` | `true` for string prompt | always `false` |
| Multi-turn | Requires managing `AsyncIterable` | Just call `send()`/`stream()` |
| stdin lifecycle | Auto-closes after first result | Stays open until `close()` |
| Agentic loop | Identical `EZ()` | Identical `EZ()` |
| Stop conditions | Same | Same |
| Session persistence | Must pass `resume` to new `query()` | Built-in via session object |
| API stability | Stable | Unstable preview (`unstable_v2_*` prefix) |

### Key finding: Zero difference in turn behavior

**There is no difference in when/why the agent stops between V1 and V2.** Both use the same CLI process, the same `EZ()` recursive generator, and the same decision logic. Claude decides to stop when it produces a response with no `tool_use` blocks. V2 simply makes it easier to send follow-up messages after Claude stops.

---

## Internals Reference

### Key minified identifiers (sdk.mjs)

| Minified | Purpose |
|----------|---------|
| `s_` | V1 `query()` export |
| `e_` | `unstable_v2_createSession` |
| `Xx` | `unstable_v2_resumeSession` |
| `Qx` | `unstable_v2_prompt` |
| `U9` | V2 Session class (`send`/`stream`/`close`) |
| `XX` | ProcessTransport (spawns cli.js) |
| `$X` | Query class (JSON-line routing, async iterable) |
| `QX` | AsyncQueue (input stream buffer) |

### Key minified identifiers (cli.js)

| Minified | Purpose |
|----------|---------|
| `EZ` | Core recursive agentic loop (async generator) |
| `_t4` | Stop hook handler (runs when no tool_use blocks) |
| `PU1` | Streaming tool executor (parallel during API response) |
| `TP6` | Standard tool executor (after API response) |
| `GU1` | Individual tool executor |
| `lTq` | SDK session runner (calls EZ directly) |
| `bd1` | stdin reader (JSON-lines from transport) |
| `mW1` | Anthropic API streaming caller |
