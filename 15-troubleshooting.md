# Lab 15 — Troubleshooting

**Roadmap topics:** ICMP · ping · traceroute · netstat · Packet Analysis
**Time:** about 4 hours
**Touches system state:** yes, in a contained way. The lab builds a private test network: a network namespace `labns` playing the part of a remote client, joined to the tower by a virtual cable (`veth-lab`, addresses `10.99.0.1` and `10.99.0.2`), plus a throwaway web server on port 8099 and one runtime-only firewall rule. Faults are injected only into that test network. Your real interface, routes, DNS and firewall configuration are not changed. One script creates all of it and one command removes it.

If your LAN or any Docker network already uses `10.99.0.0/24` (`ip route | grep 10.99`), change the addresses at the top of the script before running it.

## What you will be able to do

- Follow a fixed method from "it doesn't work" to the faulty layer.
- Say what each error message means: *Network is unreachable*, *Destination Host Unreachable*, *No route to host*, *Connection refused*, *Connection timed out*, *Name or service not known*.
- Use `ping`, `traceroute`, `mtr`, `ss`/`netstat` and `tcpdump` with intent, and read their output.
- Capture and read a TCP handshake, a refused connection, a firewall rejection and a DNS lookup.
- Diagnose injected faults without being told what was broken.

## Setup

```bash
mkdir -p ~/labwork/15 && cd ~/labwork/15
sudo dnf install -y iputils iproute traceroute mtr net-tools tcpdump nmap-ncat bind-utils curl

IF=$(ip route show default | awk '{print $5; exit}')
GW=$(ip route show default | awk '{print $3; exit}')
echo "interface=$IF gateway=$GW"
```

### The testbed

Save this script. Read the top half (`up`, `down`, `client`, `check`). **Do not read the `break` function**: section 7 uses it to hide faults from you.

```bash
cat > labnet <<'EOF'
#!/usr/bin/env bash
# labnet - a private client/server test network for Lab 15. Run with sudo.
#
#   labnet up              build the testbed and start the web server
#   labnet client CMD...   run CMD as the remote client (inside the namespace)
#   labnet check           try the request the "user" is complaining about
#   labnet break [N]       inject a fault (random, or number N)
#   labnet fix             restore the working state
#   labnet down            remove everything
set -euo pipefail

NS=labns
HOST_IF=veth-lab;  HOST_IP=10.99.0.1
NS_IF=veth-ns;     NS_IP=10.99.0.2
PORT=8099
WEBROOT=/var/tmp/labnet-www

[[ $EUID -eq 0 ]] || { echo "run with sudo" >&2; exit 1; }

nsx() { ip netns exec "$NS" "$@"; }
have_fw() { firewall-cmd --state > /dev/null 2>&1; }
fw() { if have_fw; then firewall-cmd -q "$@" || true; fi; }
exists() { ip netns list | grep -qw "$NS"; }

web_stop() { pkill -u nobody -f "http.server $PORT" 2>/dev/null || true; }
web_start() {                         # $1 = address to bind
  web_stop
  mkdir -p "$WEBROOT"
  echo "hello from the tower" > "$WEBROOT/index.html"
  chmod -R a+rX "$WEBROOT"
  ( cd "$WEBROOT" && exec setsid runuser -u nobody -- python3 -m http.server "$PORT" --bind "$1" > /dev/null 2>&1 ) &
  sleep 0.7
}
write_hosts() {                       # $1 = address for tower.lab
  mkdir -p "/etc/netns/$NS"
  printf '127.0.0.1 localhost\n%s tower.lab\n' "$1" > "/etc/netns/$NS/hosts"
}
baseline() {
  ip link set "$HOST_IF" up
  nsx ip link set lo up
  nsx ip link set "$NS_IF" up
  nsx ip addr flush dev "$NS_IF"
  nsx ip addr add "$NS_IP/24" dev "$NS_IF"
  nsx ip route replace default via "$HOST_IP"
  write_hosts "$HOST_IP"
  web_start 0.0.0.0
  fw --add-port="$PORT/tcp"
}

case "${1:-}" in
  up)
    exists && { echo "already up (use: labnet fix)"; exit 0; }
    ip netns add "$NS"
    ip link add "$HOST_IF" type veth peer name "$NS_IF"
    ip link set "$NS_IF" netns "$NS"
    ip addr add "$HOST_IP/24" dev "$HOST_IF"
    baseline
    echo "testbed up: client $NS_IP (namespace $NS)  <-->  tower $HOST_IP, web server on port $PORT"
    ;;
  client)
    exists || { echo "testbed is not up" >&2; exit 1; }
    shift; nsx "$@"
    ;;
  check)
    exists || { echo "testbed is not up" >&2; exit 1; }
    if nsx curl -s -m 6 -o /dev/null "http://tower.lab:$PORT/"; then echo "PASS: the client can fetch http://tower.lab:$PORT/"
    else echo "FAIL: the client cannot fetch http://tower.lab:$PORT/"; fi
    ;;
  fix)
    exists || { echo "testbed is not up" >&2; exit 1; }
    baseline; echo "restored"
    ;;
  down)
    fw --remove-port="$PORT/tcp"
    web_stop
    ip netns del "$NS" 2>/dev/null || true
    ip link del "$HOST_IF" 2>/dev/null || true
    rm -rf "/etc/netns/$NS" "$WEBROOT"
    echo "testbed removed"
    ;;
  break)
    exists || { echo "testbed is not up" >&2; exit 1; }
    baseline
    max=6; have_fw || max=5
    n=${2:-$(( RANDOM % max + 1 ))}
    # ------------------------------------------------------------------
    # Spoilers below. Stop reading here.
    # ------------------------------------------------------------------
    case "$n" in
      1) nsx ip addr flush dev "$NS_IF"; nsx ip addr add 10.99.1.2/24 dev "$NS_IF" ;;
      2) write_hosts 10.99.0.9 ;;
      3) web_start 127.0.0.1 ;;
      4) web_stop ;;
      5) ip link set "$HOST_IF" down ;;
      6) fw --remove-port="$PORT/tcp" ;;
      *) echo "no such fault: $n" >&2; exit 2 ;;
    esac
    echo "a fault has been injected. Start with: sudo ./labnet check"
    ;;
  *)
    sed -n '2,9p' "$0" | sed 's/^# \{0,1\}//'; exit 2
    ;;
esac
EOF
chmod +x labnet

sudo firewall-cmd --state                                   # "running" is assumed below
sudo ./labnet up
sudo ./labnet check                                         # PASS
sudo ./labnet client curl -s http://tower.lab:8099/         # hello from the tower
sudo ./labnet client ip -br addr                            # the client's view: lo and veth-ns
ip -br addr show dev veth-lab                               # the tower's end of the cable
```

