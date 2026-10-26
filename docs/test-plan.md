# Test plan and evidence

Method: validate from the core towards the services and stop at the first failure.

| # | Layer | Commands / actions | Expected | Status |
|---|---|---|---|---|
| 1 | Core (R0) | `show ip interface brief`; `show ip ospf neighbor` | Interfaces up/up; 3 neighbours FULL (RE1, RE2, RI) | Pass |
| 2 | Routing (RE1) | `show ip route ospf`; `ping 212.217.7.10`; `ping 212.217.26.20` | Routes to 212.217.7.8/30, 212.217.7.12/30, 212.217.26.0/24; pings succeed | Pass |
| 3 | LAN to Internet | CA1: `ping 212.217.26.20`, browse `http://internet.com`; RE1: `show ip nat translations` | Page shown; CA1 translated to 212.217.7.1 | Pass |
| 4 | Internet to DMZ | CI: browse `http://entreprise.com`; RE1: `show ip nat translations` | DMZ page shown; static entry 212.217.7.3 to 192.168.5.20 | Pass |
| 5 | Remote site | PC0, PC1: DHCP; PC0: `ping 212.217.26.20`, browse `internet.com`; RE2: `show ip nat translations` | 192.168.6.x lease; Internet through PAT | Pass |
| 6 | Regression | CA1: browse `http://entreprise.com` | DMZ still reachable after WAN changes | Pass |
| 7 | GRE/IPsec tunnel | `show interface tunnel 0`; `show crypto isakmp sa`; `show crypto ipsec sa`; ping and `tracert` through the tunnel; `show ip eigrp neighbors` | up/up; QM_IDLE; non-zero encrypt/decrypt counters; EIGRP adjacency | Pass |

## Tunnel evidence

| Check | Observation |
|---|---|
| Tunnel interface | Tunnel0 up/up on both routers; GRE/IP 212.217.7.1 to 212.217.7.10 and back; 172.16.0.1/30 and 172.16.0.2/30 |
| IKE phase 1 | `show crypto isakmp sa`: QM_IDLE / ACTIVE on both routers |
| IPsec phase 2 | ESP with AES-256 and SHA-HMAC, tunnel mode, protecting GRE (protocol 47); counters RE1 248/267, RE2 269/250 |
| EIGRP 100 | Neighbour on 172.16.0.0/30, around 6 minutes old, initial flaps stopped |
| Routes via tunnel | RE1 learns 192.168.6.0/24 via 172.16.0.2. RE2 learns 192.168.5.0/24 (internal) and 192.168.10.0 to 192.168.60.0/24 (D EX) via 172.16.0.1 |
| Real path | `tracert` PC0 to 192.168.10.x: 192.168.6.1, 172.16.0.1, 192.168.3.2, destination. R0 is not a hop |
| Simulation mode | ICMP PDU from CA1 to RE2: Successful |

## Troubleshooting log

| Finding | Likely cause | Treatment |
|---|---|---|
| EIGRP adjacency unstable when the tunnel comes up | IPsec negotiation still converging; hellos lost before the SAs exist | Wait for stabilisation (about 6 minutes observed), check `show crypto isakmp sa` |
| `0 packets output` on Tunnel0 | Packet Tracer display quirk | Use `show crypto ipsec sa` counters as proof |
| Config captured from R0 carried the name RE1 | Capture taken on the wrong device or config pasted on the wrong router | Check the prompt before capturing; redo `show running-config` on R0 |
| `show running-config` refused on SWN3B | User mode (`>`) | Run `enable` first |
| Public addresses .3 and .4 taken inside the /29 link | Deliberate choice | Avoids extra routes; RE1 answers ARP for the static NAT addresses |
| Tunnel traffic not translated | Tunnel0 is neither inside nor outside | Private addresses stay intact on the encrypted path |
| Default route redistributed into EIGRP | `redistribute static` includes 0.0.0.0/0 | No effect on RE2 (static, AD 1, beats EIGRP external, AD 170); filter in production |
