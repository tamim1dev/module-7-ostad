# CloudNest
# CloudNest: Git Workflow, Self-Hosted CI and Server Monitoring

Write-up for the CloudNest assignment. I played the role of Rafi, working for Nadia, and solved each task on my own Fedora machine.

- **Repository:** `https://github.com/<your-username>/<repo-name>`
- **Environment:** Fedora Linux, Git, Go, GitHub Actions self-hosted runner, Prometheus, Grafana Alloy, Grafana
- **Screenshots:** in [`docs/screenshots/`](docs/screenshots/)

## Contents

1. [Task 1: Branching](#task-1-branching)
2. [Task 2: Stash and restore](#task-2-stash-and-restore)
3. [Task 3: Rebase vs merge](#task-3-rebase-vs-merge)
4. [Task 4: Fixing a commit message](#task-4-fixing-a-commit-message)
5. [Task 5: Self-hosted runner CI](#task-5-self-hosted-runner-ci)
6. [Task 6: Metrics stack](#task-6-metrics-stack)
7. [Task 7: Manual dashboard](#task-7-manual-dashboard)
8. [Problems and fixes](#problems-and-fixes)

---

## Task 1: Branching

**Problem:** New feature work must stay separate from the main code.

**What I did:** I created a feature branch from `main` and committed the new work there, so `main` stayed untouched.

```bash
git switch -c feature/login
git add login.txt
git commit -m "Add login page"
git log --oneline --graph --all --decorate
```

**Why:** A branch is a cheap, isolated line of history. Work in progress cannot break `main`, and it only joins `main` when it is reviewed and ready.

**Proof:** ![Branches and graph](docs/screenshots/task1-branches.png)

---

## Task 2: Stash and restore

**Problem:** Halfway through a change, an urgent bug appeared on another branch. The unfinished work was not ready to commit but could not be lost.

**What I did:**

```bash
git stash push -u -m "WIP login validation"   # save tracked + untracked changes
git status                                     # working tree clean
git switch main
git switch -c bugfix/urgent
# ...fix, commit...
git switch main && git merge bugfix/urgent
git switch feature/login
git stash pop                                  # restore the work exactly
git status
```

**Why:** `git stash` shelves uncommitted changes so the working tree is clean enough to switch branches. I used `-u` so that untracked files were saved too, because plain `git stash` leaves them behind. I used `pop` rather than `apply` because I wanted the stash entry removed once the work was restored.

**Proof:**
- Dirty tree before stashing: ![dirty](docs/screenshots/task2-dirty.png)
- Clean tree and stash list: ![stashed](docs/screenshots/task2-stashed.png)
- Bugfix merged into main: ![bugfix](docs/screenshots/task2-bugfix-merged.png)
- Work restored after `stash pop`: ![restored](docs/screenshots/task2-restored.png)

---

## Task 3: Rebase vs merge

**Problem:** The feature branch fell behind `main`. Nadia wanted it updated twice, once with a clean linear history and once with the full history preserved, so she could compare the two.

**What I did:** I made two copies of the same branch from the same starting point and updated each differently.

```bash
git branch feature-rebase
git branch feature-merge

git switch feature-rebase
git rebase main                                  # linear history

git switch feature-merge
git merge main -m "Merge main into feature-merge"   # preserves history

git log --oneline --graph --all --decorate
```

**Comparison:**

| | Rebase | Merge |
|---|---|---|
| History shape | Straight line | Fork and join |
| Merge commit | None | Yes |
| Commit hashes | Rewritten (new hashes) | Unchanged |
| Original context | Lost (looks as if work started from the latest main) | Kept (shows when the branch diverged and rejoined) |
| Risk | Dangerous on branches others have already pulled | Safe on shared branches |
| Readability | Easy to follow with `git log` | Busier graph |

**Recommendation:** Rebase on private, unpublished feature branches to keep history clean. Use merge for shared branches, where rewriting history would break other people's work.

**Proof:**
- Before: ![before](docs/screenshots/task3-before.png)
- After rebase (linear): ![rebase](docs/screenshots/task3-rebase.png)
- After merge (merge commit): ![merge](docs/screenshots/task3-merge.png)
- Side by side: ![both](docs/screenshots/task3-both.png)

---

## Task 4: Fixing a commit message

**Problem:** A commit was made with the message "asdf fix", which is not acceptable in the project history.

**What I did:** I fixed it before pushing.

```bash
git log --oneline                               # shows "asdf fix"
git commit --amend -m "Fix typo in login page"  # if it is the latest commit
# for an older commit: git rebase -i HEAD~N, change "pick" to "reword"
git log --oneline
```

**Why:** A commit message is part of the commit's hash, so changing it creates a new commit object. That is harmless locally, but if the commit had already been pushed it would require a force-push and rewrite history that others may have pulled. That is why the fix had to happen before anyone saw it.

**Proof:**
- Before: ![before](docs/screenshots/task4-before.png)
- After: ![after](docs/screenshots/task4-after.png)

---

## Task 5: Self-hosted runner CI

**Problem:** CloudNest is cancelling its paid CI service. The project must build and test on every push, using a machine the company controls.

**What I did:**

1. Created a small Go project with a build and a test.
2. Created a dedicated `runner` user on the Fedora machine.
3. Registered a self-hosted GitHub Actions runner (Settings → Actions → Runners) with the custom label `cloudnest`.
4. Installed it as a systemd service so it survives reboots.
5. Wrote a workflow that runs on every push and targets that runner.

```yaml
# .github/workflows/ci.yml
name: CI
on: push

jobs:
  build-test:
    runs-on: [self-hosted, linux, cloudnest]
    steps:
      - uses: actions/checkout@v4
      - name: Show which machine runs this
        run: hostname && whoami
      - name: Build
        run: go build ./...
      - name: Test
        run: go test ./... -v
```

**Why:**
- **Self-hosted runner:** The company controls the hardware and environment and avoids per-minute hosted CI costs. The runner makes an outbound connection to GitHub, so no inbound ports are needed.
- **Custom label:** `runs-on` only matches runners that have all the listed labels, so jobs can never fall back to a GitHub-hosted machine.
- **Dedicated user:** Workflow code runs arbitrary commands, so it should not run as root or as my personal account.
- **systemd service:** A runner started by hand stops when the terminal closes.
- **Security note:** Self-hosted runners should only be used on private or trusted repositories, because a pull request from a stranger could execute code on the machine.

**Proof:**
- Runner shown as Idle: ![runner](docs/screenshots/task5-runner-idle.png)
- Service status: ![service](docs/screenshots/task5-service.png)
- Green workflow run: ![green](docs/screenshots/task5-green-run.png)
- Job log showing my machine's hostname: ![hostname](docs/screenshots/task5-hostname.png)

---

## Task 6: Metrics stack

**Problem:** The server sometimes slows down and nobody knows why. System metrics need to be collected, stored, and made viewable, with a modern telemetry agent doing the collection.

**Architecture:**

```
Grafana Alloy  ──remote_write──▶  Prometheus  ◀──queries──  Grafana
(collects host metrics)          (stores them)              (visualizes)
```

| Component | Role |
|---|---|
| Grafana Alloy | Telemetry agent. Collects host metrics with its built-in unix exporter and pushes them to Prometheus. |
| Prometheus | Time-series database. Stores the metrics and answers PromQL queries. |
| Grafana | Visualization. Reads from Prometheus. |

All three were installed with `dnf` (Alloy and Grafana from Grafana's RPM repository) and run as systemd services provided by the packages.

**Install:**

```bash
sudo dnf install -y alloy grafana golang-github-prometheus
```

**Prometheus** was started with the remote-write receiver enabled, through a systemd drop-in override:

```ini
[Service]
ExecStart=
ExecStart=<original command> --web.enable-remote-write-receiver
```

**Alloy** (`/etc/alloy/config.alloy`):

```alloy
prometheus.exporter.unix "host" { }

prometheus.scrape "host" {
  targets         = prometheus.exporter.unix.host.targets
  scrape_interval = "15s"
  forward_to      = [prometheus.remote_write.local.receiver]
}

prometheus.remote_write "local" {
  endpoint {
    url = "http://localhost:9090/api/v1/write"
  }
}
```

```bash
sudo systemctl enable --now prometheus alloy grafana-server
```

In Grafana I added Prometheus as a data source (`http://localhost:9090`).

**Why these choices:**
- **Alloy as the single collector:** Alloy collects the host metrics itself with its built-in unix exporter, so there is no separate Node Exporter to install and maintain. The task asked for a modern telemetry agent, and one agent means fewer moving parts.
- **Push instead of pull:** Alloy pushes to Prometheus with `remote_write`, so Prometheus only has to store and query. Collection logic lives in one place, the agent.
- **Packages over manual binaries:** `dnf` handles upgrades and ships tested systemd unit files.

**Verification:**
- `node_cpu_seconds_total` returns data in the Prometheus UI.
- Alloy's UI at `localhost:12345` shows all components healthy.
- Grafana's data source test passes, and Explore returns data.

**Proof:**
- Prometheus service: ![prometheus](docs/screenshots/task6-prometheus.png)
- Alloy component graph: ![alloy](docs/screenshots/task6-alloy-ui.png)
- Grafana data source test: ![datasource](docs/screenshots/task6-grafana-datasource.png)
- Prometheus query result: ![query](docs/screenshots/task6-prometheus-query.png)
- Grafana Explore: ![explore](docs/screenshots/task6-grafana-explore.png)

---

## Task 7: Manual dashboard

_TODO: complete after building the dashboard (CPU, memory, disk, network panels)._

---

## Problems and fixes

| Problem | Cause | Fix |
|---|---|---|
| `dnf install alloy` failed with `Curl error (77): Problem with the SSL CA cert` | The Grafana repo file hard-coded `sslcacert=/etc/pki/tls/certs/ca-bundle.crt`, which curl could not use | Removed the `sslcacert` line so dnf uses the system default, ran `dnf clean all`, then reinstalled |
| _Add any other issue you hit (e.g. SELinux denial, expired runner token)_ | | |
