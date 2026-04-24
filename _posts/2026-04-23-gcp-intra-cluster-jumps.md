---
title: Intra-Cluster Jumps on GCP VMs with Internal IPs
categories: [Technical]
math: false
mermaid: false
---

A short companion to the [SSH Practical Tutorial]({% post_url 2026-04-23-ssh-practical-tutorial %}) — this time for a specific GCP setup that trips people up: a cluster of VMs that have **no external IPs**, only internal ones.

## The problem

You've SSH'd into some box in a GCP project — maybe a jump host, maybe a VM you reached through IAP — and now you want to hop over to another VM in the same VPC. But every VM in the cluster looks like this:

```
   vm-01   internal IP: 10.138.0.11   external IP: —
   vm-02   internal IP: 10.138.0.12   external IP: —
   vm-03   internal IP: 10.138.0.13   external IP: —
```

No external IP means plain `ssh alice@vm-02` from your laptop won't reach it. But from *inside* the VPC, the VMs can see each other fine.

## The command

From any machine inside the VPC (with `gcloud` installed and authenticated):

```bash
gcloud compute ssh <vm-name> --internal-ip --zone=<zone>
```

The `--internal-ip` flag tells `gcloud` to connect over the VM's internal address instead of trying — and failing — to find an external one. `--zone` is required because VM names are only unique within a zone.

Example:

```bash
gcloud compute ssh vm-02 --internal-ip --zone=us-central1-a
```

## Picking a user to land as

By default `gcloud compute ssh` logs you in as **the current OS user** (whoever you're shelled in as on the machine running the command). That's often *not* the user whose files and processes you actually care about on the target VM — shared clusters typically have service accounts or team user accounts that own the real state.

Once you're on the target VM, look at who actually exists on the box:

```bash
cat /etc/passwd
```

Each line is one user. The ones worth paying attention to have a real login shell (`/bin/bash`, `/bin/zsh`) and a home directory under `/home/`:

```
root:x:0:0:root:/root:/bin/bash
alice:x:1001:1001::/home/alice:/bin/bash
training-svc:x:1002:1002::/home/training-svc:/bin/bash
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
```

The `nologin` / `/nonexistent` entries are system accounts — ignore them.

## Switching to the right user

Once you've spotted the user you need:

```bash
sudo su <username>
```

For example:

```bash
sudo su training-svc
```

You're now operating as `training-svc`, with their environment, their `~`, and their file permissions. `exit` drops you back to your original user on the same VM.

## The whole flow

```
   your laptop
       │   (IAP / jump host / Cloud Shell — however you got in)
       ▼
   some VM inside the VPC
       │
       │   gcloud compute ssh vm-02 --internal-ip --zone=us-central1-a
       ▼
   vm-02, logged in as your own user
       │
       │   cat /etc/passwd        ← see who lives here
       │   sudo su training-svc   ← become the right user
       ▼
   vm-02, now acting as training-svc
```

Three commands, and you're where you need to be.
