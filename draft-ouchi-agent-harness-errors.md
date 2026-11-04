---
title: "Problem Details for AI Agent Harness Errors"
abbrev: "Agent Harness Errors"
category: std
docname: draft-ouchi-agent-harness-errors-latest
submissiontype: IETF
number:
date:
v: 3
# area: "Applications and Real-Time"
keyword:
 - AI agent
 - LLM
 - harness
 - problem details
 - error handling

author:
 -
    fullname: Arata Ouchi
    email: ouchiarata@philtz.com

normative:
  RFC2119:
  RFC8174:
  RFC8259:
  RFC8126:
  RFC9457:

informative:
  RFC9110:
  JSON-RPC:
    title: "JSON-RPC 2.0 Specification"
    target: https://www.jsonrpc.org/specification
    date: 2013
    author:
      - org: JSON-RPC Working Group
  MCP:
    title: "Model Context Protocol Specification"
    target: https://modelcontextprotocol.io/specification
    author:
      - org: Model Context Protocol
  A2A:
    title: "Agent2Agent (A2A) Protocol Specification"
    target: https://a2a-protocol.org/
    author:
      - org: A2A Project
  OTEL-GENAI:
    title: "Semantic Conventions for Generative AI Systems"
    target: https://opentelemetry.io/docs/specs/semconv/gen-ai/
    author:
      - org: OpenTelemetry

--- abstract

AI agents run language models inside a control program, the "harness",
that parses model output, invokes tools, manages context, and enforces
limits. Failures in this layer, such as malformed structured output,
exceeded turn limits, or context window overflow, are currently reported
in incompatible, implementation-specific ways. This document defines a
problem type and extension members for Problem Details (RFC 9457) that
describe agent harness errors in a machine-readable way, including
whether an error is transient and whether it can be repaired by the
model itself. It also establishes an IANA registry of harness error
codes.

--- middle

# Introduction

An AI agent is commonly built as a loop: a harness sends a prompt to a
language model, interprets the model's output, executes any requested
tool calls, appends the results to the context, and repeats until a
stopping condition is reached.

Errors can arise at several points in this loop. Some originate in
upstream model APIs (rate limiting, overload), some in tools (timeouts,
permission denial), and some in the harness itself (output that cannot
be parsed, schema violations, exhausted turn or token budgets). Existing
specifications cover parts of this space:

* Protocol-level errors in agent protocols such as {{MCP}} and {{A2A}}
  use {{JSON-RPC}} error codes, which describe failures of the protocol
  exchange, not of the agent's reasoning loop.
* Model APIs report termination reasons (for example, output truncated
  at a token limit, or a refusal), but these are not error codes and
  differ between providers.
* Agent frameworks define their own exception types with incompatible
  names and granularity.
* Observability conventions such as {{OTEL-GENAI}} provide an
  `error.type` attribute but do not define its values.

As a result, a client that orchestrates agents from several vendors, or
a supervising agent that delegates to sub-agents, cannot reliably decide
how to react to a failure: retry unchanged, feed the error back to the
model so it can correct itself, adjust a limit, escalate to a human, or
give up.

This document defines:

1. A Problem Details {{RFC9457}} problem type for agent harness errors
   ({{problem-type}}).
2. Extension members that carry a stable error code, the error's
   origin, and recovery hints ({{members}}).
3. An initial set of error codes ({{codes}}) and an IANA registry for
   them ({{iana}}).
4. Guidance for carrying these objects outside HTTP, including in
   JSON-RPC error data and telemetry ({{transport}}).

## Scope

This document covers errors detected by the harness during an agent
run. It does not define model behavior, tool semantics, agent-to-agent
protocols, or how a harness should implement retries. It does not
replace protocol-level error codes; it complements them.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

Agent:
: A system that uses one or more language models to decide on and
  perform actions, usually over multiple steps.

Harness:
: The non-model software that drives an agent: it constructs model
  inputs, interprets model outputs, executes tools, manages context, and
  enforces limits.

Turn:
: One model invocation together with the processing of its output.

Run:
: A sequence of turns performed to accomplish one task.

Structured output:
: Model output that is required to conform to a machine-readable format,
  such as JSON conforming to a schema, or a tool call with typed
  arguments.

Model feedback:
: Text that the harness supplies to the model in a subsequent turn to
  describe an error, so that the model can correct its behavior.

# Problem Type {#problem-type}

