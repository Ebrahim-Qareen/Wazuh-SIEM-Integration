# Build Guide — Wazuh SIEM Integration Lab

Order matters: manager first, then agents, then each integration on top of a working agent feed. Every step ends with a check you can see in the dashboard.

## 1. Manager, indexer and dashboard (single node)

```bash
# Ubuntu 22.04, 4 vCPU / 8 GB RAM minimum
curl -sO https://packages.wazuh.com/4.x/wazuh-install.sh
sudo bash ./wazuh-install.sh -a
```

The installer prints the admin password at the end — store it. Log in at `https://<manager-ip>`.

**Check:** `Server management → Status` shows `wazuh-manager`, `wazuh-indexer`, `filebeat` all green.

## 2. Windows agents + Sysmon

Install the agent from `Agents → Deploy new agent`, pointing it at the manager IP. Then add Sysmon for real endpoint telemetry:

```powershell
# Install Sysmon with a filtering config (SwiftOnSecurity base is a good start)
sysmon64.exe -accepteula -i sysmonconfig.xml
```

Add the Sysmon channel to the agent's `ossec.conf`:

```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

Restart the agent service. **Check:** `Discover → wazuh-alerts-*`, filter `data.win.system.channel: "Microsoft-Windows-Sysmon/Operational"` — events appear.

## 3. File Integrity Monitoring + VirusTotal

FIM in `ossec.conf` (`syscheck` block), with `report_changes` and `whodata` so you get *who* changed the file:

```xml
<syscheck>
  <directories check_all="yes" report_changes="yes" whodata="yes">/var/www/html</directories>
  <directories check_all="yes" whodata="yes">C:\Windows\System32\drivers\etc</directories>
</syscheck>
```

VirusTotal integration on the manager (`ossec.conf`), keyed to FIM events:

```xml
<integration>
  <name>virustotal</name>
  <api_key>YOUR_VT_API_KEY</api_key>
  <group>syscheck</group>
  <alert_format>json</alert_format>
</integration>
```

**Check:** modify a file in a monitored path → a `syscheck` alert fires; drop an EICAR test file → a `virustotal` alert with `data.virustotal.positives` appears.

## 4. Suricata → Wazuh

Install Suricata on the Ubuntu sensor, enable `eve.json`, and have the agent read it:

```xml
<localfile>
  <log_format>json</log_format>
  <location>/var/log/suricata/eve.json</location>
</localfile>
```

**Check:** run `nmap -sS <target>` from Kali → Suricata `alert` events land in `wazuh-alerts-*` under `data.alert.signature`.

## 5. Active response

Auto-block repeat offenders (`ossec.conf` on the manager):

```xml
<active-response>
  <command>firewall-drop</command>
  <location>local</location>
  <rules_id>100110</rules_id>
  <timeout>600</timeout>
</active-response>
```

**Check:** trigger the SSH brute-force rule (`100110`) → the source IP is dropped for 10 minutes; confirm in `Server management → Logs`.

## 6. Load the custom rules

```bash
sudo cp rules/local_rules.xml /var/ossec/etc/rules/
sudo cp decoders/local_decoder.xml /var/ossec/etc/decoders/
sudo systemctl restart wazuh-manager
```

Before restart, paste a sample log into `Tools → Ruleset Test` and confirm the intended rule ID fires. A rule that never matches is almost always a field-name mismatch — check the decoded field names in the test output first.

## Pitfalls

- `whodata` on Windows needs the Audit policy for the directory enabled, or you get changes with no user.
- VirusTotal free tier is 4 requests/minute — don't point it at a noisy path.
- Suricata `eve.json` grows fast; rotate it or the agent lags.
