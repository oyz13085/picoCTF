# picoCTF 2023 — PcapPoisoning

**Category:** Forensics  
**Difficulty:** Medium  
**Author:** Mubarak Mikail

---

## Description

> How about some hide and seek heh? Download this file and find the flag.

---

## Tools Used

- Wireshark

---

## Solution

### Step 1 — Statistics → Conversations

The first step with any unknown PCAP is to get a bird's eye view of all traffic using:

```
Statistics → Conversations → TCP tab
```

This immediately revealed something suspicious:

| Address A | Address B | Bytes A→B | Bytes B→A |
|-----------|-----------|-----------|-----------|
| 172.16.0.2 | 10.253.0.6 | **156 bytes** | 0 bytes |
| 172.16.0.2 | 192.168.1.10 | 40 bytes | 0 bytes |
| 172.16.0.2 | 192.168.1.10 | 40 bytes | 0 bytes |
| ... (1000+ identical rows) | | | |

`172.16.0.2` was the only IP that actually **transferred meaningful data** (156 bytes). All other rows were identical 40-byte SYN packets to sequential ports on `192.168.1.10` — pure noise designed to bury the real traffic.

<img width="1431" height="625" alt="image" src="https://github.com/user-attachments/assets/8ac1aa57-19df-4b7d-8c9f-9a19e054d123" />


---

### Step 2 — Identifying the noise

The 1000+ TCP streams to `192.168.1.10` on ports 0, 1, 2, 3... were all:
- SYN only (no SYN-ACK, no data)
- Exactly 40 bytes
- 0 bytes returned

This is a **SYN flood used as camouflage** — the attacker generated hundreds of fake connection attempts to make analysts dismiss the traffic as noise and miss the single real stream.

---

### Step 3 — Filtering for the suspicious IP

Applied filter:
```
ip.addr == 172.16.0.2
```

This revealed two key packets:

**Packet 4 — FTP credentials in plaintext:**
```
Request: username root    password toor
```
The attacker logged in via FTP using default credentials — this is the initial access vector.

**Packet 507 — TCP Retransmission to 10.253.0.6:**
Inspecting the hex panel on the right side revealed the flag hidden directly in the packet data.

---

### Step 4 — Reading the flag from hex

The flag was visible in the raw hex/ASCII panel of packet 507:

```
picoTCF{
P64P_4N4 L7S1S_SU
55355FUL _5b6a606
1}
```

<img width="1568" height="782" alt="image" src="https://github.com/user-attachments/assets/a2079953-19a4-4a7d-b843-88b3edcfa2ea" />


No decoding or decryption needed — the flag was stored in plaintext inside the TCP packet payload, hidden among thousands of decoy packets.

---

## Flag

```
picoCTF{P64P_4N4L7S1S_5U55355FUL_5b6a6061}
```

---

## Key Takeaways

- **Statistics → Conversations is always your first step** in an unknown PCAP — it shows the full communication map at a glance and lets you spot anomalies immediately
- **Volume is not suspicion** — the 1000+ SYN packets were noise. The single stream with actual bytes transferred was the real signal. Always look at `Bytes A→B` not just packet count
- **SYN flood as camouflage** — attackers deliberately generate junk traffic to overwhelm analysts and hide real activity. Sorting by bytes transferred cuts through this instantly
- **FTP sends credentials in plaintext** — `username` and `password` visible in packet 4 shows why FTP is considered insecure and replaced by SFTP/FTPS in modern environments
- **Flags can hide anywhere** — not just in HTTP bodies or application layer data. Always check raw hex panels of suspicious packets

---

## Attack Flow Summary

```
Attacker connects via FTP → plaintext credentials (root:toor) visible
        ↓
Real data exfiltrated to 10.253.0.6 (156 bytes)
        ↓
1000+ SYN flood packets to 192.168.1.10 generated as decoy noise
        ↓
Flag hidden in TCP packet payload of the single real stream
        ↓
Found by: Statistics → Conversations → filter by bytes transferred
```
