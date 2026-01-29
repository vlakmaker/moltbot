# Production spec: Command Approval Workflow

This spec covers the production approval workflow for executing sensitive actions (starting with `exec`), with Discord-first UX but generalizable to other channels/tools.

## Goals
- Safe execution of sensitive actions behind explicit human approval.
- Low-friction UX: user knows exactly what will run, where, and what happened.
- Deterministic routing: no ambiguous recipients; DM vs channel behavior is predictable.
- Auditable: every request/decision/result is logged with enough context.
- Extensible: can be reused for other “requires confirmation” actions (send message, config patch, etc.).

## Non-goals
- Full sandboxing of arbitrary code (tool layer handles allowlists/timeouts).
- Fully interactive terminal streaming (nice-to-have).

---

## Core UX requirements (additions)

### A) Canonical approval request ID (ID unification)
Problem: multiple IDs exist (tool exec approval id, gateway UUID, Discord interaction ids), and they don’t match what the user sees.

Requirement:
- Choose **one canonical ApprovalRequest id (UUID)** and use it everywhere:
  - DM approval card
  - origin interaction “Approval required…” response
  - logs and audit trail
  - internal state storage keys
- Any other IDs are treated as **internal** and not shown as the primary reference.

UI:
- Display both:
  - `Request: <uuid>` (full)
  - `Short: <uuid[0:8]>` (primary for humans)
- Provide a “Copy request id” affordance where possible.

### B) Mandatory feedback loop (no “did it run?” ambiguity)
Problem: after approving, the user has to ask for status.

Requirement:
- When an approval decision is made, send an immediate acknowledgement:
  - DM: “Approved (Allow once). Executing…”
- On completion, always send a completion message to:
  1) the original requester surface (slash command result / channel thread), and
  2) the approver DM.

Completion message must include:
- exit code
- duration
- output snippet (first N lines / N chars)
- link/attachment for full output when truncated

Failure message must include:
- error summary
- actionable next step (Retry button; or “Fix DM privacy” if DM failed)

### C) Delivery fallback behavior
If DM approval UI can’t be delivered:
- Origin interaction must say: “Could not DM approver” + reason + next action.
- Provide one of:
  - fallback approve/deny buttons in-channel (config-gated, default OFF), or
  - manual `/approve <requestId>` command

---

## Entities & Data model

### ApprovalRequest
- `id` (uuid; canonical)
- `createdAt`, `expiresAt`
- `status`: `pending | approved | denied | expired | executing | executed | failed`
- `reason`: e.g. `ask:on-miss`, `requires-elevation`, `policy:external-side-effect`
- `initiator`: `{ channel, accountId, userId(canonical), originContext }`
- `action`: `{ kind, intentSummary, payload(structured) }`
- `preview`: `{ exactCommandLine, riskNotes[] }`
- `policy`: `{ approvers[], delivery(dmPreferred, allowInChannelButtons), defaultEphemeral }`
- `trace`: `{ sessionKey, toolCallId }`

### ApprovalDecision
- `requestId`
- `deciderUserId`
- `decision`: `allow_once | allow_for | allow_always | deny`
- `allowForMs?`

### ApprovalGrant (policy cache)
- `principal`
- `actionKind`
- constraints: cmd match / cwd / host
- expiry

---

## Canonical recipient routing (no ambiguity)

All stored config/state MUST use canonical recipient strings:
- Discord DM: `user:<discordUserId>`
- Discord channel: `channel:<discordChannelId>`

Validation:
- Startup validation fails (hard error) if bare numeric Discord IDs appear in fields that expect canonical recipients.

---

## End-to-end flow

1) Parse and normalize intent → structured payload.
2) Evaluate grants/policy; if no grant, create ApprovalRequest.
3) Persist request.
4) Notify approver(s) via DM with buttons.
5) Ack requester in origin interaction (“Approval requested: <short-id>”).
6) On decision:
   - validate authz + request state
   - record decision
   - if allow: execute
7) Send result to requester and approver.

---

## Logging & audit
Structured events (minimum):
- `approval.request.created`
- `approval.request.dm_sent` / `approval.request.dm_failed`
- `approval.decision.recorded`
- `approval.execution.started`
- `approval.execution.finished` / `approval.execution.failed`

Persist:
- request store (for recovery across restart)
- append-only audit log (optional)

---

## Acceptance criteria
- Approving a request always produces visible “executing…” then “done” feedback.
- Request id shown to user matches the id in logs/state.
- No ambiguous recipient errors in normal operation.
- If DM delivery fails, user gets a clear next step and fallback mechanism.