This document registers the following problem type:

Type URI:
: `https://iana.org/assignments/http-problem-types#agent-harness-error`

Title:
: Agent Harness Error

Recommended HTTP status code:
: Varies; see {{http-status}}.

A problem details object of this type MUST include the `code` member
defined in {{members}}. The `title` member SHOULD be the title
registered for that code. The `detail` member SHOULD describe the
specific occurrence in a form suitable for human operators.

Consumers MUST use the `code` member, not `title` or `detail`, to
determine the kind of error.

Until the IANA registration in {{iana}} is complete, implementations
MAY use the type URI `https://agenterrors.org/problems/harness-error`.
Consumers MUST treat the two type URIs as equivalent.

> RFC Editor: please remove the preceding paragraph before publication.

# Extension Members {#members}

The following extension members are defined for the
`agent-harness-error` problem type. Consumers MUST ignore members they
do not recognize, as required by {{RFC9457}}.

## code

A string containing a registered harness error code ({{codes}}).
REQUIRED.

Codes have the form `<category>.<name>`, where both parts consist of
lowercase ASCII letters, digits, and underscores. The category groups
related codes ({{categories}}). Implementations MAY define private codes
using a category of the form `x-<organization>`; such codes MUST NOT be
registered.

A consumer that receives an unknown code within a known category SHOULD
treat it as a generic error of that category.

## origin

A string identifying the component in which the error originated.
OPTIONAL. Values are:

`model`:
: The language model produced output that the harness could not accept.

`harness`:
: The harness itself, including enforcement of its own limits.

`tool`:
: A tool invoked by the harness.

`upstream`:
: A remote service used by the harness, typically the model API.

`user`:
: A human participant, for example by denying an approval.

## retryable

A boolean. OPTIONAL. If `true`, repeating the same operation without
modification has a reasonable chance of success (for example, after a
transient upstream failure). If `false`, repeating the operation
unchanged is not expected to succeed.

If absent, consumers SHOULD use the default recorded in the registry for
the code.

## model_recoverable

A boolean. OPTIONAL. If `true`, the harness or a consumer can reasonably
expect the model to correct the error if it is informed of it in a
subsequent turn, for example by re-generating structured output that
conforms to the schema.

`retryable` and `model_recoverable` are independent. An error can be
neither (for example, a turn limit), either, or both.

If absent, consumers SHOULD use the default recorded in the registry for
the code.

## model_feedback

A string. OPTIONAL. Text intended to be supplied to the model so that
it can correct the error, such as a description of the schema violation.
This member is distinct from `detail`, which is intended for humans. See
{{security}} for considerations on its content.

## retry_after

A non-negative number of seconds after which a retry is advised.
OPTIONAL. When the problem details object is carried in an HTTP response
that also includes a `Retry-After` header field {{RFC9110}}, the two
SHOULD agree.

## attempts

A non-negative integer giving the number of attempts that the harness
already made for the failed operation, including the one that produced
this error. OPTIONAL. This lets a consumer distinguish a first failure
from a failure that persisted through internal retries.

## max_attempts

A non-negative integer giving the number of attempts the harness was
configured to make. OPTIONAL.

## limit

A JSON object describing a limit that was exceeded. OPTIONAL. It
contains:

`name`:
: A string naming the limit, for example `turns`, `input_tokens`,
  `output_tokens`, `context_tokens`, `cost`, or `wall_clock_seconds`.

`value`:
: A number giving the configured limit.

`observed`:
: A number giving the observed value, if known.

`unit`:
: A string giving the unit, if not implied by `name`. For `cost`, an
  ISO 4217 currency code.

## location

A JSON object locating the error within the run. OPTIONAL. It MAY
contain `run_id` (string), `turn` (integer, starting at 1), `tool_name`
(string), `tool_call_id` (string), and `pointer` (a JSON Pointer string
identifying the offending location within structured output).

## upstream

A JSON object containing raw information reported by an upstream
component. OPTIONAL. It MAY contain `provider` (string), `status`
(integer, an HTTP status code), `code` (string), and `stop_reason`
(string). This member preserves vendor-specific detail; consumers
SHOULD NOT depend on its contents for control flow.

## causes

An array of problem details objects that caused this error, ordered from
most to least recent. OPTIONAL. For example, an
`output.retries_exhausted` error can list the individual
`output.schema_violation` errors that preceded it.

