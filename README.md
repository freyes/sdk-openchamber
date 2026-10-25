# OpenChamber Workshop SDK

Configures OpenChamber as a systemd user service inside a workshop container.
On launch, it installs the `openchamber` snap, auto-generates a UI password,
and starts the web server on `0.0.0.0:3000`.

## Prerequisites

**The `opencode` SDK MUST be included in your `workshop.yaml` before this SDK.**

OpenChamber runs on top of the OpenCode agent engine. Without OpenCode,
the web server starts but cannot process agent requests.

Example `workshop.yaml`:

```yaml
name: my-project
base: ubuntu@26.04
sdks:
  - name: direnvrc
  - name: opencode          # REQUIRED: agent engine
  - name: try-openchamber
```

## Build

```bash
sdkcraft pack
```

This produces `openchamber_amd64.sdk`. The SDK ships hook scripts only — no
binary payload. The `openchamber` snap is installed at runtime (setup-base).

Supported bases: **ubuntu@24.04** and **ubuntu@26.04**.

## Local Testing

```bash
sdkcraft try
```

Then reference it as `try-openchamber` in your `workshop.yaml` and launch:

```bash
workshop launch workshop.yaml
```

## What the SDK Does

### setup-base (runs as root, once per revision)
1. Installs the `openchamber` snap, in order of preference: from
   `$OPENCHAMBER_SNAP_PATH` (if set), from a locally built
   `/project/openchamber_*.snap` (newest version if several), or from the Snap
   Store. Local snaps are installed with `--dangerous`, retrying with
   `--classic` if the snap uses classic confinement.
2. Enables linger for the `workshop` user so the systemd user service starts on
   boot.
3. Writes `/etc/profile.d/openchamber.sh` for PATH access in interactive shells.
4. Installs and starts the `openchamber-opencode.service` systemd user unit
   (`opencode serve` on `127.0.0.1:2018`).

### OpenCode memory limits

Before each start of `openchamber-opencode.service`, the SDK sources
`/project/openchamber-env.sh` (if it exists) and applies these variables to the
unit's cgroup:

| Variable | systemd property |
|---|---|
| `OPENCODE_MEM_MAX` | `MemoryMax` |
| `OPENCODE_MEM_HIGH` | `MemoryHigh` |

Values use systemd syntax (e.g. `2G`, `1536M`, `80%`, `infinity`). Unset
variables mean no limit. Example:

```bash
# /project/openchamber-env.sh
OPENCODE_MEM_MAX=4G
OPENCODE_MEM_HIGH=3G
```

Apply changes with `systemctl --user restart openchamber-opencode.service`.

### setup-project (runs as workshop user)
1. Reads or generates a 32-character hex password in `/project/.openchamber-password`.
2. Adds `.openchamber-password` to `/project/.gitignore` to prevent accidental commits.
3. Creates `~/.config/systemd/user/openchamber.service` from a template — the service
   runs `/snap/bin/openchamber serve --foreground --port 3000 --lan` with the
   generated UI password, connects to the OpenCode agent on `127.0.0.1:2018`, and
   loads optional overrides from `/home/workshop/.config/openchamber/startup.env`.
4. Reloads systemd and restarts the service.

### check-health (runs as root)
1. Verifies the `openchamber` binary is on PATH.
2. Verifies `openchamber.service` is active.
3. Reports `okay` or `error` via `workshopctl set-health`.

## Finding the Password

```bash
cat /project/.openchamber-password
```

## Accessing the Web UI

From your browser:

```
http://<workshop-ip>:3000
```

Enter the password from `/project/.openchamber-password` when prompted.

## Supported Bases

| Base | Status |
|---|---|
| ubuntu@24.04 | Supported |
| ubuntu@26.04 | Supported |

## License

MIT