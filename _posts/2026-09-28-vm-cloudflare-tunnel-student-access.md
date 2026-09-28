---
title: "Setup: New VM with Cloudflare Tunnel Access for a Student"
categories: [Technical]
math: false
mermaid: false
---

**Prerequisite:** You have bought a domain on the Cloudflare website (e.g. `kernelgym.com`), so its DNS is managed by Cloudflare. This guide uses `kernelgym.com` as the example domain; substitute your own throughout.

Each step is tagged with who performs it: `🔧 admin` (you) or `🎓 student`.

Replace these placeholders throughout:

| Placeholder | Meaning |
|---|---|
| `<student>` | The student's Linux username (also used as the subdomain) |
| `<admin-user>` | Your own (admin) username on the VM, created in Step 2 (run `whoami` on the VM to check) |
| `<vm-name>` | The name of the new Compute Engine VM |
| `<TOKEN>` | The tunnel token Cloudflare gives you in Step 3a |

---

## Step 1: Create the VM on Google Cloud `🔧 admin`

1. Create a Compute Engine VM with a **Debian or Ubuntu** image, in the same project, network, and zone as your cluster (e.g. `us-central1-a`).
2. **Make sure the VM has outbound internet access.** The VM does *not* need any inbound access from the internet, because the tunnel only makes outbound connections.
   - No external IP is needed. Set **External IPv4 address** to **None** when creating the VM.

---

## Step 2: SSH into the new VM `🔧 admin`

From an existing machine in the cluster:

```bash
gcloud compute ssh <vm-name> --internal-ip --zone=us-central1-a
```

This creates **your** admin account (`<admin-user>`) on the VM, with a key stored on the machine you ran the command from. It does **not** create the student's account (that happens in Step 5). All remaining VM-side steps are done in this session.

---

## Step 3: Set up the Cloudflare Tunnel `🔧 admin`

### 3a. Create the tunnel (Cloudflare Zero Trust dashboard) `🔧 admin`

1. Go to **Networks → Tunnels → Create a tunnel → Cloudflared**, and give it a name. The name should ideally be related to the VM name.
2. Select **Debian** and **64-bit**. The dashboard shows the full set of install commands, ending with `sudo cloudflared service install <TOKEN>`. Copy them from there.

> ⚠️ **Treat the token as a secret.** Anyone with it can run a connector for your tunnel. Don't share it with the student or paste it anywhere public. If it leaks, refresh it in the tunnel's settings and re-run `service install` with the new one.

### 3b. Install the connector (on the VM) `🔧 admin`

Paste the commands you copied from the dashboard in 3a. The block below is ⚠️ **representative** ⚠️ of what they look like; use the dashboard's version, since it includes your actual token and any updates Cloudflare has made.

```bash
# Add Cloudflare's signing key
sudo mkdir -p --mode=0755 /usr/share/keyrings
curl -fsSL https://pkg.cloudflare.com/cloudflare-public-v2.gpg | sudo tee /usr/share/keyrings/cloudflare-public-v2.gpg >/dev/null

# Add Cloudflare's package repository
echo 'deb [signed-by=/usr/share/keyrings/cloudflare-public-v2.gpg] https://pkg.cloudflare.com/cloudflared any main' | sudo tee /etc/apt/sources.list.d/cloudflared.list

# Install cloudflared
sudo apt-get update && sudo apt-get install -y cloudflared

# Register it as a system service using the token from 3a
sudo cloudflared service install <TOKEN>
```

### 3c. Verify the connector is running (on the VM) `🔧 admin`

```bash
sudo systemctl status cloudflared
```

This should show `active (running)`. The tunnel should also show as **Healthy** in the dashboard.

### 3d. Route the hostname (dashboard) `🔧 admin`

In the tunnel's public hostname settings, add:

| Field | Value |
|---|---|
| Subdomain | `<student>` |
| Domain | `kernelgym.com` |
| Service type | `SSH` |
| URL | `localhost:22` |

Cloudflare creates the DNS record for `<student>.kernelgym.com` automatically.

