# Disaster Recovery

This page describes how to recover the homelab at three levels of failure, from
a single corrupted service to a complete host rebuild. All artifacts referenced
here are produced by the process in [Backup Strategy](strategy.md).

For reference, the rough targets are:

| Failure | Recovery source | Expected downtime |
| --- | --- | --- |
| Corrupted service data | service-level backup (LDIF / SQLite / TSDB) | minutes |
| Lost container | vzdump image | ~5 minutes |
| Lost host | vzdump images + Ansible | under an hour |

## Service data restore

### OpenLDAP (`/var/lib/vz/backups/ldap`)

Restore the data tree from an LDIF export (run on the Proxmox host):

```bash
pct push 102 /var/lib/vz/backups/ldap/data-YYYY-MM-DD.ldif /tmp/restore.ldif
pct exec 102 -- systemctl stop slapd
pct exec 102 -- rm -f /var/lib/ldap/data.mdb
pct exec 102 -- slapadd -n 1 -l /tmp/restore.ldif
pct exec 102 -- chown openldap:openldap /var/lib/ldap/data.mdb
pct exec 102 -- systemctl start slapd
pct exec 102 -- slapcat -n 1 | grep -c '^dn:'   # sanity check
```

`slapadd -n 0 -l config-YYYY-MM-DD.ldif` restores the `cn=config` database the
same way (only needed if the server configuration itself was lost).

### Gitea (`/var/lib/vz/backups/gitea`)

The SQLite backup is a complete database copy:

```bash
pct exec 104 -- systemctl stop gitea
pct push 104 /var/lib/vz/backups/gitea/gitea-YYYY-MM-DD.db /var/lib/gitea/data/gitea.db --perms 0644
pct exec 104 -- chown git:git /var/lib/gitea/data/gitea.db
pct exec 104 -- systemctl start gitea
```

### Prometheus (`/var/lib/vz/backups/prometheus`)

The snapshot is a standard TSDB directory:

```bash
pct push 106 /var/lib/vz/backups/prometheus/tsdb-YYYY-MM-DD.tar.gz /tmp/tsdb.tar.gz
pct exec 106 -- systemctl stop prometheus
pct exec 106 -- bash -c 'rm -rf /var/lib/prometheus/*'
pct exec 106 -- tar -xzf /tmp/tsdb.tar.gz -C /var/lib/prometheus
pct exec 106 -- bash -c 'chown -R prometheus:prometheus /var/lib/prometheus'
pct exec 106 -- systemctl start prometheus
```

## Container restore

List the available images and restore one to a new CTID:

```bash
ls -lh /var/lib/vz/dump
pct restore 199 /var/lib/vz/dump/vzdump-lxc-102-YYYY_MM_DD-HH_MM_SS.tar.zst --storage local-lvm
```

Give the restored container a **different IP** before starting it, otherwise it
collides with the original:

```bash
pct set 199 -net0 name=eth0,bridge=vmbr0,ip=10.0.0.199/24,gw=10.0.0.1
pct start 199
```

If you cannot start it on the network yet, bring the interface down instead and
verify offline:

```bash
pct set 199 -net0 name=eth0,bridge=vmbr0,link_down=1
pct start 199
pct exec 199 -- systemctl is-active slapd
```

To **replace** the original container, stop and remove it, then restore to the
original CTID (or restore to a new CTID, re-assign the original IP, and remove
the old container).

## Full host rebuild

1. Reinstall Proxmox VE (see [Installation](../02-proxmox/installation.md)) and
   re-create the `vmbr0` bridge.
2. Restore the `local` storage contents, or copy the images from
   `/var/lib/vz/dump` back to the new host, then `pct restore` each container.
3. Re-apply configuration with Ansible — the playbook re-creates packages,
   `node_exporter`, Nginx, Prometheus, Grafana, and the backup job:

   ```bash
   cd ansible
   ansible-playbook playbooks/site.yml
   ```

The identity-stack configuration (OpenLDAP / Authelia / Gitea) is currently
restored from the container images rather than re-created by Ansible.

## Verified restore test

The recovery path was exercised end to end:

- **Container**: restored the OpenLDAP backup (`vzdump-lxc-102`) to CTID 199
  with the network down, started it, confirmed `slapd` was `active` and the data
  tree exported 6 DNs (including `uid=admin`), then destroyed the container.
- **OpenLDAP**: dry-run import of the LDIF (`slapadd -n 1 -u`) succeeded.
- **Gitea**: `PRAGMA integrity_check` on the backup returned `ok`.
- **Prometheus**: the snapshot archive contained a valid TSDB block layout
  (`<block>/{chunks,index,meta.json,tombstones}`).

## References

- [Proxmox Backup and Restore](https://pve.proxmox.com/wiki/Backup_and_Restore)
- [lxc/pct restore](https://pve.proxmox.com/pve-docs/chapter-pct.html)
