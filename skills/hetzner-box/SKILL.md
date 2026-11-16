---
name: hetzner-box
description: Hosting contract for the shared Hetzner box (one Caddy, many independent project repos). Use before changing a hosted project's Caddy fragment, compose networks or published ports, secrets/env file, or deploy; or when hosting a new project on the box.
---

# Hetzner box: hosting contract

One public machine serves several **independent projects**. Each project is its
own repo (any account or org, public or private) and owns everything about
itself, including its vhosts. This repo
(`Maxalon/hetzner-box`) owns only what is shared: the one Caddy, the snippets
projects may use, the `personal.slice`, and the `box` helper. **No project's
config lives here, and no project knows another exists.**

Hosting a new project? Follow [NEW-PROJECT.md](NEW-PROJECT.md) step by step;
the contract below is what those steps produce.

## The shape

```
Main IP  ─ Cloudflare (orange) ─┐
Tailnet IP ─ grey-cloud ────────┴─► Caddy (this repo, edge/)
                                     └─ import /srv/edge-sites/*.caddy
                                          ├─ verdantmanor.caddy ─► network edge
                                          ├─ darts.caddy        ─► network edge-darts
                                          └─ <name>.caddy       ─► network edge-<name>
Game IP (DNS-only) ──────────────────► raw TCP/UDP ports published by projects
```

- **Main IP**: only Caddy listens there (`:80`, `:443`). Every record for it is
  orange-cloud, so Cloudflare hides it. A DNS-only record on the main IP would
  publish it; raw ports therefore never go there.
- **Tailnet IP**: Caddy listens there too, for `tailnet_only` hosts.
- **Game IP** (Hetzner additional IPv4): raw game ports only. Caddy never binds
  it, so it can't be tied to the web projects.

## Host layout

| Path | What | Rule |
|---|---|---|
| `/srv/hetzner-box` | clone of this repo | pulled by hand |
| `/srv/projects/<name>` | clone of each project's repo (a sparse clone of just the fragment is fine; private repos use a read-only deploy key) | **nothing secret**: the whole tree is mounted read-only into Caddy |
| `/srv/secrets/<name>.env` | that project's secrets + host values (e.g. `GAME_IP`) | mode 600, never inside a clone |
| `/srv/edge-sites/<name>.caddy` | symlink → the project's fragment | the symlink is the on/off switch |

`<name>` is one lowercase slug used everywhere: clone dir, compose project
(`-p <name>`), network `edge-<name>`, secrets file, fragment symlink.

Current projects:

| name | repo | fragment | network | deploys via |
|---|---|---|---|---|
| `verdantmanor` | Kontrollzentrale/verdantmanor (org) | `edge/verdantmanor.caddy` | `edge` (historical name) | its own CI |
| `darts` | Kontrollzentrale/Dart-Track-Server (org) | `caddy/darts.caddy` | `edge-darts` | its own CI |
| `archipelago` | planned | `caddy/archipelago.caddy` | `edge-archipelago` | `box deploy` |

## The contract: what a project repo has

1. **`caddy/<name>.caddy`**: the project's site blocks, complete. Reuse the box
   snippets (`cloudflare_tls`, `basic_headers`, `tailnet_only`, defined in
   `edge/conf/Caddyfile`). Prefix any snippet of its own with `<name>_`: all
   fragments share one namespace, and a duplicate name breaks every project's
   reload.
2. **`compose.yml`** (box-deployed projects; CI-deployed ones apply the same
   rules in their own compose file), where each service Caddy reaches:
   - joins the external network `edge-<name>` with a stable alias
     (`<name>-app`), and the fragment proxies to that alias. That network is
     shared with Caddy only, so projects can't reach each other.
   - sets `cgroup_parent: personal.slice` (low CPU and IO weight, hard memory
     cap; Verdant Manor wins under contention).
   - reads secrets via `env_file: /srv/secrets/<name>.env`.
   - publishes **no** HTTP ports. Raw TCP/UDP is published as
     `"${GAME_IP:?}:<port>:<port>"`, never on a bare port (that binds every IP,
     the main one included).
3. **`.claude/skills/hetzner-box/SKILL.md`**: the stub pointing here, from
   `templates/project/`.
4. **CI**: validates the fragment against the real box config with the reusable
   workflow (see NEW-PROJECT.md, step 5).

The box side of a project is exactly two things: its network listed in
`edge/compose.yml`, and the symlink that `box enable` creates.

## Deploys

A project's **stack** ships one of two ways; its **fragment** always ships the
same way, from the clone in `/srv/projects/<name>`.

- **Box-deployed** (a manual `git pull`): `box deploy <name>` pulls the clone,
  runs `docker compose -p <name> up -d --build` from it, then checks and
  reloads Caddy.
- **CI-deployed** (projects in the org with self-hosted runners: verdantmanor,
  darts): the project's own pipeline deploys the stack wherever it always has.
  The fragment follows with `box refresh <name>` (pull + check + reload, never
  compose), run by hand or as the deploy job's last step.

`bin/box` (on the box, `/srv/hetzner-box/bin/box`):

```bash
box deploy <name>      # box-deployed: pull, compose up, reload
box refresh <name>     # CI-deployed: pull the fragment, reload
box reload             # after pulling this repo
box enable <name> caddy/<name>.caddy
box disable <name>
```

`reload` runs `caddy adapt` first. A fragment that doesn't adapt leaves the
running config untouched, so a bad push is a no-op rather than an outage. It
also means one broken fragment blocks *everyone's* reloads until it is fixed or
disabled.

## Gotchas

- **Cloudflare token scope.** A new zone in any fragment must be added to the
  token's scope (Zone → DNS → Edit) in `/srv/secrets/hetzner-box.env`.
  Otherwise its certs fail to issue *silently*.
- **Websockets through Cloudflare** work on 443. Non-HTTP game traffic can't go
  through Cloudflare at all: give it a DNS-only record → game IP.
- **`tailnet_only` hosts** need a grey-cloud record → the box's Tailscale IP, and
  Tailscale with `--snat-subnet-routes=false`, or the gate 404s everyone.
- **Never bind-mount a single file** into Caddy. A file mount pins the inode,
  so after `git pull` the reload silently applies the old config.
- **Changing the network list** in `edge/compose.yml` needs `box edge-up` (a
  container recreate, a few seconds of downtime). `box enable` connects the
  running container live, so nothing waits on that.
- **A fragment must not set global options** (the `{ … }` block); those are box
  concerns and belong in `edge/conf/Caddyfile`.
