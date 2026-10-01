# virtuoso-bridge: local-mode setup (no SSH)

How this bridge is actually run on the Glasgow TSMC65 flow, which is **not** the
setup the main README leads with. The README assumes Virtuoso is on a *remote*
host and that the bridge opens an SSH tunnel to it. Here it is not: Virtuoso and
the bridge run on the **same machine**, so there is no tunnel, no jump host and
no `VB_REMOTE_USER`.

That distinction matters because SSH is not merely unnecessary here, it is
unavailable — outbound port 22 is blocked on the university network. Following
the README's `virtuoso-bridge init user@host` path will not work.

## How this branch relates to upstream

This branch is **upstream's `main` plus this one file**, and nothing else. It is
kept that way deliberately: the bridge itself is not forked or patched here, so
there is no divergence to maintain and no risk of drifting behind on fixes.

```
upstream   https://github.com/Arcadia-1/virtuoso-bridge-lite   (the real project)
origin     https://github.com/meraj025/virtuoso-bridge-lite    (fork)
branch     local-setup = origin/main + SETUP-local.md
```

Note that `pyproject.toml` has declared `0.8.0` across many commits, so
`virtuoso-bridge --version` cannot tell you which build you have. Use
`git describe --tags`.

The tag `working-v0.8.0` marks commit `fb5af05` (21 Aug 2026), the build this
flow was first validated against. It is a record, not a pin — useful if a newer
upstream commit ever breaks something and you need a known-good point to compare
against.

## 1. Clone

```bash
git clone -b local-setup https://github.com/meraj025/virtuoso-bridge-lite.git
cd virtuoso-bridge-lite
git remote add upstream https://github.com/Arcadia-1/virtuoso-bridge-lite
```

## 2. Build the environment

Never install into the system Python — it runs on its own venv, separate from
Cadence's `cdspython`.

```bash
uv venv .venv
source .venv/bin/activate
uv pip install -e .
```

Every `virtuoso-bridge` command below assumes this venv is active.

## 3. Write the local-mode config

The config lives in your **home directory**, not in the repo, so it survives
re-clones and is not shared by git:

```bash
mkdir -p ~/.virtuoso-bridge
cat > ~/.virtuoso-bridge/.env <<'CONF'
VB_REMOTE_HOST=localhost
VB_REMOTE_USER=
VB_REMOTE_PORT=65477
VB_LOCAL_PORT=65478
CONF
```

`VB_REMOTE_HOST=localhost` with an empty `VB_REMOTE_USER` is what selects local
mode. `virtuoso-bridge status` confirms it with `[mode] local (no SSH tunnel)`.

**If several people use the same machine, change the ports.** They are a
per-user listening socket, not a shared service: two users on the defaults will
collide, and the second one to start sees a daemon that answers but drives the
wrong Virtuoso session. Pick a distinct pair each, e.g. `65479`/`65480`.

## 4. Start the bridge and load the CIW half

The bridge is two halves. This starts the client half:

```bash
virtuoso-bridge start
```

It prints a `load(...)` line. **Paste that line into the Virtuoso CIW** — that
starts the daemon inside Virtuoso, and nothing works until you do. Virtuoso must
already be running.

This is per Virtuoso session: restart Virtuoso and you load it again.

## 5. Verify

```bash
virtuoso-bridge status
```

Healthy looks like:

```
[mode] local (no SSH tunnel)
  port : 65477
[daemon] OK - connected to Virtuoso CIW
  hostname  : <your host>
  version   : virtuoso version IC25.1-64b
  workdir   : /path/to/your/workdir
[spectre] NOT FOUND
```

`[spectre] NOT FOUND` is harmless for this flow. It affects only the standalone
Spectre runner (netlist to simulate to parse PSF), which is unused when ADE
Assembler drives the simulations. To enable it, `VB_CADENCE_CSHRC` must point at
a **csh** setup script — a `cadence_setup_*.bash` will not work.

`[daemon] NO RESPONSE` almost always means step 4 was skipped, or Virtuoso was
restarted since.

Then:

```bash
virtuoso-bridge windows          # list open Virtuoso windows
virtuoso-bridge eval 'getCurrentTime()'
```

## 6. Agent skills (optional)

For driving the bridge from Claude Code, symlink the skills rather than copying
them, so they track the checkout:

```bash
ln -s "$PWD/skills/virtuoso" ~/.claude/skills/virtuoso
ln -s "$PWD/skills/spectre"  ~/.claude/skills/spectre
ln -s "$PWD/skills/netlist"  ~/.claude/skills/netlist
```

`skills/optimizer` also exists — black-box sizing via TuRBO/scipy. It is an
*alternative* to driving AOP from ADE, not a companion to it, so it is left
unlinked in this setup.

## Pairing with NNSADE

This bridge scripts the ADE setup; NNSADE optimises inside it. They are
independent installs and neither requires the other:

- NNSADE runs on Cadence's `cdspython`, and its whole install is selecting
  `nnsade_aop.py` in the AOP Options form.
- This bridge runs on its own venv and never touches AOP.

Use the bridge to build and inspect the tests, outputs and specifications; use
NNSADE to rank designs against them.

## Staying in sync with upstream

Because this branch is one commit on top of upstream, updating is a rebase that
replays that single file onto whatever upstream has become:

```bash
git fetch upstream
git rebase upstream/main          # replays SETUP-local.md onto the new main
uv pip install -e .               # dependencies may have moved
virtuoso-bridge status            # confirm the daemon still answers
```

Push the result to the fork with `git push --force-with-lease origin local-setup`
— force is expected here and is not destructive, because the branch is by
definition rewritten on every rebase and nobody should be committing to it.

Conflicts should never occur: upstream does not have a `SETUP-local.md`. If one
ever does appear, rename this file rather than merging the two.

**Re-run a real simulation after updating before trusting it.** The bridge talks
to a live Virtuoso session and the useful failures are integration failures, not
ones a unit test catches.

## Quick troubleshooting

| symptom | cause |
|---|---|
| `Connection refused to 127.0.0.1:<port>` | daemon not loaded — redo step 4, or Virtuoso is closed |
| CIW says "already running" | a daemon is bound; the `load()` did not replace it. Change ports or stop the old one |
| commands hang, no error | a modal dialog is blocking the CIW. `virtuoso-bridge dismiss-dialog` |
| `virtuoso-bridge: command not found` | the venv is not active |
| SSH timeouts in any form | port 22 is blocked here; you are meant to be in local mode |
