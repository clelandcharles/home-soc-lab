# Detection Report 01: TCP SYN Port Scan

**Analyst:** Charles Cleland

**Date:** September 28, 2026

**Lab Environment:** Home SOC Lab (VirtualBox – Kali Linux attacker, Windows 10 target, Splunk SIEM)

---

## 1. Summary

A TCP SYN scan was launched from an attacker host (10.10.10.10, Kali Linux) against the target host (10.10.10.20, Windows 10) using Nmap, attempting connections against 1,000 TCP ports. Windows Firewall blocked all attempted connections and logged the activity, which was forwarded to Splunk using a Universal Forwarder then identified through log analysis. This exercise showed the full lifecycle from attack to detection, creating attacker activity, capturing it in host logs, and identifying it through SIEM searches.

---

## 2. Environment

| Role | Hostname | IP Address | OS |
|---|---|---|---|
| Attacker | Kali | 10.10.10.10 | Kali Linux |
| Target | Windows-10-Targ | 10.10.10.20 | Windows 10 |
| SIEM | (host machine) | 10.10.2.2 | Splunk Enterprise (free tier) |

Both VMs operate on an isolated internal virtual network (labnet), which is not reachable from the home network or internet. A separate NAT adapter on both VMs provides internet access and Splunk log forwarding.

**Logging configuration on target:**
- Sysmon (SwiftOnSecurity config) – process, network, and file system collection
- Windows Security Auditing enabled for:
    - Filtering Platform Connection
    - Filtering Platform Packet Drop
- Splunk Universal Forwarder – forwarding Sysmon, Security, and System event logs to the Splunk indexer

---

## 3. Attack Details

**Tool:** Nmap 7.94

**Command:**
```
sudo nmap -sS -Pn 10.10.10.20
```

**Parameters:**
- `-sS` – TCP SYN scan (half open scan, doesn’t complete TCP handshake)
- `-Pn` – skip host directory ping, assume host is up

**Time of Scan:** 2026-09-28, 15:06-15:07 EDT
**Duration:** 46.25 seconds

**Result from attacker’s perspective:**
```
Nmap scan report for 10.10.10.20
Host is up (0.00063s latency).
All 1000 scanned ports are in ignored states.
Not shown: 1000 filtered tcp ports (no-response)
```

All 1,000 scanned ports returned no response (represented by “filtered” in attacker’s perspective), indicating that Windows Firewall dropped every probe instead of responding with a TCP RST (closed) or SYN-ACK (open). From the attacker’s side, this scan yielded no actionable information about open services.

---

## 4. Detection

**Log source:** Windows Security Event Log (WinEventLog:Security), forwarded via Splunk Universal Forwarder

**Relevant Event IDs:**
- `5152` – Windows Filtering Platform blocked a packet
- `5156` – Windows Filtering Platform allowed a connection
- `5157` – Windows Filtering Platform blocked a connection

**Splunk searches used:**
```spl
index=main "10.10.10.10" (EventCode=5156 OR EventCode=5157 OR EventCode=5152)
```

**Result:** 2,012 matching events within a ~60 second window, all received from `10.10.10.10`

**Summarized with:**
```spl
index=main "10.10.10.10" (EventCode=5156 OR EventCode=5157 OR EventCode=5152) | stats dc(Destination_Port) as unique_ports, count as total_events by Source_Address
```

This aggregation shows that a single source IP (10.10.10.10) generating connection attempts against 1,000 distinct destination ports on the target within a single minute.

---

## 5. Detection Logic

A single host attempting to contact many distinct destination ports on another host within a short window of time is signature of port scanning reconnaissance. Legitimate traffic from a single client to a single server typically is not spread across hundreds of destination ports in seconds, as normal application traffic targets one or possibly a handful of known ports repeatedly.

**Key indicators used to identify this as a scan instead of normal traffic:**
- **High port diversity from one source** – ~1000 unique destination ports from a single IP
- **Short time window** – all activity was compressed into under a minute
- **Uniform packet pattern** – consistent with automated tooling (Nmap) instead of manual user activity

---

## 6. Recommended Triage Steps

1. **Identify the source:** Confirm whether 10.10.10.10 is an authorized asset (for example a vulnerability scanner) or unexpected/unauthorized.
2. **Check scan outcome:** Review whether any ports responded as open or allowed (EventCode=5156). This scan showed all ports filtered, meaning no services were exposed to the scanner.
3. **Watch for future activity from source:** A scan typically is used as reconnaissance before a targeted exploitation attempt. Monitor the same source IP for subsequent connection attempts, authentication attempts, or exploitation traffic against any port.
4. **Correlate with critical assets:** If the target was a production server (instead of a lab VM), this would warrant escalation per the scanning/reconnaissance runbook.
5. **Consider blocking or alerting:** In a live environment, repeated scanning from the same source may justify a firewall block and a SIEM alert rule for future recurrence

---

*This report was produced in a fully isolated home lab environment for educational and portfolio purposes. No systems outside the lab’s internal virtual network were scanned or accessed.*
