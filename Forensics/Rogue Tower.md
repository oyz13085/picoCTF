# picoCTF 2025 — Rogue Tower

**Category:** Forensics  
**Difficulty:** Easy  
**Author:** Prince Niyonshuti N.

---

## Description

> A suspicious cell tower has been detected in the network. Analyze the captured network traffic to identify the rogue tower, find the compromised device, and recover the exfiltrated flag.

---

## Tools Used

- Wireshark
- CyberChef

---

## Hints

1. Look for unauthorized test network broadcasts on UDP port 55000
2. Find the device that connected to the rogue tower by checking HTTP User-Agent headers
3. The encryption key is derived from the victim device's IMSI
4. The exfiltrated data is split across multiple HTTP POST requests

---

## Background — What is a Rogue Cell Tower?

A rogue cell tower (also called an IMSI catcher or stingray) is a fake cell tower that tricks nearby devices into connecting to it instead of a legitimate carrier tower. Once connected, it can:

- Harvest device identifiers (IMSI numbers)
- Intercept communications
- Exfiltrate data from compromised devices

In this challenge, the rogue tower tricks devices into registering with a C2 server and then exfiltrates data from the most compromised device.

---

## Solution

### Step 1 — Initial recon

Opening the PCAP in Wireshark reveals several types of traffic:

- UDP broadcasts on port 55000
- DNS queries to `8.8.8.8` for `device-XXXXXXXXX.network.com` hostnames
- HTTP `GET /api/register` requests to `198.51.100.140`
- HTTP `POST /upload` requests to `198.51.100.58`

The pattern of multiple internal IPs all doing DNS lookup then `/api/register` is a classic **C2 beaconing / registration** pattern — infected devices checking in with the rogue tower's server.

---

### Step 2 — Finding the rogue tower (UDP port 55000)

Filter for UDP broadcasts:
```
udp.port == 55000
```

Right-click a packet → **Follow → UDP Stream**, iterate through streams. In Stream 8, an unauthorized broadcast message appears revealing the rogue cell tower's **Cell ID: 90461**.

This is the rogue tower announcing itself on the network.

---

### Step 3 — Finding the compromised device

Filter for HTTP traffic and check User-Agent headers:
```
http
```

Right-click → **Follow → HTTP Stream**, search for the Cell ID `90461`. The stream reveals which device connected to the rogue tower.

The compromised device is: **`10.100.50.122`**

This is confirmed by filtering:
```
ip.src == 10.100.50.122
```

This device is the only one sending `POST /upload` requests — actively exfiltrating data — while other devices only sent `/api/register`.

---

### Step 4 — Finding the IMSI

Filter for DNS queries from the compromised device:
```
ip.src == 10.100.50.122 && dns
```

The DNS query hostname reveals the device's IMSI:
```
device-310410308555787.network.com
         ^^^^^^^^^^^^^^
         IMSI = 310410308555787
```

---

### Step 5 — Deriving the encryption key

The IMSI follows standard cellular network structure:

```
310410 | 08555787
^^^^^^   ^^^^^^^^
MCC+MNC  MSIN (unique per device — this is the key)
(constant across all devices)
```

All devices in this capture share the `310410` prefix (US carrier network code). The **varying suffix** is the actual encryption key:

```
XOR key = 08555787
```

---

### Step 6 — Extracting the exfiltrated data

Filter for POST requests from the compromised device:
```
http.request.method == "POST" && ip.src == 10.100.50.122
```

Use **File → Export Objects → HTTP** to see all uploads:

| Packet | Size |
|--------|------|
| 17 | 9 bytes |
| 18 | 9 bytes |
| 19 | 9 bytes |
| 20 | 9 bytes |
| 21 | 9 bytes |
| 22 | 3 bytes |

The data is split across 6 POST requests. Concatenating all chunks in packet order gives the full Base64 encoded payload:

```
QFFWWnZjfkxCCFJABmhbBFxUakEFQAtFb1xXVgEHAAQBRQ==
```

---

### Step 7 — Decrypting the flag (CyberChef)

Using CyberChef with the following recipe:

```
1. From Base64
2. XOR
   Key: 08555787
   Key format: UTF8
   Scheme: Standard
```

Input:
```
QFFWWnZjfkxCCFJABmhbBFxUakEFQAtFb1xXVgEHAAQBRQ==
```

---

## Flag

```
picoCTF{r0gu3_c3ll_t0w3r_dbc40831}
```

---

## Key Takeaways

- **Rogue cell towers** harvest IMSI numbers from devices that connect to them — a real-world attack used by law enforcement and adversaries alike
- **IMSI structure**: MCC (3 digits) + MNC (2-3 digits) + MSIN (unique subscriber number). The key was only the MSIN portion, not the full IMSI
- **C2 beaconing pattern**: multiple devices doing DNS lookup then HTTP registration to the same external IP is a strong indicator of compromise
- **Data exfiltration detection**: `POST /upload` to an external IP in small repeated chunks is a classic low-and-slow exfil pattern
- **XOR encryption** with a device identifier as the key is a common lightweight obfuscation technique in malware
- Always re-examine assumptions — the hint said "derived from IMSI", not "is the IMSI". The key was only the variable portion

---

## Attack Flow Summary

```
Rogue tower broadcasts on UDP 55000
        ↓
Devices connect and register via GET /api/register → 198.51.100.140
        ↓
Compromised device (10.100.50.122) exfiltrates data
via POST /upload → 198.51.100.58
        ↓
Data is XOR encrypted with IMSI suffix (08555787)
and Base64 encoded, split across 6 POST requests
        ↓
Reconstructed and decrypted → picoCTF{r0gu3_c3ll_t0w3r_dbc40831}
```
