# 🚀 Professional VPS Configuration Guide: Security, Networking, and Deployment

[🇪🇸 Leer en Español](../es/vps-configuration.md)

This guide details the process of transforming a VPS into a secure and efficient production infrastructure. It is designed and tested specifically for **Ubuntu 24.04 LTS**, although most steps are compatible with other Debian-based distributions.

---

## 📋 Architecture Overview

- **Operating System:** Ubuntu 24.04 LTS (or similar distributions like Debian).
- **Access:** SSH via cryptographic keys (RSA / ED25519).  
- **Private Network:** Tailscale (Mesh VPN) + MagicDNS.  
- **Security:** UFW firewall blocking all administrative traffic outside the VPN.  
- **Deployment:** Docker + Dokploy (self-hosted PaaS).

---

## Phase 1: Initial Hardening and User Management (On the VPS)

Never operate a server as the root user. The first step is to create a user with sudo privileges.

### 1. System Update (VPS)

```bash
apt update && apt upgrade -y
```

### 2. System User Creation (VPS)

Replace `username` with your preferred alias.

```bash
adduser username
usermod -aG sudo username
```

### 3. SSH Key Transfer (VPS)

We copy the key you already use as root to the new user to avoid losing access.

```bash
mkdir -p /home/username/.ssh
cp /root/.ssh/authorized_keys /home/username/.ssh/
chown -R username:username /home/username/.ssh
chmod 700 /home/username/.ssh
chmod 600 /home/username/.ssh/authorized_keys
```

> Note: At this point, open a new terminal on your **local device** and verify access:
> `ssh username@PUBLIC_IP`

---

## Phase 2: Connectivity with Tailscale and MagicDNS

Tailscale creates a virtual private network (mesh VPN) based on WireGuard that allows your devices to communicate securely as if they were on the same local network, without needing to open ports on the public firewall.

### 1. Local Device Preparation

Before configuring the VPS, your working device must be inside the private network:

