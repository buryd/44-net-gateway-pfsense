# 44 Net gateway on pfSense

Worked example. Replace `44.63.100.0/28` with the prefix ARDC assigns you.

pfSense is the firewall and the default gateway for every 44 host. A Linux VM behind it is only the IPIP mesh endpoint. You reach hosts from the public internet through WireGuard. A stranger reaches a host only when you add a transit rule for that host and port and, for general internet traffic, an `ampr.org` DNS name.

## What you are building

```
Internet
  |  IP protocol 4 (IPIP) to your public address
  |  WireGuard UDP to pfSense
  v
pfSense WAN
  |-- WireGuard 10.44.255.0/24     you, full access to hosts
  |-- TRANSIT  10.44.0.1  <---->  Linux VM 10.44.0.2
  |                                  tunl0 44.63.100.14/32
  |                                  ampr-ripd  --->  other 44 gateways
  |                                             --->  amprgw 169.228.34.84
  |-- AMPRLAN  44.63.100.1/28      hosts .2 – .13
  |-- LAN      192.168.1.0/24      home PCs
```

FreeBSD IPIP tunnels are point-to-point. The 44 Net mesh needs one tunnel and a different outer endpoint for each remote gateway. Linux `tunl0` plus `ampr-ripd` does that. The VM has no NIC on the 44 VLAN, so every decapsulated packet comes back through pfSense before it reaches a host.

## Requirements

- A public IPv4 on the pfSense WAN. CGNAT will not work for the IPIP mesh.
- The ISP path must pass IP protocol 4 in both directions.
- An ARDC portal account and an assignment. Use case: IPIP Tunnel Mesh, from `44.63.0.0/16`. Request a `/28` or `/29`.
- A hostname that resolves to the WAN address. Use dynamic DNS when the WAN address changes. The portal resolves that hostname when it builds the mesh.
- A ham allocation is for amateur radio use under the ARDC terms. Log into the portal at least every few months so the assignment stays active.

Confirm `169.228.34.84` in the portal before use. That is the long-standing UCSD amprgw address and it can change.

If the ISP blocks protocol 4, stop here and use 44Net Connect (ARDC WireGuard) instead. This design depends on IPIP.

## Address plan

| Role | Address |
| --- | --- |
| Prefix | `44.63.100.0/28` |
| pfSense AMPRLAN | `44.63.100.1/28` |
| Hosts | `44.63.100.2` – `44.63.100.13` |
| Linux tunl0 | `44.63.100.14/32` |
| pfSense TRANSIT | `10.44.0.1/29` |
| Linux transit NIC | `10.44.0.2/29` |
| WireGuard | `10.44.255.1/24` on pfSense, peer `10.44.255.2/32` |
| Home LAN | your existing LAN, shown as `192.168.1.0/24` |

Network and broadcast of the `/28` are `.0` and `.15`. Leave them unused.

Each 44 host has one NIC, on the 44 VLAN, with a static address, mask `255.255.255.240`, and gateway `44.63.100.1`. A second NIC on the home LAN would bypass the firewall.

## Portal

1. Register at the ARDC portal.
2. Request the prefix. Use case IPIP Tunnel Mesh. Size `/28` or `/29`. Description can be your callsign.
3. Create a gateway. Encap type IPIP. Hostname is the name that resolves to the pfSense WAN address.
4. Link the prefix to that gateway.
5. Create an `ampr.org` A record only for a host you intend to publish. amprgw does not deliver general internet traffic to an address with no DNS name. Other mesh gateways can still send packets, which is why the transit rules default to deny.

## pfSense interfaces

Create two interfaces besides WAN and LAN.

**AMPRLAN.** VLAN or dedicated NIC. IPv4 `44.63.100.1/28`. MSS `1440`. Use `1380` when WAN is PPPoE (the IPIP header costs 20 bytes; PPPoE costs another 8).

**TRANSIT.** VLAN or dedicated NIC to the VM only. IPv4 `10.44.0.1/29`. The VM is the only device on this network.

**Gateway.** System → Routing → Gateways. Name `AMPRGW`, interface TRANSIT, address `10.44.0.2`.

**Static route.** `44.0.0.0/8` via `AMPRGW`. The connected route `44.63.100.0/28` is more specific, so local hosts stay on AMPRLAN.

**GUI.** System → Advanced → Admin Access. Listen on the LAN address. AMPRLAN, TRANSIT, and WAN do not serve the GUI.