If firewalld is not running on your tower, the testbed still works, but the firewall rejections described below will not appear and fault 6 is skipped. If your default zone *drops* instead of rejecting, the rejections show up as timeouts.

A **network namespace** is a separate copy of the network stack: its own interfaces, addresses, routes, neighbour table and firewall rules. It is the kernel feature that gives each container its own network (Lab 16). Here it gives you a "remote machine" whose packets really cross an interface and pass through the tower's firewall, which traffic from the tower to itself never does.

The client can reach only the tower, not the internet. Internet-facing examples below are run from the tower itself.

Shorthand used from here on:

```bash
c() { sudo ~/labwork/15/labnet client "$@"; }      # run a command as the client
c ping -c 2 10.99.0.1
```

---

## 1. A method

Random poking wastes time and sometimes makes things worse. Work in a fixed order and let each result eliminate a layer.

1. **State the symptom precisely.** What, from where, to where, since when, and the exact error text. "The API is down" becomes "from my laptop, `curl https://api…/health` times out after 30 s; it worked yesterday".
2. **Ask what changed.** A deploy, an update, a reboot, a config edit, a certificate expiry. `dnf history`, `journalctl --since`, `last reboot`, your shell history.
3. **Test from more than one place.** From the server itself, from another host on the LAN, from outside. Where the result changes is where the fault is.
4. **Climb the ladder**, bottom up. Stop at the first rung that fails.
5. **Change one thing at a time**, and re-test with the same command.
6. **Write down what it was.**

### The ladder

| # | Question | Command | A failure here means |
|---|---|---|---|
| 1 | Is the link up? | `ip -br link` | cable, Wi-Fi association, interface down, driver |
| 2 | Do I have an address? | `ip -br addr` | DHCP failed (`169.254.x.x` or nothing), wrong static config |
| 3 | Can I reach my gateway? | `ping -c 3 GATEWAY`, `ip neigh` | wrong subnet or gateway, ARP failing, local network fault |
| 4 | Can I reach the internet by address? | `ping -c 3 1.1.1.1` | missing default route, router or ISP fault |
| 5 | Can I resolve names? | `getent hosts example.com`, `dig example.com` | resolver unreachable or misconfigured: **"it's always DNS"** |
| 6 | Can I reach the port? | `nc -zv HOST PORT` | firewall (timeout or prohibited), nothing listening (refused), wrong bind address |
| 7 | Does the application answer correctly? | `curl -v URL` | TLS, HTTP errors, the application itself: read its logs |

And on the **server** side, when clients cannot connect:

| # | Question | Command |
|---|---|---|
| a | Is the process running? | `systemctl status UNIT`, `docker ps` |
| b | Is it listening, and on which address? | `sudo ss -tlnp \| grep PORT` |
| c | Does it answer locally? | `curl -v http://127.0.0.1:PORT/` |
| d | Does the firewall allow it? | `sudo firewall-cmd --list-all` |
| e | Do packets arrive at all? | `sudo tcpdump -i IF -nn port PORT` |
| f | What does it log? | `journalctl -u UNIT -e`, `docker logs` |

Step (c) versus a remote test splits the problem in half immediately: works locally and fails remotely means bind address, firewall or network. Fails locally means the application.

### What each error means

Learn these six. Each one names a layer.

| Message | Who said it | Meaning |
|---|---|---|
| `Name or service not known` / `Could not resolve host` | your resolver | DNS: rung 5 |
| `Network is unreachable` | **your own kernel**, instantly | no route to that destination: rungs 2–4 |
| `Destination Host Unreachable` (in ping) | your own machine, or a router | ARP got no answer for the next hop or the host: rung 3 |
| `No route to host` | an ICMP error arrived, or ARP failed | the host is unreachable, **or a firewall rejected the packet** with ICMP "administratively prohibited" |
| `Connection refused` | **the destination host** (TCP RST) | host reachable; nothing is listening on that port, or it is bound to another address |
| `Connection timed out` | nobody: silence | packets dropped by a firewall, host down, or routing broken on the way back |

Refused is good news: the network works. Timeout tells you the least.

### Walk the ladder once

```bash
c ip -br link                              # 1
c ip -br addr                              # 2
c ping -c 2 10.99.0.1                      # 3: for this client, the tower is the gateway
c getent hosts tower.lab                   # 5
c nc -zv tower.lab 8099                    # 6
c curl -sv http://tower.lab:8099/ 2>&1 | head -12      # 7
```

And from the tower to the real internet:

```bash
ip -br link show dev "$IF"; ip -br addr show dev "$IF"
ping -c 2 "$GW"; ping -c 2 1.1.1.1
getent hosts example.com
nc -zv example.com 443
curl -sI https://example.com | head -1
```

---

## 2. ICMP

**ICMP** (Internet Control Message Protocol) is IP's signalling channel. It carries no application data. It is how hosts and routers report problems and how diagnostic tools probe the network. It has no ports; messages have a **type** and a **code**.

| Type | Name | Sent when | Used by |
|---|---|---|---|
| 8 | Echo Request | someone pings | `ping` |
| 0 | Echo Reply | answering a ping | `ping` |
| 3 | Destination Unreachable | a packet could not be delivered. The code says why | error reporting |
| 11 | Time Exceeded | a packet's TTL reached zero at a router | `traceroute` |
| 5 | Redirect | a router knows a better first hop | routing |
| 12 | Parameter Problem | a malformed header | |

The codes of type 3 are the informative ones:

