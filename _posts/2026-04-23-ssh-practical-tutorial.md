---
title: SSH Practical Tutorial
categories: [Technical]
math: false
mermaid: false
---

A hands-on guide to using SSH — not just to log in, but to manage clusters cleverly.

Throughout this tutorial we'll use two personas:

- **Alice** — the **client**. She wants to log into servers and get work done.
- **Bob** — the **server admin**. He runs a server and decides who gets in.

---

## 1. Part 1 — SSH 101

### 1.1 Client and Server: the basic picture

SSH is a protocol for running commands on a remote machine over an encrypted channel. One side initiates the connection (the **client**); the other side listens for connections (the **server**, which runs a daemon called `sshd`).

```
   ┌──────────────┐                          ┌──────────────┐
   │   Alice's    │   ssh alice@server       │   Bob's      │
   │   laptop     │  ──────────────────►     │   server     │
   │              │    (port 22, encrypted)  │              │
   │  ssh client  │  ◄──────────────────     │  sshd        │
   └──────────────┘                          └──────────────┘
```

That's it. Everything else in this tutorial is detail on top of that one picture.

---

### 1.2 The `~/.ssh` directory

Both Alice and Bob have a `~/.ssh` directory in their home folder. What's inside depends on the role they're playing in a given connection.

**Alice's `~/.ssh` (as a client)**

```
~/.ssh/
├── id_ed25519          # private key — NEVER share
├── id_ed25519.pub      # public key  — share freely
├── known_hosts         # fingerprints of servers Alice has connected to
└── config              # shortcuts and per-host defaults
```

**Bob's `~/.ssh` (on the server, for the account Alice logs into)**

```
~/.ssh/
└── authorized_keys     # list of public keys allowed to log in as this user
```

**The server itself also has a key: the host key**

User keys identify *a person*. **Host keys** identify *the server itself*. They're separate, and they live outside anyone's home directory — in `/etc/ssh/`:

```
/etc/ssh/
├── ssh_host_ed25519_key        # server's private host key — root-only
├── ssh_host_ed25519_key.pub    # server's public host key  — what clients see
├── ssh_host_rsa_key(.pub)      # legacy, often also present
└── sshd_config                 # the server daemon's config
```

These are generated automatically when `sshd` is installed — Bob doesn't normally touch them. When a client connects, `sshd` proves ownership of the matching private host key, which is how the client knows it's talking to the real server and not an impostor.

Symmetry to keep in mind:

```
   Users  have keys in  ~/.ssh/                 → proves "I am Alice"
   Hosts  have keys in  /etc/ssh/               → proves "I am that server"
```

The client's `~/.ssh/known_hosts` file (which we'll meet in Step 4) is just a record of host public keys the client has seen before.

**Permissions matter**

SSH will **refuse to use** keys and config files if permissions are too loose. This is a feature.

| Path                       | Required mode |
|----------------------------|---------------|
| `~/.ssh`                   | `700`         |
| `~/.ssh/authorized_keys`   | `600`         |
| `~/.ssh/id_ed25519`        | `600`         |
| `~/.ssh/id_ed25519.pub`    | `644`         |
| `~/.ssh/config`            | `600`         |

If something silently doesn't work, check permissions first.

---

### 1.3 Connecting a client to a new server

This is the core workflow. Four steps.

**Step 1 — Alice generates a keypair (once, on her laptop)**

```bash
ssh-keygen -t ed25519 -C "alice@laptop"
```

This creates two files:

- `~/.ssh/id_ed25519` — the **private** key. Treat it like a password.
- `~/.ssh/id_ed25519.pub` — the **public** key. It's just text, safe to share.

> Use `ed25519`. It's smaller and faster than RSA and is the modern default. Only fall back to RSA for ancient servers that don't support it.

#### A note on `-C`

The `-C` flag sets a **comment** on the key. It plays no role in the cryptography — it's just a human-readable label stored at the end of the public key:

```
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI...blob... alice@laptop
└── type ──┘ └──── actual public key ──────┘ └── comment ──┘
```

