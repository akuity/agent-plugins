# Instance access: SSO vs admin account

This is how an instance gets the login people use for its UI, CLI, or direct MCP endpoint. Kargo also needs this authentication before Project, Stage, Freight, and Promotion tools are available. **Setting it up is always the user's own action** in the portal UI or their own shell. The agent must never ask for a password, token, or client secret in the conversation.

The products differ in what this setup gates. **Argo CD** platform tools work without an instance admin account or OIDC. **Kargo** Project, Stage, Freight, and Promotion tools require an admin account or OIDC. Platform lifecycle reads and `apply_kargo_instance` remain available, so onboarding can still create the instance, Project, and Warehouse. The user can configure authentication at creation or later. The flow first requires it when reading Freight in step 6.

An instance MCP endpoint offers interactive login when the instance has an identity provider configured or password authentication enabled. Without SSO, it serves a password login form for the instance's admin or Argo CD local account. A fresh instance may have neither configured, so set up an account in the portal instead of trying to retrieve a Kubernetes bootstrap secret.

## SSO mode

Use when the user has an IdP wired into the instance (or is ready to wire one).

- Argo CD uses the instance's Dex or OIDC configuration. Kargo uses its OIDC configuration. Both can be managed declaratively or from the instance settings in the portal.
- **Dex-backed instances** accept loopback callbacks on the `localhost` hostname. They do not accept `127.0.0.1`, `::1`, custom schemes, or hosted clients.
- **Direct OIDC** requires the identity provider administrator to register `https://ARGO_CD_HOSTNAME/mcp/oauth/callback` (Argo CD) or `https://KARGO_HOSTNAME/mcp/oauth/callback` (Kargo) on the public OAuth client shown under **Settings → MCP Access**. Individual MCP client callbacks do not need to be registered, and the issuer must use HTTPS.
- If interactive sign-in is unavailable, the user can register the direct endpoint with a native instance token from their own shell. The token instructions are in steps 4 and 5 below.
- Wiring a corporate identity provider needs client details from the user. If they do not have them ready, admin-account mode can cover initial setup.

## Admin account mode (day-0 default)

**1. The user enables the account and sets the password in the portal — the default path.** No secret exists anywhere near the conversation this way:

- Argo CD: the instance's portal settings → **System Accounts** — enable the admin account and set its password there.
- Kargo: the instance's portal settings → **System Accounts** — enable the admin account and set its password there. OIDC is under **OIDC Config** instead.

The password is theirs alone; nothing about it reaches you. Point them at the page, wait for them to say it is done, and verify by outcome (step below).

**Declarative fallback, opt-in only.** A user who prefers their shell to the UI can configure the same thing through the platform endpoint's config documents — at creation or any time after (a config-only apply patches the existing instance; nothing requires it to ride with the instance manifest):

- Argo CD — in the instance's `argocd-cm` data: `accounts.admin: "login,apiKey"` and `accounts.admin.enabled: "true"`; in `argocd-secret` stringData: `admin.password: <bcrypt-hash>`. Never set `admin.passwordMtime` — it is restricted and the apply fails.
- Kargo — in `kargo-cm` data: `adminAccountEnabled: "true"`; in `kargo-secret` stringData: `adminAccountPasswordHash: <bcrypt-hash>`.

Only offer this when the user asks for it, and hold the password discipline: the plaintext never enters the chat and never a command line either (process arguments and shell history capture it). The user picks a strong random password, keeps it, and generates the bcrypt hash in their own shell with `htpasswd` prompting for the password (no `-b` — that flag would put the password on the command line):

```
htpasswd -nBC 10 "" | tr -d ':\n' | sed 's/$2y/$2a/'
```

The prompt reads from the terminal, so the pipe still receives only the hash. Even then, prefer that **they** run the apply with the hash in their own shell (`akuity kargo apply` / `akuity argocd apply` with the config documents) rather than handing the hash to you: a hash is one-way, but it still persists in whatever it transits, and a weak password makes it crackable. If the user types the plaintext password into chat, stop — a password that entered the conversation is exposed; have them pick a new one and set it in the portal instead.

**2. Verify the config landed** — whichever path they used. For Kargo, the platform endpoint's instance read shows `apiCm.adminAccountEnabled: true`; absent or false means it has not landed yet. (The same read shows the admin token TTL, default 24h.) A cheap Kargo read tool succeeding is the same answer from the other side.

**3. Connect interactively — the default.** With the admin account landed and the instance healthy, the endpoint serves a browser password login: the user registers it plainly (`claude mcp add --transport http <name> https://<hostname>/mcp`), runs `/mcp`, chooses authenticate, and enters the admin username (`admin` on both products) and the password only they know — typed into the browser form, never into chat. The session is the instance's native token and carries no refresh token: when it expires (default 24h) the login simply runs again. Configured SSO takes precedence — an instance with SSO never shows the password form.

**4. For automation, a headless client, or when the browser flow is unavailable, the user mints a token in their own shell instead**, once the instance is healthy. The MCP endpoint accepts any token the instance itself accepts — it validates the bearer against the instance's live API. Tokens are minted by the instances themselves, not by the platform or the akuity CLI — which is why this step is the user's. So:

- Argo CD: `argocd login <argocd-hostname> --username admin --grpc-web`, then `argocd account generate-token --account admin --grpc-web` prints an API token (the `apiKey` capability configured above is what allows this; the session token the login stores works too, it just expires sooner). A `Failed to invoke grpc call. Use flag --grpc-web` warning during login is benign — the login still succeeds; passing `--grpc-web` on both commands avoids it.
- Kargo: `kargo login https://<kargo-hostname> --admin` — the CLI stores the session token under the `bearerToken` key in `~/.config/kargo/config` (YAML); that token is the bearer. Admin tokens default to a 24h TTL.

**5. The user registers the endpoints with the token header, in their own shell** — the token never enters the conversation. Command substitution keeps it out of the pasted command too:

```
claude mcp add --transport http argocd-<name> https://<argocd-hostname>/mcp \
  --header "Authorization: Bearer $(argocd account generate-token --account admin --grpc-web)"
claude mcp add --transport http kargo-<name> https://<kargo-hostname>/mcp \
  --header "Authorization: Bearer $(awk '/^bearerToken:/ {print $2}' ~/.config/kargo/config)"
```

Have them check each substitution prints a value first (run the inner command alone): an empty substitution still registers the server, without any error, and only surfaces later as 401s. Registration itself has client-side traps — directory scope, mid-session visibility — covered under "MCP client registration gotchas" in `../../../references/endpoints-and-auth.md`.

Tokens and sessions expire: on a header-registered endpoint a later 401 means re-mint and re-register; on a browser-login session it means reconnect via `/mcp` and log in again — there is no refresh token. Neither is a platform failure. And never ask the user to paste the password or token into chat — hand them the commands above with placeholders and wait for them to say the endpoints are registered.
