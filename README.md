# OpenChamber Workshop SDK

Configures OpenChamber as a systemd user service inside a workshop container.
On launch, it installs the `openchamber` snap, auto-generates a UI password,
and starts the web server on `0.0.0.0:3000`.

> [!WARNING]
> **The `openchamber` snap is NOT installed from the Snap Store.**
> The Store snap does not support classic confinement yet, so, until it does,
> `setup-base` downloads the newest build (including pre-releases) from
> <https://github.com/freyes/snap-openchamber/releases> and installs it with
> `snap install --dangerous --classic`. This means:
>
> - The snap is **unsigned and unasserted** (`--dangerous`), and it is
>   **classic-confined**, i.e. it runs with full access to the system.
>   Only use this SDK if you trust that GitHub repository.
> - The snap **does not auto-refresh** from the Store. It is only (re)installed
>   when `setup-base` runs and no snap is installed yet.
> - `setup-base` needs network access to `api.github.com` and `github.com`.
>   The downloaded file is verified against the SHA-256 digest published by
>   GitHub for the release asset.
>
> This is a temporary workaround. It will be reverted to `snap install openchamber`
> once the Store snap supports classic confinement.

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
   `/project/openchamber_*.snap` (newest version if several), if already
   installed, keeps it, or else from the latest
   [GitHub release](https://github.com/freyes/snap-openchamber/releases)
   (temporary, see the warning above; the Snap Store is not used). Local and
   release snaps are installed with `--dangerous`, retrying with `--classic` if
   the snap uses classic confinement.
2. Installs the [oh-my-openagent](https://github.com/code-yeongyu/oh-my-openagent)
   (omo) plugin globally for the `workshop` user with
   `opencode plugin oh-my-openagent --global`, before the OpenCode service
   starts (see [oh-my-openagent configuration](#oh-my-openagent-configuration)).
3. Enables linger for the `workshop` user so the systemd user service starts on
   boot.
4. Writes `/etc/profile.d/openchamber.sh` for PATH access in interactive shells.
5. Installs and starts the `openchamber-opencode.service` systemd user unit
   (`opencode serve` on `127.0.0.1:2018`).

### oh-my-openagent configuration

| File | Content |
|---|---|
| `~/.config/opencode/opencode.jsonc` | `oh-my-openagent` added to `plugin` (global install) |
| `~/.config/opencode/omo.json` | Copy of `/project/omo.json` if it exists, otherwise the SDK's default model/reasoning profile |
| `~/.omo/omo.jsonc` | Written only if missing; disables the default builder agent (`default_builder_enabled: false`) |

`omo.json` is refreshed from `/project/omo.json` every time `setup-base` runs.

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