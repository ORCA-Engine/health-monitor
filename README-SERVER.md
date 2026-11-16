# Running the Health Monitor server on the Linux jumpserver

Beginner-friendly guide for running `server/` (the FastAPI app) on the Ubuntu
jumpserver. Commands assume the repository is checked out at:

```
/home/orca-jumpserver/Projects/health-monitor
```

and the Python virtual environment lives at the repository root:

```
/home/orca-jumpserver/Projects/health-monitor/.venv
```

Use exactly that path. Do not create `server/.venv`.

## What you are setting up

- `server/` is the FastAPI app that receives agent heartbeats, probes sites,
  stores data in SQLite, and sends Telegram alerts.
- The app reads its config from files in `server/`, so it must be started from
  inside `server/` (see **Run the server**). Paths like `sites.json` and the
  SQLite file are relative to that directory.
- The process must keep running after you close SSH, so it is started inside
  `tmux`.

## Prerequisites

1. **Python 3.11 or newer.** Check it:

   ```bash
   python --version
   ```

   The project declares `requires-python = ">=3.11"`.

2. **tmux** (a persistent terminal that survives SSH disconnects).

Install tmux on Ubuntu/Debian if needed:

   ```bash
   sudo apt update
    sudo apt install -y tmux
   ```

Use a `python` interpreter that reports version 3.11 or newer.

## First-time setup

### 1. Open a persistent terminal

```bash
tmux new-session -s health-monitor
```

This creates a tmux session named `health-monitor` and puts you in a shell
inside it. Everything you start here keeps running when you disconnect SSH.

### 2. Go to the checkout and check Python

```bash
cd /home/orca-jumpserver/Projects/health-monitor
python --version
```

### 3. Create the virtual environment (only if it does not exist)

```bash
python -m venv .venv
```

Do not delete and recreate `.venv` on every run, and do not create a second one
inside `server/`.

A virtual environment keeps this project's Python packages separate from system
Python. Creating it is a one-time step; activating it selects that environment
for your current shell.

### 4. Activate it

```bash
source .venv/bin/activate
```

Your prompt now starts with `(.venv)`. This tells you the environment is active.

### 5. Install the server

```bash
cd server
python -m pip install .
```

Run install from `server/` so the project metadata in `server/pyproject.toml`
is used. You only need to re-run this when the code or dependencies change -
not every time you start the server.

### 6. Configure site targets

The app reads `server/sites.json`. Copy the example without overwriting an
existing file:

```bash
cp -n sites.example.json sites.json
```

Then edit it with real targets:

```bash
nano sites.json
```

Each entry needs `site_name`, `site_id`, and optional `router_ip` / `pc_ip`.
The `site_id` must match the ID the Windows agent sends. The server pings these
addresses, so they must be reachable from the jumpserver.

### 7. Optional: `.env`

There is a working default config, so `.env` is optional. If you want to change
settings (Telegram alerts, port, summary time):

```bash
cp -n .env.example .env
nano .env
```

- `TELEGRAM_BOT_TOKEN` and `TELEGRAM_CHAT_ID` are credentials. Keep them only in
  `.env`, never in the README, chat, or git. `server/.env` is git-ignored.
- `chmod 600 .env` is a reasonable precaution on a shared server.
- The example sets `SERVER_PORT=8080`. The code default is `8000`. Pick one and
  use the same port in your health check.
- `.env` is read at startup, so restart the server after changing it.

## Run the server

From the **activated** shell, inside `server/`:

```bash
cd /home/orca-jumpserver/Projects/health-monitor/server
python -m app.main
```

Leave it running. The console prints log lines; that is normal.

### Check it from a second SSH shell

In a new SSH login (not inside tmux), run:

```bash
curl http://localhost:8000/healthz
```

Expect `{"status":"ok"}`. If you changed the port in `.env`, use that port
instead. This also confirms the app started before you detach.

## Detach safely (leave the server running)

1. Press `Ctrl+b`, release the keys, then press `d`.
2. Wait until you see a message like `[detached (from session health-monitor)]`.
3. Only then close SSH.

Do **not** use `Ctrl+C`, `exit`, or kill the process to "detach". Those stop the
server (or end the tmux session) instead of leaving it running.

## Come back later

```bash
tmux ls
tmux attach-session -t health-monitor
```

If `tmux ls` already lists `health-monitor`, **attach** to it. Do not run
`tmux new-session -s health-monitor` again; tmux rejects an existing session name.
Starting another server under a different session name can cause a port conflict.

## Stop and restart

Inside the attached session:

- `Ctrl+C` stops the server gracefully (uvicorn shuts down on SIGINT).
- Wait for the shell prompt, then start it again with
  `python -m app.main`.
- `exit` only after the server has stopped.

After a reboot, tmux sessions and the running server are gone; files, the venv,
and saved SQLite data remain. Log in, create a session, activate the venv, and
start the app again. These instructions don't configure automatic startup.

## Subsequent starts (the short version)

First check `tmux ls`. If the session exists, attach using the commands above;
if the server is still running, don't run the startup commands again.
If no session exists, run:

```bash
tmux new-session -s health-monitor
cd /home/orca-jumpserver/Projects/health-monitor
source .venv/bin/activate
cd server
python -m app.main
```

No reinstall, no venv re-creation. Then `Ctrl+b`, release, `d` to detach.

## Troubleshooting

- **`tmux: command not found`** - `sudo apt install -y tmux`.
- **`ModuleNotFoundError: No module named 'app'` or missing `sites.json`** -
  wrong directory. You must be in `server/` with the venv active.
- **`Address already in use`** - something already holds the port:
  `sudo ss -ltnp | grep 8000`. Stop the old instance (attach the existing tmux
  session and press `Ctrl+C`) instead of starting a second one.
- **Health check fails but the console looks fine** - confirm the port in
  `server/.env`, and that you used `localhost` on the jumpserver itself.

## Notes

- The venv is per-shell: every new SSH login and every new tmux window needs
  `source .venv/bin/activate` before `python` means the venv's Python. The
  installed packages are not global, so the system Python will not see them.
- `.gitignore` currently ignores `server/.venv/` but not the root `.venv/`, so
  `git status` may show the root venv as untracked. Leave it out of commits.
