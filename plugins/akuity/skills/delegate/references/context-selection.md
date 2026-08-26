# Choosing conversation contexts

Contexts are not just labels. They define the resources in scope and which environment-aware capabilities the Deployment Agent can use. Choose them from the work the human asked for, not only from whether the target is Argo CD or Kargo.

Tool availability can change between platform versions. Trust the conversation's advertised tools first; use this guide to choose the intended scope.

## Context shapes

Each item in `contexts` contains one context type:

```jsonc
[
  { "argoCdApp": { "instanceId": "<argocd-instance>", "name": "<app>", "namespace": "argocd" } },
  { "argoCdInstance": { "instanceId": "<argocd-instance>" } },
  { "argoCdCluster": { "instanceId": "<argocd-instance>", "clusterId": "<cluster>" } },
  { "k8sNamespace": { "instanceId": "<argocd-instance>", "clusterId": "<cluster>", "name": "<namespace>" } },
  { "kargoProject": { "instanceId": "<kargo-instance>", "name": "<project>" } }
]
```

The example shows every shape together only for reference. Do not copy the whole list into one conversation. Resolve exact names and ids with MCP read/list tools before creating or changing the conversation; never guess them.

## Pick the context by the work

| Context | Use it for | What it does not cover |
| --- | --- | --- |
| `argoCdApp` | Detailed spec, status, resource tree, events, diffs, sync/refresh/rollback, live workload investigation, or repository work for one app or an exact set of apps | Apps that are not in the selected set; instance-wide image, CVE, deprecated-API, or resource inventory |
| `argoCdInstance` | Application inventory and summaries, instance settings/version, or fleet-wide Kubernetes resource, image, CVE, deprecated-API, and container questions | Per-app detail, actions, or repository work |
| `argoCdCluster` | Application inventory and fleet questions narrowed to one cluster | Per-app detail/actions, repository work, or instance settings/version |
| `k8sNamespace` | The resource tree, workloads, logs, and live Kubernetes investigation for one namespace or an exact set of namespaces | Argo CD app detail/actions, repository work, or instance-wide image/CVE and deprecated-API inventory |
| `kargoProject` | Project structure, stages, warehouses, freight, promotions, and operational work for one project or an exact set of projects | Kargo projects that are not in the selected set |
| no context | A general Kubernetes, Argo CD, Kargo, or Akuity question that needs no environment lookup | Inspecting or changing environment resources |

An app, namespace, or Kargo project used by the Deployment Agent must appear in the selected context set. Selecting one app does not make every app in its instance part of the conversation.

## Common choices

- **"Summarize all apps" using list-level fields**: use `argoCdInstance`, or `argoCdCluster` when the request names one cluster. This is enough for app identity, source, destination, sync, health, summary, and last-operation information.
- **"Give me a detailed summary of apps A, B, and C"**: use three exact `argoCdApp` contexts so the Deployment Agent can perform per-app reads for those apps.
- **"Give me detailed summaries for every app"**: list the apps first with an MCP inventory tool, then attach the exact app contexts needed. If the set is large, use the list-level summary or ask the human to narrow it instead of creating an oversized conversation scope.
- **"What images, CVEs, or deprecated APIs exist here?"**: use `argoCdInstance` or `argoCdCluster`, not app or namespace context.
- **"What is happening in this namespace?"**: use the exact `k8sNamespace`.
- **"Explain or operate this Kargo pipeline"**: use the exact `kargoProject`. `kargoInstanceId` selects the conversation's Kargo instance but does not identify a project by itself.
- **A conceptual question with no environment lookup**: no context is fine. The platform surface still requires `instanceId` or `kargoInstanceId` when creating the conversation; that selects where the conversation belongs, not its resource context.

## Multiple contexts and follow-ups

- Prefer a set of the same context type when the request spans several apps, namespaces, clusters, or Kargo projects.
- Do not mix broad and narrow contexts merely to expose more capabilities. It makes the target ambiguous and gives the Deployment Agent more scope than the request needs.
- Keep the context type stable during a conversation. Change it only when the human changes the requested scope or when a discovery step resolves the exact resources required by that request.
- On an ordinary follow-up, resend the current contexts unchanged because `create_intelligence_message.contexts` is a full replacement set.
- When the scope changes, send the complete new desired context set, not only additions or removals, and preserve the complete runbook set in the same call.

## Special conversations

- **On-call Agent**: an incident requires exactly one `argoCdApp` or `k8sNamespace` context. Instance, cluster, Kargo project, multiple, or empty contexts are rejected.
- **Promotion Advisor**: `kargoPromotionAnalysis` establishes the project scope from the requested project. Do not broaden or replace that scope during the analysis.