| Code | Meaning | Typical sender |
|---|---|---|
| 0 | network unreachable | a router with no route |
| 1 | host unreachable | the last router: ARP for the host failed |
| 3 | **port unreachable** | the destination host: no UDP listener on that port. (TCP uses RST instead) |
| 4 | **fragmentation needed** but "don't fragment" was set | a router with a smaller MTU. Path MTU discovery depends on this |
| 9, 10, 13 | **administratively prohibited** | a firewall configured to *reject* |

### ICMP and firewalls

"Block ICMP for security" is folklore that breaks things. Blocking type 3 code 4 breaks path MTU discovery: small packets work, large ones vanish, and you get connections that open and then hang. IPv6 cannot function without ICMPv6 at all. Blocking echo hides nothing from a port scanner. Rate-limit if you must; do not blanket-block.

```bash
sudo firewall-cmd --get-icmptypes | tr ' ' '\n' | head -20
sudo firewall-cmd --list-icmp-blocks              # usually empty
sysctl net.ipv4.icmp_echo_ignore_all              # 0: this host answers pings
sysctl net.ipv4.icmp_ratelimit
```

firewalld's default action is **reject**, which answers with ICMP "host prohibited" so that clients fail fast with a clear error. A **drop** policy says nothing, and clients wait for a timeout.

### Watch ICMP

Terminal 1:

```bash
sudo tcpdump -i veth-lab -nn icmp
```

Terminal 2:

```bash
c ping -c 2 10.99.0.1                             # echo request / echo reply
c nc -u -zv -w 2 10.99.0.1 9999                   # UDP to a port with no listener
c nc -zv -w 2 10.99.0.1 8098                      # TCP to a port the firewall does not allow
```

In terminal 1 you see `ICMP echo request` and `ICMP echo reply` pairs, then for the two refusals `ICMP host 10.99.0.1 unreachable - admin prohibited`. Both probes were stopped by firewalld before reaching any socket, and the client was told so. Stop the capture with Ctrl-C.

### Exercises 2

1. From the capture: which ICMP type and code did the firewall rejection use? (Add `-v` to tcpdump for detail.)
2. `ping 127.0.0.1`, `ping` your own LAN address, and `ping` the gateway. Which of these packets leave the machine?
3. What ICMP message makes `traceroute` work, and what would happen to traceroute through a router that never sends it?
4. A site loads its small pages but stalls on large downloads over a VPN. Which ICMP message is probably being blocked, and how would you test from the client?

<details><summary>Solutions</summary>

1. Type 3 (destination unreachable), code 10 (host administratively prohibited).
2. Only the gateway ping. Traffic to any of the machine's own addresses is delivered over loopback. Check with `ip route get <address>`: `local … dev lo`.
3. Type 11, time exceeded. That hop shows as `* * *`; later hops still appear if they answer.
4. Type 3 code 4, fragmentation needed. Test with `ping -M do -s 1472 host` and step the size down until it passes: the largest passing size plus 28 is the path MTU.

</details>

---

## 3. ping

`ping` sends ICMP echo requests and reports each reply. It answers three questions: is the host reachable at the IP level, how long is the round trip, and is anything being lost.

```bash
ping -c 4 1.1.1.1
```

```
64 bytes from 1.1.1.1: icmp_seq=1 ttl=57 time=11.8 ms
64 bytes from 1.1.1.1: icmp_seq=2 ttl=57 time=12.4 ms
…
4 packets transmitted, 4 received, 0% packet loss, time 3004ms
rtt min/avg/max/mdev = 11.812/12.106/12.431/0.253 ms
```

| Field | Meaning |
|---|---|
| `icmp_seq` | sequence number. Gaps = lost packets. Out of order = reordering |
| `ttl` | the reply's remaining time-to-live. Senders start at 64 (Linux, macOS), 128 (Windows) or 255 (network devices), minus one per router. 57 suggests a Linux-like host 7 hops away |
| `time` | round trip in milliseconds |
| `packet loss` | anything above 0% on a wired LAN is a fault. On Wi-Fi and across the internet, a little is normal |
| `mdev` | variation in round-trip time (jitter). High mdev = unstable link |

| Option | Meaning |
|---|---|
| `-c N` | send N and stop. **Always use it in scripts** |
| `-i S` | interval in seconds (below 0.2 needs root) |
| `-W S` | wait S seconds for each reply |
| `-w S` | overall deadline |
| `-s N` | payload size in bytes (default 56, giving 64 with the ICMP header) |
| `-M do` | set "don't fragment": for MTU testing |
| `-t N` | set the outgoing TTL |
| `-I IF` | send from this interface or address |
| `-4` / `-6` | force IPv4 / IPv6 |
| `-n` | do not look up names |
| `-q` | summary only |
| `-D` | print a timestamp on each line |
| `-A` | adaptive: as fast as replies return |

```bash
ping -c 3 -q "$GW"                       # quick health check of the first hop
ping -c 20 -i 0.2 -q "$GW"               # 20 probes in 4 seconds: a loss and jitter sample
ping -c 3 -W 1 203.0.113.1               # an address that will not answer: 100% loss
ping -c 1 -t 1 1.1.1.1                   # TTL 1: the first router answers "Time to live exceeded"
ping -c 1 no-such-host.invalid           # Name or service not known: a DNS failure, not a network one
ping -D -i 5 "$GW" | tee ping.log        # a timestamped log to leave running while you chase an intermittent fault (Ctrl-C)
```

### MTU testing

Ethernet's MTU is 1500 bytes. An ICMP echo carries 28 bytes of headers (20 IP + 8 ICMP), so the largest unfragmented payload is 1472.

```bash
ping -c 2 -M do -s 1472 "$GW"            # fits
ping -c 2 -M do -s 1473 "$GW"            # "message too long, mtu=1500": refused locally
ping -c 2 -M do -s 1472 1.1.1.1          # does the whole path carry 1500? If not, reduce until it passes
tracepath -n 1.1.1.1 | tail -3           # reports the path MTU
```

Tunnels, VPNs and PPPoE lines lower the usable MTU. The symptom of a mismatch is the classic "ping works, SSH connects, large transfers hang".

### What ping cannot tell you

- **No reply does not prove the host is down.** ICMP may be filtered while TCP works. Test the actual port: `nc -zv host 443`.
- **A reply does not prove the service works.** It proves the IP stack answers.
- Routers often give ICMP low priority, so ping time to a router overstates the delay traffic sees passing *through* it.

