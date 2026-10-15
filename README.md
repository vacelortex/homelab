# Homelab

A self-hosted server running on an old ThinkPad, built to practise Linux administration and server hardening.

This repo covers the host itself: how it is installed, secured and accessed. Each service running on it has its own repo (see [Services](#services)).

This README is a running log of what I set up and why.

## Hardware and OS

| | |
|---|---|
| Machine | Lenovo ThinkPad (type 20M6) |
| OS | Debian stable, installed headless (no desktop environment) |
| Hostname | `homelab` |
| Network | Wi-Fi (the laptop has no standard Ethernet port) |
| Access | SSH over Tailscale, key-only |

## What I have done so far

### 1. Installation

- Wrote the Debian netinst image to a USB stick with Rufus in DD image mode, so the stick is an exact copy of the official image.
- Installed with only **SSH server** and **standard system utilities** selected. No desktop environment.
  - Why: a GUI idles at over 1 GB of RAM that the server can use instead, and fewer installed packages means a smaller attack surface and less to patch.
- Left the root password empty during install. This disables direct root login and gives my own user `sudo` rights.
  - Why: admin rights are requested per command instead of working in a permanent root shell.
- Used the whole disk without encryption.
  - Why: full-disk encryption needs a passphrase typed on the laptop's own keyboard at every boot, which does not work for a machine that should come back up unattended after a power cut. This is a trade-off I may revisit.

### 2. Base configuration

- Updated all packages: `sudo apt update && sudo apt full-upgrade`
- Stopped the laptop suspending when the lid is closed, using a systemd-logind drop-in. See [`config/logind/lid.conf`](config/logind/lid.conf).

### 3. SSH hardening

- Added my ed25519 public key to `~/.ssh/authorized_keys` on the server.
- Disabled password authentication and root login over SSH with a drop-in config. See [`config/ssh/hardening.conf`](config/ssh/hardening.conf).
  - Why: a key cannot be guessed, while a password on an open SSH port is a constant target for brute-force attempts.
- Verified the change:
  - `ssh user@host` logs in with the key.
  - `ssh -o PubkeyAuthentication=no user@host` is refused with `Permission denied (publickey)`.

### 4. Firewall

- Installed `ufw` with a default-deny policy for incoming traffic.

```
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw enable
```


- Why: services running on the host are not reachable from the network unless I have added a rule for them.
- Limitation: Docker writes its own firewall rules for published container ports, and these are evaluated before ufw's. A port published by a container is reachable even if ufw would deny it. I control container exposure through which ports are published and which address they bind to (see [Docker](#7-docker)).
- The port 22 rule was later narrowed to the Tailscale interface only (see [Remote access](#5-remote-access)).

### 5. Remote access

- Installed Tailscale on the server and on my laptop, which puts both on a private WireGuard-based network.
- I can now SSH to the server from any network using its Tailscale address, with the same key-only login as before.
  - Why: no ports are forwarded on my home router, so SSH is not exposed to the internet at all. Only devices signed in to my Tailscale network can reach it.
- Disabled key expiry for the server in the Tailscale admin console so it does not drop off the network unattended. Expiry stays on for my laptop.
- Added a host entry in `~/.ssh/config` on my laptop, so `ssh homelab` connects without typing the address.
- Verified by connecting from a mobile hotspot and from my school campus instead of my home Wi-Fi.
- Restricted SSH in ufw to the Tailscale interface only and removed the general port 22 rule, so devices on my home network can no longer reach SSH either.

### 6. Users and access

Other people who need a shell on the server get their own account. Nobody shares an account or a key.

- Each account is created without a password, so it can only be used with an SSH key.
- Accounts are not in the `sudo` group. Only my own user can administer the system.
- Network access is given by sharing the machine with their Tailscale account, so they can reach this server and nothing else of mine.
- The steps are scripted in [`scripts/add-user.sh`](scripts/add-user.sh):
```
sudo bash scripts/add-user.sh <username> '<their public key line>'
```
- To remove access: `sudo deluser --remove-home <username>`, and stop sharing the machine in Tailscale.
- Why: separate accounts mean least privilege per person, a record of who did what, and that one person's access can be removed without affecting anyone else.

### 7. Docker

- Installed Docker Engine and the Compose plugin from Docker's official apt repository, following the official instructions for Debian.
  - The repository is tied to Docker's signing key with a `Signed-By` line, so apt only accepts packages signed by that key. See [`config/apt/docker.sources`](config/apt/docker.sources).
- Verified with `sudo docker run hello-world`.
- No users are in the `docker` group. Docker commands are run with `sudo`.
  - Why: membership in the `docker` group is effectively root on the host, which matters when other people have accounts.
- Container ports are only published when they need to be reachable, and admin ports are bound to `127.0.0.1` or not published at all.

## Services

| Service | Repo |
|---|---|
| Minecraft server (Docker) | [mikkeliski/secure-mc](https://github.com/mikkeliski/secure-mc) |

## Planned

- [ ] Automatic security updates
- [ ] Put container ports under ufw's control (ufw-docker or the `DOCKER-USER` chain)
- [ ] Script the whole host setup so it can be reproduced on a fresh install

## Config files

```
config/
  logind/lid.conf        -> /etc/systemd/logind.conf.d/lid.conf
  ssh/hardening.conf     -> /etc/ssh/sshd_config.d/hardening.conf
  apt/docker.sources     -> /etc/apt/sources.list.d/docker.sources
scripts/
  add-user.sh            Create a key-only account without sudo rights
```

## What I learned

- Pasting several `sudo` commands at once can lose the later ones: the password prompt discards the input that was pasted after it. Running `sudo -v` first avoids this.
- `Permission denied (publickey)` with the right username means the key offered by the client does not match the one on the server. Comparing fingerprints with `ssh-keygen -lf` on both sides finds it faster than comparing the keys by eye.
- WSL has its own home folder and `~/.ssh`, separate from the Windows side, so a key made in one is not used by the other.
- `su` asks for the target account's password, not mine. With root disabled and key-only accounts, `sudo -i` and `sudo -u <user> -i` are the way to switch user.
