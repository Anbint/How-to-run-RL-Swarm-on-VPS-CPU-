# Gensyn RL-Swarm – CPU VPS Setup Guide  
*(with optional Node/Yarn & CodeAssist prep)*

This guide shows how to set up a **Gensyn RL-Swarm node on a fresh Ubuntu VPS (CPU-only)** and keep it running reliably. It also includes basic troubleshooting and upgrade steps.

---

## VPS Requirements

You should run this guide on a VPS with at least:
- **OS:** Ubuntu 22.04 LTS (64-bit)
- **CPU (recommended):** 4 vCPUs for smoother rounds
- **RAM (recommended):** 24 GB (helps avoid OOM when Python/Yarn spawn workers)
- **Disk:** 30 GB SSD (40 GB+ if you also plan to run CodeAssist / other tools)
- **Network:** stable connection, ~10 Mbps up/down, no super-aggressive firewall

This guide assumes:
- You have root or sudo SSH access to the VPS
- Basic terminal / SSH knowledge
- You want to run a **CPU-only RL-Swarm node**
- Optionally, you may later run **CodeAssist / other tools** on the same VPS

---

## TL;DR – Quick Start (for experienced users)

```bash
# 0) SSH into your VPS (Ubuntu 22.04+)
ssh root@YOUR_VPS_IP

# 1) Base deps
apt update && apt upgrade -y
apt install -y git python3 python3-venv python3-pip build-essential screen curl

# 2) Clone repo
cd ~
git clone https://github.com/gensyn-ai/rl-swarm.git
cd rl-swarm

# 3) Python venv + uv
python3 -m venv .venv
source .venv/bin/activate
pip install -U pip uv

# 4) First run (NOT in screen; you’ll need browser login)
bash run_rl_swarm.sh
# - Login via http://localhost:3000 (using SSH port-forward)
# - Answer prompts: HF push = "n", model name = press Enter

# 5) Long-run mode (after first run is OK)
screen -S swarm
source .venv/bin/activate
bash run_rl_swarm.sh
# detach with: Ctrl+A, D
```
# See full details below if you’re new or something breaks.

⸻

## 0. Connect to your VPS

From your local machine:
```bash
ssh root@YOUR_VPS_IP
# or, if you have a non-root user:
ssh youruser@YOUR_VPS_IP
```

⸻

## 1. Install base packages

On the VPS (after SSH):
These are the minimal packages you need for RL-Swarm.
```bash
apt update && apt upgrade -y

apt install -y \
  git \
  python3 \
  python3-venv \
  python3-pip \
  build-essential \
  screen \
  curl
```
<img width="340" height="138" alt="Screenshot 2025-11-30 at 11 27 24" src="https://github.com/user-attachments/assets/23e65d42-da69-49ac-afdb-ce983f2cc9fc" />

Why: 
- git – clone the rl-swarm repo
-	python3, python3-venv, python3-pip – Python + virtualenv
- build-essential – compilers for native Python wheels
- screen – keep the node running after you disconnect
- curl – for downloading scripts / APIs

⸻

### 1.1 Optional: install Node.js & Yarn

Most CPU RL-Swarm nodes do not need Node/Yarn.
However, some tools (BlockAssist, dashboards, etc.) may require them.

If logs later complain about missing node or yarn, run:
```bash
# Node.js 20.x
curl -fsSL https://deb.nodesource.com/setup_20.x | bash - \
  && apt update \
  && apt install -y nodejs

# Yarn repository & key
mkdir -p /etc/apt/keyrings
curl -fsSL https://dl.yarnpkg.com/debian/pubkey.gpg \
  | gpg --dearmor -o /etc/apt/keyrings/yarn.gpg

echo "deb [signed-by=/etc/apt/keyrings/yarn.gpg] https://dl.yarnpkg.com/debian stable main" \
  > /etc/apt/sources.list.d/yarn.list

apt update && apt install -y yarn

# Quick sanity check:
node -v
npm -v
yarn -v
```
<img width="677" height="183" alt="Screenshot 2025-11-30 at 11 29 35" src="https://github.com/user-attachments/assets/b28180c1-70e1-4c2b-ad68-ecc48d63cbd5" />

