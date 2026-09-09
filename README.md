# Eco.AI Agentic Coding Meta-Harness

AI meta-harness for application development based on Adapted COM (ACOM) technology from EcoOS. It owns
the EcoOS domain tools, marketplace RAG, project-generation policy, bounded
role orchestration, human plan approval, and a shared event contract. Agent
backends are replaceable: the built-in agent, Pi, Codex, and Claude Code can
be selected per role.

## Install

Two customer-grade flows, both one command, no source checkout, no wine.
The installer always downloads **native** eco-cli / eco-wizard builds for
your OS (never wine), plus the prebuilt marketplace RAG index, verified
against the release manifest's sha256 checksums. Missing API keys never
block install or launch — finish configuration in the in-app `/setup`
wizard (or later in `~/.eco-harness/.env`).

### Linux

```bash
curl -fsSL https://github.com/peerf-eco/eco-coder-releases/releases/latest/download/install.sh | sh
```

or download and inspect first: `curl -fsSLO https://github.com/peerf-eco/eco-coder-releases/releases/latest/download/install.sh && sh install.sh`.

- Requires `curl` or `wget` (everything else — including Python 3.11 — is
  provided by `uv` automatically, per-user, no sudo).
- Shims land in `~/.local/bin/eco-harness` and `~/.local/bin/eco-harness-update`;
  if `~/.local/bin` is not on your `PATH`, add it:
  `echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc` (or your shell profile).

### macOS

```bash
curl -fsSL https://github.com/peerf-eco/eco-coder-releases/releases/latest/download/install.sh | sh
```

Same as Linux; downloaded binaries are Apple-Silicon/Intel native (arm64 /
x86_64) and the installer strips the Gatekeeper quarantine attribute
(`xattr -d com.apple.quarantine`) so first launch is not blocked. Gatekeeper
may still warn on first run of an unsigned binary — right-click → Open, or
System Settings → Privacy & Security → Allow.

### Windows

From PowerShell:

```powershell
irm https://github.com/peerf-eco/eco-coder-releases/releases/latest/download/install.ps1 | iex
```

or from cmd.exe:

```bat
powershell -Command "irm 'https://github.com/peerf-eco/eco-coder-releases/releases/latest/download/install.ps1' | iex"
```

or download and run: `powershell -ExecutionPolicy Bypass -File install.ps1`.
`uv` provides Python 3.11 if missing; the app installs to
`%USERPROFILE%\.eco-harness` and `eco-harness.cmd` / `eco-harness-update.cmd`
shims are added to your **user PATH** (available in new terminals).

### After install (all native OSes)

```bash
eco-harness                 # serve the UI + API on http://localhost:8000
eco-harness-update          # self-update (wheel + binaries + RAG index)
eco-harness doctor          # one-command health report
```

Open http://localhost:8000 — the setup wizard at `/setup` walks you through
the OpenRouter key (or **Skip — add later in `.env`**; the app runs degraded
but starts regardless).

Installed layout (`ECO_HOME`, default `~/.eco-harness`):

```text
~/.eco-harness/
├── venv/                  # Python 3.11 venv (uv-managed)
├── bin/                   # native eco-cli / eco-wizard builds (per-OS)
├── data/                  # marketplace_index.sqlite + marketplace_cache/
├── .env                   # OPENAI_API_KEY, ECO_API_TOKEN, … (chmod 600)
├── workspace.yaml         # UI settings overrides
├── output/                # generated projects + session registry
└── traces/                # per-session LLM traces
```

Uninstall: delete `~/.eco-harness` (Windows: `%USERPROFILE%\.eco-harness`)
and the shim files / user-PATH entry.

### Docker (any host with Docker)

```bash
# Linux / macOS (note `-s --` to pass flags through the pipe):
curl -fsSL https://github.com/peerf-eco/eco-coder-releases/releases/latest/download/install.sh | sh -s -- --docker
```

