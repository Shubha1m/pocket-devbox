# pocket-devbox — Turn Your Android Phone into an Always-On Claude Code Server

This is a battle-tested, step-by-step guide for turning any Android phone into a dedicated Claude Code server you can SSH into from any computer — from anywhere in the world. Written from a real setup session, including every error hit and how it was resolved.

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Phase 1: Termux & Ubuntu Setup (On Phone)](#phase-1-termux--ubuntu-setup-on-phone)
- [Phase 2: Node.js & Claude Code (On Phone)](#phase-2-nodejs--claude-code-on-phone)
- [Phase 3: SSH Server (On Phone)](#phase-3-ssh-server-on-phone)
- [Phase 4: Tailscale Tunnel (Phone + Mac)](#phase-4-tailscale-tunnel-phone--mac)
- [Phase 5: First SSH Connection (From Mac)](#phase-5-first-ssh-connection-from-mac)
- [Phase 6: SSH Key Authentication (From Mac)](#phase-6-ssh-key-authentication-from-mac)
- [Phase 7: Disable Password Auth (From SSH Session)](#phase-7-disable-password-auth-from-ssh-session)
- [Phase 8: SSH Convenience Config (On Mac)](#phase-8-ssh-convenience-config-on-mac)
- [Phase 9: tmux Session Persistence (On Phone via SSH)](#phase-9-tmux-session-persistence-on-phone-via-ssh)
- [Phase 10: Auto-Start on Boot (On Phone via SSH)](#phase-10-auto-start-on-boot-on-phone-via-ssh)
- **Part 2: Git & Multi-Device Access (Optional but Recommended)**
- [Phase 11: Git SSH Keys (On Phone via SSH)](#phase-11-git-ssh-keys-on-phone-via-ssh)
- [Phase 12: Clone Repos & Work (On Phone via SSH)](#phase-12-clone-repos--work-on-phone-via-ssh)
- [Phase 13: Adding More Devices](#phase-13-adding-more-devices)
- [Why Git and Not SSHFS / Folder Mounting](#why-git-and-not-sshfs--folder-mounting)
- [Errors We Hit & How We Fixed Them](#errors-we-hit--how-we-fixed-them)
- [Important Gotchas](#important-gotchas)
- [Daily Workflow](#daily-workflow)
- [Post-Setup Hardening Checklist](#post-setup-hardening-checklist)
- [Security Architecture](#security-architecture)

---

## Overview

**What this does:** Your Android phone runs Termux → proot Ubuntu → Node.js → Claude Code. You SSH into it from your Mac (or any device). Your Mac is just a thin client displaying text. Claude Code runs entirely on the phone.

**Why:** Free ARM server with unlimited bandwidth. No monthly VPS costs. Works from anywhere via Tailscale. Your Mac saves battery since it's only rendering terminal text.

**Cost:** $0/month (you already own the phone and Tailscale is free for personal use).

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Your Mac (Thin Client)                │
│                  Terminal / Termius / VS Code            │
└──────────────┬──────────────────────────────────────────┘
               │
               │  SSH (AES-256 encrypted, port 8022)
               │  Key-only auth (password disabled)
               │
       ┌───────┴────────┐
       │  Tailscale      │
       │  WireGuard      │
       │  Encrypted      │
       │  Tunnel         │
       │  (works over    │
       │  any network)   │
       └───────┬─────────┘
               │
┌──────────────┴──────────────────────────────────────────┐
│              Android Phone (Server)                      │
│              Tailscale IP: 100.x.x.x                    │
│                                                         │
│  ┌─────────────────────────────────────────────────┐    │
│  │  Termux (Android terminal emulator)             │    │
│  │  ├── sshd (port 8022, key-auth only)            │    │
│  │  ├── sv-enable sshd (auto-start)                │    │
│  │  ├── termux-wake-lock (prevent CPU sleep)       │    │
│  │  │                                              │    │
│  │  │  ┌───────────────────────────────────────┐   │    │
│  │  │  │  proot-distro (Ubuntu 25.10)          │   │    │
│  │  │  │  ├── Node.js 20.x                     │   │    │
│  │  │  │  ├── Claude Code 2.x                  │   │    │
│  │  │  │  ├── tmux (session persistence)       │   │    │
│  │  │  │  └── ~/projects/                      │   │    │
│  │  │  └───────────────────────────────────────┘   │    │
│  │  └──────────────────────────────────────────────┘    │
│  Android (full-disk encryption)                         │
└─────────────────────────────────────────────────────────┘
```

**Encryption layers (outside-in):**
1. **Tailscale / WireGuard** — encrypts all traffic between Mac and phone across the internet
2. **SSH** — AES-256 encrypted channel with Ed25519 key authentication
3. **Android full-disk encryption** — protects data at rest if the phone is physically stolen

---

## Prerequisites

| Item | Details |
|------|---------|
| Android phone | Any modern Android phone (ARM64). Keep plugged in with battery protection enabled. |
| Mac | Any Mac with Terminal. |
| Tailscale account | Free at tailscale.com. Same account on both devices. |
| Claude account | Claude Pro, Max, or Teams for Claude Code access. |
| Termux | Install from the Play Store or F-Droid. Both work — Play Store is easier. |

---

## Phase 1: Termux & Ubuntu Setup (On Phone)

Everything in this phase happens directly on the phone screen.

### 1.1 Install Termux

Download and install Termux from the [Play Store](https://play.google.com/store/apps/details?id=com.termux) or from [F-Droid](https://f-droid.org/en/packages/com.termux/). Either works.

> **Note:** Some guides warn against the Play Store version. In our setup, the Play Store version worked fine.

Open Termux. You'll see a terminal with a `~ $` prompt.

### 1.2 Update Termux

```bash
pkg update && pkg upgrade
```

Say `Y` to any prompts.

### 1.3 Install proot-distro and Ubuntu

```bash
pkg install proot-distro
proot-distro install ubuntu
```

This downloads and extracts a full Ubuntu ARM64 filesystem. Takes a few minutes.

> **If you need to start over** (e.g., broken Ubuntu install):
> ```bash
> proot-distro remove ubuntu
> proot-distro install ubuntu
> ```
> This wipes the Ubuntu filesystem completely but doesn't touch Termux itself.
>
> **Watch for typos** — `proot-distro remove ubunto` (misspelled) will give:
> ```
> Error: unknown distribution 'ubunto' was requested to be removed.
> ```
> Just fix the spelling and retry.

### 1.4 Log into Ubuntu

```bash
proot-distro login ubuntu
```

You should see:
```
root@localhost:~#
```

You might see a warning:
```
Warning: CPU doesn't support 32-bit instructions, some software may not work.
```
This is normal on ARM64 — ignore it. It just means 32-bit x86 software won't run, which doesn't affect anything we're doing.

### 1.5 Update Ubuntu

```bash
apt update && apt upgrade -y
```

### 1.6 Install basic tools

```bash
apt install -y curl git
```

### 1.7 Validate Phase 1

```bash
cat /etc/os-release   # Should show Ubuntu
which curl git         # Both should return paths
```

---

## Phase 2: Node.js & Claude Code (On Phone)

Still inside proot Ubuntu (`root@localhost:~#` prompt).

### 2.1 Install Node.js 20

```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | bash -
apt install -y nodejs
```

### 2.2 Verify Node.js

```bash
node --version   # Expected: v20.x.x
npm --version    # Expected: 10.x.x
```

**Actual output from our setup:**
```
v20.20.2
10.8.2
```

If npm shows a "new major version available" notice, ignore it — don't upgrade npm separately, it can break things in proot.

### 2.3 Install Claude Code

```bash
npm install -g @anthropic-ai/claude-code
```

**Expected output:**
```
added 3 packages in 6s
```

### 2.4 Verify Claude Code

```bash
claude --version
```

**Expected:** `2.x.x (Claude Code)` (version number may differ)

### 2.5 Authenticate Claude Code

```bash
claude
```

This opens an authentication flow. It will display a URL — open it in a browser (on the phone or any device), log in with your Claude account, and authorize. Once authenticated, you'll see the Claude Code interactive prompt.

Type `/exit` or press `Ctrl+C` to exit.

### 2.6 Validate Phase 2 — End-to-End Test

```bash
claude -p "Say hello"
```

**Expected:** A response like `Hello! How can I help you today?`

If this works, the Claude Code pipeline is fully operational. Exit proot:

```bash
exit
```

You should be back at the Termux `~ $` prompt.

---

## Phase 3: SSH Server (On Phone)

Back in Termux (`~ $` prompt), NOT inside proot.

### 3.1 Install OpenSSH

```bash
pkg install openssh
```

**Important:** OpenSSH must be installed in Termux, not inside proot Ubuntu. Termux is the layer that manages the SSH daemon and has access to real networking.

### 3.2 Set a temporary password

```bash
passwd
```

Choose any password. This is temporary — you'll disable password auth later after setting up SSH keys.

### 3.3 Start the SSH server

```bash
sshd
```

**No output means success.** If sshd starts correctly, it prints nothing. Termux runs sshd on **port 8022** (not the standard 22).

You may see a message about `sv-enable sshd` — that's for auto-start, which we'll configure later.

### 3.4 Validate Phase 3

At this point sshd is running on the phone, but you can't reach it from your Mac yet because the phone is on mobile data (carrier NAT blocks inbound connections). That's what Phase 4 solves.

---

## Phase 4: Tailscale Tunnel (Phone + Mac)

### 4.1 Why Tailscale?

When your phone is on mobile data, it gets a carrier-internal IP (like `192.0.0.8`) that your Mac can't reach. Running `ifconfig` in Termux shows:

```
rmnet_data7: flags=65<UP,RUNNING>  mtu 1436
        inet 192.0.0.8  netmask 255.255.255.224
```

There's no `wlan0` interface because the phone isn't on Wi-Fi. Even on Wi-Fi, if your Mac is on a different network, it still can't reach the phone directly.

Tailscale solves this by creating a WireGuard-encrypted tunnel between your devices. Each device gets a stable `100.x.x.x` IP that always works, regardless of what network either device is on.

**Tailscale is free** for personal use (up to 100 devices).

### 4.2 Install Tailscale on the phone

1. Open the **Play Store** on your phone
2. Search for **Tailscale**
3. Install and open it
4. Sign up / sign in
5. Toggle it **ON**

### 4.3 Install Tailscale on your Mac

Download from the **Mac App Store** (search "Tailscale").

Sign in with the **same account** you used on the phone.

> **Alternative (Homebrew):**
> ```bash
> brew install --cask tailscale
> ```
> We initially installed via `brew install tailscale` (the CLI version) but switched to the App Store version for the menu bar UI. If you want to switch:
> ```bash
> brew uninstall tailscale
> ```
> Then install from the App Store.

### 4.4 Get the phone's Tailscale IP

Open the Tailscale app on your phone. It shows your device's Tailscale IP — something like `<phone-tailscale-ip>`.

**Write this down.** This is the IP you'll use for all SSH connections.

### 4.5 Validate Phase 4

From your Mac terminal:

```bash
ping <phone-tailscale-ip>    # Replace with your phone's Tailscale IP
```

If you get responses, the tunnel is working. Press `Ctrl+C` to stop.

---

## Phase 5: First SSH Connection (From Mac)

### 5.1 Connect via SSH

From your **Mac** terminal:

```bash
ssh -p 8022 <termux-username>@<phone-tailscale-ip>
```

> **How to find your Termux username:** On the phone in Termux, run `whoami`. It returns something like `<termux-username>`. This is Android's internal user ID for the Termux app — it's normal and expected.

### 5.2 Host fingerprint verification

On first connection, you'll see:

```
The authenticity of host '[<phone-tailscale-ip>]:8022 ([<phone-tailscale-ip>]:8022)' can't be established.
ED25519 key fingerprint is: SHA256:<your-phone-host-fingerprint>
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

Type `yes`. This saves the phone's fingerprint so SSH can detect if something changes (man-in-the-middle protection).

> **Best practice:** Before typing `yes`, verify the fingerprint by running this on the phone in Termux:
> ```bash
> ssh-keygen -lf $PREFIX/etc/ssh/ssh_host_ed25519_key.pub
> ```
> The fingerprints should match.

### 5.3 Enter the password

Enter the password you set in Phase 3. You should see the Termux welcome message:

```
Welcome to Termux
...
~ $
```

**You're now controlling your phone from your Mac.**

### 5.4 Validate Phase 5

Test running a command:

```bash
proot-distro login ubuntu
claude -p "Say hello"
```

If you get a response, everything works through the full stack: Mac → Tailscale → SSH → Termux → proot → Claude Code.

---

## Phase 6: SSH Key Authentication (From Mac)

Password auth is insecure and inconvenient. SSH keys are both stronger and faster.

### 6.1 Generate an SSH key pair on your Mac

Open a **new terminal tab on your Mac** (Cmd+T). Make sure you see your Mac prompt:

```
yourname@your-mac ~ %
```

**NOT** the Termux `~ $` prompt. This is critical — the key must be generated on your Mac, not on the phone.

> **Common mistake:** If you're already SSH'd into the phone and run `ssh-keygen` there, you'll generate keys on the phone instead of your Mac. The prompt `~ $` looks similar but check the path — the phone will show `/data/data/com.termux/files/home/.ssh/id_ed25519`, your Mac will show `/Users/yourname/.ssh/id_ed25519`.

```bash
ssh-keygen -t ed25519 -a 100 -C "mac-to-s26-ultra"
```

- **`-t ed25519`**: Modern, fast, secure key type
- **`-a 100`**: 100 rounds of key derivation (more brute-force resistant)
- **`-C "mac-to-s26-ultra"`**: Label so you remember what this key is for

**Prompts:**

```
Enter file in which to save the key (/Users/<your-mac-username>/.ssh/id_ed25519):
```
Press **Enter** to accept the default.

```
Enter passphrase (empty for no passphrase):
```
Either set a passphrase (recommended — acts as a second factor) or press Enter to skip. If you set one, you'll enter it each time you SSH in.

**Output:**
```
Your identification has been saved in /Users/<your-mac-username>/.ssh/id_ed25519
Your public key has been saved in /Users/<your-mac-username>/.ssh/id_ed25519.pub
The key fingerprint is:
SHA256:<your-key-fingerprint> mac-to-s26-ultra
```

### 6.2 Copy the public key to the phone

From the same **Mac** terminal tab:

```bash
ssh-copy-id -p 8022 <termux-username>@<phone-tailscale-ip>
```

Enter the Termux password when prompted.

**Expected output:**
```
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/Users/<your-mac-username>/.ssh/id_ed25519.pub"
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed
...
Number of key(s) added:        1
```

### 6.3 Test key-based login

```bash
ssh -p 8022 <termux-username>@<phone-tailscale-ip>
```

If you set a passphrase, you'll see:
```
Enter passphrase for key '/Users/<your-mac-username>/.ssh/id_ed25519':
```

This is your **key passphrase** — not the Termux password. That's the correct behavior. Enter it and you should get the Termux welcome prompt.

---

## Phase 7: Disable Password Auth (From SSH Session)

Now that key auth works, disable password login to prevent brute-force attacks.

### 7.1 Add hardened SSH settings

From inside the SSH session (Termux `~ $` prompt):

```bash
cat >> $PREFIX/etc/ssh/sshd_config << 'EOF'

# Hardened settings
PasswordAuthentication no
PubkeyAuthentication yes
MaxAuthTries 3
ClientAliveInterval 60
ClientAliveCountMax 3
EOF
```

**What each setting does:**

| Setting | Purpose |
|---------|---------|
| `PasswordAuthentication no` | Blocks all password login attempts |
| `PubkeyAuthentication yes` | Explicitly enables key-based auth |
| `MaxAuthTries 3` | Disconnects after 3 failed attempts (default was 6) |
| `ClientAliveInterval 60` | Sends a keepalive every 60 seconds |
| `ClientAliveCountMax 3` | Disconnects after 3 missed keepalives (3 minutes of silence) |

### 7.2 Restart sshd

```bash
pkill sshd && sshd
```

> **Warning:** This will kill your current SSH session immediately:
> ```
> Connection to <phone-tailscale-ip> closed by remote host.
> Connection to <phone-tailscale-ip> closed.
> ```
> This is expected. But sshd may not restart properly from a killed session.

### 7.3 If SSH connection is refused after restart

If you try to SSH back in and get:
```
ssh: connect to host <phone-tailscale-ip> port 8022: Connection refused
```

**Fix:** Open the Termux app directly on the phone and run:
```bash
sshd
```
No output means it started successfully. Then try SSH from your Mac again.

This happened in our setup because `pkill sshd` killed the daemon but the `&& sshd` part didn't reliably execute after the session dropped.

### 7.4 Verify key auth still works

From your Mac:
```bash
ssh -p 8022 <termux-username>@<phone-tailscale-ip>
```

Should ask for your **key passphrase** (if set) and let you in.

### 7.5 Verify password auth is blocked

From your Mac:
```bash
ssh -p 8022 -o PubkeyAuthentication=no <termux-username>@<phone-tailscale-ip>
```

**Expected:**
```
<termux-username>@<phone-tailscale-ip>: Permission denied (publickey,keyboard-interactive).
```

If you see `Permission denied` — password auth is successfully disabled. Your server is now key-only.

---

## Phase 8: SSH Convenience Config (On Mac)

Instead of typing `ssh -p 8022 <termux-username>@<phone-tailscale-ip>` every time, configure a shortcut.

### 8.1 Add SSH config entry

On your **Mac** (not in an SSH session):

```bash
cat >> ~/.ssh/config << 'EOF'

Host phone
    HostName <phone-tailscale-ip>
    Port 8022
    User <termux-username>
    IdentityFile ~/.ssh/id_ed25519
    ServerAliveInterval 60
    ServerAliveCountMax 3
EOF
```

> **Replace `<phone-tailscale-ip>`** with your phone's actual Tailscale IP.
>
> **Replace `<termux-username>`** with your actual Termux username (from `whoami`).

**What each setting does:**

| Setting | Purpose |
|---------|---------|
| `HostName` | The phone's Tailscale IP |
| `Port 8022` | Termux's SSH port (not standard 22) |
| `User` | Termux username (Android app user ID) |
| `IdentityFile` | Path to your private key |
| `ServerAliveInterval 60` | Sends keepalive every 60s to prevent idle disconnects |
| `ServerAliveCountMax 3` | Drops connection after 3 missed keepalives |

### 8.2 Test the shortcut

```bash
ssh phone
```

That's it. Two words to SSH into your phone from anywhere in the world.

---

## Phase 9: tmux Session Persistence (On Phone via SSH)

tmux lets you start a Claude Code session, detach, close your Mac, and reattach later — the session keeps running on the phone.

### 9.1 Install tmux

SSH in, enter proot, install tmux:

```bash
ssh phone
proot-distro login ubuntu
apt install -y tmux
```

### 9.2 Start a named tmux session

```bash
tmux new -s claude
```

Your terminal is now inside a tmux session. You'll see a green status bar at the bottom.

### 9.3 Run Claude Code inside tmux

```bash
claude
```

Work normally. Claude Code is now running inside a persistent session.

### 9.4 Detach from tmux

Press **Control+B**, release both keys, then press **D**.

> **This is not Ctrl+B+D simultaneously.** It's a two-step sequence:
> 1. Press and release `Control+B` (this is the tmux prefix)
> 2. Press `D` (this sends the "detach" command)
>
> On Mac, use the **Control** key (not Cmd).

You'll see:
```
[detached (from session claude)]
```

The session is still running in the background.

### 9.5 Reattach to the session

```bash
tmux attach -t claude
```

Your Claude Code session is exactly where you left it.

### 9.6 Important tmux limitations

- **tmux sessions survive:** SSH disconnects, Mac sleeping, network interruptions (as long as the SSH keepalive maintains the outer connection)
- **tmux sessions do NOT survive:** Phone reboots, Termux being killed by Android, exiting proot
- **Each `proot-distro login ubuntu` is a new environment.** If you exit proot and log back in, previous tmux sessions are gone. The tmux server runs inside proot, so it only lives as long as that proot session.

---

## Phase 10: Auto-Start on Boot (On Phone via SSH)

Ensure sshd starts automatically whenever Termux opens or the phone reboots.

### 10.1 Install termux-services

From the Termux prompt (not inside proot):

```bash
pkg install termux-services
```

### 10.2 Enable sshd auto-start

```bash
sv-enable sshd
```

> **If you get:** `fail: sshd: unable to change to service directory: file does not exist`
>
> **Fix:** Close and reopen the Termux app, then run `sv-enable sshd` again. The service manager needs a fresh Termux session to initialize after installation.

No output means success.

### 10.3 Set up wake lock

Prevent Android from sleeping the CPU:

```bash
termux-wake-lock
```

Create a boot script as a backup:

```bash
mkdir -p ~/.termux/boot
cat > ~/.termux/boot/start.sh << 'EOF'
#!/data/data/com.termux/files/usr/bin/sh
termux-wake-lock
EOF
chmod +x ~/.termux/boot/start.sh
```

### 10.4 Android battery settings (Do on the phone)

- **Settings → Battery → Termux → Set to "Unrestricted"**
- **Settings → Apps → Termux → Battery → Turn OFF "Optimize battery usage"**
- **Do NOT dismiss the Termux persistent notification** — it keeps the app alive

---

# Part 2: Git & Multi-Device Access (Optional but Recommended)

Part 1 gives you a working Claude Code server you can SSH into from your Mac. Part 2 makes it a true portable dev environment — your repos live on the phone, synced via Git, accessible from any device in the world.

**Why do this:**
- Your phone becomes the single source of truth for all project code
- Any device (Mac, Windows, iPad) can SSH in and work
- Git provides backup, sync, and version history
- You're not dependent on any one computer being on

---

## Phase 11: Git SSH Keys (On Phone via SSH)

Set up a dedicated SSH key so the phone can push/pull from GitHub.

### 11.1 Generate a key inside proot

SSH into the phone and enter proot:

```bash
ssh phone
proot-distro login ubuntu
```

Generate the key:

```bash
ssh-keygen -t ed25519 -C "s26-claude-server"
```

Press Enter for the default path. Set a passphrase if you want (you'll enter it on each git push/pull).

**Output:**
```
Your identification has been saved in /root/.ssh/id_ed25519
Your public key has been saved in /root/.ssh/id_ed25519.pub
```

### 11.2 Copy the public key

```bash
cat ~/.ssh/id_ed25519.pub
```

**Output (example):**
```
ssh-ed25519 AAAAC3Nz... s26-claude-server
```

Copy this **entire line**.

### 11.3 Add the key to GitHub

1. Go to **GitHub → Settings → SSH and GPG keys → New SSH key**
2. **Title:** `S26 Claude Server`
3. **Key type:** `Authentication key` (not signing key)
4. **Key:** Paste the entire line from the previous step
5. Click **Add SSH key**

> **This won't affect your Mac's Git access.** Each device has its own SSH key. GitHub accepts multiple keys — they're completely independent.

### 11.4 Test the connection

```bash
ssh -T git@github.com
```

First time you'll see GitHub's host fingerprint:

```
The authenticity of host 'github.com (140.82.112.4)' can't be established.
ED25519 key fingerprint is SHA256:+DiY3wvvV6TuJJhbpZisF/zLDA0zPMSvHdkr4UvCOqU.
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

Type `yes`.

If you set a passphrase, enter it when prompted.

**Expected:**
```
Hi <your-username>! You've successfully authenticated, but GitHub does not provide shell access.
```

If you see your GitHub username, Git SSH is working.

### 11.5 Configure Git identity

```bash
git config --global user.name "Your Name"
git config --global user.email "your-email@example.com"
```

Use the same email associated with your GitHub account.

### 11.6 Set up ssh-agent (Required if you set a passphrase)

> **Skip this step** if you did NOT set a passphrase on your Git SSH key in step 11.1.

If your Git SSH key has a passphrase, Claude Code won't be able to `git push` — it can't enter the passphrase interactively. You'll see this error:

```
git@github.com: Permission denied (publickey).
fatal: Could not read from remote repository.
```

**Fix:** Start `ssh-agent` and load your key before launching Claude Code:

```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
# Enter your passphrase when prompted
```

Now the agent holds your key in memory for the rest of the session. Claude Code can push/pull without being asked for the passphrase.

**Do this every time** before starting Claude Code, or add the agent to your `.bashrc` so it starts automatically:

```bash
echo 'eval "$(ssh-agent -s)" > /dev/null 2>&1' >> ~/.bashrc
```

You'll still need to run `ssh-add ~/.ssh/id_ed25519` once per session (and enter the passphrase), but the agent will already be running.

**The workflow becomes:**

```bash
ssh phone
proot-distro login ubuntu
ssh-add ~/.ssh/id_ed25519     # Enter passphrase once
cd ~/projects/my-repo
claude                         # Claude Code can now git push freely
```

---

## Phase 12: Clone Repos & Work (On Phone via SSH)

### 12.1 Create a projects directory

```bash
mkdir -p ~/projects
cd ~/projects
```

### 12.2 Clone a repo

```bash
git clone git@github.com:<your-username>/<repo-name>.git
```

**Example from our setup:**
```bash
git clone git@github.com:<your-github-username>/learning-pydantic-ai.git
```

**Expected output:**
```
Cloning into 'learning-pydantic-ai'...
Enter passphrase for key '/root/.ssh/id_ed25519':
remote: Enumerating objects: 138, done.
remote: Counting objects: 100% (138/138), done.
...
Resolving deltas: 100% (43/43), done.
```

### 12.3 Run Claude Code on your repo

```bash
cd learning-pydantic-ai
claude
```

Claude Code now has full access to your repository. It can read, edit, create files, and run commands — all on the phone.

### 12.4 The sync workflow

After Claude Code makes changes:

```bash
# Stage and commit
git add -A
git commit -m "describe what changed"

# Push to GitHub
git push
```

On your Mac (or any other device), pull the latest:

```bash
cd ~/Code/learning-pydantic-ai
git pull
```

### 12.5 Clone multiple repos

Repeat for any repos you want available:

```bash
cd ~/projects
git clone git@github.com:<your-username>/repo-two.git
git clone git@github.com:<your-username>/repo-three.git
```

All repos live in `~/projects/` on the phone.

---

## Phase 13: Adding More Devices

The whole point of this setup is device independence. Here's how to connect from any new device.

### Windows PC

1. **Install Tailscale** from tailscale.com — sign in with the same account
2. **Open Windows Terminal** (or PowerShell / PuTTY)
3. **Generate an SSH key:**
   ```powershell
   ssh-keygen -t ed25519 -C "windows-to-s26-ultra"
   ```
4. **Copy the key to the phone:**
   ```powershell
   type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh -p 8022 <termux-username>@<phone-tailscale-ip> "cat >> ~/.ssh/authorized_keys"
   ```
   (You'll need to temporarily enable password auth on the phone, or copy the key manually)
5. **Connect:**
   ```powershell
   ssh -p 8022 <termux-username>@<phone-tailscale-ip>
   ```

### Another Mac

1. **Install Tailscale** — same account
2. **Generate SSH key:** `ssh-keygen -t ed25519 -C "mac2-to-s26-ultra"`
3. **Copy key:** `ssh-copy-id -p 8022 <termux-username>@<phone-tailscale-ip>`
4. **Add SSH config** (same as Phase 8)
5. **Connect:** `ssh phone`

### iPad / iPhone

1. **Install Tailscale** from the App Store — same account
2. **Install Termius** (SSH client) from the App Store
3. **Add connection** in Termius: host = phone's Tailscale IP, port = 8022
4. Import or generate an SSH key in Termius

### Per-device checklist

For each new device you add:

- [ ] Tailscale installed and signed in (same account)
- [ ] SSH key generated on the device
- [ ] Public key added to the phone's `~/.ssh/authorized_keys`
- [ ] Test SSH connection works
- [ ] (Optional) Add SSH config shortcut

---

## Why Git and Not SSHFS / Folder Mounting

We initially tried mounting the Mac's folder onto the phone using SSHFS so Claude Code could edit Mac files directly. Here's why that didn't work and why Git is better:

### What we tried

**SSHFS inside proot Ubuntu:**
```bash
apt install -y sshfs
mkdir -p ~/mac-code
sshfs <your-mac-username>@<mac-tailscale-ip>:/Users/<your-mac-username>/Code ~/mac-code -p 22 -o follow_symlinks
```

**Error:**
```
fuse: failed to open /dev/fuse: Permission denied
```

**Cause:** proot doesn't have access to `/dev/fuse`. FUSE (Filesystem in Userspace) requires kernel-level access that proot can't provide — it's a userspace emulation layer, not a real container or VM.

**SSHFS in Termux (outside proot):**
```bash
pkg install sshfs
```

**Error:**
```
Error: Unable to locate package sshfs
```

**Cause:** SSHFS is not packaged for Termux.

### Why Git is better anyway

Even if SSHFS had worked, it would have these problems:

| Issue | SSHFS | Git |
|-------|-------|-----|
| **Mac must be on** | Yes — if Mac sleeps, mount breaks | No — repos are local on phone |
| **Network latency** | Every file read/write goes over the network | All file I/O is local (fast) |
| **Multi-device access** | Only works from devices that can mount the Mac | Any device pushes/pulls independently |
| **Offline work** | Impossible | Full repo available offline |
| **Version history** | None — edits are destructive | Full git history |
| **Backup** | None — single copy on Mac | GitHub is your backup |
| **Conflict resolution** | Last write wins (data loss risk) | Git merge handles conflicts |

**Bottom line:** Git gives you better performance, better reliability, multi-device support, and built-in backup. The only trade-off is running `git push` / `git pull` to sync, which takes seconds.

---

## Errors We Hit & How We Fixed Them

### Error: `unknown distribution 'ubunto'`

```
Error: unknown distribution 'ubunto' was requested to be removed.
```

**Cause:** Typo — `ubunto` instead of `ubuntu`.

**Fix:** Use the correct spelling: `proot-distro remove ubuntu`.

### Error: No `wlan0` in ifconfig, only `rmnet_data7`

```
rmnet_data7: flags=65<UP,RUNNING>  mtu 1436
        inet 192.0.0.8  netmask 255.255.255.224
```

**Cause:** Phone is on mobile data, not Wi-Fi. The `192.0.0.8` IP is a carrier-internal address behind NAT — unreachable from your Mac.

**Fix:** Use Tailscale to create a tunnel. The Tailscale IP (`100.x.x.x`) works over any network.

### Error: `Connection refused` after `pkill sshd && sshd`

```
ssh: connect to host <phone-tailscale-ip> port 8022: Connection refused
```

**Cause:** `pkill sshd` killed the SSH daemon and the current session. The `&& sshd` part that was supposed to restart it didn't execute reliably because the session was already dead.

**Fix:** Open Termux directly on the phone and run `sshd` manually. In the future, avoid running `pkill sshd` while SSH'd in. Instead, do it from the phone directly if you need to restart sshd.

### Error: `sv-enable sshd` — file does not exist

```
fail: sshd: unable to change to service directory: file does not exist
```

**Cause:** `termux-services` was just installed but the service directories haven't been created yet.

**Fix:** Close and reopen the Termux app. The service manager initializes on fresh launch. Then run `sv-enable sshd` again.

### Error: SSHFS `fuse: failed to open /dev/fuse: Permission denied`

```
fuse: failed to open /dev/fuse: Permission denied
```

**Cause:** SSHFS requires FUSE (kernel-level filesystem access). proot is a userspace emulation — it doesn't have access to `/dev/fuse`.

**Fix:** SSHFS cannot work inside proot. Use Git to sync repos instead (see Part 2).

### Error: SSHFS `Unable to locate package sshfs` in Termux

```
Error: Unable to locate package sshfs
```

**Cause:** SSHFS is not packaged for Termux's package repository.

**Fix:** SSHFS is not available in Termux either. Use Git to sync repos instead (see Part 2).

### Error: SSH-ception — running commands on the wrong machine

When SSH'd into the phone, the prompt is `~ $`. When on the Mac, the prompt is `yourname@your-mac ~ %`. These look different but it's easy to confuse them, especially with multiple tabs.

**Examples of things that go wrong:**
- Running `ssh-keygen` on the phone instead of the Mac (generates keys in the wrong place)
- Running `ssh <phone-ip>` from inside an existing SSH session to the phone (SSH into the phone from the phone — asks for password instead of key passphrase)

**Fix:** Always check your prompt before running commands:
- `~ $` = You're on the phone (Termux)
- `root@localhost:~#` = You're on the phone (proot Ubuntu)
- `yourname@your-mac ~ %` = You're on your Mac

When in doubt, run `hostname` — the phone shows `localhost`, your Mac shows its hostname.

---

## Important Gotchas

### Termux vs. proot — where to run what

| Task | Where to run | Why |
|------|-------------|-----|
| `sshd`, SSH config | Termux (`~ $`) | Termux has real network access |
| `sv-enable sshd` | Termux (`~ $`) | Service manager is a Termux feature |
| `termux-wake-lock` | Termux (`~ $`) | Hardware access is in Termux |
| Node.js, Claude Code | proot Ubuntu (`root@localhost:~#`) | Full Linux environment needed |
| tmux | proot Ubuntu | Runs alongside Claude Code |
| Git, project work | proot Ubuntu | Where your projects live |

### Surfshark (or other VPNs) vs. Tailscale

If you have a consumer VPN like Surfshark, it does NOT replace Tailscale. Consumer VPNs route your traffic through their servers to hide your browsing — they don't create a direct tunnel between your devices. Your Mac still can't reach your phone through Surfshark.

On most Android devices, only one VPN can be active at a time. You'll need to turn off Surfshark when using Tailscale (or vice versa).

### tmux sessions don't survive reboots

If the phone reboots (OS update, crash, manual restart):
- sshd will auto-restart (via `sv-enable sshd`)
- You can SSH back in
- But all tmux sessions inside proot are gone
- Any running Claude Code sessions are lost

**Mitigation:** Always push work to Git before leaving sessions unattended for long periods. Disable automatic OS updates.

### The `npm` upgrade notice

After installing Claude Code, npm may show:
```
New major version of npm available! 10.8.2 -> 11.12.1
```

**Do NOT upgrade npm** inside proot unless you have a specific reason. Major version upgrades can break things in the proot environment. The installed version works fine.

---

## Daily Workflow

```bash
# 1. Open Terminal on Mac
# 2. Connect to phone
ssh phone

# 3. Enter Ubuntu
proot-distro login ubuntu

# 4. Start or reattach tmux
tmux new -s claude        # First time
tmux attach -t claude     # Subsequent times

# 5. Launch Claude Code
claude

# 6. Work as normal
# ...

# 7. Push your work
git add -A && git commit -m "WIP" && git push

# 8. Detach tmux (Control+B, then D)

# 9. Exit proot
exit

# 10. Exit SSH
exit

# Phone keeps running. Claude Code keeps running inside tmux.
# Come back anytime with `ssh phone`.
```

---

## Post-Setup Hardening Checklist

These are recommended follow-up tasks after the core setup is working:

- [ ] **Android full-disk encryption** — Settings → Security → Encryption. Should be enabled by default on modern Samsung devices. Verify it.
- [ ] **Tailscale ACLs** — In the Tailscale admin console, restrict access so only your Mac can reach the phone on port 8022.
- [ ] **Tailscale MagicDNS** — Enable MagicDNS in Tailscale settings, then use `ssh -p 8022 <termux-username>@s26-ultra` instead of the IP.
- [x] **Git SSH keys inside proot** — Done in Part 2, Phase 11.
- [ ] **Claude Code credential permissions** — `chmod 700 ~/.claude && chmod 600 ~/.claude/*` inside proot.
- [ ] **Disable auto OS updates** — Settings → Software update → Auto download over Wi-Fi → OFF.
- [ ] **Battery settings** — Termux set to "Unrestricted". Battery optimization OFF.
- [ ] **Shell history protection** — Add `export HISTIGNORE="*API_KEY*:*SECRET*:*TOKEN*:*PASSWORD*"` to `.bashrc` in proot.
- [ ] **Backup config script** — Create a script that backs up Termux configs, SSH keys, and package lists.
- [ ] **Connection resilience** — Install `autossh` or `mosh` on your Mac for auto-reconnecting SSH.

---

## Security Architecture

| Layer | Protection | Against |
|-------|-----------|---------|
| Android full-disk encryption | AES-256 at rest | Physical theft of the phone |
| SSH key-only auth (Ed25519) | No passwords on the wire | Brute-force attacks, credential stuffing |
| SSH key passphrase | Second factor | Someone stealing your Mac / key file |
| MaxAuthTries 3 | Rate limiting | Brute-force on the SSH port |
| Password auth disabled | No password attack surface | All password-based attacks |
| Tailscale (WireGuard) | Encrypted tunnel, NAT traversal | Network eavesdropping, direct port exposure |
| Tailscale ACLs | Device-level access control | Unauthorized devices on your Tailscale network |
| File permissions (600/700) | OS-level access control | Other apps/processes reading secrets |
| HISTIGNORE | Secrets excluded from shell history | Secrets leaking into `.bash_history` |

**What's NOT exposed to the internet:** The SSH port (8022) is only reachable through Tailscale. It's not port-forwarded, not on a public IP, and not discoverable by port scanners. The only way in is through your authenticated Tailscale network, and then through your SSH key.
