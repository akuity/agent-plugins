# Akuity MCP endpoints and authentication

The Akuity platform serves MCP on two kinds of endpoint, and they differ by **whose access they authenticate**, not by which resources they can reach. The platform endpoint authorizes an organization principal: it is the only home of platform objects, and it also reaches the resources inside instances, so a caller with organization access can run a whole workflow there end to end. Each instance's own endpoint authorizes an instance user under that instance's RBAC and reaches only that instance's resources — for someone who holds instance access but no organization access, it is the only surface there is. Pick the endpoint by the access the caller holds, not by the resource kind.

## The platform endpoint

One URL for the region that hosts the organization—`https://akuity.cloud/mcp` for US organizations or `https://eu.akuity.cloud/mcp` for EU organizations—and the primary surface for everything: platform objects (Argo CD and Kargo instances, clusters, Kargo agents, and their health) **and** the resources inside instances. The plugin configures the US endpoint by default; in Claude Code the EU endpoint is selected at install time with `--config endpoint=https://eu.akuity.cloud/mcp`, and in Codex another region is connected manually. The composite apply tools carry Applications, Kargo projects, warehouses, stages, and credentials alongside the instance manifest (see `manifest-kinds.md`), and the endpoint also serves the application sync, freight, promotion, and delete tools. Every organization-level call names the organization and, where relevant, the instance explicitly.

- **Login**: the server requires authentication before its tools work. The client opens a browser OAuth flow against Akuity Cloud, including the organization's SSO, and authorizes the user under the organization's own RBAC.
- **OAuth callback**: native clients may use any loopback callback port. Hosted HTTPS and custom-scheme clients are also supported after a one-time consent screen that names the redirect destination. The plugin does not pin a callback port.
- **Automation**: CI uses an Akuity API key in an `Authorization: Bearer <key>` header instead of the browser flow.
- **Organization**: an API key pins the organization. An interactive login with exactly one org membership binds automatically. Otherwise the organization must be supplied — there is currently no tool that lists organizations, so if it is not in the session context, ask the user for their organization id.

## Instance endpoints

Each provisioned Argo CD and Kargo instance can serve its own MCP endpoint on its own hostname: `https://<instance-hostname>/mcp`, the same host that serves that instance's UI, API, and CLI. It reaches the in-instance resources — Argo CD Applications and repository credentials, Kargo projects, credentials, warehouses, stages, promotions — but never platform objects, and it authorizes as an **instance user** under the instance's own RBAC. This is the working surface for people whose access lives inside the instance: an app team with Argo CD or Kargo roles but no organization membership has no platform endpoint at all, and creates, syncs, and deletes apps or runs promotions here. A caller who does hold organization access can do the same in-instance work through the platform endpoint instead, with explicit instance ids, and needs an instance endpoint only when instance-scoped authorization is the point. No workflow in this plugin requires connecting one.

- **Registration**: read the instance's hostname from the platform endpoint's read tools, then register `https://<hostname>/mcp` manually with the MCP client. The plugin configures only the platform endpoint; client-specific commands are in the plugin README.
- **Login**: the endpoint advertises the instance's own login. Dex-backed instances accept loopback callbacks on the `localhost` hostname. With direct OIDC, register `https://<instance-hostname>/mcp/oauth/callback` on the public OAuth client named in the instance's **Settings → MCP Access** page; individual MCP client callbacks do not need registration. Either way, calls run as an **instance user**, not an Akuity organization user.
- **No SSO still logs in when password auth is enabled**: an instance without an identity provider serves a password login form of its own—for Argo CD's admin or local accounts with the `login` capability, and for Kargo's admin account—so a loopback client can use the browser flow. The password is verified by the instance itself, the session is the instance's native token, and configured SSO always takes precedence. These sessions carry no refresh token; when the native token expires (default 24h), reconnect and log in again. An instance with neither SSO nor an enabled admin/local account advertises no login. For automation, register a bearer token from the user's own shell so the token never enters the conversation. The onboarding skill's `instance-access.md` reference covers enabling the admin account, the password discipline, and minting tokens.
- **Authorization**: every call runs under the instance's own RBAC (Argo CD `policy.csv`, Kargo's Kubernetes RBAC). A permission denial is the instance answering, not a transient error — report it, do not retry.
- **No instance argument**: the hostname pins the instance, so instance-side tools take no instance id.

## Enablement gates

MCP access is gated at three levels, each owned by someone different. Knowing who owns which gate is the difference between a fix and a dead end:

- **MCP Access availability:** there is no customer-side switch. If **MCP Access** is unavailable in the portal, contact an Akuity representative or support.
- **The platform endpoint, org-wide** — an Organization Owner or custom role with Organization update permission enables **Organization Settings → MCP Access → "Enable platform MCP endpoint"**. A platform-endpoint call rejected with "the MCP server is not enabled for this organization" means this toggle is off or MCP access is unavailable for the organization. The same page manages MCP access for all instances in bulk and renders copy-paste connection commands.
- **Per-instance access** — set at creation via `mcpServer.enabled: true` in the instance spec (under `spec.instanceSpec` for Argo CD, `spec.kargoInstanceSpec` for Kargo), or afterward in the instance's own **Settings → MCP Access → "Allow access to this instance"** (instance-update permission required). The one setting has **two effects**: it serves the instance's own `/mcp` endpoint, and it allows the platform endpoint's in-instance tools to target the instance. Instances created through the platform MCP endpoint get it enabled automatically. Without it the instance's own `/mcp` returns **404**, and a platform-endpoint in-instance call is denied with "not permitted to use this tool on the instance it names, or MCP is not enabled for that instance" — configuration answers, not outages.
- **Kargo only: instance authentication:** configure an admin account or OIDC before using Freight, Stage, Project, and Promotion tools or the instance's direct endpoint. Platform lifecycle reads and `apply_kargo_instance` remain available, so this setup does not have to block instance creation. The instance owner can enable the admin account and set a password under **Settings → System Accounts**, configure **Settings → OIDC Config**, or apply the config declaratively. The onboarding skill's `instance-access.md` covers these options.

## MCP client registration gotchas

These are MCP client behaviors, not Akuity ones, and each looks like a platform failure until you know it:

- `claude mcp add` without `--scope` registers into the **current directory's** local scope — run it from the directory the session runs in, or the session never sees the server. The command's output names the project directory it wrote to; check it.
- A server added mid-session stays invisible until the session restarts, even though `claude mcp list` shows it registered — restart the session, then authenticate or connect via `/mcp`.
- After a dropped connection (network change, VPN), the tools do not come back on their own even once `claude mcp list` shows the server connected — run `/mcp` and reconnect it.
- Codex loads newly installed plugin skills and MCP servers in a new session; start one after `codex plugin add` before diagnosing missing tools.
- When registering with a token header built by command substitution, verify the substitution prints a value first: an empty one registers the server without error and only surfaces later as 401s.

## Failure discipline

If a login or `tools/list` fails on a freshly registered instance endpoint, stop and say so — nothing that depends on that endpoint can work. If a call fails and the error does not tell you enough to fix it, say that plainly and stop; two attempts with no new information is a stop condition.
