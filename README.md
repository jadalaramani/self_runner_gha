# self_runner_gha
self_runner_gha



## 1. Launch EC2 Instance

* AMI: **Amazon Linux 2**
* Instance type: `t2.micro` (for testing) or bigger if jobs are heavy


## 2. Install Dependencies

```bash
sudo yum update -y
sudo yum install -y curl tar git jq
```

Install libraries for .NET runtime (needed by GitHub runner):

```bash
sudo ./bin/installdependencies.sh
```

---

## 3. Create Runner User (recommended)

```bash
sudo adduser github
sudo passwd github   # set a password (optional, for su/ssh login)
sudo usermod -aG wheel github
su - github
```

Now you’re logged in as `github`.

---

## 4. Download and Extract GitHub Runner

Check the latest release here: [https://github.com/actions/runner/releases](https://github.com/actions/runner/releases)

Example for v2.320.0:

```bash
mkdir ~/actions-runner && cd ~/actions-runner
curl -o actions-runner.tar.gz -L https://github.com/actions/runner/releases/download/v2.320.0/actions-runner-linux-x64-2.320.0.tar.gz
tar xzf actions-runner.tar.gz
```

---

## 5. Configure Runner

Go to your repo → **Settings → Actions → Runners → New self-hosted runner → Linux**.
GitHub will give you a command with a **registration token** (expires in 1 hour).

Run it (example):

```bash
./config.sh --url https://github.com/jadalaramani/self_runner_gha --token <YOUR_RUNNER_TOKEN>
```

* Runner name → press Enter or give unique name (`ec2-runner-1`)
* Labels → press Enter (defaults to `self-hosted, linux, x64`)
* Work folder → press Enter (defaults to `_work`)

✅ At this point, your runner is **registered** with the repo.

---

## 6. Start the Runner

To actually process jobs, run:

```bash
./run.sh
```

You should see:

```
√ Connected to GitHub
Current runner version: '2.320.0'
√ Runner successfully added
Listening for Jobs
```

Leave this process running — it’s now waiting for jobs.

---

## 7. Test Workflow

Create `.github/workflows/test.yml` in your repo:

```yaml
name: Test Runner

on: [push]

jobs:
  build:
    runs-on: self-hosted
    steps:
      - uses: actions/checkout@v4
      - run: echo "Hello from my EC2 self-hosted runner!"
```

Push a commit → GitHub Actions should show the job picked up by your runner.

---

## 8. Keep Runner Alive (Important)

Since you’re only using **run.sh**:

* The runner will stop if you close the SSH session.
* To keep it alive, use a terminal multiplexer like `screen` or `tmux`.

Example with `screen`:

```bash
screen -S github-runner
cd ~/actions-runner
./run.sh
```

Press `Ctrl+A D` to detach → runner keeps running in background.
Reattach later with:

```bash
screen -r github-runner
```
👉 Do you want me to also show you how to **auto-start `run.sh` on boot with just GitHub’s commands** (using `crontab` or `rc.local`), so you don’t need to manually attach to `screen` every time?
