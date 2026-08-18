# DeepSeek Harness (`dsh`) Sandbox Kit

Docker Sandboxes kit for running [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness)
(`dsh`) — DeepSeek's open-source, plugin-everything agent harness — inside an
isolated sandbox.

An interactive sandbox serves the harness Web UI on a published port. A
`--task` run answers one prompt through the headless profile and exits. Your
DeepSeek API key never enters the container: the sandbox proxy injects it on
requests to `api.deepseek.com`.

## Quick start

```bash
# one-time: let sbx load kits from this source
sbx settings set kit.allowedSources '["docker.io/","github.com/shelajev/"]'

sbx create --name dsh-current \
  --kit git+https://github.com/shelajev/dsh-sbx-kit.git dsh .

sbx run dsh-current            # serves the Web UI
sbx ports dsh-current          # prints the host URL to open
```

Open the printed `http://127.0.0.1:<port>` in a browser on your laptop. In the
UI: **Choose workspace** → add the project directory (the sandbox workspace path
`dsh` started in) → start a session.

This is a `kind: sandbox` kit, so it goes on `sbx create` / `sbx run --kit`, not
`sbx kit add` — published ports are fixed at container creation and `kit add`
skips them.

Bind a DeepSeek key on the host once, and every sandbox from this kit picks it
up:

```bash
sbx secret set deepseek -t "$DEEPSEEK_API_KEY"
```

Without a binding, `sbx` prompts at create time for where the `deepseek`
credential lives (env var or file) and for permission to send it to
`api.deepseek.com`, then records both in `~/.config/sbx/credentials.yaml`.
Declining the domain does not fail creation — requests just go out
unauthenticated and DeepSeek answers 401.

One-shot, no browser:

```bash
sbx run --task "summarize this repository and identify its main packages" \
  --kit git+https://github.com/shelajev/dsh-sbx-kit.git dsh .
```

## How the two surfaces map onto `sbx`

`dsh` has no shipped TUI; its surfaces are the Web UI and the headless profile,
so the kit binds one to each launch mode:

| Launch | argv | What you get |
|---|---|---|
| `sbx run <name>` (TTY) | `dsh web` | Web UI on container `3080`, bridged to `3081`, published by the runtime |
| `sbx run --task "…"` | `dsh --profile headless "…"` | One persisted session, final answer on stdout, exit 0 on completion |
| `sbx exec <name> -- …` | anything | e.g. `dsh --profile headless "run the tests"` |

## Reaching the Web UI

`dsh web` accepts only `127.0.0.1` or `0.0.0.0` as its bind host, and the CLI
refuses `0.0.0.0` outright:

```
error: --host 0.0.0.0 is intentionally not supported yet for safety:
it would expose remote code execution to the network; use 127.0.0.1 instead
```

A published sandbox port, however, is forwarded to the container's `eth0`
address, which loopback never answers. So the kit leaves `dsh` on loopback and
ships a raw TCP forwarder,
[`dsh-web-bridge`](files/home/.local/bin/dsh-web-bridge), started by the kit's
startup hook:

```
browser → 127.0.0.1:<ephemeral host port> → container 3081 (bridge) → 127.0.0.1:3080 (dsh)
```

It is a byte-for-byte TCP relay, so the UI's WebSocket downlink upgrades
through it unchanged. Nothing but the relay listens on the container's network
interface, and `dsh` keeps its own posture.

Host ports are allocated ephemerally on `127.0.0.1`; `sbx ports <sandbox>` shows
the current one. Pin it if you want a stable bookmark:

```bash
sbx ports dsh-current --publish 3081:3081/tcp
```

**Browse it over loopback.** `dsh`'s `/api` browser-trust fence accepts any
`localhost`/`127.0.0.1` authority (any port) and answers `403 forbidden` to
anything else, and the configuration plane — Settings, Models, credentials, the
directory picker — is restricted further to loopback same-origin requests. A URL
like `http://<LAN-ip>:3081` therefore loads the page and then fails every API
call. If you really need that, add the authority yourself:

```bash
sbx exec dsh-current -- dsh web --trusted-host my-box.local:3081
```

## Credentials

```yaml
credentials:
  - service: deepseek
    apiKey:
      name: DEEPSEEK_API_KEY
      proxyManaged: true
```

`DEEPSEEK_API_KEY` inside the container holds the `proxy-managed` sentinel. The
host-side proxy replaces it with your real key on requests to
`api.deepseek.com` — as `Authorization: Bearer <key>` for the chat-completions
route and as `x-api-key` for the `/anthropic/v1` endpoint the built-in
`web_search` tool uses. The agent, its session logs, and anything it writes to
the workspace only ever see the sentinel.

`dsh` resolves its key from the credential seam first and the environment
second, so the env var alone is enough — you never open **Settings → Models**
and paste a key. Doing that instead writes the literal key to
`$DSH_HOME/.credentials.yaml` *inside* the sandbox; the proxy-managed path
exists precisely to avoid that.

Other providers (Anthropic, OpenAI, a company gateway…) still work through
**Settings → Models**, but their hosts are not in this kit's allow list and their
keys would live in the container. Fork the kit and add a `credentials` entry
plus the matching `permissions.network.allow` host for anything you want on the
proxy-managed path.

## Permission mode

The kit sets `DSH_PERMISSION_MODE=danger-full-access`.

That is not bravado, it is the only mode that runs. `dsh` confines its own bash
tool with bubblewrap or Landlock and **fails closed** when neither backend is
usable (`SANDBOX_UNAVAILABLE` — "refusing to run the command unconfined"). Inside
a sandbox container both are out: `bwrap` cannot create a user namespace, and
the sandbox kernel reports Landlock as unsupported/disabled. In
`danger-full-access`, `dsh-bash-sandbox` never consults the sandbox provider at
all, and the approval policy becomes `never` — which is also what a headless
`--task` run needs, since there is no human to answer a prompt.

