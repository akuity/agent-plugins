# Resource manifests: which kind goes to which endpoint

Resources are supplied as Kubernetes-style manifests everywhere (`apiVersion`, `kind`, `metadata`, `spec` as YAML/JSON documents). The platform endpoint's apply tools are **composite** and carry the whole lifecycle: one call takes an instance together with its platform children *and* the resources that live inside it. Each instance's own endpoint accepts the in-instance kinds too — under the instance's own RBAC, for callers whose access lives there (`endpoints-and-auth.md` has the split).

## Platform endpoint

On the platform applies, each in-instance kind rides in its own list argument of the call, not in one concatenated manifest string — the tool's input schema names the arguments.

- The Argo CD apply takes the `ArgoCD` instance manifest (`argocd.akuity.io/v1alpha1`, spec under `spec.instanceSpec`), plus `Cluster` registrations (`spec.data.size`, lowercase — e.g. `small`), plus Argo CD `Application` / `ApplicationSet` / `AppProject` manifests, plus repository-credential `Secret`s, plus instance config documents (the `argocd-cm` ConfigMap and `argocd-secret` Secret).
- The Kargo apply takes the `Kargo` instance manifest (`kargo.akuity.io/v1alpha1`, spec under `spec.kargoInstanceSpec`), plus `KargoAgent` registrations (`spec.data`), plus Kargo `Project` / `Warehouse` / `Stage` manifests (and promotion tasks, analysis templates), plus credential `Secret`s, plus config documents (`kargo-cm` / `kargo-secret`).
- Both address the instance by name (`idType: NAME`). Reads echo enum forms back — a cluster created with `size: small` reads back as `CLUSTER_SIZE_SMALL`.
- **Namespaces are not project-aware on the platform applies.** An Argo CD `Application` may omit `metadata.namespace` (it defaults to the instance's argocd namespace), but every Kargo `Warehouse`, `Stage`, and credential `Secret` **must set `metadata.namespace: <project name>`** — omitted, the resource lands in the instance's own `kargo` namespace, applies cleanly, and is silently never picked up. `Project` itself is cluster-scoped.
- **Ordering inside one call is handled for you**: the Kargo apply creates `Project`s before the resources inside them, so a single call can carry a project together with its whole pipeline.

Facts about platform manifests that no schema tells you:

- **Omit `spec.version` when creating instances.** The platform defaults it to the latest stable version for the organization. No MCP tool lists versions; the akuity CLI's `akuity argocd instance versions` does. Do not set the literal value `latest`: it is unsupported for this workflow and can leave Kargo tools unavailable even when the instance reports healthy.
- **`mcpServer.enabled` in the instance spec gates MCP access to the instance on both surfaces** (under `spec.instanceSpec` for Argo CD, `spec.kargoInstanceSpec` for Kargo): it serves the instance's own `/mcp` endpoint **and** allows the platform endpoint's in-instance tools — application reads and syncs, freight, promotions, the single-resource deletes — to target the instance. Instances created through the platform endpoint get it enabled automatically; a pre-existing instance without it answers those tools with a permission denial naming MCP.
- **A Kargo instance needs an admin account or OIDC before Project, Stage, Freight, and Promotion tools are available.** Platform lifecycle reads and `apply_kargo_instance` remain available, so authentication does not have to be part of the initial instance manifest. The user can configure it later under **Settings → System Accounts** or **Settings → OIDC Config**, or apply `kargo-cm` and `kargo-secret` declaratively. Argo CD does not require an admin account or OIDC for the platform endpoint's Argo CD tools.
- **A child that references a different instance's id, such as a `KargoAgent`'s `spec.data.remoteArgocd`, must name an instance that already reports healthy.** A still-provisioning or no-longer-listed instance is rejected. Read the current id after the instance becomes healthy; do not reuse one recorded before it.

## Instance endpoint kinds

An instance's own endpoint validates and authorizes resources under the instance's RBAC. The `apply_argocd_resource` and `apply_kargo_resource` tools accept multiple `---`-separated documents in the `manifest` argument. Platform apply tools use organization authorization and accept the same resource kinds through their typed list arguments.

- Argo CD: `Application`, `AppProject`, `ApplicationSet`, and repository-credential `Secret`s — labeled `argocd.argoproj.io/secret-type: repository` or `repo-creds`, applied through Argo CD's repository APIs and keyed by the `url` field rather than the Secret name.
- Kargo: `Project`, `Stage`, `Warehouse`, promotion tasks, analysis templates, credential `Secret`s, and project-level RBAC kinds. An omitted `metadata.namespace` defaults to the project named in the call's `kargoProject.name` argument — set that argument to your project on every call.
- **Instance-wide configuration never goes here**: the `argocd-cm`/`argocd-secret` documents, trust config (`argocd-ssh-known-hosts-cm`, `argocd-tls-certs-cm`), image updaters, and `kargo-cm`/`kargo-secret` belong to the platform applies above. Each surface owns its side — the instance applies reject these kinds.

## Apply semantics

- **Applies are per-resource, not atomic — on every endpoint.** One resource can fail (e.g. a transient timeout inside the instance) while its siblings succeed: the instance endpoints return one result per document, and a failed platform apply may still have landed the resources it processed before the error. Apply is upsert, so recovery is the same everywhere — fix what the error names and re-send; a re-sent succeeded resource is harmless.
- **On the Kargo instance endpoint, a `Project` must be the first document** of the call that creates resources inside it. (The platform apply orders kinds itself — see above.)
- **Apply is strictly additive on every endpoint** — nothing is pruned; removal is always an explicit delete tool. The single-resource delete tools (`delete_argocd_resource` / `delete_kargo_resource`) exist on the platform endpoint (naming the instance) and on the instance endpoints alike, and the platform endpoint also carries the platform-object deletes — `delete_argocd_instance`, `delete_argocd_instance_cluster`, `delete_kargo_instance`, `delete_kargo_instance_agent`. All of them are destructive: confirm with the user, naming the object, before calling one. Deleting an Application with `cascade: true` also deletes its deployed workloads; never choose cascade on the user's behalf.
