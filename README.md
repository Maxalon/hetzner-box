# hetzner-box

The shared parts of my Hetzner box: one Caddy, the snippets projects may use, a
low-priority `personal.slice`, and the `box` helper. Every hosted project
(Verdant Manor, Dart Track, Archipelago, …) lives in its own repo, wherever
suits it (Verdant Manor and Dart Track stay in the org for their self-hosted
deploy runners), and owns its own vhosts as a Caddy fragment. Nothing
project-specific lives here.

**The contract** (layout, what a project repo needs, deploys, gotchas) is
[`skills/hetzner-box/SKILL.md`](skills/hetzner-box/SKILL.md). Adding a project:
[`NEW-PROJECT.md`](skills/hetzner-box/NEW-PROJECT.md).

## For agents

Every hosted project commits the stub skill from
`templates/project/.claude/skills/hetzner-box/`, which points agents at the
contract. That works in cloud sessions too (repo skills load there; plugins
declared in settings don't). For local Claude Code you can also install this
repo as a plugin:

```bash
claude plugin marketplace add Maxalon/hetzner-box
```

```bash
claude plugin install hetzner-box@hetzner-box
```

## One-time setup: migrating from the verdantmanor-owned edge

Until now the edge ran from `/srv/verdantmanor_edge` (compose project
`verdantmanor_edge`), and its Caddyfile held every vhost. The cert volumes stay
the same, so nothing is re-issued. Downtime is the few seconds between stopping
the old Caddy and starting the new one.

**0. Before you start**, merge the verdantmanor PR that adds
`edge/verdantmanor.caddy` and promote it to `main`. Do **not** `git pull` in
`/srv/verdantmanor_edge` afterwards: that PR deletes the old edge files.

```bash
# 1. Layout
sudo mkdir -p /srv/projects /srv/edge-sites /srv/secrets
sudo chmod 700 /srv/secrets
git clone https://github.com/Maxalon/hetzner-box.git /srv/hetzner-box
sudo ln -s /srv/hetzner-box/bin/box /usr/local/bin/box

# 2. Secrets: the old token + email, plus the two IPs
install -m 600 /srv/hetzner-box/edge/.env.example /srv/secrets/hetzner-box.env
grep -E '^(CLOUDFLARE_API_TOKEN|ACME_EMAIL)=' /srv/verdantmanor_edge/.env.edge   # copy these over
tailscale ip -4                                                                   # TAILNET_IP

# 3. Verdant Manor's fragment: a sparse clone, edge/ only (same deploy key as before)
git clone --filter=blob:none --sparse --branch main <verdantmanor-repo-url> /srv/projects/verdantmanor
git -C /srv/projects/verdantmanor sparse-checkout set edge

# 4. Dart Track: its deploy now writes to /srv/projects/darts (see "Dart Track"
#    below). Create it for the deploy user, then merge/re-run its deploy:
sudo install -d -o darttrack-deploy -g darttrack-deploy /srv/projects/darts
#    ...after its Deploy run: ls /srv/projects/darts/caddy/darts.caddy

# 5. Enable the fragments (links only; the new Caddy isn't up yet, so no reload)
ln -s /srv/projects/verdantmanor/edge/verdantmanor.caddy /srv/edge-sites/verdantmanor.caddy
ln -s /srv/projects/darts/caddy/darts.caddy /srv/edge-sites/darts.caddy

# 6. The slice
sudo box install-slice
docker info --format '{{.CgroupDriver}}'   # must print systemd for cgroup_parent: personal.slice

# 7. Swap the Caddy
docker compose -f /srv/verdantmanor_edge/edge/compose.yml -p verdantmanor_edge down
box edge-up
box status
```

Then check: `curl -sSI https://verdantmanor.cards` → 200,
`curl -sSI https://darts.kontrollzentrale.de/...` → the API, and from a tailnet
device `https://cms.verdantmanor.cards/admin/login` → 200. Once all is well:
`rm -rf /srv/verdantmanor_edge /srv/dart-track-server`.

**Rollback** (the old clone is untouched until you delete it):
```bash
docker compose -f /srv/hetzner-box/edge/compose.yml --env-file /srv/secrets/hetzner-box.env down
docker compose -f /srv/verdantmanor_edge/edge/compose.yml -p verdantmanor_edge --env-file /srv/verdantmanor_edge/.env.edge up -d
```

### Dart Track

Dart Track stays in the org and keeps deploying from its self-hosted runner.
Its Deploy workflow copies `docker-compose.yml` and `caddy/darts.caddy` into
`/srv/projects/darts` (formerly `/srv/dart-track-server`) and brings the stack
up there. Its compose file pins the project name `dart-track-server`, so the
move keeps the same containers and the same data volume. When its fragment is
enabled (`/srv/edge-sites/darts.caddy` exists), the job ends with
`box reload`. Before that, it skips the reload, so it can be merged ahead of
the cutover.