If you omit `-C`, `ssh-keygen` uses `$(whoami)@$(hostname)` from the machine where you ran it — e.g. `alice@Alices-MacBook-Pro.local`. That works, but a chosen label like `alice@laptop` is friendlier.

Why does the comment matter? When Bob has 30 keys piled into `authorized_keys`, the comment is the only way to tell which line belongs to whom — crucial when someone leaves and you need to revoke their access.

You can change the comment later without regenerating the key:

```bash
ssh-keygen -c -C "alice@laptop-2026" -f ~/.ssh/id_ed25519
```

**Step 2 — Alice sends her public key to Bob**

Email, chat, paste into a ticket — however. Public keys aren't secret.

```bash
cat ~/.ssh/id_ed25519.pub
# ssh-ed25519 AAAAC3Nza...long blob... alice@laptop
```

**Step 3 — Bob installs it on the server**

On the server, logged in as the user account Alice should use:

```bash
mkdir -p ~/.ssh && chmod 700 ~/.ssh
echo "ssh-ed25519 AAAAC3Nza... alice@laptop" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

Each line in `authorized_keys` is one public key. Add more users by appending more lines.

**Step 4 — Alice connects**

```bash
ssh alice@server.example.com
```

### 1.4 What happens on the first connection

Alice sees something like this:

```
The authenticity of host 'server.example.com (203.0.113.5)' can't be established.
ED25519 key fingerprint is SHA256:abc123...
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

This is the client asking: *do you trust that the machine on the other end really is the server you meant to reach?* If Alice says yes, the server's public host key is recorded in `~/.ssh/known_hosts`. From then on, SSH silently verifies it on every connection.

If that host key ever changes, SSH will refuse to connect with a loud warning. This usually means one of two things:

1. The admin legitimately rebuilt the server.
2. Someone is intercepting the connection.

Either way, it's worth investigating before clicking past.

**The whole dance in one picture**

```
   Alice's laptop                                 Bob's server
   ──────────────                                 ────────────

   id_ed25519         (private, stays on laptop)
   id_ed25519.pub     ───── email / paste ─────►  ~/.ssh/authorized_keys
                                                  (Bob appends the pubkey)

   ssh alice@server   ─── TCP :22 connect ─────►  sshd accepts
                      ◄────── challenge ────────  "prove you hold the
                                                   matching private key"
                      ──── signed response ────►  verify vs authorized_keys
                      ◄────── shell granted ────  match → spawn shell
   $ █
```

---

### 1.5 Quick win: the `~/.ssh/config` file

Typing `ssh alice@server.example.com -p 2222 -i ~/.ssh/id_ed25519` every time is painful. Create `~/.ssh/config`:

```ssh-config
Host mybox
    HostName server.example.com
    User     alice
    Port     2222
    IdentityFile ~/.ssh/id_ed25519
```

Now this works:

```bash
ssh mybox
```

A quick note on each field:

| Field          | What it means                                                        |
|----------------|----------------------------------------------------------------------|
| `HostName`     | The real address to connect to (DNS name or IP).                     |
| `User`         | Login username on the server.                                        |
| `Port`         | The port **the server's `sshd` is listening on**. Default is `22`. Some admins run `sshd` on a non-standard port like `2222` to reduce bot-scan log noise — if Bob did that, Alice must match it here. |
| `IdentityFile` | Which private key to use for this host.                              |

You can also set **defaults that apply to every host** by adding a `Host *` block to the same `~/.ssh/config` file. The `*` is a wildcard that matches any hostname.

Here's how a complete config file looks with both specific entries and defaults:

```ssh-config
# ~/.ssh/config  (on Alice's laptop)

Host mybox
    HostName server.example.com
    User     alice
    Port     2222
    IdentityFile ~/.ssh/id_ed25519

Host lab-*
    User     alice
    IdentityFile ~/.ssh/id_ed25519_lab

Host *
    ServerAliveInterval 60      # keepalive every 60s so NAT doesn't kill idle sessions
    AddKeysToAgent      yes     # auto-add keys to ssh-agent on first use
```

