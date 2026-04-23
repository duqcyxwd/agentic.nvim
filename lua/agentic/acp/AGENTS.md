# Provider system

Generic ACP client: one `ACPClient` for every provider, no per-provider
adapters (ADR 0005). Provider quirks live in protected `ACPClient` methods.

Adding a provider is a config entry under `acp_providers` in
`lua/agentic/config_default.lua`. Runtime flow, lifecycle, and framing: skill
`agentic-acp-protocol-flow`.

## Protocol contracts

- `set_config_option(session_id, config_id, ...)` takes `ConfigOption.id`,
  never the category string (`"mode"`, `"model"`). `id` and `category` are
  different fields.
- The `session/request_permission` response is double-nested on purpose, per
  the ACP spec: `{ outcome = { outcome = "selected", optionId = ... } }`.
- Sessions live only with the provider, listed via `session/list`. Local session
  persistence was removed in PR #169; do not add a disk cache fallback.
- The raw-input diff fallback reads `replace_all` (snake_case) only. No known
  provider sends `replaceAll`.

## A request handler MUST always answer; a notification handler need not

`ACPClient:_handle_notification` dispatches two shapes of inbound JSON-RPC
message, and they carry opposite obligations:

- A **notification** (`session/update`) has no `id`. Nothing is waiting on it, so
  dropping it when the subscriber is gone is correct.
- A **request** (`session/request_permission`) has an `id`. The provider blocks
  until that `id` is answered, and the subprocess is **shared across every
  session** (ADR 0004), so one unanswered request hangs the agent for all of them.
  Every exit path owes a `__send_result`.

`ACPClient:__with_subscriber` therefore takes an `on_missing` callback:

```lua
--- @param on_missing fun()|nil Runs when no subscriber answers
function ACPClient:__with_subscriber(session_id, callback, on_missing)
```

It fires on **both** misses — the synchronous one (no subscriber when the message
arrives) and the deferred one (the subscriber was dropped while the `vim.schedule`
body sat in the queue). The subscriber is re-resolved **inside** the schedule for
exactly that reason; `cancel_session` nils it mid-flight.

Only `__handle_request_permission` passes `on_missing`, answering
`{ outcome = "cancelled" }`. The three notification call sites pass nothing and
keep returning silently.

Regression:
`lua/agentic/acp/acp_client.test.lua::"answers cancelled when the subscriber is gone"`.

### The dispatch body MUST answer even when it throws

A missing subscriber is not the only way to strand the `id`. The pre-dispatch
type guard in `__handle_request_permission` only validates the payload's **top
level**; everything below it runs unchecked inside the `vim.schedule` body, where
three reachable throws left the `id` unanswered:

- `subscriber.on_request_permission` throws (a UI defect)
- `subscriber.on_tool_call_update` throws (same)
- `__build_tool_call_message` throws on a payload the type guard accepts.
  `update.content` is type-checked but its **elements** are not, so
  `toolCall = { toolCallId = "t1", content = { 5 } }` indexes a number.

`vim.schedule` swallows the error, so the transport read loop survives — which is
why this failed silently rather than crashing. The shared subprocess (ADR 0004)
just waits, and every session hangs with it.

The dispatch body is therefore wrapped in a `pcall` that answers
`{ outcome = "cancelled" }` on failure. Two rules govern it:

- It MUST `Logger.notify` the error, **including when `answer` is already a
  no-op**. A silent cancel hides genuine UI bugs in `on_request_permission`
  behind a permission prompt that merely "didn't appear".
- `answer` MUST be idempotent. The subscriber can invoke its callback and _then_
  throw; the `id` is already answered at that point, and a second `__send_result`
  on one `id` is a protocol violation. The guard's cancel is a no-op once
  answered, so a real `selected` outcome is never overwritten.

Do NOT harden `__build_tool_call_message` instead. One guard at the dispatch
boundary closes all three variants; hardening the builder is a larger diff for
the same bug and still would not cover a throwing subscriber callback.

Regressions:

- `lua/agentic/acp/acp_client.test.lua::"answers cancelled when on_request_permission throws"`
- `lua/agentic/acp/acp_client.test.lua::"answers cancelled when on_tool_call_update throws"`
- `lua/agentic/acp/acp_client.test.lua::"answers cancelled when the nested tool call content is malformed"`
- `lua/agentic/acp/acp_client.test.lua::"answers once when the handler answers and then throws"`

