# SOC Red Team Wiki

<div align="center">

![SOC Red Team Logo](assets/logo.png)

**A comprehensive and structured guide for Security Operations Center Red Team activities, methodologies, tools, and incident response**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Last Updated](https://img.shields.io/badge/Last%20Updated-Feb%202026-blue.svg)](#changelog)
[![Documentation Level](https://img.shields.io/badge/Documentation-Professional-green.svg)](#)

</div>

---

## Table of Contents

- [00 - META](#00-meta)
- [01 - FOUNDATIONS](#01-foundations)
- [02 - RECON & ENUMERATION](#02-recon--enumeration)
- [03 - WEB SECURITY](#03-web-security)
- [04 - CREDENTIALS & CRYPTOGRAPHY](#04-credentials--cryptography)
- [05 - FORENSICS & TRAFFIC ANALYSIS](#05-forensics--traffic-analysis)
- [06 - LOGGING & SIEM](#06-logging--siem)
- [07 - INCIDENT PLAYBOOKS](#07-incident-playbooks)
- [08 - CLOUD SECURITY](#08-cloud-security)
- [09 - AUTOMATION](#09-automation)
- [10 - APPENDIX](#10-appendix)

---

<div align="center">

![Red Team Workflow](assets/workflow-diagram.png)

</div>

---

## 00-META

### META-01: Documentation Standards

**Purpose**: Consistency in reporting and knowledge sharing.

- **Tone**: Technical, concise, and objective.
- **Format**: Use Markdown for source docs, export to PDF/Word for distribution.
- **Screenshots**: Must include clear captions and highlight relevant areas.

### META-02: Update and Versioning

**Cadence**: Monthly reviews for tool updates.

**Version Log**:
- v1.0 - Initial Wiki Creation
- v1.1 - Added Cloud Security Module

### META-03: Rules of Engagement and Legal

- **Scope**: Always verify target IP/Domain against SOW.
- **Ethical Boundary**: No testing outside of approved windows.
- **Compliance**: GDPR, HIPAA, and local cybersecurity laws adherence.

### META-04: Evidence & Reference Standards

This wiki uses three tiers of evidence. Every claim must fall into one.

#### Tier 1 — Authoritative Source

Published standards, official documentation, or peer-reviewed research.

Cited inline as: `[Ref: <Document> — <Organization>]`

**Examples**: NIST SP 800-61, MITRE ATT&CK, Sysinternals Documentation, RFC 793.

#### Tier 2 — Practitioner Judgment [PJ]

Operational decisions, analyst heuristics, and workflow preferences derived from direct incident response, penetration testing, or SOC operations. These are experience-based observations that cannot be attributed to a single published source. They represent patterns observed across multiple engagements.

Cited inline as: `[Ref: Practitioner Judgment]`

**These claims are**:
- Defensible under peer review by experienced analysts.
- Subject to environmental variation (different orgs, different tooling).
- NOT presented as universal truths.

#### Tier 3 — Unverified / Anecdotal

**Not used in this wiki**. If a claim cannot meet Tier 1 or Tier 2 standards, it is removed.

#### Reference Block Format

Each major section ends with a consolidated reference block:

```
**Section References**
- [1] Document Name — Organization — URL
- [PJ] Practitioner Judgment — Defined in META-04
```

---

## 01-FOUNDATIONS

<div align="center">

![Networking & Systems Architecture](assets/foundation-architecture.png)

</div>

### FND-01: Networking Basics

- **OSI Model**: Understanding Layer 2 (Data Link) through Layer 7 (Application).
- **Protocols**: TCP/IP, DNS, HTTP/S, SMB, SSH, RDP.
- **Subnetting**: CIDR notation and VLAN tagging.

### FND-02: Linux Essentials

**File System Structure**:
- `/etc` - Configuration files
- `/var/log` - System logs
- `/bin` - Essential commands
- `/home` - User directories

**Permissions & Access**:
- `chmod` - Change file permissions
- `chown` - Change file ownership
- SUID/GUID bits - Special permissions

**CLI Mastery**:
- `grep` - Pattern matching
- `awk` - Text processing
- `sed` - Stream editing
- `find` - File searching
- `netstat` - Network statistics

### FND-03: Windows Essentials

**Active Directory**:
- Domain Controllers
- Organizational Units (OUs)
- Group Policy Objects (GPOs)
- Forest/Trees topology

**Registry Structure**:
- HKLM (Local Machine)
- HKCU (Current User)
- Auditing for persistence mechanisms

**PowerShell**:
- Execution policies
- Administrative task automation
- System administration cmdlets

---

## 02-RECON & ENUMERATION

<div align="center">

![Reconnaissance Workflow](assets/recon-workflow.png)

</div>

### REC-01: Nmap

**Purpose**: It's the gold standard. Tells you exactly what's living on the network.

[Ref: Nmap.org – Introduction/Legal]

#### When I Use This

- **SOC**: Validating firewall tickets. Hunting weird anomalies. Checking if "patched" boxes actually took the patch.
- **Red Team**: Mapping the attack surface. Hunting for that one dev box everyone forgot about.

[Ref: Practitioner Judgment]

#### Common Scan Profiles I Reuse

##### Quick Discovery
```bash
nmap -sn 192.168.1.0/24
```
Ping sweep. Finds live hosts fast.

[Ref: Nmap Network Scanning – Chapter 3: Host Discovery]

##### Full TCP (The noisy one)
```bash
nmap -p- -sV -sC -T4 -oA full_tcp <IP>
```
Hits all 65k ports. Scripts & versions on. Takes forever, but you won't miss anything.

[Ref: Nmap Network Scanning – Chapter 4 & 7: Port Scanning & Service Detection]

##### UDP Top 20
```bash
nmap -sU --top-ports 20 <IP>
```
Everyone forgets UDP. Good for finding hidden SNMP or DNS.

[Ref: Nmap.org – UDP Scan (-sU)]

#### Decision Logic & Analyst Judgment

- **IF** connect scan gets blocked **THEN** force SYN scan `-sS`.
- **IF** RDP/SMB is wide open **THEN** STOP immediately. Screenshot it. Report it.
- **IF** >100 ports open **THEN** it's likely a honeypot. Watch out.
- **IF** ancient version (e.g. Apache 2.2) **THEN** check CVEs first.
- **ABORT**: Latency hits >500ms. We're killing the network.
- **NO GO**: Gov/Mil ranges. SOW authorization required. Triple check this.

[Ref: Practitioner Judgment]

#### Output Handling

- Name it: `YYYY-MM-DD_<target>_<scantype>`.
- Use `-oA` every single time.
- Dump it in `/evidence/scans/`.

[Ref: Nmap.org – Output Formats] | [Nmap.org – Man Page](https://nmap.org/book/man.html)

---

### REC-02: Masscan

**Use Case**: Massive scale internet scanning (up to 25 million pps).

**Command**: `masscan -p80,443 --rate 1000 10.0.0.0/8`

**Purpose**: Speed. Pure chaos. Nmap takes days; this takes minutes. Asynchronous, stateless, massive throughput. No handshakes, just SYN packets.

[Ref: Masscan GitHub README – Asynchronous transmission]

#### When I Use This

- **SOC**: Panic-checking 10k endpoints for a zero-day (e.g., Log4j). Boss wants answers now.
- **Red Team**: Finding live ranges in a /16. Don't care about OS versions yet, just "is it alive?".

[Ref: Practitioner Judgment]

#### Common Scan Profiles I Reuse

##### Internet-wide fast scan
```bash
masscan -p80,443 --rate 10000 --banners <IP_RANGE>
```
"Yolo" scan. Grabs banners. Melts stateful firewalls.

**Risk**: Don't run this from office Wi-Fi. You will kill the router.

[Ref: Masscan GitHub README – Banner checking]

##### Targeted port sweep
```bash
masscan -p0-65535 <IP> --rate 1000
```
Need every open port on a critical asset immediately.

**Risk**: Can crash old IoT stacks.

[Ref: Practitioner Judgment]

##### Verification follow-up
```bash
masscan -p80 <IP> ; nmap -sV -p80 <IP>
```
Masscan finds it. Nmap identifies it.

**Risk**: Noisy logs. Blue team will see you.

[Ref: Practitioner Judgment]

#### Decision Logic & Analyst Judgment

- **IF** stateful firewall (Palo Alto) **THEN** cap at `--rate 500`.
- **IF** >20% packet loss **THEN** slow down. You're missing data.
- **IF** sensitive single host **THEN** use Nmap. Masscan is a sledgehammer.
- **IF** internal subnet **THEN** double-check auth. Looks like a worm.

[Ref: Practitioner Judgment]

#### Output Handling

- File: `masscan_results.json` or `.xml`.
- Usage: Pipe `.xml` to Nmap `-iL` or grep JSON.

[Ref: Masscan GitHub README – Output formats]

---

### REC-03: Shodan

**Intelligence**: Finding exposed IoT devices and industrial control systems.

**Filters**: `city:`, `org:`, `product:`, `os:`.

**Purpose**: Pre-scan surveillance. Maps exposure without sending packets. Avoids alerting the SOC before you're ready.

[Ref: Shodan Help Center – The Basics]

#### When I Use This

- **SOC**: Validating "critical" alerts from the scanner. Checking if the IP has a history of hosting malware.
- **Red Team**: Route planning. Finding the one unpatched Jenkins box to pivot through.

[Ref: Practitioner Judgment]

#### Query Patterns I Reuse

##### Org-based
```
org:"Target Corp" http.title:"Dashboard"
```
Admin panels.

**Risk**: Misses cloud assets (AWS/Azure) not registered to the Org ID.

##### ASN-based
```
asn:AS12345 port:3389
```
Maps the entire block for exposed RDP.

**Risk**: High noise on guest/public Wi-Fi ranges.

##### Port + Product
```
port:443 product:"VPN"
```
Remote access gateways.

**Risk**: Generic web servers faking headers.

##### SSL / Certificate
```
ssl.cert.subject.CN:"*.target.com"
```
Subdomain enumeration via cert transparency.

**Risk**: Misses legacy self-signed boxes.

[Ref: Shodan Help Center – Search Query Fundamentals]

#### Decision Logic & Analyst Judgment

- **IF** port is open in Shodan → **THEN** verify with Nmap. Data is often stale.
- **IF** "Last Update" >30 days → **THEN** assume it's changed. Rescan.
- **IF** Shodan contradicts Nmap → **THEN** trust Nmap. Live data wins.
- **IF** ICS/SCADA found → **THEN** STOP. Report immediately. Do not touch.

[Ref: Practitioner Judgment]

#### Output Handling

- **Screenshot**: Raw result including banner and timestamp.
- **Export**: `shodan download --limit 1000 target_data target_query`.
- **Reference**: Jira ticket gets the direct match URL.

[Ref: Shodan CLI Documentation]

---

### REC-04: OSINT Framework

**Domain Research**: Whois, DNSDumpster, Sublist3r.

**Social**: LinkedIn, Maltego, theHarvester.

**Purpose**: A structured mindset, not a tool list. Without a framework, OSINT is just aimless browsing. This structures the intake of information into actionable intelligence.

[Ref: OSINT Framework – About]

#### How I Use This

- **SOC**: Enrichment. Transforming a naked IP address into a story (Owner → History → Reputation) to decide on escalation.
- **Red Team**: Target definition. Mapping the human and digital footprint before touching any infrastructure.

[Ref: Practitioner Judgment]

#### OSINT Layers I Move Through

##### Domain & Infrastructure
**Question**: "Where does the organization live digitally?"

**Pitfall**: Assuming shared hosting IPs belong solely to the target.

##### Identity & Personnel
**Question**: "Who operates the key infrastructure?"

**Pitfall**: Conflating personal developer accounts with corporate assets.

##### Technology Stack
**Question**: "What is the underlying architecture?"

**Pitfall**: Trusting default headers or banners that haven't been scrubbed.

##### External Exposure
**Question**: "What assets have been forgotten?"

**Pitfall**: Neglecting third-party SaaS pointers (subdomain takeover risk).

##### Relationships & Trust Paths
**Question**: "Who does the target business trust?"

**Pitfall**: Overlooking insecure supply chain partners with trusted access.

[Ref: OSINT Framework – Domain/Public Records]

#### Decision Logic & Analyst Judgment

- **IF** ownership is obfuscated → **THEN** pivot to historical DNS or passive records.
- **IF** OSINT data conflicts with active scan → **THEN** trust OSINT for attribution, scanner for state.
- **IF** volume of data exceeds utility (e.g., 500+ emails) → **THEN** stop. Diminishing returns.
- **IF** high-impact leaks found (e.g., credentials) → **THEN** verify age/validity without active testing.

[Ref: Practitioner Judgment]

#### Output Handling

- **Notes**: Capture method, timestamp, and source URL for every claim.
- **Tickets/Reports**: Synthesize into a narrative identity, not a data dump.
- **NOT Saved**: Irrelevant personal data (family, non-professional life). Toxic to retain.

[Ref: Practitioner Judgment]

**Section References — 02-RECON-ENUM**

- [1] Nmap Network Scanning — Nmap.org — https://nmap.org/book/man.html
- [2] Masscan — Robert David Graham, GitHub — https://github.com/robertdavidgraham/masscan
- [3] Shodan Help Center — Shodan — https://help.shodan.io/
- [4] Shodan CLI Documentation — Shodan — https://cli.shodan.io/
- [5] OSINT Framework — https://osintframework.com/
- [6] NIST SP 800-115: Technical Guide to Information Security Testing — NIST — https://csrc.nist.gov/pubs/sp/800-115/final
- [PJ] Practitioner Judgment — Defined in META-04

---

## 03-WEB-SECURITY

<div align="center">

![Web Application Attack Surface](assets/web-security-mapping.png)

</div>

### WEB-01: OWASP Top 10

- **A01:2021**: Broken Access Control.
- **A03:2021**: Injection (SQLi, Command Injection).
- **A06:2021**: Vulnerable and Outdated Components.

### WEB-02: Burp Suite

**Proxy**: Intercepting and modifying HTTP traffic.

**Intruder**: Brute forcing and fuzzing.

**Repeater**: Manual vulnerability testing and payload refinement.

**Purpose**: The surgical table. Scanners are dumb; they miss logic and context. Burp is for understanding *why* the app broke, and crafting the specific payload to prove it.

[Ref: Practitioner Judgment]

#### When I Use This

- **SOC**: De-obfuscating attack traffic. Replaying a suspicious request to see if the WAF triggers or the backend 500s.
- **Red Team**: Manual exploitation. Thinking like a human to bypass logic flaws that automated fuzzers can't see.

[Ref: Practitioner Judgment]

#### Workflows I Reuse

##### Auth Testing
**Question**: "Is the login susceptible to brute-force or credential stuffing?"

**Mistake**: Forgetting to extract and carry the CSRF token for every attempt.

##### Access Control Validation
**Question**: "Can IDOR occur between two users of the same privilege level?"

**Mistake**: Testing with users who have different roles (Vertical vs Horizontal escalation confusion).

##### Injection Confirmation
**Question**: "Is that SQL error actionable or just verbose?"

**Mistake**: Sending raw payloads without URL encoding, causing syntax errors before the query hits.

##### Logic Flaw Exploration
**Question**: "Can I manipulate business logic (price, quantity) via the backend?"

**Mistake**: Trusting client-side validation or UI disabled states.

##### API Behavior Analysis
**Question**: "does the API choke on unexpected data types (arrays vs strings)?"

**Mistake**: Ignoring the `Content-Type` header (sending XML when it expects JSON).

[Ref: PortSwigger – Methodology]

#### Decision Logic & Analyst Judgment

- **IF** scanner alerts XSS → **THEN** verify manually. Scanners fail at context (DOM vs Reflected).
- **IF** unauthenticated request succeeds → **THEN** determine if data is public or leaked. Don't assume bug.
- **IF** WAF blocks automation → **THEN** switch to manual, slow requests. Time delays > evasion scripts.
- **IF** testing DELETE/UPDATE → **THEN** use a dedicated test account. Never corrupt production data.

[Ref: Practitioner Judgment]

#### Output Handling

- **Saved**: The critical Request/Response pair verifying the flaw.
- **Attached**: The exact payload syntax, not the entire Repeater tab.
- **NOT Saved**: Passive proxy history. It's just noise.

[Ref: Practitioner Judgment]

### WEB-03: Browser DevTools

- **Network Tab**: Analyzing XHR/API calls.
- **Console**: Executing JS for XSS verification.
- **Storage**: Inspecting Cookies and LocalStorage tokens.

**Section References — 03-WEB-SECURITY**

- [1] OWASP Top 10:2021 — OWASP Foundation — https://owasp.org/Top10/
- [2] PortSwigger Web Security Academy — PortSwigger — https://portswigger.net/web-security
- [3] Burp Suite Documentation — PortSwigger — https://portswigger.net/burp/documentation
- [PJ] Practitioner Judgment — Defined in META-04

---

## 04-CREDENTIALS & CRYPTOGRAPHY

<div align="center">

![Password Attack Vectors](assets/password-attacks.png)

</div>

### CRY-01: Password Fundamentals

- **Hashing**: MD5, SHA-1, SHA-256, bcrypt, scrypt.
- **Salting**: Importance of nonces in preventing rainbow table attacks.

### CRY-02: Hashcat

**GPU Power**: Utilizing OpenCL/CUDA for high-speed cracking.

**Modes**:
- `-m 0` (MD5)
- `-m 1000` (NTLM)
- `-m 1800` (sha512crypt)

### CRY-03: John the Ripper

**CPU Cracking**: Versatile tool for shadow files and custom formats.

**Jumbo Version**: Support for almost any hash type.

**Section References — 04-CREDENTIALS-CRYPTO**

- [1] Hashcat Wiki — Hashcat — https://hashcat.net/wiki/
- [2] John the Ripper Documentation — Openwall — https://www.openwall.com/john/doc/
- [PJ] Practitioner Judgment — Defined in META-04

---

## 05-FORENSICS & TRAFFIC ANALYSIS

<div align="center">

![Network Forensics Process](assets/forensics-workflow.png)

</div>

### FOR-01: Wireshark

**Filters**: `http.request.method == "POST"`, `ip.addr == 192.168.1.1`.

**Streams**: Following TCP/TLS streams to reconstruct sessions.

**Purpose**: Logs are summary; packets are evidence. Wireshark is the final arbiter for proving *what* happened down to the byte level when logs are ambiguous or missing.

[Ref: Wireshark User's Guide – Introduction]

#### When I Use This

- **SOC**: "Did the firewall drop the packet *after* the payload was delivered?" Differentiating between a blocked attempt and a successful exploit.
- **Red Team**: Troubleshooting C2 jitter profiles. Ensuring encryption handshakes look legitimate to avoiding JA3 fingerprinting.

[Ref: Practitioner Judgment]

#### Traffic Patterns I Look For

##### Beaconing
**Question**: "Is this heartbeat automated?"

**Analysis**: Looking for mathematical precision in `Time Delta` between packets (e.g., exactly 5.0s). Humans are messy; code is precise.

**Misinterpretation**: NTP and authorized telemetry also beacon. Context is key.

##### Credential Leakage
**Question**: "Did the user send their password in cleartext?"

**Analysis**: Inspecting the Request Body of HTTP POSTs or Telnet/FTP streams.

**Misinterpretation**: Seeing Base64 and assuming it's encrypted. It's encoding, not encryption.

##### Suspicious POST Bodies
**Question**: "Is this a webshell interaction?"

**Analysis**: High asymmetry (Tiny request -> Huge response).

**Misinterpretation**: Valid file uploads also follow this pattern. Check the *content*, not just size.

##### Abnormal DNS Behavior
**Question**: "Is DNS being used as a covert channel?"

**Analysis**: High entropy domains or TXT records carrying payload data (e.g., `base64_string.malware.com`).

**Misinterpretation**: CDNs and AV signature updates also use weird, long DNS queries.

##### Encrypted Traffic Anomalies
**Question**: "Is this malware hiding in SSL?"

**Analysis**: Inspecting the TLS Client Hello. Does the JA3 hash match standard browsers or known hacking tools?

**Misinterpretation**: Custom legacy apps often use outdated TLS stacks that look suspicious.

[Ref: Practitioner Judgment]

#### Decision Logic & Analyst Judgment

- **IF** packet payload is encrypted (TLS) → **THEN** pivot to metadata analysis (Certificate issuer, validity, SNI).
- **IF** the 3-way handshake is missing → **THEN** do not confirm the session state. You are looking at a fragment.
- **IF** Retransmission rate > 10% → **THEN** stop analysis. Network instability is corrupting your view.
- **IF** capture file > 500MB → **THEN** filter with `tshark` or `tcpdump` first. Wireshark will crash.

[Ref: Wireshark User's Guide – Statistics]

#### Output Handling

- **Saved**: Export *only* the specific "Specified Packets" proving the finding.
- **Trimmed**: Isolate the relevant TCP Stream.
- **Discarded**: Broadcast traffic (ARP, SSDP) that inevitably clutters the view.

[Ref: Practitioner Judgment]

---

### FOR-02: Tcpdump

**Purpose**: Wireshark is the lab; Tcpdump is the field. It captures evidence on servers where installing a GUI is impossible or reckless. It is the only way to see what is happening *right now* on a headless Linux box under attack.

[Ref: Tcpdump Man Page – Overview]

#### When I Use This

- **SOC**: "Is the C2 traffic actually leaving the interface?" Verifying firewall rules in real-time.
- **Red Team**: "Did my exploit land?" Listening for the callback on a pivot box without touching disk (piping to netcat).

[Ref: Practitioner Judgment]

#### Command Patterns I Reuse

##### The "Don't crash the box" Ring Buffer
```bash
tcpdump -i eth0 -n -s0 -w /tmp/evidence.pcap -C 100 -W 5
```
Safety: Rotates 5 files of 100MB. Prevents filling the disk and crashing the service you're trying to save.

##### The "Just the Headers" Triage
```bash
tcpdump -i any -n -c 100 'port 443'
```
Speed: Quick verification of connectivity. `-n` (No DNS) is mandatory; resolving IPs takes seconds you don't have.

##### The "Find the Beacon" Filter
```bash
tcpdump -i eth0 -n "dst host <C2_IP> or dst port 443"
```
Focus: Ignoring the noise of SSH/monitoring to see the specific threat.

[Ref: Practitioner Judgment]

#### Decision Logic & Analyst Judgment

- **IF** connecting via SSH → **THEN** verify your filter excludes port 22. You will loop traffic otherwise.
- **IF** system is high-load → **THEN** use `-s 64` (snaplen) to capture headers only. Drop the payload to save CPU.
- **IF** investigating a container → **THEN** you must capture on the bridge interface (`docker0`), not just `eth0`.
- **IF** text output is scrolling too fast → **THEN** stop. You are missing data. Write to file (`-w`) and analyze later.

[Ref: Practitioner Judgment]

#### Common Analyst Mistakes

- **Forgetting `-n`**: Waiting for DNS timeouts while the attacker moves laterally.
- **Grepping live output**: `tcpdump | grep` is lossy. You see lines, but you lose the packet structure context.
- **Capturing to root partition**: Filling `/` causes the OS to panic. Always write to `/tmp` or `/var`.

[Ref: Practitioner Judgment]

#### Output Handling

- **Saved**: Ring buffer PCAPs (`file.pcap0`, `file.pcap1`).
- **Discarded**: Live text output. It is untrusted validation, not evidence.
- **Exfiltration**: SCP the PCAP off the box immediately. Do not analyze on the compromised host.

[Ref: Practitioner Judgment]

---

### FOR-03: Memory & Disk Forensics

- **Volatility**: Analyzing RAM dumps for hidden processes and connections.
- **Autopsy**: GUI-based disk image analysis and artifact recovery.

**Section References — 05-FORENSICS-TRAFFIC**

- [1] Wireshark User's Guide — Wireshark Foundation — https://www.wireshark.org/docs/wsug_html_chunked/
- [2] Tcpdump Man Page — TCPDUMP.org — https://www.tcpdump.org/manpages/tcpdump.1.html
- [3] Volatility 3 Framework — Volatility Foundation — https://github.com/volatilityfoundation/volatility3
- [4] RFC 793: Transmission Control Protocol — IETF — https://www.rfc-editor.org/rfc/rfc793
- [PJ] Practitioner Judgment — Defined in META-04

---

## 06-LOGGING & SIEM

<div align="center">

![SIEM & Logging Architecture](assets/siem-architecture.png)

</div>

### LOG-01: Linux Logging

- **Syslog**: `/var/log/syslog`, `/var/log/auth.log`.
- **Journalctl**: `journalctl -u ssh.service` for systemd logs.

### LOG-02: Windows Sysmon

**Visibility**: Event IDs for Process Creation (1), Network Connection (3), and File Creation (11).

**Config**: Using SwiftOnSecurity or customized XML templates.

**Purpose**: Windows Event Logs are blind; Sysmon gives them sight. It correlates disparate actions (process, network, file) via a unique `ProcessGUID` to tell the *entire* story of an execution, not just that it happened.

[Ref: Sysinternals – Sysmon]

#### When I Use This

- **SOC**: "Does this script actually connect to a foreign IP?" Differentiating between an admin doing their job and a C2 beacon.
- **Red Team**: "Did my execution chain trigger detection?" Verifying if evasion techniques successfully bypassed logging.

[Ref: Practitioner Judgment]

#### Event Patterns I Care About

##### Suspicious Parent/Child Chains
**Question**: "Did the execution flow make sense?"

**Pattern**: `winword.exe` spawning `powershell.exe`. Office apps should almost never launch terminals.

**Misread**: Admins launching scripts via RMM tools (e.g., Kaseya, SCCM) often create weird parentage.

##### Living-off-the-land (LOLBins)
**Question**: "Is trusted code interacting with the internet?"

**Pattern**: `rundll32.exe` or `regsvr32.exe` making outbound network connections.

**Misread**: Printer drivers and legacy updaters often look like DLL injection attacks.

##### Unexpected Network Connections
**Question**: "Is this CLI tool phoning home?"

**Pattern**: Non-browser processes (`notepad.exe`, `cmd.exe`) initiating outbound TCP connections.

**Misread**: Windows Update components (`svchost.exe`) connect to diverse CDN IPs constantly.

##### Persistence Mechanisms
**Question**: "Is malware trying to survive a reboot?"

**Pattern**: File creation events in `Startup` folders or modification of `Run` keys.

**Misread**: Chrome and Adobe updaters legitimately drop files here.

##### Credential Access Attempts
**Question**: "Is a process touching the keys to the castle?"

**Pattern**: A non-system process requesting `GrantedAccess` to `lsass.exe`.

**Misread**: EDR/AV agents scanning LSASS memory for integrity checks.

[Ref: Practitioner Judgment]

#### Decision Logic & Analyst Judgment

- **IF** Sysmon reveals an attack chain → **THEN** escalate. High fidelity means high confidence.
- **IF** SIEM alert contradicts Sysmon → **THEN** trust Sysmon data. Log parsers often strip context.
- **IF** only a single event exists → **THEN** pivot to `ProcessGUID`. Look at the 10 minutes *before* the event.
- **IF** signatures are valid but behavior is bad → **THEN** investigate. Signed binaries are often abused (BYOV).

[Ref: Practitioner Judgment]

#### Output Handling

- **Preserved**: `ProcessGUID` and `ParentProcessGUID` are the primary keys for all correlation.
- **Correlated**: Grouping events by `LoginId` to reconstruct the user's session scope.
- **Ignored**: High-frequency noise like browser `ImageLoad` events (unless debugging specific DLL issues).

[Ref: Sysinternals – Sysmon configuration]

---

### LOG-03: SIEM (Splunk & ELK)

- **Ingestion**: Forwarders (Beats, Splunk Forwarders).
- **Alerting**: Writing KQL (Kibana Query Language) or SPL (Search Processing Language) queries.

**Section References — 06-LOGGING-SIEM**

- [1] Sysmon — Microsoft Sysinternals — https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon
- [2] SwiftOnSecurity Sysmon Config — GitHub — https://github.com/SwiftOnSecurity/sysmon-config
- [3] Windows Event Log Reference — Microsoft — https://learn.microsoft.com/en-us/windows/security/threat-protection/auditing/security-auditing-overview
- [4] MITRE ATT&CK Data Sources — MITRE — https://attack.mitre.org/datasources/
- [PJ] Practitioner Judgment — Defined in META-04

---

## 07-INCIDENT-PLAYBOOKS

<div align="center">

![Incident Response Timeline](assets/incident-timeline.png)

</div>

### IR-01: Web App Incident

1. **Detection**: Anomalous traffic spikes or 500 error logs.
2. **Containment**: Terminate sessions, rotate API keys.
3. **Eradication**: Patching the vulnerability (e.g., SQLi).

**Purpose**: When the phone rings at 2 AM, IQ drops by 50 points. This playbook is the external brain to prevent panic-induced mistakes during a web application crisis.

[Ref: Practitioner Judgment]

#### The "Oh S*t" Moment (Trigger)

- WAF logs show 10,000 SQL errors in 1 minute.
- Database CPU spikes to 100% with no legitimate traffic increase.
- Threat Intelligence warns of active exploitation of a stack component (e.g., Log4j).

[Ref: Practitioner Judgment]

#### What Not To Do (Panic Checks)

- **Do NOT** reboot the server immediately. You destroy RAM evidence of the webshell or process injection.
- **Do NOT** block the IP without checking if it's a NAT gateway (blocking 10,000 users).
- **Do NOT** edit code directly on production to "fix" it.

[Ref: Practitioner Judgment]

#### Decision Logic & Analyst Judgment

- **IF** data exfiltration is confirmed → **THEN** taking the app offline > business continuity.
- **IF** the vector is unknown → **THEN** preserve full web logs *before* restarting services or flushing caches.
- **IF** public disclosure is imminent → **THEN** Legal/PR takes command. Tech support stops talking.

[Ref: NIST SP 800-61 Rev. 2 – Incident Handling]

#### Output Handling

- **Draft**: A raw timeline of every command run (UTC).
- **Evidence**: Web server logs, WAF logs, and database transaction logs stored *offline*.
- **Report**: Just the facts (method, impact, timeline). No "I think".

[Ref: Practitioner Judgment]

---

### IR-02: Ransomware

**MITRE ATT&CK**: T1486 (Data Encrypted for Impact), T1490 (Inhibit System Recovery)

1. **Isolation**: Disconnect affected machines from the network.
2. **Identification**: Determine ransomware variant via encrypted file extensions.
3. **Recovery**: Restore from offline backups after OS hardening.

**Purpose**: Ransomware response fails when speed replaces judgment. This playbook exists to prevent irreversible damage caused by premature restoration, attacker re-entry, or evidence destruction.

[Ref: NIST SP 800-61 Rev. 2 – Incident Handling]

#### What Usually Goes Wrong

- Treating ransomware as a *single host problem* instead of a lateral movement problem.
- Restoring backups before understanding the initial access vector.
- Powering systems off blindly, destroying volatile indicators of encryption staging.
- Letting executives force "restore now" decisions without risk framing.
- Assuming encryption equals data exfiltration (often untrue, sometimes catastrophic to assume).

[Ref: Practitioner Judgment]

#### Key Decision Points

- **IF** encryption is still actively spreading → **THEN** isolation outranks business uptime.
- **IF** backups are reachable from the infected network → **THEN** assume backups are compromised.
- **IF** ransomware binary executed as SYSTEM → **THEN** credential compromise is likely broader.
- **IF** exfiltration indicators are absent → **THEN** avoid breach disclosure assumptions.
- **IF** attacker C2 is still beaconing → **THEN** restoration is premature.

[Ref: NIST SP 800-61 Rev. 2 – Containment Strategy]

#### Evidence I Trust Most

- Sysmon process trees showing encryption tooling and parent execution flow.
- File system timestamp clustering (sudden mass writes).
- Network telemetry showing SMB or RDP fan-out before encryption.
- Absence of large outbound transfers (important negative signal).
- Ransom notes alone are *not* evidence of scope.

[Ref: Practitioner Judgment]

#### Stop Conditions

- Encryption confirmed as contained to a single, non-critical host.
- Backups verified offline, immutable, and untouched.
- Initial access vector reasonably identified.
- Executive pressure exceeds evidentiary confidence → escalate risk, do not accelerate.

[Ref: Practitioner Judgment]

---

### IR-03: Credential Compromise

**MITRE ATT&CK**: T1078 (Valid Accounts), T1110 (Brute Force), T1528 (Steal Application Access Token)

1. **Invalidation**: Reset passwords and revoke MFA tokens.
2. **Audit**: Review login history for geographical anomalies.

**Purpose**: Credential compromise is dangerous because it looks benign. The threat isn't login success — it's *what was done quietly afterward*.

[Ref: NIST SP 800-61 Rev. 2 – Incident Categorization]

#### What Usually Goes Wrong

- Treating password resets as incident closure.
- Over-reliance on geo-IP anomalies (VPNs exist; attackers know this).
- Ignoring token-based access (OAuth, API keys, refresh tokens).
- Resetting credentials before reviewing historical activity.
- Assuming MFA guarantees safety.

[Ref: Practitioner Judgment]

#### Key Decision Points

- **IF** login is valid but behavior is abnormal → **THEN** treat as compromise, not anomaly.
- **IF** access token lifetime exceeds password reset → **THEN** reset is ineffective.
- **IF** privileged role used → **THEN** expand scope beyond the user account.
- **IF** activity aligns with business hours → **THEN** do not downgrade severity.
- **IF** no malicious actions observed → **THEN** continue monitoring before containment escalation.

[Ref: NIST SP 800-61 Rev. 2 – Analysis Phase]

#### Evidence I Trust Most

- IAM audit logs showing permission use, not just authentication.
- API calls made *after* successful login.
- Sysmon network events tied to the user session.
- Access to resources the user normally never touches.
- MFA challenge logs (approve/deny patterns matter).

[Ref: Practitioner Judgment]

#### Stop Conditions

- Compromise window bounded with no lateral or privileged activity.
- All authentication artifacts (passwords, tokens, sessions) rotated.
- No persistence mechanisms identified.
- Evidence supports benign explanation with corroboration.

[Ref: Practitioner Judgment]

---

### IR-05: Lateral Movement / Active Intrusion

**MITRE ATT&CK**: T1021 (Remote Services), T1570 (Lateral Tool Transfer), T1003 (OS Credential Dumping)

1. **Isolation**: Sever (don't just block) the breach point.
2. **Containment**: Reset compromised credentials and lock down lateral paths (WinRM/SMB).
3. **Eradication**: Hunt for persistence (Scheduled Tasks, WMI, Services).

**Purpose**: This is the "Hands-on-Keyboard" phase. The attacker is inside, authenticated, and moving. Speed is the only metric that matters. If you wait for a meeting, you lose the domain.

[Ref: NIST SP 800-61 Rev. 2 – Containment Strategy]

#### The "Oh S*t" Moment (Trigger)

- `PsExec.exe` or `wmic` execution on a non-IT workstation.
- Domain Admin account logging in to a receptionist's PC.
- Cobalt Strike Beacon (Jitter ~0) detected on a server.

[Ref: Practitioner Judgment]

#### What Not To Do (Panic Checks)

- **Do NOT** just block the IP. They have credentials; they will just VPN back in from a Starbucks.
- **Do NOT** spectate the attacker to "see what they do". This isn't a movie; they are exfiltrating data *now*.
- **Do NOT** reset krbtgt unless you are ready to break every service account in the org.

[Ref: Practitioner Judgment]

#### Decision Logic & Analyst Judgment

- **IF** Domain Admin is compromised → **THEN** initiate total forest shutdown procedures. Trust nothing.
- **IF** only one host is affected → **THEN** isolate it immediately (VLAN quarantine / physical pull).
- **IF** attacker is using RDP → **THEN** kill the session *and* disable the account.
- **IF** you find one C2 beacon → **THEN** assume there are three more. Hunt for the persistence.

[Ref: Practitioner Judgment]

#### Evidence I Trust Most

- **Windows Event 4624**: Specific Logon Types (Type 3 = Network, Type 10 = RDP).
- **Sysmon Event 1**: `ParentImage` anomalies (e.g., `services.exe` spawning `cmd.exe`).
- **Sysmon Event 3**: Outbound connections from `lsass.exe` or `spoolsv.exe`.
- **PowerShell Logs (4104)**: Base64 encoded commands or "Invoke-Expression".

[Ref: Practitioner Judgment]

#### Stop Conditions

- No new hosts accessed for >4 hours.
- All compromised accounts identified and reset (Password + MFA).
- Ticket Granting Ticket (krbtgt) rotated (if Domain compromise suspected).
- Golden Ticket attack vector ruled out.

[Ref: Practitioner Judgment]

**Section References — 07-INCIDENT-PLAYBOOKS**

- [1] NIST SP 800-61 Rev. 2: Computer Security Incident Handling Guide — NIST — https://csrc.nist.gov/pubs/sp/800-61/r2/final
- [2] MITRE ATT&CK Enterprise Matrix — MITRE — https://attack.mitre.org/matrices/enterprise/
- [3] Microsoft Security Event Log Reference — Microsoft — https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/plan/appendix-l--events-to-monitor
- [4] Sysmon — Microsoft Sysinternals — https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon
- [PJ] Practitioner Judgment — Defined in META-04

---

## 08-CLOUD-SECURITY

<div align="center">

![Cloud Security Model](assets/cloud-security-model.png)

</div>

### CLD-01: Cloud Fundamentals

- **Shared Responsibility**: Provider (Infrastructure) vs User (Data/Config).
- **Services**: AWS EC2/S3, Azure VMs, GCP Cloud Run.

### CLD-02: Identity & Access Management (IAM)

- **Principle of Least Privilege**: Minimum access required for tasks.
- **Policies**: JSON-based permissions and Role-Based Access Control (RBAC).

---

## 09-AUTOMATION

<div align="center">

![Automation & Scripting](assets/automation-tools.png)

</div>

### AUT-01: Python for Security

**Libraries**:
- `requests` - Web requests
- `scapy` - Packet crafting
- `impacket` - Network protocols

**Scripting**: Automating recon and log parsing.

### AUT-02: Bash & PowerShell

**Bash**: One-liners for mass-editing configuration and parsing logs.

**PowerShell**:
- `Invoke-WebRequest` - Web requests
- `Get-WinEvent` - Event log retrieval
- `ActiveDirectory` module - AD administration

---

## 10-APPENDIX

<div align="center">

![Resource Directory](assets/appendix-resources.png)

</div>

### A-01: Cheat Sheets

Essential quick-reference resources:

- **RevShells.com** — https://www.revshells.com
- **GTFOBins** — https://gtfobins.github.io
- **LOLBAS** — https://lolbas-project.github.io

### A-02: Glossary

**Common Terms**:

- **C2** - Command and Control.
- **CVE** - Common Vulnerabilities and Exposures.
- **TTP** - Tactics, Techniques, and Procedures.
- **OSINT** - Open Source Intelligence.
- **SIEM** - Security Information and Event Management.
- **EDR** - Endpoint Detection and Response.
- **IAM** - Identity and Access Management.
- **BYOV** - Bring Your Own Vulnerability.

---

## Changelog

**Current Version**: v1.1 (February 2026)

### Recent Updates

- Enhanced documentation structure with visual elements
- Added professional methodology diagrams
- Integrated image placeholders for key sections
- Improved formatting and readability

### Future Enhancements

- Additional incident playbook scenarios
- Container security and Kubernetes guidance
- Supply chain security guidelines
- Advanced persistence techniques

---

<div align="center">

### Getting Started

To contribute or access this wiki:

1. Clone the repository
2. Review the META section for documentation standards
3. Start with FOUNDATIONS if new to security concepts
4. Reference INCIDENT-PLAYBOOKS for hands-on scenarios

**Questions or Contributions?**

Please submit issues, pull requests, or discussions through GitHub.

---

**Professional Security Knowledge Management**

[Back to top](#soc-red-team-wiki)

</div>
