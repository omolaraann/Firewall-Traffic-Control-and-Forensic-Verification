# 🔥 Firewall Traffic Control and Forensic Verification

## 📌 Overview

This repository contains my practical evidence and documentation for **SBT-DF203: Basic Networking Skills for Digital Forensics — Lab 6: Firewall Traffic Control and Forensic Verification**.

The practical investigates the behaviour of a Linux host-based firewall when HTTP traffic from a designated laboratory client is allowed and subsequently blocked using a narrowly scoped `iptables` rule. The investigation correlates firewall configuration and rule counters with packet-level evidence obtained using Wireshark/TShark and application-level behaviour observed using `curl`.

The laboratory was performed in an authorised isolated virtual environment using a Linux server VM, a laboratory client VM, Apache2, `iptables`, `curl`, and TShark/Wireshark.

---

## 🎯 Objectives

The objectives of this practical were to:

* 🛡️ Explain host-based and network-based firewalls.
* 🔗 Identify and interpret the `INPUT`, `OUTPUT`, and `FORWARD` iptables chains.
* 💾 Preserve and hash the original firewall configuration before making changes.
* 🌐 Establish the baseline network configuration of the protected server.
* 🖥️ Verify that Apache2 was listening on TCP port 80.
* ✅ Establish successful baseline HTTP connectivity from the laboratory client.
* 📡 Capture an allowed HTTP session.
* 🚫 Implement a narrowly scoped source-specific TCP/80 `DROP` rule.
* 🔍 Verify that the rule was correctly inserted.
* 📦 Capture and analyse the resulting blocked HTTP connection attempt.
* 📊 Correlate packet behaviour with `iptables` packet and byte counters.
* 🔬 Compare allowed and blocked TCP/HTTP traffic.
* ⚖️ Explain the difference between `DROP` and `REJECT`.
* ♻️ Remove the firewall rule safely and confirm restoration.

---

## 🧪 Laboratory Environment

| Component           | Configuration                           |
| ------------------- | --------------------------------------- |
| 🖥️ Server OS       | Kali Linux                              |
| 💻 Client           | Authorised laboratory client VM         |
| 🌐 Web server       | Apache2                                 |
| 🛡️ Firewall        | Linux `iptables` / UFW-managed rules    |
| 📡 Packet capture   | TShark / Wireshark                      |
| 🔎 HTTP client      | `curl`                                  |
| 🔌 Server interface | `eth0`                                  |
| 📍 Server IPv4      | `192.168.232.128/24`                    |
| 🌐 Server network   | `192.168.232.0/24`                      |
| 🚪 Default gateway  | `192.168.232.2`                         |
| 🌍 HTTP service     | TCP/80                                  |
| 🔒 Network type     | Isolated virtual laboratory environment |

> **Note:** The blocked client IP is recorded from the actual laboratory evidence and is not hard-coded here until the client-side baseline is confirmed.

---

## 📁 Repository Structure

```text
SBT-DF203-Lab6/
├── 📂 evidence/
│   ├── http_allowed.pcapng
│   └── http_blocked.pcapng
│
├── 📂 working/
│   └── analysis copies
│
├── 📂 exported/
│   └── exported analysis artefacts
│
├── 📂 reports/
│   ├── iptables_before.rules
│   ├── iptables_before.txt
│   ├── iptables_before_sha256.txt
│   ├── server_interfaces.txt
│   ├── server_routes.txt
│   ├── apache_listener.txt
│   ├── iptables_after_add.txt
│   ├── rule_verification.txt
│   ├── iptables_after_test.txt
│   ├── http_allowed_sha256.txt
│   ├── http_blocked_sha256.txt
│   ├── allowed_vs_blocked.tsv
│   └── iptables_restored.txt
│
├── 📂 screenshots/
│   ├── 01_original_firewall.png
│   ├── 02_network_apache.png
│   ├── 03_baseline_curl.png
│   ├── 04_allowed_capture.png
│   ├── 05_drop_rule.png
│   ├── 06_blocked_curl.png
│   ├── 07_blocked_capture.png
│   ├── 08_rule_counters.png
│   ├── 09_allowed_vs_blocked.png
│   └── 10_restored_access.png
│
├── 📂 scripts/
│   └── supporting scripts
│
└── 📄 README.md
```

---

## 🔐 Evidence Preservation

Before modifying the firewall, the existing firewall ruleset was exported and preserved.

