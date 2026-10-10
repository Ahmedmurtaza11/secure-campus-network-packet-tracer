# Secure Campus Network in Cisco Packet Tracer

A small campus network with four LANs in Cisco Packet Tracer. One router connects four switches.

This started as a lab from my Computer Networks course. The task was to subnet a network, assign the addresses, and check connectivity with ping. I finished that, then decided to expand it into a secured network to practise more.

## What the lab asked for

- Subnetting: 192.168.1.0/24 split into four /26 subnets (borrowed 2 bits)
- IP addressing for every device
- Routing between the LANs on a 2811 router (with an NM-2FE2W module for two extra FastEthernet ports)
- Ping tests to confirm every LAN can reach the others

## What I added on top

- DHCP pools on the router, servers on static IPs
- One VLAN per switch
- An extended ACL that blocks the Guest LAN from two servers
- Port security and switch hardening

## Topology

One Cisco 2811 router, four 2960 switches, 16 PCs and 4 servers.

| LAN | VLAN | Subnet | Gateway | Switch | Router port |
|---|---|---|---|---|---|
| Admin | 10 | 192.168.1.0/26 | 192.168.1.1 | Switch1 | fa0/0 |
| Students | 20 | 192.168.1.64/26 | 192.168.1.65 | Switch2 | fa0/1 |
| Staff | 40 | 192.168.1.128/26 | 192.168.1.129 | Switch4 | fa1/0 |
| Guest | 30 | 192.168.1.192/26 | 192.168.1.193 | Switch3 | fa1/1 |

Subnet mask for every LAN: 255.255.255.192 (62 usable hosts).

Static server addresses: 192.168.1.6, 192.168.1.70, 192.168.1.134, 192.168.1.198.

Note: Switch3 and Switch4 are cabled to the opposite router ports from what I first assumed. I confirmed the real mapping with `show cdp neighbors` (see "Problems I hit").

## Addressing method

1. Four LANs need four subnets.
2. 2^2 = 4, so borrow 2 bits: /24 becomes /26 (255.255.255.192).
3. Block size = 256 - 192 = 64, so the networks start at .0, .64, .128 and .192.

## DHCP

Each LAN has its own pool on the router. The first six addresses of each subnet are excluded (gateway and server addresses).

```
ip dhcp excluded-address 192.168.1.1 192.168.1.6
ip dhcp excluded-address 192.168.1.65 192.168.1.70
ip dhcp excluded-address 192.168.1.129 192.168.1.134
ip dhcp excluded-address 192.168.1.193 192.168.1.198

ip dhcp pool LAN1
 network 192.168.1.0 255.255.255.192
 default-router 192.168.1.1
```

LAN2 to LAN4 follow the same pattern with their own network and gateway.

## VLANs

Each switch has one VLAN for all its access ports.

```
vlan 10
 name Admin
interface range fa0/1 - 24
 switchport mode access
 switchport access vlan 10
```

Switch2, Switch3 and Switch4 use VLAN 20, 30 and 40.

## Access control list

The Guest LAN (192.168.1.192/26) cannot reach Server0 (192.168.1.6) or Server1 (192.168.1.70). Everything else is allowed.

```
ip access-list extended BLOCK_SW3
 deny ip 192.168.1.192 0.0.0.63 host 192.168.1.6
 deny ip 192.168.1.192 0.0.0.63 host 192.168.1.70
 permit ip any any
interface fa1/1
 ip access-group BLOCK_SW3 in
```

Every ACL ends with a hidden deny, so the last `permit ip any any` line keeps the rest of the traffic working.

## Switch hardening

On all four switches:

```
interface range fa0/1 - 5
 switchport port-security
 switchport port-security maximum 2
 switchport port-security mac-address sticky
 switchport port-security violation restrict
interface range fa0/7 - 24
 shutdown
```

- Fa0/6 is the uplink to the router and is left untouched
- An enable secret and a login banner are set on each switch (the password is not included in this repo)

## Test results

Tested from PC14 on the Guest LAN:

| Test | Result |
|---|---|
| Ping 192.168.1.6 | Blocked (destination host unreachable) |
| Ping 192.168.1.70 | Blocked (destination host unreachable) |
| Ping 192.168.1.134 | Allowed |

`show access-lists` after testing: 8 matches on each of the two deny rules and on the permit rule. The block still worked after the switch hardening. `show ip route` shows all four LANs as connected.

## Problems I hit

1. **Switch3 and Switch4 on opposite router ports.** My first ACL sat on the wrong interface and blocked nothing. I renamed the switches, ran `show cdp neighbors`, and moved the ACL to fa1/1.
2. **A rogue DHCP service.** DHCP was switched on in Server0 with a wrong gateway, so PCs on that LAN got a bad gateway and cross-LAN pings failed. I found it with `ipconfig` and the server's Services tab, turned it off, and renewed the leases with `ipconfig /renew`.
3. **Small typos.** The gateway field held the server's own IP, a subnet mask was typed into the DNS box, and a few IPs had digits swapped. `ipconfig` and the server settings page found all of them.

## How to open it

1. Install Cisco Packet Tracer (built with 9.0.1).
2. Open the `.pkt` file from this repo.
3. Switch to Realtime mode and test with ping from the PC Command Prompt.

## Files

- `network.pkt`: the Packet Tracer project
- `configs/`: router and switch configs (enable secret removed)
- `screenshots/`: topology, ACL test, port security and graphics

## What I would add next

- DHCP snooping, so only the router can hand out addresses
- A DNS and web server
- NAT with an internet connection
- Router-on-a-stick, so VLANs span more than one switch
