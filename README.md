# Akuity Agent Plugins

Agent workflows and client configuration for the [Akuity Platform](https://akuity.io)'s hosted MCP service. This repository is a plugin marketplace for Claude Code and Codex. It ships one plugin, `akuity`, which connects the platform MCP endpoint (US by default) and adds maintained skills for workflows that need sequencing, platform knowledge, or a consistent approval flow.

## Recommended installation

Installing the plugin gives the agent both the Akuity MCP tools and the maintained workflow skills. It configures `https://akuity.cloud/mcp` by default. In Claude Code, EU organizations can select `https://eu.akuity.cloud/mcp` at install time. The plugin does not configure self-hosted, non-production, or direct instance endpoints, or the EU endpoint for Codex.

### Claude Code

```text
/plugin marketplace add akuity/agent-plugins
/plugin install akuity@akuity
```

EU organizations install from a shell with the EU endpoint instead:

```shell
claude plugin install akuity@akuity --config endpoint=https://eu.akuity.cloud/mcp
```

Then run `/mcp`, select `akuity`, and authenticate in the browser.

### Codex

```shell
codex plugin marketplace add akuity/agent-plugins
codex plugin add akuity@akuity
```

Then start a new session and authenticate the `akuity` MCP server when prompted.

Installing the plugin is the trust decision: its MCP server starts automatically without a separate per-server approval prompt. The plugin is optional; users can connect an MCP client manually without installing it. See the [plugin README](plugins/akuity/README.md#install-and-authenticate) for manual connection and automation guidance.

## Requirements

- There is no customer-side switch. If **MCP Access** is unavailable in the portal, contact an Akuity representative or support.
- An Organization Owner or a custom role with Organization update permission has enabled the platform endpoint under **Organization Settings → MCP Access**.
- MCP access is enabled for every Argo CD or Kargo instance the platform endpoint should reach.
- The plugin configures the US endpoint by default. In Claude Code, EU organizations pass `--config endpoint=https://eu.akuity.cloud/mcp` at install time. EU organizations using Codex, and self-hosted or non-production platforms on any client, must use the manual client commands in the [plugin README](plugins/akuity/README.md#other-endpoints-and-automation).

The complete endpoint model, enablement gates, and login flows are documented in [`plugins/akuity/references/endpoints-and-auth.md`](plugins/akuity/references/endpoints-and-auth.md).

## What's inside

| Plugin | Skill | What it does |
| --- | --- | --- |
| [`akuity`](plugins/akuity/) | `akuity:onboard` | Run onboarding end to end or only for the missing parts: provision Argo CD and Kargo instances, connect clusters, deploy an app to every environment, and build a Kargo promotion pipeline. |
| | `akuity:delegate` | Delegate work to Akuity Intelligence: talk normally with the Deployment Agent, hand a degraded app to the On-call Agent, or ask the Promotion Advisor for a pre-promotion risk verdict, with every proposed action relayed for human approval. |

In Claude Code, invoke a skill directly with `/akuity:onboard` or `/akuity:delegate`, or describe the outcome and let Claude choose. In Codex, describe the desired outcome and let Codex choose the bundled skill.

## Layout

```text
agent-plugins/
├── .agents/plugins/marketplace.json       # Codex marketplace catalog
├── .claude-plugin/marketplace.json        # Claude Code marketplace catalog
└── plugins/
    └── akuity/
        ├── .claude-plugin/plugin.json      # Claude Code manifest and platform endpoint
        ├── .codex-plugin/plugin.json       # Codex manifest
        ├── codex.mcp.json                  # Codex platform endpoint (US)
        ├── skills/
        └── references/
```

Plugin releases use the version in each client manifest so installed clients can pick up updates.