### Exercises 3

1. Measure 50 pings to your gateway at 0.2-second intervals. Report loss, average and mdev. On Wi-Fi, repeat while a large download runs. What changes?
2. Compare the TTLs in replies from your gateway, `1.1.1.1` and `8.8.8.8`. Estimate the hop count to each.
3. Find the path MTU from the tower to `1.1.1.1` with `ping -M do`.
4. Ping the client's address `10.99.0.2` from the tower, then `10.99.0.77`. Explain the two different failure outputs of the second (wait five seconds).
5. Write a one-line health check that prints `gateway OK` or `gateway DOWN` and returns a matching exit status.

<details><summary>Solutions</summary>

```bash
ping -c 50 -i 0.2 -q "$GW"                                              # 1: under load, mdev and max rise; some loss may appear
for h in "$GW" 1.1.1.1 8.8.8.8; do ping -c1 -W1 "$h" | grep -o 'ttl=[0-9]*'; done     # 2: 64 minus ttl, or 255 minus ttl for a router
for s in 1472 1464 1452 1420 1400; do ping -c1 -W1 -M do -s $s 1.1.1.1 > /dev/null 2>&1 && { echo "passes at $s: MTU $((s + 28))"; break; }; done   # 3
ping -c 3 10.99.0.77      # 4: "Destination Host Unreachable" from 10.99.0.1 (your own address): ARP for .77 got no answer.
ip neigh show dev veth-lab   #    and the neighbour entry is FAILED
ping -c 2 -W 1 -q "$GW" > /dev/null && echo "gateway OK" || { echo "gateway DOWN"; false; }    # 5
```

</details>

---

## 4. traceroute

Every IP packet carries a **TTL**. Each router decrements it, and a router that brings it to zero discards the packet and sends back ICMP *time exceeded*. `traceroute` exploits that: it sends probes with TTL 1, then 2, then 3… and each reply reveals one more router on the path.

```bash
traceroute -n 1.1.1.1
```

```
 1  192.168.1.1     1.2 ms   1.0 ms   1.1 ms
 2  10.20.0.1       9.8 ms   9.5 ms  10.1 ms
 3  * * *
 4  203.0.113.45   11.2 ms  11.0 ms  11.4 ms
 5  1.1.1.1        11.9 ms  11.7 ms  12.0 ms
```

Each line is one hop, with three probes and their round-trip times.

| You see | It means |
|---|---|
| `* * *` at one hop, later hops answer | that router does not send (or rate-limits) ICMP errors. **Not a fault** |
| `* * *` from some hop **to the end** | packets are being dropped from that point, or the destination ignores the probe type. Retry with `-I` or `-T` before concluding anything |
| latency jumps at a hop **and stays high** after | the delay is real and begins at that link |
| latency high at **one** hop only, normal after | that router is slow to generate ICMP. Ignore it |
| several addresses on one line | load balancing: probes took different paths |
| `!H`, `!N`, `!X` after a time | host unreachable, network unreachable, administratively prohibited |
| the same hops repeating | a routing loop |

Linux `traceroute` sends **UDP** probes to high ports by default, and many firewalls drop those. Know the alternatives:

| Option | Probe type | Note |
|---|---|---|
| (default) | UDP to ports 33434 and up | often filtered |
| `-I` | ICMP echo, like Windows `tracert` | needs root |
| `-T -p 443` | TCP SYN to a port | needs root. Passes most firewalls, and tests the path **your application's traffic takes** |
| `-n` | no reverse DNS | faster |
| `-q 1` | one probe per hop | faster |
| `-w 1` | wait 1 s per probe | faster on dead paths |
| `-m 20` | maximum hops | |
| `-4` / `-6` | address family | |

```bash
traceroute -n -q 1 -w 1 1.1.1.1
sudo traceroute -n -I 1.1.1.1
sudo traceroute -n -T -p 443 example.com
tracepath -n 1.1.1.1                     # no root needed; also discovers the path MTU
```

The path shown is the **outbound** path. Replies may return by a different route, and traceroute cannot see that.

### mtr

`mtr` combines traceroute and ping: it probes every hop continuously and keeps loss and latency statistics per hop. It is the tool for intermittent problems and for evidence to send to an ISP.

```bash
mtr -n 1.1.1.1                           # interactive: q to quit
mtr -n -r -w -c 30 1.1.1.1               # report mode: 30 cycles, then print a table
sudo mtr -n -r -w -c 30 -T -P 443 example.com     # TCP probes to port 443
```

```
HOST: tower              Loss%   Snt   Last   Avg  Best  Wrst StDev
  1.|-- 192.168.1.1       0.0%    30    1.1   1.3   0.9   4.2   0.6
  2.|-- 10.20.0.1         0.0%    30    9.7   9.9   9.1  14.0   0.9
  3.|-- ???              100.0    30    0.0   0.0   0.0   0.0   0.0
  4.|-- 203.0.113.45      0.0%    30   11.1  11.3  10.8  13.9   0.5
  5.|-- 1.1.1.1           0.0%    30   11.8  12.0  11.5  15.2   0.7
```

The rule for reading loss: **loss at a hop matters only if it continues to the final hop.** Hop 3 above shows 100% loss while everything beyond it is fine, so hop 3 simply does not answer probes. Loss that appears at hop 2 and persists at 3, 4 and 5 is a real problem at or just before hop 2.

Hop 1 is your own router. Loss or high `StDev` there is your LAN or Wi-Fi.

### Exercises 4

1. Trace the route to `1.1.1.1` three ways (UDP, ICMP, TCP 443). Do they show the same number of hops? Do any hide hops the others show?
2. How many hops to your gateway? What does the first line of every traceroute from the tower show?
3. Run `mtr -n -r -w -c 50` to a site you use. Is there loss at hop 1? Does any hop's loss continue to the end?
4. Run `c traceroute -n 10.99.0.1` from the client. How many hops, and why?
5. Explain how traceroute finds hop 4 using what you know about TTL and ICMP type 11.

<details><summary>Solutions</summary>

