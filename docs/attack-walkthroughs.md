# Attack Walkthroughs

Each attack is run from Kali against the lab, mapped to ATT&CK, tied to the rule that catches it, and closed with what a Tier 1 analyst should actually do. Rule IDs match `rules/local_rules.xml`.

## 1. SSH brute force — T1110.001

**Run:** `hydra -l root -P rockyou.txt ssh://10.10.10.50`
**Fires:** rule `100110` (level 10) after 8 failed logins from one source in 120s. Active response drops the IP.
**Analyst actions:** confirm the source is external/unexpected → check whether any attempt *succeeded* (rule 5715 / `sshd` accepted) → if a success followed the burst, escalate as possible compromise, not brute force. False positives: a misconfigured backup job hammering SSH with a stale key.

## 2. SQL injection on DVWA — T1190

**Run:** DVWA SQLi box, payload `1' OR '1'='1' -- -`
**Fires:** rule `100120` (level 12) on the Apache access log via PCRE2.
**Analyst actions:** pull the full `url` and `srcip` → check response size/HTTP status for signs the injection worked → look for follow-on requests (UNION SELECT, `information_schema`). False positives: security scanners you own (Nessus/Burp) — correlate with your scan schedule before escalating.

## 3. Command injection — T1059.004

**Run:** DVWA command-injection box, `127.0.0.1; cat /etc/passwd`
**Fires:** rule `100122` (level 13), Apache log + `auditd` on the web host.
**Analyst actions:** if `auditd` shows the web user actually spawned `cat`/`sh`, this is a confirmed RCE — isolate the host. The web-log alert alone is an *attempt*; the auditd correlation is what makes it an incident.

## 4. Web shell dropped — T1505.003

**Run:** attacker writes `shell.php` into `/var/www/html`
**Fires:** rule `100130` (level 14) — FIM new-file in web root — plus a `virustotal` hit if the shell is known.
**Analyst actions:** the FIM `whodata` field tells you which process wrote it. If it was the web server process, tie it back to the injection that allowed the write. Preserve the file, don't just delete it.

## 5. Credential dumping — T1003.001

**Run:** `mimikatz` / procdump against LSASS on a Windows endpoint
**Fires:** rule `100140` (level 14) — Sysmon EID 10, handle to `lsass.exe` with `0x1010`/`0x1fffff` access.
**Analyst actions:** check `sourceImage` — a signed AV/EDR process reading LSASS is a false positive; `rundll32`, `powershell`, or an unsigned binary is not. Escalate immediately; assume the host's cached credentials are exposed.

## 6. Persistence — T1136.001 / T1053.005

**Run:** create a local admin (`net user /add`, add to Administrators) and a scheduled task
**Fires:** rules `100150` (new account, level 12) and `100151` (scheduled task, level 11) from Windows Security events.
**Analyst actions:** correlate with the credential-access or lateral-movement alert that preceded it — persistence rarely arrives first. Confirm the creating account (`subjectUserName`) is authorised to make admins.

## Reading the alert levels

`0–6` log only · `7–11` medium, triage in queue · `12–14` high, enrich (VT/AbuseIPDB) and review · `15` critical, auto-block. The level is a *starting* priority — the correlation you do (did it succeed? who ran it?) is what decides the real severity.
