---
name: delegate
description: >-
  Delegate work to Akuity's native AI agents through the Akuity MCP endpoints:
  start or continue a normal context-aware conversation with the Deployment
  Agent,
  hand a degraded application to the On-call Agent and relay its proposed
  remediation for human approval, request a promotion risk verdict from the
  Promotion Advisor before releasing, and read back work a native agent
  performed. Use when asked to ask the Deployment Agent a question, continue an
  Akuity Intelligence conversation, hand an incident to Akuity, check whether
  a promotion is safe, or find out what an Akuity agent did.
---

You are delegating work to Akuity-native AI agents over MCP and reporting their results back to a human. You are the *client* here: the native agent has the selected runbooks and resource context. You start its work, follow it, and carry decisions between it and the human.

The names and JSON shapes in this skill are illustrations from real runs, not contracts. Your MCP client may prefix tool names (e.g. `mcp__<server>__create_intelligence_conversation`), and field names, enum spellings, and argument shapes may drift between platform versions — trust the advertised tool list and schemas first, and match what you see here by purpose. What does not drift is the discipline: what needs a human decision, and what counts as evidence.

## The one rule that matters

**Never approve a proposed action on your own authority.** Whether the native agent stops before a state change is decided by the organization's Intelligence **tool policies**, not by the kind of conversation: a tool covered by a `require_approval` policy (optionally narrowed to apps, projects, or argument patterns) stops as a pending step; a tool covered by a `deny` policy is auto-rejected with the reason *"Denied by organization policy."*; a mutable tool with **no** matching policy runs immediately — in incidents and normal conversations alike (a `k8s-patch-resource` step that goes straight to `SUCCEEDED`, with no pending step). When it does stop, your job is to show the human what it wants to do — tool name, arguments, and why — and only answer once they have decided. Relay; do not decide. When it did not stop, you are not the gate: report what it did, with the evidence described in Scenario A, and say plainly that the organization's policy, not you, let it run.

A proposal reaches you in one of **three** shapes, and all of them require a human decision:

| Shape | How it looks | How you answer |
| --- | --- | --- |
| **Pending step** | a step with `status: "AI_CONVERSATION_STEP_STATUS_PENDING_APPROVAL"` | `set_intelligence_tool_approval` (step 4a) |
| **Suggested change** | a `suggestedChanges` entry with `applied: false`, usually alongside a plain-English question like *"Please confirm if you would like this fix to be implemented now."* | prose reply via `create_intelligence_message` — read step 4b first |
| **Prose question** | an assistant message whose `content` ends in a question to the human — *"Would you like to proceed with a manual sync?"* — with **no** pending step and **no** `suggestedChanges` entry | prose reply via `create_intelligence_message`; the agent then calls the tool, which surfaces as a pending step |

Only the first carries a `toolCallId`. The other two have no approval handle at all, which makes them easy to mistake for commentary. They are not commentary: `applied: false` plus a question addressed to a human, or a bare question about a state change, is an action the agent has **withheld** pending consent. Show it to the human exactly as you would a pending step — after one check: scan `messages[].steps[]` first, and if a `SUCCEEDED` mutating step (`k8s-patch-resource`) already covers that same change, the change has landed and `applied: false` is stale narration. Report it as applied; do not ask the human to apply it again. The step always outranks the flag.

## Before you start

1. **The Akuity MCP server must be connected** — the platform endpoint this plugin wires, or an instance's own endpoint (see `../../references/endpoints-and-auth.md`, relative to this file). Confirm the delegation tools are present before planning work — if `create_intelligence_conversation` is not in your tool list under any prefix, stop and tell the human the server is not connected or the organization lacks the entitlement.
2. **Know which surface you are on** — it changes the arguments (table below).
3. **Resolve the organization on the platform surface.** Call `list_organizations` to map the human's organization name to its id, and confirm the target when more than one organization is available. Instance surfaces need no organization id.
4. **Choose contexts for the requested depth.** Contexts define both the conversation's resource scope and the capabilities available to the Deployment Agent. Before creating or changing a normal conversation, read [`references/context-selection.md`](references/context-selection.md) and select the smallest exact scope that supports the request.

### Surfaces

| Surface | Host | Tenancy arguments |
| --- | --- | --- |
| Platform | the Akuity platform host | You **must** pass `organizationId`, plus `instanceId` / `kargoInstanceId` where relevant |
| Per-instance | an Argo CD or Kargo instance's own hostname | The server injects them from the host. Passing them is an **error** (`-32602`, "not accepted on this endpoint") and they are absent from the advertised schemas |

On an instance host the dispatcher is also pinned to the product: `incident` exists only on Argo CD hosts, `kargoPromotionAnalysis` only on Kargo hosts.

## Tools

