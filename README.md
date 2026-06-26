# citadel-learning

Kubernetes learning cluster on bare metal Ubuntu Server, following the [KubeCraft HomeLabOS](https://www.skool.com/kubecraft) curriculum.

## Hardware

- **Host:** Gigabyte X399 DESIGNARE EX-CF, AMD Threadripper 1950X (16c/32t), 64GB DDR4
- **OS:** Ubuntu Server 26.04 LTS (bare metal, no hypervisor)
- **Storage:** ZFS RAIDZ1, 4× Seagate IronWolf 6TB

## Workflow

Directory-based cluster isolation using [devpod](https://devpod.sh/) + devcontainers. Each cluster repo has its own `kubeconfig` and `.envrc`; direnv exports `KUBECONFIG` automatically when you `cd` in.

```
.devcontainer/devcontainer.json   ← Debian base image
.envrc                            ← export KUBECONFIG=$(pwd)/kubeconfig
.gitignore                        ← excludes kubeconfig (never committed)
```

Tools (flux, kubectl, k9s, direnv) are installed via dotfiles on container creation, not baked into the image.

## Repos

| Repo | Purpose |
|---|---|
| `citadel-learning` (this one) | Course work and experimentation |
| `citadel-cluster` | Personal homelab services |

## Dev Log

See [DEVLOG.md](DEVLOG.md) for a running log of progress, decisions, and wrong turns.