**Outbound NAT.** Leave Automatic or Hybrid. Traffic from AMPRLAN to TRANSIT is between internal interfaces, so the 44 source address is preserved. Traffic from a host to the general internet exits WAN and is translated to the house address. That translation does not accept inbound connections.

Check this after the VM is up: on the Linux transit NIC, packets from a host toward another 44 network must show the host’s 44 address as the source. If you ever see them rewritten, add a No-NAT outbound rule for source `44.63.100.0/28` destination `44.0.0.0/8`.

## pfSense firewall

Rules are on the interface where the packet enters. A pass on AMPRLAN lets a host start a session. The reply matches state and is allowed back through TRANSIT without a pass rule. A new session from the mesh or the internet needs its own pass on TRANSIT.

### Aliases

- `AMPR_NET` = `44.63.100.0/28`
- `AMPR44` = `44.0.0.0/8`
- `LAN_NET` = `192.168.1.0/24`
- `WG_NET` = `10.44.255.0/24`
- One host alias per published service, for example `PUB_NODE` = `44.63.100.10`

### WAN

1. Pass IPv4 protocol IPENCAP, destination WAN address. This is the associated rule for the NAT entry below. Enable logging.
2. Pass UDP, destination WAN address, destination port = your WireGuard port, to *This Firewall*.
3. Leave the default deny in place. Create no port forwards to a 44 host.

### NAT, port forward

- Interface WAN
- Protocol IPENCAP
- Destination WAN address
- Redirect target `10.44.0.2`
- Associate the WAN pass rule above

Outbound IPIP from the VM uses the VM address `10.44.0.2`. Automatic outbound NAT on WAN rewrites that outer source to the public address the portal has for you. The inner 44 header is untouched.

### TRANSIT

Evaluate top to bottom.

1. Pass ICMP echo-request to `AMPR_NET` only if you want outside ping. Otherwise omit it.
2. Pass the published services, one rule each. Example: TCP, destination `PUB_NODE`, port `7300`. Log these.
3. Reject IPv4 destination `LAN_NET`.
4. Reject IPv4 destination `AMPR_NET`. Log this rule. It is the default for strangers and for other 44 gateways.

SSH to the VM is not in this list. Admin of the VM comes from LAN or WireGuard, below.

### AMPRLAN

1. Reject destination *This Firewall*, ports `22`, `80`, `443`, and any other management port.
2. Reject destination `LAN_NET`.
3. Reject destination `10.44.0.0/29`.
4. Pass destination *This Firewall*, UDP/TCP `53` and UDP `123`, if hosts use pfSense for DNS and NTP.
5. Pass destination `AMPR44`.
6. Pass TCP destination any, ports `80` and `443`, gateway WAN. This is outbound update traffic. It is translated to the house address. It does not publish the host.
7. Reject IPv4 any. Log it.

### WIREGUARD

1. Pass source `WG_NET` destination `AMPR_NET`.
2. Pass source `WG_NET` destination `10.44.0.2` TCP `22`, if you want to admin the VM remotely.
3. Reject the rest, or add a separate pass to `LAN_NET` if the same VPN should also reach home PCs.

### LAN

1. Pass destination `AMPR_NET`.
2. Pass destination `10.44.0.2` TCP `22`.

There is no rule anywhere that lets a 44 host open a new session to the home LAN.

## WireGuard

VPN → WireGuard on pfSense. Current CE 2.7 and Plus include it.

- Tunnel address `10.44.255.1/24`
- Listen on WAN, UDP. Pick a port and match the WAN rule.
- Peer allowed IPs: `10.44.255.2/32`
- On the laptop or phone, split tunnel. AllowedIPs = `44.63.100.0/28, 10.44.255.0/24`
- Add `192.168.1.0/24` to AllowedIPs only if that peer should also reach the home LAN

After the tunnel is up, a host is `44.63.100.10`. The session is between your peer and pfSense. The host has no listener exposed to the internet by this path.

Keep SSH, RDP, VNC, SMB, and WinRM on this path only.

## Linux gateway

Debian or Ubuntu, one vCPU, 512 MB RAM is enough. One NIC, address `10.44.0.2/29`, default gateway `10.44.0.1`. Install `ampr-ripd` and `iptables`. Do not put a default route out `tunl0` in the main table. The VM reaches the internet, and you, through pfSense.

`/etc/sysctl.d/44net.conf`:

