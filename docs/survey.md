# Survey: Error Handling in Existing Agent Harnesses

This document summarizes how five open-source agent harnesses define and
handle errors. It is informative input to
[draft-ouchi-agent-harness-errors](../draft-ouchi-agent-harness-errors.md).

## Method

Each repository was shallow-cloned and read only. No code was installed,
built, or run. The survey covered the agent loop, tool execution, model
provider adapters, context management, permissions and sandboxing,
sub-agents, and MCP handling. For each error condition we recorded its
trigger, how it is surfaced (fed back to the model, thrown to the caller,
or recorded in a result), its retry behavior, and the matching
draft code.

| Harness | Repository | Commit | Date |
|---|---|---|---|
| Hermes Agent | [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent) | `785c2e51` | 2026-11-04 |
| OpenClaw | [openclaw/openclaw](https://github.com/openclaw/openclaw) | `b4e1202d` | 2026-11-04 |
| Flue | [withastro/flue](https://github.com/withastro/flue) | `deee93b` | 2026-10-12 |
| pi (pi-agent-core, pi-ai) | [earendil-works/pi](https://github.com/earendil-works/pi) | `0a180b0` | 2026-11-04 |
| DeepSeek Harness | [deepseek-ai/deepseek-harness](https://github.com/deepseek-ai/deepseek-harness) | `fcc0ad46` | 2026-11-04 |

Limitations:
- Hermes Agent, OpenClaw, and DeepSeek Harness are large. Their core loops
  were read in depth; peripheral packages (channels, desktop clients,
  experimental backends) were only sampled.
- Flue delegates its inner loop to pi, which is surveyed separately.
- Line-level findings reflect the commits above and may change.

## Key Findings

### 1. Upstream errors are classified by matching message text

Hermes Agent, OpenClaw, Flue, and pi decide retryability, quota
exhaustion, and context overflow by applying regular expressions to
provider error messages. pi accumulates provider-specific patterns over
time, each tied to an issue number, and keeps negative lists to resolve
collisions (for example, a throttling message that matches the overflow
pattern). Flue notes that typed errors do not survive persistence, so it
embeds string markers in messages.

DeepSeek Harness is the exception. It states that code must route on the
error `code` and never by parsing `message`, and it discards error codes
from third-party SDKs as "not our taxonomy".

This is the strongest motivation for a stable, machine-readable error code.

### 2. Shared behaviors (candidate normative requirements)

| Behavior | Harnesses |
|---|---|
| Tool calls in output truncated by a token limit are not executed | Flue, Hermes, pi, DeepSeek |
| Every tool call receives a result; interrupted calls get a synthetic result | Flue, OpenClaw, pi, DeepSeek |
| Tool errors are returned to the model and the loop continues; upstream errors end the turn | all five |
| Tool calls with an unknown outcome are not re-executed automatically | Flue, Hermes, OpenClaw, DeepSeek |
| Loop repetition is handled by escalating from warning, to blocking the call, to halting the turn | Hermes, OpenClaw, DeepSeek (advisory only) |

### 3. "Retryable" is not a property of the error alone

- OpenClaw decides between `rotate_profile`, `fallback_model`,
  `surface_error`, and `continue_normal` based on the stage of the
  run, the configured fallbacks, and replay safety.
- Hermes attaches `should_compress`, `should_rotate_credential`, and
  `should_fallback` hints.
- pi honors a provider `x-should-retry` header that overrides the status
  code.
- pi and DeepSeek stop retrying when the server asks for a longer delay
  than local policy allows.

### 4. Loop limits live outside the core loop

pi, DeepSeek Harness, and OpenClaw have no turn limit in the core loop.
Flue delegates its limits to pi. In Hermes the limit is unbounded by
default. Instead, limits are enforced by outer layers: goal rounds,
retry budgets, loop detectors, and timeouts.

### 5. Errors are written for three audiences

The same failure produces different text for the model (recovery
instructions), the user (a hint for what to do next), and operators
(diagnostics with correlation IDs). Flue separates public `details`
from dev-only text. OpenClaw never shows raw provider text to users.

## Coverage of Draft Codes

### Codes needed beyond the initial draft

| Proposed code | Harnesses | Description |
|---|:-:|---|
| `tool.outcome_unknown` | 5/5 | Interrupted call; whether it executed is unknown |
| `upstream.timeout` | 5/5 | Stream stall, idle timeout, no first event |
| `harness.misconfigured` | 5/5 | Invalid configuration or composition |
| `upstream.quota_exhausted` | 4/5 | Billing or quota exhausted; unlike rate limiting, not transient |
| `harness.session_busy` | 4/5 | Concurrent operation on a session |
| `harness.environment_unavailable` | 4/5 | Sandbox or execution environment unavailable |
| `harness.storage_failed` | 4/5 | Persistence failure or corruption |
| `loop.depth_exceeded` | 3/5 | Sub-agent nesting limit |
| `tool.invalid_output` | 3/5 | Tool result violates the tool's own output schema |
| `tool.output_truncated` | 3/5 | Tool result truncated or spilled to a file |
| `upstream.model_unavailable` | 3/5 | Model not found, retired, or not entitled |
| `harness.superseded` | 3/5 | Another writer took over the session |
| `harness.interrupted` | 2/5 | Crash or restart, as distinct from cancellation |

Other candidates seen in one harness:
- `output.model_gave_up`
- `output.missing_required_action`
- `output.tool_call_as_text`
- `upstream.outcome_unknown` (the request was dispatched, then failed)
- `upstream.session_lost`
- `upstream.policy_blocked`
- `upstream.tls_failed`
- `tool.arguments_pruned`
- `context.compaction_deferred`
- `harness.delivery_failed`

`policy.approval_denied` should distinguish four cases: the user
rejected, the request timed out, no approval channel was available, and
the parameters changed after approval. Hermes, OpenClaw, and DeepSeek
Harness all make these distinctions.

### Members needed beyond the initial draft

| Proposed member | Rationale | Harnesses |
|---|---|---|
| `execution` (`not_started` / `started` / `unknown` / `completed`) | Retry safety depends on whether anything ran | all five; DeepSeek encodes it as `TOOL_NOT_STARTED` / `TOOL_OUTCOME_UNKNOWN` and `ABORTED_BEFORE_DISPATCH` / `ABORTED` |
| `severity` (`error` / `warning` / `info`) | Advisory errors that do not end the turn | Hermes, OpenClaw, DeepSeek |
| `repairs` | Record silent normalization: JSON repair, argument type coercion, tool-name fuzzy matching, dropped replay state | pi, Hermes, DeepSeek |
| `outcome` (incl. `pending`, `interrupted`) | Deferred completion, `pause_turn`, resumable handoff at a limit | pi, Hermes, OpenClaw, Flue |
| `retry_decision` | Who decided retryability; retry refused by local policy | pi, DeepSeek |
| `actions` | Recommended next steps: compact, rotate credential, fall back to another model, start a new session | Hermes, OpenClaw |
| `terminal` | "Do not retry or fail over", distinct from `retryable: false` | OpenClaw |
| `user_message` | User-facing text, separate from `model_feedback` and `detail` | all five |
| `issues[]` | Multiple validation errors as `{pointer, message}` | Flue, Hermes, pi, DeepSeek |
| `phase` | queue, preflight, provider, tool_execution, or post_turn | OpenClaw, Hermes |
| `subcode` | Implementation-specific reason | all five |
| correlation IDs | session, provider request ID, log reference | all five |
| `retry_at`; unbounded `retry_after` | Absolute reset time; an `Infinity` delay | Hermes, OpenClaw, DeepSeek |

## Security Observations

- OpenClaw records trusted error classes in WeakSets because "error names
  and prototypes are forgeable". It also guards against spoofed approval
  results embedded in tool output (CWE-841).
- DeepSeek Harness snapshots own properties of foreign errors to guard
  against hostile getters, and guarantees that some MCP errors contain
  no expanded configuration or upstream text.
- Hermes sanitizes tool error text by stripping role tags, CDATA, and
  fences, and caps its length before feeding it to the model.
- Several harnesses withhold raw validator messages from the model,
  because those messages can contain model input. OpenClaw sends
  argument lengths and hashes instead of content.

These observations support treating `model_feedback` as a
prompt-injection surface and support adding content-free diagnostics.