1. Hop counts usually agree; the set of responding hops often differs, because routers treat the probe types differently, and the final hop may answer only one type.
2. One hop: it is directly connected. The first line is always your default gateway.
3. Read the `Loss%` column from the bottom up.
4. One. The client and tower are on the same segment, so the first probe (TTL 1) reaches the destination itself.
5. It sends a probe with TTL 4. Routers 1, 2 and 3 each decrement and forward it. Router 4 decrements it to zero, discards it, and returns ICMP time exceeded from its own address, which traceroute prints with the elapsed time.

</details>

---

## 5. netstat (and ss)

`netstat` is the classic tool for sockets, routes and interface statistics. It belongs to `net-tools`, which is deprecated and not installed by default on Rocky; `ss` and `ip` replace it. You still need to read it: it is in every older guide, in interview questions, on other Unix systems and inside many container images.

| netstat | Modern | Shows |
|---|---|---|
| `netstat -tulpn` | `ss -tulpn` | listening TCP and UDP sockets with processes |
| `netstat -an` | `ss -an` | all sockets |
| `netstat -tn` | `ss -tn` | TCP connections |
| `netstat -rn` | `ip route` | routing table |
| `netstat -i` | `ip -s link` | interface counters |
| `netstat -s` | `nstat -az`, `ss -s` | protocol statistics |
| `netstat -g` | `ip maddr` | multicast groups |
| `netstat -c` | `watch ss …` | repeat continuously |

Flags: `-t` TCP, `-u` UDP, `-l` listening only, `-a` all, `-n` numeric, `-p` process, `-r` routes, `-i` interfaces, `-s` statistics. The same letters mean the same thing in `ss`.

```bash
sudo netstat -tulpn
sudo ss -tulpn                           # compare column by column
netstat -rn                              # the Flags column: U up, G uses a gateway, H host route
netstat -i                               # RX-ERR, RX-DRP, TX-ERR, TX-DRP should be zero or stable
netstat -s | head -40
netstat -s | grep -iE 'retransmit|reset|overflow|drop|fail'
```

### Socket states

```bash
ss -tan | awk 'NR > 1 {print $1}' | sort | uniq -c | sort -rn
```

| State | Meaning | Many of them means |
|---|---|---|
| `LISTEN` | waiting for connections | |
| `ESTAB` | connected | normal load |
| `SYN-SENT` | we sent SYN, no answer yet | **outbound connections are being dropped**: remote firewall, dead host |
| `SYN-RECV` | SYN received, handshake not finished | a SYN flood, or clients that cannot be answered |
| `TIME-WAIT` | closed, waiting out stray packets (60 s) | a high connection rate. Normal on busy servers |
| `CLOSE-WAIT` | the peer closed; **our application has not** | a file-descriptor leak in the application |
| `FIN-WAIT-1/2` | we closed; waiting for the peer | peers vanishing |

### ss, beyond netstat

```bash
ss -tn state established                           # filter by state
ss -tn state established '( dport = :443 or sport = :443 )'
ss -tn dst 1.1.1.1                                 # by peer
sudo ss -tlnp 'sport = :8099'                      # who owns the test web server's port
ss -tln                                            # for LISTEN sockets, Recv-Q is the accept queue in use, Send-Q its limit
ss -tin state established | head -12               # -i: per-connection TCP detail: rtt, cwnd, retrans
ss -tm | head                                      # -m: socket memory
ss -s                                              # totals
ss -tn state established | awk 'NR > 1 {split($4, a, ":"); c[a[1]]++} END {for (ip in c) print c[ip], ip}' | sort -rn | head    # connections per peer
```

`Recv-Q` equal to `Send-Q` on a listening socket means the accept queue is full: the application is not accepting connections fast enough, and new clients time out even though the port is "open".

```bash
nstat -az | grep -E 'TcpRetransSegs|TcpExtListenOverflows|TcpExtListenDrops|TcpExtTCPTimeouts|TcpOutRsts'
```

Counters are totals since boot. Read them twice, a minute apart, and look at the difference. Rising retransmits mean loss somewhere on the path. Rising listen overflows mean a saturated application.

### Exercises 5

1. Produce identical information with `netstat` and `ss`: every listening TCP socket with its process. Which columns differ?
2. Count the tower's TCP sockets by state. Are any in `CLOSE-WAIT` or `SYN-SENT`? What would each indicate?
3. While `c curl -s --limit-rate 10 http://tower.lab:8099/ > /dev/null &` runs, find that connection on the tower side with `ss` and show its `rtt`.
4. Has this machine retransmitted TCP segments since boot? Is the count increasing right now?
5. Which process owns port 8099, running as which user, started from which command line?

<details><summary>Solutions</summary>

```bash
sudo netstat -tlnp; sudo ss -tlnp                                               # 1
ss -tan | awk 'NR > 1 {print $1}' | sort | uniq -c                              # 2
c curl -s --limit-rate 10 http://tower.lab:8099/ > /dev/null &                  # 3
sleep 1; ss -tin 'sport = :8099' | grep -A1 10.99.0.2 | grep -o 'rtt:[^ ]*'
nstat -az TcpRetransSegs; sleep 30; nstat -az TcpRetransSegs                    # 4
sudo ss -tlnp 'sport = :8099'; ps -o user,pid,cmd -p "$(pgrep -u nobody -f 'http.server 8099')"   # 5
```

</details>

---

## 6. Packet analysis

When the tools above disagree with each other, or with what the application claims, look at the packets. A capture shows what was actually sent and received, and ends arguments.

`tcpdump` captures on the server. Wireshark is the graphical analyser you run on a laptop, usually on a capture file made with tcpdump.

```bash
sudo tcpdump -D                                    # interfaces available for capture
sudo tcpdump -i veth-lab -nn -c 5                  # 5 packets on the test link, then stop
```

| Option | Meaning |
|---|---|
| `-i IF` / `-i any` | interface, or all of them |
| `-nn` | do not resolve host names or port names. **Always use it** |
| `-c N` | stop after N packets |
| `-v`, `-vv` | more protocol detail |
| `-e` | show the link-layer header: MAC addresses |
| `-A` | print payload as text: for HTTP and other plaintext protocols |
| `-X` | print payload as hex and text |
| `-s 0` | capture whole packets (the default on current versions) |
| `-w FILE` | write raw packets to a pcap file |
| `-r FILE` | read a pcap file |
| `-tttt` | full date and time on each line |
| `-q` | less detail per line |
| `-l` | line-buffered: needed when piping to `grep` |

