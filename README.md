# 🌐 OSPF Neighbor States 1-7 — The Full Adjacency Formation Process

![Cisco](https://img.shields.io/badge/Cisco-Packet%20Tracer-1BA0D7?style=for-the-badge&logo=cisco&logoColor=white)
![OSPF](https://img.shields.io/badge/Protocol-OSPF-teal?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Completed-brightgreen?style=for-the-badge)
![Level](https://img.shields.io/badge/Level-CCNA-blue?style=for-the-badge)
![Topic](https://img.shields.io/badge/Focus-Neighbor%20State%20Machine-purple?style=for-the-badge)

> A hands-on Cisco Packet Tracer lab capturing **all 7 OSPF neighbor states** — Down, Init, 2-Way, ExStart, Exchange, Loading, and Full — with real, simultaneous debug output from both sides of the adjacency. Not the textbook definitions: the actual packets and log lines behind every transition.

---

## 📖 Overview

Most CCNA material lists the 7 OSPF neighbor states by name but rarely shows what they actually look like on the wire or in the CLI. This lab builds the simplest possible topology — two directly connected routers — specifically to isolate and capture the complete adjacency formation process from both sides at once, with real debug-level evidence for every single state transition.

**Goals of this lab:**
- Capture live evidence of all 7 OSPF neighbor states as they happen
- Identify exactly which packet or event triggers each state transition
- Observe the Master/Slave negotiation (ExStart) and DR/BDR election (2-Way) as they occur within the state machine
- Confirm final adjacency using the standard `%OSPF-5-ADJCHG` syslog message

---

## 🗺️ Network Topology

```
   10.1.1.1/32                                    20.1.1.1/32
   (Loopback)                                      (Loopback)

[PC] 192.168.1.2 ─┐                                          ┌─ 192.168.2.2 [PC]
                  ├─[   R1   ]════ 1.1.1.0/8 ════[   R2   ]──┤
                  ┘   2901        Gi0/0: .1  Gi0/0: .2  2901   ┘
              g0/1 192.168.1.1/24                    g0/1 192.168.2.1/24

Routing Protocol: OSPF Area 0 — a simple point-to-point-style link between two routers,
purpose-built to isolate and observe the full adjacency formation process
```

### R1 — `router ospf 1`
```
network 1.1.1.0 0.0.0.255 area 0
network 10.1.1.1 0.0.0.0 area 0
network 192.168.1.0 0.0.0.255 area 0
```

### R2 — `router ospf 1`
```
network 1.1.1.0 0.0.0.255 area 0
network 20.1.1.1 0.0.0.0 area 0
network 192.168.2.0 0.0.0.255 area 0
```

---

## 🔢 The 7 OSPF Neighbor States — Quick Reference

| # | State       | What happens here                                                      |
|---|-------------|---------------------------------------------------------------------------|
| 1 | **Down**        | No Hello packets have been received from the neighbor yet             |
| 2 | **Init**        | A Hello packet has been received, but it doesn't yet list this router — one-way communication |
| 3 | **2-Way**       | Both routers see each other in their Hello packets — the neighbor table is built, and **DR/BDR election** happens here on multi-access networks |
| 4 | **ExStart**     | Routers negotiate **Master/Slave** roles and an initial DBD sequence number |
| 5 | **Exchange**    | Routers exchange **Database Description (DBD)** packets — summaries of their link-state databases |
| 6 | **Loading**     | Routers send **Link State Request (LSR)** packets for any LSAs they're missing, and receive **Link State Updates (LSU)** in return |
| 7 | **Full**        | Databases are fully synchronized — the adjacency is complete and operational |

---

## 🔍 Live Evidence — Captured From Both Sides

### 1️⃣ Down State — R1
```
00:31:51: OSPF: No full nbrs to build Net Lsa for interface GigabitEthernet0/0
```
> Before any Hello exchange completes, R1 has no full neighbors on this interface — this is the **Down** state.

### 2️⃣ Init State — Hello Packet Received
```
00:31:56: OSPF: Rcv DBD from 20.1.1.1 on GigabitEthernet0/0 ... state INIT
```
> R1 receives a packet from R2 (`20.1.1.1`) while still in **Init** — R1 knows R2 exists but two-way communication isn't confirmed yet.

### 3️⃣ 2-Way State — DR/BDR Election & Neighbor Table Build
**From R1's perspective:**
```
00:31:56: OSPF: DR/BDR election on GigabitEthernet0/0
00:31:56: OSPF: Elect BDR 0.0.0.0
00:31:56: OSPF: Elect DR 20.1.1.1
00:31:56: OSPF: Elect BDR 10.1.1.1
00:31:56: OSPF: Elect DR 20.1.1.1
00:31:56:       DR: 20.1.1.1 (Id)   BDR: 10.1.1.1 (Id)
```
**From R2's perspective:**
```
00:31:55: OSPF: DR/BDR election on GigabitEthernet0/0
00:31:55: OSPF: Elect BDR 0.0.0.0
00:31:55: OSPF: Elect DR 20.1.1.1
00:31:55:       DR: 20.1.1.1 (Id)   BDR: none
```
> **Confirmed:** DR/BDR election runs during the **2-Way** state, immediately after both routers recognize each other and build their neighbor table — R2 (`20.1.1.1`) becomes DR, R1 (`10.1.1.1`) becomes BDR.

### 4️⃣ ExStart State — Master/Slave Negotiation (captured from both sides simultaneously)
**R1's log:**
```
00:31:56: OSPF: Send DBD to 20.1.1.1 ... flag 0x7 ...
00:31:56: OSPF: NBR Negotiation Done. We are the SLAVE
```
**R2's log — same adjacency, opposite role:**
```
00:31:55: OSPF: Rcv DBD from 10.1.1.1 ... state EXSTART
00:31:55: OSPF: First DBD and we are not SLAVE
00:31:55: OSPF: NBR Negotiation Done. We are the MASTER
```
> **This is the clearest possible proof of the ExStart negotiation:** for the exact same adjacency, R1 logs itself as **SLAVE** while R2 simultaneously logs itself as **MASTER**. The router with the higher Router ID (R2 = `20.1.1.1` > R1 = `10.1.1.1`) always wins Master status — the Master then controls the DBD sequence numbering for the rest of the exchange.

### 5️⃣ Exchange State — Database Description Packet Exchange
```
00:31:56: OSPF: Rcv DBD from 20.1.1.1 ... state EXCHANGE
00:31:56: OSPF: Send DBD to 20.1.1.1 ...
00:31:56: OSPF: Rcv DBD from 20.1.1.1 ... state EXCHANGE
00:31:56: OSPF: Send DBD to 20.1.1.1 ...
```
> Both routers exchange **DBD packets** describing (not sending in full — just summarizing) the contents of their link-state databases, so each side can figure out what it's missing.

### 6️⃣ Loading State — Requesting and Building the Database
```
00:31:56: Exchange Done with 20.1.1.1 on GigabitEthernet0/0
00:31:56: OSPF: Database request to 20.1.1.1
00:31:56: OSPF: sent LS REQ packet to 1.1.1.2, length 12
00:31:56: OSPF: Send DBD to 20.1.1.1 ...
```
```
00:31:55: OSPF: Rcv DBD from 10.1.1.1 ... state LOADING
```
> Once the DBD summaries reveal missing LSAs, each router sends a **Link State Request (LSR)** for exactly what it needs and receives the full LSAs back — this is the step that actually **builds the link-state database table**.

### 7️⃣ Full State — Adjacency Complete
```
00:31:55: Synchronized with 10.1.1.1 on GigabitEthernet0/0, state FULL
00:31:55: %OSPF-5-ADJCHG: Process 1, Nbr 10.1.1.1 on GigabitEthernet0/0 from LOADING to FULL, Loading Done
```
> **Confirmed:** the databases are now fully synchronized, and the standard `%OSPF-5-ADJCHG` syslog message marks the transition from LOADING to **FULL** — the adjacency is complete and the routers can now exchange routing updates normally.

---

## 📊 State Transition Summary

| State     | Triggering Event                                    | Key Log Evidence                              |
|-----------|--------------------------------------------------------|--------------------------------------------------|
| Down      | No Hello received yet                                   | *(absence of activity / "No full nbrs")*         |
| Init      | A Hello packet is received (one-way)                    | `Rcv DBD ... state INIT`                          |
| 2-Way     | Both routers see each other; DR/BDR elected              | `Elect DR` / `Elect BDR` / `DR: ... BDR: ...`     |
| ExStart   | Master/Slave roles negotiated                            | `NBR Negotiation Done. We are the MASTER/SLAVE`   |
| Exchange  | DBD packets exchanged                                     | `... state EXCHANGE` / `Exchange Done`            |
| Loading   | LS Request/Update builds the database                    | `Database request` / `sent LS REQ packet`         |
| Full      | Databases fully synchronized                              | `%OSPF-5-ADJCHG ... from LOADING to FULL`         |

---

## 🎯 Key Learnings

- The 7 OSPF neighbor states form a strict, sequential state machine — a stuck adjacency almost always freezes at one specific state, and knowing what each state actually represents is the fastest way to diagnose *why*.
- **DR/BDR election happens during 2-Way**, immediately after the neighbor table is built — not before, and not during ExStart.
- **ExStart is exclusively about Master/Slave negotiation** — the router with the higher Router ID always becomes Master and controls DBD sequencing for the rest of the process.
- **Exchange** only trades *summaries* (DBD packets) of the link-state database, not the full LSAs themselves — the full data only moves during **Loading**, via explicit Link State Request/Update packets.
- The `%OSPF-5-ADJCHG` syslog message with `from LOADING to FULL, Loading Done` is the definitive, standard confirmation that an OSPF adjacency has fully formed.
- Capturing debug output from **both sides of the same adjacency at once** (as done here) is one of the most effective ways to truly understand a negotiation-based protocol — seeing R1 log "SLAVE" and R2 log "MASTER" for the identical event makes the concept unambiguous in a way textbook diagrams cannot.

---

## ✅ Outcomes

- Captured and fully documented real, simultaneous debug evidence for all 7 OSPF neighbor states
- Pinpointed exactly which log line or packet triggers each state transition
- Verified DR/BDR election timing and Master/Slave negotiation behavior with direct CLI evidence
- Built a precise, reference-quality troubleshooting guide for diagnosing stuck OSPF adjacencies
- Added the most detailed and exam-critical fundamentals lab yet to my Cisco OSPF portfolio

---

## 🛠️ Tools Used

- Cisco Packet Tracer
- Cisco IOS (OSPF)
- `debug ip ospf adj` (equivalent simulated debug output), `show ip ospf neighbor`

---

## 👤 Author

**Maaz Khan**
CCNA Certified | Network & NOC Engineer
📍 Lower Dir, KPK, Pakistan
🔗 [LinkedIn](https://www.linkedin.com/in/maazkhanms) · [GitHub](https://github.com/maazkhanms)

---

⭐ If you found this lab useful, consider starring the repo — more RIP, OSPF, EIGRP, IPv6, and routing-fundamentals labs coming in this series!
