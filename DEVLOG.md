# Dev Log

---

## 2026-06-26 — Container workflow confirmed; devpod mental model corrected

Ran `devpod up --recreate citadel-learning` to rebuild the container with the updated `devcontainer.json`. The terminal hung after the container came up — devpod holds the foreground SSH connection open rather than returning the prompt. Ctrl+C detaches cleanly; the container keeps running.

A batch of port-forwarding errors appeared during startup:

```
error Error port forwarding 6443: listen tcp 127.0.0.1:6443: bind: address already in use
error Error port forwarding 10250: listen tcp 127.0.0.1:10250: bind: address already in use
```

These look alarming but aren't. devpod attempts to forward k3s-related ports from the container to the host. Because the container runs with `--network=host`, it shares citadel's network stack entirely — k3s already owns those ports on the host, and the container and host are the same network namespace. devpod's forwarding attempts fail immediately, which is the correct outcome. kubectl works fine.

The other thing that needed correcting was where to run devpod commands from. The general devpod mental model is client on your workstation, Docker provider on the remote machine. That's not how this is set up. devpod is installed on citadel, at `~/.local/bin/devpod`, using the local docker provider. citadel is both the client and the host. The correct flow from fz55 is: `ssh citadel`, then `devpod ssh citadel-learning`. Running `devpod ssh citadel-learning` from fz55 gets `command not found`.

Inside the container, the default user is `vscode` — the devcontainers base image creates that user regardless of whether VS Code is involved. It's part of the devcontainer standard.

direnv needs a one-time approval after container creation:

```bash
direnv allow
```

After that, KUBECONFIG exports automatically on entry. Confirmed working:

```
kubectl get nodes
NAME      STATUS   ROLES           AGE     VERSION
citadel   Ready    control-plane   4d23h   v1.36.2+k3s1
```

Cluster is reachable from inside the container. The stack is ready: kubectl, flux, k9s, direnv, zsh, neovim — all present. Starting the Kubecraft HomeLabOS curriculum from here.

---

## 2026-06-26 — Two approaches to devcontainer tooling

Updated `devcontainer.json` to add Zsh and Neovim alongside the existing dotfiles-based tool setup. Added `"runArgs": ["--network=host"]` so `127.0.0.1:6443` resolves to citadel's k3s API server from inside the container. Copied `~/.kube/config` into the project root as `kubeconfig` (gitignored) — `.envrc` exports `KUBECONFIG=$(pwd)/kubeconfig` automatically on `cd`.

**ghcr.io feature registry note:** The curriculum uses devcontainer features hosted on `ghcr.io/devcontainers/features/` (Homebrew, common-utils). These require the Docker daemon to obtain an anonymous token from ghcr.io, which was denied on citadel — likely a registry auth configuration issue. The same outcomes are achieved via `postCreateCommand` using apt instead. Homebrew can be installed manually inside the container if needed:

```bash
NONINTERACTIVE=1 bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"
```

Current `devcontainer.json`:
```json
{
  "image": "mcr.microsoft.com/devcontainers/base:debian",
  "postCreateCommand": "sudo apt-get update -qq && sudo apt-get install -y zsh neovim",
  "runArgs": ["--network=host"]
}
```

Zsh is installed but bash remains the default shell — our bash-oriented dotfiles work without modification. Switch with `chsh -s $(which zsh)` inside the container to try the curriculum's shell.

Recreate the container to pick up changes:
```
devpod up --recreate citadel-learning
```

### Approach comparison

