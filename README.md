<img src="./assets/ua_logo_green_rgb.png" alt="University of Alberta Logo" width="50%" />

# `vulcan-status` — Slurm HPC Login Status Banner

Fast, login-safe, colorblind-accessible MOTD and cluster status renderer for Slurm HPC login nodes. Built for the **Vulcan** AI/ML cluster at the University of Alberta (AMII) and Digital Research Alliance of Canada, and configurable via `/etc/vulcan-motd/motd.conf` for any Slurm site.

![vulcan-status login banner](assets/motd-example.png)

---

## ✨ Highlights

- **Zero Slurm load on the login path**: A root systemd timer (`vulcan-motd-bake`) runs the expensive cluster-wide Slurm queries once every 10 minutes and atomically writes flat `key=value` files to `/run/vulcan-motd/`. Login sessions read tmpfs in milliseconds.
- **Host-feasible `usable` GPUs**: Distinguishes between raw idle GPUs (`avail`) and GPUs whose host node actually has enough unallocated CPU and RAM to schedule a GPU job right now (`usable`).
- **Queue-native Fairshare & `LevelFS` verdicts**: Translates Slurm's 0–1 Fairshare priority factor into a 5-tier scale (`boosted`, `ahead`, `nominal`, `behind`, `throttled`) alongside the account's `LevelFS` share-to-usage ratio (`good`, `nominal`, `bad`).
- **Hardened for shared login nodes**:
  - **SCP/SFTP/Rsync safe**: `/etc/profile.d/zz-vulcan-motd.sh` enforces strict interactive shell, TTY, and `TERM` guards so non-interactive transfers never see stray bytes.
  - **Terminal escape sanitization**: Strips ANSI/OSC control characters (`\000-\037`, `\177`) from all user-controlled Slurm fields (job names, reasons, TRES, account and project names) before truncation or display.
  - **Stale beats empty & never blocks login**: Every external command has a 3-second timeout (`Q_TIMEOUT`), the entire login hook is wrapped in `timeout 6`, and if `slurmctld` or `slurmdbd` is down (`heartbeat`), blocks degrade gracefully instead of hanging.
  - **Colorblind-safe palette**: Status bars and tier labels use a blue–yellow/lightness ramp (bold turquoise, teal, sky blue, orange, bold light red) so distinctions never rely on red-vs-green alone.
- **Built-in interactive explainer**: Running `vulcan-status --help` renders the live banner with inline plain-language explanations beneath every section.

---

## 📦 Repository Layout

| File | Target Path | Purpose |
|---|---|---|
| `usr/local/bin/vulcan-status` | `/usr/local/bin/vulcan-status` | Per-user status renderer (header, links, cluster, jobs, fairshare, storage, notice, and `--help` explainer). |
| `usr/local/sbin/vulcan-motd-bake` | `/usr/local/sbin/vulcan-motd-bake` | Root cache baker; runs every 10 min to write `/run/vulcan-motd/cluster` (including `gpu_usable`) and `/run/vulcan-motd/heartbeat` atomically. |
| `etc/vulcan-motd/motd.conf` | `/etc/vulcan-motd/motd.conf` | Site configuration (cluster name, branding, custom logo/links files, storage paths, fairshare/disk thresholds, cache TTLs). |
| `etc/vulcan-motd/announce.txt` | `/etc/vulcan-motd/announce.txt` | Optional free-form operator notice file (`» Notice` block hides automatically when empty). |
| `etc/profile.d/zz-vulcan-motd.sh` | `/etc/profile.d/zz-vulcan-motd.sh` | Login hook with interactive/TTY guards, root/`.hushlogin`/`VULCAN_MOTD_OFF` suppression, and once-per-session guard. |
| `etc/systemd/system/vulcan-motd-bake.service` | `/etc/systemd/system/vulcan-motd-bake.service` | Systemd oneshot unit (`Nice=19`, `IOSchedulingClass=idle`, `TimeoutStartSec=60`). |
| `etc/systemd/system/vulcan-motd-bake.timer` | `/etc/systemd/system/vulcan-motd-bake.timer` | Systemd 10-minute timer (`OnBootSec=1min`, `OnCalendar=*:0/10`, `RandomizedDelaySec=60`). |
| `etc/ansible/playbooks/89-motd.yaml` | `/etc/ansible/playbooks/89-motd.yaml` | Optional Ansible playbook to clear stock Ubuntu scripts from `/etc/update-motd.d/` so only `vulcan-status` renders. |

---

## 🖥️ Banner Blocks Explained

