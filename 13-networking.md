# Lab 13 — Networking

**Roadmap topics:** TCP/IP Stack · Subnetting · Ethernet & arp/rarp · DHCP · IP Routing · DNS Resolution · Netfilter · SSH · File Transfer
**Time:** about 5 hours. Split it: sections 1–5 (how packets move), then 6–9 (names, firewall, SSH, transfer).
**Touches system state:** lightly, and every change is temporary or reverted in Cleanup: one test route, one line in `/etc/hosts`, one firewall port opened with an automatic timeout, one extra key in your `authorized_keys`. **No interface, address, DNS or sshd setting is changed**, because the tower is remote and on Wi-Fi.

## Rules for network changes on a remote machine

1. Keep a second SSH session open and idle before touching anything in sections 5, 7 or 8. If the first session breaks, the second is your way back.
2. Prefer changes that undo themselves: runtime-only firewall rules with a timeout, routes that vanish at reboot.
3. Validate before applying (`sshd -t`, `nmcli` showing what would change) and never restart networking or `sshd` on a hunch.

## What you will be able to do

- Explain what happens, layer by layer, when one machine talks to another, and show evidence of each step.
- Work out network, broadcast and host range for any CIDR block in your head or on paper.
- Read the interface, neighbour and routing tables.
- Trace a name lookup and find where it goes wrong.
- Read the tower's firewall, including the part Docker manages, and open a port safely.
- Use SSH keys, config files, tunnels and jump hosts, and audit an SSH server's configuration.
- Move files between machines with the right tool.

## Setup

```bash
mkdir -p ~/labwork/13 && cd ~/labwork/13
sudo dnf install -y iproute iputils bind-utils nmap-ncat tcpdump ipcalc traceroute curl wget rsync

# Facts about this machine, used throughout. Saved to a file so that a new shell can reload them.
cat > vars.sh <<'EOF'
IF=$(ip route show default | awk '{print $5; exit}')                      # main interface
GW=$(ip route show default | awk '{print $3; exit}')                      # default gateway
IP=$(ip -4 -o addr show dev "$IF" | awk '{print $4; exit}' | cut -d/ -f1) # this machine's address
CIDR=$(ip -4 -o addr show dev "$IF" | awk '{print $4; exit}')             # address with prefix
CON=$(nmcli -t -f NAME,DEVICE con show --active | awk -F: -v d="$IF" '$2 == d {print $1; exit}')   # NetworkManager profile
EOF
source vars.sh
echo "interface=$IF  address=$CIDR  gateway=$GW  profile=$CON"
```

In any new terminal: `source ~/labwork/13/vars.sh`.

---

## 1. The TCP/IP stack

Networking is built in layers. Each layer does one job and uses the layer below without knowing how it works.

| Layer | Job | Unit | Address | Protocols | Look at it with |
|---|---|---|---|---|---|
| **4 Application** | what the programs say to each other | message | name, URL | HTTP, DNS, SSH, TLS, Bolt | `curl`, `dig`, `ssh` |
| **3 Transport** | deliver to the right program; TCP adds reliability and ordering | segment (TCP), datagram (UDP) | **port** | TCP, UDP | `ss`, `nc` |
| **2 Internet** | get a packet across networks to the right machine | packet | **IP address** | IPv4, IPv6, ICMP | `ip addr`, `ip route`, `ping` |
| **1 Link** | get a frame to the next device on the same physical network | frame | **MAC address** | Ethernet, Wi-Fi, ARP | `ip link`, `ip neigh` |

(The seven-layer OSI model splits these further. "Layer 7" means application, "layer 4" transport, "layer 3" IP, "layer 2" Ethernet. Those four numbers are the ones people say out loud.)

**Encapsulation.** Going down the stack, each layer wraps the data from above in its own header:

```
                                  [ HTTP request ]                    application
                     [ TCP hdr | HTTP request ]                       + source and destination PORT
            [ IP hdr | TCP hdr | HTTP request ]                       + source and destination IP
[ Eth hdr | IP hdr | TCP hdr | HTTP request | Eth trailer ]           + source and destination MAC
```

At each router the Ethernet wrapper is removed and a new one is added for the next hop. The IP packet inside travels end to end.

### Interfaces: layers 1 and 2

```bash
ip -br link                       # brief: name, state, MAC
ip -br addr                       # brief: name, state, addresses
ip addr show dev "$IF"            # full detail
ip -s link show dev "$IF"         # statistics: packets, errors, drops
ip -d link show type bridge       # bridges (docker0 and friends)
ip -br link show type veth        # one end of each container's virtual cable
```

Reading `ip addr show`:

```
3: wlp4s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 8c:1d:96:aa:bb:cc brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.50/24 brd 192.168.1.255 scope global dynamic noprefixroute wlp4s0
       valid_lft 80234sec preferred_lft 80234sec
    inet6 fe80::…/64 scope link noprefixroute
```

| Part | Meaning |
|---|---|
| `UP` | administratively enabled. `LOWER_UP` = there is a carrier (cable connected, Wi-Fi associated) |
| `mtu 1500` | largest packet in bytes this link carries without fragmenting |
| `link/ether` | MAC address. `brd ff:ff:ff:ff:ff:ff` is the broadcast MAC |
| `inet …/24` | IPv4 address and prefix length |
| `dynamic` | obtained by DHCP. `valid_lft` is the lease time remaining |
| `scope global` / `link` / `host` | valid everywhere / on this link only / on this machine only |

Interface names: `lo` loopback, `enp3s0` or `eno1` Ethernet, `wlp4s0` Wi-Fi, `docker0` and `br-…` bridges, `veth…` container links, `tun`/`wg` VPN tunnels.

Older guides use `ifconfig`, `route`, `arp` and `netstat` from the `net-tools` package. They are deprecated and absent from a minimal Rocky install.

| Old | Current |
|---|---|
| `ifconfig` | `ip addr`, `ip link` |
| `route -n` | `ip route` |
| `arp -n` | `ip neigh` |
| `netstat -tulpn` | `ss -tulpn` |

### Ports and sockets: layer 4

An IP address finds a machine. A **port** (0–65535) finds a program on it. A connection is identified by five values: protocol, source IP, source port, destination IP, destination port.

| | TCP | UDP |
|---|---|---|
| Connection | yes: handshake, then a stream | none: independent datagrams |
| Reliability | retransmits, orders, controls flow | none; the application copes |
| Used by | HTTP, SSH, databases | DNS, DHCP, NTP, QUIC, VoIP |

Ports 0–1023 are "well known" and need root to listen on. Clients pick an **ephemeral** source port, on Linux from the range in `/proc/sys/net/ipv4/ip_local_port_range`.

```bash
sudo ss -tulpn                    # listening sockets and their processes (Lab 06)
ss -tan | head                    # all TCP sockets with state
ss -tn state established          # established connections: find your own SSH session
ss -s                             # summary counts
grep -wE '22|53|80|443|5432|7687' /etc/services | head
getent services ssh https
cat /proc/sys/net/ipv4/ip_local_port_range
```

TCP opens with a **three-way handshake** (SYN, SYN-ACK, ACK) and closes with FIN/ACK in each direction, or abruptly with RST. Lab 15 captures one.

```bash
nc -zv "$GW" 53                   # can I open a TCP connection to this port? (-z: just test, -v: say so)
nc -zv 127.0.0.1 22
nc -zv 127.0.0.1 1                # refused: the machine answered with RST. Nothing is listening.
```

"Connection refused" (an answer came back) and "timed out" (nothing came back) are different failures with different causes. Lab 15 depends on that distinction.

### All four layers in one command

```bash
curl -sv https://example.com -o /dev/null 2>&1 | grep -E '^\*|^[<>]' | head -30
```

Read it top to bottom: the name is resolved (DNS, application layer), `Trying <ip>:443` (IP and TCP), `Connected` (handshake done), TLS negotiation, then `> GET /` and `< HTTP/…` (HTTP).

```bash
curl -s -o /dev/null -w 'dns %{time_namelookup}s  tcp %{time_connect}s  tls %{time_appconnect}s  first-byte %{time_starttransfer}s  total %{time_total}s\n' https://example.com
```

That timing line splits a slow request into its layers. Keep it somewhere.

### Exercises 1

1. List every interface on the tower with its state and addresses. Which are physical, which are virtual, and what created each virtual one?
2. What is your main interface's MTU and MAC address? How many receive errors or drops has it recorded?
3. Find your own SSH connection in `ss`. What are the four address/port values?
4. Which well-known ports do `domain`, `ntp`, `postgresql` and `mysql` use? Answer from `/etc/services`.
5. Use the `curl -w` line against a site you run or use. Which phase dominates?

<details><summary>Solutions</summary>

```bash
ip -br addr; ip -br link show type veth; ip -br link show type bridge       # 1: docker0/br-*/veth* come from Docker
ip -s link show dev "$IF"                                                   # 2
ss -tn state established '( sport = :22 )'                                  # 3 (use your sshd port if it is not 22)
grep -wE '^(domain|ntp|postgresql|mysql)' /etc/services                     # 4: 53, 123, 5432, 3306
```