| Outcome | Curriculum approach | Our approach |
|---|---|---|
| Base image | `mcr.microsoft.com/devcontainers/base:debian` | Same |
| Shell | Zsh — `common-utils` devcontainer feature (ghcr.io) | apt install zsh; bash stays default; `chsh` to switch |
| Package manager in container | Homebrew — `homebrew` devcontainer feature (ghcr.io) | apt; Homebrew installable manually post-create |
| Editor | `brew install neovim` | `apt install neovim` in postCreateCommand (older version) |
| kubectl | `brew install kubectl` | Binary downloaded in dotfiles `setup.sh` → `/usr/local/bin/` |
| flux | `brew install fluxcd/tap/flux` | Binary downloaded in dotfiles `setup.sh` → `/usr/local/bin/` |
| k9s | `brew install k9s` | Binary downloaded in dotfiles `setup.sh` → `/usr/local/bin/` |
| direnv | `brew install direnv` | Binary downloaded in dotfiles `setup.sh` → `/usr/local/bin/` |
| KUBECONFIG wiring | `.envrc` exporting `$(pwd)/kubeconfig` | Same |
| Cluster network access | `--network=host` in `runArgs` | Same |
| Tool version pinning | Explicit in `Brewfile` or `brew install pkg@version` | Pinned download URLs in `setup.sh` (currently unpinned for some tools) |
| Portability | Homebrew handles Linux + macOS | Script is Linux-specific |
| Rebuild reproducibility | Self-contained in `devcontainer.json` | Split: `devcontainer.json` + devpod global dotfiles setting + dotfiles repo |
| Dotfile integration | Separate concern — dotfiles not involved by default | Integrated — devpod clones dotfiles on container create |

### Key tradeoff

The curriculum approach puts everything in `devcontainer.json` — the repo is self-documenting and someone can `git clone` + `devpod up` with no prior setup. Our approach relies on a devpod global setting (`dotfilesRepository`) that lives outside the repo, so reproduction requires knowing about that setting. The payoff is that the same dotfiles work everywhere and tool versions are managed centrally.

Both approaches share the same KUBECONFIG pattern: `.envrc` + gitignored `kubeconfig` file per project directory. That's the core isolation mechanism regardless of how the tools get there.

---

## 2026-06-26

Getting the devpod workflow running on citadel.

I installed Ubuntu 26.04 on citadel hesitantly; it was new and I knew that meant rough edges, but the LTS label made me less careful than I should have been. When Docker wouldn't set up through apt I did some research, late at night, not thoroughly. Found enough to confirm what I'd already half-decided: that I'd made a mistake not going with 24.04. I left it there.

A few days later I actually checked the assumption. Docker has supported Ubuntu 26.04 since day one. The problem was a previous install attempt had created
```
/etc/apt/sources.list.d/docker.list
```
but left it empty. Added the repo properly and it worked immediately.

The course I'm following (KubeCraft HomeLabOS) is built around a Raspberry Pi. I'm adapting it for a Threadripper 1950X with 64GB RAM, more headroom, same approach. The idea is directory-based cluster isolation via devpod: each cluster lives in its own repo with its own kubeconfig and `.envrc`. direnv exports `KUBECONFIG` automatically when you cd in, so you can't accidentally kubectl at the wrong cluster.

Tools (flux, kubectl, k9s, direnv) are installed into the container via a dotfiles setup script on creation, not baked into the image. devpod up pulled the Debian base image, cloned the dotfiles, ran the setup script, and everything came up.

A few other things that needed sorting:

devpod needs an SSH agent on the host to forward credentials into the container for cloning private repos. Set it up in the wrong place first; bashrc exits early for non-interactive shells, which is what devpod uses internally.
```
SSH_AUTH_SOCK needs to be in ~/.profile, not ~/.bashrc
```

devpod also kills idle workspaces by default. First successful run shut itself down after a minute because nothing was connected to the browser UI.
```
devpod context set-options -o EXIT_AFTER_TIMEOUT=false
```

citadel's GitHub SSH key started as a deploy key scoped to the private dotfiles repo. GitHub won't let the same key be both a deploy key and an account-level key, so I ended up promoting it to account-level, which also means citadel can push directly rather than relaying through another machine.

Container is running. k3s is already installed on citadel; next is starting the service, exporting the generated kubeconfig into the repo root, and running `direnv allow` so the container picks it up automatically.
