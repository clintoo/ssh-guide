# SSH — Getting Started (Beginner-friendly, thorough one-page guide)

SSH (Secure Shell) is the standard, secure way to remotely log in to servers, copy files, forward ports, and run commands over an encrypted channel. This page gives you everything you need to *get started safely*: concepts, concrete commands, configuration examples, common workflows, security best practices, and troubleshooting.

---

## Quick overview — what SSH does

* Secure remote shell (terminal): `ssh user@host`
* Secure file copy: `scp`, `sftp`
* Tunneling / port forwarding: `-L`, `-R`, `-D`
* Key-based authentication (recommended) instead of passwords
* Can be run from Linux, macOS, Windows (native OpenSSH or clients like PuTTY)

---

## 1) Install / enable SSH

### On Ubuntu / Debian (server & client)

```bash
# client (usually already installed)
sudo apt update
sudo apt install -y openssh-client

# server (to accept SSH connections)
sudo apt install -y openssh-server
sudo systemctl enable --now ssh
sudo systemctl status ssh   # shows running status
```

### On CentOS / RHEL

```bash
sudo yum install -y openssh-server openssh-clients
sudo systemctl enable --now sshd
sudo systemctl status sshd
```

### On macOS

OpenSSH client is preinstalled. To use SSH server (remote login): System Preferences → Sharing → enable Remote Login or:

```bash
sudo systemsetup -setremotelogin on
```

### On Windows 10/11

* Newer Windows include OpenSSH client. Use PowerShell: `ssh`
* To install OpenSSH server: Settings → Apps → Optional Features → Add OpenSSH Server, or use PowerShell/winget/choco.
* Alternatively use WSL or PuTTY (GUI).

---

## 2) Generate an SSH key pair (strong, recommended)

Use **ed25519** (recommended) or RSA (4096 bits) for compatibility.

```bash
# Recommended: ed25519 (fast, secure)
ssh-keygen -t ed25519 -a 100 -C "your-email@example.com" -f ~/.ssh/id_ed25519

# If you need RSA for old systems:
ssh-keygen -t rsa -b 4096 -C "your-email@example.com" -f ~/.ssh/id_rsa
```

* `-a 100` increases KDF rounds (slows brute-force on passphrase)
* `-C` adds a comment (email)
* You'll be asked for a passphrase — **use one** (or use ssh-agent to avoid typing often)

Files created:

* `~/.ssh/id_ed25519` (private key — keep secret!)
* `~/.ssh/id_ed25519.pub` (public key — safe to share)

Set secure permissions:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519
chmod 644 ~/.ssh/id_ed25519.pub
```

---

## 3) Install your public key on the server (several ways)

### Easiest: `ssh-copy-id` (recommended)

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@server.example.com
# then connect:
ssh user@server.example.com
```

### Manual method (if `ssh-copy-id` unavailable)

```bash
# On your local machine
cat ~/.ssh/id_ed25519.pub

# On the server (logged in via password or console)
mkdir -p ~/.ssh
chmod 700 ~/.ssh
# paste the public key into the file (append)
echo "ssh-ed25519 AAAA... your-comment" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

---

## 4) Basic SSH usage

```bash
# Connect (default port 22)
ssh user@host

# Specify port
ssh -p 2222 user@host

# Use a specific private key file
ssh -i ~/.ssh/id_ed25519 user@host

# Run a remote command and exit
ssh user@host 'ls -la /var/www'

# Copy file to remote
scp localfile.txt user@host:/home/user/

# Copy directory recursively
scp -r localdir user@host:/home/user/
```

---

## 5) Helpful options & quick reference

* `-i /path/to/key` → use identity file
* `-p PORT` → remote port
* `-C` → enable compression
* `-N` → do not execute remote command (useful with port forwards)
* `-f` → send to background (with -N)
* `-L local_port:dest_host:dest_port` → **local** port forward
* `-R remote_port:dest_host:dest_port` → **remote** port forward (reverse tunnel)
* `-D local_port` → dynamic SOCKS proxy (useful for proxying browser via remote host)

Example: create a SOCKS proxy on localhost:1080:

```bash
ssh -D 1080 -C -N user@jumphost.example.com
# Then configure your browser to use SOCKS5 proxy at 127.0.0.1:1080
```

---

## 6) SSH config file (make life easier)

Create/edit `~/.ssh/config` to define host aliases, ports, keys, and reuse settings:

```text
Host myserver
  HostName server.example.com
  User alice
  Port 2222
  IdentityFile ~/.ssh/id_ed25519
  ForwardAgent no
  ServerAliveInterval 60

# Jump/bastion host example (modern)
Host private-host
  HostName internal.example.local
  User bob
  ProxyJump jumpuser@jumphost.example.com

# Connection multiplexing (speed up multiple sessions)
Host *
  ControlMaster auto
  ControlPath ~/.ssh/cm-%r@%h:%p
  ControlPersist 10m
```

Use `ssh myserver` instead of long commands.

---

## 7) Agent & passphrase management

Start the agent and add keys (keeps passphrase in memory):

```bash
# Start agent and add key (Linux/macOS)
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519

# List added keys
ssh-add -l
```

On macOS, to integrate with Keychain: `ssh-add -K ~/.ssh/id_ed25519` (older macOS) or use SSH agent integration in newer versions.

On Windows, use Pageant (PuTTY) or Windows OpenSSH agent.

**Warning:** Forwarding agent (`ForwardAgent yes`) lets remote hosts use your agent — it's convenient but creates risk if the remote host is compromised.

---

## 8) File transfer: `scp`, `sftp`, `rsync`

```bash
# scp copy (local -> remote)
scp file.txt user@host:/path/

