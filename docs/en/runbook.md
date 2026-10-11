# Runbook: time synchronization in a Samba AD domain

[← README](../../README.en.md) · [Rollback](rollback.md) · [Lessons learned](lessons-learned.md)

> **Notice: fictitious data.** All names, addresses and values in this document are fictitious (domain `example.local`, ranges reserved for documentation by RFC 5737). Adapt them to your environment before using, and make each change with a backup and a test workstation.

## 1. Scenario

- Domain `example.local`, with one controller `dc01.example.local` (`192.0.2.10`): Samba AD DC compiled from source under `/usr/local/samba`, Rocky Linux 10.x, virtual machine (KVM).
- Windows workstations in computer OUs organized by department.
- Networks that query the time server: `192.0.2.0/24` (main network, with servers and some workstations), `198.51.100.0/24` and `203.0.113.0/24` (workstation networks). The phone, guest and IoT networks have no domain machines and were **deliberately not allowed**.

## 2. Problem

Domain devices with clocks that differ by minutes: some ahead, some behind.

## 3. Diagnosis

Diagnose before changing anything, and write the values down.

**On the server** (read-only):

```bash
date; timedatectl                         # time and time zone
chronyc tracking; chronyc sources -v      # synchronization and sources
grep -v -E '^\s*(#|$)' /etc/chrony.conf   # configuration in use
samba -b | grep -i NTP_SIGND_SOCKET_DIR   # signing socket path
ls -ld /path/to/ntp_signd                 # directory permissions
```

**On a Windows workstation** (PowerShell as administrator, read-only):

```
w32tm /query /source
w32tm /query /status
w32tm /stripchart /computer:dc01.example.local /samples:3 /dataonly
w32tm /query /configuration
nltest /dsgetdc:example.local /TIMESERV
```

What we found:

- The `dc01` server had the correct time (about 0.1 ms off against public NTP servers) and `chronyd` running, **but not serving clients**: `chrony.conf` had neither `allow` nor `ntpsigndsocket`.
- Workstations measured before the fix:

| Workstation | Difference from the server | Time source | Last synchronization |
|---|---|---|---|
| WS-B | about 1.75 s | Local CMOS Clock | never |
| WS-C | about 11.2 s | Local CMOS Clock | never |
| WS-A | about 128 s (2 min 8 s) | Local CMOS Clock | never |

- The clients' configuration was already correct (`Type: NT5DS`, NTP client enabled), and the domain already advertised `dc01` as a time server (`TIMESERV` and `GTIMESERV` flags in `nltest`).
- **Cause:** the server was not answering **signed** time (MS-SNTP) to the Windows clients. With no valid answer, each clock followed its own hardware and drifted.
- The UDP 123 path between the networks was validated with `w32tm /stripchart`.

## 4. Server changes (dc01)

All reversible (see [rollback.md](rollback.md)). Samba needed no new option in `smb.conf`. Do one step at a time and check the result before the next.

**Step 1. Give the `chrony` user access to the signing socket directory** (shell commands):

```bash
chgrp chrony /usr/local/samba/var/lib/ntp_signd
chmod 750 /usr/local/samba/var/lib/ntp_signd
```

Expected result: `drwxr-x--- root chrony`. The socket inside it stays `root:root srwxrwxrwx`.

**Step 2. Back up the chrony configuration** (shell command):

```bash
cp -a /etc/chrony.conf /etc/chrony.conf.bak-YYYYMMDD
```

**Step 3. Append to the end of `/etc/chrony.conf`** the lines from [configs/chrony.conf.example](../../configs/chrony.conf.example). They are **file content**, not commands: do not type them in the terminal. The original lines stay untouched.

```
# Time server for the example.local domain
allow 192.0.2.0/24
allow 198.51.100.0/24
allow 203.0.113.0/24
ntpsigndsocket /usr/local/samba/var/lib/ntp_signd
```

**Step 4. Restart `chronyd`** (shell command):

```bash
systemctl restart chronyd
```

The log now shows `MS-SNTP authentication enabled` and `chronyd` starts listening on UDP 123. Right after the restart, `chronyc tracking` may show `Not synchronized` for a minute or two: wait before testing any client.

