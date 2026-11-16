---
name: hetzner-box
description: This project is hosted on the shared Hetzner box. Use before changing its Caddy fragment, its compose networks or published ports, its secrets/env file, or how it deploys; or when hosting a new project on the box.
---

The box's hosting contract lives in the public `Maxalon/hetzner-box` repo. Read
it before editing anything listed above; it is the single source of truth, so
follow it over anything you remember:

```bash
curl -fsSL https://raw.githubusercontent.com/Maxalon/hetzner-box/main/skills/hetzner-box/SKILL.md
```

On the box itself it is at `/srv/hetzner-box/skills/hetzner-box/SKILL.md`.

This project's values in that contract:

| | |
|---|---|
| name | `<name>` |
| fragment | `caddy/<name>.caddy` |
| edge network | `edge-<name>` |
| secrets file | `/srv/secrets/<name>.env` |
| deploy | `box deploy <name>` (or, if its own CI ships the stack, `box refresh <name>`) |