</details>

---

## 2. Subnetting

An IPv4 address is 32 bits, written as four decimal octets. A **prefix length** (`/24`) says how many leading bits identify the **network**; the rest identify the **host** within it. The **netmask** is the same information written as an address: `/24` = `255.255.255.0`.

Within any subnet, two addresses are reserved: the first (all host bits 0) is the **network address**, and the last (all host bits 1) is the **broadcast address**. Usable hosts = 2^(host bits) − 2.

Two machines can talk directly (layer 2) only if they are in the same subnet. Otherwise the packet goes to a router. Getting the mask wrong is therefore a connectivity fault, not a cosmetic one.

| Prefix | Netmask | Addresses | Usable hosts | Block size in the changing octet |
|---|---|---|---|---|
| /8 | 255.0.0.0 | 16,777,216 | 16,777,214 | |
| /16 | 255.255.0.0 | 65,536 | 65,534 | |
| /20 | 255.255.240.0 | 4,096 | 4,094 | 16 (third octet) |
| /22 | 255.255.252.0 | 1,024 | 1,022 | 4 (third octet) |
| /23 | 255.255.254.0 | 512 | 510 | 2 (third octet) |
| **/24** | 255.255.255.0 | 256 | 254 | |
| /25 | 255.255.255.128 | 128 | 126 | 128 |
| /26 | 255.255.255.192 | 64 | 62 | 64 |
| /27 | 255.255.255.224 | 32 | 30 | 32 |
| /28 | 255.255.255.240 | 16 | 14 | 16 |
| /29 | 255.255.255.248 | 8 | 6 | 8 |
| /30 | 255.255.255.252 | 4 | 2 | 4 |
| /31 | 255.255.255.254 | 2 | 2 (point-to-point links) | |
| /32 | 255.255.255.255 | 1 | a single host | |

### The method

For `192.168.10.77/27`:

1. **Which octet changes?** /27 is past 24, so the fourth.
2. **Block size** = 256 − mask octet = 256 − 224 = **32**. (Or 2^(32−27) = 32.)
3. **Blocks** start at multiples of 32: 0, 32, 64, 96, 128…
4. **Which block holds 77?** 64 ≤ 77 < 96. So the network is `192.168.10.64`.
5. **Broadcast** = next block − 1 = `192.168.10.95`.
6. **Hosts** = `.65` to `.94`, which is 30 addresses.

For a prefix between /16 and /24 the same steps apply to the **third** octet. `10.4.200.9/20`: mask octet 240, block size 16, blocks 0, 16 … 192, 208. 200 falls in 192–207. Network `10.4.192.0`, broadcast `10.4.207.255`, hosts `10.4.192.1` to `10.4.207.254` (4,094).

Check yourself with the tool, after doing it by hand:

```bash
ipcalc 192.168.10.77/27
ipcalc 10.4.200.9/20
ipcalc "$CIDR"                    # your own network
```

### Ranges to recognise

| Range | Meaning |
|---|---|
| `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16` | **private** (RFC 1918): not routed on the internet; reused everywhere behind NAT |
| `127.0.0.0/8` | loopback: this machine |
| `169.254.0.0/16` | link-local: an interface gave itself an address because **DHCP failed**. Also the cloud metadata address `169.254.169.254` |
| `100.64.0.0/10` | carrier-grade NAT; also used by Tailscale |
| `172.17.0.0/16` | Docker's default bridge network |
| `10.42.0.0/16`, `10.43.0.0/16` | k3s defaults for pods and services |
| `192.0.2.0/24`, `198.51.100.0/24`, `203.0.113.0/24` | reserved for documentation and examples |
| `0.0.0.0/0` | "everything": the default route. As a listen address, "all interfaces" |
| `224.0.0.0/4` | multicast |

Overlapping private ranges are a practical problem: if your home LAN, a Docker network and a VPN all use `192.168.1.0/24`, routing between them breaks. Choose container and cluster ranges deliberately.

**IPv6** in one paragraph: 128-bit addresses in hexadecimal, `::` abbreviating one run of zeros. Subnets are almost always `/64`. `::1` is loopback, `fe80::/10` is link-local (every interface has one), `fd00::/8` is the private range. There is no broadcast and no ARP; neighbour discovery uses ICMPv6. `ip -6 addr`, `ip -6 route`, `ping -6`.

### Exercises 2

Do these on paper, then verify with `ipcalc`.

1. `172.16.5.130/25`: network, broadcast, usable range, host count.
2. `10.20.30.40/28`: the same.
3. `192.168.100.200/22`: the same.
4. Are `10.0.1.200` and `10.0.2.5` in the same subnet with a /23 mask? With /22?
5. Split `192.168.50.0/24` into four equal subnets. List each network address and its usable range.
6. What is the smallest subnet that holds 50 hosts? 500 hosts? 2 hosts?
7. Your tower: what are the network and broadcast addresses of its LAN, and how many hosts fit?
8. A host shows the address `169.254.12.7`. What happened?

<details><summary>Solutions</summary>

1. Block 128. Network `172.16.5.128`, broadcast `172.16.5.255`, hosts `.129`–`.254`, 126.
2. Block 16; 40 is in 32–47. Network `10.20.30.32`, broadcast `10.20.30.47`, hosts `.33`–`.46`, 14.
3. Third octet, block 4; 100 is in 100–103. Network `192.168.100.0`, broadcast `192.168.103.255`, hosts `192.168.100.1`–`192.168.103.254`, 1,022.
4. /23: blocks of 2 in the third octet. `10.0.1.200` is in `10.0.0.0/23` (0–1), `10.0.2.5` is in `10.0.2.0/23` (2–3). **Different.** /22: blocks of 4, both in `10.0.0.0/22` (0–3). **Same.**
5. Four subnets need 2 more bits: /26. `192.168.50.0/26` (.1–.62), `192.168.50.64/26` (.65–.126), `192.168.50.128/26` (.129–.190), `192.168.50.192/26` (.193–.254).
6. /26 (62 hosts). /23 (510 hosts). /30 (2 hosts), or /31 on a point-to-point link.
7. `ipcalc "$CIDR"`.
8. It asked for a DHCP lease, got no answer, and assigned itself a link-local address. It can reach nothing beyond its own segment. Check the DHCP server, the cable or the Wi-Fi association.

</details>

---

## 3. Ethernet and ARP / RARP

On a local network, delivery is by **MAC address**: 48 bits, written `8c:1d:96:aa:bb:cc`, assigned to the network interface. The first three bytes identify the manufacturer.

IP works with IP addresses, the link layer with MAC addresses. **ARP** (Address Resolution Protocol) connects them: "who has 192.168.1.1? Tell 192.168.1.50". The request is broadcast to the whole segment, the owner replies with its MAC, and the answer is cached.

When a machine sends a packet:

- **destination in my subnet** → ARP for the destination itself, send the frame to its MAC
- **destination elsewhere** → ARP for the **gateway**, send the frame to the gateway's MAC, with the final destination's IP still inside

That is why the subnet mask matters, and why a host with a wrong gateway can reach its neighbours and nothing else.

```bash
ip neigh                          # the neighbour (ARP) table
ip neigh show dev "$IF"
ip -s neigh show dev "$IF"        # with usage counters
```

```
192.168.1.1 dev wlp4s0 lladdr a4:91:b1:11:22:33 REACHABLE
```

| State | Meaning |
|---|---|
| `REACHABLE` | confirmed recently |
| `STALE` | not confirmed lately; will be re-checked on next use. Normal |
| `DELAY`, `PROBE` | being re-checked |
| `FAILED` | no reply to ARP: **the host is down, or not on this segment** |
| `INCOMPLETE` | request sent, no reply yet |
| `PERMANENT` | a static entry |

Watch ARP on the wire. Terminal 1:

```bash
sudo timeout 20 tcpdump -i "$IF" -nn -e arp
```

Terminal 2:

```bash
sudo ip neigh flush dev "$IF"     # forget cached entries; the kernel re-resolves at once, your session survives
ping -c 2 "$GW"
arping -c 3 -I "$IF" "$GW"        # ARP-level ping: works even when the target blocks ICMP
```

In terminal 1 you see `Request who-has <gw> tell <you>` sent to `ff:ff:ff:ff:ff:ff`, then `Reply <gw> is-at <mac>`.

