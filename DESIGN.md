# 44 Net gateway on pfSense, using 44Net Connect

Worked example. Replace every 44 address with the values from your Connect dashboard.

pfSense terminates one 44Net Connect WireGuard tunnel. Connect assigns that tunnel a single IPv4 `/32`. A second allocation, the routed subnet, is attached to the same tunnel, and the hosts sit on that subnet. Connect routes the subnet to the tunnel over your existing internet connection, including through NAT or CGNAT. pfSense is the firewall in front of the hosts.

Your own phone or laptop uses a second Connect tunnel and its own `/32`. pfSense allows that one address through to the hosts. Every other internet source stops on the WireGuard interface unless you publish one host and one port.

## Topology

```
Your phone  --Connect WireGuard /32-->  44Net Connect
Internet strangers ------------------>  44Net Connect
                                              |
                                     WireGuard UDP
                                              |
                                         pfSense WAN
                                              |
                                         tunnel /32
                                         interface WG44
                                              |
                                         AMPRLAN
                                    routed subnet from Connect
                                         hosts .130–.134

Home LAN 192.168.1.0/24 stays on the normal WAN gateway.
```

One peer on the gateway tunnel. A second peer on that same tunnel cannot share `0.0.0.0/0`. The phone is its own tunnel in the Connect dashboard, and it is not a peer on pfSense.

## Addresses

These are documentation placeholders. `44.190.40.10/32` is the shape of a tunnel address in the Connect setup help. `44.27.45.128/29` is the shape of a routed subnet in the Connect subnet guides. Your dashboard will issue different numbers. Enter the tunnel address as a `/32` even when the generated config file shows a `/24`.

| Role | Example | Source |
| --- | --- | --- |
| pfSense tunnel | `44.190.40.10/32` | Tunnel config, Interface address |
| Routed subnet | `44.27.45.128/29` | Connect network request, then attached to the pfSense tunnel |
| Host gateway | `44.27.45.129/29` | First usable address, on pfSense AMPRLAN |
| Hosts | `44.27.45.130` – `44.27.45.134` | Static on the 44 VLAN |
| Operator phone | `44.190.41.20/32` | A second Connect tunnel, on the phone only |
| Home LAN | `192.168.1.0/24` | Existing trusted network |

On a `/29`, `.128` is the network address and `.135` is the broadcast. Leave both unused. A `/29` has five host addresses after the gateway. Connect auto-approves a request up to a `/28` (thirteen hosts after the gateway). Attach any extra subnet to this same tunnel.

Each host has one NIC, on the 44 VLAN, mask `255.255.255.248` for this `/29`, gateway `44.27.45.129`. A second NIC on the home LAN would bypass the firewall.

## Connect dashboard

Portal account and a verified callsign live at the ARDC portal. Tunnels and subnets are managed at `connect.44net.cloud`.

1. On pfSense, install WireGuard only if `VPN > WireGuard` is missing. Current pfSense CE 2.7 and Plus already have that menu.
2. `VPN > WireGuard > Tunnels > Add Tunnel`. Generate a keypair. Copy the public key.
3. In Connect, create a tunnel, choose a server, and paste that public key. Keep the downloaded config private. It contains the private key. Do not commit it to the public git repository.
4. Request a network, `/29` or `/28`. After it is approved, edit the pfSense tunnel, open **Route additional Networks Through This Tunnel**, and select that subnet.
5. Create a second tunnel for the phone, with the phone's own public key. Leave the routed subnet attached only to the pfSense tunnel.

From the pfSense tunnel config, copy the endpoint address, UDP port, peer public key, pre-shared key if one is present, persistent keepalive, and MTU.

## pfSense tunnel

Set the IPv4 default gateway to the WAN gateway before assigning the WireGuard interface. Leave it there. With the default left on Automatic, pfSense can move the house default route onto the tunnel.

**Tunnel.** Interface address `44.190.40.10/32`. Skip the IPv6 address. This design numbers hosts with the IPv4 subnet only.

**Peer.** Uncheck Dynamic Endpoint. Endpoint and port come from the config. Keepalive comes from the config, usually `25`, which is what keeps the tunnel up behind NAT. Allowed IPs: `0.0.0.0/0`. That entry tells WireGuard to accept packets from any public source and to send the subnet's traffic to Connect. It does not by itself change the firewall default route.

**Enable.** `VPN > WireGuard > Settings`, enable WireGuard. Confirm a recent handshake under `Status > WireGuard`.

**Assign.** `Interfaces > Assignments`, add the `tun_wg` interface, enable it, and name it `WG44`. Set MTU from the Connect config, or `1420` when the config omits it. Put the same `/32` on this interface as a static IPv4 address if it is not already present on the tunnel. Use it in one place if pfSense rejects the duplicate.

**Gateway.** On that interface, add a gateway named `WG44_GW`. For a `/32` tunnel, set the gateway address to the tunnel address itself, `44.190.40.10`, which is the pfSense client-tunnel pattern. Do not select it as the system default. Disable gateway monitoring on it. A monitor target that never answers marks the gateway down, and pfSense then stops policy-routing the hosts.