| Tool | Use |
| --- | --- |
| `create_intelligence_conversation` | Start work with the On-call Agent, Promotion Advisor, or Deployment Agent |
| `create_intelligence_message` | Send a follow-up while preserving its conversation scope (see below). **Returns empty by design** — the reply arrives asynchronously; you must poll |
| `get_intelligence_conversation` | Full state: messages, steps, incident/promotion metadata, runbooks. Use to follow progress and read results |
| `list_intelligence_conversations` | Cheap listing/filtering. Returns each conversation with an **empty `messages` array** — use it for status sweeps, not timelines |
| `set_intelligence_tool_approval` | Approve or reject one proposed tool call, after the human decides |
| `list_intelligence_tools` | What the native agent can do in this org, with a `mutable` flag per tool |

There is also a readable resource per conversation: `akuity://orgs/{organization_id}/ai/conversations/{id}` on the platform surface and `akuity://ai/conversations/{id}` on instance surfaces. Use it when attaching a conversation as context. Follow active work by polling `get_intelligence_conversation`.

### Preserve conversation scope on every follow-up

Although `create_intelligence_message` appends one user message, its `contexts` and `runbooks` arguments are the complete replacement sets for the conversation, not attachments to that message. With the current API, omitting either repeated field sends an empty list and clears the stored set.

Before every `create_intelligence_message` call:

1. Read the full conversation with `get_intelligence_conversation`. Do not use a list result: list responses omit messages and are not the source of truth for a follow-up.
2. Pass the returned `contexts` and `runbooks` back unchanged as tool arguments. Copy the advertised objects exactly; do not reconstruct, shorten, or paste them into `content`.
3. If the human explicitly changes contexts or runbooks, send the complete desired replacement sets — not only the additions or removals.

If the full read fails, do not send the message: there is no safe conversation scope to preserve. After the empty send response, poll the full conversation as usual and make sure its contexts and runbooks still match the intended sets.

## Scenario A — hand a degraded app to the On-call Agent

**1. Open the incident.** Exactly one context is required, and it must be an application or a namespace. Anything else is rejected with `InvalidArgument`.

```jsonc
// create_intelligence_conversation
{
  "organizationId": "<org>",          // omit on an instance host
  "instanceId": "<argocd-instance>",  // omit on an instance host; required otherwise
  "incident": true,
  "contexts": [
    { "argoCdApp": { "instanceId": "<argocd-instance>", "name": "guestbook-prod", "namespace": "argocd" } }
    // or: { "k8sNamespace": { "instanceId": "<inst>", "clusterId": "<cluster>", "name": "team-a" } }
  ]
}
```

The response carries the conversation, including `id` — keep it. The On-call Agent starts automatically; the platform infers which runbooks apply and lists them under `runbooks` (empty when the instance has none configured). Tell the human the incident number (`incident.incidentNumber`) so they can find it in the portal.

**2. Follow it.** Poll `get_intelligence_conversation` every 5–15 seconds. Investigations can run for several minutes, so partial progress is normal and worth surfacing.

Read, in order of usefulness:

- `processing` — `false` means the agent is done for now
- `messages[].steps[]` — what it actually did: `functionName`, `name`, `status`, `mcpServerName`
- the last `assistant` message's `content` — its findings; `thinkingProcess` if the human wants reasoning
- `incident` — `summary` first, then `rootCause` / `resolution` / `resolvedAt`. These keys are always present but filled in only as the agent determines them, so treat an empty string as "not concluded yet", not as "no root cause"
- `executionPlan.tasks[]` — the plan it is working through, when present

**`processing: false` with no pending step is not automatically success.** Before reporting an investigation as finished, check the steps and `suggestedChanges` on the last assistant messages.

- A genuinely read-only investigation — inspect the app, read events, pull logs — ends with no pending step *and* no suggested change. That is success: report the findings.
- An agent that diagnosed the problem and wants to fix it ends the same way *except* for a `suggestedChanges` entry with `applied: false` and a question aimed at a human. That is a proposal waiting on a decision, not a conclusion. Reporting it as "done" strands the incident.
- An agent that asks in plain prose — *"Would you like to proceed with a manual sync?"* — with **no** pending step and **no** `suggestedChanges` entry is waiting just the same. Nothing in the structured fields marks it; only the last assistant message's `content` does. Read it before calling the investigation done.
- A conversation in which no `require_approval` policy covered the tool ends with a `k8s-patch-resource` step at `SUCCEEDED` and no approval anywhere. That is a change that has landed on the live cluster: report it as applied, with the patch and the affected resource, and check `incident.resolvedAt` for whether the agent also closed the incident.

Escalate to step 3 for any of the three proposal shapes: a pending step, an unapplied suggested change, or a prose question.

**Stop conditions.** If `processingError` is set, report it and stop: `processingErrorIsFatal: true` means retrying is pointless (token limit, processing limit, or an invalid tool configuration the administrator must fix). A non-fatal error may clear on its own.