```powershell
# Windows — irm | iex cannot take flags; download first, then run:
curl.exe -fsSLO https://github.com/peerf-eco/eco-coder-releases/releases/latest/download/install.ps1
powershell -ExecutionPolicy Bypass -File install.ps1 -Docker
```

Writes `~/.eco-harness/docker-compose.yml` with your absolute host paths,
pulls the prebuilt multi-arch image (linux/amd64 + linux/arm64) from
ghcr.io, starts it, then downloads the native binaries and RAG index from
the same public GitHub release (sha256-verified, exactly like the native
flow) into the mounted `~/.eco-harness` (visible to the container as
`/data`). The in-container update uses the public release manifest by
default — no S3 or extra configuration involved.

```bash
# update the image:
docker compose -f ~/.eco-harness/docker-compose.yml pull && \
docker compose -f ~/.eco-harness/docker-compose.yml up -d
# update the binaries + RAG index (re-runs the manifest-driven download):
docker compose -f ~/.eco-harness/docker-compose.yml exec eco-harness python -m eco_harness update
# health:
docker compose -f ~/.eco-harness/docker-compose.yml exec eco-harness python -m eco_harness doctor
```

### Key paths / env vars in installed mode

| What | Where |
|---|---|
| App home (`ECO_HOME`) | `~/.eco-harness` (`bin/`, `data/`, `.env`, `output/`, `traces/`) |
| User project (`ECO_PROJECT_DIR`) | picked in the UI folder browser (or pre-set with `--project-dir`); worktrees and generated artifacts live under it |
| Binary lookup order | explicit → `ECO_<NAME>_PATH` → `<repo>/bin` → `$ECO_HOME/bin` (`.exe` on Windows) → `/opt` → legacy siblings → `PATH` |
| Prebuilt index | `~/.eco-harness/data/marketplace_index.sqlite` (refreshed by `eco-harness update`) |
| Config file | `~/.eco-harness/.env` (or `~/.eco-harness/workspace.yaml` for UI settings) |

## Where to install

Here are opinionated recommendations.

### TL;DR

| | Native install (recommended) | Docker mode |
|---|---|---|
| **Windows** | `%USERPROFILE%\.eco-harness` | `%USERPROFILE%\.eco-harness` |
| **Linux** | `$HOME/.eco-harness` | `$HOME/.eco-harness` |
| **macOS** | `$HOME/.eco-harness` | `$HOME/.eco-harness` |

The installers already default to exactly these. **You do not need to change anything** — these are the right choices. The rest of this answer explains why and what the tradeoffs are.

---

### Default paths are the right default

The installer creates `~/.eco-harness` (Linux/macOS) / `%USERPROFILE%\.eco-harness` (Windows) and puts:

```
~/.eco-harness/
├── bin/              ← eco-cli / eco-wizard native binaries
├── data/
│   ├── marketplace_index.sqlite   ← RAG vector index
│   ├── marketplace_cache/         ← component source corpus
│   └── .index_sha256            ← update dedup marker
├── venv/             ← uv-managed Python 3.11 + eco-harness package
├── .env              ← your API keys and overrides
└── docker-compose.yml  ← only in Docker mode
```

This is intentionally **per-user, not per-system** — it requires no sudo, survives OS upgrades, and doesn't pollute `/opt` or Program Files. On Linux/macOS the shims land in `~/.local/bin` (already on most `$PATH`s). On Windows the shims go to a user-PATH directory and the installer adds it automatically.

---

### When you might want a different location

**Change ECO_HOME** only if:

- You want a **shared install across multiple user accounts on the same machine** → pick a directory you own with no spaces (e.g. `D:\eco-harness` on Windows). Pass it before install:
  ```bash
  # Linux/macOS
  ECO_HOME=/opt/eco-harness curl -fsSL .../install.sh | sh

  # Windows PowerShell
  $env:ECO_HOME = 'D:\eco-harness'; irm .../install.ps1 | iex
  ```
  Then make sure `ECO_HOME` is exported in your shell profile / Windows Environment Variables so `eco-harness-update` and the shims keep working across sessions.

