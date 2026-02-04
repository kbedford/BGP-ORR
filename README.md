# BGP-ORR

---

# BGP (ORR) vs Normal Route Reflection (RR)

This lab demonstrates how **BGP Optimal Route Reflection (ORR)** in Junos improves route-reflection behaviour compared to **traditional route reflection** when multiple egress/exit paths exist for the same prefix.

The key takeaway:

* **Normal RR:** the route reflector selects **one** best path (based on its own view) and reflects the **same** best path to all clients.
* **ORR:** the route reflector can reflect **different** best paths to different clients, based on the **client’s IGP shortest path** to each candidate egress.

---

## Topology

<img width="625" height="417" alt="image" src="https://github.com/user-attachments/assets/b2d91ee5-d508-4ba6-ab79-e217c36c0641" />


**Prefix under test:** `203.0.113.0/24`
**Candidate exits (BGP next-hops):** `10.255.0.2` (R2) and `10.255.0.3` (R3)

---

## Why ORR Matters

In large networks (ISP/backbone / DC fabrics), multiple egress points often advertise the same external prefix. With **normal route reflection**, clients may be forced to use a suboptimal exit because RR selects a single best path based on its own IGP location.

What is BGP ORR?

As defined by RFC 9107, BGP ORR, instead of sending the best-path based on RR point of view, optimal BGP path selection is based on a client’s perspective. The vRR  sends a specific best-path to a particular BGP client that is calculated based on the position of the router in the topology uses IGP cost as the measure.

With **ORR enabled**, the RR still receives multiple paths, but it selects the best path **per-client (or per-client-group)** by evaluating **IGP distance from the client** to each candidate exit (primary/backup). This reduces:

* unnecessary tromboning / suboptimal routing
* wasted bandwidth
* latency impact (clients can exit closer to their best egress)

---

## What I Captured

To prove the difference, I captured outputs in **two modes**:

1. **ORR enabled**
2. **Normal RR (ORR disabled)**

The comparison focuses on:

* What RR advertises to **R1**
* What **R1** installs
* What **R4** installs
* How the RR sees ORR IGP metrics

---

# Mode 1 — ORR Enabled

## 1) Verify BGP sessions

```text
root@RR> show bgp summary 
Threading mode: BGP I/O
Default eBGP mode: advertise - accept, receive - accept
Groups: 3 Peers: 4 Down peers: 0
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
inet.0               
                       2          1          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
10.255.0.1            65000          9         10       0       2        2:56 Establ
  inet.0: 0/0/0/0
10.255.0.2            65000        463        456       0       0     3:26:31 Establ
  inet.0: 0/1/1/0
10.255.0.3            65000         31         28       0       1       12:26 Establ
  inet.0: 1/1/1/0
10.255.0.4            65000        398        397       0       0     2:58:06 Establ
  inet.0: 0/0/0/0
```

(Optional) I cleared all peers to force clean advertisement behaviour:

```text
root@RR> clear bgp neighbor all   
Cleared 4 connections
```

---

## 2) ORR configuration on RR

### RR-R1 group (client group)

```text
root@RR> show configuration protocols bgp group RR-R1 | display set 
set protocols bgp group RR-R1 type internal
set protocols bgp group RR-R1 local-address 10.255.0.10
set protocols bgp group RR-R1 family inet unicast
set protocols bgp group RR-R1 cluster 10.255.0.10
set protocols bgp group RR-R1 optimal-route-reflection igp-primary 10.255.0.2
set protocols bgp group RR-R1 optimal-route-reflection igp-backup 10.255.0.3
set protocols bgp group RR-R1 optimal-route-reflection export ORR-EXPORT
set protocols bgp group RR-R1 neighbor 10.255.0.1
```

### RR-R4 group (client group)

```text
root@RR> show configuration protocols bgp group RR-R4 | display set 
set protocols bgp group RR-R4 type internal
set protocols bgp group RR-R4 local-address 10.255.0.10
set protocols bgp group RR-R4 family inet unicast
set protocols bgp group RR-R4 cluster 10.255.0.10
set protocols bgp group RR-R4 optimal-route-reflection igp-primary 10.255.0.3
set protocols bgp group RR-R4 optimal-route-reflection igp-backup 10.255.0.2
set protocols bgp group RR-R4 neighbor 10.255.0.4
```

