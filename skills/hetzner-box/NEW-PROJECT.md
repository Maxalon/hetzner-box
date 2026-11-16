# Hosting a new project on the box

Steps 1–5 happen in the **project's** repo. Steps 6–8 are box changes. The
user runs them; give them the exact commands. Done means `curl -sSI
https://<host>` returns the project's response and `box status` lists the
fragment.

1. **Pick `<name>`**: a lowercase slug, unique on the box (check
   `edge/compose.yml` networks in `Maxalon/hetzner-box`).
2. **Copy the templates** from `templates/project/` in `Maxalon/hetzner-box`
   into the project repo, replacing every `<name>`:
   - `compose.yml` → repo root, adapted to the project's services
   - `caddy/<name>.caddy` → its real host(s) and upstream alias/port
   - `.claude/skills/hetzner-box/SKILL.md` → the stub
3. **Classify each exposed service.**
   - HTTP/websocket: a site block in the fragment, orange-cloud DNS → main IP,
     no published port.
   - Raw TCP/UDP: a `"${GAME_IP:?}:<port>:<port>"` port, DNS-only record → game
     IP, and `GAME_IP=` in the secrets file.
   - Friends/admin-only web UI: add `basicauth` or `import tailnet_only` (the
     latter needs grey-cloud → Tailscale IP).
4. **List the secrets** the project needs as an `.env.example` in its repo (no
   values). The real file is `/srv/secrets/<name>.env`.
5. **Add CI** that checks the fragment against the real box config:
   ```yaml
   # .github/workflows/caddy.yml
   on:
     pull_request: { paths: ['caddy/**'] }
     push: { paths: ['caddy/**'] }
   jobs:
     caddy:
       uses: Maxalon/hetzner-box/.github/workflows/validate-fragment.yml@main
       with:
         name: <name>
         fragment: caddy/<name>.caddy
   ```
6. **Register the network** in `Maxalon/hetzner-box` `edge/compose.yml`:
   ```yaml
   networks:            # under caddy.networks too: - edge-<name>
     edge-<name>:
       external: true
       name: edge-<name>
   ```
7. **On the box** (the user runs these):
   ```bash
   git -C /srv/hetzner-box pull
   git clone <repo-url> /srv/projects/<name>          # private repo: read-only deploy key
   install -m 600 /dev/null /srv/secrets/<name>.env   # then fill it in
   box enable <name> caddy/<name>.caddy
   box deploy <name>
   ```
   A **CI-deployed** project (its own pipeline ships the stack) clones sparse
   (`git sparse-checkout set caddy`), skips `box deploy`, and adds
   `box refresh <name>` as its deploy job's last step.
8. **DNS + token**: create the record(s) from step 3. If the host is on a new
   zone, add that zone to the Cloudflare token's scope first.
