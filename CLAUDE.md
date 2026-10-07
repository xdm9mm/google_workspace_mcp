# This fork — Eddington agent context

This is the account owner's personal fork of [taylorwilsdon/google_workspace_mcp](https://github.com/taylorwilsdon/google_workspace_mcp),
deployed to give AI agents access to a personal Google account (Gmail, Drive,
Calendar) as part of the "Eddington" home-lab project. `upstream` is the original
project; `origin` (this fork) is where the Eddington-specific work happens.

This file is new — everything else in the repo is upstream's, including `README.md`,
which is left alone so `git fetch upstream && git merge upstream/main` stays conflict-free.

## Status: LIVE as of 2026-10-06

**Decided: self-host it.** Built on `hLaptop-L2`, pushed to the private registry, pulled
and running on `hServer-L1` (`/srv/shared/google-workspace-mcp` — real LAN address in
the private eddington-shared repo's `mcp/servers.md`, not here), verified against the
real account: Gmail search, and a real Calendar `list_calendars` call returning live
data (including an imported external calendar) both confirmed. Registered with Claude
Code at user scope on `hLaptop-L2`.

## Read this first

The full decision record lives in the sibling repo `eddington-shared` (private, checked
out alongside this one — `../eddington-shared` if you're on the same machine as Mark's
other Eddington work):

- `notes/2026-09-23-outlook-agent-access.md`, "Related: same idea for Google" section —
  why this candidate surfaced, the concerns list, and the hosted-vs-self-host trade-off
  that got resolved 2026-10-06. (Yes, it's filed under the Outlook note — the Google idea
  was added there, not split out, since it was still an open question when written.)
- `TODO.md` — "MCP & agents" section, Done list.
- `mcp/servers.md#google-workspace-mcp-gmail--drive--calendar` — connection details,
  same hosting/transport pattern as `ha-mcp`/`unifi-network-mcp`/`synology-mcp`/
  `ap-imap-mcp`/`outlook-mcp`.
- `README.md`'s "Local vs External" section — this server is **Local** (runs on
  `hServer-L1`) even though Gmail/Drive/Calendar are **External**.

## Hard rule: keep this fork generic

Nothing committed here should include real LAN IPs, the account owner's personal
account identifiers (email address, etc.), or any secret/token. That detail belongs in
`eddington-shared` (private) or in local, gitignored `.env` files (see `.env.example`),
referenced by name only. This fork is public on GitHub (forks of a public repo can't be
made private), so treat everything committed here as world-readable.

**No agent working in this repo should ever generate, request, hardcode, or otherwise
handle the account owner's real Google password, OAuth client ID/secret, or stored
refresh token.** Those live only in the password manager, local gitignored `.env`
files, and the `store_creds` Docker volume on the deploy host — never in a file
committed here.

## What's already built here (this fork's own additions, not upstream's)