```bash
sudo iptables-save | tee reports/iptables_before.rules

sudo iptables -L -n -v --line-numbers | \
tee reports/iptables_before.txt

sha256sum reports/iptables_before.rules | \
tee reports/iptables_before_sha256.txt
```

The original ruleset was preserved before introducing the experimental rule so that the initial firewall state could be demonstrated and compared with the post-test and restored states.

### 🔑 Original Ruleset SHA-256

```text
72623b5d5b59a58f57db29b9df34a514f14cc97f8b51bdb4d5e52597724ab290
```

The original firewall state showed an `INPUT` policy of `DROP` and existing UFW-managed rules. The existing configuration was preserved rather than flushed or replaced.

---

## 🌐 Server Network Baseline

The server's network interfaces were recorded using:

```bash
ip -br address | tee reports/server_interfaces.txt
```

The primary interface used for the laboratory network was:

```text
eth0
192.168.232.128/24
```

The routing table was recorded using:

```bash
ip route | tee reports/server_routes.txt
```

The relevant laboratory network route was:

```text
192.168.232.0/24 dev eth0
```

The default gateway was:

```text
192.168.232.2
```

These records establish the network identity and routing context of the protected server before the firewall experiment.

---

## 🖥️ Apache HTTP Listener Verification

The HTTP service was verified before firewall modification using:

```bash
sudo ss -lntp | grep ':80' | tee reports/apache_listener.txt
```

The output confirmed that Apache2 was listening on:

```text
*:80
```

This established that the HTTP service was available before the blocking rule was introduced.

The additional listeners on TCP port 8000 were associated with Docker and were outside the scope of this practical.

---

## ✅ Baseline HTTP Test

The laboratory client was used to establish normal HTTP connectivity before applying the experimental firewall rule.

The client IP address was recorded using:

```bash
ip -br address
```

The HTTP request was performed using:

```bash
curl -v --connect-timeout 5 \
http://192.168.232.128/firewall_lab.html \
2>&1 | tee baseline_curl.txt
```

The baseline result provides application-level evidence of the initial state of HTTP connectivity.

A successful baseline establishes the reference condition against which the subsequent blocked connection is compared.

---

## 📡 Allowed HTTP Packet Capture

After successful baseline verification, an allowed HTTP session was captured from the server.

```bash
sudo tshark -i IFACE \
-f 'host CLIENT_IP and tcp port 80' \
-a duration:30 \
-w evidence/http_allowed.pcapng
```

The resulting PCAPNG file was preserved as:

```text
evidence/http_allowed.pcapng
```

A SHA-256 hash was generated:

```bash
sha256sum evidence/http_allowed.pcapng | \
tee reports/http_allowed_sha256.txt
```

The allowed capture provides packet-level evidence of the normal TCP and HTTP exchange.

The expected successful sequence is:

```text
Client SYN
    ↓
Server SYN-ACK
    ↓
Client ACK
    ↓
HTTP GET
    ↓
HTTP Response
```

The final report documents the actual packet evidence observed in the capture.

---

## 🚫 Firewall DROP Rule

After preserving the original configuration and capturing the allowed baseline, a narrowly scoped firewall rule was introduced for the designated blocked laboratory client.

```bash
sudo iptables -I INPUT 1 \
-s BLOCKED_CLIENT_IP \
-p tcp \
--dport 80 \
-j DROP
```

The rule is restricted by:

* 🎯 Source IP address
* 🔗 TCP protocol
* 🚪 Destination port 80

This limits the experiment to the intended HTTP traffic.

The rule was verified using:

```bash
sudo iptables -L INPUT -n -v --line-numbers \
| tee reports/iptables_after_add.txt
```

and:

```bash
sudo iptables -C INPUT \
-s BLOCKED_CLIENT_IP \
-p tcp \
--dport 80
```

Successful verification was recorded in:

```text
reports/rule_verification.txt
```

---

## ⛔ Blocked HTTP Test

With the source-specific `DROP` rule active, an HTTP request was generated from the designated blocked client:

```bash
curl -v --connect-timeout 10 \
http://192.168.232.128/firewall_lab.html
```

The resulting client behaviour was preserved as evidence.

Because `DROP` silently discards matching packets rather than actively notifying the client, the expected TCP behaviour is that the client may send a SYN without receiving the expected SYN-ACK. TCP may subsequently retransmit the SYN before the connection attempt eventually times out.

The actual result is documented using the captured evidence rather than assumed from the expected behaviour.

---

## 📦 Blocked Traffic Capture

