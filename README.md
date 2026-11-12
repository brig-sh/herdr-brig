# herdr-brig

Run each coding agent in its own [Brig](https://github.com/brig-sh/brig)
microVM, and manage them all from [herdr](https://herdr.dev).

One key creates a sandbox for a project, adds it to herdr as a remote
machine, and starts Claude Code in it. Another key removes it. The agent reaches
only the host directories and credentials Brig names for it: the project you
mount, its guest home, and the logins you forward.

herdr connects to the sandbox over SSH, with `brig exec` as the
`ProxyCommand`, so the guest has no open port. The plugin automates the steps
in Brig's [docs/herdr.md](https://github.com/brig-sh/brig/blob/main/docs/herdr.md).

## Requirements

- macOS (Apple Silicon) or Linux with KVM.
- [Brig](https://github.com/brig-sh/brig) with the `herdr` profile.
  `brig agent ls` lists it.
- [herdr](https://herdr.dev) 0.9 or later.

## Install

### 1. Install the plugin

```bash
herdr plugin install brig-sh/herdr-brig
```

Check: `herdr plugin action list --plugin brig-sh.herdr-brig` lists `new` and
`remove`.

### 2. Bind the keys

Add to `~/.config/herdr/config.toml`:

```toml
[[keys.command]]
key = "prefix+m"
type = "plugin_action"
command = "brig-sh.herdr-brig.new"
description = "brig: new sandbox"

[[keys.command]]
key = "prefix+shift+m"
type = "plugin_action"
command = "brig-sh.herdr-brig.remove"
description = "brig: remove sandbox"
```

Reload:

```bash
herdr config check
herdr server reload-config
```

With the default prefix, `ctrl+b m` creates a sandbox and `ctrl+b M` removes
one. Without bindings, run from a shell in a herdr pane:

```bash
herdr plugin action invoke brig-sh.herdr-brig.new
herdr plugin action invoke brig-sh.herdr-brig.remove
```

### 3. Give the sandbox a Claude Code login

```bash
brig secret import herdr
```

This copies your Claude Code login from the keychain into Brig's secret
store. Brig gives it to each sandbox on every run, in a memory-backed mount,
so Claude Code starts logged in. Repeat the import when the login expires.

To keep the login on the host instead, write a policy with a broker route, as
in Brig's [broker.md](https://github.com/brig-sh/brig/blob/main/docs/broker.md#use-it),
and choose it when you create a sandbox. See [Policies](#policies).

## What the plugin does

| Step | Plugin | You |
| --- | --- | --- |
| ssh access | On the first run, adds the `Host herdr*+*` block to `~/.ssh/config` | Confirm |
| Project | Mounts a directory read-write at `/work/<name>` | Confirm the focused pane's directory, type another, or `-` for none |
| Policy | `brig policy attach <policy> herdr -n <name>` | Pick one, or `-` for none |
| Boot | `brig run -d` | Name the sandbox |
| Register | `herdr machine add` | — |
| Workspace | Opens one workspace in the project | — |
| Agent | Starts Claude Code | Confirm |
| Remove | Disables the machine, removes the sandbox, removes the machine | Pick the sandbox and confirm |

The ssh block is the one in Brig's
[docs/herdr.md](https://github.com/brig-sh/brig/blob/main/docs/herdr.md#set-it-up-once).
herdr reaches a sandbox over ssh, and the block runs `brig exec` as the
`ProxyCommand`. The plugin inserts it before the first `Host` or `Match` line,
keeps a backup as `~/.ssh/config.bak-herdr-brig-<timestamp>`, and restores the
backup if ssh does not resolve `herdr+<name>` through it.

The plugin stops with a message when `brig` or `herdr` is missing, when Brig
has no `herdr` profile, when you decline the ssh block, when a machine with
the same name exists, or when the chosen policy cannot apply together with the
policies bound to the profile.

## Usage

### New sandbox

`ctrl+b m` opens a popup:

```
Project directory to mount, or - for none [/Users/you/code/api]:
Sandbox name [api]:

Policy for this sandbox:
  1) devwork
  -) none
Choose 1 or - [-]:
Start Claude Code in it now? [Y/n]
```

| Prompt | Default | Notes |
| --- | --- | --- |
| Project directory | The focused pane's directory, or `-` | Mounted read-write at `/work/<basename>`. `-` mounts nothing. |
| Sandbox name | The directory name, or `scratch` | `[a-z0-9-]`, at most 31 characters. Also the machine label. |
| Policy | `-`, or `$HERDR_BRIG_POLICY` | Lists your policies from `brig policy ls`. Bound to this sandbox only. Policies bound to the whole profile apply as well and are not listed. |
| Profile | `herdr` | Asked only when more than one profile name starts with `herdr`. |

The default directory is the focused pane's only when that pane is on this
machine. A pane on a sandbox machine reports whatever directory its guest
says, so the default is then `-`.

The plugin refuses to mount, whether suggested or typed:

- `/`, your home, and any directory above it.
- Every top-level dot directory in your home (`~/.ssh`, `~/.config`,
  `~/.local`, `~/.claude`, …) and `~/Library`.
- A directory on `PATH`, inside one, or holding one.
- `/usr`, `/etc`, `/opt`, `/bin`, `/sbin`, `/System`, `/Library`,
  `/Applications`, `~/bin`, `~/go` and `~/Applications`, which a plugin's
  `PATH` may not include.

A guest can write to anything mounted, and these are where a written file
becomes code or configuration the host runs.

#### Git checkouts

A mounted checkout is your working copy: the agent's edits appear in it, and
the guest can write to its `.git`, including hooks and config that git on the
host runs.

A linked worktree or a submodule has a `.git` file that points into another
repository's `.git`, which the guest cannot see. Its files are mounted, but
git does not work in the sandbox; the popup says so.

#### Policies

A policy decides what the sandbox may reach, and can route a credential
through Brig's broker. Policies are yours, in `~/.config/brig/policies`. See
Brig's [policies.md](https://github.com/brig-sh/brig/blob/main/docs/policies.md).

The plugin binds the chosen policy to the one sandbox:

```bash
brig policy attach devwork herdr -n api-fix
```

It applies together with any policy bound to the whole `herdr` profile. Two
policies that each define a route with the same name cannot apply together.
The plugin checks with `brig policy check` before it creates anything, and
stops if they clash.

Removing the sandbox removes its binding.

An example policy, `devwork`, that keeps both credentials on the host: the
Claude Code login, and a GitHub token limited to one organisation. Brig's
[policies.md](https://github.com/brig-sh/brig/blob/main/docs/policies.md#presets)
documents the `claude-code` and `github-token` presets.

```yaml
apiVersion: brig.sh/v1alpha1
kind: Policy
metadata:
  name: devwork
  description: Claude, and git over HTTPS to one GitHub org, through the broker
spec:
  egress:
    default: deny
  routes:
    - name: anthropic
      preset: claude-code
      credential:
        secretRef: { name: claude-credentials }
    - name: github
      preset: github-token
      credential:
        secretRef: { name: gh-token }
      request:
        allow:
          repos: ["your-org/*"]
```

Store the token first, for example `gh auth token | brig secret create gh-token`.
The guest gets a session token for each route, never the credential:

- git in the guest is pointed at the `github` route for `https://github.com/`
  and `git@github.com:` URLs. The broker adds the token.
- A repository outside `your-org/*` gets a 403 from the broker. The match is
  case-sensitive and GitHub is not, so list each spelling of an owner you use.
- `gh` and `api.github.com` are not covered; only git is.

For several organisations with separate tokens, write one policy for each and
pick per sandbox.

With `egress: default: deny`, a sandbox reaches only the hosts and routes its
policies name. With no policy, it has the network posture of the `herdr`
profile.

#### Steps

| Step | Command |
| --- | --- |
| Bind policy | `brig policy attach <policy> herdr -n api-fix`, then `brig policy check` |
| Boot | `brig run -d herdr@api-fix <dir>`, or `--no-project` |
| Add machine | `herdr machine add herdr+api-fix --label api-fix` |
| Open workspace | `herdr --machine api-fix workspace create --cwd /work/api`; closes the default workspace in `/root` |
| Start agent | `herdr --machine api-fix agent start api-fix --kind claude --pane <id>` |

About 15 seconds with the image cached. The popup closes on
success and stays open on an error. If `herdr machine add` fails, the plugin
removes the sandbox.

### Remove sandbox

`ctrl+b M` lists the herdr machines that are Brig sandboxes, asks which one,
asks for confirmation, then runs:

```bash
herdr machine disable <id>
brig rm herdr@api-fix
herdr machine remove <id>
```

An enabled machine reconnects, and the reconnect's `brig exec` boots the
sandbox again. Disabling it first prevents that.

`brig rm` deletes the guest home. The mounted project stays on the host. The
plugin then removes the sandbox's policy binding.

## Working with a sandbox

Each sandbox is a herdr machine and an SSH host named `<profile>+<name>`. The
examples use a sandbox named `api`.

### Files

| Location | Host | Guest |
| --- | --- | --- |
| Project | `~/code/api` | `/work/api` |
| Guest home | `~/.brig/homes/brig-herdr-api` | `/root` |

Both are shared directories. `brig rm` deletes the guest home.

Other paths, over SSH:

```bash
scp notes.txt herdr+api:/root/notes.txt
scp herdr+api:/root/out.log .
ssh herdr+api 'ls -l /root'
```

### Agents

Use herdr's agent commands with `--machine <label>`. Pane ids are per
machine:

```bash
herdr --machine api agent list
herdr --machine api agent read w1:p1 --source recent --lines 40
herdr --machine api agent prompt w1:p1 "Run the tests and summarise the failures" --wait
```

`--wait` returns when the agent is idle, done or blocked. An agent started by
the plugin is named after the sandbox, so `agent prompt api "..."` also
works.

Run a command in the guest without SSH:

```bash
brig exec herdr@api -- cat /work/api/README.md
```

## Settings

Set these in the environment herdr runs plugins with.

| Variable | Default | Effect |
| --- | --- | --- |
| `HERDR_BRIG_POLICY` | none | The policy the popup offers as its default |

## Limits

- **Network.** Under a policy with `egress: default: deny`, the guest reaches
  only the hosts and routes the policy names. A host it does not name does
  not resolve: `ssh git@github.com` fails with
  `Could not resolve hostname github.com`.
- **SSH to GitHub.** The ssh block sets `ForwardAgent no`, and the guest has
  no key. Use git over HTTPS through a policy with a `github-token` route,
  such as `devwork` above.
- **Session tokens.** Each SSH connection is a new `brig exec` with a new
  token. A pane keeps the token it started with. After a broker restart, a
  running agent gets `401 … the session token was refused`; start it again
  in a new pane.
- **Remove race.** A `brig exec` that herdr started before the disable can
  boot the sandbox again after `brig rm`. Remove waits for such execs, then
  watches `brig ls` for 15 s after each `brig rm` and removes the sandbox
  again if it is back, up to 5 times. What boots it a second time is not
  known.

## Development

| File | Contents |
| --- | --- |
| `herdr-plugin.toml` | Manifest: two actions and two popup panes |
| `bin/herdr-brig` | Implementation |

Actions have no terminal. Each action opens a popup pane (`new-form`,
`remove-form`) that runs the same script with one. The action passes the
focused pane's directory to the popup in `HERDR_BRIG_CWD`, read from
`HERDR_PLUGIN_CONTEXT_JSON`.

To work on a local checkout, uninstall the released plugin and link the
checkout. A linked plugin runs from that directory, so script edits apply on
the next invocation:

```bash
herdr plugin uninstall brig-sh.herdr-brig
herdr plugin link ~/code/herdr-brig
```

After a manifest change, relink:

```bash
herdr plugin unlink brig-sh.herdr-brig
herdr plugin link ~/code/herdr-brig
```

Debugging:

```bash
bash -n bin/herdr-brig
/bin/bash -n bin/herdr-brig                          # macOS bash 3.2
herdr plugin log list --plugin brig-sh.herdr-brig --limit 5
HERDR_BRIG_CWD=~/code/api bin/herdr-brig new-form    # run a form in your terminal
```

The plugin log records actions, not popups.

Rules:

- Input comes from the user, never from a guest. The plugin runs on the host
  with your herdr access.
- Pass input to `brig` and `herdr` as argv. Do not build shell strings from
  it.
- Stay compatible with bash 3.2: no `read -i`, associative arrays or
  `mapfile`.
- Parse output with `sed`, `awk` and `grep`, not `jq`. A herdr plugin process
  may not have your shell's `PATH`.

## License

Apache License 2.0. See [LICENSE](LICENSE).