- You want it **off the system drive** on Windows → same as above, e.g. `D:\eco-harness`.

**Do not** change ECO_HOME just to put shims in a different place — the installer handles shim placement automatically.

---

### Windows-specific notes

**Spaces in paths** — `%USERPROFILE%` is `C:\Users\Your Name` (has a space). The installer handles this correctly, but if you set a custom `ECO_HOME`, avoid spaces or double-quote paths in your `.env`:
```ini
ECO_HOME=D:\eco-harness
```

**Long Path Warning** — Windows has a 260-char path limit by default. `D:\eco-harness` is short and safe. If your Windows username has non-ASCII or very long characters, the default `%USERPROFILE%\.eco-harness` stays below the limit, but deeply nested project paths may hit it — keep projects in `D:\projects` rather than `C:\Users\Your Very Long Name\Documents\Visual Studio 2022\Projects\...`.

**WSL vs native Windows** — if you run the harness in WSL2 Linux, install natively on Windows for the IDE integration and use `ECO_CLI_PATH` in `.env` to point at the Windows binary from WSL:
```bash
# in WSL .env
ECO_CLI_PATH=/mnt/c/Users/you/.eco-harness/bin/eco-cli.exe
```
It really does work, because WSL2 has a feature called interop that lets Linux processes run native Windows executables as if they were regular Linux commands. Let me walk through it.

The mechanism (two pieces)
**1. WSL2's interop feature.** When you type notepad.exe or any Windows .exe in a WSL2 bash terminal, WSL launches the real Windows binary through a thin Windows process. From Linux's point of view it looks like a normal ELF process: it returns an exit code, accepts stdin/stdout, can be piped, etc. The Linux kernel in WSL2 has a special drvfs/9p file system that knows how to load .exe files.
**2. Path translation layer.** /mnt/c/Users/you/ in WSL2 is the same files on the same NTFS disk as C:\Users\you\ in Windows. They're not two different directories — they're one disk seen through two lenses. So /mnt/c/Users/you/.eco-harness/bin/eco-cli.exe IS C:\Users\you\.eco-harness\bin\eco-cli.exe IS the same file.
When eco_harness (running as a Linux Python process) resolves ECO_CLI_PATH=/mnt/c/Users/you/.eco-harness/bin/eco-cli.exe, it just subprocess.Popens the .exe file. WSL's interop hands it to the Windows kernel. The Linux side never needs to "understand" PE format — it just calls the Windows loader.

Docker mode on Windows always uses the Linux container image regardless of where ECO_HOME lives — ECO_HOME on Windows is the **host data directory**, bind-mounted into the container at `/data`.

** Direct WSL2 Linux install is better if you code in WSL**
See more details on windows install possibilities below.

---

### Linux-specific notes

**Root-owned servers** — if installing as root on a shared server and you want a single install for all users, pick `/opt/eco-harness`:
```bash
sudo ECO_HOME=/opt/eco-harness bash install.sh    # root install
# then add to /etc/environment or a profile.d/ script:
echo 'export PATH="/opt/eco-harness/bin:$PATH"' | sudo tee /etc/profile.d/eco-harness.sh
```
For single-user workstations, `~/.eco-harness` is simpler.

**NFS/home directories** — if your `$HOME` is NFS-mounted, `~/.eco-harness` works but first-run can be slow. A local SSD path like `/tmp/eco-harness-$USER` is faster, but you lose data on reboot. Only relevant for thin-client / network-home setups.

---

### macOS-specific notes

**Apple Silicon (M1/M2/M3)** — the installer detects `arm64` and downloads the native Apple Silicon binary. `~/.eco-harness` works perfectly on both Intel and Apple Silicon Macs. The Gatekeeper prompt on first run requires right-click → Open on the shim, or `sudo xattr -rd com.apple.quarantine ~/.local/bin/eco-harness` once.