# Error Codes {#codes}

## Categories {#categories}

| Category | Description |
|----------|-------------|
| output   | Model output could not be accepted by the harness. |
| loop     | A limit on the agent loop was reached, or the loop is not progressing. |
| context  | Context assembly or management failed. |
| tool     | Tool resolution or execution failed. |
| policy   | A policy, guardrail, refusal, or human decision stopped the operation. |
| upstream | A remote dependency, typically the model API, failed. |
| harness  | The harness failed for another reason. |
{: title="Error categories"}

## Initial Codes

The columns R and M give the registry defaults for `retryable` and
`model_recoverable` respectively.

| Code | Title | Origin | R | M |
|------|-------|--------|---|---|
| output.malformed | Malformed Output | model | false | true |
| output.schema_violation | Output Schema Violation | model | false | true |
| output.truncated | Output Truncated | model | false | false |
| output.empty | Empty Output | model | true | true |
| output.unknown_tool | Unknown Tool Requested | model | false | true |
| output.invalid_tool_arguments | Invalid Tool Arguments | model | false | true |
| output.retries_exhausted | Output Retries Exhausted | harness | false | false |
| loop.max_turns_exceeded | Maximum Turns Exceeded | harness | false | false |
| loop.budget_exceeded | Budget Exceeded | harness | false | false |
| loop.timeout | Run Timeout | harness | false | false |
| loop.repetition_detected | Repetition Detected | harness | false | true |
| context.window_exceeded | Context Window Exceeded | harness | false | false |
| context.compaction_failed | Context Compaction Failed | harness | true | false |
| tool.not_available | Tool Not Available | tool | false | true |
| tool.execution_failed | Tool Execution Failed | tool | false | true |
| tool.timeout | Tool Timeout | tool | true | true |
| tool.permission_denied | Tool Permission Denied | harness | false | true |
| tool.sandbox_violation | Sandbox Violation | harness | false | false |
| policy.guardrail_triggered | Guardrail Triggered | harness | false | false |
| policy.model_refusal | Model Refusal | model | false | false |
| policy.content_filtered | Content Filtered | upstream | false | false |
| policy.approval_denied | Approval Denied | user | false | true |
| upstream.rate_limited | Upstream Rate Limited | upstream | true | false |
| upstream.overloaded | Upstream Overloaded | upstream | true | false |
| upstream.unavailable | Upstream Unavailable | upstream | true | false |
| upstream.authentication_failed | Upstream Authentication Failed | upstream | false | false |
| upstream.invalid_request | Upstream Invalid Request | upstream | false | false |
| upstream.invalid_response | Upstream Invalid Response | upstream | true | false |
| upstream.agent_failed | Remote Agent Failed | upstream | false | true |
| harness.cancelled | Run Cancelled | harness | false | false |
| harness.internal_error | Harness Internal Error | harness | false | false |
{: title="Initial harness error codes"}

## Code Semantics

output.malformed:
: The model output could not be parsed in the required syntax, for
  example invalid JSON.

output.schema_violation:
: The output was syntactically valid but did not conform to the required
  schema. `location.pointer` SHOULD identify the offending location.

output.truncated:
: The output ended before completion because an output token limit was
  reached. Retrying unchanged is not expected to succeed; raising the
  limit or reducing the requested output may.

output.empty:
: The model produced no usable content.

output.unknown_tool:
: The model requested a tool that is not offered to it.

output.invalid_tool_arguments:
: The model requested an offered tool with arguments that are missing,
  malformed, or do not conform to the tool's input schema.

output.retries_exhausted:
: The harness re-prompted the model after one or more `output.*` errors
  and the configured number of attempts was exhausted. `causes` SHOULD
  list the underlying errors.

loop.max_turns_exceeded:
: The run reached its configured maximum number of turns.

loop.budget_exceeded:
: A token, cost, or other resource budget for the run was exhausted.
  `limit` SHOULD be present.

loop.timeout:
: The run exceeded its wall-clock time limit.

loop.repetition_detected:
: The harness determined that the run is not progressing, for example
  because the same tool call with the same arguments was repeated.

context.window_exceeded:
: The assembled input exceeds the model's context window and could not
  be reduced.

context.compaction_failed:
: An attempt to reduce the context, for example by summarization, failed.

tool.not_available:
: A requested tool exists but cannot currently be used, for example
  because its server is disconnected.