### Filters

A filter expression limits what is captured. Filter hard: an unfiltered capture on a busy server is unreadable and large.

| Filter | Captures |
|---|---|
| `host 10.99.0.2` | to or from that address |
| `src 10.99.0.2` / `dst 10.99.0.1` | one direction |
| `net 10.99.0.0/24` | a subnet |
| `port 53` | to or from that port, TCP or UDP |
| `tcp port 443`, `udp port 53` | protocol and port |
| `portrange 8000-8100` | a range |
| `icmp`, `arp`, `tcp`, `udp`, `ip6` | by protocol |
| `host A and port 80`, `port 80 or port 443`, `not port 22` | combinations. Quote the expression; parentheses need quoting too |
| `'tcp[tcpflags] & (tcp-syn\|tcp-rst) != 0'` | only SYN and RST packets: connection attempts and refusals |
| `'tcp[tcpflags] == tcp-syn'` | only initial SYNs |

**When capturing on your real interface over SSH, always add `not port 22`** (or your SSH port). Otherwise every packet you capture is printed over SSH, which creates more packets to capture.

### Reading a TCP line

```
20:15:01.123456 IP 10.99.0.2.48122 > 10.99.0.1.8099: Flags [S], seq 1234567890, win 64240, options [mss 1460,…], length 0
└── time ─────┘    └ source.port ┘   └ dest.port ─┘  └ flags ┘ └ sequence ───┘  └ window ┘                    └ payload bytes
```

| Flag | Meaning |
|---|---|
| `[S]` | SYN: open a connection |
| `[S.]` | SYN-ACK: accept it (`.` is ACK) |
| `[.]` | ACK only |
| `[P.]` | PUSH + ACK: carries data |
| `[F.]` | FIN + ACK: closing this direction |
| `[R]`, `[R.]` | RST: abort, or "nothing here" |

### Capture 1: a complete HTTP exchange

Terminal 1:

```bash
sudo tcpdump -i veth-lab -nn -A port 8099
```

Terminal 2:

```bash
c curl -s http://tower.lab:8099/
```

Read the capture against this script:

| Packets | What |
|---|---|
| `.2 > .1 [S]` | client: SYN |
| `.1 > .2 [S.]` | server: SYN-ACK |
| `.2 > .1 [.]` | client: ACK. **The three-way handshake** |
| `.2 > .1 [P.]` with `GET / HTTP/1.1`, `Host: tower.lab:8099` | the request, readable |
| `.1 > .2 [.]` | server acknowledges |
| `.1 > .2 [P.]` with `HTTP/1.0 200 OK` and `hello from the tower` | the response, readable |
| `[F.]` each way, and final `[.]` | orderly close |

Every byte of an unencrypted protocol is visible to anyone on the path. That is the argument for TLS, made concrete.

### Capture 2: refused, rejected, and silent

Keep terminal 1 running, but widen the filter. Ctrl-C it and start:

```bash
sudo tcpdump -i veth-lab -nn 'host 10.99.0.2 and not arp'
```

Terminal 2, three failures:

```bash
sudo ./labnet break 4                    # (you may know this one: the web server is stopped)
c curl -sS -m 5 http://tower.lab:8099/   # Connection refused
sudo ./labnet fix

c curl -sS -m 5 http://tower.lab:8098/   # a port the firewall does not allow, so it REJECTS: No route to host

# a rule that silently DROPS port 8097 from the client, removed automatically after two minutes
sudo firewall-cmd --timeout=2m --add-rich-rule='rule family="ipv4" source address="10.99.0.2" port port="8097" protocol="tcp" drop'
c curl -sS -m 8 http://tower.lab:8097/   # Connection timed out
```

| Client error | In the capture |
|---|---|
| Connection refused | `[S]` answered at once by `[R.]`. The host is there; the port is closed |
| No route to host | `[S]` answered by `ICMP host 10.99.0.1 unreachable - admin prohibited`. A firewall **rejected** it |
| Connection timed out | `[S]`, then the same `[S]` again after 1 s, 2 s, 4 s… and nothing coming back. A firewall **dropped** it (or the host is gone): retransmission with backoff into silence |

These three pictures are worth remembering. Almost every "can't connect" is one of them.

### Capture 3: DNS and TLS on the real interface

```bash
sudo tcpdump -i "$IF" -nn -c 4 port 53 &
sleep 1; dig +short example.com > /dev/null; wait
```

```
IP 192.168.1.50.41234 > 192.168.1.1.53: 4242+ A? example.com. (40)
IP 192.168.1.1.53 > 192.168.1.50.41234: 4242 2/0/0 A 23.215.0.136, A 23.215.0.138 (72)
```

A query (`A?`), an ID that matches request to reply, and the answer. Plain DNS is also readable by anyone on the path.

```bash
sudo tcpdump -i "$IF" -nn -c 12 -A 'tcp port 443 and host example.com' 2>/dev/null | grep -aE 'Flags|example' &
sleep 1; curl -s https://example.com > /dev/null; wait
```

After the handshake you see the TLS ClientHello, which contains the site name in clear text (the SNI field), and then unreadable encrypted data. Encryption hides content. It does not hide who is talking to whom, or how much.

### Capture files and Wireshark

```bash
sudo tcpdump -Z root -i veth-lab -nn -w ~/labwork/15/web.pcap port 8099 &
sleep 1; c curl -s http://tower.lab:8099/ > /dev/null; c curl -s http://tower.lab:8099/missing > /dev/null
sleep 1; sudo pkill -INT -x tcpdump; wait 2>/dev/null
sudo chown "$USER": ~/labwork/15/web.pcap
tcpdump -nn -r ~/labwork/15/web.pcap | head -20           # read it back, with any filter you like
tcpdump -nn -r ~/labwork/15/web.pcap 'tcp[tcpflags] & tcp-syn != 0'
tcpdump -nn -A -r ~/labwork/15/web.pcap | grep -a 'HTTP/1'
```