> Note: ORR uses a dedicated ORR export policy hook (`optimal-route-reflection export`).
> In this lab I used a simple “accept all” policy so ORR can advertise routes normally:
>
> `set policy-options policy-statement ORR-EXPORT then accept`

---

## 3) ORR state (IGP-aware view on RR)

This command is the best “proof” that ORR is calculating metrics per ORR peer-group:

```text
root@RR> show isis bgp-orr 
BGP ORR Peer Group: RR-R1
  Primary: 10.255.0.2, active
  Backup: 10.255.0.3
IPv4/IPv6 ORR Routes
--------------------
Prefix                  L Version   Metric Type
10.0.5.0/31             2      92      120 int 
10.255.0.1/32           2      92       20 int 
10.255.0.2/32           2      92        0 int 
10.255.0.3/32           2      92       20 int 
10.255.0.4/32           2      92       20 int 
128.49.0.199/32         2      92       20 int 
128.49.11.40/32         2      92        0 int 
128.49.11.190/32        2      92       20 int 
128.49.12.143/32        2      92       20 int 

BGP ORR Peer Group: RR-R4
  Primary: 10.255.0.3, active
  Backup: 10.255.0.2
IPv4/IPv6 ORR Routes
--------------------
Prefix                  L Version   Metric Type
10.0.5.0/31             2      92      100 int 
10.255.0.1/32           2      92       20 int 
10.255.0.2/32           2      92      100 int 
10.255.0.3/32           2      92        0 int 
10.255.0.4/32           2      92       20 int 
128.49.0.199/32         2      92        0 int 
128.49.11.40/32         2      92      100 int 
128.49.11.190/32        2      92       20 int 
128.49.12.143/32        2      92       20 int 
```

---

## 4) What RR advertises (ORR behaviour)

### RR → R1 (client)

```text
root@RR> show route advertising-protocol bgp 10.255.0.1 203.0.113.0/24 detail 

inet.0: 26 destinations, 27 routes (26 active, 0 holddown, 0 hidden)
  203.0.113.0/24 (2 entries, 2 announced)
 BGP group RR-R1 type Internal
     Nexthop: 10.255.0.2
     Localpref: 100
     AS path: [65000] I 
     Cluster ID: 10.255.0.10
     Originator ID: 10.255.0.2
```

### RR → R4 (client)

```text
root@RR> show route advertising-protocol bgp 10.255.0.4 203.0.113.0/24 detail 

inet.0: 26 destinations, 27 routes (26 active, 0 holddown, 0 hidden)
* 203.0.113.0/24 (2 entries, 2 announced)
 BGP group RR-R4 type Internal
     Nexthop: 10.255.0.3
     Localpref: 100
     AS path: [65000] I 
     Cluster ID: 10.255.0.10
     Originator ID: 10.255.0.3
```

✅ **This is the ORR “money shot”:** RR advertises the **same prefix** to two different clients with **different next-hops**, based on IGP-aware optimality.


## 5) Client verification (R1)

R1 receives `203.0.113.0/24` from RR with next-hop **10.255.0.2**:

```text
root@R1> show route receive-protocol bgp 10.255.0.10 203.0.113.0/24 detail 

inet.0: 23 destinations, 23 routes (23 active, 0 holddown, 0 hidden)
* 203.0.113.0/24 (1 entry, 1 announced)
     Accepted
     Nexthop: 10.255.0.2
     Localpref: 100
     AS path: I  (Originator)
     Cluster list:  10.255.0.10
     Originator ID: 10.255.0.2
```

Installed route:

