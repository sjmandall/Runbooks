### Update packages and install SSH server

```bash
sudo apt update
sudo apt install openssh-server
```

### Start and check SSH service

```bash
sudo service ssh start
sudo service ssh status
```

### Edit sshd_config to listen on all interfaces

```bash
sudo nano /etc/ssh/sshd_config
```

Ensure these lines are set (uncomment / add as needed):

```text
Port 22
AddressFamily any
ListenAddress 0.0.0.0
ListenAddress ::
PermitRootLogin no
# Optional for demo:
PasswordAuthentication yes
```

### Validate SSH config syntax

```bash
sudo sshd -t
```

No output means the config is valid.

### Restart SSH service

```bash
sudo service ssh restart
```

### Confirm SSH is listening on port 22

```bash
ss -tlnp | grep ssh
# or
sudo netstat -tlnp | grep ssh
```

You should see `0.0.0.0:22` and `:::22`.

### Test SSH locally from Windows to WSL

From Windows PowerShell / CMD:

```powershell
ssh <your_wsl_username>@localhost
```

Enter your WSL user password to confirm SSH works locally.

***

## 2. Create a separate user for your mentor (recommended)

In WSL Ubuntu:

```bash
sudo adduser mentor
```

Follow the prompts to set a password for `mentor`.

(Optional) Create a shared directory and symlink Windows path:

```bash
sudo mkdir -p /home/mentor/shared
sudo ln -s /mnt/c/Users/<WindowsUser>/Documents/<target-path> /home/mentor/shared/files
sudo chown -h mentor:mentor /home/mentor/shared/files
```

Now when `mentor` logs in:

```bash
cd ~/shared/files
ls
```

They see the files you want them to inspect.

***

## 3. Install Tailscale in WSL Ubuntu

### Install Tailscale (official script)

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```


### Start the Tailscale daemon

In one WSL terminal:

```bash
sudo tailscaled
```

Keep this running (it’s the background service). [snippets](https://snippets.page/posts/install-tailscale-wsl2/)

### Bring Tailscale up (authenticate this node)

In another WSL terminal:

```bash
sudo tailscale up
```

- It will print a URL; open it in your browser.
- Log in with your chosen Tailscale account.
- Approve the device if needed. [tailscale](https://tailscale.com/docs/install)

### Get your WSL Tailscale IP

```bash
tailscale ip -4
```

Example output:

```text
100.90.12.34
```

This `100.x.x.x` IP is what your mentor will SSH to. [zenn](https://zenn.dev/imudak/articles/wsl-ssh-tailscale-autostart)

***

## 4. Install Tailscale on mentor’s machine

On your mentor’s laptop:

1. Download and install Tailscale from the official site. [tailscale](https://tailscale.com/docs/install)
2. Log in using the **same tailnet** (account/org) as you.
3. Verify in the Tailscale UI that your WSL node appears with its hostname and `100.x.x.x` IP. [tailscale](https://tailscale.com/learn/ngrok-alternatives)

***

## 5. Mentor SSH command to access your WSL

Provide your mentor with:

- Tailscale IP from `tailscale ip -4` (e.g. `100.90.12.34`).
- The username they should use (`mentor` user you created, or your WSL username).

From the mentor machine:

```bash
ssh mentor@100.90.12.34
# or
ssh <your_wsl_username>@100.90.12.34
```

Once logged in, they can access your Windows files via WSL:

```bash
cd /mnt/c/Users/<WindowsUser>/Documents/<target-path>
ls
```

Or via the shared directory:

```bash
cd ~/shared/files
ls
```

Now they have remote access to your files without being on the same network. [tailscale](https://tailscale.com/docs/install/windows/wsl2)

***

## 6. Commands to run each time before your mentor connects

When you start a new session or after WSL restart:

```bash
# 1) Start SSH
sudo service ssh start

# 2) Start Tailscale daemon (if not already running)
sudo tailscaled &

# 3) Bring Tailscale up (if needed)
sudo tailscale up

# 4) Confirm Tailscale IP
tailscale ip -4
```

Then tell your mentor to connect with `ssh mentor@<tailscale-ip>`.

***