What goes wrong at this layer: **duplicate IP addresses** (two machines answer ARP for one address; connections flap, and `arping -D` detects it), and **ARP spoofing** (a hostile machine on the LAN answers for the gateway's address).

**RARP** (Reverse ARP) asked the opposite question: "my MAC is X, what is my IP?", for diskless machines at boot. It is obsolete, replaced first by BOOTP and then by **DHCP**, which does the same and far more. You need to know the name and what replaced it.

IPv6 has no ARP. **NDP** (Neighbour Discovery Protocol, part of ICMPv6) does the job: `ip -6 neigh`.

### Link details

```bash
sudo dnf install -y ethtool iw 2>/dev/null
sudo ethtool "$IF" 2>/dev/null | grep -E 'Speed|Duplex|Link detected'       # wired links
nmcli -f IN-USE,SSID,BSSID,CHAN,RATE,SIGNAL,SECURITY dev wifi list           # Wi-Fi: the network in use is starred
iw dev "$IF" link 2>/dev/null                                                # signal in dBm, bitrate
cat /proc/net/wireless
```

For a server on Wi-Fi, signal strength and bitrate are a layer-1 health check. Below about −70 dBm, expect retransmissions and stalls that look like application problems.

### Virtual Ethernet: how containers are wired

```bash
ip -br link show type bridge      # docker0: a software switch
bridge link                       # which veth interfaces are plugged into which bridge
docker network ls
docker network inspect bridge --format '{{range .IPAM.Config}}{{.Subnet}} via {{.Gateway}}{{end}}'
ip neigh show dev docker0         # the host's ARP entries for containers
```

Each container gets a `veth` pair: one end inside the container named `eth0`, the other on the host plugged into a bridge. To the host, containers are machines on a small private LAN whose router is the host itself.

### Exercises 3

1. What is your gateway's MAC address, and what state is its neighbour entry in?
2. How many devices on your LAN has the tower talked to recently?
3. Ping an unused address in your own subnet. What does `ip neigh` show for it afterwards? Compare with pinging an unused address in a *different* subnet. Explain the difference.
4. Which veth interface belongs to which running container? (Hint: inside the container, `cat /sys/class/net/eth0/iflink` gives the host-side interface index.)

<details><summary>Solutions</summary>

```bash
ip neigh show "$GW"                                             # 1
ip neigh show dev "$IF" | grep -vc FAILED                       # 2
ping -c 2 -W 1 "${IP%.*}.253"; ip neigh | grep "${IP%.*}.253"   # 3 (assumes a /24 where .253 is unused): FAILED or INCOMPLETE.
                                                                #    The kernel ARPed for it and got no reply.
ping -c 2 -W 1 203.0.113.5; ip neigh | grep 203.0.113.5         #    no entry. It was never ARPed for:
                                                                #    the packet went to the gateway's MAC.
# 4
for c in $(docker ps -q); do
  idx=$(docker exec "$c" cat /sys/class/net/eth0/iflink 2>/dev/null)
  printf '%s  %s\n' "$(docker inspect -f '{{.Name}}' "$c")" "$(ip -o link | awk -F': ' -v i="$idx" '$1 == i {print $2}')"
done
```

</details>

---

## 4. DHCP

**DHCP** (Dynamic Host Configuration Protocol) gives a machine its network settings automatically. The exchange is four messages, "DORA", over UDP ports 67 (server) and 68 (client):

| Step | Direction | Meaning |
|---|---|---|
| **D**iscover | client → broadcast | "Is there a DHCP server? I have no address." |
| **O**ffer | server → client | "You may have 192.168.1.50." |
| **R**equest | client → broadcast | "I accept 192.168.1.50 from that server." |
| **A**ck | server → client | "Yours for 24 hours. Here are your mask, router, DNS servers." |

A lease carries the address, subnet mask, default gateway, DNS servers, lease time, and optionally a domain name, NTP servers and more. The client renews at half the lease time by asking the same server directly. If renewal keeps failing until the lease expires, the address is dropped.

On Rocky, **NetworkManager** is the DHCP client.

```bash
nmcli dev show "$IF" | grep -E 'IP4|DHCP4'
nmcli -f ipv4.method,ipv4.addresses,ipv4.gateway,ipv4.dns con show "$CON"     # "auto" = DHCP, "manual" = static
nmcli -f DHCP4 con show "$CON"                                                # every option the server sent
ip -4 addr show dev "$IF" | grep -E 'dynamic|valid_lft'                       # "dynamic" and time remaining
sudo journalctl -u NetworkManager --no-pager | grep -i dhcp | tail -10
sudo ls /var/lib/NetworkManager/ 2>/dev/null                                  # stored leases
```

### Servers and DHCP

A server should be found at the same address every time. Two ways:

- **DHCP reservation**: tell the router "this MAC always gets this IP". The server stays on DHCP. Central, and survives a reinstall of the server. Usually the better choice at home.
- **Static configuration** on the server. Independent of the DHCP server being up, but must not collide with the DHCP pool.

Static configuration with `nmcli`, **for reference; do not run this over SSH on the tower**. A wrong value disconnects you.

```bash
# sudo nmcli con mod "$CON" ipv4.method manual ipv4.addresses 192.168.1.50/24 ipv4.gateway 192.168.1.1 ipv4.dns "1.1.1.1 9.9.9.9"
# sudo nmcli con up "$CON"            # apply
# sudo nmcli con mod "$CON" ipv4.method auto ipv4.addresses "" ipv4.gateway "" ipv4.dns ""    # back to DHCP
```

Profiles are stored in `/etc/NetworkManager/system-connections/*.nmconnection`. `nmtui` is the text-menu equivalent. If you manage the tower with Ansible, the `community.general.nmcli` module makes the same changes.

To see DORA on the wire you would capture `sudo tcpdump -i "$IF" -nn 'port 67 or port 68'` while a client joins. Your phone reconnecting to Wi-Fi will not show up (the tower only hears broadcasts), but its Discover and Request are broadcast, so you may catch those.

Containers do not use DHCP. Docker assigns addresses itself from each network's subnet.

### Exercises 4

1. Is the tower's address static or from DHCP? If DHCP: which server gave it, how long is the lease, and how long is left?
2. Which DNS servers and which default gateway did DHCP supply?
3. Does your router reserve the tower's address? How would you tell from the tower alone, and what would you check on the router?
4. What address would the tower end up with if the DHCP server disappeared and the lease ran out? What would still work?

<details><summary>Solutions</summary>

```bash
nmcli -f ipv4.method con show "$CON"; nmcli -f DHCP4 con show "$CON" | grep -E 'dhcp_server_identifier|dhcp_lease_time|expiry'   # 1
nmcli -f DHCP4 con show "$CON" | grep -E 'domain_name_servers|routers'                                                           # 2
```

3. From the tower alone you cannot tell; a long-lived client usually keeps being offered the same address either way. Check the router's DHCP page for a static lease bound to the tower's MAC (`ip -br link show dev "$IF"`).
4. NetworkManager would keep retrying and the interface would have no IPv4 address (some clients fall back to `169.254.x.x`). Services bound to localhost and container-to-container traffic would carry on; anything needing the LAN or internet, including the tunnel and your SSH session, would not.

</details>

---

## 5. IP routing

Every packet leaving a machine is matched against the **routing table** to decide which interface it leaves by and which next hop it is handed to.

```bash
ip route
```

```
default via 192.168.1.1 dev wlp4s0 proto dhcp src 192.168.1.50 metric 600
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1
192.168.1.0/24 dev wlp4s0 proto kernel scope link src 192.168.1.50 metric 600
```

| Part | Meaning |
|---|---|
| `default` | `0.0.0.0/0`: matches everything; used when nothing more specific does |
| `via 192.168.1.1` | hand the packet to this next-hop router |
| `dev wlp4s0` | send it out of this interface |
| no `via` (`scope link`) | **directly connected**: destinations are on this segment, so ARP for them |
| `proto dhcp` / `kernel` / `static` | who installed the route: DHCP, the kernel when an address was added, or an administrator |
| `src` | the source address to use |
| `metric` | preference when two routes are otherwise equal; lower wins |

**Longest prefix wins.** A packet to `172.17.0.5` matches both `default` (/0) and `172.17.0.0/16`; the /16 is more specific, so it goes to `docker0`.

Ask the kernel what it would do, without sending anything:

```bash
ip route get 1.1.1.1              # via the gateway, out of the main interface
ip route get "$GW"                # directly connected: no "via"
ip route get 172.17.0.2           # out of docker0
ip route get 127.0.0.1            # local
ip route show table local | head  # addresses the machine itself owns
ip rule                           # which routing tables are consulted, in order
```

`ip route get` is the first command for "why can't I reach X": it shows the interface, next hop and source address the kernel will choose.

### A temporary route

`203.0.113.0/24` is reserved for examples, so routing it somewhere affects nothing real. Routes added with `ip route` live in memory and vanish at reboot.

```bash
ip route get 203.0.113.5                          # currently: the default route
sudo ip route add 203.0.113.0/24 via "$GW" dev "$IF" metric 50
ip route | grep 203.0.113
ip route get 203.0.113.5                          # now matches the specific route
sudo ip route add blackhole 198.51.100.0/24       # a route that silently discards
ping -c 1 -W 1 198.51.100.1                       # fails immediately: "Invalid argument" or no reply
sudo ip route del 203.0.113.0/24
sudo ip route del blackhole 198.51.100.0/24
```

Persistent static routes belong in the NetworkManager profile: `nmcli con mod "$CON" +ipv4.routes "10.50.0.0/16 192.168.1.254"`.

### The tower is a router

A Linux machine forwards packets between its interfaces when `ip_forward` is on. Docker turns it on, because the host routes between the container bridges and the LAN.

```bash
sysctl net.ipv4.ip_forward                        # 1
sudo nft list ruleset 2>/dev/null | grep -i masquerade | head
```

A container at `172.17.0.2` sending to the internet: the packet goes to its gateway `172.17.0.1` (the host's `docker0`), the host's routing table chooses the default route, and on the way out a **masquerade** (source NAT) rule rewrites the source address to the host's LAN address, because nothing outside knows how to reach `172.17.0.0/16`. Your home router then does the same again toward the internet. Section 7 looks at those rules.

### Path

```bash
tracepath -n 1.1.1.1 | head -12                   # each router on the way, and the path MTU. Lab 15 goes deeper.
```

### Exercises 5

1. How many routes does the tower have? Which were created by Docker?
2. For each destination, predict the interface and next hop, then check with `ip route get`: your gateway, `8.8.8.8`, `172.17.0.9`, your own address, `127.0.0.53`.
3. Add a route sending `192.0.2.0/24` via your gateway, verify, and delete it. Then add a /32 host route for `192.0.2.7` to `blackhole`, check which route `192.0.2.7` and `192.0.2.8` each take, and clean up.
4. If the default route were deleted, what would still be reachable? (Do not test this over SSH unless your laptop is on the same subnet as the tower.)

<details><summary>Solutions</summary>

```bash
ip route | wc -l; ip route | grep -E 'docker|br-'                # 1
for d in "$GW" 8.8.8.8 172.17.0.9 "$IP" 127.0.0.53; do ip route get "$d" | head -1; done     # 2
sudo ip route add 192.0.2.0/24 via "$GW"; sudo ip route add blackhole 192.0.2.7/32           # 3
ip route get 192.0.2.7; ip route get 192.0.2.8      # .7 hits the /32 blackhole (longest prefix); .8 takes the /24
sudo ip route del blackhole 192.0.2.7/32; sudo ip route del 192.0.2.0/24
```

4. Everything on directly connected networks: the LAN subnet, the Docker networks, loopback. Nothing beyond the gateway. The error for other destinations would be "Network is unreachable", produced locally and instantly.

</details>

---

## 6. DNS resolution

Applications ask for names. Something must turn `example.com` into an address. On Linux that is a chain:

```
application → getaddrinfo() → /etc/nsswitch.conf decides the order:
                                 1. files  → /etc/hosts
                                 2. dns    → the servers in /etc/resolv.conf
                                 3. myhostname → the machine's own name
```

```bash
grep '^hosts' /etc/nsswitch.conf          # the order of sources
cat /etc/hosts                            # static names; consulted first
cat /etc/resolv.conf                      # which DNS servers, and search domains
nmcli dev show "$IF" | grep DNS           # where resolv.conf's contents came from
```

`/etc/resolv.conf` on Rocky is **written by NetworkManager** (the comment at its top says so). Hand edits are overwritten at the next connection change. DNS settings are changed in the connection profile:

```bash
# sudo nmcli con mod "$CON" ipv4.dns "1.1.1.1 9.9.9.9" ipv4.ignore-auto-dns yes && sudo nmcli con up "$CON"
```

Rocky does not run a local caching resolver by default (Ubuntu does: `systemd-resolved`, where resolv.conf points at `127.0.0.53` and `resolvectl status` shows the real servers).

### Two kinds of lookup tool

| Tool | Uses | Answers the question |
|---|---|---|
| `getent hosts NAME` | the full chain above, like an application | "what will my programs get?" |
| `dig`, `host`, `nslookup` | DNS only, ignoring `/etc/hosts` | "what does DNS say?" |

Use `getent` to reproduce an application's view and `dig` to investigate DNS itself. When they disagree, `/etc/hosts` is why.

```bash
getent hosts example.com
getent ahosts example.com                 # all addresses, IPv4 and IPv6
dig example.com
```

Reading `dig`:

```
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 4242
;; flags: qr rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1

;; QUESTION SECTION:
;example.com.                   IN      A

;; ANSWER SECTION:
example.com.            214     IN      A       23.215.0.136

;; Query time: 18 msec
;; SERVER: 192.168.1.1#53(192.168.1.1) (UDP)
```

| Part | Meaning |
|---|---|
| `status: NOERROR` | the query worked. `NXDOMAIN` = the name does not exist. `SERVFAIL` = the resolver could not get an answer. `REFUSED` = it will not answer you |
| `ANSWER SECTION` | name, **TTL** in seconds (how long it may be cached), class, record type, value |
| `Query time` | how long the resolver took |
| `SERVER` | which resolver answered |

```bash
dig +short example.com                    # just the answer
dig +noall +answer example.com            # the answer lines with TTLs
dig example.com AAAA +short               # IPv6 address
dig example.com MX +short                 # mail servers
dig example.com NS +short                 # authoritative name servers
dig example.com TXT +short                # text records: SPF, domain verification
dig example.com SOA +short                # zone's start of authority: primary server, serial, timers
dig www.github.com CNAME +short           # an alias
dig -x 1.1.1.1 +short                     # reverse: address to name (PTR)
dig @1.1.1.1 example.com +short           # ask a specific resolver
dig @9.9.9.9 example.com +short
dig +trace example.com | grep -vE '^;|^$' | tail -15     # walk the delegation from the root yourself
host example.com
```

| Record | Maps |
|---|---|
| `A` / `AAAA` | name → IPv4 / IPv6 address |
| `CNAME` | name → another name |
| `MX` | domain → mail servers, with priority |
| `NS` | zone → its authoritative servers |
| `TXT` | name → arbitrary text |
| `PTR` | address → name |
| `SOA` | zone metadata |
| `SRV` | service → host and port |
| `CAA` | which certificate authorities may issue for the domain |

### How resolution works

Your machine asks a **recursive resolver** (your router, your ISP, 1.1.1.1). If it has no cached answer it asks a **root** server, which refers it to the **TLD** servers (`.com`), which refer it to the domain's **authoritative** servers, which answer. `dig +trace` performs those steps from your machine. Answers are cached for their TTL at every level, which is why a changed record takes up to the old TTL to be seen everywhere.

### Overriding with /etc/hosts

```bash
echo "203.0.113.10  lab13.test" | sudo tee -a /etc/hosts
getent hosts lab13.test                   # 203.0.113.10: applications see it
dig +short lab13.test                     # nothing: DNS knows no such name
ping -c 1 -W 1 lab13.test | head -1       # ping resolves through NSS, like any application
sudo sed -i '/lab13\.test/d' /etc/hosts   # remove it
```

Legitimate uses: testing a site before its DNS changes, naming machines on a small LAN. It is also the first place to look when one machine resolves a name differently from every other.

### Search domains and short names

`search home.lan` in resolv.conf makes `ping nas` try `nas.home.lan`. The `ndots` option controls when a name is tried as-is first. This matters in Kubernetes, where `ndots:5` and several search domains turn one lookup of an external name into many.

### DNS in Docker

```bash
docker run --rm busybox cat /etc/resolv.conf
```

On the default bridge a container copies the host's resolvers. On a **user-defined** network it gets `127.0.0.11`, Docker's embedded DNS, which resolves other containers on that network **by name** and forwards the rest. That is why Compose services reach each other as `http://api:8000` (Lab 16).

### Exercises 6

1. Which resolver does the tower use? Time a lookup through it, and through `1.1.1.1` and `9.9.9.9`. Then repeat the same query: what happened to the query time and the TTL?
2. Look up a domain you run a site on. Do the `A` records point at your own public address, or at someone else's network? Who are its name servers? What does that tell you about how traffic reaches the site?
3. What is the reverse (PTR) name of your default gateway? Of `8.8.8.8`?
4. Resolve a name that does not exist. What status does `dig` report, and what does `getent hosts` return as an exit status?
5. Make `getent` and `dig` disagree about a name, show it, and undo it.
6. Run `dig +trace` for a domain and list the chain of servers consulted.

<details><summary>Solutions</summary>

```bash
# 1
grep nameserver /etc/resolv.conf
for s in "" @1.1.1.1 @9.9.9.9; do dig $s example.com | grep -E 'Query time|SERVER'; done
dig +noall +answer example.com; sleep 3; dig +noall +answer example.com
# the second query is faster (cached at the resolver) and its TTL has counted down

# 2
dig +short A yourdomain.example; dig +short NS yourdomain.example; curl -s https://ifconfig.me; echo
# If the A records are not your public IP, a proxy or tunnel provider is answering for the name,
# and visitors never connect to your home address directly.

# 3
dig -x "$GW" +short; dig -x 8.8.8.8 +short

# 4
dig nosuchname-xyz.example.com | grep status       # NXDOMAIN
getent hosts nosuchname-xyz.example.com; echo $?   # 2

# 5: the /etc/hosts demonstration above

# 6
dig +trace example.com | grep -E 'Received .* from'
```

</details>

---

## 7. Netfilter

**Netfilter** is the packet-processing framework inside the Linux kernel. It offers five **hooks** in a packet's path through the machine, where rules can inspect, accept, drop, or rewrite it.

```
                          ┌──────────────┐
  packet in ─▶ PREROUTING ┤ routing      ├─▶ FORWARD ─▶ POSTROUTING ─▶ packet out
                          │ decision     │                   ▲
                          └──────┬───────┘                   │
                                 ▼                           │
                               INPUT                      OUTPUT
                                 │                           ▲
                                 ▼                           │
                           local process ────────────────────┘
```

| Hook | Sees | Typical use |
|---|---|---|
| PREROUTING | every arriving packet, before routing | **DNAT**: port forwarding, published container ports |
| INPUT | packets addressed to this machine | the host firewall: which services are reachable |
| FORWARD | packets passing through to somewhere else | routers, **traffic to and from containers** |
| OUTPUT | packets created by local processes | egress filtering |
| POSTROUTING | everything leaving, after routing | **SNAT / masquerade** |

**Connection tracking** (conntrack) makes the firewall stateful. Every packet is classified as `new`, `established`, `related` or `invalid`. One early rule accepts `established,related`, and after that you only write rules about which **new** connections to allow.

```bash
sudo cat /proc/sys/net/netfilter/nf_conntrack_count /proc/sys/net/netfilter/nf_conntrack_max
```

### The tools in front of it

| Tool | What it is | On Rocky 9 |
|---|---|---|
| **nftables** (`nft`) | the current kernel rule engine and its command | the real backend |
| **iptables** | the previous generation, with fixed tables (`filter`, `nat`, `mangle`, `raw`) and chains named after the hooks | the `iptables` command is `iptables-nft`: it translates into nftables. Still what Docker and most tutorials speak |
| **firewalld** (`firewall-cmd`) | a daemon offering zones and named services, which writes nftables rules for you | **the supported way to manage the host firewall** |

One kernel engine, three ways to talk to it. Change rules through firewalld; **read** them with `nft` when you need the truth.

### firewalld

firewalld sorts interfaces and source addresses into **zones**, each with a trust level and a list of what is allowed in.

```bash
sudo firewall-cmd --state
sudo firewall-cmd --get-default-zone
sudo firewall-cmd --get-active-zones            # which zone each interface is in. Look for a "docker" zone.
sudo firewall-cmd --list-all                    # the default zone, in full
sudo firewall-cmd --list-all --zone=docker 2>/dev/null
sudo firewall-cmd --get-zones
sudo firewall-cmd --get-services | tr ' ' '\n' | wc -l
sudo firewall-cmd --info-service=ssh            # a "service" is a named set of ports
sudo firewall-cmd --info-service=https
```

```
public (active)
  target: default
  interfaces: wlp4s0
  services: cockpit dhcpv6-client ssh
  ports:
  forward: yes
  masquerade: no
  rich rules:
```

`target: default` means anything not listed is rejected. Every entry under `services` and `ports` is a hole you should be able to justify. (`cockpit`, a web admin console on port 9090, is allowed by default on Rocky whether or not it is installed.)

**Runtime versus permanent.** firewalld holds two configurations. A plain command changes the **runtime** one: effective at once, gone at the next reload or reboot. Adding `--permanent` changes the **saved** one: no effect until `--reload`. The safe habit is to test at runtime and make permanent only what worked.

| Task | Command |
|---|---|
| allow a service, now | `sudo firewall-cmd --add-service=https` |
| allow a port, now | `sudo firewall-cmd --add-port=8080/tcp` |
| …for five minutes only | add `--timeout=5m` |
| remove | `--remove-service=…`, `--remove-port=…` |
| make it permanent | repeat with `--permanent`, or `sudo firewall-cmd --runtime-to-permanent` |
| apply the permanent config | `sudo firewall-cmd --reload` |
| allow only from one subnet | `sudo firewall-cmd --add-rich-rule='rule family=ipv4 source address=192.168.1.0/24 port port=8080 protocol=tcp accept'` |
| put a source in a more trusted zone | `sudo firewall-cmd --zone=trusted --add-source=192.168.1.0/24` |
| emergency: drop everything | `sudo firewall-cmd --panic-on` (this also cuts your SSH session) |

### Open a port safely

Terminal 1, a throwaway web server on all interfaces:

```bash
cd ~/labwork/13 && echo "hello from the tower" > index.html
python3 -m http.server 8099 --bind 0.0.0.0
```

Terminal 2, on the tower: confirm it listens.

```bash
ss -tln | grep 8099
curl -s http://127.0.0.1:8099/            # works: loopback traffic is not filtered
```

From **another device on the same LAN** (your laptop if it is at home, or a phone browser), open `http://<tower-ip>:8099/`. It fails: firewalld rejects it, and `curl` reports "No route to host" (the text for an ICMP "administratively prohibited" reply). If you are not on the LAN, follow along with the rule listings.

Open it for five minutes. The timeout is your safety net: if you forget, it closes without you.

```bash
sudo firewall-cmd --add-port=8099/tcp --timeout=5m
sudo firewall-cmd --list-ports
```

Retry from the other device: it works. Then close it and stop the server (Ctrl-C in terminal 1):

```bash
sudo firewall-cmd --remove-port=8099/tcp
sudo firewall-cmd --list-ports
```

For a real service the permanent sequence is: `--add-service=NAME` at runtime, test, `--runtime-to-permanent`. In Ansible it is the `ansible.posix.firewalld` module with `permanent: true` and `immediate: true`.

### Reading the real rules

```bash
sudo nft list tables                            # one table from firewalld, others created by Docker through iptables-nft
sudo nft list table inet firewalld | less       # what firewalld generated
sudo nft list ruleset | grep -c .               # the size of the whole thing
sudo iptables -S | head -30                     # filter table as iptables commands
sudo iptables -L -n -v | head -40               # with packet and byte counters per rule
sudo iptables -t nat -S                         # NAT rules: Docker's DNAT and MASQUERADE
```

### Docker and the firewall: read this twice

Docker manages its own netfilter rules, and they are **not subject to your firewalld zone**. When you publish a container port with `-p 7474:7474`:

1. Docker adds a **DNAT** rule at PREROUTING, rewriting the destination to the container's address.
2. The packet is now addressed to a container, so it is **forwarded**. It never reaches the INPUT hook, where firewalld's `services` and `ports` lists apply.
3. Docker's own rules in the FORWARD path accept it.

**Result: `-p 7474:7474` is reachable from every network the host is on, even though firewalld lists no such port.** `firewall-cmd --list-all` does not show it.

```bash
docker ps --format 'table {{.Names}}\t{{.Ports}}'
sudo iptables -t nat -S DOCKER 2>/dev/null       # one DNAT line per published port
sudo ss -tlnp | grep -E 'docker-proxy'           # the listeners Docker holds on the host
```

In the `Ports` column, `0.0.0.0:7474->7474/tcp` is published on all interfaces. `127.0.0.1:7474->7474/tcp` is reachable only from the host itself.

For anything reached only through a local reverse proxy, an SSH tunnel or the Cloudflare tunnel (all of which connect from the host), publish on loopback:

```
-p 127.0.0.1:7474:7474            # docker run
ports: ["127.0.0.1:7474:7474"]    # Compose
```

To filter published ports by source instead, rules go in the `DOCKER-USER` chain, which Docker leaves for administrators and evaluates before its own.

### Exercises 7

1. Which zone is your main interface in? What does that zone allow in? Is every entry needed?
2. Is there a `docker` zone? Which interfaces are in it and what is its target?
3. List every container port published on the tower, and classify each as loopback-only or all-interfaces. For each all-interfaces one: does it need to be?
4. Open port 8099 with a 2-minute timeout, confirm it is listed, wait, and confirm it disappears by itself.
5. Find the masquerade rule that lets containers reach the internet. Which source range does it apply to?
6. What is the difference in effect between `--add-port=8080/tcp`, `--permanent --add-port=8080/tcp`, and the two followed by `--reload`?
7. A packet arrives for a published container port. Name, in order, the netfilter hooks it passes through. Then do the same for a packet to the host's `sshd`.

<details><summary>Solutions</summary>

```bash
sudo firewall-cmd --get-active-zones; sudo firewall-cmd --list-all                  # 1
sudo firewall-cmd --list-all --zone=docker                                          # 2: target ACCEPT, interfaces docker0 / br-*
docker ps --format '{{.Names}}\t{{.Ports}}'                                         # 3
sudo firewall-cmd --add-port=8099/tcp --timeout=2m; sudo firewall-cmd --list-ports; sleep 125; sudo firewall-cmd --list-ports   # 4
sudo iptables -t nat -S POSTROUTING | grep MASQUERADE                               # 5: -s 172.17.0.0/16 ! -o docker0
```

6. Runtime only: works now, lost at reload or reboot. Permanent only: saved, **not active** until reload. Permanent then reload: saved and active, and the reload discards any runtime-only rules.
7. Container port: PREROUTING (DNAT) → routing → FORWARD → POSTROUTING. Host sshd: PREROUTING → routing → INPUT → the sshd process.

</details>

---

## 8. SSH

SSH gives an encrypted, authenticated channel to another machine. Two separate identity checks happen on every connection.

**1. The server proves itself to you**, with its **host key**. The first time, you are shown the key's fingerprint and asked to accept it; it is stored in `~/.ssh/known_hosts`. Every later connection compares. A mismatch produces the loud `REMOTE HOST IDENTIFICATION HAS CHANGED` warning: either the server was reinstalled, or something is intercepting you.

```bash
for k in /etc/ssh/ssh_host_*_key.pub; do ssh-keygen -lf "$k"; done      # this server's fingerprints
ssh-keygen -F localhost 2>/dev/null | head -3                           # is a host already in known_hosts?
# ssh-keygen -R hostname                                                # forget a host, after a legitimate rebuild
```

Verifying the fingerprint through another channel on first connection is what actually defeats interception. Typing "yes" without looking does not.

**2. You prove yourself to the server**, ideally with a **key pair**. The private key stays on your machine. The public key goes in `~/.ssh/authorized_keys` on the server. The server sends a challenge only the private key can answer. Nothing secret crosses the network, and there is no password to guess.

### Keys

Work with a throwaway key so that your real ones are untouched. The tower plays both client and server.

```bash
cd ~/labwork/13
cp -p ~/.ssh/authorized_keys authorized_keys.bak 2>/dev/null           # safety copy of your real access

ssh-keygen -t ed25519 -f ./lab_key -N '' -C "lab13-throwaway"          # -N '': no passphrase, for the lab only
ls -l lab_key lab_key.pub                                              # private 600, public 644
cat lab_key.pub                                                        # type, key, comment: safe to share
ssh-keygen -lf lab_key.pub                                             # its fingerprint
```

`ed25519` is the type to use. A real key should have a passphrase, so that a stolen file is not a stolen identity; `ssh-agent` saves you retyping it:

```bash
# eval "$(ssh-agent -s)"; ssh-add ~/.ssh/id_ed25519; ssh-add -l
```

Install the public key and log in with it:

```bash
cat lab_key.pub >> ~/.ssh/authorized_keys          # what ssh-copy-id does for you on a remote host
chmod 700 ~/.ssh; chmod 600 ~/.ssh/authorized_keys

ssh -i ./lab_key -o IdentitiesOnly=yes localhost 'echo "logged in as $(whoami) on $(hostname)"'
```

Add `-p PORT` if your sshd is not on 22. If sshd does not listen on loopback or refuses `localhost`, use the tower's own address (`$IP`) here and as `HostName` in the config file below. Accept the host key when asked, having compared it with the fingerprints printed above.

To a real remote host the equivalent is `ssh-copy-id -i ~/.ssh/id_ed25519.pub user@host`.

sshd silently ignores keys when permissions are too open. The requirements: home directory not writable by group or other, `~/.ssh` mode 700, `authorized_keys` mode 600, and on Rocky the right SELinux label (`restorecon -Rv ~/.ssh`).

When key login fails, ask the client what happened, then the server:

```bash
ssh -v -i ./lab_key -o IdentitiesOnly=yes localhost true 2>&1 | grep -E 'Offering|Server accepts|Authenticat|identity file'
sudo journalctl -u sshd -n 10 --no-pager
```

### The client config file

`~/.ssh/config` stores per-host settings, so that `ssh tower` replaces a long command line. The lab uses a separate file through `-F` to leave your own config alone.

```bash
cat > ~/labwork/13/ssh_config <<EOF
Host labtower
    HostName localhost
    User $USER
    IdentityFile ~/labwork/13/lab_key
    IdentitiesOnly yes
    ServerAliveInterval 30
    ServerAliveCountMax 4

Host *
    AddKeysToAgent yes
EOF
chmod 600 ~/labwork/13/ssh_config
ssh -F ~/labwork/13/ssh_config labtower hostname
```

| Option | Use |
|---|---|
| `HostName`, `User`, `Port`, `IdentityFile` | the basics |
| `IdentitiesOnly yes` | offer only the named key, not everything in the agent |
| `ServerAliveInterval 30` | send a keepalive every 30 s: stops idle sessions dying on NAT and flaky Wi-Fi |
| `ProxyJump bastion` | reach this host through another (`ssh -J bastion target`) |
| `ProxyCommand` | reach it through an arbitrary command, which is how SSH over a Cloudflare tunnel works (`cloudflared access ssh --hostname %h`) |
| `LocalForward 7474 127.0.0.1:7474` | open a tunnel every time you connect |
| `ControlMaster auto`, `ControlPath`, `ControlPersist 10m` | reuse one connection for many sessions: makes Ansible and repeated `scp` much faster |
| `ForwardAgent yes` | lets the remote host use your agent. Avoid except to hosts you fully trust; prefer `ProxyJump` |

On your laptop, an entry for the tower with `ServerAliveInterval` and a `LocalForward` or two is worth five minutes.

### Running commands and tunnels

```bash
S="ssh -F $HOME/labwork/13/ssh_config"
$S labtower 'uptime; df -h / | tail -1'            # run commands and return
$S labtower 'sudo -n true' && echo "passwordless sudo" || echo "sudo needs a password or a tty (use: ssh -t)"
echo "local text" | $S labtower 'cat > /tmp/lab13-piped.txt'; cat /tmp/lab13-piped.txt; rm /tmp/lab13-piped.txt
```

**Port forwarding** carries other connections inside the SSH session.

| Form | Meaning | Use |
|---|---|---|
| `-L 8098:127.0.0.1:8099` | **local**: connections to port 8098 on *my* machine come out on the server and go to `127.0.0.1:8099` there | reach a service that listens only on the server's loopback |
| `-R 9000:127.0.0.1:3000` | **remote**: connections to port 9000 on the *server* come back to my machine's port 3000 | show a local dev server to a remote machine |
| `-D 1080` | **dynamic**: a SOCKS proxy through the server | browse as if from the server |
| `-N` | no remote command, just the forwards | |
| `-f` | go to the background after authenticating | |

Start the test web server bound to loopback only, then tunnel to it:

```bash
cd ~/labwork/13
python3 -m http.server 8099 --bind 127.0.0.1 >/dev/null 2>&1 &  WEB=$!
$S -N -L 8098:127.0.0.1:8099 labtower &  TUN=$!
sleep 1
curl -s http://127.0.0.1:8098/                     # "hello from the tower", through the tunnel
kill "$TUN" "$WEB"
```

Here both ends are the same machine, so it only proves the mechanism. The real use is from your laptop:

```bash
# on the laptop:  ssh -N -L 7474:127.0.0.1:7474 -L 7687:127.0.0.1:7687 tower
# then open http://localhost:7474 in the laptop's browser
```

That reaches a database console published only on the tower's loopback (section 7), without opening any firewall port and without exposing it publicly. It is the standard way to reach admin interfaces.

### The server side

sshd's configuration is `/etc/ssh/sshd_config` plus drop-ins in `/etc/ssh/sshd_config.d/*.conf`, which Rocky includes **first**. For most options the first value read wins, so a drop-in can silently override what you see in the main file. Always check the **effective** configuration:

```bash
ls /etc/ssh/sshd_config.d/
sudo sshd -T | grep -Ei '^(port|listenaddress|permitrootlogin|passwordauthentication|pubkeyauthentication|kbdinteractiveauthentication|permitemptypasswords|maxauthtries|allowusers|allowgroups|x11forwarding|allowtcpforwarding|clientaliveinterval|logingracetime) '
```

| Setting | Recommended | Why |
|---|---|---|
| `PermitRootLogin` | `no` | every action traces to a named account plus sudo |
| `PasswordAuthentication` | `no` | removes password guessing entirely |
| `KbdInteractiveAuthentication` | `no` | closes the other route to password prompts |
| `PubkeyAuthentication` | `yes` | |
| `PermitEmptyPasswords` | `no` | |
| `AllowUsers` or `AllowGroups` | the accounts that need SSH | everyone else is refused before authentication |
| `MaxAuthTries` | `3` | |
| `X11Forwarding` | `no` | unused on a headless server |
| `ClientAliveInterval` / `ClientAliveCountMax` | e.g. `300` / `2` | drops dead sessions |

Your Ansible hardening has probably set most of these. The command above is how you verify what a playbook actually achieved.

The procedure for changing sshd on a remote machine, **for reference**:

```bash
# sudo cp -a /etc/ssh/sshd_config /etc/ssh/sshd_config.bak.$(date +%F)
# sudoedit /etc/ssh/sshd_config.d/10-local.conf        # your overrides in a low-numbered drop-in
# sudo sshd -t                                         # syntax check: silence means OK
# sudo systemctl reload sshd                           # reload: existing sessions are not disconnected
# then, FROM A NEW TERMINAL, prove you can still log in before closing the old one
```

Changing the port additionally needs the SELinux port label and the firewall opened **first**: `sudo semanage port -a -t ssh_port_t -p tcp 2222`, then `sudo firewall-cmd --add-port=2222/tcp`.

### Exercises 8

1. Print the tower's ed25519 host key fingerprint. On your laptop, find the entry for the tower in `known_hosts` and confirm it is the same key (`ssh-keygen -F tower -l`).
2. Audit the tower's effective sshd configuration against the table. List anything that differs and decide whether it is intentional.
3. Break key login on purpose: `chmod 644 lab_key` and connect. Read the error. Fix it.
4. Using the lab config, run `df -h` on `labtower` and save the output to a local file in one command.
5. Start `python3 -m http.server 8097 --bind 127.0.0.1`, forward local port 9097 to it through SSH, fetch a page through the tunnel, then tear everything down.
6. On your laptop, write a `~/.ssh/config` entry for the tower with keepalives and a tunnel to one loopback-only service. Test it.
7. Which accounts on the tower have an `authorized_keys` file, and how many keys does each hold?

<details><summary>Solutions</summary>

```bash
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub                                         # 1
sudo sshd -T | grep -Ei '^(permitrootlogin|passwordauthentication|kbdinteractive|x11forwarding|maxauthtries|allow)'   # 2
chmod 644 lab_key; ssh -i ./lab_key -o IdentitiesOnly=yes localhost true                 # 3: "UNPROTECTED PRIVATE KEY FILE"
chmod 600 lab_key
ssh -F ~/labwork/13/ssh_config labtower 'df -h' > tower-df.txt                           # 4
python3 -m http.server 8097 --bind 127.0.0.1 >/dev/null 2>&1 &  WEB=$!                   # 5
ssh -F ~/labwork/13/ssh_config -N -L 9097:127.0.0.1:8097 labtower &  TUN=$!
sleep 1; curl -sI http://127.0.0.1:9097/ | head -1; kill "$TUN" "$WEB"
sudo sh -c 'for f in /home/*/.ssh/authorized_keys /root/.ssh/authorized_keys; do [ -f "$f" ] && echo "$f: $(grep -cE "^(ssh|ecdsa|sk)-" "$f")"; done'   # 7
```

</details>

---

## 9. File transfer

| Tool | Protocol | Best for |
|---|---|---|
| `scp` | SSH | a quick one-off copy of a file or two |
| `sftp` | SSH | interactive browsing and transfer |
| **`rsync`** | SSH (or local) | anything large, repeated, or that must resume: **the default choice** |
| `curl`, `wget` | HTTP(S), FTP | downloading from web servers and APIs |
| `tar \| ssh` | SSH | streaming a tree without an intermediate file |
| `nc` | raw TCP | quick unencrypted transfer on a trusted network |

Plain FTP sends passwords and data unencrypted. Do not use it. NFS and SMB are for *mounting* a remote filesystem, a different job from copying files.

Make some data. `labtower` (the tower itself, over SSH) stands in for the remote machine.

```bash
cd ~/labwork/13
mkdir -p src/sub && seq 1 50000 > src/big.txt && echo "small" > src/sub/small.txt && ln -s big.txt src/link
S_OPTS="-F $HOME/labwork/13/ssh_config"
ssh $S_OPTS labtower 'mkdir -p ~/labwork/13/remote'
```

### scp

```bash
scp $S_OPTS src/big.txt labtower:labwork/13/remote/            # local → remote (relative paths start in the remote home)
scp $S_OPTS labtower:labwork/13/remote/big.txt ./pulled.txt    # remote → local
scp $S_OPTS -r src labtower:labwork/13/remote/src-scp          # -r directory
scp $S_OPTS -p src/big.txt labtower:/tmp/lab13-big.txt         # -p keep times and mode
# -P 2222 port (capital P, unlike ssh)    -C compress    -l 8000 limit to 8000 kbit/s
```

### sftp

```bash
sftp $S_OPTS labtower
```

| Command | Action |
|---|---|
| `ls`, `cd`, `pwd`, `mkdir` | on the remote side |
| `lls`, `lcd`, `lpwd` | on the local side |
| `get FILE`, `get -r DIR` | download |
| `put FILE`, `put -r DIR` | upload |
| `reget`, `reput` | resume |
| `bye` | quit |

Scripted: `sftp -b batchfile host`. Giving a user **only** file transfer, with no shell, is done in `sshd_config` with `ForceCommand internal-sftp` and `ChrootDirectory`.

### rsync

rsync compares source and destination and sends only the differences. An interrupted transfer continues where it stopped, and repeating a finished one sends almost nothing.

```bash
rsync -av -e "ssh $S_OPTS" src/ labtower:labwork/13/remote/src-rsync/
rsync -av -e "ssh $S_OPTS" src/ labtower:labwork/13/remote/src-rsync/     # again: nothing to send
echo "changed" >> src/sub/small.txt
rsync -av -e "ssh $S_OPTS" src/ labtower:labwork/13/remote/src-rsync/     # only small.txt
```

| Option | Meaning |
|---|---|
| `-a` | archive: recursive, preserving permissions, times, symlinks, owner and group (owner needs root) |
| `-v` / `-P` | verbose / progress per file, and keep partial files (`--partial --progress`) |
| `-z` | compress in transit: worth it on slow links, wasteful on a LAN or for compressed data |
| `-n` | **dry run**: show what would happen |
| `--delete` | remove files at the destination that are gone from the source: makes a true mirror |
| `--exclude='*.log'`, `--exclude-from=FILE` | skip matches |
| `-H -A -X` | also preserve hard links, ACLs, extended attributes (SELinux labels) |
| `--bwlimit=5000` | limit to 5000 KiB/s |
| `-e "ssh -p 2222"` | options for the SSH transport |
| `--checksum` | compare contents, not size and time (slow, thorough) |

**The trailing slash on the source** decides the result:

```bash
rsync -avn -e "ssh $S_OPTS" src/ labtower:labwork/13/remote/A/     # the CONTENTS of src land in A/
rsync -avn -e "ssh $S_OPTS" src  labtower:labwork/13/remote/A/     # the directory itself lands as A/src/
```

**Always dry-run `--delete` first.** With source and destination swapped, or the slash wrong, `--delete` empties the wrong side.

```bash
rm src/link
rsync -avn --delete -e "ssh $S_OPTS" src/ labtower:labwork/13/remote/src-rsync/     # shows "deleting link"
rsync -av  --delete -e "ssh $S_OPTS" src/ labtower:labwork/13/remote/src-rsync/
```

A realistic use from your laptop, pulling a backup from the tower:

```bash
# rsync -avzP --delete tower:/var/backups/neo4j/ ~/backups/tower-neo4j/
```

### Downloading: curl and wget

```bash
curl -fsSLO https://dl.rockylinux.org/pub/rocky/RPM-GPG-KEY-Rocky-9      # save under the remote name
cmp RPM-GPG-KEY-Rocky-9 /etc/pki/rpm-gpg/RPM-GPG-KEY-Rocky-9 && echo "identical to the installed key"
curl -fsSL -o key.txt https://dl.rockylinux.org/pub/rocky/RPM-GPG-KEY-Rocky-9   # choose the name
curl -sI https://dl.rockylinux.org/pub/rocky/RPM-GPG-KEY-Rocky-9 | head -5      # headers only
wget -q https://dl.rockylinux.org/pub/rocky/RPM-GPG-KEY-Rocky-9 -O key2.txt
```

| curl option | Meaning |
|---|---|
| `-O` / `-o FILE` | save under the remote name / a name you give |
| `-L` | follow redirects |
| `-f` | **fail** (non-zero exit) on HTTP errors, instead of saving the error page |
| `-sS` | silent, but still show errors |
| `-C -` | resume a partial download |
| `-I` | headers only |
| `-H 'Header: v'`, `-d DATA`, `-X METHOD` | API calls |
| `--retry 5` | retry transient failures |

`curl -fsSL` is the combination for scripts. `wget -c` resumes; `wget -r -np` mirrors a directory tree.

### Verify what you moved

```bash
sha256sum src/big.txt
ssh $S_OPTS labtower 'sha256sum labwork/13/remote/src-rsync/big.txt'
sha256sum src/big.txt > SHA256SUMS; sha256sum -c SHA256SUMS
```

Downloads from the internet usually publish a checksum file, signed. Checking it is how you know you received what was published.

### Streaming

```bash
tar -czf - src | ssh $S_OPTS labtower 'tar -xzf - -C labwork/13/remote/ && ls labwork/13/remote/src'
ssh $S_OPTS labtower 'sudo -n tar -czf - -C /etc ssh 2>/dev/null' > etc-ssh.tgz; ls -l etc-ssh.tgz     # needs passwordless sudo
```

### Exercises 9

1. Copy `/etc/os-release` from "remote" to local three ways: `scp`, `sftp` in batch mode, `rsync`. Verify with checksums that all three are identical.
2. Create a 100 MB file with `head -c 100M /dev/urandom > blob`. Start an `rsync -P` of it to `labtower`, interrupt with Ctrl-C partway, and rerun. What happens? Then try the same with `scp`.
3. Mirror `src/` to a remote directory, delete one local file, add another, and bring the remote into line with one command. Dry-run first and read the output.
4. Exclude every `.txt` file under `sub/` from a sync.
5. Download a file with `curl` from a URL that returns 404, once without `-f` and once with. Compare the exit status and what ended up on disk.
6. From your laptop: rsync a project directory to the tower, excluding `.git` and `node_modules`. Time a second run.

<details><summary>Solutions</summary>

```bash
# 1
scp $S_OPTS labtower:/etc/os-release ./os1
echo "get /etc/os-release os2" | sftp $S_OPTS -b - labtower
rsync -e "ssh $S_OPTS" labtower:/etc/os-release ./os3
sha256sum os1 os2 os3

# 2
head -c 100M /dev/urandom > blob
rsync -P -e "ssh $S_OPTS" blob labtower:labwork/13/remote/        # Ctrl-C, then run again: it resumes from the partial file
scp $S_OPTS blob labtower:labwork/13/remote/blob2                 # Ctrl-C, rerun: it starts again from zero
rm blob

# 3
rm src/sub/small.txt; echo new > src/new.txt
rsync -avn --delete -e "ssh $S_OPTS" src/ labtower:labwork/13/remote/src-rsync/
rsync -av  --delete -e "ssh $S_OPTS" src/ labtower:labwork/13/remote/src-rsync/

# 4
rsync -avn --exclude='sub/*.txt' -e "ssh $S_OPTS" src/ labtower:labwork/13/remote/src-rsync/

# 5
curl -sO https://dl.rockylinux.org/pub/rocky/NO-SUCH-FILE; echo "exit $?"; ls -l NO-SUCH-FILE      # exit 0, an HTML error page saved
rm NO-SUCH-FILE
curl -fsO https://dl.rockylinux.org/pub/rocky/NO-SUCH-FILE; echo "exit $?"; ls NO-SUCH-FILE        # exit 22, nothing saved

# 6  (on the laptop)
# time rsync -az --exclude='.git' --exclude='node_modules' ./myproject/ tower:projects/myproject/
```

</details>

---

## Challenge: the whole journey

Document, with one command and its output for each step, everything that happens when the tower runs `curl https://example.com/`. Write it to `~/labwork/13/journey.md`.

1. **Name**: which sources are consulted in what order, which resolver is asked, and what it answers.
2. **Route**: which interface, next hop and source address the kernel picks for that destination.
3. **Link**: whose MAC address the frame is sent to, and why it is not the web server's.
4. **Transport**: the TCP connection as seen in `ss` (run a slow request in another terminal: `curl -s --limit-rate 1 https://example.com/ >/dev/null`).
5. **Firewall**: why the outgoing connection and its replies are allowed, in terms of conntrack states.
6. **NAT**: at which point the source address is rewritten on the way to the internet, and by which device.
7. **Application**: the TLS and HTTP exchange from `curl -v`.
8. **Timing**: how long each phase took.

Then answer the same eight questions for a request from the internet to a site you host on the tower, going the other way. Where does the firewall matter, and where does it not?

<details><summary>Commands that produce the evidence</summary>

```bash
grep '^hosts' /etc/nsswitch.conf; grep nameserver /etc/resolv.conf; dig +noall +answer example.com    # 1
A=$(dig +short example.com | head -1); ip route get "$A"                                              # 2
ip neigh show "$GW"            # 3: the gateway's MAC. The server is not on this segment, so the frame goes to the router.
ss -tn dst "$A"                # 4: while the slow curl runs
sudo nft list table inet firewalld | grep -m3 -E 'ct state'                                           # 5
# 6: no NAT on the tower for its own traffic; the home router masquerades at its WAN interface
curl -sv https://example.com/ -o /dev/null 2>&1 | grep -E '^\* (Connected|SSL|ALPN|  subject)|^[<>] ' # 7
curl -s -o /dev/null -w 'dns %{time_namelookup} tcp %{time_connect} tls %{time_appconnect} total %{time_total}\n' https://example.com/   # 8
```

For the inbound direction through a tunnel: the tunnel daemon on the tower makes an **outbound** connection to the provider, and visitors' requests travel back down it. No inbound port is open, the INPUT chain never sees those requests as new connections, and the daemon reaches your service on loopback. That is why loopback-only published ports are sufficient, and why the host firewall and Docker's rules matter mostly for traffic from your own LAN.

</details>

## Check yourself

<details><summary>Which address changes at every router hop, and which stays the same end to end?</summary>
The MAC addresses change: each hop puts the packet in a new frame addressed to the next device. The IP addresses stay the same (apart from NAT).
</details>

<details><summary>Network, broadcast and usable hosts of <code>10.1.2.200/26</code>?</summary>
Block size 64; 200 falls in 192–255. Network <code>10.1.2.192</code>, broadcast <code>10.1.2.255</code>, hosts <code>.193</code>–<code>.254</code>, 62 of them.
</details>

<details><summary>A host can ping its neighbours but nothing outside the LAN. Two likely causes?</summary>
A missing or wrong default route, or a wrong gateway address. A wrong subnet mask can produce the same symptom for part of the address space.
</details>

<details><summary>An interface has the address <code>169.254.7.12</code>. What does that tell you?</summary>
DHCP failed and the interface assigned itself a link-local address.
</details>

<details><summary>Why can <code>getent hosts name</code> and <code>dig name</code> give different answers?</summary>
<code>getent</code> follows <code>nsswitch.conf</code>, so it checks <code>/etc/hosts</code> first. <code>dig</code> queries DNS servers only.
</details>

<details><summary>What is the difference between "Connection refused" and "Connection timed out"?</summary>
Refused: the destination machine replied with a TCP reset, so it is reachable and nothing is listening on that port (or a firewall actively rejected). Timed out: nothing came back at all, so the packet was dropped by a firewall, the host is down, or routing is broken.
</details>

<details><summary>You published a container port with <code>-p 8080:80</code>. firewalld does not list port 8080. Is it reachable from the LAN?</summary>
Yes. Docker's DNAT sends the traffic through the FORWARD path, which firewalld's zone rules for the host do not govern. Publish on <code>127.0.0.1</code> if it should not be.
</details>

<details><summary>What is the difference between a runtime and a permanent firewalld change?</summary>
Runtime changes apply immediately and are lost at reload or reboot. Permanent changes are saved and take effect only after <code>--reload</code>.
</details>

<details><summary>What does <code>ssh -L 5432:127.0.0.1:5432 server</code> do?</summary>
It listens on port 5432 on your own machine and carries each connection through the SSH session to <code>127.0.0.1:5432</code> as seen from the server, giving you access to a service that only listens on the server's loopback.
</details>

<details><summary>Why is <code>rsync -a src/ dest/</code> different from <code>rsync -a src dest/</code>?</summary>
With the trailing slash, the contents of <code>src</code> are placed in <code>dest</code>. Without it, the directory itself is placed there, giving <code>dest/src/</code>.
</details>

<details><summary>What must you do before reloading sshd after a config change on a remote server?</summary>
Run <code>sshd -t</code> to validate the syntax, keep the current session open, and confirm a new login works from a second terminal before closing anything.
</details>

## Cleanup

```bash
source ~/labwork/13/vars.sh 2>/dev/null
pkill -u "$USER" -f 'http.server 809' 2>/dev/null
pkill -u "$USER" -f 'labwork/13/ssh_config' 2>/dev/null

# authorized_keys: remove the lab key, then prove nothing else changed
sed -i '/lab13-throwaway/d' ~/.ssh/authorized_keys
diff ~/.ssh/authorized_keys ~/labwork/13/authorized_keys.bak && echo "authorized_keys is back to its original content"

sudo sed -i '/lab13\.test/d' /etc/hosts
sudo firewall-cmd --remove-port=8099/tcp 2>/dev/null; sudo firewall-cmd --list-ports
for r in 203.0.113.0/24 192.0.2.0/24; do sudo ip route del "$r" 2>/dev/null; done
for r in 198.51.100.0/24 192.0.2.7/32; do sudo ip route del blackhole "$r" 2>/dev/null; done
ssh-keygen -R localhost >/dev/null 2>&1          # optional: forget the localhost host key you accepted
rm -f /tmp/lab13-*
```

Only after the `diff` reported no differences (and with your second SSH session still open until you have logged in afresh once):

```bash
rm -rf ~/labwork/13
```