```text
root@R1> show route 203.0.113.0/24 detail 

inet.0: 23 destinations, 23 routes (23 active, 0 holddown, 0 hidden)
203.0.113.0/24 (1 entry, 1 announced)
        *BGP    Preference: 170/-101
                Next hop type: Indirect, Next hop index: 0
                Address: 0x8a5ba14
                Next-hop reference count: 2
                Kernel Table Id: 0
                Source: 10.255.0.10
                Next hop type: Router, Next hop index: 592
                Next hop: 10.0.1.1 via ge-0/0/1.0, selected
                Session Id: 140
                Protocol next hop: 10.255.0.2  <<<<<<<<<<<<<<<<<
                Indirect next hop: 0x8d1e080 1048580 INH Session ID: 327, INH non-key opaque: 0x0, INH key opaque: 0x0
                State: <Active Int Ext>
                Local AS: 65000 Peer AS: 65000
                Age: 6:34       Metric2: 110 
                Validation State: unverified 
                ORR Generation-ID: 0 
                Task: BGP_65000.10.255.0.10
                Announcement bits (2): 0-KRT 5-Resolve tree 4 
                AS path: I  (Originator)
                Cluster list:  10.255.0.10
                Originator ID: 10.255.0.2
                Accepted
                Localpref: 100
                Router ID: 10.255.0.10
                Thread: junos-main 
```

Forwarding check:

```text
root@R1> show route forwarding-table destination 203.0.113.1 
Routing table: default.inet
Internet:
Destination        Type RtRef Next hop           Type Index    NhRef Netif
203.0.113.0/24     user     0                    indr  1048580     2
                              10.0.1.1           ucst      592    16 ge-0/0/1.0

Routing table: __juniper_services__.inet
Internet:
Destination        Type RtRef Next hop           Type Index    NhRef Netif
default            perm     0                    dscd      517     2

Routing table: __pfe_private__.inet
Internet:
Destination        Type RtRef Next hop           Type Index    NhRef Netif
default            perm     0                    dscd      530     2

Routing table: __master.anon__.inet
Internet:
Destination        Type RtRef Next hop           Type Index    NhRef Netif
default            perm     0                    rjct      545     1	
```

---

## 7) Client verification (R4)

R4 receives `203.0.113.0/24` from RR with next-hop **10.255.0.3**:

```text
root@R4> show route receive-protocol bgp 10.255.0.10 203.0.113.0/24 detail 

inet.0: 23 destinations, 23 routes (23 active, 0 holddown, 0 hidden)
* 203.0.113.0/24 (1 entry, 1 announced)
     Accepted
     Nexthop: 10.255.0.3
     Localpref: 100
     AS path: I  (Originator)
     Cluster list:  10.255.0.10
     Originator ID: 10.255.0.3
```

Installed route:

```text
root@R4> show route 203.0.113.0/24 detail 

inet.0: 23 destinations, 23 routes (23 active, 0 holddown, 0 hidden)
203.0.113.0/24 (1 entry, 1 announced)
        *BGP    Preference: 170/-101
                Next hop type: Indirect, Next hop index: 0
                Address: 0x8a5c414
                Next-hop reference count: 2
                Kernel Table Id: 0
                Source: 10.255.0.10
                Next hop type: Router, Next hop index: 592
                Next hop: 10.0.2.0 via ge-0/0/1.0, selected
                Session Id: 140
                Protocol next hop: 10.255.0.3  <<<<<<<<<<
                Indirect next hop: 0x8d1e080 1048577 INH Session ID: 324, INH non-key opaque: 0x0, INH key opaque: 0x0
                State: <Active Int Ext>
                Local AS: 65000 Peer AS: 65000
                Age: 6:47       Metric2: 20 
                Validation State: unverified 
                ORR Generation-ID: 0 
                Task: BGP_65000.10.255.0.10
                Announcement bits (2): 0-KRT 5-Resolve tree 4 
                AS path: I  (Originator)
                Cluster list:  10.255.0.10
                Originator ID: 10.255.0.3
                Accepted
                Localpref: 100
                Router ID: 10.255.0.10
                Thread: junos-main 
```

Forwarding check:

```text
root@R4> show route forwarding-table destination 203.0.113.1 
Routing table: default.inet
Internet:
Destination        Type RtRef Next hop           Type Index    NhRef Netif
203.0.113.0/24     user     0                    indr  1048577     2
                              10.0.2.0           ucst      592    16 ge-0/0/1.0

Routing table: __juniper_services__.inet
Internet:
Destination        Type RtRef Next hop           Type Index    NhRef Netif
default            perm     0                    dscd      517     2

Routing table: __pfe_private__.inet
Internet:
Destination        Type RtRef Next hop           Type Index    NhRef Netif
default            perm     0                    dscd      530     2

Routing table: __master.anon__.inet
Internet:
Destination        Type RtRef Next hop           Type Index    NhRef Netif
default            perm     0                    rjct      545     1

```

