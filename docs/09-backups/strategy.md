# Backup Strategy

The homelab uses **two complementary layers** of backups: full container images
(vzdump) and service-level application data (LDIF / SQLite / TSDB snapshots).

| Layer | What | Tool | Where | Schedule | Retention |
| --- | --- | --- | --- | --- | --- |
| Container images | Whole LXC rootfs (`101`–`106`) | `vzdump` (Proxmox) | `/var/lib/vz/dump` | daily `21:00` | `keep-last=30` |
| OpenLDAP | `cn=config` + data tree as LDIF | `slapcat` | `/var/lib/vz/backups/ldap` | daily `22:00` | 14 days |
| Gitea | SQLite database (online backup) | `sqlite3 .backup` | `/var/lib/vz/backups/gitea` | daily `22:00` | 14 days |
| Prometheus | TSDB snapshot | admin API + `tar` | `/var/lib/vz/backups/prometheus` | daily `22:00` | 14 days |

Both jobs live on the Proxmox host (the only machine with access to every
container), and both write to the `local` storage on the host's internal disk.

## Container images (vzdump)

A single Proxmox backup job captures every container with snapshot mode, `zstd`
compression, and `prune-backups keep-last=30`:

```bash
# Show the configured job
cat /etc/pve/jobs.cfg

# List existing backups
ls -lh /var/lib/vz/dump
```

View or edit the job in the web UI under **Datacenter → Backup**. Because the
containers use LVM-thin, snapshot mode is near-instant and non-disruptive.

## Service-level backups

Application data is captured inside the running containers so it can be restored
without restoring an entire container. The script is deployed by the Ansible
[`backup` role](../08-automation/ansible.md) to
`/usr/local/sbin/homelab-backup.sh` and scheduled via
`/etc/cron.d/homelab-backup`:

```bash
# Run on demand
/usr/local/sbin/homelab-backup.sh

# Inspect artifacts
ls -lhR /var/lib/vz/backups
```

What it does:

- **OpenLDAP** — `slapcat -n 0` (config) and `slapcat -n 1` (data) to LDIF.
- **Gitea** — `sqlite3 gitea.db ".backup ..."` (online, consistent) followed by a
  `PRAGMA integrity_check`.
- **Prometheus** — `POST /api/v1/admin/tsdb/snapshot` (needs
  `--web.enable-admin-api`, see [Prometheus](../05-services/prometheus.md)), then
  archives the snapshot.

The script serializes itself with `flock`, skips any container that is not
running, and prunes its own artifacts older than the retention window.

## Ansible

The directories, script, and cron entry are all managed by Ansible, so a rebuild
re-creates them automatically:

```bash
cd ansible
ansible-playbook playbooks/site.yml --tags backup
```

Retention and schedule are variable-driven in
`inventory/group_vars/all/main.yml` (`backup_retention_days`, `backup_hour`).

## References

- [Proxmox Backup and Restore](https://pve.proxmox.com/wiki/Backup_and_Restore)
- [vim & vzdump](https://pve.proxmox.com/pve-docs/chapter-vzdump.html)
- [OpenLDAP slapcat](https://www.openldap.org/software/man.cgi?query=slapcat)
- [SQLite Online Backup API](https://www.sqlite.org/backup.html)
- [Prometheus TSDB snapshots](https://prometheus.io/docs/prometheus/latest/querying/api/#snapshot)
