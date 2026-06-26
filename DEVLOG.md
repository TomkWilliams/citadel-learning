# Dev Log

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