⸻

## 2. Clone the RL-Swarm repository
```bash
cd ~
git clone https://github.com/gensyn-ai/rl-swarm.git
cd rl-swarm
```
This downloads the latest rl-swarm code into ~/rl-swarm.

⸻

## 3. Create Python virtualenv & install uv
We use a virtual environment so the node has isolated Python dependencies.
```bash
cd ~/rl-swarm

python3 -m venv .venv
source .venv/bin/activate

pip install -U pip uv
```

<img width="649" height="169" alt="Screenshot 2025-11-30 at 11 30 31" src="https://github.com/user-attachments/assets/92188255-b8cc-48c9-8bdf-124fefb4d1a3" />

- .venv – local environment inside rl-swarm
- uv – fast Python package resolver/runner used by the project

Whenever you SSH back into the VPS and want to run the node:
```bash
cd ~/rl-swarm
source .venv/bin/activate
```

⸻

## 4. First test run (wallet + testnet connection)
For the first run, do this outside of screen so you can see prompts and fix errors easily.
This run will:
- Install JS dependencies (if needed)
- Start the local web UI on localhost:3000
- Let you log into the Gensyn Testnet
- Confirm that your node can join the swarm

### 4.1 Start the RL-Swarm script

With the virtualenv active:
```bash
cd ~/rl-swarm
source .venv/bin/activate
bash run_rl_swarm.sh
```

<img width="615" height="218" alt="Screenshot 2025-11-30 at 11 33 37" src="https://github.com/user-attachments/assets/181208d0-f041-4c8e-80e6-98737c0741d6" />

You should see an ASCII banner and logs like:
> Please login to create an Ethereum Server Wallet
> Node.js is already installed: v20.x
> yarn install v1.22.22 …
> Building server
> Started server process: 50753
>> Failed to open http://localhost:3000. Please open it manually.
>> Waiting for modal userData.json to be created...

<img width="505" height="159" alt="Screenshot 2025-11-30 at 11 41 17" src="https://github.com/user-attachments/assets/18980410-ec88-4628-aaaa-23331129fd51" />

This means the wallet UI is running on port 3000 on the VPS and the script is waiting for you to log in.

⸻

### 4.2 Open the wallet UI via SSH port-forwarding
The UI is on the VPS, so we tunnel it to your local machine.
On your local laptop/desktop, open a new terminal:
```bash
ssh -L 3000:localhost:3000 root@YOUR_VPS_IP
```
- Replace YOUR_VPS_IP with your VPS address
- Leave this terminal open (don’t press Ctrl+C) – it forwards traffic from your local localhost:3000 to the VPS

Now open a browser on your local machine and visit:
```
http://localhost:3000
```

You should see the “Welcome to the Gensyn Testnet” page.
Log in / connect your Gensyn account as instructed.

<img width="953" height="801" alt="Screenshot 2025-11-30 at 11 48 19" src="https://github.com/user-attachments/assets/9da9c388-cf05-48f0-b87a-588ddb4ffabf" />


When you see something like “YOU ARE SUCCESSFULLY LOGGED IN TO THE GENSYN TESTNET.”, the UI step is done.
You can close the browser tab; keep the SSH tunnel terminal open until the first run finishes.

⸻

### 4.3 Answer the prompts in the VPS terminal

Go back to the VPS terminal where run_rl_swarm.sh is running.

After login, the script continues and asks:

>> Would you like to push models you train in the RL swarm to the Hugging Face Hub? [y/N]

	•	Type n and press Enter (unless you explicitly want to push models).

Then:

>> Enter the name of the model you want to use in huggingface repo/name format,
   or press [Enter] to use the default model.

	•	Just press Enter to use the default model from the config
(e.g. Qwen/Qwen2.5-Coder-0.5B-Instruct).

