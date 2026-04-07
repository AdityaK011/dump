---
title: "Summary: Cellular Networks and IP Addresses"
---

> **Full notes:** [[notes/Networking/cellular-networks-and-ip-addresses|Cellular Networks and IP Addresses -->]]

## Key Concepts

### Cell Attachment Sequence
- Phone scans for PSS/SSS signals, decodes MIB and SIB1 to learn network identity
- 4-step RACH procedure: random preamble, timing advance + TC-RNTI, RRC request, contention resolution
- Result: RRC connection to eNodeB (4G) or gNB (5G)
- NAS registration forwarded to core network for authentication

### Core Network Authentication
- **4G**: eNodeB -> MME (via S1-MME) -> HSS (Diameter protocol) -> PDN connection established -> IP assigned
- **5G**: gNB -> AMF (via N2) -> AUSF/UDM -> PDU session established -> IP assigned

### Three-Layer Identity System
- **IMSI/SUPI** (permanent): 15 digits = MCC + MNC + MSIN. 4G sends IMSI in plaintext (IMSI-catcher vulnerability). 5G encrypts MSIN using ECIES with home network public key, transmits as SUCI
- **TMSI/GUTI** (temporary): used for all signaling after attach. 4G = GUMMEI + 32-bit M-TMSI. 5G = GUAMI + 32-bit 5G-TMSI, reallocated unpredictably per service request
- **RNTI** (radio-layer): 16-bit C-RNTI, valid only within one cell, always changes on handover

### GTP Tunneling
- GTP-U tunnels over UDP port 2152 encapsulate user IP packets inside outer IP/UDP/GTP headers
- Each tunnel segment identified by 32-bit TEID (separate for uplink/downlink)
- During handover: outer tunnel endpoints change, inner IP stays the same
- P-GW (4G) / UPF (5G) is completely unaware of handovers

### When IPs Actually Change
- Wi-Fi <-> cellular transition (different address pools, unless ePDG/N3IWF used)
- Airplane mode toggle (releases PDN/PDU session entirely)
- International roaming with local breakout (home-routed preserves home IP via S8 tunnel)
- 5G SSC Mode 2 (break-before-make) and Mode 3 (make-before-break)
- CGNAT: public IP can change independently of private IP

## Quick Reference

```
4G Data Path:
UE -> [air] -> eNodeB -> [S1-U GTP] -> S-GW -> [S5 GTP] -> P-GW -> Internet

5G Data Path:
UE -> [air] -> gNB -> [N3 GTP-U] -> UPF -> Internet

Identity Layers:
+-----------+-------------+------------------+-------------------+
| Layer     | 4G Name     | 5G Name          | Scope             |
+-----------+-------------+------------------+-------------------+
| Permanent | IMSI        | SUPI (via SUCI)  | Global            |
| Temporary | GUTI        | 5G-GUTI          | Tracking Area     |
| Radio     | C-RNTI      | C-RNTI           | Single cell only  |
+-----------+-------------+------------------+-------------------+

IP Stability Rule:
  Handover between towers  -> IP STAYS (GTP re-anchored)
  Session torn down        -> IP CHANGES (new PDN/PDU)
```
