# Reading back native-agent conversations

Platform-endpoint conversations are visible to users who can access their target instance. Direct-endpoint conversations are private to their creator. Query only the conversations visible to the current principal.

## Finding conversations

`list_intelligence_conversations` returns each conversation with an **empty `messages` array**, which makes it cheap. Filters:

- `incidentOnly: true` with `incidentStatus: "INCIDENT_STATUS_UNRESOLVED"` — open incidents
- `scheduledTaskOnly: true` with `scheduledTaskStatus: "SCHEDULED_TASK_STATUS_ACTIVE"` (or `_SUSPENDED`) — conversations driven by a scheduled task
- `instanceId` / `kargoInstanceId` — scope to the conversations opened against one Argo CD or Kargo instance
- `application`, `namespace`, `clusterId`, `kargoProject` — scope to a resource
- `titleContains`, `startTime` / `endTime` (RFC3339, on last update time), `limit` / `offset`
- `hasUserInteractions: true` — human-involved conversations only, excluding purely automated investigations

## Reading one conversation

`get_intelligence_conversation` returns the full record. Where to look, by question:

| Question | Where to look |
| --- | --- |
| What happened, in order? | `messages[]` — `role`, `content`, `createTime` |
| Which tools ran, and did they work? | `messages[].steps[]` — `functionName`, `functionArguments`, `status`, `mcpServerName`; the step's `summary` is model text, not the result |
| What is waiting on a human? | any step with `status: …PENDING_APPROVAL`; any `suggestedChanges` entry with `applied: false`; **and** a last `assistant` message whose `content` ends in a question to the human with no pending step and no `suggestedChanges` (a prose proposal). Check the steps first: a `SUCCEEDED` `k8s-patch-resource` step for the same change means nothing is waiting, whatever `applied` says |
| Did a proposed fix actually land? | a `k8s-patch-resource` step that `SUCCEEDED` — **not** `suggestedChanges[].applied`, and not the agent's summary; both are self-reported |
| Which runbooks applied? | `runbooks[]`, plus `incident.rootCause` / `resolution` |
| Was it resolved, and when? | `incident.resolvedAt`, `incident.resolution` |
| What did the advisor conclude? | `promotionAnalysis.decision`, `riskLevel`, `summary` |
| Did the current principal start it? | `ownedByMe` |

## What the native agent could have done

`list_intelligence_tools` answers "what can the native agent do in this org." Each entry has `name`, `displayName`, `description`, `mutable`, and `availabilityRequirements` (the context types a tool needs). Do not use `mutable` as the only safety check; inspect the proposed action and approval state as described in the skill.
