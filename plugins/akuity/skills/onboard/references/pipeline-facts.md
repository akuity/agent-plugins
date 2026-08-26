# Kargo pipeline facts

These are not discoverable from the tool schemas, and most of the failures they prevent are silent. Use them when writing the Application, Project, Warehouse, Stage, and credential manifests.

## Ownership and scoping

- **Warehouse and Stage resources must each carry `spec.shard` set to the name of the Kargo agent** registered for this Argo CD instance. Project has no shard field. Without it, the warehouse produces no Freight and the pipeline remains idle without reporting an error.
- **Set `metadata.namespace: <project name>` explicitly on every Warehouse, Stage, and credential Secret sent through the platform endpoint's Kargo apply.** An omitted namespace there does not default to the project — the resource lands in the instance's own `kargo` namespace, applies cleanly, and is silently never picked up: the same no-error idle pipeline as a missing shard. (Only on a Kargo instance's own endpoint does an omitted namespace default to the project named in the call's `kargoProject.name` argument.) `Project` itself is cluster-scoped and takes no namespace.

## Image selection and freight

- **A warehouse image subscription defaults to `imageSelectionStrategy: SemVer` with `strictSemvers: true`.** If the image's tags are not semver, set an explicit strategy (e.g. `NewestBuild` or `Lexical`) or you get the same silent no-freight failure as a missing shard.
- **Never write `allowTags` or `ignoreTags` in a subscription — use `allowTagsRegexes` / `ignoreTagsRegexes`.** Kargo v1.11 removed support for the old fields (every instance this flow creates runs ≥ 1.11, since versions default to latest), but they are still in the schema, so a manifest carrying one **applies cleanly** and only fails at discovery: the warehouse reports "Unable to discover artifacts: … AllowTags is deprecated and unsupported as of v1.11.0". Older Kargo docs and examples all show `allowTags` — do not copy them. The shapes also differ: the new fields are **lists of regexes** (`allowTags` was a single regex string; `ignoreTags` was literal tag names). This applies to image and git subscriptions alike. Only on a pre-existing instance older than 1.11 do the old fields still work — check the instance's version before migrating one.
- The warehouse polls the registry on `spec.interval` (default `5m0s`). Its `status.discoveredArtifacts` can list several candidate image references, controlled by each subscription's `discoveryLimit` (default `20`), but automatic Freight creation uses only the best current artifact set. Treat an automatically created newest Freight as discovery output, never as the user's release choice. A private registry produces no discovered artifacts until its image credential exists in the project.
- If the user chooses "latest," resolve it immediately to the explicit discovered tag and Freight and show that value in the plan. Never approve or execute a floating `latest` pointer whose meaning can change while the flow is running.
- **Freight is produced by the warehouse, never created by hand in this flow.** After the user chooses a Stage-to-version matrix, compute the distinct required versions and make sure each has Freight. For a missing semver image, temporarily set the image subscription's `constraint` so that version is the best match; for another strategy, use an exact `allowTagsRegexes` entry. Wait for the Freight, repeat for other missing versions, then restore the user's intended steady-state subscription. Never leave an exact or upper-bound discovery constraint behind merely because it was useful for onboarding. `semverConstraint` is not an image-subscription field in Kargo v1.11 — it was removed in v1.9; `constraint` is the replacement. Git and Helm subscriptions have their own similarly named fields, so check the versioned CRD instead of copying one across resource types.
- **Re-read freight for the target stage before every promotion.** Eligibility is per-stage (the stage's freight requests decide what it accepts), and the promote tool's own contract requires querying the freight list with the target stage first. A "freight is not available to Stage" error means it was skipped.
- **Gate each downstream promotion on verification**: after a promotion succeeds, the freight's `status.verifiedIn.<stage>` entry appears a few seconds later, and the next stage only accepts verified freight. Poll for it rather than promoting downstream on the promotion result alone.
- **Each Freight follows a contiguous path through a linear Stage chain, but one release plan may contain several Freight.** Build the plan backward from the complete desired Stage-to-version table. For `dev=v2`, `staging=v2`, `prod=v1`, promote `v1` through `dev → staging → prod` first, then `v2` through `dev → staging`. A downstream Stage cannot see Freight until it is verified upstream unless someone separately approves that Freight for the Stage; do not turn that exceptional hotfix mechanism into the default onboarding path.
- **When the apps were created without a first sync, each environment's first promotion is its first deployment.** In pipeline scope the scaffold starts with the tag approved for each Stage. A hop may therefore find no image change, or it may temporarily replace a later Stage's final tag while an older Freight passes through on its way downstream. An empty diff is fine: `git-commit` returns the existing HEAD and `argocd-update` still drives the sync — that sync is the point.

## The Applications the pipeline manages

- Each Application must carry the annotation `kargo.akuity.io/authorized-stage: "<project>:<stage>"` naming the stage that manages it — without it the `argocd-update` promotion step is refused.
- **Give the apps no `syncPolicy.automated`** — the `argocd-update` step drives syncs from promotion, and automated sync would race it. With a pipeline in scope, do not sync the apps at creation either: each environment's first deployment is its first promotion, so nothing reaches a cluster outside the pipeline. Only a scope with no pipeline (apps and nothing more) deploys with one explicit sync call per app.

## Promotion steps

- **Every stage needs promotion steps**, at `spec.promotionTemplate.spec.steps`. A stage with none rejects all promotions.
- The first stage requests freight with `requestedFreight: [{origin: {kind: Warehouse, name: <warehouse>}, sources: {direct: true}}]`; each downstream stage uses `sources: {stages: [<upstream stage>]}` instead of `direct`.
- The default step shape for a simple scaffold that keeps all environment directories on one branch (verified on Kargo v1.11; step configs move between versions, so the instance's versioned docs win):

```yaml
promotionTemplate:
  spec:
    steps:
      - uses: git-clone
        config:
          repoURL: https://github.com/<org>/<repo>.git
          checkout:
            - branch: main
              path: ./repo
      - uses: yaml-update
        config:
          path: ./repo/<env-dir>/deployment.yaml
          updates:
            - key: spec.template.spec.containers.0.image
              value: ${{ imageFrom("<image repo>").RepoURL }}:${{ imageFrom("<image repo>").Tag }}
      - uses: git-commit
        config:
          path: ./repo
          message: Promote <app> ${{ imageFrom("<image repo>").Tag }} to <env>
      - uses: git-push
        config:
          path: ./repo
      - uses: argocd-update
        config:
          apps:
            - name: <app for this environment>
```

- `yaml-update` addresses list elements with dotted numeric indexes (`containers.0.image`), not brackets.
- **Do not pin `argocd-update.sources[].desiredRevision` to the pushed commit when several Stage applications track the same branch.** A later Stage's push advances that branch and can make an earlier Stage's health check expect a commit the Application no longer observes. With a shared branch, the app-only `argocd-update` above triggers sync and checks the operation without a stale commit pin. A per-Stage branch or another isolated immutable source may safely use `desiredRevision`; inspect and preserve an existing repo's strategy, and confirm complex layouts with the version-matched Kargo and Argo CD docs before writing the steps.
- **`argocd-update` works only if both links exist**: the Kargo agent has `spec.data.remoteArgocd` set to the Argo CD instance id, and the target Application carries the `authorized-stage` annotation above. Missing either shows up as a promotion failure at the last step, after git has already been updated.

## Credentials

- **Credentials are the user's to create, never yours.** Do not collect a token in chat or place one in an MCP tool argument. Your job is the exact command with a placeholder, and verification by outcome afterward.
- **Each credential is needed only when the thing that uses it actually runs — never front-load them, and never block unrelated work on one.** The Kargo git credential matters only once a Promotion executes: a private repo fails at `git-clone` without read access; a publicly readable repo can clone but fails at `git-push` without write access. A user who never promotes never needs it. The image credential matters only for a private registry: without it the Warehouse silently never produces Freight. The Argo CD repository credential matters only for a private repo: without it apps cannot read the repo and show comparison errors. Raise each at the moment its consumer runs, build everything else in the meantime, and put whatever is still missing in the closing checklist with its consequence.
- **Creating them needs no instance login**: labeled credential Secrets applied from the user's own shell with the akuity CLI — the same platform API and login the agent install already used. Command substitution keeps the token out of the pasted command and out of shell history:

```
akuity kargo apply --organization-id <org> --name <kargo-instance-name> -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: git-creds
  namespace: <project>
  labels:
    kargo.akuity.io/cred-type: git
stringData:
  repoURL: https://github.com/<org>/<repo>.git
  username: <user>
  password: $(gh auth token)
EOF
```

  A private image registry needs a second Secret with `kargo.akuity.io/cred-type: image` and `repoURL: <registry>/<repo>`. Where no substitution like `gh auth token` exists, leave `password:` as a placeholder for the user to fill in before running.
- The Kargo credential shape, for verification: a Secret **in the project's namespace** with the label `kargo.akuity.io/cred-type` (`git` or `image`) and data of `repoURL`, `username`, `password`. A Secret missing the label — or one that landed outside the project's namespace — is accepted and then silently never used as a credential: the same no-error idle pipeline as a missing shard.
- The `repoURL` must match the URL the warehouse subscription and promotion steps use **exactly**, `.git` suffix included — a mismatched URL is another silent no-credential failure.
- **Argo CD does not share Kargo's credentials**: a private repo also needs a repository credential on the Argo CD side — a Secret labeled `argocd.argoproj.io/secret-type: repository` with `stringData` of `type: git`, `url`, `username`, `password`, applied the same way (`akuity argocd apply --organization-id <org> --name <argocd-instance-name> -f -`), or the instance UI's repository settings. Argo CD's repository store is keyed by the repository `url` — the entry's name is not preserved, and removing it later means deleting by repository URL.
- **With instance access set up** (`instance-access.md`), the instance CLIs work too: `argocd repo add <repo-url> --username <user> --password <token>` on the Argo CD side, and on the Kargo side — the subcommand is `repo-credentials`; **`kargo create credentials` does not exist** and fails with an unknown-flag error:

```
kargo create repo-credentials --project=<project> <name> \
  --git --repo-url=<repo-url> --username=<user> --password=<token>
```

  Use `--image` with `--repo-url=<registry>/<repo>` for a private image registry credential, `--helm` for chart repositories. A GitHub user can pass `--password "$(gh auth token)"` so the token never leaves their shell.