The blocked connection attempt was captured using TShark:

```bash
sudo tshark -i IFACE \
-f 'host BLOCKED_CLIENT_IP and tcp port 80' \
-a duration:35 \
-w evidence/http_blocked.pcapng
```

The resulting evidence file was:

```text
evidence/http_blocked.pcapng
```

Its integrity was recorded using:

```bash
sha256sum evidence/http_blocked.pcapng | \
tee reports/http_blocked_sha256.txt
```

The blocked capture is compared with the allowed capture to identify differences in TCP connection establishment and HTTP application activity.

---

## 📊 Firewall Counter Correlation

After the blocked HTTP attempt, the INPUT chain was examined again:

```bash
sudo iptables -L INPUT -n -v --line-numbers \
| tee reports/iptables_after_test.txt
```

The packet and byte counters associated with the source-specific DROP rule provide corroborating evidence that traffic matching the rule was processed by the firewall.

The investigation therefore correlates three evidence sources:

```text
💻 Client behaviour
        +
📡 Packet capture
        +
🛡️ iptables rule counters
        ↓
🔎 Correlated forensic finding
```

An increase in the DROP rule's packet counter supports the conclusion that packets matching the rule were processed by that firewall rule.

---

## 🔬 Allowed vs Blocked Traffic

The allowed and blocked captures were examined using TShark.

The analysis focuses on:

* 🔵 Client SYN packets
* 🟢 Server SYN-ACK packets
* 🔗 TCP ACK packets
* 📤 HTTP GET requests
* 📥 HTTP responses
* 🔁 TCP retransmissions
* 🤝 Connection completion
* ⏱️ Connection timeout behaviour

The following command was used:

```bash
for PCAP in evidence/http_allowed.pcapng evidence/http_blocked.pcapng
do
    echo "===== $PCAP ====="

    tshark -r "$PCAP" \
    -Y 'tcp.flags.syn==1 || http.request || http.response || tcp.analysis.retransmission' \
    -T fields \
    -e frame.number \
    -e frame.time_relative \
    -e ip.src \
    -e tcp.srcport \
    -e ip.dst \
    -e tcp.dstport \
    -e tcp.flags \
    -e tcp.analysis.retransmission \
    -e http.request.uri \
    -e http.response.code
done | tee reports/allowed_vs_blocked.tsv
```

### 📋 Comparison Matrix

| Indicator                   | Allowed Session  | Blocked Session  |
| --------------------------- | ---------------- | ---------------- |
| 🔵 Client SYN visible       | To be documented | To be documented |
| 🟢 Server SYN-ACK visible   | To be documented | To be documented |
| 🤝 TCP handshake completed  | To be documented | To be documented |
| 📤 HTTP GET visible         | To be documented | To be documented |
| 📥 HTTP response visible    | To be documented | To be documented |
| 🔁 Retransmissions          | To be documented | To be documented |
| 🛡️ Firewall counter change | N/A              | To be documented |
| 💻 Client result            | To be documented | To be documented |

---

## ⚖️ DROP vs REJECT

A `DROP` rule silently discards matching traffic. For a TCP connection attempt, this can result in the client repeatedly retransmitting connection-establishment packets because the expected response is not received.

A `REJECT` rule behaves differently because it actively returns an error response appropriate to the traffic type. For TCP traffic, this may involve a TCP reset. Consequently, a client normally receives an explicit indication that the connection was refused rather than waiting through silent packet loss.

From a forensic perspective, this distinction is useful because packet captures can provide evidence about whether traffic was silently discarded or actively rejected.

The laboratory test uses `DROP`, so the report focuses on the actual evidence generated by the `DROP` condition.

---

## 🕵️ Distinguishing Firewall Blocking from Service Failure

An HTTP connection failure does not, by itself, prove that a firewall blocked the traffic.

Other possible explanations include:

* ❌ Apache being stopped.
* ❌ No process listening on TCP/80.
* ❌ Incorrect server IP address.
* ❌ Network routing failure.
* ❌ Client-side connectivity problems.
* ❌ Another firewall rule affecting the traffic.
* ❌ Virtual network configuration problems.

For this reason, the investigation establishes Apache's listening state before the firewall change, preserves the original firewall ruleset, captures network traffic, and examines firewall rule counters.

The combination of these evidence sources provides a stronger basis for interpreting the blocked connection than the client timeout alone.

---

## ♻️ Firewall Rule Removal and Restoration

After the required blocked-traffic evidence was collected, the exact experimental rule was removed.

