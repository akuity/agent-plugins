# Terraform mode: the platform layer as code

Read this before writing or changing any Terraform in Terraform mode. It covers what Terraform owns, where every value comes from, the shapes that must be explicit, adopting existing resources, and what the agent runs and never runs. The flow deltas are in `SKILL.md` ("Terraform mode"); this file carries the facts.

Terraform mode manages the **platform layer** — Argo CD instances, cluster registrations, Kargo instances and agents — with the [`akuity/akp` provider](https://registry.terraform.io/providers/akuity/akp/latest/docs), laid out like Akuity's own [akp-infra](https://github.com/akuity/akp-infra). Git is the write path for that layer and the user (or their CI) runs `terraform apply`; the MCP tools are the read and verify surface. The config layer — Applications, Kargo Project, Warehouse, Stages — is unchanged: direct apply or GitOps mode, as chosen at scope time. Terraform mode plus GitOps mode is exactly akp-infra plus akp-platform.

## What Terraform owns

- Terraform: `akp_instance` (Argo CD), `akp_kargo_instance`, `akp_kargo_agent`, `akp_cluster`, and optionally `akp_kargo_default_shard_agent`. Data sources `akp_instance`, `akp_kargo_instance`, `akp_cluster(s)`, `akp_kargo_agent(s)` read what other stacks created.
- MCP: every read and health check in the flow, and everything inside the instances under whichever config-layer mode the user chose.
- **Once a platform object is in Terraform state, never write it through MCP.** `apply_argocd_instance`, `apply_kargo_instance`, and the platform deletes on a Terraform-managed instance, cluster, or agent create drift that the next `terraform apply` reverts, and a spec change made through MCP can be lost silently. Changing or removing one means a commit and an apply, not an MCP call — the same rule GitOps mode applies to the config layer.
- The provider can also carry the config layer (`akp_instance.argocd_resources` for Application, AppProject, ApplicationSet; `akp_kargo_instance.kargo_resources` for Project, Warehouse, Stage, PromotionTask, labeled credential Secrets and more). This is **not the default**: promotions write image tags to git and controllers rewrite Stage and Freight status, so Stages in Terraform state fight the pipeline. Offer it only when the user asks, and extend the ownership rule to every resource it carries.

## Layout

Three independent root modules, each with its own state and a committed `.terraform.lock.hcl`:

```
01-argocd/     akp_instance
02-kargo/      data.akp_instance (by name) → akp_kargo_instance → akp_kargo_agent (Akuity-managed, remote_argocd)
03-clusters/   data.akp_instance (by name) → akp_cluster per workload cluster
```

Each directory holds `providers.tf`, `variables.tf`, `main.tf`, `outputs.tf`, and a `terraform.tfvars.example`. Stacks find each other **by instance name through data sources**, never `terraform_remote_state`, so each stays independently applyable. Start from akp-infra's files rather than a blank page — copy the stack and edit — but know which of its choices are the skill's defaults and which are options, or the whole template gets copied:

| akp-infra choice | In this flow |
| --- | --- |
| Self-hosted Kargo agent per cluster in `03-clusters` plus `akp_kargo_default_shard_agent` | **Default is one Akuity-managed agent in `02-kargo`** (`akuity_managed = true`, `remote_argocd` = the Argo CD instance id), matching the skill's step 4: nothing to install. Self-hosted agents are an option the user chooses; then the default shard goes in `03-clusters` as akp-infra does. |
| `akp_cluster "kargo"` registering the Kargo control plane as an Argo CD destination (`direct_cluster_spec`, `cluster_type = "kargo"`) | Only with GitOps mode — it is the bootstrap Application's destination. Direct-apply mode does not need it. |
| Admin account: `argocd_cm`/`argocd_secret`, `kargo_cm`/`kargo_secret` with `bcrypt(var.admin_password)` and `ignore_changes` | Optional, offered in round one ("Credentials and state" below). Without it the instance login stays a portal action exactly as in `instance-access.md`. |
| `declarative_management_enabled = true` | Runs an in-cluster application controller so Applications and AppProjects in the control plane itself can be managed from git (akp-platform's app-of-apps). Only the everything-in-git variant of GitOps mode needs it, and **enabling it requires the organization's app-of-apps feature**; without that feature the apply is denied. Leave it out otherwise. |
| `workspace = "default"` on Kargo resources | Set `workspace` only when the user names one; omitted, the provider uses the organization's default (an instance created without it reads back `workspace = "default"`). |
| `tune_agent_resources` kustomization shrinking agent CPU requests | akp-infra's fix for k3d, kind, and minikube, where default requests can leave agent pods Pending and `ensure_healthy` waiting. Not needed on every small cluster (a default k3d node scheduled all agent pods without it); never for real clusters. |
| Manual kubeconfig parsing in `modules/cluster` | One of two `kube_config` forms; see "Agent installation". |

An existing Terraform repository wins over this layout: add to their structure, follow their module and variable conventions, and do not restructure. Ask about anything the existing code does not answer.

## Where each value comes from

The provider schema settles names, nesting, types, and enums. It does not settle values. Keep the two apart.

**Attribute names, nesting, required or optional, enums** — the schema, read from the installed provider:

1. Pin one exact provider version in `required_providers` (`version = "0.15.0"`, not `~>`), taken from the registry or the `terraform init` output. Do not take a version string from a fetched documentation page's summary; one spike run saw `~> 1.0` invented that way.
2. After `terraform init`, run `terraform providers schema -json`. It is the authority for every attribute; read the resource's block before writing it, and settle any doubt there rather than in memory, in an example, or in a template. The registry docs for the pinned version carry the prose semantics (import ID formats, what a field means); read them for that, not for attribute names.
3. `terraform validate` after writing each stack, `terraform plan` once credentials are in the user's shell. A failure is a signal to re-read the schema and make **one** correction backed by it. Two failures with no new information is a stop condition: say what is missing and ask.

`validate` is static: it does not evaluate `file()`, `yamldecode()`, or data sources, so a kubeconfig path or an instance name that does not exist passes validate and fails at plan. Do not report a stack as working before its plan has run.

**Values the schema leaves open** — each has one legitimate source:

| Value | Source | Never |
| --- | --- | --- |
| `argocd.spec.version`, `kargo.spec.version` (both Required) | The user names the version at scope time. If they want to see what the organization can run, `akuity argocd instance versions` and `akuity kargo instance versions` (no login needed; add `--server` for a non-default portal) or the portal's instance creation page list the choices; when offering that list, leave out `latest` and any `unstable` or pre-release build. The platform accepts a listed base version (`v3.5.2`) and the `-ak.N` builds listed with it (`v3.5.2-ak.94`) alike. Never pick a version on the user's behalf. | `latest` (unsupported), a documentation example, the template's default, an image registry's tag list — all four were tried in the spike and all are wrong or stale. |
| `akp_cluster.namespace` (Required) | `akuity` — the agent namespace convention the CLI and portal install into. | Anything else unless the user's clusters already run agents elsewhere. |
| `spec.data.size` | The user, or `small` for a first cluster; the enum is in the schema. Omit it on an Akuity-managed Kargo agent — the schema says so. | |
| Import IDs | The **Import** section of the pinned version's registry docs, with ids and names from the MCP read tools. | A guessed composite. |
| Organization name | Round one, confirmed against `list_organizations`. | |
| `mcp_server.enabled` (when the schema has it) | `true` on both instances: the in-instance tools from milestone 4 on need it ("MCP access on Terraform-created instances" below). | Leaving it out on a new instance because the attribute is optional — a Terraform-created instance then has MCP access off. |

## Credentials and state

- The provider authenticates from `AKUITY_API_KEY_ID` and `AKUITY_API_KEY_SECRET` in the user's shell — an organization API key with org-admin permission that **the user creates in the portal** (the organization's **API Keys** tab). `AKUITY_SERVER_URL` selects the EU or a non-production portal. Nothing of this goes into a `.tf`, a `.tfvars`, or the conversation; the provider block carries only `org_name`.
- **Admin accounts through Terraform are opt-in.** If the user wants the Argo CD or Kargo admin login declared in code, use akp-infra's shape: a `sensitive` `admin_password` variable fed by `TF_VAR_admin_password` from their shell, `bcrypt(var.admin_password)` in `argocd_secret."admin.password"` / `kargo_secret.adminAccountPasswordHash`, `argocd_cm."accounts.admin" = "apiKey,login"` / `kargo_cm.adminAccountEnabled = "true"`, and `lifecycle { ignore_changes = [argocd_secret] }` (Kargo: `[kargo.spec.version, kargo_secret]`) because `bcrypt()` salts differently on every plan. The password never enters the chat and never a command line; the user exports the variable themselves. This replaces the portal **System Accounts** step for that instance; the Kargo tools still need the account before Freight reads and promotions work (`instance-access.md`).
- State holds secrets (the bcrypt hash, kubeconfig material). akp-infra gitignores `terraform.tfstate` and recommends a remote backend for anything shared. **Ask which backend at scope time and do not invent one**; local state is fine for a personal first run.
- `ignore_changes = [kargo.spec.version]` is needed regardless of admin accounts: the platform bumps Kargo patch versions server-side and the plan would otherwise fight it. The cost is that a version bump in the variable is then a no-op; to upgrade deliberately, remove the entry temporarily, set the new version and apply, then restore it (akp-infra `docs/day-2.md`).

## Agent installation

`akp_cluster.kube_config` is optional. Set, the provider connects to the cluster and installs the Argo CD agent during apply; `ensure_healthy = true` then blocks the apply until the agent reports healthy, so a finished apply means a live agent. Unset, the apply only registers the cluster and the agent is installed the way step 3 already does it, with `akuity argocd cluster install-agent` in the user's shell.

Two forms of `kube_config`, pick one deliberately:

- **`config_path` + `config_context`** — the provider reads the user's kubeconfig itself (`config_path = pathexpand(var.kubeconfig_path)`; Terraform does not expand `~`). Simplest, and the schema also offers `exec` and `token` for kubeconfigs that authenticate through a plugin. Use this first.
- **Parsed certificates** — akp-infra's `modules/cluster` decodes the kubeconfig with `yamldecode(file(...))`, extracts `client-certificate-data`/`client-key-data`, and rewrites a `0.0.0.0` API server address to `127.0.0.1` because k3d binds there. It only works for kubeconfigs with embedded certificates (k3d, kind, minikube, most on-prem); EKS, GKE, and AKS kubeconfigs use an exec plugin and have no certificate data to extract.

If neither form can reach the cluster from the machine running Terraform, register without `kube_config` and install the agent with the CLI. **Never compose an `exec` block or credential fields you were not given.**

**A create that fails is rolled back, on a best-effort basis.** When the agent does not become healthy within the provider's ten-minute wait, or the manifest apply fails, the provider deletes the registration it just made and the resources it installed in the cluster, and writes nothing to state. Cleanup can itself fail: a manifest deletion error is only logged and the run continues, and if the platform-side delete fails the error says `failed to clean up cluster`. An empty state therefore does not prove nothing was left behind. Before retrying, check for leftovers with `list_argocd_instance_clusters` and, in the user's shell, `kubectl -n akuity get all`; a leftover registration blocks the retry on the name. To find out why an agent never became healthy, re-apply once with `ensure_healthy = false`, then look at the pods (`kubectl -n akuity get pods` in the user's shell; `ImagePullBackOff` and `Pending` are the usual answers), and set it back to `true` once the cause is fixed.

`akp_kargo_agent` has no `ensure_healthy`, and an Akuity-managed agent needs no `kube_config`: its health is confirmed from the platform reads after the apply, not by Terraform.

**The first Kargo agent becomes the instance's default shard, and the platform refuses to delete a default-shard agent.** `terraform destroy` on a Kargo stack with a single agent therefore fails at the agent (`cannot delete default shard agent, change default shard before deleting`) before it reaches the instance. Before such a destroy, the user clears the assignment in their shell (`akuity kargo agent update <agent> --instance-name <instance> --default-shard=false`) or moves it to another agent; a stack that manages the shard explicitly with `akp_kargo_default_shard_agent` destroys that resource first by dependency order and does not hit this.

## Shapes that must be explicit

Set these attributes explicitly on every `akp_cluster`. For an adopted cluster, copy the live values so import produces no updates or replacement. Defaults apply only to new clusters:

- `spec.namespace_scoped` — preserve the live value, including `true`; changing it forces replacement. For a new cluster, use `false` unless the user requests a namespace-scoped agent.
- `spec.description` and `spec.data.project` — preserve the live strings. For a new cluster, use `""` when the user has not supplied a value.
- `spec.data.size` — preserve the live value for an adopted cluster (often `auto`); for a new cluster, follow "Where each value comes from" above.

On an Akuity-managed `akp_kargo_agent`: `spec.data.akuity_managed = true`, `spec.data.remote_argocd = data.akp_instance.<argocd>.id`, no `size`.

## Adopting existing resources

An instance or cluster created in the portal is imported, never recreated (the name collision would fail anyway). The loop per resource, from akp-infra's `docs/importing-existing.md`:

1. Write the resource block first, matching the live object: read it with the MCP tools (`list_argocd_instances`, `list_argocd_instance_clusters`, `list_kargo_instances`, `list_kargo_instance_agents`) and copy name, version, size, and the explicit attributes above.
2. Take the import ID format from the **Import** section of the pinned provider version's registry page. Verified on provider 0.13.0: `akp_instance` and `akp_kargo_instance` import by **name**; `akp_cluster` by `<argocd_instance_id>/<cluster_name>`. The instance id comes from the same MCP read.
3. Prefer declarative `import { to = ..., id = ... }` blocks (Terraform 1.5+): they are plan-reviewable, repeatable, and the user applies them like any other change. If the user prefers `terraform import`, they run it.
4. `terraform plan` must reach zero. In-place updates mean the config disagrees with reality — fix the config, do not let Terraform "correct" a production instance on first contact. **A destroy/recreate after import is a stop**: an immutable or replacement-forcing attribute does not match; fix it and never apply a replacement that was not intended.

Known provider issues (observed on 0.13.0; re-check on the pinned version):

- `akp_kargo_agent` cannot be imported when the organization uses workspaces (the read resolves the wrong workspace and surfaces `Cannot import non-existent remote object`). akp-infra's workaround is a `manage_kargo_agent = false` switch that leaves the existing agent unmanaged.
- Updating an imported `akp_cluster` can fail with `invalid Cluster spec: parsing time "" as "2006-01-02T15:04:05Z07:00"` (empty `maintenance_mode_expiry`). akp-infra's workaround is an `adopted = true` switch: no `kube_config`, `ensure_healthy = false`, so the config matches imported state and no update is issued.

Imported instances keep their existing admin password: `argocd_secret` / `kargo_secret` are write-only and under `ignore_changes`.

## MCP access on Terraform-created instances

Instances created through the platform MCP endpoint get `mcpServer.enabled` set automatically. **Instances created by Terraform do not**, unless the configuration sets it. The platform lifecycle reads used to verify milestones 1–3 (`list_argocd_instances`, `list_argocd_instance_clusters`, `list_kargo_instances`, `list_kargo_instance_agents`) are not gated by it and keep working. Every in-instance tool — application reads and syncs, freight, promotions, the single-resource deletes — is denied with a message naming MCP until it is on.

The pinned provider's schema decides the path (`terraform providers schema -json`, not the version string):

- **`akp_instance` and `akp_kargo_instance` carry `mcp_server`** (providers released after 0.15.0): set `mcp_server = { enabled = true }` in stacks 01 and 02 so access is on from creation. Terraform then owns the toggle: switching it off in the portal shows as drift on the next plan and the apply restores it. The attribute is optional and computed, so leaving it out is valid and means Terraform neither sets nor tracks it — which on a new instance leaves access off. The `akp_instance` and `akp_kargo_instance` data sources expose the same attribute for a read-back.
- **No `mcp_server` in the schema** (0.15.0 and earlier): the user enables access for both instances under **Organization Settings → MCP Access → Endpoints → Instance access** (or the instance's own **Settings → MCP Access** page). Raise this right after stack 02 applies, before anything in milestone 4 is attempted. A later `terraform apply` does not clear it: the platform preserves a stored `mcpServer` when an update omits it.

On an adopted instance the same attribute applies: write the live value so the import plans no update, and change it through a commit and an apply like any other Terraform-owned field.

## What the agent runs, and what only the user runs

- Agent, in the user's clone: `terraform fmt`, `terraform init`, `terraform validate`, `terraform providers schema -json`, and `terraform plan` once the API key is exported in the shell the command runs in. `init` writes `.terraform/` and the lock file; say so.
- **User, or their CI: `terraform apply`**, always after reviewing the plan; `terraform destroy` if they ever want it; `terraform import` if they choose it over import blocks. Terraform creates and deletes real infrastructure and holds their credentials — the agent writes the files and reads the plan, the user executes.
- Never edit `terraform.tfstate` or run `terraform state` subcommands; never pass `-auto-approve`.
- Never change the user's machine-wide Terraform setup to make a run work: no writing `~/.terraformrc` or `TF_CLI_CONFIG_FILE`, no plugin cache directories, no provider mirrors, no exported variables outside the shell they gave you. If `init` cannot reach the registry, say so and stop.

## Verify after each apply

The user's `terraform apply` finishing is not the health signal (except for `akp_cluster` with `ensure_healthy`). After each stack, read the platform: `list_argocd_instances` until the instance is healthy after stack 01; `list_kargo_instances` and `list_kargo_instance_agents` after stack 02 (the agent's `remoteArgocd` must name the healthy Argo CD instance id); `list_argocd_instance_clusters` after stack 03. Only then continue — stack 02's data source lookup of the Argo CD instance and the agent's `remote_argocd` both need the instance to exist and be healthy, and a reference to a still-provisioning instance is rejected.