**3. Relay a proposed remediation.** Present it verbatim; never summarize away the specifics.

For a **pending step**:

- `functionName` and `functionArguments` — exactly what will happen
- the assistant's reasoning around it
- the affected app/namespace/cluster from the incident metadata

For a **suggested change**:

- the `patch` — exactly what will happen
- the `old` → `new` diff, at minimum the fields that actually differ
- the assistant message's `content` and `thinkingProcess` — its reasoning, and its own caveats (pod restarts, availability impact)
- `context.argoCdApp` — which app and instance it targets

For a **prose question**:

- the question itself, quoted, and the action it describes (a sync, a rollback, a patch) — there is no `functionArguments` or `patch` yet, so say so
- the assistant message's `content` and `thinkingProcess` — its diagnosis and reasoning
- the affected app/namespace/cluster from the incident metadata or the conversation's `contexts`

In every case, say plainly whether the change lands on the **live cluster** or in Git. A live patch to an auto-synced app gets reverted as drift, so approving it buys minutes and the durable fix belongs in the manifest source. The human needs that to decide well.

**4a. Answer a pending step.** From that step: `id` is the `toolCallId`, `parentMessageId` is the `messageId`.

```jsonc
// set_intelligence_tool_approval
{
  "organizationId": "<org>",     // omit on an instance host
  "conversationId": "<conv>",
  "messageId": "<step.parentMessageId>",
  "toolCallId": "<step.id>",
  "status": "AI_TOOL_APPROVAL_STATUS_APPROVED"   // or AI_TOOL_APPROVAL_STATUS_REJECTED
}
```

Then keep polling: an approval releases the action and the agent continues.

**4b. Answer a suggested change or a prose question.** `set_intelligence_tool_approval` cannot help because there is no `toolCallId`, and the MCP API has no separate call that applies a suggested change. After the human decides, use the scope-preserving `create_intelligence_message` procedure above and name the decision and proposed change precisely in `content`. The reply asks the native agent to take the next action and may cause it to call a mutating tool immediately when no `require_approval` policy covers that tool. Send it only after the human has decided.

Then poll until you see a pending step, a terminal step for the mutating call, or a clear refusal or error. If the agent only repeats the proposal without taking the requested next step, stop and report that it did not act.

**`suggestedChanges[].applied` is not authoritative state.** It is part of the agent's response, so confirm the result from a terminal mutating step or independent live state before reporting that a change landed.

What actually counts as evidence, in order:

1. **A step for the mutating call** — `functionName: "k8s-patch-resource"` with a terminal `SUCCEEDED`. `k8s-verify-resource-patch` does **not** count: it is a dry run that succeeds whether or not anything was applied.
2. **Independent ground truth** — the live resource's own fields, the app's health. Prefer this for anything that matters.

If no mutating step exists and live state does not confirm the change, do not report it as applied. Do not repeat an approval unless a new pending step requires a new decision.

Stop polling when repeated progress checks show no new resource reads, pending action, or terminal result. Tell the human that the conversation is not making progress and direct its owner to manage it in the Akuity Portal.

**Approval is owner-only.** Only the principal that created the conversation can approve its actions. If you started it, you can relay the human's decision; if somebody else started it, say so rather than trying.

## Scenario B — get a promotion verdict from the Promotion Advisor

Do this *before* triggering a production promotion. The verdict informs the human's decision — it is never itself an authorization to promote.

```jsonc
// create_intelligence_conversation
{
  "organizationId": "<org>",             // omit on an instance host
  "kargoInstanceId": "<kargo-instance>", // omit on an instance host; required otherwise
  "kargoPromotionAnalysis": {
    "project": "<kargo-project>",
    "stage": "<target-stage>",
    "freight": "<freight-id>",   // required — an empty value is rejected with InvalidArgument; get the id from the freight list tool
    "reAnalyze": false           // true to redo an analysis that already ran
  }
}
```

Poll `get_intelligence_conversation` until `processing` is `false`, then read `promotionAnalysis`:

- `decision` and `riskLevel` — the verdict
- `summary` — the reasoning to quote to the human
- `currentFreight` vs `freight` — what is being replaced
- `freightStageStatuses[]` — where this freight already ran: `stage`, `verified`, `rolledBack`, `exists`
- `commitDiffUrl` — the change under review

**Act on it:**

- Low risk / proceed → report the verdict and stop there, unless the human's original request explicitly said to promote when safe. "Is this promotion safe?" authorizes analysis, not the promotion — promote only on their go-ahead.
- Elevated risk → **warn and wait.** Quote `summary` and the specific signals (unverified in a prior stage, a rollback in history) and let the human choose.
- High risk, or `rolledBack: true` on a prior stage → **hold.** Do not promote; report why.
- No verdict (still processing, or `processingError`) → treat as "no answer", not as approval. Hold and say so.