The sandbox is the boundary; the harness inside it does not need a second one.

If you want `dsh`'s approval prompts back in the Web UI, and you have a base
image where one of the backends actually works:

```bash
sbx run --kit "git+https://github.com/shelajev/dsh-sbx-kit.git?permissionMode=workspace-write" dsh .
```

## Network policy

Allowed:

- `registry.npmjs.org` — installing `@deepseek-ai/dsh` at create time, and
  `dsh plugin … add <pkg>` later
- `api.deepseek.com` — model requests and the built-in web search

Deliberately **not** allowed:

- `harness-telemetry.deepseeksvc.com` — `dsh`'s OTLP session-telemetry exporter.
  Upstream ships it `DISABLED`; the kit leaves that default alone and simply
  never allows the host, so a stray `DSH_TELEMETRY_MODE=FULL` cannot quietly
  ship session logs (message text, tool arguments, workspace paths) off the box.
  Allow the host in a fork if you want telemetry.
- Package registries other than npm. An agent that runs `pip install` gets an
  HTTP 403 whose body explains the block. Extend the fork's allow list, or grant
  it per-host on the fly — remembering that metadata and content usually live on
  different hosts:

  ```bash
  sbx policy allow network pypi.org,files.pythonhosted.org
  ```

  `apt-get` is the sharp one: `apt-get update` refreshes *every* source in
  `/etc/apt/sources.list.d/`, and this template pre-adds Docker's repo, so one
  blocked source fails the whole run even when your package lives elsewhere. It
  needs four hosts at once — `archive.ubuntu.com`, `security.ubuntu.com`,
  `ports.ubuntu.com` (arm64), and `download.docker.com`. The kit's
  `AGENTS.md` tells the agent this so it reports the block instead of retrying.

Only npm is allowed by default because that is all the kit itself needs:
logging every URL `npm install -g @deepseek-ai/dsh` requests — 587 packages,
platform-specific native addons included — shows `registry.npmjs.org` and
nothing else.

## Tuning

Two knobs, both one-line edits in [`spec.yaml`](spec.yaml):

| Knob | Where | Default |
|---|---|---|
| dsh version | `@deepseek-ai/dsh@latest` in `setup.install` | `latest` — pin an exact version (`@0.1.0-rc.7`) for anything reproducible, since dsh is a preview with breaking changes |
| permission mode | `DSH_PERMISSION_MODE` under `environment` | `danger-full-access`; see above |

They are literals rather than a v2 `args:` block on purpose. The published
grammar has one, but `sbx` v0.38.0 rejects it under strict decoding —

```
INVALID: artifact: invalid spec.yaml: yaml: unmarshal errors:
  line 4: field args not found in type spec.specFileV2
```

— and there is no `--kit-arg` flag to supply values with. Worth switching when a
release ships both.

## Persistence

`DSH_HOME` is `/home/agent/.dsh` on the persistent home volume, so profiles,
`settings.yaml`, `.credentials.yaml`, session logs, and workspace registrations
survive restarts. The `web` profile is materialized during install, so the first
interactive start takes about a second instead of re-linking its module tree on
demand.

## Smoke tests

```bash
# CLI is installed and boots its launcher
sbx exec dsh-current -- dsh --version

# the profile tree composes inside the sandbox
sbx exec dsh-current -- sh -lc 'dsh --profile web --dump-config | head'

# one headless turn (needs a bound DeepSeek key; without one it prints
# MISSING_CREDENTIAL, which still proves the profile booted)
sbx exec dsh-current -- dsh --profile headless "reply with the word ok"

# the bridge is up and the UI answers through it
sbx exec dsh-current -- curl -s -o /dev/null -w '%{http_code}\n' \
  -H 'Host: 127.0.0.1:3081' http://127.0.0.1:3081/
```

## Local clone

```bash
./run.sh                # sandbox named dsh-current, kit loaded from this directory
./run.sh my-sandbox     # any other name
```

## How it works

- **Image:** `docker/sandbox-templates:shell-docker` (Ubuntu 26.04) already ships
  Node 22.22 — `dsh` needs `^22.19 || >=24` — plus `socat`, `curl`, `git`, and a
  passwordless-sudo `agent` user, so the kit installs exactly one npm package.
- **Install (once):** `npm install -g @deepseek-ai/dsh@<version>` as uid 1000
  with the sandbox proxy configured for npm, retried up to five times; then
  `dsh --profile web --dump-config` to materialize `$DSH_HOME/profiles/web`.
- **Startup (every start):** `dsh-web-bridge`, detached with `setsid`, idempotent
  (a second copy notices the port is already served and exits 0).
- **Entrypoint:** `dsh`, with `web` appended for interactive launches and
  `--profile headless` for `--task` launches.
- **Model default:** `deepseek-official` / `deepseek-v4-flash`, from the shipped
  web profile.

## Known rough edges

- `dsh` is a developer preview; its profile layout, row ids, and CLI grammar can
  change between releases. Pin `version` if that matters to you.
- The Web UI's own URL line says `http://127.0.0.1:3080`, which is the
  container's view. The URL you open is the published one from
  `sbx ports <sandbox>`.
- `dsh plugin … add` shells out to pnpm and may need hosts beyond
  `registry.npmjs.org` (a git-hosted plugin needs `github.com`).
- Every sandbox from this kit runs `dsh` unconfined *inside* the container by
  design. Treat the workspace you mount as the blast radius.

## License

Apache 2.0. See [LICENSE](LICENSE). DeepSeek Harness itself is MIT-licensed and
is not vendored here — the kit installs it from npm at sandbox-create time.