tool.execution_failed:
: A tool was invoked and reported failure.

tool.timeout:
: A tool did not complete within its time limit.

tool.permission_denied:
: The harness's permission configuration did not allow the tool call.

tool.sandbox_violation:
: A tool attempted an operation prohibited by its sandbox.

policy.guardrail_triggered:
: An input or output guardrail configured in the harness stopped the
  operation.

policy.model_refusal:
: The model declined to perform the request.

policy.content_filtered:
: A content filter applied by an upstream provider blocked the input or
  output.

policy.approval_denied:
: A human denied an approval that the harness requested.

upstream.rate_limited:
: The upstream service rejected the request due to rate limiting.
  `retry_after` SHOULD be present if known.

upstream.overloaded:
: The upstream service reported that it is temporarily overloaded.

upstream.unavailable:
: The upstream service could not be reached or returned a server error.

upstream.authentication_failed:
: The upstream service rejected the harness's credentials.

upstream.invalid_request:
: The upstream service rejected the request as invalid, for reasons not
  covered by a more specific code.

upstream.invalid_response:
: The upstream service returned a response that does not conform to its
  protocol.

upstream.agent_failed:
: A remote agent to which the harness delegated work reported that the
  delegated task failed. `causes` SHOULD contain the remote agent's
  harness error object, if available. See {{a2a}}.

harness.cancelled:
: The run was cancelled by its caller.

harness.internal_error:
: The harness failed for a reason not covered by any other code.

## Choosing Between Codes

When more than one code applies, the harness SHOULD report the most
specific code and MAY list others in `causes`. When an error from a
lower layer is the direct cause, the harness SHOULD report the code that
best describes what the consumer needs to do. For example, output that
is invalid JSON because it was truncated SHOULD be reported as
`output.truncated`, not `output.malformed`.

# Carrying Harness Errors {#transport}

## HTTP {#http-status}

When a harness error is returned in an HTTP response, the response
SHOULD use the `application/problem+json` media type {{RFC9457}}. The
following status codes are RECOMMENDED:

| Category or code | Status |
|------------------|--------|
| upstream.rate_limited | 429 |
| upstream.overloaded, upstream.unavailable | 503 |
| upstream.* (other), upstream.agent_failed | 502 |
| loop.timeout, tool.timeout | 504 |
| policy.*, tool.permission_denied | 403 |
| harness.internal_error | 500 |
| others | 422 |
{: title="Recommended HTTP status codes"}

## Outside HTTP

A harness error object is a JSON object {{RFC8259}} and MAY be used
outside HTTP, for example as the result of a library call, in a log
record, or in a streamed event. In such cases the `status` member SHOULD
be omitted.

## JSON-RPC {#jsonrpc}

When a harness error is reported in a {{JSON-RPC}} error response, the
harness error object SHOULD be placed in the `data` member of the
JSON-RPC error object. The JSON-RPC `code` and `message` members
continue to be governed by the protocol in use; this document does not
allocate JSON-RPC error codes. A receiver that understands this document
SHOULD prefer the `code` in `data` over the JSON-RPC `code` for deciding
how to react.

A receiver can recognize a harness error object in `data` by the value
of its `type` member.

{{mcp}} and {{a2a}} give specific rules for the Model Context Protocol
and the Agent2Agent protocol.

## Telemetry

When harness errors are recorded using {{OTEL-GENAI}} or similar
conventions, the `code` value SHOULD be used as the value of the
`error.type` attribute.

# Model Context Protocol Binding {#mcp}

This section defines how harness errors are carried in {{MCP}}, in both
directions: an MCP server reporting an error to an MCP client (the
harness), and a harness translating MCP errors it receives into harness
error codes. MCP distinguishes protocol errors, which are JSON-RPC error
responses, from tool execution errors, which are successful
`tools/call` results with `isError` set to `true`. This binding
preserves that distinction.

## Metadata Key {#mcp-key}

Harness error objects are carried in MCP `_meta` objects under the key
`org.agenterrors/harness-error`. The value is a harness error object.
The key uses a reverse-DNS prefix under a domain controlled by the
editors of this document, as recommended by MCP for `_meta` keys.

## Server to Client

An MCP server that reports a tool failure SHOULD do so as a tool
execution error, so that the model can see it and correct itself:

* `isError` is `true`;
* `content` contains at least one text item. If the harness error object
  has a `model_feedback` member, the first text item SHOULD contain that
  text; otherwise it SHOULD contain the `detail` text;