# sftp interactive
sftp user@host
sftp> put localfile
sftp> get remotefile

# rsync over SSH (fast, efficient)
rsync -avz -e "ssh -p 2222" ./localdir/ user@host:/remote/dir/
```

---

## 9) Port forwarding and tunnel examples (practical use-cases)

### Local port forward: access a remote DB locally

```bash
ssh -L 5433:127.0.0.1:5432 user@dbserver.example.com -N
# Now connect locally to port 5433 -> forwarded to remote 127.0.0.1:5432
psql -h 127.0.0.1 -p 5433 -U dbuser dbname
```

### Remote port forward (expose local service to remote)

```bash
ssh -R 9000:localhost:3000 user@remoteserver.example.com -N
# Remote server can access your local service at localhost:9000
```

### Dynamic SOCKS proxy

```bash
ssh -D 1080 user@jumphost -N
# Configure browser to use SOCKS5 at 127.0.0.1:1080
```

### Reverse SSH (make a machine behind NAT accessible)

```bash
# On the machine behind NAT:
ssh -R 2222:localhost:22 user@public-host.example.com -N
# Then from public-host:
ssh -p 2222 localhost   # connects into the NATed machine
```

---

## 10) Harden your SSH server (recommended server config)

Edit `/etc/ssh/sshd_config` (then `sudo systemctl restart sshd`):

Suggested settings:

```
PermitRootLogin no                # disallow root login
PasswordAuthentication no         # force key-based auth (only after keys are in place)
ChallengeResponseAuthentication no
UsePAM yes
AllowUsers alice bob              # restrict who can log in
X11Forwarding no                  # disable if not needed
PermitEmptyPasswords no
Port 22 (or change to non-standard port if desired)
PubkeyAuthentication yes
```

Also:

* Use `ufw` or firewall to allow only needed ports (e.g., `sudo ufw allow OpenSSH`).
* Install `fail2ban` to block repeated bad auth attempts.
* Regularly update `openssh` packages.

**Important:** If you disable `PasswordAuthentication`, ensure at least one working key exists for your admin user or you may lock yourself out.

---

## 11) Verify host keys & avoid MITM

The *first time* you connect to a host, SSH stores the server's host key in `~/.ssh/known_hosts`. If this key changes later, SSH will warn you (`WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!`).

**Best practice:** When adding a server, verify the fingerprint with the system admin (server-side command they can run):

```bash
# On server (admin)
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

Then compare that fingerprint with what your client shows on first connect. If unknown and you accept blindly, you risk MITM.

Add host key non-interactively (if you trust network):

```bash
ssh-keyscan -t ed25519 server.example.com >> ~/.ssh/known_hosts
```

---

## 12) Troubleshooting (common errors & fixes)

* `Permission denied (publickey)`

  * Means server didn't accept your key. Check `~/.ssh/authorized_keys` on server, permissions (`chmod 700 ~/.ssh`, `chmod 600 ~/.ssh/authorized_keys`), correct username, or use `ssh -i key`.

* `Host key verification failed.`

  * Known host key changed. Remove old entry: `ssh-keygen -R hostname` then re-add or verify with admin.

* `Connection refused`

  * SSH server not running (`sudo systemctl status sshd`) or firewall blocking port.

* `No route to host` / `Network is unreachable`

  * Network/DNS issue — check connectivity and host/IP.

* `Too many authentication failures`

  * Client tried many keys. Use `ssh -i ~/.ssh/id_ed25519 -o IdentitiesOnly=yes user@host`.

* Use verbose mode for debugging:

```bash
ssh -v user@host        # verbose
ssh -vvv user@host      # very verbose
```

---

## 13) Extra tips & best practices

* Use **ed25519** keys if supported; fallback to RSA 4096 if needed.
* Always protect private keys (`chmod 600`) and keep backups (encrypted).
* Use an agent (`ssh-agent`) to avoid repeatedly typing passphrases.
* Use `~/.ssh/config` to simplify multi-host setups and ProxyJump for bastions.
* Never share your private key. Revoke (remove) public key from `authorized_keys` if compromised.
* Consider hardware-backed keys (YubiKey / FIDO2) for high-security accounts (`ssh-keygen -t ed25519-sk`).
* For Git over SSH, add your public key to Git hosting accounts (GitHub/GitLab/Bitbucket).

---

## 14) Example workflow: deploy key and SSH in 5 minutes (summary)

1. Generate key locally: `ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519`
2. Copy key to server: `ssh-copy-id -i ~/.ssh/id_ed25519.pub user@server`
3. Test login: `ssh user@server`
4. (Optional) Harden server: edit `/etc/ssh/sshd_config` → `PermitRootLogin no`, `PasswordAuthentication no` → `sudo systemctl restart sshd`
5. Use `ssh myalias` via `~/.ssh/config` for convenience.

---

## 15) Quick reference cheat sheet

```text
# Connect
ssh user@host                # default port 22
ssh -p 2222 user@host        # different port
ssh -i ~/.ssh/id_ed25519 user@host

# Copy files
scp file.txt user@host:/path/
scp -r dir/ user@host:/path/

# Tunnels
ssh -L 8080:127.0.0.1:80 user@host     # local forward
ssh -R 9000:localhost:3000 user@host   # remote forward
ssh -D 1080 user@host -N                # dynamic SOCKS

# Debugging
ssh -v user@host
ssh -vvv -i key user@host

# Agent
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
```