**Ordering matters.** SSH reads the file top-down and, for each option, **the first matching value wins**. So put specific hosts at the top and the `Host *` catch-all at the bottom — otherwise a default would override a more specific setting below it.

With this file, `ssh mybox` uses the `mybox` block *and* inherits `ServerAliveInterval` / `AddKeysToAgent` from `Host *`. `ssh lab-03` matches the `lab-*` block and also inherits the `Host *` defaults. Anything else just gets the defaults.

---

## 2. Part 2 — Managing a Cluster

The rest of this tutorial moves past single-server use. In the real world you rarely have just one machine — you have a cluster, and SSH has features specifically for that.

### 2.1 The jump host: a hub-and-spoke cluster

Once a cluster grows past a handful of machines, giving every VM its own public SSH endpoint becomes a liability:

- Every machine is a separate attack surface.
- Every machine needs its own firewall rules, host-key rotation, and log pipeline.
- Auditing "who logged in where and when" means pulling logs from N hosts.

The standard fix is a **jump host** (also called a *bastion host*): a single, hardened machine that is the only one reachable from the public internet. All other machines — the actual cluster — sit on a private network and accept SSH *only* from the jump host. Topologically it's a hub-and-spoke:

```
                       ┌───────────────┐
                       │    Alice's    │
                       │    laptop     │
                       └───────┬───────┘
                               │  public internet
                               │  (encrypted SSH)
                               ▼
                       ┌───────────────┐
                       │   jump host   │  ← only public SSH endpoint
                       │     (hub)     │    hardened, audited, minimal
                       └───────┬───────┘
                               │  private network
              ┌────────────────┼────────────────┬──────────────┐
              ▼                ▼                ▼              ▼
          ┌───────┐        ┌───────┐        ┌───────┐      ┌───────┐
          │ vm-01 │        │ vm-02 │        │ vm-03 │  ... │ vm-N  │
          └───────┘        └───────┘        └───────┘      └───────┘
                    (no public SSH — only reachable through the hub)
```

**Two ways to use a jump host — one of them is wrong**

**The naïve way — two manual hops.** From Alice's laptop:

```bash
$ ssh alice@jump.example.com      # step 1: log into the jump host
alice@jump:~$ ssh alice@vm-01     # step 2: from there, ssh to the target
```

This works, but it tempts people into putting Alice's **private key on the jump host** so step 2 doesn't need a password. Don't do that. The jump host is the most exposed machine in the whole cluster. If it ever falls, every private key sitting on it is gone too.

**The right way — `ProxyJump`.** SSH can open a TCP tunnel *through* the jump host to the target VM, and then run the SSH handshake end-to-end with that VM:

```bash
$ ssh -J alice@jump.example.com alice@vm-01
```

The key property: **the session with `vm-01` is encrypted end-to-end between Alice's laptop and `vm-01`**. The jump host is just a TCP relay — it cannot read the session, and Alice's private key never has to leave her laptop.

**The trust model**

Every hop is a **separate SSH authentication**. Alice's public key must be installed on both the jump host *and* each VM:

```
   Alice's laptop                              Jump host
   ──────────────                              ─────────
   id_ed25519       ──── auth hop 1 ────►      ~/.ssh/authorized_keys
                                               (contains Alice's pubkey)
                                                      │
                                                      │  TCP relay only
                                                      │  (no auth happens here)
                                                      ▼
                                               vm-01
                    ──── auth hop 2 ────►      ~/.ssh/authorized_keys
                                               (contains Alice's pubkey too)
```

The jump host does **not** forward or delegate credentials. These are two independent authentications — same laptop, same private key, two servers.

**Alice's client config**

This is where `~/.ssh/config` earns its keep. One pattern block covers the whole cluster:

```ssh-config
# ~/.ssh/config  (on Alice's laptop)

Host jump
    HostName jump.example.com
    User     alice
    IdentityFile ~/.ssh/id_ed25519

Host vm-*
    User     alice
    IdentityFile ~/.ssh/id_ed25519
    ProxyJump jump

Host *
    ServerAliveInterval 60
    AddKeysToAgent      yes
```