<img width="706" height="109" alt="Screenshot 2025-11-30 at 11 58 59" src="https://github.com/user-attachments/assets/e0fbdb01-abf8-46e5-a136-4bf51e363ffa" />

Soon you should see logs like:
```
[INFO] - ✅ Connected to Gensyn Testnet
[INFO] - ===========!!!Joining CodeZero Swarm!!!===========
[INFO] - 🐝 Hello [raging rabid pigeon] !
[INFO] - Using Model: Qwen/Qwen2.5-Coder-0.5B-Instruct
[INFO] - Starting round: 17674/1000000.
Map: 100%|████████| 1/1 [00:00<00:00, ... examples/s]
```

<img width="702" height="186" alt="Screenshot 2025-11-30 at 12 05 12" src="https://github.com/user-attachments/assets/2cf11f91-05c8-4783-a311-98517f9c2926" />

	•	The nickname in brackets (e.g. raging rabid pigeon) is your node’s Swarm ID.
	•	If you see ✅ Connected to Gensyn Testnet and rounds starting, your node is training correctly.

- **At this point the first test run is successful.**
- You can press Ctrl+C in this terminal to stop it and switch to background mode with screen.

⸻

## 5. Run RL-Swarm inside screen (so it survives SSH disconnects)

### 5.1 Start a new screen session
```bash
cd ~/rl-swarm
screen -S swarm
```

Now you are inside a new screen session named swarm.

### 5.2 Activate venv and start the node
Inside the screen session:

```bash
source .venv/bin/activate
bash run_rl_swarm.sh
```

<img width="515" height="251" alt="Screenshot 2025-11-30 at 12 06 08" src="https://github.com/user-attachments/assets/a33bcd25-5795-4549-a9d2-1f88f3f55766" />

Let it run. You’ll see rounds being joined and processed.

Sometimes you may see logs like:

```
Error with swarm payloads, continuing with local state: [Errno 111] Connection refused
```

These are usually transient network / peer issues. As long as rounds keep progressing, you’re fine.

### 5.3 Detach from screen

To leave the node running in the background:
	•	Press Ctrl+A, then D (detach)

You’ll return to a normal shell, but the RL-Swarm node keeps running.

### 5.4 Re-attach later

List active screen sessions:

```
screen -ls
```

Re-attach to the swarm session:

```
screen -r swarm
```

⸻

## 6. Check that your node is really working
*(on-chain txns + rewards)*

This section helps you confirm two things:
1.	Your node is active and submitting transactions on-chain.
2.	Your rewards are being updated roughly every 3 hours.

### 6.1 Get your wallet address (EOA)
1. Open the RL-Swarm dashboard: https://dashboard.gensyn.ai/?application=RLSwarm￼
2. Log in with the same wallet/account you used during setup.
3. In the “RL SWARM : YOUR NODES” table, copy your wallet address
(the green 0x... EOA on the right side).

<img width="1076" height="178" alt="Screenshot 2025-11-30 at 22 10 03" src="https://github.com/user-attachments/assets/5eb810d6-43f2-493a-b9c2-b77aa0c78ead" />

You’ll use this address on the explorer.

### 6.2 Verify on-chain activity (is the node actually running?)
1. Go to the Gensyn testnet explorer: https://gensyn-testnet.explorer.alchemy.com/￼
2. Paste your wallet address into the search bar.
3. Open the “Internal txns” tab.

<img width="1175" height="593" alt="Screenshot 2025-11-30 at 12 25 35" src="https://github.com/user-attachments/assets/1121f264-b27d-4d93-b917-e1f85e230081" />

If your node is configured correctly and participating in rounds, you should see recent internal transactions from your address to contracts such as:
- SwarmCoordinator
- SemiModularAccountBytecode
- SemiModularAccount
- etc.

If you see these internal txns appearing over time → your node is alive and submitting work to the protocol.
If the tab stays empty for many hours while the node is running, you likely need to debug (see section 7).

