# Roadmap

[Português](ROADMAP.md) · **English** · [Español (Honduras)](ROADMAP.es-HN.md)

Planned improvements for the project and what we learned so far. They are ideas with no guaranteed dates, and the order is not final.

## Short term

Small improvements that fit what already exists.

- [ ] **Close the validation gaps.** Measure WS-A with `w32tm /query /source` and `w32tm /stripchart` (today there is only the user's report), check one workstation on each workstation network in different OUs after the GPO refresh cycle, and verify whether any computer OU has GPO inheritance blocked.
- [ ] **Server check script.** Gather the validation commands into a single script (chrony synchronized, signing enabled, UDP 123 listening, group and mode of the `ntp_signd` directory) and return "ok" or "fail" with the reason. It only enters the repository after being tested as a script.
- [ ] **Drift alert.** Monitor `chronyc tracking` (`Leap status` and offset) and the socket directory permission, with a warning if anything changes. Without it, a problem only shows up when someone notices the wrong time.
- [ ] **Correction time.** Carefully evaluate the `w32time` polling parameters (`MinPollInterval` and `MaxPollInterval`) for workstations that take long to correct. They were not changed because the policy that sets them touches many values at once.
- [ ] **GPO by script.** Evaluate creating and configuring the GPO by script (the PowerShell `GroupPolicy` module) against a Samba AD, to make the rollout repeatable. It needs testing, because not every GPO feature behaves the same with Samba.
- [ ] **Automatic documentation checks.** Check links and Markdown formatting on every change (GitHub Actions).

## Long term

Larger changes that need a project of their own.

- [ ] **Time server redundancy.** Today there is a single time server. Evaluate a second domain controller running chrony, and a fallback source for when the internet goes down (for example, a local reference clock), weighing the risk of serving time with no external reference.
- [ ] **Devices outside the domain.** Cameras, printers and phones were left out on purpose. Evaluate serving them in a controlled way, per network, with firewall rules and without widening `allow` beyond what is needed.
- [ ] **History and dashboard.** Record the drift of the server and the workstations over time (for example, with Prometheus and Grafana, or Zabbix) to see trends and problem workstations.
- [ ] **Fleet-wide check.** A report listing each workstation's source and drift without opening machine by machine. The mechanism is still to be defined (for example, a GPO startup script that writes the result to a share).
- [ ] **Deployment with Ansible.** An idempotent playbook that applies the directory permission, the `chrony.conf` and the service restart on one or more Samba AD controllers.
- [ ] **Multiple sites.** Adapt the design to environments with more than one site and more than one controller.

## What we learned

- **The problem was not where it looked.** The server had the right time, and the cause was it not answering signed time to the clients. Measuring before changing things avoided "fixing" the wrong thing.
- **Read the server before applying a recipe.** On a Samba compiled from source, paths change (`/usr/local/samba`, not `/var/lib/samba`). Confirm the socket path with `samba -b | grep -i NTP_SIGND_SOCKET_DIR`.
- **Keep commands apart from file contents.** Pasting lines that belong in `chrony.conf` into the terminal produces confusing errors. Every block should say where it goes.
- **One step at a time, with a backup and a way back.** Each change was small, validated and reversible before the next one.
- **Windows corrects the time gradually.** For small differences it speeds the clock up or down instead of jumping: 11 s took about 30 minutes. Measure again before concluding it failed.
- **Confirm the GPO by its values, not by its name.** `gpresult` shows the GPO name even when it was filtered out. The `NtpServer` and `Type` values live under `...\W32time\Parameters`.
- **In NT5DS mode, `SpecialPollInterval` does not apply.** The polling interval comes from other parameters.
- **Test the restart before trusting it.** Restarting the Samba service and the whole server showed that the socket directory permission persists.
- **Restrict before you link.** For the GPO, limit its reach (security filter on one workstation) before creating the link, and widen it in phases.
- **A report is not a measurement.** When someone says "the time is already right", record it as a report and measure later.
- **Better safe than sorry.** The workstations corrected themselves once the server was fixed, but the GPO remains as a second layer so nobody is left without a time source.