* `_meta` contains the harness error object under the key in
  {{mcp-key}}.

The following codes SHOULD be reported as tool execution errors:
`tool.execution_failed`, `tool.timeout`, `tool.permission_denied`,
`tool.not_available`, `tool.sandbox_violation`,
`output.invalid_tool_arguments`, and any `upstream.*` error encountered
by the tool.

Errors that prevent the request from being processed as a tool call at
all SHOULD be reported as JSON-RPC errors with the harness error object
in `data`, using the JSON-RPC codes required by MCP:

| Harness error code | JSON-RPC code |
|--------------------|---------------|
| output.unknown_tool | -32602 (Invalid params) |
| harness.internal_error | -32603 (Internal error) |
| other errors not reported as tool execution errors | -32603 |
{: title="MCP JSON-RPC codes for harness errors"}

Because the `content` text is placed in the model's context, a server
MUST NOT put information in it that it would not also put in
`model_feedback`.

## Client Side: Translating MCP Errors

A harness acting as an MCP client SHOULD convert MCP errors into harness
error objects as follows. If a harness error object is present (in
`_meta` or in JSON-RPC `data`), its `code` SHOULD be used as is, and the
harness SHOULD set `origin` to `tool` if absent. Otherwise:

| MCP signal | Harness error code |
|------------|--------------------|
| `tools/call` result with `isError: true` | tool.execution_failed |
| JSON-RPC -32602 for a tool name the model chose that the server does not offer | output.unknown_tool |
| JSON-RPC -32602 for other reasons | output.invalid_tool_arguments |
| JSON-RPC -32601 (Method not found) | tool.not_available |
| JSON-RPC -32700, -32600 | harness.internal_error |
| JSON-RPC -32603 or other server error | tool.execution_failed |
| Request timeout or cancellation by the client due to timeout | tool.timeout |
| Transport closed or server not connected | tool.not_available |
| HTTP 401 or 403 from an HTTP-based transport | upstream.authentication_failed |
{: title="Translating MCP errors"}

In each case the harness SHOULD preserve the original JSON-RPC code and
message in the `upstream` member, with `provider` set to `mcp`.

## Sampling

When an MCP server requests sampling from a client and the client's
harness fails, the client SHOULD return a JSON-RPC error with the
harness error object in `data`. For example, if the user declines the
sampling request, the code is `policy.approval_denied`.

# Agent2Agent Protocol Binding {#a2a}

This section defines how harness errors are carried in {{A2A}}. In A2A,
failures of a task are reported through the task's status, while
failures of the protocol exchange are reported as protocol errors. This
binding preserves that distinction.

## Extension Declaration

An A2A agent that emits harness error objects SHOULD declare this in its
Agent Card as an extension, with the extension URI
`https://agenterrors.org/ext/a2a/v1` and `required` set to `false`.
That URI resolves to a description of this binding. Clients can
activate the extension using the A2A extension activation mechanism.
Agents MUST still report errors in the base A2A form, so that clients
that do not support this extension continue to work.

## Task Failures

When a task ends because of a harness error, the remote agent SHOULD:

* set the task state as given in {{a2a-states}};
* include in the status message a text part containing the `detail`
  text; and
* include in the status message a data part whose data is the harness
  error object.