| Block | Data Source | What It Shows |
|---|---|---|
| **Header** | Local + `motd.conf` | ASCII logo, site title/subtitle, node name, username, and current timestamp. Automatically switches between `wide` ($\ge 70$ cols) and `compact` layouts. |
| **Links** | `motd.conf` / built-in | Quick links to documentation, Open OnDemand, metrics portal, and hosted services. |
| **Cluster** | `/run/vulcan-motd/cluster` (10-min root cache, live fallback) | Nodes up/down, running/pending queue totals, cluster-wide CPU & RAM bars, and GPUs (`allocated/total · avail · usable`). |
| **Your jobs** | Live `squeue -u $USER` | Up to `MAX_JOBS` (default 3) running/pending jobs with elapsed time, time left, GRES, or pending hold reason (`Priority`, `Resources`, `Dependency`). |
| **Fairshare** | Per-user cache (`15 min`) $\rightarrow$ `sshare` | Per-account priority factor (`0.00`–`1.00`), 5-tier standing (`boosted` $\ge 0.80$, `ahead` $\ge 0.60$, `nominal` $\ge 0.40$, `behind` $\ge 0.20$, `throttled` $< 0.20$), and `LevelFS` verdict (`good`, `nominal`, `bad`). |
| **Storage** | Per-user cache (`15 min`) $\rightarrow$ `df -h -P` | Quota utilization bars for `home`, `scratch`, and up to `MAX_PROJECTS` (default 3) `~/projects/*` symlinks. |
| **Notice** | `/etc/vulcan-motd/announce.txt` | Operator announcements. Hidden when `announce.txt` is empty. |

---

## 🔧 Adapting for Any Site (`/etc/vulcan-motd/motd.conf`)

Out of the box, `vulcan-status` defaults to Vulcan's branding and service links. Any Slurm site can customize branding, links, filesystem paths, and thresholds in `/etc/vulcan-motd/motd.conf` without modifying the script:

```bash
# /etc/vulcan-motd/motd.conf — example custom site override
CLUSTER_NAME="mycluster"
SITE_TITLE="Example University // Research Computing"
SITE_SUBTITLE="High Performance Computing"

# Optional custom ASCII logo & links ("Label:|url" per line)
LOGO_FILE=/etc/vulcan-motd/logo.txt
LINKS_FILE=/etc/vulcan-motd/links.txt

# Storage mount bases
HOME_BASE=/home
SCRATCH_BASE=/scratch

# Fairshare & storage color thresholds
FS_BOOST=0.80
FS_AHEAD=0.60
FS_NOMINAL=0.40
FS_BEHIND=0.20
DISK_MID=33
DISK_HIGH=66
```

Example `/etc/vulcan-motd/links.txt`:

```text
Cluster Documentation:|docs.example.edu
Open OnDemand Portal:|ood.example.edu
Grafana Metrics:|metrics.example.edu
```

---

## 🛠️ CLI Usage

Users can run `vulcan-status` at any time during a session:

```bash
vulcan-status                   # Render the full status banner
vulcan-status --help            # Render the banner with plain-language explanations under each block
vulcan-status --only cluster    # Render a single block (header|links|cluster|jobs|fairshare|storage|notice)
vulcan-status --no-cache        # Bypass cached files and query live
vulcan-status --plain           # Disable ANSI colors
vulcan-status --width 60        # Force compact layout width
sudo vulcan-status --user alice # Inspect another user's banner as root
```

To suppress the automatic banner at login, users can either `touch ~/.hushlogin` or set `export VULCAN_MOTD_OFF=1`.

---

## 🚀 Installation & Deployment

```bash
# 1. Install binaries
sudo install -m 0755 usr/local/bin/vulcan-status /usr/local/bin/vulcan-status
sudo install -m 0755 usr/local/sbin/vulcan-motd-bake /usr/local/sbin/vulcan-motd-bake

# 2. Install site config and notice placeholder
sudo mkdir -p /etc/vulcan-motd
sudo cp -n etc/vulcan-motd/motd.conf /etc/vulcan-motd/motd.conf
sudo cp -n etc/vulcan-motd/announce.txt /etc/vulcan-motd/announce.txt

# 3. Install login hook
sudo install -m 0644 etc/profile.d/zz-vulcan-motd.sh /etc/profile.d/zz-vulcan-motd.sh

# 4. Install and enable the 10-minute cache baker timer
sudo install -m 0644 etc/systemd/system/vulcan-motd-bake.service /etc/systemd/system/
sudo install -m 0644 etc/systemd/system/vulcan-motd-bake.timer /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now vulcan-motd-bake.timer
sudo systemctl start vulcan-motd-bake.service
```

---

## 🤝 Support & Maintainers

Many Bothans died to bring us this information. This project is provided as-is, but reasonable questions may be answered based on coffee intake or mood. ;)

- **Alliance / Vulcan cluster support:** **[support@tech.alliancecan.ca](mailto:support@tech.alliancecan.ca)**
- **Repository maintainers (U of A deployments):** Open an issue or email **[khoja1@ualberta.ca](mailto:khoja1@ualberta.ca)** (Rahim Khoja) or **[kali2@ualberta.ca](mailto:kali2@ualberta.ca)** (Karim Ali).

## 📜 License

This project is released under the **MIT License** - one of the most permissive open-source licenses available.

**What this means:**
- ✅ Use it for anything (personal, commercial, whatever)
- ✅ Modify it however you want
- ✅ Distribute it freely
- ✅ Include it in proprietary software

**The only requirement:** Keep the copyright notice somewhere in your project.

That's it! No other strings attached. The MIT License is trusted by major projects worldwide and removes virtually all legal barriers to using this code.

**Full license text:** [MIT License](./LICENSE)

## 🧠 About University of Alberta Research Computing

The [Research Computing Group](https://www.ualberta.ca/en/information-services-and-technology/research-computing/index.html) supports high-performance computing, data-intensive research, and advanced infrastructure for researchers at the University of Alberta and across Canada.

We help design and operate compute environments that power innovation — from AI training clusters to national research infrastructure.