1. **Installation:** Download and install Tailscale on your local device from [tailscale.com/download](https://tailscale.com/download).
2. **Authentication:** Log in with your chosen account (Google, Microsoft, GitHub, etc.). This will be your "Tailnet" administrator.
3. **Status:** Ensure Tailscale is listed as "Connected" on your device.

### 2. VPS Installation

Now we will link the server to your same Tailscale account.

```bash
# Download and install the Tailscale binary (VPS)
curl -fsSL https://tailscale.com/install.sh | sh

# Start the service and generate the authentication link (VPS)
sudo tailscale up
```

**Required Action:**

- When running `sudo tailscale up`, a URL will appear in the VPS terminal.
- Copy and paste that URL into your **local device's** browser.
- Authorize the connection by clicking **Connect**. The VPS is now part of your private network.

### 3. Connectivity Verification (Local Device)

In your **local device's** terminal, run:

```bash
tailscale status
```

You should see your VPS listed with an internal IP address (range `100.x.y.z`). Test basic connection:

```bash
ping 100.x.y.z_OF_YOUR_VPS
```

### 4. MagicDNS Configuration (Optional - From Browser)

#### Step 1: Enable MagicDNS in the Tailscale Panel

1. Go to your [Tailscale Console](https://login.tailscale.com/admin/dns).
2. Go to the **DNS** tab.
3. Look for the **MagicDNS** section and click the **Enable MagicDNS** button.

#### Step 2: Personalize your VPS Name (Browser)

By default, Tailscale uses the name the server has in the system (e.g., `ubuntu-24-04-lts`). Let's give it a shorter one:

1. In the Tailscale console, go to the **Machines** tab.
2. Find your VPS in the list.
3. Click the three dots (`...`) at the end of the row and select **Edit machine name**.
4. Disable "Use OS hostname" and type a short name, for example: `server`.
5. Click **Save**.

#### Step 3: Simplify SSH access on your local device

Now let's configure your local device (PC, Laptop, etc.) so it recognizes that name and knows which user to use automatically.

- On your **local device**, open the terminal and edit (or create) your SSH configuration file:

```bash
nano ~/.ssh/config
```

- Paste the following block at the beginning of the file (replace `your_user` with the username you created on the VPS):

```plaintext
Host vps
    HostName server
    User your_user
```

- Save with `Ctrl+O`, `Enter` and exit with `Ctrl+X`.

**Result:** Now, to enter your server from your device, you just have to type:
`ssh vps`

---

## Phase 3: Perimeter Security (Firewall and SSH on the VPS)

We will configure the firewall so it only "listens" to your devices through Tailscale.

> Critical Warning: Do not activate `ufw` until you verify from another device that `tailscale up` works correctly and you can access through the Tailscale interface. Enabling `ufw` without checking can lock you out.

### 1. Firewall Configuration (UFW - VPS)

The key rule is to allow all traffic coming from the Tailscale virtual interface.

```bash
sudo apt install ufw -y
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow public web traffic (Optional if using Cloudflare or similar)
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Allow EVERYTHING from the Tailscale private network (SSH, Dokploy, etc.)
sudo ufw allow in on tailscale0

sudo ufw enable
```

### 2. SSH Hardening (VPS)

We disable password access and root access.

```bash
sudo nano /etc/ssh/sshd_config
```

Modify or add the following lines:

```text
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
```

Restart the service:

```bash
sudo systemctl restart ssh
```

### 3. SSH Port Change (Optional - VPS)

If you want an extra layer of "security by obscurity":
In `sshd_config`, change `Port 22` to a custom one (e.g., `Port 5522`).

#### Step 1: Choose a Port (VPS)

Choose a number between 1024 and 65535. Avoid common ones (8080, 3000, 8000).

- _Suggestion:_ Let's use **5522**.

#### Step 2: Edit SSH Configuration (VPS)

- Edit the configuration file:

```bash
sudo nano /etc/ssh/sshd_config
```

- Look for the line `#Port 22` (if it has a #, delete it).

- Change it to:

```plaintext
Port 5522
```

- Save (`Ctrl+O`, `Enter`) and exit (`Ctrl+X`).

#### Step 3: Adjust Firewall (UFW - VPS)

- Open the new port in the firewall:

```bash
sudo ufw allow 5522/tcp
```

>Note: If you want to maintain maximum security, trust the Tailscale rule (`tailscale0`) that already allows traffic.

- Delete the old rule for port 22 (Optional):

```bash
sudo ufw delete allow 22/tcp
```

#### Step 4: Restart SSH (VPS)

```bash
sudo service ssh restart
```

#### Step 5: Acid Test (Testing - Local Device)

1. **DO NOT CLOSE YOUR CURRENT VPS TERMINAL.**
2. Open a **new** terminal on your **local device**.
3. Try connecting specifying the port with `-p`:

```bash
ssh -p 5522 your_user@TAILSCALE_IP
```

If you entered successfully, congratulations! You have changed the port.

---

## Phase 4: Container Infrastructure and Dokploy (On the VPS)

Dokploy will act as your control panel to deploy projects from GitHub.

### 1. Docker Installation (VPS)

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker ${USER}
```

Note: After adding the user to the `docker` group, you must log out and back in, or run `newgrp docker`.

### 2. Dokploy Installation (VPS)

```bash
curl -sSL https://dokploy.com/install.sh | sudo sh
```

### 3. Dashboard Access (Browser - Local Device)

Thanks to the previous configuration, the Dokploy panel (port 3000) is not publicly accessible. Only you can enter using:

- `http://server:3000` (using MagicDNS)  
- or `http://100.x.y.z:3000` (Tailscale IP)

### 4. Disable access via `IP:port` (Optional - VPS)

To improve security, it is recommended to restrict access to Dokploy via the server's IP address and port.

>**Important**: Before this step, ensure you have a domain with HTTPS working correctly.

Once verified, run this command on the **server (VPS)**:

```bash
docker service update --publish-rm "published=3000,target=3000,mode=host" dokploy
```

---

## Phase 5: Workflow Optimization (Local Device)

To make your developer experience smooth from your local device or other trusted devices.

### 1. Access from mobile devices

- Install the Tailscale app on your mobile device.  
- Install an SSH client (e.g., Termius).  
- Create a new key on the device and add the public key to the `~/.ssh/authorized_keys` file on the **server**.

---

## 🛠 Maintenance and Best Practices

- **Automatic Updates (VPS):** Install `unattended-upgrades` for automatic security patches.  
- **Backups (Provider):** Enable weekly snapshots on your VPS provider.  
- **Docker (VPS):** Do not install databases or languages directly on the host; always use containers managed by Dokploy.

---

## Quick Security and Operational Notes

- Before `ufw enable` — verify `tailscale up` and remote access from another machine.  
- If you change the SSH port, leave a session open until you confirm you can reconnect.  
- After `usermod -aG docker` relogin or `newgrp docker`.  
- Ensure Dokploy is not publicly exposed: bind to `tailscale0` or behind a proxy.

---
