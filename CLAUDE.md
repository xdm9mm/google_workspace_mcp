# This fork — Eddington agent context

This is Mark's personal fork of [taylorwilsdon/google_workspace_mcp](https://github.com/taylorwilsdon/google_workspace_mcp),
kept as the vehicle for evaluating self-hosted Gmail/Google Workspace access for AI
agents, as part of his home-lab project, "Eddington." `upstream` is the original project;
`origin` (this fork) is where the Eddington-specific work happens.

This file is new — everything else in the repo is upstream's, including `README.md`,
which is left alone so `git fetch upstream && git merge upstream/main` stays conflict-free.

## Status: NOT DECIDED — this is exploratory

Unlike the parallel Outlook project, whether to build this at all is still open. Today
Google access (Gmail, Drive, Calendar) comes from hosted, cloud-provided MCP connectors —
this fork exists to evaluate the leading self-hosted candidate hands-on, not because
self-hosting has already been chosen. Don't treat anything below as a green light to
deploy — resolve the open decision first (see "Next steps").

## Read this first

The full decision record lives in the sibling repo `eddington-shared` (private, checked
out alongside this one — `../eddington-shared` if you're on the same machine as Mark's
other Eddington work):

- `notes/2026-09-23-outlook-agent-access.md`, "Related: same idea for Google" section —
  why this candidate surfaced, the concerns list, and the hosted-vs-self-host trade-off.
  (Yes, it's filed under the Outlook note — the Google idea was added there, not split out,
  since it was still an open question when written.)
- `TODO.md` — "MCP & agents" section tracks the self-host-or-not decision as an open item.
- `mcp/servers.md` — the hosting/transport pattern used for this project's other
  self-hosted MCP servers (`ha-mcp`, `unifi-network-mcp`, `synology-mcp`: Docker
  containers on a home server). If self-hosting is chosen, this should follow the same
  pattern; it isn't listed there since nothing is decided or deployed.
- `README.md`'s "Local vs External" section — a self-hosted server here would be **Local**
  even though Gmail/Drive/Calendar are **External**. The *current* hosted connectors are
  fully External (both the API and the MCP server itself run off Mark's hardware) — that's
  the actual trade-off this decision is about.

## Hard rule: keep this fork generic

Nothing committed here should include real LAN IPs/hostnames, Mark's personal account
identifiers, or any secret/token. That detail belongs in `eddington-shared` (private) or
in local, gitignored `.env` files, referenced by name only. This fork is public on GitHub
(forks of a public repo can't be made private), so treat everything committed here as
world-readable.

## Next steps for whoever picks this up

1. **Resolve the decision first**: is self-hosting actually worth it over the existing
   hosted connectors? Hosted needs no token custody, upkeep, or expiry handling;
   self-hosting trades that for Local-first placement, one agent-agnostic server, and
   control over guardrails. Write the outcome back into the shared note once decided —
   this repo isn't the place to record that decision, `eddington-shared` is.
2. **If self-hosting wins**, work through the concerns already flagged in the shared note:
   - **7-day refresh-token expiry**: this project's OAuth consent screen needs to be
     published to "In production" (not left in "Testing") to avoid weekly breakage.
     Confirm how Google's verification requirement applies to a personal-only,
     single-user app.
   - **Headless auth**: unlike Outlook's device-code flow, Google's redirect-based OAuth
     is awkward on a headless box — likely means completing consent once on a machine
     with a browser and copying the token over.
   - **Scope minimization**: Gmail's full-access scopes are "restricted." Request the
     narrowest scopes this project supports, and use its read-only/tool-filtered modes
     where available.
   - **Guardrails**: same requirement as the Outlook project — trash-not-permanent-delete,
     draft-then-send — enforced server-side since this must stay agent-agnostic.
   - **Paid-account gaps**: confirm which Workspace services (Chat, parts of Docs) this
     project needs that a plain personal Gmail account can't provide.
3. **Once/if deployed**: add the connection details to `eddington-shared/mcp/servers.md`
   and update `eddington-shared/TODO.md`.