(`-Z root`: on RHEL-family systems tcpdump switches to an unprivileged `tcpdump` user before writing a capture file, and that user cannot write into your home directory. `-Z root` keeps it as root. The alternative is to write under `/var/tmp`.)

Copy `web.pcap` to your laptop (`scp`) and open it in Wireshark. Right-click a packet and choose *Follow → TCP Stream* to see a conversation reassembled. *Statistics → Conversations* summarises who talked to whom. Display filters there use a different syntax: `tcp.port == 8099`, `http.response.code == 404`, `tcp.analysis.retransmission`, `dns`.

Live capture into Wireshark on the laptop, with nothing stored on the server:

```bash
# ssh tower 'sudo tcpdump -i any -U -nn -w - not port 22' | wireshark -k -i -
```

Long captures should rotate: `-C 100 -W 5` keeps five files of 100 MB.

Capture files hold whatever crossed the wire, including credentials for plaintext protocols and session cookies. Treat them as sensitive: capture narrowly, store with mode 600, delete when done.

### Containers

```bash
sudo tcpdump -i docker0 -nn -c 10 not arp                  # traffic on the default bridge
# inside one container's network namespace, using the host's tcpdump:
# sudo nsenter -t "$(docker inspect -f '{{.State.Pid}}' NAME)" -n tcpdump -i eth0 -nn -c 10
```

`nsenter -n` is how you debug a container that has no tools inside it: borrow its network namespace and use the host's.

### Exercises 6

1. Capture exactly the three handshake packets of one connection from the client, and nothing else.
2. Capture ARP on `veth-lab`, flush the client's neighbour cache (`c ip neigh flush dev veth-ns`), and ping. Explain each packet.
3. Request `/missing` from the client and find the status line of the response in a capture.
4. Capture with `-e` and identify the two MAC addresses. Match them against `ip link show veth-lab` and `c ip link show veth-ns`.
5. Capture one `ping -c 1` each way and report the ICMP `id` and `seq` values in request and reply.
6. From `web.pcap`: how many TCP connections does it contain? How many packets carried payload?
7. Capture the firewall rejecting a connection to port 8098 with `-v`. What are the ICMP type and code, and which original packet does the ICMP message quote?

<details><summary>Solutions</summary>

```bash
# 1  (one terminal captures, the other runs: c curl -s http://tower.lab:8099/ > /dev/null)
sudo tcpdump -i veth-lab -nn -c 3 'port 8099 and (tcp[tcpflags] & tcp-syn != 0 or tcp[tcpflags] == tcp-ack)'
# simpler and good enough: sudo tcpdump -i veth-lab -nn -c 3 port 8099

# 2
sudo tcpdump -i veth-lab -nn -e -c 4 'arp or icmp' & sleep 1; c ip neigh flush dev veth-ns; c ping -c 1 10.99.0.1; wait
# who-has 10.99.0.1 tell 10.99.0.2 (to the broadcast MAC), is-at <mac>, then echo request and echo reply

# 3
sudo tcpdump -i veth-lab -nn -A -c 12 port 8099 2>/dev/null | grep -a 'HTTP/1' & sleep 1; c curl -s http://tower.lab:8099/missing > /dev/null; wait

# 4
sudo tcpdump -i veth-lab -nn -e -c 2 icmp & sleep 1; c ping -c 1 10.99.0.1 > /dev/null; wait
ip link show veth-lab | grep ether; c ip link show veth-ns | grep ether

# 6
tcpdump -nn -r web.pcap 'tcp[tcpflags] == tcp-syn' | wc -l            # connections = initial SYNs
tcpdump -nn -r web.pcap 'tcp[tcpflags] & tcp-push != 0' | wc -l       # packets with data

# 7
sudo tcpdump -i veth-lab -nn -v -c 2 'icmp or port 8098' & sleep 1; c nc -zv -w 2 10.99.0.1 8098; wait
# ICMP type 3 code 10, "host unreachable - admin prohibited", quoting the client's SYN to port 8098
```

</details>

---

## 7. Challenge: find the fault

The testbed can break itself in six ways. Each run of `break` restores the working state and then injects one fault at random. Your job is to find it using the method and the tools, and say which rung of the ladder failed and why.

```bash
sudo ./labnet break
sudo ./labnet check                      # FAIL: this is the user's complaint
```

Rules:

- Do not read the `break` function, and do not run `labnet fix` until you can state the cause.
- Diagnose from both sides: the client (`c …`) and the tower.
- Write one line per fault: symptom, the command that exposed the cause, the cause.
- Then `sudo ./labnet fix`, confirm with `check`, and run `break` again. Repeat until you have met all six (or request them by number, 1 to 6).

A routine that works for every one of them:

```bash
c curl -sS -m 6 http://tower.lab:8099/   # 1. the exact error text
c ip -br link; c ip -br addr; c ip route # 2. the client's link, address and routes
c getent hosts tower.lab                 # 3. what the name resolves to
c ping -c 2 -W 1 10.99.0.1               # 4. reach the server by address
c ip neigh                               # 5. did ARP succeed?
c nc -zv -w 3 10.99.0.1 8099             # 6. the port, by address
ip -br link show veth-lab; ip -br addr show veth-lab      # 7. the server's side of the link
sudo ss -tlnp | grep 8099                # 8. is it listening, and on which address?
curl -s -m 3 http://127.0.0.1:8099/      # 9. does it answer locally?
sudo firewall-cmd --list-ports           # 10. is the port allowed?
sudo tcpdump -i veth-lab -nn -c 6        # 11. what happens on the wire during a request?
```

<details><summary>The six faults (open only after diagnosing)</summary>

