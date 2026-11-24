---
name: trustabl-pre-coding
description: >-
  Pre-coding reliability constraints, used for writing and reviewing agent definitions with: claude_sdk, claude_skill, mcp
allowed-tools: Read
disable-model-invocation: false
---

# Trustabl Pre-Coding Reliability Constraints

<!-- generated: 2026-11-24 | rules: 6ac572308206b00fb779238f8e990ab8c16604cc | schema: 41 | sdks: claude_sdk, claude_skill, mcp | template: 4 -->

Before writing any agent code, apply every constraint below. Rules are
ordered by severity. A violation here will fire the corresponding finding
in post-build scan — prevent it now.

## How to Apply These Constraints

Work against this document; do not assume a definition is correct because it
looks right. After writing or changing any tool, agent, subagent, or skill
definition, run this loop before moving on.

1. CHECK YOUR WORK
   Re-read the definition you just wrote against every constraint in this
   document whose "When this applies" matches it. Check explicitly — do not
   assume the constraint was satisfied.

2. NAME THE VIOLATION
   State the specific rule ID, not "this looks wrong". "CSDK-005 — this tool
   raises without a structured error contract" is actionable; "error handling
   needs work" is not. If nothing matches, the definition passes; move on.

3. MATCH THE REPAIR TO THE VIOLATION
   Apply that rule's own Directive. Where the repair goes is set by scope,
   and how to proceed is set by severity:

     tool   → change the tool definition
     agent  → change the agent constructor call
     repo   → change project configuration, not code

     critical / high  → make the change, then state which rule required it
     medium / low     → apply the directive directly

   If the Directive cannot be applied as written — it conflicts with another
   constraint here, or the fix is outside the file you are editing — stop and
   say so rather than approximating it.

4. KEEP A TRAIL
   Note the rule ID and the change that cleared it. Do not reintroduce a
   pattern you already repaired in this session, and do not re-apply a repair
   that did not clear the violation — report it instead.

Scope: this loop applies to agent, tool, subagent, and skill definitions —
the surfaces the constraints below govern. It is not a general code-review
procedure.

## Tool-Call Claims

This section binds every tool call that has an effect to a typed action claim
and a check that runs before the call has any effect. It follows Explanation-Bound Tool
Execution (EBTE, arXiv 2607.25364): the decision-relevant part of the model's
reasoning becomes a small typed object, and code outside the model compares
that object with facts the model cannot change.

A claim is untrusted input. It never grants authority, never widens what the
agent is already allowed to do, and never replaces an existing control. The
deterministic controls (allow and deny lists, schema validation, sandbox and
network policy, approval gates) stay the floor; the claim check can only add a
reason to stop.

### The claim

Every tool with an effect takes one extra argument, `claim`, typed as
`ToolInvocationExplanation`. A tool has an effect when it creates, changes,
deletes, sends, pays, deploys or runs anything, or sends data the model chose
to a destination outside the application. A tool that only reads data the
application already holds, for the current user, may skip the claim. Mark each
tool in the tool registry as needing a claim or not, so the mediation step can
tell a skipped claim from a missing one. Tools the application does not define
(built-in SDK tools, third-party MCP servers) cannot take the argument; they
keep the existing controls only.

Keep the enforced fields small, typed and categorical. Free text is advisory
only and no check reads it.

```
ToolInvocationExplanation
  schema_version     "ebte-0.1"
  intent_class       one value from a closed enum the application defines
                     (for example: read, search, create, update, delete, send)
  selected_tool      the tool's registered name
  expected_effect
    operation        one value from a closed enum (read, create, update, delete, send, execute)
    resource_type    a closed enum value, or null
    resource_ids     bounded list of identifiers, or empty
    max_items        integer upper bound on items touched
    destination      allowed destination identifier, or null
  risk_tier          low | medium | high
  uncertainty        low | medium | high
  untrusted_context  list of { source, digest } for each untrusted input the
                     call depends on (tool output, fetched page, file, email)
  evidence_refs      list of { type: intent | policy | payload, digest }
  summary            optional short text; advisory, never checked
```

Rules for the schema:

- Use enums and bounded lists, not prose. A field a check cannot compare is
  not an enforced field.
- `selected_tool` must equal the tool actually being called.
- `untrusted_context` must list every untrusted input the call depends on. An
  empty list is a claim too, and it is checked.
- Python uses the names above; TypeScript and Go may use camelCase, as long as
  one spelling is used across the application.

### The mediation step

Put one mediation step between the model's tool call and the tool's effect.
It is a function the model cannot call or skip, and it runs on every call.

1. Read `claim` from the call's arguments. When the step runs in a wrapper
   that can change the arguments, remove `claim` before the tool function is
   called. A hook that can only allow or reject leaves it in place. Either
   way, nothing after the step acts on the claim.
2. Validate the claim against the schema. A missing claim on a tool that needs
   one, a malformed claim, or an unknown `schema_version` is a deny. The model
   gets the coarse reason and can call again with a valid claim.
3. Resolve the facts from the server side, never from the claim or the
   conversation: the tool registry entry, the task's authorized intent, the
   current policy, the route the call is allowed to take, and the context-risk
   state for this run.
4. Compare. Each check gives allow, review or deny:
   - `selected_tool` differs from the called tool, or the tool is not on the
     authorized route → deny.
   - `intent_class` or `expected_effect.operation` is outside the authorized
     intent → deny.
   - The arguments go beyond `expected_effect` (more items, another
     destination, other resources), or the claim goes beyond the intent → deny.
     Required order: arguments ≤ claim ≤ authorized intent.
   - `untrusted_context` leaves out an untrusted input the run did use → deny.
   - A digest in `evidence_refs` is stale or does not match → deny.
   - A fact cannot be resolved → review. Unknown is never a match.
   - `uncertainty` is high, or `risk_tier` is higher than the server's risk
     for the tool → review. A declared risk can raise the server's risk, never
     lower it.
   - The tool is high risk on the server side → review, even when every
     check matches.
5. Combine: the result is the most restrictive outcome of every check, then
   the most restrictive of that result and the existing authorization and
   approval controls. Deny beats review, review beats allow.
6. Act on it. Only allow runs the tool. Review runs nothing: it returns a
   draft or a confirmation request through the SDK's approval mechanism, and
   an approval is a new decision that re-resolves the facts and runs the
   checks again. Deny returns an error to the model.
