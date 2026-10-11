# How to roll back

[← README](../../README.en.md) · [Runbook](runbook.md) · [Lessons learned](lessons-learned.md)

Each part of the project can be undone on its own. The recommended order is **the GPO first and then the server**: that way workstations are never left pointing at a server that stopped answering in signed form.

> Fictitious data: `dc01.example.local`, `example.local` and the ranges `192.0.2.0/24`, `198.51.100.0/24` and `203.0.113.0/24`.

## 1. GPO "Time Sync - NT5DS"

Choose the rollback level, from lightest to most complete:

| What to do | Effect |
|---|---|
| Remove `Domain Computers` from the security filter | The GPO applies only to the machines left in the filter (for example, the test workstation) |
| Clear **Link Enabled** on the link at the root | The GPO stops applying to everyone, but still exists |
| Delete the link and, if you want, the GPO itself | Permanent removal |

On the workstations, the policy values disappear at the next refresh cycle (90 to 120 minutes) or with `gpupdate /force`, and the **local** configuration takes over, which was already `NT5DS`. No local configuration was changed on the workstations during the project: everything came from the GPO.

To confirm on a workstation:

```
gpupdate /force
reg query "HKLM\SOFTWARE\Policies\Microsoft\W32time\Parameters"   # the policy values should be gone
w32tm /query /configuration                                       # Type: NT5DS (Local)
```

## 2. Server (dc01)

> **Warning:** rolling back the server brings back the original problem. Workstations stop receiving signed time and are again left without a domain time source.

Shell commands, on the server:

```bash
cp -a /etc/chrony.conf.bak-YYYYMMDD /etc/chrony.conf    # restores the backup made before the change
systemctl restart chronyd
chgrp root /usr/local/samba/var/lib/ntp_signd           # optional: restores the directory's original group
```

To confirm:

```bash
grep -v -E '^\s*(#|$)' /etc/chrony.conf    # no allow or ntpsigndsocket
ss -ulpn | grep ':123'                     # chronyd stops listening on UDP 123
ls -ld /usr/local/samba/var/lib/ntp_signd  # drwxr-x--- root root
```

`smb.conf` and Samba were not changed, so there is nothing to undo in them and the Samba service does not need to be restarted.

## 3. Manual synchronization on a workstation

`w32tm /resync` changes no configuration: it only requests a synchronization. There is nothing to roll back in that step.