Now all of these just work:

```bash
ssh vm-01                         # interactive shell on vm-01
scp report.txt vm-03:/tmp/        # copy a file to vm-03
rsync -av data/ vm-07:/srv/data/  # rsync through the jump
```

Alice's laptop doesn't need to know the VMs' private IPs — the jump host resolves the names on the internal network.

**What Bob (the admin) has to do**

For the client side to work, Bob sets up three things:

1. **On the jump host:** Alice's pubkey goes in her `~/.ssh/authorized_keys`.
2. **On each VM:** Alice's pubkey goes in her `~/.ssh/authorized_keys` there too.
3. **Firewall:** the VMs' `sshd` only accepts connections from the jump host's internal IP.

Step 2 is tedious at any real scale — copying a key to 30 machines by hand is the sort of thing that gets skipped or botched. That's the problem the next section tackles.

**A common hardening trick**

Often Bob wants the jump host to be *nothing but* a jump host — no interactive shells, no file transfers. SSH supports this with `authorized_keys` options. Prefixing a key line with `restrict,port-forwarding` lets it tunnel TCP (which is what `ProxyJump` needs) but forbids everything else:

```
restrict,port-forwarding ssh-ed25519 AAAAC3Nza... alice@laptop
```

Alice can still run `ssh vm-01` through the jump host, but `ssh jump` on its own gets no shell. The jump host becomes a pure traffic relay with no useful attack surface if someone hijacks a key.

---

## 3. Part 3 — Practical cluster setup

You have the jump host pattern from Part 2. But how do you actually *produce* N cluster VMs, each with Alice's pubkey, your tools, and the right firewall rules — without repeating yourself N times?

The simplest pattern that works in every major cloud is the **golden image**: configure one VM perfectly, snapshot its disk, clone the snapshot to spin up the rest. Setup happens once; provisioning new nodes becomes a single API call.

### 3.1 The golden image pattern

**The workflow**

```
       ┌──────────────┐
       │ vm-template  │   ← configure ONCE
       │              │     • Alice's pubkey in authorized_keys
       │   (clean)    │     • tools, packages, agents
       │              │     • (host keys cleared before snapshot)
       └──────┬───────┘
              │  snapshot the disk
              ▼
       ┌──────────────┐
       │  disk image  │     stored by the cloud provider
       └──────┬───────┘
              │  clone N times
       ┌──────┴───────┬──────────────┬──────────────┐
       ▼              ▼              ▼              ▼
   ┌───────┐      ┌───────┐      ┌───────┐      ┌───────┐
   │ vm-01 │      │ vm-02 │      │  ···  │      │ vm-N  │
   └───────┘      └───────┘      └───────┘      └───────┘
   each boots with a fresh host key,
   but inherits Alice's pubkey for free
```

Every cloud calls this something slightly different, but it's the same mechanism:

| Cloud         | What they call it          |
|---------------|----------------------------|
| AWS           | AMI (Amazon Machine Image) |
| GCP           | Custom image               |
| Azure         | Managed image / capture    |
| DigitalOcean  | Snapshot                   |

**Step 1 — configure one VM**

Spin up a single VM — call it `vm-template` — on the same private network behind the jump host. Set up everything you want the whole cluster to have:

```bash
# On vm-template (reached via the jump host)

# 1. Create Alice's user and install her pubkey
sudo useradd -m alice
sudo mkdir -p /home/alice/.ssh && sudo chmod 700 /home/alice/.ssh
echo "ssh-ed25519 AAAAC3Nza... alice@laptop" \
    | sudo tee /home/alice/.ssh/authorized_keys
sudo chmod 600 /home/alice/.ssh/authorized_keys
sudo chown -R alice:alice /home/alice/.ssh

# 2. Install whatever the cluster needs
sudo apt update && sudo apt install -y tmux htop build-essential
```

