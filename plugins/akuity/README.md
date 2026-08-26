# Akuity agent plugin

Agent workflows for the Akuity Platform, driven through its MCP endpoints. Installing this plugin connects the Akuity platform MCP server (the US endpoint by default) and adds maintained skills for workflows that need sequencing, platform knowledge, or a consistent approval flow.

## What installing gets you

- **The platform MCP server**: one entry, `akuity`, for the platform endpoint. It defaults to `https://akuity.cloud/mcp`; in Claude Code the EU endpoint can be selected at install time (see below), while Codex always uses the US endpoint. Installing wires the server but does not log you in — see "Install and authenticate" below. The endpoint model is documented in [`references/endpoints-and-auth.md`](references/endpoints-and-auth.md).
- **Workflow skills** (`skills/`): each skill guides an agent through one workflow over the MCP tools. The platform endpoint carries every workflow end to end — instance-level resources included; each instance also serves its own MCP endpoint for working with that instance's resources under its own RBAC, which is the surface for users who hold instance access but not organization access.

## Requirements

- There is no customer-side switch. If **MCP Access** is unavailable in the portal, contact an Akuity representative or support.
- An Organization Owner or a custom role with Organization update permission has enabled the platform endpoint under **Organization Settings → MCP Access**.
- Instances the agent should reach have MCP access enabled under **Organization Settings → MCP Access → Instance access**, or on the instance's own **Settings → MCP Access** page.

## Install and authenticate

The plugin is served through the public [`akuity/agent-plugins`](https://github.com/akuity/agent-plugins) marketplace.

### Claude Code

```text
/plugin marketplace add akuity/agent-plugins
/plugin install akuity@akuity
```

This connects the US endpoint. EU organizations set the `endpoint` option at install time from a shell instead:

```shell
claude plugin install akuity@akuity --config endpoint=https://eu.akuity.cloud/mcp
```

The value is stored under `pluginConfigs` in `~/.claude/settings.json`, where it can be changed later.

After installing, run `/mcp`, select `akuity`, and authenticate in the browser. The platform endpoint accepts the client's native loopback callback; no fixed callback port is required.

### Codex

```shell
codex plugin marketplace add akuity/agent-plugins
codex plugin add akuity@akuity
```

After installing, start a new session so Codex loads the bundled skills and MCP server, then authenticate `akuity` when prompted.

The plugin is optional. Connect manually when the plugin is unavailable, when a Codex user's organization is in the EU region, when using a self-hosted or non-production platform, or when connecting a direct instance endpoint.

## Other endpoints and automation

Prefer the connection snippet shown in the Akuity Portal. The commands below illustrate the supported client syntax.

For the EU platform endpoint, replace the URL with `https://eu.akuity.cloud/mcp`. For a direct endpoint, use `https://ARGO_CD_HOSTNAME/mcp` or `https://KARGO_HOSTNAME/mcp` and a server name that identifies the instance.

```shell
claude mcp add --transport http akuity-eu https://eu.akuity.cloud/mcp
codex mcp add akuity-eu --url https://eu.akuity.cloud/mcp

claude mcp add --transport http argocd-INSTANCE_NAME https://ARGO_CD_HOSTNAME/mcp
codex mcp add argocd-INSTANCE_NAME --url https://ARGO_CD_HOSTNAME/mcp

claude mcp add --transport http kargo-INSTANCE_NAME https://KARGO_HOSTNAME/mcp
codex mcp add kargo-INSTANCE_NAME --url https://KARGO_HOSTNAME/mcp
```

For CI or automation, register the platform endpoint manually with an Akuity API key in an `Authorization: Bearer AKUITY_API_KEY` header. Keep the key in a secret manager or environment variable; do not put it in prompts, committed configuration, or screenshots.

## Skills

| Skill | What it does |
| --- | --- |
| `akuity:onboard` | Onboarding, whole or in part: provision Argo CD and Kargo instances, connect clusters, deploy an app to every environment, and wire a Kargo promotion pipeline that releases only what the user approves — the user picks how far to go. |
| `akuity:delegate` | Delegate work to Akuity Intelligence: talk normally with the Deployment Agent, hand a degraded app to the On-call Agent, or ask the Promotion Advisor for a pre-promotion risk verdict; relay every proposed action for human approval and read back what the native agent actually did. |

## Layout

```text
akuity/
├── .claude-plugin/plugin.json   # Claude Code manifest and platform endpoint (US default, EU via --config)
├── .codex-plugin/plugin.json    # Codex manifest
├── codex.mcp.json               # Codex platform endpoint (US)
├── references/                  # facts shared by every skill
│   ├── endpoints-and-auth.md    # the two endpoint kinds, login flows, enablement gates
│   └── manifest-kinds.md        # which resource kinds go to which endpoint, apply semantics
└── skills/
    ├── onboard/
    │   ├── SKILL.md             # the flow: what to ask, the dependency order, boundaries
    │   └── references/
    │       ├── instance-access.md
    │       ├── pipeline-facts.md
    │       └── troubleshooting.md
    └── delegate/
        ├── SKILL.md             # the approval discipline, the scenarios, the pitfalls
        └── references/
            ├── context-selection.md
            └── reading-conversations.md
```