**MacPorts / Homebrew** — the harness does not冲突 with either. uv (bundled by the installer) manages its own Python independently of any system Python or Homebrew python.

---

### Docker mode path anatomy

When you run `install.sh --docker` (or install.ps1 -Docker), the installer writes `~/.eco-harness/docker-compose.yml` and a `.env`. The `.env` contains:

```
ECO_HOME=/data          # inside the container
```

Inside the container the mounted host directory is `/data`, so `~/.eco-harness/data/` on the host appears as `/data/` inside the container. The shims on your host call `docker compose -f ~/.eco-harness/docker-compose.yml exec eco-harness python -m eco_harness ...` — they never need to know the container's internal paths.

The Windows/macOS/Linux distinction is fully handled by Docker. You do not need separate ECO_HOME values for Docker mode.

---

### Summary recommendation

**Use the defaults.** The installers picked sensible per-user paths that require no admin rights, survive upgrades, and avoid conflicts. The only time to change ECO_HOME is if you have a specific reason (shared multi-user install, off-system-drive on Windows, NFS home directory). Everything else — shims, PATH, Docker mounts, update tracking — works correctly out of the box.


### Where to install on Windows with WSL2

It depends on your editor/terminal setup. Here's the honest trade-off and a clear decision rule:

## WSL2 Linux install is better *if* you code in WSL

**Pick WSL2 when:** you use VS Code with the **Remote-WSL** extension (or just use the WSL bash terminal).

Reasons the local dev loop is materially better:
- `make`, `source .venv/bin/activate`, `curl | sh`, `docker compose`, `python -m pytest` — all POSIX, all run natively on the real Linux kernel WSL2 provides. On native Windows, `make` and `source` need emulation layers (Git Bash / MSYS2) that sometimes conflict with the harness's own scripts.
- Docker Desktop already runs on the WSL2 backend on Windows, so `install.sh --docker` and `docker compose exec eco-harness python -m eco_harness update` work *exactly* like on a real Linux machine — no path translation.
- VS Code Remote-WSL integrates the harness's shims and `eco-harness serve` directly into the IDE's terminal without a `wsl` prefix.
- The harness codebase itself is POSIX-first (`*.sh`, `docker-compose.yml`, `Makefile`, `scripts/release/*.sh`).

## Native Windows install is better *if* you live in PowerShell/CMD

**Pick native Windows when:** you use the Windows-side terminal (PowerShell, CMD, Windows Terminal) and/or VS Code on Windows (not Remote-WSL), and you want the harness callable from every Windows app without a `wsl` prefix.

Reasons:
- Shims land on the Windows PATH (`~/.local/bin` on Linux side of WSL is *not* in Windows' PATH) — Start menu, taskbar, and any Windows tool can invoke `eco-harness` directly.
- `.env` on NTFS is readable by any Windows-side tool; in WSL2 it lives on ext4 and Windows-side apps can't read it directly.
- `install.ps1 -Docker` and the `.env` Windows paths (`ECO_CLI_PATH`, `ECO_WORKTREE_HOST_DIR`) are designed for this.

## The decision rule

- **VS Code + Remote-WSL** → install in WSL2.
- **VS Code on Windows / PowerShell / CMD / Windows Terminal** → install natively on Windows.
- You can have both — but then you have two copies of `~/.eco-harness` (ext4 + NTFS) and the `.env`/API keys live separately. If you go that route, keep the WSL2 one as the primary dev copy and use the Windows one only for the CLI tools called from PowerShell.

A quick check on your setup would tell you which side you're on: open your terminal and run `echo $SHELL` — if it's `/bin/bash` under WSL, you're a WSL2 user; if it's `pwsh`/`powershell`, you're a Windows-side user.
