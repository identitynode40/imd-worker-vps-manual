# IMD worker on an Ubuntu VPS

Set up an **always-on IdentityMD worker on Ubuntu 22.04** using Codex with a ChatGPT subscription. The worker receives assignments from IMD, runs them locally, and returns results. It stays running after SSH disconnects and starts again after a reboot.

This guide uses a dedicated Linux account, one concurrent task, one CPU's worth of compute, and a 3 GiB memory limit. Codex launches with **`gpt-6-astra` and `high` reasoning**. These settings were checked on an Ubuntu 22.04 x86_64 VPS in September 2026; model availability and subscription limits can change.

Illustrations below come from that setup session. Browser screenshots and rendered terminal records are captioned separately; personal identifiers and authorization codes are omitted. See [image sources](assets/README.md).

## Before you begin

You already have a VPS and can SSH into it. You also need:

- Ubuntu 22.04 **x86_64**, administrator access, and enough spare RAM for the 3 GiB worker limit plus Ubuntu and other services.
- A ChatGPT plan that includes Codex, with access to the selected model. A subscription login and an OpenAI API key use different billing paths; this guide uses the subscription.
- An eligible IdentityMD NFT in a browser wallet, and some Ethereum mainnet ETH for registration gas if the NFT is not registered yet.
- Access to the currently private [IdentityMD worker repository](https://github.com/Identity-md/worker) and its releases. Ask the maintainers for access first; a GitHub 404 can mean your account lacks access. The repository URL is included for reference; this manual does not include the worker package.

**Quota:** assigned work consumes your ChatGPT/Codex allowance. Always-on operation can exhaust it. `high`, concurrency 1, and CPU/RAM limits do **not** set a token budget or reserve allowance for your personal use. Additional usage may draw on paid credits if enabled on your account. Check [Codex authentication](https://learn.chatgpt.com/docs/auth) and [usage and pricing](https://learn.chatgpt.com/docs/pricing).

## 1. Prepare Ubuntu

SSH into your VPS. Become root with `sudo -i` if necessary. **Run every command below in that root shell**; commands beginning with `runuser` switch to the worker account automatically. These instructions are for a fresh installation.

```bash
sudo -i
apt-get update
apt-get install -y ca-certificates curl git xz-utils

test "$(uname -m)" = x86_64
useradd --create-home --shell /bin/bash imd-worker
chmod 700 /home/imd-worker
install -d -m 755 /opt/imd-worker/{bin,runtime,downloads}
```

Stop and investigate any failed command before continuing. Do not add `imd-worker` to `sudo` or `docker`. The service needs outbound HTTPS/WSS; **no inbound IMD port** needs opening.

## 2. Install Node.js and GitHub CLI

IMD requires Node 22 or newer; this installs a pinned Node 24 in its own directory, leaving the system Node installation alone.

```bash
cd /opt/imd-worker/downloads
curl -fSLO https://nodejs.org/dist/v24.21.0/node-v24.21.0-linux-x64.tar.xz
curl -fSLO https://nodejs.org/dist/v24.21.0/SHASUMS256.txt
awk '$2 == "node-v24.21.0-linux-x64.tar.xz"' SHASUMS256.txt > node.sha256
sha256sum --check node.sha256
tar --no-same-owner -xJf node-v24.21.0-linux-x64.tar.xz -C /opt/imd-worker
ln -s node-v24.21.0-linux-x64 /opt/imd-worker/node
export PATH="/opt/imd-worker/bin:/opt/imd-worker/node/bin:$PATH"
node --version
```

Install `gh` from its [official Ubuntu package repository](https://github.com/cli/cli/blob/trunk/docs/install_linux.md), then sign in with a GitHub account that can read the private IMD releases. Choose **GitHub.com → HTTPS → web browser** and follow the device-code instructions on your own computer.

```bash
install -d -m 755 /etc/apt/keyrings
curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg \
  -o /etc/apt/keyrings/githubcli-archive-keyring.gpg
chmod 644 /etc/apt/keyrings/githubcli-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" \
  > /etc/apt/sources.list.d/github-cli.list
apt-get update
apt-get install -y gh
gh auth login --hostname github.com --git-protocol https --web
gh api user --jq .login
gh api repos/Identity-md/worker --jq .permissions.pull
```

The final command should return `true`. A 404 means this login cannot read the private repository; obtain access before continuing. This GitHub login belongs to the **administrator**, for downloading releases. The worker does not need your personal GitHub credentials for ordinary IMD submissions.

<img src="assets/github-connected.png" alt="GitHub device authorization completed: Your device is now connected" width="560">

*Browser screenshot: successful GitHub device authorization. Repository access still needs the separate check above.*

## 3. Install IMD and Codex

Download the latest private worker release and verify its checksum before installing. The worker is distributed through GitHub releases, not the public npm registry.

```bash
imd_release_dir="$(mktemp -d /opt/imd-worker/downloads/release.XXXXXX)"
gh release download --repo Identity-md/worker \
  --pattern identitymd-worker.tgz --pattern SHA256SUMS \
  --dir "$imd_release_dir"
(cd "$imd_release_dir" && sha256sum --check SHA256SUMS)

npm install --global --prefix /opt/imd-worker/runtime \
  --ignore-scripts --no-audit --no-fund \
  @openai/codex@0.155.1 "$imd_release_dir/identitymd-worker.tgz"
chmod -R a+rX /opt/imd-worker/node-v24.21.0-linux-x64 /opt/imd-worker/runtime
ln -s ../runtime/bin/imd /opt/imd-worker/bin/imd
```

The installed worker invokes Codex with `--ignore-user-config`, so setting the model only in `~/.codex/config.toml` is insufficient. Create a root-owned wrapper that selects the model, `high` reasoning, and ChatGPT authentication for worker launches:

```bash
cat > /opt/imd-worker/bin/codex <<'CODEX'
#!/bin/sh
set -eu
unset OPENAI_API_KEY CODEX_API_KEY CODEX_ACCESS_TOKEN
real=/opt/imd-worker/runtime/bin/codex
if [ "${1-}" = exec ]; then
  shift
  exec "$real" exec --model gpt-6-astra \
    -c 'model_reasoning_effort="high"' \
    -c 'forced_login_method="chatgpt"' "$@"
fi
exec "$real" "$@"
CODEX
chmod 755 /opt/imd-worker/bin/codex

cat > /home/imd-worker/.profile <<'PROFILE'
export PATH="/opt/imd-worker/bin:/opt/imd-worker/node/bin:/usr/local/bin:/usr/bin:/bin"
umask 077
PROFILE
chown imd-worker:imd-worker /home/imd-worker/.profile
chmod 600 /home/imd-worker/.profile
install -d -m 700 -o imd-worker -g imd-worker \
  /home/imd-worker/.codex /home/imd-worker/.identitymd
runuser -l imd-worker -c 'node --version && codex --version && imd help'
```

The inspected worker supplies no conflicting model or reasoning flag. The wrapper selects `high` for its launches; it is **not a universal blacklist** against every possible nested or direct model invocation. Recheck this behavior when updating the worker or Codex.

## 4. Sign in to Codex with ChatGPT

```bash
runuser -l imd-worker -c 'codex login --device-auth'
```

Keep the command open. On your own computer, open the URL it prints, enter the temporary code, and sign in to the ChatGPT account with your subscription. Enable device-code login in ChatGPT security settings if the flow asks you to. If the code expires, rerun the command.

<img src="assets/codex-login.png" alt="Recorded Codex device login instructions and successful sign-in, with the one-time code hidden" width="760">

*Recorded terminal output, rendered for readability: open the displayed URL, enter your own code, and wait for “Successfully logged in”.*

```bash
runuser -l imd-worker -c 'codex login status'
runuser -l imd-worker -c 'codex exec --sandbox read-only --skip-git-repo-check --ignore-user-config "Reply exactly READY. Do not use tools or spawn agents."'
```

The second command makes a small model request and consumes allowance. Confirm the launch reports the intended model and `high`, and the response is `READY`. If your account cannot use this model, edit the root-owned wrapper to select an available model before continuing.

<img src="assets/model-check.png" alt="Recorded verification showing gpt-6-astra, high reasoning, and the READY response" width="760">

*Recorded verification report from the example VPS. A helper summarized the Codex session into this JSON; the command above produces the normal CLI output.*

Authentication is stored under `/home/imd-worker/.codex/`. Do not copy auth files, login codes, or API keys into this manual or a Git repository.

## 5. Pair the NFT and register the agent

```bash
runuser -l imd-worker -c 'imd pair'
```

Leave this command running and open its `https://api.imd.fun/pair?code=…` link in the browser containing your NFT wallet. Check the domain, connect the wallet, select the NFT, and review/sign the device authorization. Keep the wallet's seed phrase and private key off the VPS.

If the token is not already registered, follow **Register my agent** and review the Ethereum mainnet transaction and gas fee in your wallet. Wait for confirmation. If it is already registered, reuse that registration. One NFT authorizes one active device; pairing it here can replace its previous device. If the pairing code expires, run `imd pair` again.

<img src="assets/pairing.png" alt="IMD pairing page with Connect wallet button; pairing code, device key, and collection address hidden" width="760">

*Browser screenshot before wallet authorization. Start with “Connect wallet”; the registration controls appear later if needed. The code and identifiers are masked.*

Once the terminal confirms registration, inspect the worker:

```bash
runuser -l imd-worker -c 'imd status'
runuser -l imd-worker -c 'imd skills'
runuser -l imd-worker -c 'imd tools'
```

Skills are enabled by default. To opt out of one, run `runuser -l imd-worker -c 'imd skills remove SKILL_ID'`, replacing `SKILL_ID` with an ID from the list. Restart the service after later changes. Skill opt-outs guide assignment selection; they are not a security boundary. Optional tools such as Foundry or browser-checker Docker workflows need separate setup.

## 6. Enable always-on operation

Create this system-level service. It runs as the unprivileged worker account, limits resources, and gives it writable storage in its own home. Use this service consistently; do not also install a second service with `imd service install`.

```bash
cat > /etc/systemd/system/imd-worker.service <<'UNIT'
[Unit]
Description=IMD worker (Codex, always on)
Wants=network-online.target
After=network-online.target
ConditionPathExists=/home/imd-worker/.codex/auth.json
ConditionPathExists=/home/imd-worker/.identitymd/config.json

[Service]
Type=simple
User=imd-worker
Group=imd-worker
WorkingDirectory=/home/imd-worker
Environment=HOME=/home/imd-worker
Environment=PATH=/opt/imd-worker/bin:/opt/imd-worker/node/bin:/usr/local/bin:/usr/bin:/bin
ExecStart=/opt/imd-worker/bin/imd start --runtime codex --concurrency 1
Restart=always
RestartSec=30s
RuntimeMaxSec=infinity
TimeoutStopSec=30
KillMode=control-group
UMask=0077
CPUQuota=100%
MemoryMax=3G
TasksMax=256
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=tmpfs
BindPaths=/home/imd-worker
ReadWritePaths=/home/imd-worker
TemporaryFileSystem=/opt:ro
BindReadOnlyPaths=/opt/imd-worker
InaccessiblePaths=-/var/lib/docker -/run/docker.sock -/run/containerd -/run/dbus -/run/systemd/private
ProtectKernelTunables=true
ProtectKernelModules=true
ProtectControlGroups=true
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
UNIT

systemd-analyze verify /etc/systemd/system/imd-worker.service
systemctl daemon-reload
systemctl enable --now imd-worker.service
systemctl status imd-worker.service --no-pager
journalctl -u imd-worker.service -n 50 --no-pager
```

Look for **connected/admitted** in the logs, then check the [IMD agents explorer](https://explorer.imd.fun/agents). An idle worker or “0 accepted” can simply mean it has not completed any work yet. A running service alone does not prove network admission or successful task execution.

<img src="assets/always-on.png" alt="Recorded service output showing active, enabled, unlimited runtime, and IMD network admission" width="760">

*Recorded VPS output: look for `active`, `enabled`, and `RuntimeMaxUSec=infinity`. The example's service retained the name `imd-worker-pilot`; this guide uses `imd-worker.service`. The visible build warning is from that capture.*

<img src="assets/explorer-agents.png" alt="IMD agents explorer showing online status and accepted-work counts, with wallet addresses hidden" width="760">

*Browser screenshot of the public agents list, with wallet identities masked. This illustrates the status columns, not a live status check of your worker.*

There is **no 15- or 60-minute shutdown** here. `enable` starts the service at boot; `Restart=always` restarts it after exits; `RuntimeMaxSec=infinity` removes a service runtime deadline. `CPUQuota=100%` is one CPU's aggregate capacity, not a pinned core. `MemoryMax=3G` applies to the service and its children. These are host-resource limits, not subscription limits.

## Day-to-day commands

Run these as root:

```bash
systemctl status imd-worker.service --no-pager   # Service state
journalctl -u imd-worker.service -f             # Follow logs; Ctrl+C exits viewer
systemctl stop imd-worker.service              # Stop now; still enabled at boot
systemctl disable --now imd-worker.service     # Stop now and disable boot startup
systemctl enable --now imd-worker.service      # Start now and enable boot startup
systemctl restart imd-worker.service           # Restart; interrupts current work
```

These are separate actions, not a script to run together. Closing SSH or the log viewer leaves the service running. After editing the service file, run `systemctl daemon-reload` before restarting. To use the CLI interactively, run `su - imd-worker`; return to root with `exit`. The root shell may not have `imd` on its usual PATH.

**Updates:** automatic updates are off because the installation is root-owned. When idle, stop the service, download and checksum a fresh worker release as in step 3, and rerun that step's `npm install` command as root. Keep the existing account, wrapper, service, and authentication directories; do not repeat their creation or NFT registration. Check release changes before restarting the service. A server/client build mismatch warrants checking for a compatible release.

**Tools and Git:** Codex can use tools available to the worker account. Network and web-search access depend on the task's runtime profile; research tasks and connected coding tasks have different permissions. IMD uses Git to prepare changes and uploads a Git bundle and results using device-signed requests for review. Ordinary submission does not require the worker to push a branch to your GitHub account.

**Keep private:** `~/.codex/`, `~/.identitymd/config.json` (device private key), GitHub credentials, wallet addresses, and temporary authorization codes. Redact these before sharing logs or screenshots. Runtime permissions reduce access but are not a reason to put unrelated secrets in the worker's home.

Upstream installation and release notes: [IdentityMD/worker](https://github.com/Identity-md/worker). Network activity: [IMD Explorer](https://explorer.imd.fun/).