**AMPRLAN.** VLAN or dedicated NIC. IPv4 `44.27.45.129/29`. MSS `1380` when the tunnel MTU is `1420` (MTU minus 40). Raise or lower both together if Connect gives a different MTU.

**WireGuard rule tab.** Firewall rules that belong on this tunnel go on the assigned `WG44` tab. Leave the group WireGuard tab empty. Group rules match first and do not get `reply-to`, so return traffic would follow the WAN default route and the inbound session would fail.

## Routing and NAT

Hosts use pfSense as their gateway. A pass rule on AMPRLAN sets the gateway to `WG44_GW`, so traffic from the subnet enters the Connect tunnel with the host's own 44 address as the source.

`reply-to` on the `WG44` rules sends replies to inbound sessions back out the same tunnel.

Outbound NAT mode: Hybrid.

Add this rule at the top of the outbound NAT list:

- Interface `WG44`
- Source `44.27.45.128/29`
- Destination any
- **Do not NAT** checked

Automatic mode NATs traffic leaving an assigned WireGuard interface to the tunnel `/32`. That would make every host answer as `44.190.40.10` and break sessions aimed at the host addresses. The Do-not-NAT rule keeps each host on its own address. Leave the existing WAN NAT rules in place so the home LAN still translates out the ISP address.

Check after a host is up: a query from `44.27.45.130` to a public "what is my IP" service shows `44.27.45.130`. The house WAN address and the tunnel `/32` both mean the policy route or the NAT exception is wrong. `Diagnostics > Routes` shows the default route on the WAN gateway.

## Firewall

A rule lets traffic that enters that interface start a session. Replies follow the state table. Connect delivers the subnet to the tunnel as soon as the tunnel is up, so an empty pass list on `WG44` is what keeps the hosts closed.

### Aliases

- `AMPR_NET` = `44.27.45.128/29`
- `OPERATOR_44` = `44.190.41.20/32`
- `LAN_NET` = `192.168.1.0/24`
- `PUB_HOST` = `44.27.45.130`, only if you publish a service
- `MGMT_PORTS` = `22`, `80`, `443`, plus the real pfSense GUI port if you moved it

### WG44

Top to bottom.

1. Pass source `OPERATOR_44`, destination `AMPR_NET`. This is you, from the phone's Connect tunnel, to every host.
2. Pass a published service only when you want one. Example: TCP, destination `PUB_HOST`, port `7300`. Log it.
3. Reject destination `LAN_NET`.
4. Reject destination `AMPR_NET`. Log it. This is the default for the rest of the internet and for everyone else on 44Net.
5. Reject destination *This Firewall*.

The tunnel `/32` is a public address on pfSense. With no pass to *This Firewall* on this tab, the GUI and SSH are not served on that address.

### AMPRLAN

1. Reject destination *This Firewall*, ports `MGMT_PORTS`.
2. Reject destination `LAN_NET`.
3. Pass source `AMPR_NET`, destination any, gateway `WG44_GW`.

Hosts reach the internet through Connect as their own 44 addresses. They can reach each other on the VLAN. They cannot open a session to the home LAN or the firewall GUI.

### LAN

1. Pass destination `AMPR_NET`.

LAN rules keep the default gateway. Home PCs reach the 44 hosts directly, and their other traffic stays on the ISP link.

### WAN

No port forwards to a 44 host. Inbound host traffic arrives inside the Connect tunnel, on `WG44`.

The GUI listens on the LAN address only.

## Phone tunnel

On the phone, the Connect config keeps its own `/32`. Set the phone's Allowed IPs to `44.27.45.128/29` so only traffic to the hosts uses the tunnel. After the handshake, a host is `44.27.45.130`.

SSH, RDP, VNC, SMB, and WinRM stay covered by the `OPERATOR_44` rule only. Repeat that same allow list on each host firewall.

## Publishing one service

All three, when strangers should reach it:

1. A `WG44` pass above the reject rule, for that host and port only.
2. The same allow on the host firewall.
3. Nothing else. Connect is already routing the address. There is no separate DNS gate in front of the firewall.

## Tests

1. `Status > WireGuard` shows a handshake.
2. The default route is the WAN gateway. A LAN PC still loads a normal website.
3. From a host, ping `44.27.45.129`.
4. From that host, a public IP echo shows the host's 44 address.
5. From the internet, with the phone tunnel down, the host accepts no connection.
6. Bring up the phone tunnel. The host answers from that phone.
7. From a host, a connection to `192.168.1.x` fails.
8. After you add one published port, that port answers and a different port on the same host does not.

## References

- 44Net Connect: `https://wiki.ampr.org/wiki/44Net_Connect`
- pfSense tunnel: `https://wiki.ampr.org/wiki/44Net_Connect/Quick_Start/PFsense`
- Routed subnet: `https://wiki.ampr.org/wiki/44Net_Connect/Routed_Subnet`
- pfSense WireGuard client and return routing: Netgate docs, WireGuard VPN Client Configuration Example, and WireGuard Routing
