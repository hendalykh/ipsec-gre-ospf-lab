# Site-to-Site IPsec VPN over GRE with OSPF

A two-site VPN built in Cisco Packet Tracer: a GRE tunnel carrying OSPF between two private LANs, encrypted with IPsec.

**Stack:** GRE (IP protocol 47) → IPsec (IKEv1, AES-256, SHA) → OSPF single-area

---

## The problem

Two offices need to reach each other's private subnets across an untrusted network, and they need **dynamic routing** rather than static routes that break every time the topology changes.

Neither protocol solves this alone:

| | GRE | IPsec (crypto map) |
|---|---|---|
| Carries multicast | Yes | No |
| Supports routing protocols | Yes | No |
| Encrypts | No | Yes |

OSPF discovers neighbors using multicast (224.0.0.5). Classic crypto-map IPsec protects unicast only, so OSPF hellos never form an adjacency across it. GRE carries multicast happily but sends everything in plaintext.

**Solution:** run OSPF inside GRE, and encrypt the GRE with IPsec. Each protocol covers the other's gap.

---

## Topology

```
   192.168.1.0/24                                      192.168.2.0/24
        PC1                                                  PC2
         |                                                    |
      f0/1|                                                |f0/1
      [ R1 ]------------ 203.0.113.0/30 ------------------[ R2 ]
      f0/0 .1                                          .2 f0/0
         |                                                    |
         +========= Tunnel0  10.10.10.0/30  ==================+
                    GRE over IPsec, OSPF area 0
```

### Addressing

| Device | Interface | Address | Role |
|---|---|---|---|
| R1 | f0/0 | 203.0.113.1/30 | Untrusted transit |
| R1 | f0/1 | 192.168.1.1/24 | Site A LAN gateway |
| R1 | Tunnel0 | 10.10.10.1/30 | Tunnel endpoint |
| R2 | f0/0 | 203.0.113.2/30 | Untrusted transit |
| R2 | f0/1 | 192.168.2.1/24 | Site B LAN gateway |
| R2 | Tunnel0 | 10.10.10.2/30 | Tunnel endpoint |
| PC1 | — | 192.168.1.10/24 | Site A host |
| PC2 | — | 192.168.2.10/24 | Site B host |

Three separate subnets by design: the **underlay** (203.0.113.0/30) carries the tunnel, the **overlay** (10.10.10.0/30) is the tunnel itself, and the LANs ride inside it. `203.0.113.0/30` is deliberately never advertised into OSPF — it is transport, not part of the routing domain.

---

## How a packet travels

1. PC1 sends to PC2: `192.168.1.10 → 192.168.2.10`
2. R1 routes it out Tunnel0 per its OSPF-learned route
3. GRE encapsulates: `[203.0.113.1 → 203.0.113.2][GRE][original packet]`
4. The packet reaches f0/0, where the crypto map matches it against the ACL
5. IPsec encrypts the GRE payload and forwards it
6. R2 decrypts, strips GRE, and delivers to PC2

Order matters: **GRE encapsulates first, IPsec encrypts second.** This is why the crypto map is applied to the physical interface rather than the tunnel, and why the crypto ACL matches GRE between the *physical* addresses.

---

## Configuration

### R1

```cisco
hostname R1
!
interface FastEthernet0/0
 ip address 203.0.113.1 255.255.255.252
 no shutdown
!
interface FastEthernet0/1
 ip address 192.168.1.1 255.255.255.0
 no shutdown
!
interface Tunnel0
 ip address 10.10.10.1 255.255.255.252
 tunnel source FastEthernet0/0
 tunnel destination 203.0.113.2
!
! ---------- IKE Phase 1 ----------
crypto isakmp policy 10
 encr aes 256
 hash sha
 authentication pre-share
 group 5
 lifetime 86400
!
crypto isakmp key SecureKey123 address 203.0.113.2
!
! ---------- IKE Phase 2 ----------
crypto ipsec transform-set TS esp-aes 256 esp-sha-hmac
!
! Interesting traffic: GRE between the physical endpoints
ip access-list extended VPN-TRAFFIC
 permit gre host 203.0.113.1 host 203.0.113.2
!
crypto map CMAP 10 ipsec-isakmp
 set peer 203.0.113.2
 set transform-set TS
 match address VPN-TRAFFIC
!
interface FastEthernet0/0
 crypto map CMAP
!
! ---------- Routing ----------
router ospf 1
 router-id 1.1.1.1
 network 10.10.10.0 0.0.0.3 area 0
 network 192.168.1.0 0.0.0.255 area 0
 passive-interface FastEthernet0/1
```

### R2

Identical, with peer addresses and the crypto ACL reversed:

```cisco
hostname R2
!
interface FastEthernet0/0
 ip address 203.0.113.2 255.255.255.252
 no shutdown
!
interface FastEthernet0/1
 ip address 192.168.2.1 255.255.255.0
 no shutdown
!
interface Tunnel0
 ip address 10.10.10.2 255.255.255.252
 tunnel source FastEthernet0/0
 tunnel destination 203.0.113.1
!
crypto isakmp policy 10
 encr aes 256
 hash sha
 authentication pre-share
 group 5
 lifetime 86400
!
crypto isakmp key SecureKey123 address 203.0.113.1
!
crypto ipsec transform-set TS esp-aes 256 esp-sha-hmac
!
ip access-list extended VPN-TRAFFIC
 permit gre host 203.0.113.2 host 203.0.113.1
!
crypto map CMAP 10 ipsec-isakmp
 set peer 203.0.113.1
 set transform-set TS
 match address VPN-TRAFFIC
!
interface FastEthernet0/0
 crypto map CMAP
!
router ospf 1
 router-id 2.2.2.2
 network 10.10.10.0 0.0.0.3 area 0
 network 192.168.2.0 0.0.0.255 area 0
 passive-interface FastEthernet0/1
```

---

## Verification

### Phase 1 established

```
R1#show crypto isakmp sa
IPv4 Crypto ISAKMP SA
dst             src             state          conn-id slot status
203.0.113.2     203.0.113.1     QM_IDLE           1089    0 ACTIVE
```

`QM_IDLE` means Phase 1 completed and is sitting ready — idle here indicates health, not a stall.

### Phase 2 encrypting

```
R1#show crypto ipsec sa
 Crypto map tag: CMAP, local addr 203.0.113.1

  local  ident (addr/mask/prot/port): (203.0.113.1/255.255.255.255/47/0)
  remote ident (addr/mask/prot/port): (203.0.113.2/255.255.255.255/47/0)
  current_peer 203.0.113.2 port 500
   PERMIT, flags={origin_is_acl,}
  #pkts encaps: 31, #pkts encrypt: 31, #pkts digest: 0
  #pkts decaps: 30, #pkts decrypt: 30, #pkts verify: 0
  #send errors 1, #recv errors 0
```

Three things confirm success:

- **`prot 47`** — the protected traffic is GRE, so IPsec is wrapping the tunnel as intended
- **`origin_is_acl`** — the SA was built from the crypto ACL, meaning the ACL matched
- **encaps and decaps both incrementing** — traffic is encrypted in both directions

`#send errors 1` is the first packet, dropped while the SA negotiated. IPsec builds on demand, so the initial packet is always lost.

### OSPF adjacency across the encrypted tunnel

```
R1#show ip ospf neighbor

Neighbor ID     Pri   State           Dead Time   Address         Interface
2.2.2.2           0   FULL/  -        00:00:33    10.10.10.2      Tunnel0
```

This is the result the whole design exists for: a multicast routing protocol running over an encrypted tunnel. The `FULL/ -` state shows no DR/BDR role, because GRE tunnels are point-to-point by default and only broadcast/NBMA networks hold elections.

---

## Gotchas

**Crypto ACL must match GRE between physical IPs.** The instinctive `permit ip 192.168.1.0 0.0.0.255 192.168.2.0 0.0.0.255` matches nothing — by the time IPsec inspects the packet, GRE has already rewritten the header, and the LAN addresses are buried in the payload. The result is a tunnel that works perfectly and encrypts nothing. Only the encaps counters reveal it.

**Crypto map goes on the physical interface.** Applying it to Tunnel0 places the check before encapsulation, when the packet does not yet look like GRE, so the ACL never matches.

**Phase 1 parameters must match exactly.** Encryption, hash, authentication method, and DH group all have to agree. A single mismatch causes a silent failure with no useful error message.

**Do not advertise the transit link into OSPF.** Advertising 203.0.113.0/30 can cause recursive routing, where the tunnel attempts to reach its own destination through itself and flaps.

---

## Packet Tracer limitations

Two places where the simulator diverges from real IOS:

**Security license on the 2911.** The 2911 requires `license boot module c2900 technology-package securityk9` before any crypto command works. In this Packet Tracer build the command is accepted but never persists to the running config, so the security package can't be activated and `crypto isakmp` remains unavailable. Worked around by using **2811 routers**, which run IOS 12.4 and predate technology-package licensing.

**Transport mode unavailable.** GRE over IPsec should use `mode transport`, since GRE has already added an outer IP header and tunnel mode would add a redundant third. Packet Tracer does not implement the transform-set sub-mode — the prompt never enters `(cfg-crypto-trans)` — so this lab runs in the default tunnel mode, costing 20 bytes of overhead per packet.

---

## What this demonstrates

- GRE tunnel design and the underlay/overlay distinction
- IKE Phase 1 and Phase 2 negotiation and how they differ
- Crypto ACLs as traffic classifiers rather than filters
- Why encapsulation order dictates where the crypto map is applied
- OSPF over a tunnel interface, including point-to-point neighbor behavior
- Verifying encryption from SA state and counters instead of assuming a successful ping means success
