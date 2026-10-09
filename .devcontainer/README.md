# .devcontainer

Two things in one image, and the split is deliberate — see the header comment in
`Containerfile`.

## What came from the trusted templates

    template:  ghcr.io/infrashift/trusted-devcontainer-templates/python   (features, digests)
    editor:    the neovim-go template's terminal stack (tmux, neovim), plus
               pyrefly (a native Rust type checker + language server, installed userland with uv -- no Node) and the lazyvim feature at 1.2.2 with extras=lang.python

A `devcontainer.json` cannot *reference* a template at build time -- a template
is applied, and what it produced is what is committed here. Every feature is
digest-pinned, and every one depends on the same `bootstrap` digest.
`scripts/test.sh` proves every reference is still pinned.

## What this repository changed, and why each one

| Change | Why |
| --- | --- |
| The template's `sshd` feature is **not** declared | The workspace runs the platform's own sshd: the devpod jobspec forces `entrypoint.sh` as root, and `config/sshd_config` *replaces* `/etc/ssh/sshd_config`. Two SSH setups, one of them dead |
| The dnf `upgrade` and `install` are two `RUN` steps | The forge's pipeline mounts pinned repo files over `/etc/yum.repos.d` per step, and an upgrade rewrites them with Fedora's stock metalinks inside its own step (neovim-go seed, pipeline 25) |
| `containerUser: user` (uid 1001, **gid 0**) | The platform contract, assumed by the portal's `WORKSPACE_SSH_USER`, `sshd_config`, the jobspec's volume-init chown and `/home/user/workspace` |
| `openssh-server`, `openssh-clients` | Not in the trusted base. Without the server the workspace refuses every connection; without the client `git push` says `ssh: command not found` |
| `entrypoint.sh`, `config/sshd_config`, `config/ssh-login.sh` | The workspace runtime contract — the third is the `ForceCommand` the second names |
| `workspace-skel/` | Copied into an EMPTY host volume by the jobspec's prestart task |
| `/etc/profile.d/local-bin.sh`, `/etc/profile.d/neovim-python.sh` | Every feature installs into, or links into, `~/.local/bin`; sshd starts the login shell without it. EDITOR/VISUAL=nvim |
| `services.json` + `services/db/` | A PostgreSQL companion, as the vscode `python` seed carries: the forge builds it beside the devcontainer and the devpod root deploys it next to the workspace (`DB_HOST`/`DB_PORT` in the login shell). The `postgresql` package is its client (`psql`) |
| `python` feature at 1.7.1 (`sha256:7303d322…`), ahead of the template's 1.7.0 | 1.7.0 skipped `uv python install` when `uv-ruff` had already fetched the interpreter as a tool dependency, leaving no `~/.local/bin/python3.14` (trusted-devcontainer-features#22). The template moves at its next release |

## Logging in

An interactive terminal login -- `ssh -t`, or the portal's `ssh <workspace>` --
lands in a tmux session named `dev`: Neovim on the left, a shell on the right.
pyrefly (types) and ruff (lint + format) attach to Python buffers; both are native binaries on `PATH`, installed userland under `~/.local` -- no Node anywhere in the image. Neovim downloads nothing at runtime (Mason is off; every plugin,
parser and server was installed when the image was built). `ssh host cmd`, VS
Code's server and its integrated terminal get a plain shell. To skip the layout:

    ssh -t <workspace> DEV_SESSION=off bash -l                               # one login
    mkdir -p ~/.config/dev-session && touch ~/.config/dev-session/disabled    # always

`~/.config` is image-owned and is reset by a redeploy; to make a change stick,
make it here, in this repository, and let the forge build it.

## What the image carries, for the devpod verify

    WORKSPACE_TOOLS=nvim,tmux,python3.14,uv,ruff,pyrefly,make,jq,yq,git,git-lfs,syft,grype

`tmux` in that list also turns on the check that an interactive login lands in
the layout and a non-interactive one does not.

## The three copies

`entrypoint.sh`, `config/sshd_config` and `config/ssh-login.sh` are copies of
`terraform/live/devpod-vscode/container/config/`. **If the devpod root's copies
change, these must change with them.**