Unlike the Outlook/AP forks, **no Dockerfile or entrypoint changes were needed** —
upstream already ships a working `Dockerfile` with native streamable-HTTP support, and
its own `docker-compose.yml`/`client_secret.json` pattern was swapped for pure env vars
(`GOOGLE_OAUTH_CLIENT_ID`/`_CLIENT_SECRET`/`_REDIRECT_URI`, consumed by
`auth/google_auth.py`'s `load_client_secrets_from_env()`) — no file to mount or keep in
sync.

- `docker-compose.yaml` / `docker-compose.override.yaml` — same base-declares-`image:`,
  override-adds-`build:` split used by `ap-imap-mcp`/`outlook-mcp`, so hServer-L1 only
  ever pulls the registry image, never builds from source. `docker-compose.yaml` mounts
  `/app/store_creds` as a named volume for the encrypted-at-rest-on-disk token store.
- `.env.example` — documents every env var this deployment actually uses; see its
  comments for the exact reasoning behind each choice. The short version:
  - `WORKSPACE_MCP_PERMISSIONS=gmail:send drive:readonly calendar:full` — Gmail capped
    at the `send` tier (includes `gmail.modify` for trashing/labels, never the
    `mail.google.com` full-access scope, so permanent delete isn't possible regardless
    of what an agent is asked to do — matches the Outlook/AP "no permanent delete"
    pattern). Drive is `readonly` (no concrete write use case was ever decided). Calendar
    is `full` (the vision note's "calendar interface" idea needs real read/write
    scheduling; calendar has no readonly-with-write middle tier, and calendar data is
    lower-stakes/more recoverable than email or files). **Do not also set
    `WORKSPACE_MCP_TOOLS`** — `main.py` treats `--tools`/`WORKSPACE_MCP_TOOLS` and
    `--permissions`/`WORKSPACE_MCP_PERMISSIONS` as mutually exclusive and hard-exits if
    both are set.
  - `WORKSPACE_MCP_HOST=0.0.0.0` — legacy streamable-HTTP (no `MCP_ENABLE_OAUTH21`) has
    no per-request MCP-level auth, so `main.py`'s `resolve_bind_host_for_transport()`
    defaults to `127.0.0.1` unless told otherwise. Set explicitly to the same
    LAN-is-the-perimeter trust model already used by every other server in this project.
  - `GOOGLE_MCP_CREDENTIALS_DIR=/app/store_creds` — **required**, not cosmetic. Without
    it, upstream's credential store defaults to `~/.google_workspace_mcp/credentials`
    (the container user's home), which isn't inside `docker-compose.yaml`'s mounted
    volume — tokens would silently vanish on every container recreation. Found by
    inspecting `docker exec`'d container filesystem after the first real OAuth grant
    landed somewhere unexpected; see the shared note's Deployment section.

## The actual auth flow (why this was harder than Outlook/AP)

Google rejects OAuth redirect URIs on any host that isn't `localhost`/a loopback address
or a public-TLD domain — confirmed hands-on in Cloud Console (a bare LAN IP redirect was
flatly refused: "must end with a public top-level domain"). So unlike
Outlook's device-code flow or IMAP's password-only auth, there's no way for
`hServer-L1` to run its own first-time (or weekly re-auth) consent directly. The actual
flow, confirmed working 2026-10-06:

1. Run this image **locally** (`hLaptop-L2` or any machine with a browser), with
   `WORKSPACE_MCP_BASE_URI=http://localhost`, `WORKSPACE_MCP_PORT=8000`,
   `GOOGLE_OAUTH_REDIRECT_URI=http://localhost:8000/oauth2callback` (this exact URI is
   registered on the OAuth client — see Cloud Console config below).
2. Call the `start_google_auth` MCP tool (`service_name`, `user_google_email`) to get
   the authorization URL; open it in a browser, sign in as the account owner, click
   through the expected "Google hasn't verified this app" warning (Testing-mode apps
   always show this, regardless of how careful the setup was), and approve the scopes.
3. The local container's own `/oauth2callback` (same FastAPI app as the MCP transport
   in streamable-http mode — confirmed in `auth/oauth_callback_server.py`) receives the
   code and stores credentials to `GOOGLE_MCP_CREDENTIALS_DIR`.
4. `docker cp` the resulting `<email>.json` out of the local container and into
   `hServer-L1`'s `google-workspace-mcp-data` volume (`docker cp` the file into
   `/app/store_creds/` inside the running production container, then `docker exec -u
   root ... chown app:app` it — the file arrives owned by the copying user, not the
   container's `app` user), then restart the production container so it picks up the
   new token from disk.

**This whole dance repeats roughly every 7 days** — see "Known limitation" below.

## Known limitation: 7-day refresh-token expiry (accepted, not solved)

**Decided 2026-10-06: accept it, don't try to dodge it.** Gmail read scopes
(`gmail.readonly`, bundled into the `send` tier) and Drive read (`drive.readonly`) are
both Google **Restricted** scopes (confirmed directly in Cloud Console's Data Access
page, not just from memory of Google's docs) — moving the OAuth consent screen from
Testing to Production with Restricted scopes requires Google's formal
verification/CASA security assessment, which is weeks of process built for apps with
real external users, not a personal single-user tool. There is no config flag or
narrower-but-still-useful scope selection that avoids this for an agent that needs to
actually read Gmail. (`gmail.send`, `drive.file`, and Calendar scopes are only
"Sensitive," not Restricted, and could go to Production unverified — but the point of
this server is Gmail/Drive *read* access, so staying Restricted is unavoidable if it's
going to do anything useful.)

Practical effect: the refresh token obtained above stops working after ~7 days (Testing
mode's OAuth user cap / token lifetime policy), at which point every tool call will fail
auth and the local-auth-then-copy-to-server dance above has to be repeated. No
auto-renewal exists. If this becomes too much friction in practice, the real fix is
Google's verification process, not a workaround — revisit then.

## Google Cloud project

- **Project:** `eddington-google-mcp` (ID `celtic-current-510901-r1`) — dedicated
  project, separate from `eddington-apps` (which has its own OAuth client for the
  `myhome` app's calendar feature). Least-privilege separation, matching this project's
  dedicated-account pattern elsewhere (`agent`/`mcp-service` DSM accounts).
- **APIs enabled:** Gmail, Google Drive, Google Calendar (not Docs/Sheets/Slides/Forms/
  Chat/Tasks/Contacts/AppScript/Custom Search — this fork supports those too, but they
  were never part of the scoped decision).
- **OAuth consent screen:** External, Testing (see "Known limitation" above for why it
  stays in Testing). Test user: the account owner's own Google account.
- **OAuth client:** `Eddington Google Workspace MCP`, type Web application. A LAN-IP
  redirect URI was attempted and rejected by Google (see "The actual auth flow" above);
  only the `localhost` one actually works as a live redirect target.
- **Data Access scopes registered:** `gmail.readonly`, `gmail.labels`, `gmail.modify`,
  `gmail.compose`, `gmail.send`, `drive.readonly`, `calendar`, `calendar.events` (the
  full set implied by `gmail:send`/`drive:readonly`/`calendar:full`).

## Still open

- **Re-auth cadence**: the 7-day dance above is manual and undocumented as a recurring
  task anywhere yet — consider adding it to `../TODO.md` if it becomes a real recurring
  chore, or revisit Google verification if it's too much friction.
- Only registered with Claude Code on `hLaptop-L2` so far, not Mark-PC or other
  machines.
- No draft-then-send gate on `send_gmail_message` (sends immediately) — same accepted
  gap as the Outlook/AP servers.