7. Tell the model only a coarse reason (for example "claim does not match
   route"). Keep the per-check detail in the audit trail, so the reasons
   cannot be used to probe the checks.

### Where the step goes

Put the step where the SDK gives control before a tool runs. When the SDK has
no such hook, put it in one shared wrapper that every tool goes through, as
the first thing it does — never copied into each tool by hand.

#### Claude Agent SDK

Run the mediation step in a `PreToolUse` hook (or the `canUseTool` /
`can_use_tool` callback) and return a deny or ask decision from it. Map review
to ask.

#### MCP

Run the mediation step inside the server's tool handler, before any side
effect. A claim from the client is as untrusted as any other argument; the
facts come from the server.

### Check before moving on

Before moving on, confirm each of these. If one does not hold, say which one
and why, rather than working around it.

- Every tool with an effect takes a `claim` argument, and the mediation step reads it before
  the tool has any effect. Nothing after the step acts on the claim.
- The mediation step runs on every call path, including retries, handoffs and
  sub-agents. A tool that can be reached without it is not covered.
- Every fact the step compares against comes from the server side. Nothing in
  the claim, the prompt or a tool result decides which facts are used.
- A missing claim on a tool that needs one is a deny, an unresolved fact is a
  review, and only allow runs the tool.
- The claim check is combined with the existing controls, and the most
  restrictive outcome wins. It never turns another control's deny or review
  into allow.
- The audit trail holds the schema version, the outcome, the check outcomes
  and digests — not raw untrusted content and not the `summary` text.

---

## Claude Agent SDK subagent wiring safety

### Tool Rules

---

#### [CSDK-003] Network call has no timeout
**Severity:** high | **Confidence:** 0.85

**Directive:** Pass `timeout=` (typically 5–30s) to the request.

**Why:** An agent tool that makes a network request without a timeout can hang indefinitely, blocking the conversation loop and exhausting the agent's wall-clock budget.

**When this applies:** When defining a tool.

---

#### [CSDK-004] Path parameter used in I/O without validation
**Severity:** high | **Confidence:** 0.70

**Directive:** Resolve the path with `Path(...).resolve()` and assert it sits under an allowed root.

**Why:** The tool accepts a path-like parameter and passes it to file or directory operations without resolving or sandboxing the path.

**When this applies:** When defining a tool.

---

#### [CSDK-009] Tool fetches a caller-controlled URL (SSRF)
**Severity:** high | **Confidence:** 0.60

**Directive:** Don't fetch arbitrary URLs.

**Why:** This tool issues an HTTP request to a URL built from a parameter or an interpolated string rather than a fixed literal.

**When this applies:** When defining a tool.

---

#### [CSDK-010] TypeScript Claude SDK tool shells out to the OS
**Severity:** high | **Confidence:** 0.70

**Directive:** Avoid shelling out from agent-callable tools.

**Why:** This Claude Agent SDK tool, authored in TypeScript, invokes a child-process API.

**When this applies:** When defining a tool.

---

#### [CSDK-011] TypeScript Claude SDK tool evaluates dynamic code
**Severity:** high | **Confidence:** 0.90

**Directive:** Remove `eval`/`new Function`.

**Why:** This TypeScript Claude SDK tool calls `eval()` or constructs a `new Function(...)`.

**When this applies:** When defining a tool.

---

#### [CSDK-013] TypeScript Claude SDK tool fetches a caller-controlled URL (SSRF)
**Severity:** high | **Confidence:** 0.60

**Directive:** Validate the destination before fetching: resolve the host and reject private, loopback, and link-local ranges; prefer an allowlist of exact hosts.

**Why:** This TypeScript Claude SDK tool issues an HTTP request to a URL built from a tool argument or an interpolated string rather than a fixed literal.

**When this applies:** When defining a tool.

---

#### [CSDK-020] TypeScript Claude SDK tool HTTP call has no timeout
**Severity:** high | **Confidence:** 0.60

**Directive:** Attach a deadline.

**Why:** This TypeScript Claude SDK tool makes an outbound HTTP call (fetch / axios / got / undici) with no deadline — no signal, timeout, or abortSignal option.

**When this applies:** When defining a tool.

---

#### [CSDK-023] TypeScript side-effecting tool lets the model choose the recipient or amount with no visible bound
**Severity:** high | **Confidence:** 0.50

**Directive:** Do not let the model supply the recipient or the amount unconstrained.

**Why:** This Claude Agent SDK tool, created with tool(name, description, schema, handler), has a name that signals a send/notify or a refund/charge/pay side effect, and its schema takes a free-form recipient or amount parameter with no visible bound: no allow-list constraining who the message reaches, and no cap, max, or enum constraining how much money moves.

**When this applies:** When defining a tool.

---

#### [CSDK-107] Tool body calls eval/exec/compile on dynamic input
**Severity:** high | **Confidence:** 0.85

**Directive:** Remove eval/exec/compile from the tool.

**Why:** The tool body invokes Python's eval, exec, or compile.

**When this applies:** When defining a tool.

---

#### [CSDK-108] Tool body spawns a subprocess
**Severity:** high | **Confidence:** 0.70

**Directive:** Prefer a typed library API over shelling out (e.g.

**Why:** The tool body spawns an OS process (subprocess.*, os.system, os.popen, or os.spawn*).

**When this applies:** When defining a tool.

---

#### [CSDK-002] Tool parameters are not type-annotated
**Severity:** medium | **Confidence:** 0.90

**Directive:** Annotate every parameter with a type.

**Why:** Without parameter type annotations, the SDK cannot generate an input schema to validate model output.

**When this applies:** When defining a tool.

---

#### [CSDK-006] Mutating tool has no idempotency key
**Severity:** medium | **Confidence:** 0.55

**Directive:** Add an `idempotency_key: str` parameter; verify the receiving API respects it.

**Why:** Tool name suggests a side effect (create/send/refund/…).

**When this applies:** When defining a tool.

---

#### [CSDK-008] Tool exposes **kwargs without explicit input_schema
**Severity:** medium | **Confidence:** 0.80

**Directive:** Either declare each accepted parameter on the function signature with a type annotation so the SDK can derive the schema, or pass an explicit `input_schema=` to the `@tool` decorator — either a JSON Schema dict or a Pydantic `BaseModel` subclass — so the contract published to the model matches what the body actually reads from `kwargs`.

**Why:** The Claude Agent SDK derives a tool's JSON input schema from the function signature.

**When this applies:** When defining a tool.

---

#### [CSDK-016] TypeScript Claude SDK mutating tool has no idempotency key
**Severity:** medium | **Confidence:** 0.50

**Directive:** Add an `idempotencyKey` (or `requestId`) field to the tool's Zod schema, thread it to the downstream API, and confirm that API treats a repeated key as a no-op rather than a second mutation.

**Why:** This TypeScript tool's name implies a side effect (create/send/refund/…) but its Zod schema exposes no idempotency parameter.

**When this applies:** When defining a tool.

---

#### [CSDK-024] Claude SDK tool fetches an allow-listed host without pinning https://
**Severity:** medium | **Confidence:** 0.60

**Directive:** Pin the scheme: build the URL from a literal "https://" prefix, or reject any URL whose urlparse(url).scheme is not "https" before the request.

**Why:** This Claude SDK tool checks the destination host against an allow-list (so CSDK-009 stays silent) but builds its outbound request URL without a literal https:// prefix and without checking the URL scheme.

**When this applies:** When defining a tool.

---

#### [CSDK-025] TypeScript Claude SDK tool fetches an allow-listed host without pinning https://
**Severity:** medium | **Confidence:** 0.60

**Directive:** Pin the scheme: build the URL from a literal `https://` prefix, or reject any URL whose `new URL(x).protocol` is not "https:" before fetching.

**Why:** This TypeScript Claude SDK tool checks the destination host against an allow-list (so CSDK-013 stays silent) but passes fetch/axios/got a URL that does not begin with a literal https:// prefix and checks no URL protocol.

**When this applies:** When defining a tool.

---

#### [CSDK-027] Tool declares a claim argument that nothing checks
**Severity:** medium | **Confidence:** 0.60

**Directive:** Check the claim before the effect: a `PreToolUse` hook on `ClaudeAgentOptions(hooks={"PreToolUse": [HookMatcher(matcher="mcp__<server>__<tool>", hooks=[check])]})` (it receives the call's `tool_input` and blocks it with `permissionDecision: "deny"`), or one shared decorator stacked under `@tool`.

**Why:** This Claude Agent SDK `@tool` handler's input schema has a `claim` key, so the model fills in a typed claim about the call, but the handler never reads `args["claim"]` or the whole `args` dict, the tool has no decorator besides the one that registers it, and the repo registers no PreToolUse hook (and no can_use_tool callback that runs for it), so no mediation step visible here can check it.

**When this applies:** When defining a tool.

---

#### [CSDK-028] Tool's claim argument is only written out or returned, never checked
**Severity:** medium | **Confidence:** 0.55

**Directive:** Pass the claim to a verifier before the effect and act on its outcome: deny on a conflict between the claim and the arguments, task or policy, review on anything unresolved or high risk, and run the tool only on allow.

**Why:** This Claude Agent SDK `@tool` handler reads its `claim` input, but every read only writes it out — to print, a logger, json.dumps, str or repr — or puts it in the return value.

**When this applies:** When defining a tool.

---

#### [CSDK-001] Tool has no description
**Severity:** low | **Confidence:** 0.95

**Directive:** Add a one-paragraph docstring describing inputs, outputs, and when to use this tool.

**Why:** The Claude Agent SDK uses the tool's docstring as the description shown to the model.

**When this applies:** When defining a tool.

---

#### [CSDK-005] Tool raises exceptions without a structured error contract
**Severity:** low | **Confidence:** 0.60

**Directive:** Catch known failure modes and return a `{"error": ..., "retryable": bool}` payload.

**Why:** When a tool raises, the SDK surfaces the exception to the model as an opaque string.

**When this applies:** When defining a tool.

---

#### [CSDK-007] Ambiguous tool name
**Severity:** low | **Confidence:** 0.90

**Directive:** Rename to a verb-object form, e.g.

**Why:** Tool names like `process`, `handle`, or `run` give the model no signal about intent.

**When this applies:** When defining a tool.

---

#### [CSDK-012] TypeScript Claude SDK tool writes to the filesystem
**Severity:** low | **Confidence:** 0.50

**Directive:** Confine writes to a dedicated working directory, resolve the final path and verify it stays within that root before writing, and reject absolute or `..`-containing inputs.

**Why:** This TypeScript Claude SDK tool writes to the filesystem.

**When this applies:** When defining a tool.

---

#### [CSDK-014] TypeScript Claude SDK tool has no description
**Severity:** low | **Confidence:** 0.90

**Directive:** Pass a clear one-line description as the second argument to `tool(name, description, schema, handler)`, stating what the tool does, what it returns, and when the model should reach for it.

**Why:** The Claude Agent SDK passes a TypeScript tool's description — the second argument to `tool(name, description, schema, handler)` — to the model as the sole signal for when to invoke it.

**When this applies:** When defining a tool.

---

#### [CSDK-017] Tool description is a placeholder
**Severity:** low | **Confidence:** 0.85

**Directive:** Replace the placeholder with a real one-paragraph description covering what the tool does, what it returns, and when the model should call it.

**Why:** The description passes the has-docstring check but carries a placeholder marker instead of real content.

**When this applies:** When defining a tool.

---

#### [CSDK-018] Tool description is too short to guide model selection
**Severity:** low | **Confidence:** 0.80

**Directive:** Expand the description to at least a full sentence covering inputs, outputs, and the situation in which this tool should be used over alternatives.

**Why:** A description under 40 characters is rarely enough to convey what a tool does, what it returns, and when to call it versus a similarly named neighbor.

**When this applies:** When defining a tool.

---

#### [CSDK-019] Tool prints to stdout for diagnostics
**Severity:** low | **Confidence:** 0.65

**Directive:** Remove the print().

**Why:** The tool body calls print(), which writes diagnostics to the process's stdout.

**When this applies:** When defining a tool.

---

#### [CSDK-021] TypeScript Claude SDK tool description is a placeholder
**Severity:** low | **Confidence:** 0.85

**Directive:** Replace the placeholder with a real description covering what the tool does, what it returns, and when the model should call it rather than a neighboring tool.

**Why:** The tool sets a description, so it passes CSDK-014, but the string is a placeholder rather than real content.

**When this applies:** When defining a tool.

---

#### [CSDK-022] TypeScript Claude SDK tool description is too short to guide model selection
**Severity:** low | **Confidence:** 0.80

**Directive:** Expand the description to at least a full sentence covering inputs, outputs, and the situation in which this tool should be used over the alternatives.

**Why:** A description under 40 characters is rarely enough to convey what a tool does, what it returns, and when to call it rather than a similarly named neighbor.

**When this applies:** When defining a tool.

---

#### [CSDK-026] Side-effecting tool takes no structured claim for its calls
**Severity:** info | **Confidence:** 0.50

**Directive:** Add a reserved `claim` key to the tool's input schema, typed as a ToolInvocationExplanation (schema_version, intent_class, selected_tool, expected_effect, risk_tier, uncertainty, untrusted_context, evidence_refs; an optional summary that no check reads) and check it in a mediation step before the effect: a `PreToolUse` hook on `ClaudeAgentOptions(hooks={"PreToolUse": [HookMatcher(matcher="mcp__<server>__<tool>", hooks=[check])]})` (it receives the call's `tool_input` and blocks it with `permissionDecision: "deny"`), or one shared decorator stacked under `@tool`.

**Why:** This Claude Agent SDK `@tool` handler has a side effect — it shells out, executes code, writes a file, or its name says it sends, deletes, pays, deploys, creates or updates — and its input schema has no `claim` key, so nothing about why the model chose this call or what it expects the call to do reaches the code that runs it.

**When this applies:** When defining a tool.

### Agent Rules

---

#### [CSDK-101] Claude subagent is granted the Bash tool
**Severity:** high | **Confidence:** 0.80

**Directive:** Remove "Bash" from this subagent's tools list unless shell execution is essential to its job.

**Why:** This AgentDefinition grants the built-in Bash tool, giving the subagent arbitrary shell-command execution.

**When this applies:** When declaring an agent.

---

#### [CSDK-103] AgentDefinition sets permissionMode to bypassPermissions with a broad tool set
**Severity:** high | **Confidence:** 0.90

**Directive:** Remove permissionMode="bypassPermissions" unless this subagent runs in a fully sandboxed, non-interactive context you control (e.g.

**Why:** This AgentDefinition sets permissionMode="bypassPermissions", which disables the SDK's interactive approval gate for every tool the subagent runs, and either omits tools entirely (which inherits every tool available to subagents) or lists a side-effecting or exfiltration- capable one.

**When this applies:** When declaring an agent.

---

#### [CSDK-104] Claude subagent is granted filesystem-write built-ins
**Severity:** high | **Confidence:** 0.80

**Directive:** Remove the write built-ins from this subagent's tools list unless file mutation is essential to its job.

**Why:** This AgentDefinition grants a filesystem-write built-in (Write, Edit, MultiEdit, or NotebookEdit), letting the subagent modify files on disk.

**When this applies:** When declaring an agent.

---

#### [CSDK-105] Claude subagent is granted the WebFetch tool
**Severity:** high | **Confidence:** 0.75

**Directive:** Remove "WebFetch" from this subagent's tools list unless fetching is essential.

**Why:** This AgentDefinition grants the built-in WebFetch tool, which retrieves a model-chosen URL and feeds its content back into the agent loop.

**When this applies:** When declaring an agent.

---

#### [CSDK-120] TypeScript AgentDefinition sets permissionMode to bypassPermissions with a broad tool set
**Severity:** high | **Confidence:** 0.90

**Directive:** Remove permissionMode: "bypassPermissions" unless this agent runs in a fully sandboxed, non-interactive context you control (e.g.

**Why:** This TypeScript AgentDefinition sets permissionMode: "bypassPermissions", which disables the SDK's interactive approval gate for every tool the agent runs, and either omits tools entirely (which inherits every tool available to subagents) or lists a side-effecting or exfiltration- capable one.

**When this applies:** When declaring an agent.

---

#### [CSDK-121] TypeScript AgentDefinition is granted the Bash tool
**Severity:** high | **Confidence:** 0.80

**Directive:** Remove "Bash" from this agent definition's tools array unless shell execution is essential to its job.

**Why:** This TypeScript AgentDefinition (a typed const or an inline entry in options.agents) lists "Bash" in its tools array, giving the subagent arbitrary shell-command execution.

**When this applies:** When declaring an agent.

---

#### [CSDK-123] TypeScript AgentDefinition is granted filesystem-write built-ins
**Severity:** high | **Confidence:** 0.80

**Directive:** Remove the write built-ins from this agent definition's tools array unless file mutation is essential to its job.

**Why:** This TypeScript AgentDefinition lists a filesystem-write built-in (Write, Edit, MultiEdit, or NotebookEdit) in its tools array, letting the subagent modify files on disk.

**When this applies:** When declaring an agent.

---

#### [CSDK-124] TypeScript AgentDefinition is granted the WebFetch tool
**Severity:** high | **Confidence:** 0.75

**Directive:** Remove "WebFetch" from this agent definition's tools array unless fetching is essential.

**Why:** This TypeScript AgentDefinition lists "WebFetch" in its tools array.

**When this applies:** When declaring an agent.

---

#### [CSDK-130] TypeScript query() main agent is granted the Bash tool
**Severity:** high | **Confidence:** 0.80

**Directive:** Remove "Bash" from `options.allowedTools` unless shell execution is essential to the agent's task.

**Why:** This `query(...)` call configures the main agent thread with "Bash" in its `options.allowedTools`, granting the top-level agent arbitrary shell-command execution.

**When this applies:** When declaring an agent.

---

#### [CSDK-131] TypeScript query() main agent is granted filesystem-write or web-fetch built-ins
**Severity:** high | **Confidence:** 0.75

**Directive:** Drop the write and fetch built-ins from `options.allowedTools` unless the task requires them.

**Why:** This `query(...)` main agent's `options.allowedTools` includes a filesystem-write built-in (Write/Edit/MultiEdit/NotebookEdit) or WebFetch.

**When this applies:** When declaring an agent.

---

#### [CSDK-102] Claude subagent is granted the WebSearch tool
**Severity:** medium | **Confidence:** 0.80

**Directive:** Remove "WebSearch" from this subagent's tools list unless internet access is essential to its job.

**Why:** This AgentDefinition grants the built-in WebSearch tool, giving the subagent access to live internet content.

**When this applies:** When declaring an agent.

---

#### [CSDK-122] TypeScript AgentDefinition is granted the WebSearch tool
**Severity:** medium | **Confidence:** 0.80

**Directive:** Remove "WebSearch" from this agent definition's tools array unless internet access is essential to its job.

**Why:** This TypeScript AgentDefinition lists "WebSearch" in its tools array, giving the subagent access to live internet content.

**When this applies:** When declaring an agent.

---

#### [CSDK-114] TypeScript query() main agent sets no maxBudgetUsd spend cap
**Severity:** low | **Confidence:** 0.60

**Directive:** Set `maxBudgetUsd` in the `query()` options to a ceiling sized to the task (a few dollars for an interactive agent, less for a batch worker), and pair it with `maxTurns`.

**Why:** This `query(...)` call configures the main agent thread with inline `options` but sets no `options.maxBudgetUsd`.

**When this applies:** When declaring an agent.

### Subagent Rules

---

#### [CSDK-110] Subagent granted the built-in Bash tool
**Severity:** high | **Confidence:** 0.90

**Directive:** Remove `Bash` from the subagent's `tools:` list unless shell access is essential.

**Why:** This subagent's frontmatter grants the built-in `Bash` tool, letting it run arbitrary shell commands.

**When this applies:** When declaring a subagent.

---

#### [CSDK-111] Subagent granted filesystem-write or web-fetch built-ins
**Severity:** high | **Confidence:** 0.85

**Directive:** Remove the write and fetch built-ins from the subagent's `tools:` list unless the role genuinely requires them.

**Why:** This subagent's frontmatter grants a filesystem-write (`Write`/`Edit`/`MultiEdit`/`NotebookEdit`) or web-fetch (`WebFetch`) built-in.

**When this applies:** When declaring a subagent.

---

#### [CSDK-115] Subagent omits tools and inherits the full subagent tool pool
**Severity:** high | **Confidence:** 0.75

**Directive:** Add a `tools:` list naming the tools the role needs, for example `tools: Read, Grep, Glob` for a review or search role.

**Why:** This subagent file sits in a `.claude/agents/` directory and its frontmatter has no `tools:` key and no `disallowedTools:` entries.

**When this applies:** When declaring a subagent.

---

#### [CSDK-112] Subagent granted the WebSearch tool
**Severity:** medium | **Confidence:** 0.80

**Directive:** Remove `WebSearch` from the subagent's `tools:` list unless the role is genuinely research-oriented.

**Why:** This subagent's frontmatter grants the built-in `WebSearch` tool, so search-result content chosen by the model flows back into the subagent's context.

**When this applies:** When declaring a subagent.

### Repo Rules

---

#### [CSDK-201] Project default permission mode bypasses approvals
**Severity:** high | **Confidence:** 0.90

**Directive:** Remove the "defaultMode": "bypassPermissions" entry from .claude/settings.json (and settings.local.json), or change it to "default" so tool calls prompt, or "acceptEdits" if you only want to auto-approve file edits while still gating shell and network actions.

**Why:** The project's .claude/settings.json sets defaultMode to "bypassPermissions", which disables Claude Code's permission prompts for the whole repo.

**When this applies:** For any repo using this SDK.

---

#### [CSDK-202] Session permission mode bypasses approvals with no tool deny-list
**Severity:** high | **Confidence:** 0.90

**Directive:** Drop permission_mode="bypassPermissions" from the ClaudeAgentOptions(...) call, or set it to "default" so tool calls prompt (or "acceptEdits" to auto-approve only file edits while still gating shell and network access).

**Why:** A ClaudeAgentOptions(...) in this project sets permission_mode to "bypassPermissions", which turns off Claude Code's approval prompts for that session, and the same construction sets no disallowed_tools.

**When this applies:** For any repo using this SDK.

---

#### [CSDK-205] Claude Agent SDK session auto-approves edits with no tool deny-list
**Severity:** medium | **Confidence:** 0.70

**Directive:** Pass disallowed_tools= to ClaudeAgentOptions(...), naming the tools this session must never call — at minimum shell execution and any tool that reaches the network or credentials.

**Why:** A ClaudeAgentOptions(...) in this project sets permission_mode to "acceptEdits", which auto-approves file writes and edits without a prompt, and no construction in the project sets disallowed_tools.

**When this applies:** For any repo using this SDK.

---

#### [CSDK-206] Session bypasses approvals with a deny-list that still leaves a broad surface
**Severity:** medium | **Confidence:** 0.60

**Directive:** Review the disallowed_tools list against every tool this session can reach and confirm it denies shell execution and anything that touches the network or credentials, not just an obvious tool or two.

**Why:** A ClaudeAgentOptions(...) in this project sets permission_mode to "bypassPermissions" and the same construction sets disallowed_tools, so the named tools are denied in every permission mode including bypassPermissions — a real, SDK-enforced bound on the tool surface, not just an auto-approve list.

**When this applies:** For any repo using this SDK.

---

#### [CSDK-203] Repo ships Claude Agent SDK code without an agent-guidance doc (AGENTS.md/CLAUDE.md)
**Severity:** low | **Confidence:** 0.90

**Directive:** Add a CLAUDE.md (or an AGENTS.md) at the repository root documenting how to build, test, and lint the project, the coding conventions the agent must respect, the directories or files it must not modify, and any project-specific safety guardrails (e.g.

**Why:** A repository that builds on the Claude Agent SDK but ships no agent-guidance doc — CLAUDE.md (the Claude Code convention) or AGENTS.md (the cross-vendor one) — leaves any coding agent that opens the repo with no project-specific guidance.

**When this applies:** For any repo using this SDK.

---

#### [CSDK-204] Claude Agent SDK session sets no explicit max_turns limit
**Severity:** low | **Confidence:** 0.60

**Directive:** Pass max_turns= to ClaudeAgentOptions(...), sized to the work the session actually does — a handful of turns for a single-file edit, more for a multi-step investigation.

**Why:** No ClaudeAgentOptions(...) in this project sets max_turns, so the agent loop runs to whatever ceiling the runtime applies by default rather than to a bound sized for this task.

**When this applies:** For any repo using this SDK.

---

#### [CSDK-208] Claude Agent SDK project wires no observability
**Severity:** low | **Confidence:** 0.75

**Directive:** Install an OpenTelemetry tracer provider at process start (opentelemetry.trace.set_tracer_provider(...)) with an OTLP exporter pointed at your collector, or initialize a vendor SDK — logfire.configure(), Langfuse's CallbackHandler, or braintrust.init_logger() — before the first query() call.

**Why:** This project runs Claude Agent SDK agents but wires no observability backend.

**When this applies:** For any repo using this SDK.

---

## Claude Agent Skill quality (text-match)

### Skill Rules

---

#### [CSKILL-003] Dynamic-context command performs network egress or reads secrets
**Severity:** critical | **Confidence:** 0.85

**Directive:** Remove the network and credential access from dynamic-context commands entirely.

**Why:** A dynamic-context command in this skill performs network egress (curl / wget / nc / …) or reads credentials/secrets (gh auth, $AWS_*, ~/.aws, ~/.ssh, id_rsa, *_key, …).

**When this applies:** Any dynamic-context command performing network egress or accessing credentials.

---

#### [CSKILL-011] Bundled skill script reads credentials or secrets
**Severity:** critical | **Confidence:** 0.80

**Directive:** Remove credential and secret access from bundled scripts.

**Why:** A script bundled with this skill reads credentials or secrets (gh auth, $AWS_*, ~/.aws, ~/.ssh, id_rsa, *_key, …).

**When this applies:** Any skill directory that bundles scripts reading credentials or secrets.

---

#### [CSKILL-001] Skill auto-approves unrestricted shell
**Severity:** high | **Confidence:** 0.90

**Directive:** Replace the wildcard with the specific command prefixes the skill needs, e.g.

**Why:** This skill's allowed-tools pre-approves unrestricted shell — a bare `Bash` grant or `Bash(*)`.

**When this applies:** Every skill — always verify allowed-tools before emitting.

---

#### [CSKILL-002] Skill runs shell during load (dynamic-context execution)
**Severity:** high | **Confidence:** 0.90

**Directive:** Move the work into the skill's instructions so Claude runs it as a normal, model-visible tool call, or into a reviewed bundled script invoked explicitly.

**Why:** This skill's body uses dynamic-context execution (the inline ! `cmd` form or a ```! fenced block).

**When this applies:** Any skill body that uses the backtick-exec or fenced-exec form.

---

#### [CSKILL-010] Bundled skill script performs network egress
**Severity:** high | **Confidence:** 0.70

**Directive:** Remove network calls from bundled scripts, or gate them behind an explicit, reviewed, user-approved step.

**Why:** A script bundled in this skill's directory makes outbound network calls (curl / wget / nc / …).

**When this applies:** Any skill directory that bundles scripts making outbound network calls.

---

#### [CSKILL-030] Bundled skill file contains a hardcoded secret
**Severity:** high | **Confidence:** 0.85

**Directive:** Remove the secret from the bundled file and rotate it immediately — assume it is compromised the moment it is committed.

**Why:** A file bundled with this skill contains a hardcoded secret — a recognizable credential format such as an AWS access key (AKIA…), a GitHub token (ghp_… / github_pat_…), a Slack or Google API token, an OpenAI-style key, or a private-key block.

**When this applies:** Any file bundled within the skill directory.

---

#### [CSKILL-050] Model-invocable skill grants side-effecting tools
**Severity:** high | **Confidence:** 0.80

**Directive:** Add `disable-model-invocation: true` so only the user can invoke this skill, or narrow allowed-tools to read-only tools (Read, Grep, Glob) or to scoped grants such as `Bash(git status:*)`, `Edit(docs/**)`, or `WebFetch(domain:example.com)`.

**Why:** Claude can auto-invoke this skill (disable-model-invocation is not set) and it pre-approves an unrestricted side-effecting or exfiltration-capable tool (Bash / Write / Edit / WebFetch / NotebookEdit).

**When this applies:** Any skill where disable-model-invocation is not set to true, and Any skill pre-approving Bash, Write, Edit, WebFetch, or NotebookEdit in allowed-tools with a genuinely unrestricted grant.

---

#### [CSKILL-080] Skill claims cryptographic operations
**Severity:** high | **Confidence:** 0.75

**Directive:** State the exact primitive the skill uses (e.g.

**Why:** This skill's name or description claims a cryptographic operation — encrypting, decrypting, signing, hashing, or certificate handling — but states no concrete primitive, key source, or algorithm bound.

**When this applies:** When the condition described by this rule is met.

---

#### [CSKILL-081] Skill processes sensitive data
**Severity:** high | **Confidence:** 0.70

**Directive:** Declare exactly which sensitive fields the skill reads, why it needs them, and how long it keeps them (ideally: not at all, beyond the current turn).

**Why:** This skill's body or description names a sensitive data class — passwords, tokens, SSNs, credit card numbers, PII, or private keys — in the same sentence as an operational verb (reads, stores, sends, processes, ...), so it claims to actually handle that data rather than merely mention it, without stating what it does with that data beyond naming it.

**When this applies:** When the condition described by this rule is met.

---

#### [CSKILL-082] Security-purpose skill grants side-effecting tools
**Severity:** high | **Confidence:** 0.80

**Directive:** Narrow allowed-tools to the read-only set an audit role actually needs (Read, Grep, Glob), or to scoped grants such as `Bash(git status:*)` or `Edit(docs/**)`.

**Why:** This skill's name claims a security, audit, pentest, or compliance purpose — a role that is expected to observe and report, not mutate — yet its allowed-tools pre-approve an unrestricted side-effecting or exfiltration-capable tool (Bash / Write / Edit / WebFetch / NotebookEdit).

**When this applies:** Any skill pre-approving Bash, Write, Edit, WebFetch, or NotebookEdit in allowed-tools with a genuinely unrestricted grant.

---

#### [CSKILL-020] Skill fetches untrusted external content
**Severity:** medium | **Confidence:** 0.70

**Directive:** Avoid fetching external content from a skill.

**Why:** This skill's body references an external http(s) URL.

**When this applies:** Any skill body that references an http(s) URL.

---

#### [CSKILL-040] Skill body contains prompt-injection markers
**Severity:** medium | **Confidence:** 0.60

**Directive:** Remove the override phrasing, the invisible/smuggling characters, or the encoded blob.

**Why:** This skill's body carries a prompt-injection signal — instruction-override phrasing ("ignore-previous-instructions"), invisible Unicode used to smuggle hidden text (zero-width characters, the Unicode Tags block U+E0000–E007F, or bidirectional overrides), or a long base64 blob that may conceal instructions.

**When this applies:** Any skill containing instruction-override phrasing, invisible Unicode, or encoded blobs.

---

#### [CSKILL-060] Skill description claims read-only but grants side-effecting tools
**Severity:** medium | **Confidence:** 0.50

**Directive:** Make the description match the grants: either narrow allowed-tools to the read-only set the description promises (Read, Grep, Glob) or to scoped grants such as `Bash(git status:*)` or `Edit(docs/**)`, or correct the description to disclose the side-effecting tools the skill actually uses.

**Why:** This skill's description claims it is read-only or side-effect-free, yet its allowed-tools pre-approve an unrestricted side-effecting or exfiltration- capable tool (Bash / Write / Edit / WebFetch / NotebookEdit).

**When this applies:** Any skill whose description claims read-only but grants side-effecting tools.

---

#### [CSKILL-084] Skill requests broad, unminimized data access
**Severity:** medium | **Confidence:** 0.60

**Directive:** Scope the skill's data access to the specific records or fields the task requires and filter at the source rather than fetching everything and filtering afterward.

**Why:** This skill's body uses broad-access phrasing — "all data", "entire", "every record", "full export" — that suggests it pulls more than the task at hand needs.

**When this applies:** When the condition described by this rule is met.

---

#### [CSKILL-086] Skill body implies data retention or logging
**Severity:** medium | **Confidence:** 0.65

**Directive:** State what is stored, where it's stored, how long it's retained, and the basis for retaining it (e.g.

**Why:** This skill's body uses language implying persistence — logging, storing, caching, or writing to a database — without stating a retention policy or user-consent basis for keeping that data.

**When this applies:** When the condition described by this rule is met.

---

#### [CSKILL-061] Skill allowed-tools list has duplicate tool references
**Severity:** low | **Confidence:** 0.70

**Directive:** Remove the duplicate entry so each tool appears once in allowed-tools.

**Why:** This skill's allowed-tools list references the same tool more than once (case-insensitive, whitespace-trimmed).

**When this applies:** Any skill with duplicate entries in allowed-tools.

---

#### [CSKILL-070] Skill is missing a description
**Severity:** low | **Confidence:** 0.90

**Directive:** Add a description: field to the skill's SKILL.md frontmatter that names what the skill does, what tools it uses, and any side effects it may have.

**Why:** This skill has no description.

**When this applies:** Every skill — description is required.

---

#### [CSKILL-071] Skill is bound to a single agent, reducing portability
**Severity:** low | **Confidence:** 0.60

**Directive:** Only bind a skill to a specific agent when that coupling is intentional and documented in the skill's description.

**Why:** This skill's frontmatter binds it to one specific agent (context: fork / agent: <name>) rather than leaving it agent-agnostic.

**When this applies:** Only when frontmatter includes a context: or agent: binding field.

---

#### [CSKILL-083] Skill body lacks error-handling guidance
**Severity:** low | **Confidence:** 0.50

**Directive:** Add a short paragraph naming what should happen if a step fails — retry once, fall back to an alternative, or stop and report the failure to the user — so the skill's guidance covers the unhappy path, not just the success case.

**Why:** This skill's body contains no error-handling language at all — no mention of what to do if a step errors, a tool call fails, or a retry is warranted.

**When this applies:** When the condition described by this rule is met.

---

#### [CSKILL-085] Skill description states no purpose
**Severity:** low | **Confidence:** 0.50

**Directive:** Add one clause to the description that states the skill's purpose, e.g.

**Why:** This skill's description contains no purpose phrasing — nothing framed as "to enable…", "used for…", or "designed to…" — so it states what the skill touches without stating why.

**When this applies:** When the condition described by this rule is met.

---

#### [CSKILL-087] Skill description is a placeholder
**Severity:** low | **Confidence:** 0.85

**Directive:** Replace the placeholder in the SKILL.md `description:` field with a real one-to-two-sentence summary of what the skill does, which tools it uses, and any side effects it may have.

**Why:** This skill's description is present, so it passes the empty-description check, but the text is a placeholder stub ("TODO", "TBD", "FIXME", "placeholder", "no description", "does stuff") rather than a real account of what the skill does.

**When this applies:** When the condition described by this rule is met.

---

## MCP dynamic code execution

### Tool Rules

---

#### [MCP-004] Network call has no timeout
**Severity:** high | **Confidence:** 0.85

**Directive:** Pass timeout= (typically 5-30 seconds depending on the endpoint) to every outbound call.

**Why:** An MCP tool handler that makes a network request without a timeout can hang indefinitely on a slow or unresponsive host.

**When this applies:** When defining a tool.

---

#### [MCP-005] Path parameter used in I/O without validation
**Severity:** high | **Confidence:** 0.70

**Directive:** Resolve the path with `Path(...).resolve()` and assert it sits under an allowed root before any I/O touches it.

**Why:** The tool accepts a path-like parameter and passes it to file or directory operations without resolving or sandboxing the path.

**When this applies:** When defining a tool.

---

#### [MCP-008] Tool fetches a caller-controlled URL (SSRF)
**Severity:** high | **Confidence:** 0.60

**Directive:** Constrain the destination to an explicit allow-list of hosts (or a single fixed base URL with only a path parameter), reject requests to private, loopback, and link-local IP ranges after DNS resolution, and disable redirects to addresses outside the allow-list.

**Why:** This MCP tool issues an HTTP request to a URL built from a parameter or an interpolated string rather than a fixed literal.

**When this applies:** When defining a tool.

---

#### [MCP-009] Tool body calls eval/exec/compile on dynamic input
**Severity:** high | **Confidence:** 0.85

**Directive:** Remove eval/exec/compile from the handler.

**Why:** The tool handler invokes Python's eval, exec, or compile.

**When this applies:** When defining a tool.

---

#### [MCP-010] Tool body spawns a subprocess
**Severity:** high | **Confidence:** 0.70

**Directive:** Prefer a typed library API over shelling out.

**Why:** The tool handler spawns an OS process (subprocess.*, os.system, os.popen, or os.spawn*).

**When this applies:** When defining a tool.

---

#### [MCP-012] TypeScript MCP tool spawns a subprocess
**Severity:** high | **Confidence:** 0.70

**Directive:** Avoid shelling out from MCP tools.

**Why:** This MCP tool handler, authored in TypeScript, invokes a child-process API (child_process exec/execSync/spawn).

**When this applies:** When defining a tool.

---

#### [MCP-013] TypeScript MCP tool fetches a caller-controlled URL (SSRF)
**Severity:** high | **Confidence:** 0.60

**Directive:** Validate the destination before fetching: resolve the host and reject private, loopback, and link-local ranges; prefer an allowlist of exact hosts, or resolve an opaque ID against a server-side registry and build the URL from the trusted entry.

**Why:** This MCP tool handler, authored in TypeScript, issues an HTTP request to a URL built from a tool argument or an interpolated string rather than a fixed literal.

**When this applies:** When defining a tool.

---

#### [MCP-014] TypeScript MCP tool evaluates dynamic code
**Severity:** high | **Confidence:** 0.90

**Directive:** Remove `eval`/`new Function`.

**Why:** This MCP tool handler, authored in TypeScript, calls `eval()` or constructs a `new Function(...)`.

**When this applies:** When defining a tool.

---

#### [MCP-024] TypeScript MCP tool HTTP call has no timeout
**Severity:** high | **Confidence:** 0.60

**Directive:** Attach a deadline.

**Why:** This TypeScript MCP tool handler makes an outbound HTTP call (fetch / axios / got / undici) with no deadline — no signal, timeout, or abortSignal option.

**When this applies:** When defining a tool.

---

#### [MCP-030] Side-effecting tool lets the model choose the recipient or amount with no visible bound
**Severity:** high | **Confidence:** 0.50

**Directive:** Do not let the model supply the recipient or the amount unconstrained.

**Why:** This MCP tool's name signals a send/notify or a refund/charge/pay side effect, and it takes a free-form recipient or amount parameter with no visible bound: no allow-list constraining who the message reaches, and no cap, range, or enum constraining how much money moves.

**When this applies:** When defining a tool.

---

#### [MCP-002] Tool has no type-annotated parameters
**Severity:** medium | **Confidence:** 0.85

**Directive:** Add type hints to every parameter (def search(query: str, limit: int = 10) -> dict).

**Why:** MCP derives a tool's input JSON schema from the handler's Python type annotations.

**When this applies:** When defining a tool.

---

#### [MCP-007] Mutating tool has no idempotency key
**Severity:** medium | **Confidence:** 0.55

**Directive:** Accept an idempotency key parameter (idempotency_key / request_id) and de-duplicate server-side so a retried call is a no-op after the first success.

**Why:** The tool name signals a side effect (create/send/refund/…).

**When this applies:** When defining a tool.

---

#### [MCP-023] MCP tool prints to stdout, corrupting a stdio transport
**Severity:** medium | **Confidence:** 0.70

**Directive:** Remove the print().

**Why:** The tool handler calls print(), which writes to the process's stdout.

**When this applies:** When defining a tool.

---

#### [MCP-031] MCP tool fetches an allow-listed host without pinning https://
**Severity:** medium | **Confidence:** 0.60

**Directive:** Pin the scheme: build the URL from a literal "https://" prefix, or reject any URL whose urlparse(url).scheme is not "https" before the request.

**Why:** This MCP tool checks the destination host against an allow-list (so MCP-008 stays silent) but builds its outbound request URL without a literal https:// prefix and without checking the URL scheme.

**When this applies:** When defining a tool.

---

#### [MCP-032] TypeScript MCP tool fetches an allow-listed host without pinning https://
**Severity:** medium | **Confidence:** 0.60

**Directive:** Pin the scheme: build the URL from a literal `https://` prefix, or reject any URL whose `new URL(x).protocol` is not "https:" before fetching.

**Why:** This TypeScript MCP tool checks the destination host against an allow-list (so MCP-013 stays silent) but passes fetch/axios/got a URL that does not begin with a literal https:// prefix and checks no URL protocol.

**When this applies:** When defining a tool.

---

#### [MCP-034] Tool declares a claim argument that nothing checks
**Severity:** medium | **Confidence:** 0.60

**Directive:** Check the claim before the effect, in the tool handler, before any side effect, or one shared decorator every handler goes through: remove it from the arguments, validate it, resolve the facts server-side (tool registry, the task's authorized intent, policy, route), and compare.

**Why:** This MCP tool handler (`@mcp.tool()` / `@server.tool()`) takes a `claim` argument, so the model fills in a typed claim about the call, but the tool body never reads it and the tool has no decorator besides the one that registers it, so no mediation step visible here can check it.

**When this applies:** When defining a tool.

---

#### [MCP-035] Tool's claim argument is only written out or returned, never checked
**Severity:** medium | **Confidence:** 0.55

**Directive:** Pass the claim to a verifier before the effect and act on its outcome: deny on a conflict between the claim and the arguments, task or policy, review on anything unresolved or high risk, and run the tool only on allow.

**Why:** This MCP tool handler (`@mcp.tool()` / `@server.tool()`) reads its `claim` argument, but every read only writes it out — to print, a logger, json.dumps, str or repr — or puts it in the return value.

**When this applies:** When defining a tool.

---

#### [MCP-001] Tool has no description
**Severity:** low | **Confidence:** 0.90

**Directive:** Add a one-line docstring (or a description= argument on the registration) stating what the tool does and when a model should call it.

**Why:** An MCP server publishes each tool's description to connecting clients as the text a model uses to decide whether to call the tool.

**When this applies:** When defining a tool.

---

#### [MCP-003] Ambiguous tool name
**Severity:** low | **Confidence:** 0.85

**Directive:** Rename to a verb-object form, e.g.

**Why:** Tool names like `process`, `handle`, or `run` give a connecting model no signal about intent.

**When this applies:** When defining a tool.

---

#### [MCP-006] Tool raises exceptions without a structured error contract
**Severity:** low | **Confidence:** 0.60

**Directive:** Catch known failure modes and return a structured result the model can branch on (e.g.

**Why:** When an MCP tool handler raises, the runtime surfaces the exception to the connecting client as an opaque error.

**When this applies:** When defining a tool.

---

#### [MCP-011] TypeScript MCP tool has no description
**Severity:** low | **Confidence:** 0.85

**Directive:** Provide a concise `description` string in the registration config stating what the tool does and when a model should call it.

**Why:** An MCP server authored in TypeScript registers each tool with `server.registerTool(name, {description, inputSchema}, handler)`.

**When this applies:** When defining a tool.

---

#### [MCP-015] Go MCP tool has no description
**Severity:** low | **Confidence:** 0.85

**Directive:** Add `mcp.WithDescription("...")` to the `mcp.NewTool(...)` call (mark3labs/ mcp-go), or set the `Description` field on the `mcp.Tool` value (the official go-sdk), stating what the tool does and when a model should call it.

**Why:** An MCP server authored in Go advertises each tool's description to connecting clients as the text a model uses to decide whether to call it.

**When this applies:** When defining a tool.

---

#### [MCP-016] Ambiguous Go MCP tool name
**Severity:** low | **Confidence:** 0.85

**Directive:** Rename to a verb-object form, e.g.

**Why:** A Go MCP tool's name is the first argument to `mcp.NewTool(...)` (or the `Name` field of an `mcp.Tool`).

**When this applies:** When defining a tool.

---

#### [MCP-017] C# MCP tool has no description
**Severity:** low | **Confidence:** 0.85

**Directive:** Add a `[Description("...")]` attribute (System.ComponentModel) to the `[McpServerTool]` method, stating what the tool does and when a model should call it.

**Why:** A C# MCP server advertises each tool's description to connecting clients as the text a model uses to decide whether to call it.

**When this applies:** When defining a tool.

---

#### [MCP-018] Ambiguous C# MCP tool name
**Severity:** low | **Confidence:** 0.85

**Directive:** Rename the method (or set `[McpServerTool(Name = "...")]`) to a verb-object form, e.g.

**Why:** A C# MCP tool's name defaults to the `[McpServerTool]` method name.

**When this applies:** When defining a tool.

---

#### [MCP-019] PHP MCP tool has no description
**Severity:** low | **Confidence:** 0.85

**Directive:** Add a `description:` argument to the `#[McpTool]` attribute, e.g.

**Why:** A PHP MCP server advertises each tool's description to connecting clients as the text a model uses to decide whether to call it.

**When this applies:** When defining a tool.

---

#### [MCP-020] Ambiguous PHP MCP tool name
**Severity:** low | **Confidence:** 0.85

**Directive:** Rename the method (or set the `#[McpTool]` `name:` argument) to a verb-object form, e.g.

**Why:** A PHP MCP tool's name is the `#[McpTool]` attribute's `name:` argument, or the method name when that argument is omitted.

**When this applies:** When defining a tool.

---

#### [MCP-021] Rust MCP tool has no description
**Severity:** low | **Confidence:** 0.85

**Directive:** Add a `description = "..."` argument to the `#[tool(...)]` attribute, or a `///` doc comment on the method, stating what the tool does and when a model should call it.

**Why:** A Rust MCP server (the official rmcp crate) advertises each tool's description to connecting clients as the text a model uses to decide whether to call it.

**When this applies:** When defining a tool.

---

#### [MCP-022] Ambiguous Rust MCP tool name
**Severity:** low | **Confidence:** 0.85

**Directive:** Rename the method (or set the `#[tool]` `name = "..."` argument) to a verb-object form, e.g.

**Why:** A Rust MCP tool's name is the `#[tool]` attribute's `name = "..."` argument, or the method name when that argument is omitted.

**When this applies:** When defining a tool.

---

#### [MCP-025] MCP tool description is a placeholder
**Severity:** low | **Confidence:** 0.85

**Directive:** Replace the placeholder with a real description covering what the tool does, what it returns, and when a model should call it rather than a neighboring tool on this server.

**Why:** The docstring passes the MCP-001 has-a-description check but carries a placeholder marker instead of real content.

**When this applies:** When defining a tool.

---

#### [MCP-026] MCP tool description is too short to guide model selection
**Severity:** low | **Confidence:** 0.80

**Directive:** Expand the description to at least a full sentence covering inputs, outputs, and the situation in which this tool should be used over the alternatives.

**Why:** A description under 40 characters is rarely enough to convey what a tool does, what it returns, and when to call it rather than a similarly named neighbor.

**When this applies:** When defining a tool.

---

#### [MCP-027] TypeScript MCP tool description is a placeholder
**Severity:** low | **Confidence:** 0.85

**Directive:** Replace the placeholder with a real description covering what the tool does, what it returns, and when a model should call it rather than a neighboring tool on this server.

**Why:** The tool registration sets a description, so it passes MCP-011, but the string is a placeholder rather than real content.

**When this applies:** When defining a tool.

---

#### [MCP-028] TypeScript MCP tool description is too short to guide model selection
**Severity:** low | **Confidence:** 0.80

**Directive:** Expand the description to at least a full sentence covering inputs, outputs, and the situation in which this tool should be used over the alternatives.

**Why:** A description under 40 characters is rarely enough to convey what a tool does, what it returns, and when to call it rather than a similarly named neighbor.

**When this applies:** When defining a tool.

---

#### [MCP-029] TypeScript MCP tool writes to the filesystem
**Severity:** low | **Confidence:** 0.50

**Directive:** Confine writes to a dedicated working directory: resolve the final path, verify it stays under that root before writing, and reject absolute paths and any input containing "..".

**Why:** This TypeScript MCP tool handler writes to the filesystem.

**When this applies:** When defining a tool.

---

#### [MCP-033] Side-effecting tool takes no structured claim for its calls
**Severity:** info | **Confidence:** 0.50

**Directive:** Add a reserved `claim` parameter typed as a ToolInvocationExplanation (schema_version, intent_class, selected_tool, expected_effect, risk_tier, uncertainty, untrusted_context, evidence_refs; an optional summary that no check reads) and check it in a mediation step before the effect, in the tool handler, before any side effect, or one shared decorator every handler goes through.

**Why:** This MCP tool handler (`@mcp.tool()` / `@server.tool()`) has a side effect — it shells out, executes code, writes a file, or its name says it sends, deletes, pays, deploys, creates or updates — and it takes no `claim` argument, so nothing about why the model chose this call or what it expects the call to do reaches the code that runs it.

**When this applies:** When defining a tool.

### Repo Rules

---

#### [MCP-201] MCP server project ships no agent-guidance doc (AGENTS.md/CLAUDE.md)
**Severity:** low | **Confidence:** 0.90

**Directive:** Add an AGENTS.md at the repo root (a CLAUDE.md also satisfies this rule).

**Why:** The project implements an MCP server in code but ships no agent-guidance doc (AGENTS.md or CLAUDE.md) at the repo root.

**When this applies:** For any repo using this SDK.

---

#### [MCP-202] MCP server wires no observability
**Severity:** low | **Confidence:** 0.75

**Directive:** Install an OpenTelemetry tracer provider with an OTLP exporter at server start.

**Why:** This project implements an MCP server but wires no observability backend.

**When this applies:** For any repo using this SDK.

