# Lessons learned

[← README](../../README.en.md) · [Runbook](runbook.md) · [Rollback](rollback.md)

The summary of lessons is in [ROADMAP.en.md](../../ROADMAP.en.md). Here you will find the path of the investigation, what did not go as expected and what we would do differently.

## The path of the investigation

1. **Server first.** Time and time zone, which time service is running, sources and offset (`chronyc`), Samba version and paths, `smb.conf` and the socket permissions. Result: the server was right, it just was not serving the clients.
2. **Network.** `w32tm /stripchart` from a workstation to the server: does it answer? what is the offset? This shows whether UDP 123 gets through and quantifies the problem.
3. **Client.** `w32tm /query /source` and `/status` (source and last synchronization), `w32tm /query /configuration` (type and sources) and `nltest /dsgetdc:... /TIMESERV` (does the domain advertise a time server?). Result: correct configuration, but not synchronizing.
4. **Hypothesis and minimal test.** "The clients simply had not tried again after the server started answering signed time." Confirmed with `w32tm /resync /rediscover` on **one** workstation.
5. **Only then the GPO**, in phases.

## What did not go as expected

- **Generic path, specific installation.** The initial recipe used `/var/lib/samba/ntp_signd`. On the Samba compiled from source the path was `/usr/local/samba/var/lib/ntp_signd`. Investigating the server found this before any change.
- **Commands and file content in the same block.** The `allow` and `ntpsigndsocket` lines were pasted into the terminal and gave `command not found`. No side effect, but it showed that every block must say whether it is a command or file content.
- **`SpecialPollInterval` to speed up the correction.** The initial idea was to lower it, but in `NT5DS` mode it is not used. It was dropped before being applied.
- **Expecting a time jump.** After the `resync`, the difference stayed almost the same. It was Windows' gradual correction: only measuring again after about 30 minutes showed the result.
- **Querying the wrong registry path.** Two GPO values (`NtpServer` and `Type`) live under `...\W32time\Parameters`, not `...\TimeProviders\NtpClient`.
- **Ambiguous `gpresult`.** The GPO name also appears in the filtered list. We started confirming by the registry values.

## Risk decisions that paid off

- Restricting the scope (security filter on one workstation) **before** linking the GPO.
- Restarting the Samba service and the server to see whether the socket directory permission persists.
- Backing up before editing and having a rollback command for every change.
- Measuring before and after with the same commands.

## What we would do differently

- Record the measurements (value and time) in a table from the start.
- Also measure the workstation someone said "is already right", before closing the matter.
- Prepare a check script early (it is on the [roadmap](../../ROADMAP.en.md)).