### 3e. Restrict access with Cloudflare Access (dashboard, recommended) `🔧 admin`

1. Go to **Access → Applications → Add an application → Self-hosted**.
2. Set the domain to `<student>.kernelgym.com`.
3. Add a policy: Action **Allow**, Include **Emails**, containing the student's email address and your own.

Without this, anyone can reach the VM's SSH login prompt through Cloudflare (they would still need a valid key).

---

## Step 4: Verify the tunnel from your laptop `🔧 admin`

Add this to your laptop's `~/.ssh/config`:

```
Host <student>.kernelgym.com
   HostName <student>.kernelgym.com
   ProxyCommand /opt/homebrew/bin/cloudflared access ssh --hostname %h
```

Then run:

```bash
ssh -v <admin-user>@<student>.kernelgym.com
```

How to read the result:

| Result | Meaning |
|---|---|
| Browser login page appears | Cloudflare Access is working; log in with your email |
| You get a shell on the VM | ✅ **Tunnel works** |
| `Permission denied (publickey)` | ✅ **Tunnel works.** You reached the VM's SSH server; it rejected you only because your laptop's public key isn't in `<admin-user>`'s `~/.ssh/authorized_keys` (the key from Step 2 lives on the cluster machine, not your laptop) |
| Timeout or `cloudflared` error | ❌ Tunnel or hostname route needs fixing (recheck Step 3) |

To get a full login from your laptop, add your laptop's public key on the VM as `<admin-user>`:

```bash
echo "<your laptop's public key>" >> ~/.ssh/authorized_keys
```

---

## Step 5: Create the student's account `🔧 admin` `🎓 student`

### 5a. Generate an SSH key pair `🎓 student`

On their laptop (skip if they already have one), then send the **public** key to the admin:

```bash
ssh-keygen -t ed25519
cat ~/.ssh/id_ed25519.pub
```

### 5b. Create the account and install the key `🔧 admin`

1. On the VM, create their account and install the key:

   ```bash
   sudo adduser <student>
   sudo mkdir -p /home/<student>/.ssh
   echo "<student's public key>" | sudo tee /home/<student>/.ssh/authorized_keys
   sudo chown -R <student>:<student> /home/<student>/.ssh
   sudo chmod 700 /home/<student>/.ssh
   sudo chmod 600 /home/<student>/.ssh/authorized_keys
   ```

2. *(Optional)* If they need admin rights:

   ```bash
   sudo usermod -aG sudo <student>
   ```

---

## Step 6: Connect from the student's laptop `🎓 student`

The admin fills in the placeholders and sends the student the following:

> 1. Install `cloudflared` on your laptop (`brew install cloudflared` on a Mac; otherwise see Cloudflare's downloads page).
> 2. Run `which cloudflared` to find its path.
> 3. Add this to `~/.ssh/config`, replacing the path with the one from step 2 if it differs:
>
>    ```
>    Host <student>.kernelgym.com
>       User <student>
>       HostName <student>.kernelgym.com
>       ProxyCommand /opt/homebrew/bin/cloudflared access ssh --hostname %h
>    ```
>
> 4. Run `ssh <student>@<student>.kernelgym.com`. A browser window may open asking you to verify your email first. This also works in VS Code Remote-SSH.

The student confirms to the admin that they can log in.

---

## Checklist

- [ ] `🔧 admin` VM created with outbound internet access
- [ ] `🔧 admin` Logged in via `gcloud compute ssh`
- [ ] `🔧 admin` Tunnel created; token kept private
- [ ] `🔧 admin` `cloudflared` installed and tunnel shows **Healthy**
- [ ] `🔧 admin` Hostname `<student>.kernelgym.com` → `ssh://localhost:22`
- [ ] `🔧 admin` Access application with student's + your email
- [ ] `🔧 admin` Tunnel verified from your laptop as `<admin-user>`
- [ ] `🎓 student` Key pair generated; public key sent to admin
- [ ] `🔧 admin` Student account created with their public key
- [ ] `🔧 admin` Client instructions sent
- [ ] `🎓 student` `cloudflared` + SSH config set up; login confirmed