# Official documentation routing

Use this reference before creating or changing a platform instance, cluster, Argo CD resource, or Kargo resource. It is a routing list, not a replacement for the live MCP tool description.

## Source order

1. Read the live MCP tool description. It defines the call arguments and which list carries each manifest.
2. Read the existing resource, when one exists, and preserve its architecture unless the user asked to change it.
3. For an existing instance, read its actual version and use the matching official product docs or CRD reference for the manifest fields. For a new instance, omit `spec.version`, wait until it is healthy, then read the platform-assigned version before writing resources inside it.
4. If a field is absent from both the live tool contract and the matching official reference, ask the user or defer that setup. Never try plausible field names or nested wrappers until one applies.

A validation or ignored-field failure is a signal to re-check these sources, not permission to experiment with another guessed shape. If the live tool and the versioned resource reference disagree, the live tool controls the outer call and the installed version's CRD controls the manifest inside it; report any remaining conflict.

## Akuity Platform instances and clusters

- Reference index: https://docs.akuity.io/akuity-portal/reference/
- Declarative instance specs: https://docs.akuity.io/akuity-portal/reference/declarative-specs/
- Argo CD cluster and Akuity Agent behavior: https://docs.akuity.io/argocd/managing-instances/clusters/
- Cluster creation CLI flags and defaults: https://docs.akuity.io/akuity-portal/reference/cli/akuity_argocd_cluster_create/
- Agent installation alternatives: https://docs.akuity.io/akuity-portal/automation/agent-helm-chart/

The declarative-spec page does not define the MCP cluster wrappers; `manifest-kinds.md` does. A workload cluster is a `Cluster` with `spec.data.size`; the Kargo control plane is a `Cluster` with `spec.data.size: small` plus `spec.data.directClusterSpec: {clusterType: kargo, kargoInstanceId}`. Use those shapes as written — the apply's `clusters` argument is a free-form object, so the live schema will not spell them out, and that absence is not a reason to fall back to the portal.

## Akuity Terraform provider (Terraform mode)

- Provider docs, per version: `https://registry.terraform.io/providers/akuity/akp/<version>/docs` — replace `<version>` with the exact version pinned in `required_providers`, not `latest`. Each resource page ends with an **Import** section giving that version's import ID format.
- Reference layout and verified gotchas: https://github.com/akuity/akp-infra (`01-argocd`, `02-kargo`, `03-clusters`), with `docs/importing-existing.md` for adoption and `docs/day-2.md` for drift, upgrades, and remote state.
- Terraform `import` blocks: https://developer.hashicorp.com/terraform/language/import

The registry page carries semantics; **attribute names, nesting, and required/optional status come from `terraform providers schema -json` on the installed provider**, and the instance `version` is the user's choice; `akuity argocd instance versions` / `akuity kargo instance versions` list what the organization can run. `references/terraform-mode.md` has the full source order.

## Kargo resources

- Docs home: https://docs.kargo.io/
- Warehouses and artifact selection: https://docs.kargo.io/user-guide/how-to-guides/working-with-warehouses
- Stages and requested Freight: https://docs.kargo.io/user-guide/how-to-guides/working-with-stages
- Freight: https://docs.kargo.io/user-guide/how-to-guides/working-with-freight
- Argo CD integration: https://docs.kargo.io/user-guide/how-to-guides/argo-cd-integration
- Promotion step reference: https://docs.kargo.io/user-guide/reference-docs/promotion-steps
- Version-pinned CRDs: `https://doc.crds.dev/github.com/akuity/kargo@<instance-version>`

Read the Kargo instance version before writing a manifest and replace `<instance-version>` with its exact upstream version, for example `v1.11.2`. Use the CRD reference to settle field names. In v1.11 image subscriptions use `constraint`, `allowTagsRegexes`, and `ignoreTagsRegexes`; similarly named Git or Helm fields are not interchangeable.

## Argo CD resources

- Application spec: https://argo-cd.readthedocs.io/en/stable/user-guide/application-specification/
- Declarative setup, repositories, credentials, and clusters: https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/
- Project spec: https://argo-cd.readthedocs.io/en/stable/operator-manual/project-specification/
- ApplicationSet docs: https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/

Match the instance's upstream minor version: for an Akuity build such as `v3.2.0-ak.58`, replace `stable` with `release-3.2`. Do not flatten an existing multi-source Application, ApplicationSet, Helm source, custom sync policy, project restriction, branch layout, or destination into this skill's simple directory example. Read it and ask about any architectural choice the existing resources do not answer.
