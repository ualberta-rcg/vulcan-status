# Working on vulcan-status

- `usr/local/bin/vulcan-status` renders the banner. `usr/local/sbin/vulcan-motd-bake` collects cluster totals for the systemd timer. Keep site branding, links and storage settings in `etc/vulcan-motd/motd.conf`, not in the login hook.
- `etc/profile.d/zz-vulcan-motd.sh` runs on the login path. Keep its timeout and its checks for an interactive terminal: a file transfer must not receive banner text.
- Shared snapshots contain cluster totals only. Fairshare and storage caches are private to the user. If a directory is unsafe, skip caching; do not relax the permission checks or fall back to a shared temporary directory.
- Run `bash -n` separately on each changed Bash script. For behavior tests, use synthetic Slurm and storage responses and a temporary private runtime directory. `--help` renders the banner, so it is not a query-free smoke test. Never capture real job lists, account details or credentials in fixtures, screenshots or logs.
- Keep the README focused on using and installing the banner, and record notable changes in `CHANGELOG.md`. Report fixture results separately from live deployment checks.
- This checkout can be staged into a login node or node image; permission to edit repository files is not permission to install files, start services, or run the cleanup playbook on a live cluster.
