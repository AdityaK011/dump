---
title: "Cellular Networks and IP Addresses"
---

Every internet connection on a mobile device begins with the phone attaching to a cellular network -- a process involving radio synchronization, subscriber authentication, and tunnel establishment that most users never see. This note traces how a phone acquires an IP address, the layered identity system that protects subscriber privacy over the air, the GTP tunneling mechanism that keeps IP addresses stable across cell tower handoffs, and the specific conditions under which IP addresses actually change.

## How your phone gets -- and keeps -- an IP address

Before any privacy relay enters the picture, a phone must attach to a cellular network. This process involves a cascade of identifiers, tunnels, and anchor points that fundamentally determine when IP addresses change.

### From radio waves to the core network

When a phone powers on, it performs **cell search**: scanning for Primary and Secondary Synchronization Signals (PSS/SSS) broadcast by nearby cell towers, then decoding the Master Information Block (MIB) and System Information Block 1 (SIB1) to learn the network's identity and access parameters. It then executes a **4-step Random Access (RACH) procedure** -- transmitting a random preamble on the Physical Random Access Channel, receiving a timing advance correction and temporary radio identifier (TC-RNTI), sending an RRC connection request, and completing contention resolution. The result is an RRC (Radio Resource Control) connection with the base station: an eNodeB in 4G LTE or a gNB in 5G NR.

The base station then forwards the phone's NAS (Non-Access Stratum) registration message into the **core network**. In 4G, this means the eNodeB sends an Attach Request to the MME (Mobility Management Entity) via the S1-MME interface, which authenticates the subscriber through the HSS (Home Subscriber Server) using the Diameter protocol. In 5G, the gNB sends a Registration Request to the AMF (Access and Mobility Management Function) via the N2 interface, with authentication flowing through the AUSF and UDM. After authentication, the network establishes a data session -- a **PDN connection** in 4G or a **PDU session** in 5G -- and assigns the phone an IP address.

### Three identifiers and what they protect

The cellular network uses a layered identity system designed to minimize how often the phone's permanent identity is exposed over the air:

**IMSI / SUPI** is the permanent subscriber identity -- up to 15 digits structured as MCC (country) + MNC (operator) + MSIN (subscriber). In 4G, the IMSI is transmitted in **plaintext** during the initial attach, which is exactly what IMSI-catchers (Stingray devices) exploit. 5G fixes this: the IMSI is renamed SUPI (Subscription Permanent Identifier) and is **never sent in cleartext**. Instead, the phone transmits a SUCI (Subscription Concealed Identifier), encrypting the MSIN portion using ECIES with the home network's public key. Only the MCC+MNC are visible, for routing purposes.

**TMSI / GUTI** is the temporary identity used for all signaling after initial attach. In 4G, the GUTI (Globally Unique Temporary Identifier) consists of a GUMMEI (which identifies the specific MME) plus a 32-bit M-TMSI. In 5G, the 5G-GUTI uses a GUAMI plus a 32-bit 5G-TMSI, with the standard mandating unpredictable reallocation after each network-triggered service request to resist tracking. The GUTI may or may not change during a handover -- it depends on whether the phone enters a new Tracking Area.

**RNTI** (Radio Network Temporary Identifier) operates only at the radio layer, identifying the phone's scheduling allocation within a single cell. The **C-RNTI** (Cell-RNTI) is a 16-bit identifier assigned by the base station and used to scramble downlink control information for that specific phone. Critically, **C-RNTI always changes during a handover** because it has meaning only within a single cell.

### GTP tunnels: the reason your IP survives tower hops

The mechanism that keeps your IP address stable across cell towers is **GTP (GPRS Tunnelling Protocol)**. User traffic flows through GTP-U tunnels (UDP port 2152) that encapsulate the phone's original IP packets inside outer IP/UDP/GTP headers. In 4G, the data path is:

```
UE -> [air] -> eNodeB -> [S1-U GTP tunnel] -> S-GW -> [S5 GTP tunnel] -> P-GW -> Internet
```

Each tunnel segment uses a **TEID (Tunnel Endpoint Identifier)** -- a 32-bit dynamically allocated value that identifies the tunnel at the receiving node. Separate TEIDs exist for uplink and downlink on each interface. The phone's IP address is the **inner** IP inside the GTP encapsulation; the **outer** IPs belong to the network nodes. During a handover, only the outer tunnel endpoints change -- the target base station allocates a new S1-U TEID, the MME sends a Modify Bearer Request to the S-GW, and the S-GW updates its forwarding table. The **P-GW remains completely unaware** of the handover and continues anchoring the phone's IP address.

In 5G, the architecture is simplified: the S-GW is eliminated, and the **UPF (User Plane Function)** combines both gateway roles. Traffic flows UE -> gNB -> [N3 GTP-U tunnel] -> UPF -> Internet. The SMF (Session Management Function) programs the UPF via PFCP (Packet Forwarding Control Protocol) with Packet Detection Rules and Forwarding Action Rules. During handover, the SMF updates these rules to point at the new gNB -- but the UPF's N6 interface to the internet is untouched.

### When IP addresses actually change

Your IP stays stable during tower handoffs because the PDN connection (4G) or PDU session (5G) persists. IP addresses change when that session is disrupted:

- **Wi-Fi to cellular transitions** assign completely different IPs because Wi-Fi uses local DHCP while cellular IPs come from the carrier's P-GW/UPF pool (exception: 3GPP's ePDG and N3IWF mechanisms can maintain the same session across access types using IPSec tunnels)
- **Airplane mode toggling** releases the PDN/PDU session entirely, yielding a new IP on reconnect
- **International roaming** with home-routed traffic tunnels data back to the home P-GW via the S8 interface, preserving the home-country IP; local breakout instead routes through the visited network's P-GW, producing a new IP
- **5G SSC Mode 2 and 3** intentionally change IPs: Mode 2 ("break-before-make") releases the session for UPF reselection; Mode 3 ("make-before-break") establishes a new UPF connection with a new IPv6 prefix before releasing the old one

Most carriers also deploy **Carrier-Grade NAT (CGNAT)**, assigning private IPv4 addresses (10.x.x.x or 100.64.x.x) that map to shared public IPs. The public IP can change independently of the phone's private address.

---

## Related notes

- [[notes/Networking/quic-and-masque|QUIC and MASQUE]] -- QUIC's Connection IDs allow connections to survive the IP changes described here
- [[notes/Networking/icloud-private-relay-architecture|iCloud Private Relay Architecture]] -- how Private Relay builds on cellular IP assignment and handles IP transitions
- [[notes/Networking/dns-privacy-and-oblivious-protocols|DNS Privacy and Oblivious Protocols]] -- DNS resolution that happens after the phone obtains an IP address