**Network and security note:** in other environments, confirm that UDP 123 is open on the server's firewall and that SELinux (if enabled) lets `chronyd` read the socket directory. Keep `allow` restricted to the networks that really have domain machines.

## 5. Server validation

- `chronyc tracking`: `Leap status: Normal`, stratum 2, difference from microseconds to a few milliseconds.
- **Restarting the `samba` service:** the `ntp_signd` directory permission persisted (`root:chrony`), the socket was recreated and ports 53, 88, 389 and 445 were listening again.
- **Restarting the server:** `samba` and `chronyd` came up on their own, the permission persisted and chrony synchronized.
- In both cases, the test workstation synchronized with `Time Source Flags: 2 (Authenticated)` and `Last Sync Error: 0` (on your Windows, the labels appear in the system language).

Restarts affect the domain for a few seconds (Samba service) or minutes (server): logon, internal DNS and GPO are unavailable. Do it at a quiet time.

## 6. Results on the clients

- **WS-C** (test workstation): `w32tm /resync /rediscover` switched the source to `dc01.example.local`. For small differences Windows corrects the clock **gradually** (slew), not with a jump: the 11.2 s difference dropped to about 10 ms in roughly 30 minutes.
- **WS-B** and another workstation (WS-D) corrected themselves **on their own**, with no command, after the server started answering in signed form.
- **WS-A** (the one that was about 128 s off): the user reported that the clock was already correct. **Technical measurement still pending.**

## 7. GPO "Time Sync - NT5DS"

Created to standardize the clients' configuration. It is the **second layer**: the first is the healthy server, and no GPO replaces that.

Computer Configuration > Policies > Administrative Templates > System > Windows Time Service > Time Providers:

- **Enable Windows NTP Client:** Enabled.
- **Configure Windows NTP Client:** Enabled, with `NtpServer = dc01.example.local,0x9` and `Type = NT5DS`. The other fields stay at the policy defaults (CrossSiteSyncFlags 2, ResolvePeerBackoffMinutes 15, ResolvePeerBackoffMaxTimes 7, SpecialPollInterval 1024, EventLogFlags 0).
- **Enable Windows NTP Server:** Not configured (workstations should not serve time).

Where the values appear in the workstations' registry: `HKLM\SOFTWARE\Policies\Microsoft\W32time\Parameters` (`NtpServer` and `Type`) and `HKLM\SOFTWARE\Policies\Microsoft\W32time\TimeProviders\NtpClient` (`Enabled` and the others).

### Scope and phases

To limit risk, the GPO was applied in phases:

1. GPO created **unlinked**, with only the policies configured.
2. Security filter restricted to **one test workstation**, and only then the link at the domain root (not enforced).
3. Validation on the test workstation (`gpupdate /force`, `gpresult`, `w32tm /query /configuration`, `reg query`).
4. WMI filter `Workstations only (Windows Client)`, so servers are not affected, and validation again.
5. Security filter widened to `Domain Computers`, and validation on a second workstation.

Since the link is at the root, inheritance covers all computer OUs. Workstations receive the GPO in the normal refresh cycle (90 to 120 minutes), with no reboot.

## 8. Technical notes

- In `NT5DS` mode, `SpecialPollInterval` is **not used**. The 1024 s interval comes from `MinPollInterval` and `MaxPollInterval`. Those parameters were not changed: the "Global Configuration Settings" policy sets many values at once and is risky.
- The policy's `NtpServer` and `Type` values live under `...\W32time\Parameters`, not `...\TimeProviders\NtpClient`. Querying the wrong path gives the impression that the GPO did not arrive.
- Checking only the GPO name in `gpresult` is not enough: it shows up in both the applied and the filtered lists. Confirm by the registry values after a `gpupdate /force`.
- The Windows time service ships with `StartType` `Manual` (trigger start), which is the default.

## 9. Open items

1. **WS-A:** measure the difference and the time source with `w32tm /query /source` and `w32tm /stripchart /computer:dc01.example.local /samples:3 /dataonly`. Today there is only the user's report.
2. Check one workstation on each workstation network, in different OUs, after the GPO refresh cycle.
3. Check whether any computer OU has GPO inheritance blocked (exclamation icon in GPMC).