Never promote on a verdict you did not actually read back.

## Scenario C — have a normal conversation with the Deployment Agent

Use the Deployment Agent for Kubernetes, Argo CD, Kargo, or Akuity questions and operational work that is neither an incident investigation nor a promotion verdict.

- Read [`references/context-selection.md`](references/context-selection.md) before choosing the initial contexts or changing them later. An instance context is useful for inventory and fleet questions, but it is not an umbrella for app-specific details or actions.
- If the human names an existing conversation, read it with `get_intelligence_conversation` and continue it with the scope-preserving follow-up procedure. Do not create a replacement conversation.
- Otherwise call `create_intelligence_conversation` with the surface's tenancy arguments and the exact contexts needed for the work, but omit both `incident` and `kargoPromotionAnalysis`. Contexts are optional only for general questions that need no environment data. On the platform surface, the conversation still needs either `instanceId` or `kargoInstanceId` even when it has no contexts.
- The create response gives you the conversation id; it does not send the human's question. Send the question with `create_intelligence_message`, preserving the returned contexts and runbooks, then poll the full conversation for the reply.

A normal conversation does not weaken the approval rule. If the native agent proposes a mutable action, relay it to the human and wait for their decision just as you would during an incident.

## Scenario D — read back what a native agent did

You can query conversations visible to the current principal, including platform-endpoint conversations created by other users against an instance you can access. Direct-endpoint conversations are private to their creator. Find conversations with `list_intelligence_conversations` (cheap, with no messages), then read one with `get_intelligence_conversation` for the full record. The filter arguments and a where-to-look table for common questions are in `references/reading-conversations.md`; read it before answering "what did the Akuity agent do" questions. The evidence rule from Scenario A applies unchanged when reading history: confirm a fix with a succeeded mutating step or live state, not `suggestedChanges[].applied` or the agent's summary alone.

## Pitfalls

- **`create_intelligence_message` returns nothing.** That is by design. Poll `get_intelligence_conversation`; do not report "no reply" until `processing` is `false`.
- **Conversation payloads get large** — full tool outputs are included. Prefer `list_intelligence_conversations` for status, and when reading a conversation, extract the fields you need instead of dumping the whole object into your reply.
- **Do not use `mutable` as the only safety check.** Read the proposed `functionName`, `functionArguments`, or patch yourself. Treat a `PENDING_APPROVAL` step, an unapplied `suggestedChanges` entry, or a prose request for permission as a decision for the human.
- **An unapplied `suggestedChanges` entry is a pending decision, not a finished thought.** It has no `toolCallId` and produces no `PENDING_APPROVAL` step, so polling `processing` and scanning step statuses will never surface it as "waiting". Read `suggestedChanges` before calling an investigation done.
- **Approving in prose can silently no-op.** The agent may assert in `incident.summary` / `incident.rootCause` that it applied a patch it never called a tool to apply, leaving the incident record claiming a resolution that never happened. Verify against a `k8s-patch-resource` step and against live state before you report a fix to anyone — and remember `k8s-verify-resource-patch` is a dry run that proves nothing was applied.
- **`suggestedChanges[].applied` is not sufficient confirmation.** Check terminal mutation steps and live state before reporting whether the change landed. When the fields disagree, the step result and live state take priority.
- **Per-step `summary` text is explanatory, not authoritative state.** Read `functionName`, `functionArguments`, and `status`; quote `summary` only as the agent's interpretation.
- **Akuity Intelligence work is not gated by per-instance MCP access.** A native-agent investigation can continue when direct MCP reads or actions for the same instance are disabled. The instance MCP setting gates your direct MCP access, not the Intelligence conversation.
- **An Argo CD action approval can be refused** with `PermissionDenied: no user credentials available to verify ArgoCD permissions`. Approve that step in the portal UI or from the instance's own endpoint, and report the permission error to the human.
- **Visibility is per-principal.** A private conversation created by someone else — or by a different credential than yours — reads back as not-found. That is enforcement, not a bug; report it plainly.
- **An incident needs exactly one context**, and it must be `argoCdApp` or `k8sNamespace`. Two contexts, zero contexts, or a cluster/instance context all fail with `InvalidArgument`.
- **An instance context is not app-detail scope.** It supports application inventory, but not per-app details or actions. For a detailed summary of several apps, resolve the apps first and attach the exact set of `argoCdApp` contexts.
- **On instance hosts, do not send tenancy arguments.** The `-32602` rejection names the offending argument; drop it and retry rather than guessing.
- **Don't re-open an investigation that already exists.** Check `list_intelligence_conversations` with `incidentOnly` and the application/namespace filters first; the platform groups incidents, and a duplicate splits the timeline.