> **Shortcut: most of step 1 can be done via the cloud provider's GUI.**
> When you launch a VM in AWS, GCP, Azure, DigitalOcean etc., the web console has an **"SSH keys"** (or "SSH public key") field. Paste Alice's `id_ed25519.pub` there and the cloud's `cloud-init` boot scripts will install it into the default user's `authorized_keys` automatically — no `useradd`/`tee`/`chmod` needed.
>
> The nuance: this installs the key for the **default cloud user** (`ubuntu` on Ubuntu images, `ec2-user` on Amazon Linux, `admin` on Debian, etc.), not for a user called `alice`. If that default is fine, the whole of step 1's first block collapses to one paste in the GUI. If you specifically want a user named `alice`, you still need to `useradd` and install the key manually — the GUI field only handles the default user.

**Step 2 — verify it works**

From Alice's laptop, using the `~/.ssh/config` from Part 2:

```bash
ssh vm-template    # should land her on vm-template via the jump host
```

Do this **before** snapshotting. Fixing a broken image across 30 clones is miserable; fixing it on a single template is trivial.

**Step 3 — clean up before snapshotting**

A running VM accumulates per-instance state that shouldn't be baked into the image. Skip this and your cluster will misbehave in surprising ways:

```bash
# On vm-template, as root, just before shutdown:

# 1. Remove SSH host keys — critical, see below
sudo rm -f /etc/ssh/ssh_host_*

# 2. Clear machine-id so each clone generates its own
sudo truncate -s 0 /etc/machine-id

# 3. Clear shell history, logs, cloud-init state
sudo rm -f /root/.bash_history /home/*/.bash_history
sudo journalctl --rotate && sudo journalctl --vacuum-time=1s
sudo cloud-init clean --logs 2>/dev/null || true
```

**Why remove the host keys?** Recall from Part 1 that host keys identify *the server itself*. If you snapshot without removing them, every clone boots up with **identical host keys**. Two problems:

- Any clone can cryptographically impersonate any other — they're literally the same server as far as SSH is concerned.
- Alice's `~/.ssh/known_hosts` sees the same fingerprint for `vm-01`, `vm-02`, `vm-03`… which either silently hides a real MITM later, or loudly fails as soon as an IP changes.

With `/etc/ssh/ssh_host_*` deleted, the standard `ssh-keygen -A` hook that runs on first boot of every major cloud image will generate a fresh, unique host keypair for each clone.

**Step 4 — snapshot and clone**

Shut `vm-template` down cleanly, then in your cloud provider's UI/CLI:

1. Create a snapshot (or image) of its disk.
2. Launch new VMs from that snapshot.

Each new VM boots with:

- A brand-new host key (thanks to step 3).
- A fresh internal IP from the cloud's DHCP.
- Alice's pubkey already in `/home/alice/.ssh/authorized_keys` (baked into the image).
- All the tools and packages, already installed.

No per-VM SSH setup. No key pushing. No package installing.

**Step 5 — connect from Alice's laptop**

Because each clone has its own fresh host key, Alice will hit the first-connection fingerprint prompt once per new VM. For a trusted internal cluster, suppress that with one line in `~/.ssh/config`:

```ssh-config
Host vm-*
    User     alice
    IdentityFile ~/.ssh/id_ed25519
    ProxyJump jump
    StrictHostKeyChecking accept-new
```

`accept-new` means: if the host key isn't in `known_hosts` yet, add it silently; if it's there and has changed, still refuse with a warning. Frictionless first-connects, without abandoning MITM protection.

**The trade-off**

**The win:** setup happens **once**, on the template. Spinning up `vm-04` becomes a single cloud API call. No SSH gymnastics, no re-pushing keys, no re-installing packages.

**The limit:** every VM is born identical. If you need per-VM differences — unique hostnames, node IDs, role-specific configs, credentials — you need another layer:

- **At launch time:** pass them in via the cloud's `user-data` / `cloud-init` mechanism (runs once on first boot, before Alice ever logs in).
- **After launch:** use a configuration management tool (Ansible, Salt, Chef) to customize each VM over SSH — which is the next section.

---

*Next: pushing keys to already-running VMs with `ssh-copy-id`, per-VM customization with `cloud-init` and Ansible, and running one command across the whole cluster at once.*