### 6.3 Rewards update every ~3 hours
Back on the RL-Swarm dashboard (“YOUR NODES” table):
- Participation – increases as your node takes part in rounds.
Usually similar across active nodes (CPU & GPU) over the same time window.
- Training Rewards – updated roughly every 3 hours and depend on your device performance and uptime.

<img width="855" height="53" alt="Screenshot 2025-11-30 at 16 14 29" src="https://github.com/user-attachments/assets/9eb07b27-9a7c-4d43-a12d-22a8e006747d" />

So the expected flow is:
- Short term: use the explorer to ensure your node is really submitting on-chain txns.
- Over 3 hours: watch Participation & Training Rewards slowly increase on the dashboard as the system batches and posts updates.

⸻

## 7. When your node looks stuck (no txs / no rewards)
If after ~3 hours you don’t see new internal txns on the explorer and the dashboard shows no Participation / Rewards, try these steps in order.

⸻

### 7.1 Option 1 – Simple restart (recommended first)

<img width="622" height="85" alt="Screenshot 2025-11-30 at 12 17 29" src="https://github.com/user-attachments/assets/0007ace4-2278-4eab-9701-bec98f8b2509" />

Sometimes the node or P2P daemon just needs a fresh start.

```bash
# 1) SSH into your VPS
ssh root@YOUR_VPS_IP

# 2) Go to rl-swarm and activate venv
cd ~/rl-swarm
source .venv/bin/activate

# 3) Stop any old screen session (ignore errors if it doesn't exist)
screen -S swarm -X quit 2>/dev/null || true

# 4) Start a fresh screen session
screen -S swarm

# 5) Inside screen, start the node
source .venv/bin/activate
bash run_rl_swarm.sh

# Detach with: Ctrl+A, D
```

Wait 10–15 minutes and check logs / explorer again.

⸻

### 7.2 Option 2 – Update to the latest rl-swarm version
If restart doesn’t help, make sure you’re on the latest commit.

```bash
cd ~/rl-swarm
source .venv/bin/activate

# Stop old screen session if still running
screen -S swarm -X quit 2>/dev/null || true

# Update repo from GitHub
git fetch origin
git pull origin main

# (Optional) refresh pip/uv
pip install -U pip uv

# Start again in a new screen
screen -S swarm
source .venv/bin/activate
bash run_rl_swarm.sh
```

Then monitor logs + explorer as in section 6.

⸻

### 7.3 Option 3 – Reset node identity (new `swarm.pem`)
Use this only if **Option 1 + Option 2 did not fix the issue**.

This will give you a new PeerID / node identity.
You may lose continuity with previous participation history tied to the old key.

Before deleting anything, **make a backup of your current `swarm.pem`** so you can restore this identity later if needed.

```bash
cd ~/rl-swarm
source .venv/bin/activate

# Stop old session if still running (ignore errors)
screen -S swarm -X quit 2>/dev/null || true

# 🔐 Optional but strongly recommended: backup current swarm.pem
if [ -f swarm.pem ]; then
  cp swarm.pem swarm.pem.backup-$(date +%Y%m%d-%H%M%S)
fi

# ⚠️ Delete current node identity (a new one will be created on next run)
rm -f swarm.pem

# Start again – a new swarm.pem will be created
screen -S swarm
bash run_rl_swarm.sh
```

After that, repeat the checks:
	•	A new node name should appear in the terminal (Hello [your node name]).
	•	The explorer should start showing internal txs for your wallet after some time.
	•	The dashboard should show Participation / Rewards after the next ~3-hour update window.

If you ever want to restore the old identity, stop the node, then:

```bash
cd ~/rl-swarm
source .venv/bin/activate
screen -S swarm -X quit 2>/dev/null || true

cp swarm.pem.backup-YYYYMMDD-HHMMSS swarm.pem

screen -S swarm
bash run_rl_swarm.sh
```

(Replace swarm.pem.backup-YYYYMMDD-HHMMSS with the actual backup filename.)

Link X: https://x.com/anbint1/status/1995361164335984820 