### Permission state is per-`id`, never shared

An agent can leave several `session/request_permission` requests outstanding at
once, and the user may answer them in any order — or never answer some. The
`PermissionManager` keys `pending` by `tool_call_id` and supports out-of-order
`resolve`, so the ACP layer must match it.

`answered` and `answer` are therefore **locals of each
`__handle_request_permission` invocation**, closing over that invocation's
`message_id`. Hoisting either onto `self` (or any module-level table) crosses the
wires: a shared `answered` swallows the second answer, and a shared `message_id`
pairs one request's `optionId` with another's `id`.

Each outstanding request also owns its failures. A throw in one dispatch cancels
only that `id`, and a subscriber dropped between two dispatches cancels only the
one still queued — the sibling holding a live `answer` closure resolves normally.

`PermissionManager:clear()` fires **every** pending callback with `nil`, so all
outstanding `id`s answer `cancelled`. That is correct on teardown: each `answer`
is a distinct closure, so the manager's fan-out maps one-to-one onto the
outstanding requests with no double-send.

Regressions in
`lua/agentic/acp/acp_client.test.lua::"__handle_request_permission concurrency"`,
notably `"answers two outstanding requests in reverse order"`.

## Tool call quirk fallbacks

- **`rawInput` fallback** (OpenCode): when `content` is missing for `edit` kind
  tool calls, builds diff from `rawInput.new_string`/`rawInput.newString` fields
- **`locations` fallback**: extracts `file_path` from `update.locations[0].path`
  when not in `rawInput`
- **Unknown kinds**: logs a warning for unrecognized `kind` values so users
  report them as issues

To handle a new provider quirk, add the fallback logic in
`__build_tool_call_message` with a comment explaining which provider needs it.

## Kiro ACP extensions (`_kiro.dev/*`)

Kiro CLI emits extension notifications prefixed with `_kiro.dev/` per the ACP
spec (see https://kiro.dev/docs/cli/acp/#kiro-extensions). These are optional;
clients that don't understand them can safely ignore them.

`_handle_notification` dispatches any method matching `_kiro.dev/*` to
`__handle_kiro_notification`. Current behaviour:

| Method                             | Handling                                                                     |
| ---------------------------------- | ---------------------------------------------------------------------------- |
| `_kiro.dev/commands/available`     | Reshaped into an `available_commands_update` and forwarded to the subscriber |
| `_kiro.dev/mcp/server_initialized` | Debug-logged, no UI surface yet                                              |
| `_kiro.dev/metadata`               | Debug-logged                                                                 |
| `_kiro.dev/subagent/list_update`   | Debug-logged                                                                 |
| `_kiro.dev/compaction/status`      | Debug-logged                                                                 |
| `_kiro.dev/clear/status`           | Debug-logged                                                                 |
| `_session/terminate`               | Debug-logged (subagent teardown, no client-side bookkeeping needed)          |

Adding real handling for a new `_kiro.dev/*` extension means extending
`__handle_kiro_notification` and, if the update has a matching ACP session update
type, reshaping it to that type so existing subscriber code picks it up — same
pattern used for `_kiro.dev/commands/available`.

## Permission flow (interleaved with tool calls)

```
Provider sends "session/request_permission"
  -> SessionManager: opens diff preview (if the request carries a diff)
  -> PermissionManager:add_request(request, callback)
     -> Queues request (sequential — one prompt at a time)
     -> Renders permission buttons in chat buffer
     -> Sets up buffer-local keymaps (1,2,3,4)
  -> User presses key
     -> Sends result back to provider via callback
     -> Clears diff preview
     -> Dequeues next permission if any
```

## Protected methods in ACPClient

These protected methods can be overridden by subclasses if a future provider
requires it, but currently all providers use the default implementations:

| Method                        | Behavior                                  |
| ----------------------------- | ----------------------------------------- |
| `__handle_tool_call`          | Builds ToolCallBlock, notifies subscriber |
| `__build_tool_call_message`   | Parses ACP fields + quirk fallbacks       |
| `__handle_tool_call_update`   | Builds partial, notifies subscriber       |
| `__handle_request_permission` | Sends result back to provider             |
| `__handle_session_update`     | Routes by `sessionUpdate` type            |

