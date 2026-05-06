# picoCTF - Ph4nt0m-1ntrud3r

**Category:** Forensics
**Difficulty:** Easy
**Author:** Prince Niyonshuti N.

---

## Description

> A digital ghost has breached my defenses, and my sensitive data has been stolen! Your mission is to uncover how this phantom intruder infiltrated my system and retrieve the hidden flag. The attacker has cleverly concealed his moves in a well timely manner. Dive into the network traffic, apply the right filters and show off your forensic prowess and unmask the digital intruder!

---

## Tools Used

- Wireshark

---

## Solution

### Step 1 — Initial recon

After downloading the `.pcap` file and opening it in Wireshark, the traffic was almost entirely **TCP SYN retransmissions** from `192.168.0.2` to `192.168.1.2` on port 80. No full TCP handshake completes — which is unusual.

The challenge description hinted that *"the attacker concealed his moves in a well timely manner"* — this suggested the data was hidden in the **packet structure itself**, not in HTTP payloads.

<img width="1189" height="408" alt="image" src="https://github.com/user-attachments/assets/90a2d06e-fbea-4d02-9c7d-b50e61256743" />

### Step 2 — Spotting the anomaly

Looking closely at packet lengths, most SYN packets had `Len=8`. But a small number had `Len=12` or `Len=4` — standing out from the uniform noise.

This was the key observation: **data was being smuggled inside TCP SYN packet payloads**, which is abnormal since SYN packets typically carry no data. This is a form of **covert channel / timing-based exfiltration**.

Filtered for only the anomalous packets:
- Packets with `Len=12`
- Packets with `Len=4`

<img width="1161" height="169" alt="image" src="https://github.com/user-attachments/assets/2d04808b-adee-4086-a54e-73509477b66d" />

### Step 3 — Extracting and decoding

Inspecting the packet bytes panel, the payload data appeared to be **Base64-encoded**. Decoding each packet's payload:

| Packet | Len | Base64 payload       | Decoded       |
|--------|-----|----------------------|---------------|
| 8      | 4   | `fQ==`               | `}`           |
| 9      | 4   | `cGljb0NURg==`       | `picoCTF`     |
| 13     | 12  | `ZTEwZTgzOQ==`       | `e10e839`     |
| 15     | 12  | `XzM0c3lfdA==`       | `_34sy_t`     |
| 17     | 12  | `bnRfdGg0dA==`       | `nt_th4t`     |
| 20     | 12  | `YmhfNHJfOA==`       | `bh_4r_8`     |
| 21     | 12  | `ezF0X3c0cw==`       | `{1t_w4s`     |

### Step 4 — Reassembly

The fragments were scattered across non-sequential packets. Sorting by packet number and reassembling in logical order:

```
picoCTF  +  {1t_w4s  +  nt_th4t  +  _34sy_t  +  bh_4r_8  +  e10e839  +  }
```

---

## Flag

```
picoCTF{1t_w4snt_th4t_34sy_tbh_4r_8e10e839}
```

---

## Key Takeaways

- **Covert channel awareness** — data can be hidden in packet fields outside of normal payload (headers, lengths, SYN data)
- **SYN packets carrying data** is a red flag in normal traffic; attackers exploit this because some IDS tools ignore pre-handshake payloads
- **Uniform traffic with outliers** — when most packets look identical, the anomalies are where the data hides
- Always decode suspicious byte patterns as Base64 when they contain characters in the `[A-Za-z0-9+/=]` range