| # | Client's error | Decisive evidence | Cause | Ladder rung |
|---|---|---|---|---|
| 1 | `Network is unreachable`, instantly | `c ip -br addr` shows `10.99.1.2/24`; `c ip route` has no route to `10.99.0.0/24` and no default | the client has an address in the wrong subnet and no gateway | 2: address |
| 2 | `No route to host` after about 3 seconds | `c getent hosts tower.lab` returns `10.99.0.9`; `c ip neigh` shows `10.99.0.9 FAILED`; `c ping 10.99.0.1` works | the name points at the wrong address (a stale hosts or DNS entry) | 5: name resolution |
| 3 | `Connection refused` | `sudo ss -tlnp \| grep 8099` shows `127.0.0.1:8099`; local `curl` works | the service is bound to loopback only | server (b): bind address |
| 4 | `Connection refused` | `ss` shows nothing on 8099; local `curl` also fails; no `python3` process | the service is not running | server (a): process |
| 5 | `No route to host` after a few seconds, or a timeout; ping reports `Destination Host Unreachable` | `ip -br link show veth-lab` is `DOWN` on the tower; the client's `veth-ns` shows no carrier (`LOWERLAYERDOWN`) | the server's interface is administratively down | 1: link |
| 6 | `No route to host`, **instantly** | ping works and `c ip neigh` is `REACHABLE`; tcpdump shows `ICMP … admin prohibited`; `firewall-cmd --list-ports` lacks `8099/tcp` | the firewall does not allow the port | server (d): firewall |

Faults 3 and 4 give the client the identical error and are only told apart on the server. Faults 2, 5 and 6 can all say `No route to host`: timing (instant versus after an ARP timeout), whether ping to the address works, and the neighbour table separate them. That is why you collect evidence from several rungs before deciding.

</details>

### Stacked faults

Real incidents often have two causes, and fixing the first changes the symptom without restoring service.

```bash
sudo ./labnet break 3
sudo firewall-cmd --remove-port=8099/tcp
sudo ./labnet check
```

Diagnose. You will find one fault, fix it by hand (not with `labnet fix`), and see a *different* error. Find the second.

<details><summary>What happens</summary>

The client first gets `No route to host`: the firewall rejects before the packet reaches any socket. Re-adding the port (`sudo firewall-cmd --add-port=8099/tcp`) changes the error to `Connection refused`: now the packet arrives and nothing is listening on `10.99.0.1`, because the server is bound to `127.0.0.1`. Only after both are corrected does `check` pass. A changed error message is progress, and each message pointed at the layer to look at next.

</details>

---

## Quick reference

| Symptom | First commands |
|---|---|
| Nothing works at all | `ip -br link`, `ip -br addr`, `ip route`, `ping GATEWAY` |
| Addresses work, names do not | `cat /etc/resolv.conf`, `dig @1.1.1.1 name`, `getent hosts name` |
| One host unreachable | `ip route get IP`, `ping`, `ip neigh`, `traceroute -n` / `mtr` |
| Port unreachable from outside, fine locally | `ss -tlnp` (bind address), `firewall-cmd --list-all`, `docker ps` (published address) |
| Connects, then stalls on large transfers | `ping -M do -s 1472`, `tracepath` (MTU) |
| Slow or intermittent | `mtr -n -r -c 100`, `ping -D` log, `nstat` retransmits, Wi-Fi signal |
| Service "up" but refusing connections under load | `ss -tln` queue columns, `nstat` ListenOverflows, application logs |
| The application and the network team disagree | `tcpdump -w`, then Wireshark |
| A container cannot reach something | `docker exec C cat /etc/resolv.conf`, `nsenter -t PID -n` plus the host's tools |

## Check yourself

<details><summary>What is the difference between "Connection refused" and "Connection timed out", in packets?</summary>
Refused: the SYN is answered by a RST from the destination, so the host is reachable and the port is closed. Timed out: the SYN is retransmitted with increasing delays and nothing ever comes back, so packets are being silently dropped or the host is gone.
</details>

<details><summary>"Network is unreachable" appears instantly. Where is the problem?</summary>
On the local machine. Its own routing table has no route for the destination, so nothing was sent. Check the address, the subnet mask and the default route.
</details>

<details><summary>A host does not answer ping. Is it down?</summary>
Not necessarily. ICMP echo may be filtered while the service works. Test the port the service uses.
</details>

<details><summary>How does traceroute discover the routers on a path?</summary>
It sends probes with TTL 1, 2, 3 and so on. Each router that decrements a probe's TTL to zero returns ICMP time exceeded, revealing its address.
</details>

<details><summary>An mtr report shows 60% loss at hop 4 and 0% at hops 5 to 9. Is hop 4 the problem?</summary>
No. Traffic is passing through it without loss. That router limits how often it answers probes addressed to itself. Loss matters when it continues to the final hop.
</details>

<details><summary>What is the <code>ss</code> equivalent of <code>netstat -tulpn</code>, and what does each letter mean?</summary>
<code>ss -tulpn</code>: TCP, UDP, listening, process, numeric.
</details>

<details><summary>Many sockets in CLOSE-WAIT. What kind of bug is that?</summary>
An application bug. The peer has closed each connection and the local program has not called close, so it is leaking sockets.
</details>

<details><summary>Why add <code>not port 22</code> to a capture taken over SSH?</summary>
Each captured packet is displayed through the SSH session, which generates more packets on the same interface, which are captured and displayed in turn.
</details>

<details><summary>Which flags mark the three packets of a TCP handshake in tcpdump?</summary>
<code>[S]</code>, <code>[S.]</code>, <code>[.]</code>: SYN, SYN-ACK, ACK.
</details>

<details><summary>A web service works with <code>curl http://127.0.0.1:8080</code> on the server and fails from every other machine. Name three possible causes and the command that checks each.</summary>
Bound to loopback only (<code>ss -tlnp</code>); blocked by the host firewall (<code>firewall-cmd --list-all</code>); a container port published on 127.0.0.1 (<code>docker ps</code>). A fourth: a firewall elsewhere on the path, shown by the SYNs never arriving in <code>tcpdump</code>.
</details>

## Cleanup

```bash
sudo ~/labwork/15/labnet down
sudo firewall-cmd --list-ports                 # 8099 should not be listed
sudo firewall-cmd --list-rich-rules            # the 8097 drop rule expires by itself; if still listed, wait or remove it:
# sudo firewall-cmd --remove-rich-rule='rule family="ipv4" source address="10.99.0.2" port port="8097" protocol="tcp" drop'
ip netns list | grep labns                     # expect nothing
ip -br link | grep veth-lab                    # expect nothing
pgrep -af 'http.server 8099'                   # expect nothing
sudo pkill tcpdump 2>/dev/null
rm -rf ~/labwork/15
```