```bash
sudo iptables -D INPUT \
-s BLOCKED_CLIENT_IP \
-p tcp \
--dport 80 \
-j DROP
```

The rule's removal was checked using:

```bash
sudo iptables -C INPUT \
-s BLOCKED_CLIENT_IP \
-p tcp \
--dport 80 \
-j DROP || echo 'Rule successfully removed'
```

The restored INPUT chain was preserved using:

```bash
sudo iptables -L INPUT -n -v --line-numbers \
| tee reports/iptables_restored.txt
```

HTTP access from the previously blocked client was then tested again:

```bash
curl -v --connect-timeout 5 \
http://192.168.232.128/firewall_lab.html
```

The restoration test provides evidence that the experimental firewall modification was removed and HTTP connectivity could be tested again under the restored firewall state.

---

## 🔐 Evidence Integrity

SHA-256 hashes are maintained for the principal evidence files.

```text
reports/iptables_before_sha256.txt
reports/http_allowed_sha256.txt
reports/http_blocked_sha256.txt
```

The original firewall ruleset was hashed before modification.

The PCAPNG files were hashed after their respective captures were completed.

Original evidence files should not be modified after hashing. Analysis should be performed against working copies where appropriate.

---

## 📸 Screenshot Evidence

The screenshots directory contains numbered evidence corresponding to the major laboratory checkpoints.

| Screenshot                      | Evidence                                                  |
| ------------------------------- | --------------------------------------------------------- |
| 🖼️ `01_original_firewall.png`  | Original iptables ruleset and SHA-256 evidence            |
| 🖼️ `02_network_apache.png`     | Server network configuration, routing and Apache listener |
| 🖼️ `03_baseline_curl.png`      | Successful baseline HTTP request                          |
| 🖼️ `04_allowed_capture.png`    | Allowed TCP/HTTP packet capture                           |
| 🖼️ `05_drop_rule.png`          | Inserted and verified DROP rule                           |
| 🖼️ `06_blocked_curl.png`       | Blocked HTTP attempt and client behaviour                 |
| 🖼️ `07_blocked_capture.png`    | Blocked TCP traffic and retransmission evidence           |
| 🖼️ `08_rule_counters.png`      | Firewall packet/byte counters after blocked test          |
| 🖼️ `09_allowed_vs_blocked.png` | Comparative packet analysis                               |
| 🖼️ `10_restored_access.png`    | Rule removal and restored HTTP access                     |

Screenshots are numbered so that each figure can be referenced directly from the final forensic report.

---

## 🔎 Forensic Evidence Model

The practical uses a layered evidence model:

```text
                 🛡️ FIREWALL CONFIGURATION
                          │
                          ▼
                  iptables rule/counters
                          │
                          │
💻 CLIENT ───────────► 🖥️ SERVER
   │                       │
   │                       │
   ▼                       ▼
curl behaviour         Packet capture
   │                       │
   └───────────┬───────────┘
               ▼
        🔎 Correlated Finding
```

The purpose of the correlation is to avoid relying on a single observation. The client result indicates application-level behaviour, the PCAP provides network-level evidence, and the firewall counters provide host-level evidence of rule matching.

---

## 📝 Conclusion

This laboratory demonstrates a controlled forensic examination of host-based firewall behaviour. The investigation begins by preserving the original firewall configuration and establishing the server's network and Apache service state. An allowed HTTP baseline is then captured before a source-specific TCP/80 `DROP` rule is introduced.

The blocked condition is evaluated using client behaviour, packet capture and firewall counters. The allowed and blocked sessions are compared to identify differences in TCP connection establishment, HTTP exchange and retransmission behaviour. Finally, the experimental rule is removed and HTTP access is tested again to demonstrate restoration.

All observations in the final report are based on the commands, packet captures, firewall outputs and screenshots generated during the authorised laboratory exercise.

---

## 👤 Author

**Name:** Kafayat Animashawun
**Course:** SBT-DF203 — Basic Networking Skills for Digital Forensics
**Practical:** Lab 6 — Firewall Traffic Control and Forensic Verification
**Environment:** Authorised isolated virtual laboratory
**Date:** September 2026

---

## 🎓 Academic Integrity

All commands, observations, screenshots, packet captures, hashes and conclusions contained in this repository are generated from my own execution of the practical laboratory in an authorised environment.

No third-party, production or public network was targeted.

Original evidence was preserved and firewall modifications were removed after the required evidence was collected.
