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

```powershell
irm https://github.com/peerf-eco/eco-coder-releases/releases/latest/download/install.ps1 | iex
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