---

# Mode 2 — Normal RR (ORR Disabled)

In this mode, ORR configuration is removed. RR behaves like a traditional route-reflector and selects a **single best path** for the prefix, which it reflects to all clients.

## 1) RR BGP config (no ORR)

```text
root@RR> show configuration protocols bgp| display set 
set protocols bgp group RR-CORE type internal
set protocols bgp group RR-CORE local-address 10.255.0.10
set protocols bgp group RR-CORE family inet unicast
set protocols bgp group RR-CORE neighbor 10.255.0.2
set protocols bgp group RR-CORE neighbor 10.255.0.3
set protocols bgp group RR-R1 type internal
set protocols bgp group RR-R1 local-address 10.255.0.10
set protocols bgp group RR-R1 family inet unicast
set protocols bgp group RR-R1 cluster 10.255.0.10
set protocols bgp group RR-R1 neighbor 10.255.0.1
set protocols bgp group RR-R4 type internal
set protocols bgp group RR-R4 local-address 10.255.0.10
set protocols bgp group RR-R4 family inet unicast
set protocols bgp group RR-R4 cluster 10.255.0.10
set protocols bgp group RR-R4 neighbor 10.255.0.4
```

And confirming there is **no ORR**:

```text
root@RR> show configuration protocols bgp | display set | match optimal-route-reflection
```

---

## 2) What RR advertises (normal RR behaviour)

RR advertises the prefix to R1 & R4 , but it chooses a **single** next-hop:

```text
root@RR> show route advertising-protocol bgp 10.255.0.1 203.0.113.0/24 detail 

inet.0: 26 destinations, 27 routes (26 active, 0 holddown, 0 hidden)
* 203.0.113.0/24 (2 entries, 1 announced)
 BGP group RR-R1 type Internal
     Nexthop: 10.255.0.3
     Localpref: 100
     AS path: [65000] I 
     Cluster ID: 10.255.0.10
     Originator ID: 10.255.0.3

root@RR> 

root@RR> 

root@RR> show route advertising-protocol bgp 10.255.0.4 203.0.113.0/24 detail    

inet.0: 26 destinations, 27 routes (26 active, 0 holddown, 0 hidden)
* 203.0.113.0/24 (2 entries, 1 announced)
 BGP group RR-R4 type Internal
     Nexthop: 10.255.0.3
     Localpref: 100
     AS path: [65000] I 
     Cluster ID: 10.255.0.10
     Originator ID: 10.255.0.3

```

---

## 3) Client verification

### R1 (normal RR)

```text
root@R1> show route receive-protocol bgp 10.255.0.10 203.0.113.0/24 detail    

inet.0: 23 destinations, 23 routes (23 active, 0 holddown, 0 hidden)
* 203.0.113.0/24 (1 entry, 1 announced)
     Accepted
     Nexthop: 10.255.0.3
     Localpref: 100
     AS path: I  (Originator)
     Cluster list:  10.255.0.10
     Originator ID: 10.255.0.3
```

Installed route also reflects the same next-hop decision:

```text
root@R1> show route 203.0.113.0/24 detail                                     

inet.0: 23 destinations, 23 routes (23 active, 0 holddown, 0 hidden)
203.0.113.0/24 (1 entry, 1 announced)
        *BGP    Preference: 170/-101
                Next hop type: Indirect, Next hop index: 0
                Address: 0x8a5ba14
                Next-hop reference count: 2
                Kernel Table Id: 0
                Source: 10.255.0.10
                Next hop type: Router, Next hop index: 592
                Next hop: 10.0.1.1 via ge-0/0/1.0, selected
                Session Id: 140
                Protocol next hop: 10.255.0.3 <<<<<<<<<<<<<<<<
                Indirect next hop: 0x8d1e548 1048582 INH Session ID: 329, INH non-key opaque: 0x0, INH key opaque: 0x0
                State: <Active Int Ext>
                Local AS: 65000 Peer AS: 65000
                Age: 2:14       Metric2: 20 
                Validation State: unverified 
                ORR Generation-ID: 0 
                Task: BGP_65000.10.255.0.10
                Announcement bits (2): 0-KRT 5-Resolve tree 4 
                AS path: I  (Originator)
                Cluster list:  10.255.0.10
                Originator ID: 10.255.0.3
                Accepted
                Localpref: 100
                Router ID: 10.255.0.10
                Thread: junos-main 
...
```

