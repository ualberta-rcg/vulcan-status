<p align="center"><img src="assets/ua_logo_green_rgb.png" alt="University of Alberta" width="50%"></p>

# vulcan-status

A Slurm login banner for Vulcan and other HPC sites. Run `vulcan-status` to see cluster capacity, your jobs, fairshare and storage. Run `vulcan-status --help` for a walkthrough of the numbers.

<div align="center">

![Vulcan login banner showing cluster capacity, jobs, fairshare and storage](assets/motd-example.png)

*The banner on Vulcan. Other sites can supply their own logo, links and notices.*

</div>

## Reading the banner

- **Cluster:** a systemd timer refreshes node, queue, CPU, memory and GPU totals every ten minutes. Missing or stale totals can trigger live queries. `avail` counts unallocated GPUs; `usable` estimates how many also have a proportional share of CPU and RAM free. It is not a scheduling guarantee.
- **Your jobs:** always live from `squeue`. Fairshare and storage normally use a private 15-minute cache. The fairshare factor is a priority signal, not percent allocation used. Storage shows what `df` reports: filesystem use, or quota use on filesystems that expose it there.
- **Login:** the hook skips root, non-interactive shells and non-TTY output so it stays silent during file transfers. Use `~/.hushlogin` or `VULCAN_MOTD_OFF=1` to opt out. `vulcan-status --plain --only cluster` still works on demand.

## Install on a login node

Requires Bash, GNU coreutils and Slurm client commands. The bundled cache timer uses systemd. From the repository root, install directly or stage the same files in your site's node image:

```bash
sudo install -m 0755 usr/local/bin/vulcan-status /usr/local/bin/
sudo install -m 0755 usr/local/sbin/vulcan-motd-bake /usr/local/sbin/
sudo install -d -m 0755 /etc/vulcan-motd
sudo cp -n etc/vulcan-motd/{motd.conf,announce.txt} /etc/vulcan-motd/
sudo install -m 0644 etc/systemd/system/vulcan-motd-bake.{service,timer} /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now vulcan-motd-bake.timer
sudo systemctl start vulcan-motd-bake.service
sudo install -m 0644 etc/profile.d/zz-vulcan-motd.sh /etc/profile.d/
```

The optional [Vulcan MOTD cleanup playbook](etc/ansible/playbooks/89-motd.yaml) is **not required**. It deletes regular files directly under `/etc/update-motd.d/` unless their names start with `vulcan_`, including site-written notices. Review its `--check` output before opting in.

## Adapt it to your site

Edit `/etc/vulcan-motd/motd.conf` (root-owned: the cache service sources it as shell code). Set `SITE_TITLE`, `SITE_SUBTITLE`, `LOGO_FILE` and `LINKS_FILE` to replace Vulcan branding. The logo is plain text; links are one `Label:|https://example.org` per line. Put notices in `announce.txt`; an empty file hides the section. See the [sample config](etc/vulcan-motd/motd.conf) for storage paths, thresholds and cache settings.

## Support

Vulcan cluster support: [Digital Research Alliance of Canada](mailto:support@tech.alliancecan.ca). Repository maintainers: [Rahim Khoja](mailto:khoja1@ualberta.ca) and [Karim Ali](mailto:kali2@ualberta.ca), University of Alberta Research Computing.

[MIT License](LICENSE)