| Harness error code | A2A task state |
|--------------------|----------------|
| harness.cancelled | canceled |
| policy.model_refusal, policy.guardrail_triggered, policy.content_filtered | rejected |
| all other codes | failed |
{: #a2a-states title="A2A task states for harness errors"}

Some conditions are not errors in A2A and SHOULD NOT be reported as
harness errors while the task can continue. For example, when a harness
needs approval from a human, it uses the `input-required` state; only if
the approval is denied and the task ends does `policy.approval_denied`
apply. Likewise, needing credentials uses `auth-required`.

## Protocol Errors

A2A protocol errors SHOULD carry a harness error object in the JSON-RPC
`data` member (or the corresponding error details in other A2A
bindings) when the error was caused by a harness condition, such as
`upstream.rate_limited` or `harness.internal_error`. The A2A error code
is chosen according to A2A.

## Client Side: Translating A2A Errors

A harness that delegates to a remote agent over A2A treats the remote
agent as an upstream component. If a harness error object is present,
the harness SHOULD report `upstream.agent_failed` and place the received
object in `causes`. The `retryable` member of the outer object SHOULD be
copied from the received object. The `model_recoverable` member SHOULD
be `true` only if the harness's own model could reasonably react, for
example by rephrasing the delegated request. Otherwise:

| A2A signal | Harness error code |
|------------|--------------------|
| Task state `failed` | upstream.agent_failed |
| Task state `rejected` | policy.model_refusal |
| Task state `canceled` not requested by this harness | upstream.agent_failed |
| TaskNotFoundError (-32001) | upstream.invalid_request |
| TaskNotCancelableError (-32002) | upstream.invalid_request |
| PushNotificationNotSupportedError (-32003), UnsupportedOperationError (-32004), ContentTypeNotSupportedError (-32005) | upstream.invalid_request |
| InvalidAgentResponseError (-32006) | upstream.invalid_response |
| JSON-RPC -32700 to -32603 | upstream.invalid_request or upstream.unavailable |
| HTTP 429 | upstream.rate_limited |
| HTTP 401 or 403 | upstream.authentication_failed |
| Network failure or HTTP 5xx | upstream.unavailable |
{: title="Translating A2A errors"}

The harness SHOULD preserve the original A2A error code, message, and
task state in the `upstream` member, with `provider` set to `a2a`.

## Nested Agents

When an error passes through several agents, each harness wraps the
received object in `causes` instead of rewriting it, so that the
outermost consumer can see the original `code` and `origin`. A harness
MAY truncate `causes` to limit size, and SHOULD keep the innermost
entry. {{security}} applies to `model_feedback` received from remote
agents.

# Example

A harness asked a model for JSON conforming to a schema. The model
returned a field of the wrong type three times, and the harness gave up.

~~~ http-message
HTTP/1.1 422 Unprocessable Content
Content-Type: application/problem+json

{
  "type": "https://iana.org/assignments/http-problem-types#agent-harness-error",
  "title": "Output Retries Exhausted",
  "detail": "Structured output failed validation after 3 attempts.",
  "code": "output.retries_exhausted",
  "origin": "harness",
  "retryable": false,
  "model_recoverable": false,
  "attempts": 3,
  "max_attempts": 3,
  "location": { "run_id": "run_8f2c", "turn": 5 },
  "causes": [
    {
      "type": "https://iana.org/assignments/http-problem-types#agent-harness-error",
      "title": "Output Schema Violation",
      "code": "output.schema_violation",
      "origin": "model",
      "model_recoverable": true,
      "model_feedback": "Field 'price' must be a number, got string.",
      "location": { "turn": 5, "pointer": "/items/0/price" }
    }
  ]
}
~~~

# Security Considerations {#security}

Harness error objects may contain fragments of prompts, model output,
tool arguments, or tool results. These can include sensitive data or
secrets. Harnesses SHOULD NOT include such content in `detail`,
`model_feedback`, `upstream`, or `causes` when the object is sent to a
party that is not authorized to see it, and SHOULD redact credentials in
all cases.

The `model_feedback` member is designed to be inserted into a model's
context. If it contains text derived from untrusted sources, such as
tool results or remote agents, it can carry prompt-injection attacks. A
harness that receives a harness error object from another party (for
example, a sub-agent) MUST NOT insert that party's `model_feedback` into
its own model's context with greater trust than the party's other
output.

The `retryable` and `model_recoverable` members are hints. A consumer
that retries automatically MUST enforce its own limits on attempts,
time, and cost, since a faulty or malicious producer could otherwise
induce unbounded retries and resource consumption.

Distinguishing `policy.*` errors in detail can help an attacker probe
guardrails. Harnesses MAY report a less specific code, such as
`policy.guardrail_triggered`, to untrusted consumers.

# Privacy Considerations

Run identifiers, tool names, and error details can reveal information
about a user's activity. The considerations in {{security}} regarding
disclosure apply. Logs containing harness error objects should be
subject to the same retention and access policies as the conversation
content they describe.

# IANA Considerations {#iana}

## HTTP Problem Type

IANA is requested to register the following entry in the "HTTP Problem
Types" registry established by {{RFC9457}}:

Type URI:
: https://iana.org/assignments/http-problem-types#agent-harness-error

Title:
: Agent Harness Error

Recommended HTTP status code:
: 422

Reference:
: This document

## Agent Harness Error Codes Registry

IANA is requested to create a new registry titled "Agent Harness Error
Codes". Registrations are made under the Specification Required policy
{{RFC8126}}.

Each entry contains:

Code:
: The error code, in the syntax defined in {{members}}.

Title:
: A short human-readable title.

Default retryable:
: `true` or `false`.

Default model_recoverable:
: `true` or `false`.

Description:
: A brief description of the error.

Reference:
: The defining specification.

Designated experts should verify that a proposed code is not already
covered by an existing code, that its category is appropriate, and that
its defaults are justified. New categories require Standards Action.
Codes in categories beginning with `x-` are reserved for private use and
are not registered.

The initial contents of the registry are the codes listed in {{codes}},
each with a Reference of this document.

--- back

# Mapping from Existing Implementations
{:numbered="false"}

This appendix is informative. The following shows how common
provider-specific signals could map to harness error codes. Names are
given as examples and may change in the cited implementations.

| Signal | Harness error code |
|--------|-------------------|
| Model API stop reason indicating an output token limit (e.g. `max_tokens`, `length`) | output.truncated |
| Model API stop reason indicating refusal | policy.model_refusal |
| Model API stop reason indicating a content filter | policy.content_filtered |
| Model API stop reason indicating context window overflow | context.window_exceeded |
| HTTP 429 from a model API | upstream.rate_limited |
| HTTP 529 or 503 indicating overload | upstream.overloaded |
| Framework "max turns exceeded" exception | loop.max_turns_exceeded |
| Framework output-parser exception | output.malformed or output.schema_violation |
| Framework "retries exhausted" exception for structured output | output.retries_exhausted |
| Framework guardrail tripwire | policy.guardrail_triggered |
{: title="Example mappings"}

# Open Issues
{:numbered="false"}

This section is to be removed before publication.

* Should the problem type be split into one type URI per category,
  instead of one type with a `code` member?
* Is a separate IANA registry justified, or should codes be registered
  as individual problem types?
* Should streaming partial results that accompany an error be in scope?
* Should there be a code for model output that is well-formed but
  semantically invalid as judged by a validator (e.g. `output.rejected`)?
* Relationship to work on agent protocols elsewhere in the IETF.
* MCP `_meta` key: `org.agenterrors/harness-error` is controlled by the
  editors. If MCP adopts this mechanism (e.g. via an MCP SEP), the key
  could move to an MCP-reserved prefix.
* Should the pre-registration problem type URI
  (`https://agenterrors.org/problems/harness-error`) remain valid
  permanently, or be retired after IANA registration?
* The MCP and A2A error codes and mechanisms referenced here should be
  re-verified against the latest versions of those specifications
  before each draft revision.

# MCP and A2A Examples
{:numbered="false"}

An MCP tool execution error. The text content is what the model sees;
the `_meta` entry is for the harness.

~~~ json
{
  "jsonrpc": "2.0",
  "id": 7,
  "result": {
    "isError": true,
    "content": [
      { "type": "text",
        "text": "Argument 'date' must be YYYY-MM-DD, got '11/04'." }
    ],
    "_meta": {
      "org.agenterrors/harness-error": {
        "type": "https://iana.org/assignments/http-problem-types#agent-harness-error",
        "title": "Invalid Tool Arguments",
        "code": "output.invalid_tool_arguments",
        "origin": "model",
        "model_recoverable": true,
        "model_feedback": "Argument 'date' must be YYYY-MM-DD, got '11/04'.",
        "location": { "tool_name": "book_meeting", "pointer": "/date" }
      }
    }
  }
}
~~~

An A2A task that failed because the remote agent hit its turn limit:

~~~ json
{
  "id": "task-123",
  "status": {
    "state": "failed",
    "message": {
      "role": "agent",
      "parts": [
        { "kind": "text",
          "text": "Stopped after reaching the 50-turn limit." },
        { "kind": "data",
          "data": {
            "type": "https://iana.org/assignments/http-problem-types#agent-harness-error",
            "title": "Maximum Turns Exceeded",
            "code": "loop.max_turns_exceeded",
            "origin": "harness",
            "retryable": false,
            "model_recoverable": false,
            "limit": { "name": "turns", "value": 50, "observed": 50 }
          } }
      ]
    }
  }
}
~~~

The delegating harness reports this to its own caller as
`upstream.agent_failed` with the object above in `causes`.

# Acknowledgments
{:numbered="false"}

TODO.