### R4 (normal RR)

```text
root@R4> show route receive-protocol bgp 10.255.0.10 203.0.113.0/24 detail    

inet.0: 23 destinations, 23 routes (23 active, 0 holddown, 0 hidden)
* 203.0.113.0/24 (1 entry, 1 announced)
     Accepted
     Nexthop: 10.255.0.3
     Localpref: 100
     AS path: I  (Originator)
     Cluster list:  10.255.0.10
     Originator ID: 10.255.0.3

root@R4> show route 203.0.113.0/24 detail                                     

inet.0: 23 destinations, 23 routes (23 active, 0 holddown, 0 hidden)
203.0.113.0/24 (1 entry, 1 announced)
        *BGP    Preference: 170/-101
                Next hop type: Indirect, Next hop index: 0
                Address: 0x8a5be94
                Next-hop reference count: 2
                Kernel Table Id: 0
                Source: 10.255.0.10
                Next hop type: Router, Next hop index: 592
                Next hop: 10.0.2.0 via ge-0/0/1.0, selected
                Session Id: 140
                Protocol next hop: 10.255.0.3 <<<<<<<<<<<<<
                Indirect next hop: 0x8d1e080 1048578 INH Session ID: 325, INH non-key opaque: 0x0, INH key opaque: 0x0
                State: <Active Int Ext>
                Local AS: 65000 Peer AS: 65000
                Age: 5:15       Metric2: 20 
                Validation State: unverified 
                ORR Generation-ID: 0 
                Task: BGP_65000.10.255.0.10
                Announcement bits (2): 0-KRT 5-Resolve tree 4 
                AS path: I  (Originator)
                Cluster list:  10.255.0.10
                Originator ID: 10.255.0.3
                Accepted
                Localpref: 100
                Router ID: 10.255.0.10
                Thread: junos-main

```

✅ Both clients are receiving the same reflected best path because RR is doing a single “best-path” selection from its own perspective.

---

# Summary of Results

| Mode            | RR → R1 next-hop for `203.0.113.0/24` | RR → R4 next-hop for `203.0.113.0/24` | Behaviour                             |
| --------------- | ------------------------------------- | ------------------------------------- | ------------------------------------- |
| **ORR enabled** | `10.255.0.2`                          | `10.255.0.3`                          | **Per-client optimal** reflection     |
| **Normal RR**   | `10.255.0.3`                          | `10.255.0.3`                          | **Single best-path** reflected to all |

---

# Commands Used (Quick Reference)

**On RR**

* `show isis bgp-orr`
* `show route 203.0.113.0/24 extensive`
* `show route advertising-protocol bgp <client> 203.0.113.0/24 detail`
* `show configuration protocols bgp group <group> | display set`
* `clear bgp neighbor all` / `clear bgp neighbor <ip>`

**On Clients**

* `show route receive-protocol bgp <RR-lo0> 203.0.113.0/24 detail`
* `show route 203.0.113.0/24 detail`
* `show route forwarding-table destination 203.0.113.1`

---

## Closing Notes

This small lab shows the practical difference between classic RR and ORR:

* **RR** is scalable but can force suboptimal exits.
* **ORR** keeps the scaling benefits of RR while improving **egress optimality per client** using IGP information. Extemley handy when you have a transit network dotted across the globe to avoid suboptimal routing.
* While using primary and secondary IGP anchors allows a BGP-ORR group to maintain valid routes in the event of the primary router failing, it does not improve convergence in the event of a network event that removes reachability to the preferred egress point.  
---