```
net.ipv4.ip_forward=1
net.ipv4.conf.all.rp_filter=0
net.ipv4.conf.tunl0.rp_filter=0
net.ipv4.icmp_errors_use_inbound_ifaddr=1
```

Loose reverse-path filtering is required. Strict mode drops legitimate mesh packets because the outer path and the inner path differ.

Bring-up, with the example prefix:

```sh
modprobe ipip
ip tunnel add tunl0 mode ipip local 10.44.0.2
ip addr add 44.63.100.14/32 dev tunl0
ip link set tunl0 mtu 1480 up

ip route add 44.63.100.0/28 via 10.44.0.1

ip rule add from 44.63.100.0/28 table 44 priority 44
ip rule add to 44.0.0.0/8 table 44 priority 45
ip route add 44.63.100.0/28 via 10.44.0.1 dev eth0 table 44
ip route add default via 169.228.34.84 dev tunl0 onlink table 44
```

Use the transit NIC name you actually have in place of `eth0`. The route for your own prefix in table 44 is what stops local delivery from being encapsulated back out the tunnel.

`ampr-ripd` learns the other gateways from RIP inside the tunnel (UDP 520 to `224.0.0.9`, source `44.0.0.1`). That port stays inside the tunnel. It is not a WAN forward.

```sh
ampr-ripd -s -i tunl0 -t 44 -a 44.63.100.0/28
```

`-a` tells it not to install a route for your own prefix. `-s` writes the route set under `/var/lib/ampr-ripd/`.

Forwarding filter on the VM. Policy for which ports are open lives on pfSense. This filter only limits what the VM is willing to carry.

```sh
iptables -P FORWARD DROP
iptables -A FORWARD -i tunl0 -o tunl0 -j DROP
iptables -A FORWARD -i tunl0 -s 44.63.100.0/28 -j DROP
iptables -A FORWARD -i tunl0 -p 4 -j DROP
iptables -A FORWARD -i tunl0 -o eth0 -d 44.63.100.0/28 -j ACCEPT
iptables -A FORWARD -i eth0 -o tunl0 -s 44.63.100.0/28 -j ACCEPT

iptables -A INPUT -i eth0 -s 10.44.0.1 -j ACCEPT
iptables -A INPUT -i eth0 -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -i lo -j ACCEPT
iptables -P INPUT DROP
```

Add the usual bogon sources on `tunl0` as well: `0.0.0.0/8`, `10.0.0.0/8`, `100.64.0.0/10`, `127.0.0.0/8`, `169.254.0.0/16`, `172.16.0.0/12`, `192.168.0.0/16`, `192.0.0.0/24`, `192.0.2.0/24`, `198.18.0.0/15`, `198.51.100.0/24`, `203.0.113.0/24`, `224.0.0.0/4`, `240.0.0.0/4`. The wiki page IPIP Mesh/Firewalls is the checklist.

The VM’s own SSH listens on the transit NIC and accepts `10.44.0.1`. Reach it from the LAN rule or the WireGuard rule. Do not open SSH on `tunl0`.

## Publishing one service

Do all three. Any one of them alone is incomplete.

1. pfSense TRANSIT pass: protocol and port, destination that host only. Place it above the reject-all rule.
2. The same allow on the host firewall.
3. An `ampr.org` A record for that address, if general internet users (not only other 44 gateways) should reach it.

Example for a single packet service on `44.63.100.10` TCP `7300`. Nothing else on that host is opened by those three steps.

## Tests

Run them in this order.

1. From a host, ping `44.63.100.1`.
2. On the VM, `ip route show table 44` shows mesh routes after a few minutes, and `tcpdump -n -i tunl0` shows RIP from `44.0.0.1`.
3. From a host, ping a known distant 44 address. On the VM transit NIC the source is the host’s 44 address.
4. From the internet, with the VPN down, the unpublished host does not answer ping or a port scan.
5. Bring up WireGuard. The same host answers from the peer.
6. From a 44 host, a connection to a `192.168.1.x` address fails.
7. After you publish one port, that port answers and a different port on the same host does not.

## What this design leaves out

BGP announcement of your own prefix. That puts the block in the global table and removes the amprgw DNS check. It needs a `/24`, an ASN, and an ISP that will announce 44 Net space without an RPKI ROA. The transit deny rules would still be required, and the exposure is larger. Use BGP only when the mesh path is the wrong tool.

44Net Connect. Use it when you have CGNAT or protocol 4 is filtered. It is a different attachment method and it does not automatically join the IPIP mesh.
