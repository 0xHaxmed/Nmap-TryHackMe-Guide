# Nmap — The Complete Beginner's Guide

### A detailed walkthrough of the Nmap module on TryHackMe: all 4 Rooms + the Recap, with answered questions and command summaries

> **Legal & Ethical Notice:** every command in this guide is for learning purposes only, to be run against machines you own or are explicitly authorized to test — such as TryHackMe, Hack The Box, CTF environments, and your own personal labs (VMs). Scanning networks or devices without authorization may be illegal.

---

## How to Use This Guide

- Start with the **Fundamentals** section (Section 0) if you're new to networking. Read it once and everything after it will be much easier.
- Each Room has its own self-contained section, following the same structure: **the idea, the commands, how it works under the hood, attacker vs. defender perspective, then the answered questions**.
- Commands are written exactly as typed in the terminal. Explanations are in clear English with standard technical terminology.
- **About answered questions:** theory questions (e.g., "what is the flag?", "what's the command?") are answered directly. Practical questions that require a result from scanning your own Lab machine have answers that change with every instance, so instead I give you **the exact command to run and how to read the answer out of the output**, with a blank for you to fill in your result. That's the approach that actually builds skill.

---

## Table of Contents

0. [Fundamentals You Need Before Starting](#basics)
1. [Room 1: Nmap Live Host Discovery](#room1)
2. [Room 2: Nmap Basic Port Scans](#room2)
3. [Room 3: Nmap Advanced Port Scans](#room3)
4. [Room 4: Nmap Post Port Scans](#room4)
5. [Topic Transition Recap: Full Review and Workflow](#recap)
6. [Cheat Sheet: Every Command on One Page](#cheatsheet)
7. [Common Mistakes and How to Fix Them](#troubleshooting)
8. [References and Learning Resources](#resources)
9. [Glossary](#glossary)

---

<a id="basics"></a>

## 0) Fundamentals You Need Before Starting

### 0.1 What Is Nmap?

**Nmap** (Network Mapper) is an open-source tool for discovering devices and services on a network. Created by Gordon Lyon (known as Fyodor), it is the de facto standard in penetration testing. Nmap answers these questions for you:

| Question | Phase |
|---|---|
| Which devices (Live Hosts) are up on the network? | Host Discovery |
| Which ports are open on each device? | Port Scanning |
| What service and version is running on each port? | Service / Version Detection |
| What operating system is it? | OS Detection |
| Are there vulnerabilities or extra useful info? | Nmap Scripting Engine (NSE) |

**The logical scanning path** (and the same order the Rooms follow):

```
Live Host Discovery  ->  Port Scan  ->  Advanced/Evasion  ->  Service + OS + Scripts  ->  Save Results
     (Room 1)            (Room 2)         (Room 3)                  (Room 4)
```

### 0.2 Installation and General Syntax

```bash
# Debian / Ubuntu / Kali (often pre-installed on Kali and the AttackBox)
sudo apt update && sudo apt install nmap -y

# Check version
nmap --version
```

General syntax:

```bash
nmap [Scan Type] [Options] {target}
```

Examples of `{target}`:

```bash
nmap 10.10.10.5                 # a single host
nmap scanme.nmap.org            # a domain name
nmap 10.10.10.1-20              # a range of hosts
nmap 10.10.10.0/24              # a whole network (CIDR notation)
nmap -iL targets.txt            # read targets from a file
```

> **Note:** `scanme.nmap.org` is a site the Nmap team explicitly allows light scanning against for practice. Don't abuse it.

### 0.3 Why Do We Use `sudo` With Nmap?

Many scan types need to send **Raw Packets** (hand-crafted packets), which requires root privileges.

| Situation | What Happens |
|---|---|
| Running Nmap as root | Uses the fastest and most powerful techniques: **SYN Scan** (`-sS`) and **ARP** on the local network |
| Running Nmap without root | Falls back to **TCP Connect Scan** (`-sT`) because it doesn't need raw sockets |

### 0.4 Core Networking Concepts

#### (a) TCP/IP Layers

| Layer | Function | Examples | Where it shows up in Nmap |
|---|---|---|---|
| **Application** | Application-level protocols | HTTP, FTP, DNS, SSH | Service Detection and NSE |
| **Transport** | Delivers data between two applications via Ports | TCP, UDP | Port Scans |
| **Internet** | Addressing and routing | IP, ICMP | ICMP Ping, Traceroute |
| **Link** (Network Access) | Communication within the local network via MAC addresses | Ethernet, Wi-Fi, **ARP** | ARP Scan |

> **Quick memorization:** ARP works at the **Link** layer. IP and ICMP at the **Internet** layer. TCP and UDP at the **Transport** layer.

#### (b) The ARP Protocol

A device knows the target's **IP** but not its **MAC**. So it sends an **ARP Request** (Broadcast) asking: *"Who has this IP? Tell me."* The device that owns that IP replies with an **ARP Reply** containing its MAC address. (This is exactly what you saw in the preceding lab: `Who has router tell computer2`).

Two points that matter for Nmap:
1. ARP only works within the **same local network (Subnet)** — it isn't routed through a Router.
2. A device on the local network can't easily ignore ARP, which makes it the most accurate way to discover local hosts — even when a firewall blocks Ping.

#### (c) The ICMP Protocol

A protocol for diagnostic messages. The ones that matter most to us:

| Type | Name | Use |
|---|---|---|
| 8 | Echo Request | This is `ping` |
| 0 | Echo Reply | The ping's response |
| 13 / 14 | Timestamp Request / Reply | Asking a device for its clock time |
| 17 / 18 | Address Mask Request / Reply | Asking a device for its Subnet Mask |
| 3 | Destination Unreachable | An error message, including **Port Unreachable** (Code 3) |

#### (d) TCP: The 3-Way Handshake

TCP is a reliable protocol that establishes a connection before sending data:

```
   Client                              Server
     |  ---------- SYN ------------->    |    "I want to connect"
     |  <------- SYN + ACK ----------    |    "Agreed, and I want to connect too"
     |  ---------- ACK ------------->    |    "Done, the connection is established"
```

**TCP Flags** — the foundation for understanding every scan type:

| Flag | Meaning |
|---|---|
| **SYN** | Start a connection (Synchronize) |
| **ACK** | Acknowledge receipt |
| **FIN** | Gracefully end the connection |
| **RST** | Abruptly terminate / reject (Reset) |
| **PSH** | Push data straight to the application |
| **URG** | The data is urgent |

**The golden rule for understanding scans:** when you send a SYN to a port:
- Open -> replies `SYN/ACK`
- Closed -> replies `RST`
- A Firewall is blocking -> no reply, or an ICMP error message

#### (e) The UDP Protocol

UDP is **connectionless**: it sends without waiting for acknowledgment. Faster but unreliable. Used in DNS, DHCP, SNMP, and VoIP. This makes scanning it **slower and harder**, because an open port often doesn't reply at all (explained in Room 2).

#### (f) Common Ports Worth Memorizing

| Port | Protocol | Service |
|---|---|---|
| 20/21 | TCP | FTP |
| 22 | TCP | SSH |
| 23 | TCP | Telnet |
| 25 | TCP | SMTP |
| 53 | UDP (and TCP) | DNS |
| 67/68 | UDP | DHCP |
| 80 | TCP | HTTP |
| 110 | TCP | POP3 |
| 111 | TCP/UDP | RPCbind |
| 123 | UDP | NTP |
| 139 / 445 | TCP | NetBIOS / SMB |
| 143 | TCP | IMAP |
| 161 | UDP | SNMP |
| 443 | TCP | HTTPS |
| 3306 | TCP | MySQL |
| 3389 | TCP | RDP |

There are **65,535** ports in each protocol (1 through 65535). Ports 1 through 1023 are called **Well-Known Ports**.

### 0.5 How to Read Nmap Output

> The example below is illustrative (not from the Lab):

```
$ sudo nmap -sS 10.10.10.5
Nmap scan report for 10.10.10.5
Host is up (0.00045s latency).              <-- (1) the host is alive
Not shown: 996 closed tcp ports (reset)     <-- (2) summary of the rest
PORT    STATE SERVICE                       <-- (3) the main table
22/tcp  open  ssh
80/tcp  open  http
139/tcp open  netbios-ssn
445/tcp open  microsoft-ds
MAC Address: 02:AB:CD:12:34:56 (Unknown)    <-- (4) only shows on the local network
Nmap done: 1 IP address (1 host up) scanned in 1.52 seconds
```

1. **Host is up:** we got a reply from the device. If you see `Host seems down`, the Firewall might be blocking Ping (fix: add `-Pn`).
2. **Not shown:** ports that weren't individually listed because they share one common state, with the reason next to it (`reset` means the host replied with RST).
3. **Columns:** the port/protocol number, its state (STATE), and the service guessed from the port number (not confirmed unless you use `-sV`).
4. **MAC Address:** only appears when the target is on the same local network.

---
<a id="room1"></a>

## 1) Room 1: Nmap Live Host Discovery

**Goal:** Before scanning ports, we need to know **which devices are actually up** on the network. Scanning 65,000 ports against a powered-off machine is a waste of time. This Room covers 4 methods of host discovery: **ARP, ICMP, TCP, UDP**.

### 1.1 Enumerating Targets

#### Calculating CIDR Ranges

CIDR `/N` means the first N bits are fixed (Network) and the rest (32 - N) are variable (Hosts).

| CIDR | Subnet Mask | Number of Addresses |
|---|---|---|
| /8 | 255.0.0.0 | 16,777,216 |
| /16 | 255.255.0.0 | 65,536 |
| /24 | 255.255.255.0 | 256 |
| /29 | 255.255.255.248 | 8 |
| /32 | 255.255.255.255 | 1 (single host) |

Rule: **Number of addresses = 2 ^ (32 - N)**.

#### Ways to Write Targets in Nmap

| Format | Example | Meaning |
|---|---|---|
| Single host | `nmap 10.10.10.5` | |
| Hostname | `nmap example.com` | resolved to an IP via DNS |
| Range | `nmap 10.10.10.10-20` | from .10 to .20 |
| Multi-octet range | `nmap 10.10.0-255.101-125` | more than one octet |
| CIDR | `nmap 10.10.10.0/24` | a whole network |
| File | `nmap -iL list.txt` | each line is a target |
| Exclusion | `nmap 10.10.10.0/24 --exclude 10.10.10.1` | |

#### The `-sL` Option (List Scan): Your Best Friend for Math

Displays the targets **without sending any packet to the hosts themselves**. Useful for double-checking your range before the real scan (and it attempts Reverse DNS unless you add `-n`).

```bash
nmap -sL -n 10.10.12.13/29
```

### 1.2 The Four Types of Host Discovery

The `-sn` option means: **"don't scan ports"** (Host Discovery only). It used to be called "Ping Scan." We combine it with type-selecting options.

> **Nmap's default host discovery behavior:**
> - **You're root + target is on the local network:** uses **ARP**.
> - **You're root + target is on a remote network:** sends ICMP Echo + TCP SYN to 443 + TCP ACK to 80 + ICMP Timestamp.
> - **No root:** tries TCP Connect to ports 80 and 443.

#### Type 1: ARP Scan (Option `-PR`)

```bash
sudo nmap -PR -sn 192.168.0.0/24
```

- **Most accurate** on the local network, because it's hard to ignore even if a device blocks Ping.
- Needs **root**.
- **Does not work outside the Subnet** since ARP isn't routed across networks.
- Complementary tool: `arp-scan`.

```bash
sudo arp-scan -l                 # scan local network on default interface
sudo arp-scan --localnet
sudo arp-scan -I eth0 -l         # specify the interface
sudo arp-scan -l | less          # for reviewing
```

#### Type 2: ICMP Scan

Three variants:

| Option | Message Type | Notes |
|---|---|---|
| `-PE` | **Echo Request** (Type 8) | the classic ping. Often blocked by firewalls |
| `-PP` | **Timestamp Request** (Type 13) | may get through when Echo is blocked |
| `-PM` | **Address Mask Request** (Type 17) | may get through when Echo is blocked |

```bash
sudo nmap -PE -sn 10.10.10.0/24          # Echo
sudo nmap -PP -sn 10.10.10.0/24          # Timestamp
sudo nmap -PM -sn 10.10.10.0/24          # Address Mask
```

> **Note:** even if a network doesn't reply to Echo, try Timestamp and Address Mask — the admin may have forgotten to block them.

#### Type 3: TCP Ping

| Option | What's Sent | What It Tells You | Needs root? |
|---|---|---|---|
| `-PS[ports]` | **SYN** | a reply of `SYN/ACK` or `RST` = host is up | **No** (uses connect()) |
| `-PA[ports]` | **ACK** | a reply of `RST` = host is up | **Yes** |

```bash
sudo nmap -PS -sn 10.10.10.0/24            # SYN to Port 80 (default)
sudo nmap -PS23 -sn 10.10.10.0/24          # SYN to Port 23 (Telnet)
sudo nmap -PS21-25,80,443 -sn 10.10.10.0/24
sudo nmap -PA -sn 10.10.10.0/24            # ACK to Port 80
```

**How does it work?** Whether the port is open or closed, the host *will reply with something* (SYN/ACK or RST). Either reply proves it's alive. ACK Ping can trick firewalls that only block SYN packets, because an ACK packet looks like part of an existing connection, prompting the host to reply with RST.

#### Type 4: UDP Ping (Option `-PU`)

```bash
sudo nmap -PU -sn 10.10.10.0/24
sudo nmap -PU53 -sn 10.10.10.0/24
```

- Sends a UDP packet to a port (usually closed).
- If the host is alive, it replies with an **ICMP Port Unreachable**, proving it's up.
- If nothing comes back, we don't know (could be down or blocking it).

#### The masscan Tool (for high-speed scanning)

```bash
masscan 10.0.0.0/8 -p443 --rate 1000
```

Much faster than Nmap because it uses its own TCP/IP stack, but less accurate and with fewer features. Used for scanning huge ranges. **Only use it against authorized targets.**

### 1.3 DNS: the `-R` and `-n` Options

By default Nmap tries **Reverse DNS** only on hosts that respond (IP -> Hostname), because names leak info (like `db-server`, `dc01`).

| Option | Function |
|---|---|
| `-n` | **Never** do DNS lookups (faster and quieter) |
| `-R` | Do Reverse DNS on **every** IP, even those that appear down |
| `--dns-servers 8.8.8.8` | Use a specific DNS server |

### 1.4 Attacker and Defender Perspective

| | Attacker (Red Team) | Defender (Blue Team) |
|---|---|---|
| **What they do** | Start with ARP on the local network since it's the most accurate. Try multiple types when ICMP is blocked | Watch for heavy ARP traffic (tools like `arpwatch`) and bursts of SYN/ICMP across a wide range |
| **Weakness / Strength** | ARP is "loud" on the local network | Blocking Echo alone isn't enough — must also monitor Timestamp and Address Mask |

### 1.5 Room Options Summary

| Option | Function |
|---|---|
| `-sL` | Show targets only (no scanning) |
| `-sn` | Host discovery only (no port scan) |
| `-PR` | ARP Scan |
| `-PE` / `-PP` / `-PM` | ICMP: Echo / Timestamp / Address Mask |
| `-PS` / `-PA` | TCP SYN Ping / TCP ACK Ping |
| `-PU` | UDP Ping |
| `-n` / `-R` | No DNS / DNS for all hosts |
| `-Pn` | Skip discovery and treat every host as up |

### 1.6 Answered Questions — Room 1

#### Theory Questions (fixed answers)

| Question | Answer | Explanation |
|---|---|---|
| What's the first IP Nmap scans for target `10.10.12.13/29`? | **10.10.12.8** | /29 = 8 addresses; the block containing .13 is .8 to .15, and the first is .8 |
| How many IPs will Nmap scan for `10.10.0-255.101-125`? | **6400** | (256) × (25) = 6400 |
| ARP Scan option | `-PR` (with `-sn`) | |
| ICMP Echo option | `-PE` | |
| ICMP Timestamp option | `-PP` | |
| ICMP Address Mask option | `-PM` | |
| TCP SYN Ping option | `-PS` | |
| TCP ACK Ping option | `-PA` | |
| UDP Ping option | `-PU` | |
| TCP SYN Ping against the Telnet port | `-PS23` | Telnet = Port 23 |
| Which TCP Ping does **not** need root? | **TCP SYN Ping** | |
| Which TCP Ping needs root? | **TCP ACK Ping** | |
| The fastest scanning tool mentioned | **masscan** | |
| Option for Reverse DNS on every host | `-R` | |
| Option to disable DNS lookups | `-n` | |
| Which TCP/IP layer is ARP in? | **Link** | |
| Which layer are ICMP and IP in? | **Internet** | |
| Which layer are TCP and UDP in? | **Transport** | |

**How to verify the first two calculations yourself:**

```bash
nmap -sL -n 10.10.12.13/29 | head -n 3                # first IP
nmap -sL -n 10.10.0-255.101-125 | tail -n 1           # last line: "Nmap done: 6400 IP addresses"
```

#### Practical Questions (depend on your own Lab)

| What the question asks | Command | How to read the answer |
|---|---|---|
| Number of hosts replying to ARP | `sudo nmap -PR -sn MACHINE_IP/24` | last line: `Nmap done: 256 IP addresses (N hosts up)` |
| Number of hosts replying to ICMP Echo | `sudo nmap -PE -sn MACHINE_IP/24` | same approach |
| Number of hosts replying to ICMP Timestamp | `sudo nmap -PP -sn MACHINE_IP/24` | same approach |
| Number of hosts replying to TCP SYN / ACK | `sudo nmap -PS -sn ...` and `-PA` | same approach |
| Number of hosts in a given subnet | `sudo nmap -sn SUBNET/24` | same approach |
| Hostname returned from DNS | `nmap -R -sn MACHINE_IP/24` | shown next to the IP |

> **Note:** when a question asks for the number of hosts, read the `(N hosts up)` phrase. If your result differs between types, that's normal — each firewall blocks a different type.

---
<a id="room2"></a>

## 2) Room 2: Nmap Basic Port Scans

**Goal:** Once we know which hosts are alive, we find out **which ports are open** and what that means. Here we learn the three basic scan types: **TCP Connect, TCP SYN, UDP**, then how to control scan scope and speed.

### 2.1 The Six Port States

This is the single most important table in the whole Room. Nmap classifies every port into one of **6 states**:

| State | Meaning | What happened? |
|---|---|---|
| **open** | A service is listening and accepting connections | A positive reply arrived (e.g., SYN/ACK) |
| **closed** | The port responds but no service is listening | A `RST` arrived |
| **filtered** | Nmap can't tell, because something (a Firewall) is blocking the packets | No reply, or an ICMP error |
| **unfiltered** | The port is reachable but Nmap can't tell if it's open or closed | Only shows up in an ACK scan |
| **open\|filtered** | Nmap can't distinguish between open and blocked | No reply (common with UDP and Null/FIN/Xmas scans) |
| **closed\|filtered** | Can't distinguish between closed and blocked | Rare (shows up in Idle Scan) |

> **Remember:** `closed` isn't a problem — it's information: *the host is alive with no firewall on that port*. `filtered` is the first sign a firewall is present.

### 2.2 TCP Connect Scan (Option `-sT`)

```bash
nmap -sT 10.10.10.5
```

**How it works:** completes the full **3-Way Handshake**, then tears down the connection — exactly like any regular application would.

```
 Open Port                             Closed Port
  Nmap --SYN--------> Target           Nmap --SYN--------> Target
  Nmap <--SYN/ACK---- Target           Nmap <--RST-------- Target
  Nmap --ACK--------> Target           (state: closed)
  Nmap --RST/ACK ---> Target
  (state: open)
```

| Pros | Cons |
|---|---|
| No root required | **Noisy:** the service logs it (since the connection completed) |
| Works everywhere | Slower than SYN Scan |

### 2.3 TCP SYN Scan (Option `-sS`) — "Half-Open" or "Stealth Scan"

```bash
sudo nmap -sS 10.10.10.5
```

**How it works:** sends only SYN. If the target replies with SYN/ACK, Nmap immediately kills the connection with `RST` **before** completing the handshake.

```
 Open Port                             Closed Port                   Blocked
  Nmap --SYN--------> Target           Nmap --SYN--------> Target     Nmap --SYN--> (Firewall)
  Nmap <--SYN/ACK---- Target           Nmap <--RST-------- Target     (no reply)
  Nmap --RST--------> Target           (closed)                       (filtered)
  (open)
```

| Pros | Cons |
|---|---|
| **Faster** than Connect | Needs root (Raw Packets) |
| **Stealthier:** never finishes the connection, so some applications won't log it | Doesn't hide from modern Firewalls and IDS |
| **Default** when running Nmap as root | |

> **Why is it called "Stealth"?** Because the application layer never sees a completed connection. Today's IDS and firewalls detect it easily — the name is more historical than literally true anymore.

### 2.4 UDP Scan (Option `-sU`)

```bash
sudo nmap -sU 10.10.10.5
sudo nmap -sU --top-ports 20 10.10.10.5     # faster: only the top 20 ports
```

**How it works:** sends a UDP packet to the port.

| Reply | State |
|---|---|
| A reply from the service (UDP) | **open** |
| No reply at all | **open\|filtered** (could be open and silent, or blocked) |
| **ICMP Port Unreachable** (Type 3, Code 3) | **closed** |
| Other ICMP Unreachable codes (1, 2, 9, 10, 13) | **filtered** |

**Why is UDP slow?** Because Linux rate-limits ICMP messages (roughly 1 per second), so determining `closed` takes time. Silence also triggers retransmissions. So **don't scan all 65535 UDP ports** — use `--top-ports`.

> **Tip:** many UDP services only reply to valid data. That's why Nmap sends special payloads for well-known ports (like DNS and SNMP). Combining UDP scan with `-sV` gives better accuracy.

**Top UDP services worth hunting for in CTFs:** DNS (53), SNMP (161), TFTP (69), NTP (123), DHCP (67/68).

### 2.5 Controlling Scan Scope

By default, Nmap scans only the **top 1000 most common ports** (not ports 1 through 1000!).

| Option | Function |
|---|---|
| `-p22` | single port |
| `-p22,80,443` | multiple ports |
| `-p1-1023` | a range |
| `-p-` | **all** 65535 ports |
| `-F` | **Fast**: top **100** ports |
| `--top-ports 10` | top N ports |
| `-r` | scan in sequential (not random) order |
| `--open` | show only open ports |

```bash
sudo nmap -sS -p- 10.10.10.5                 # all ports (best for CTFs)
sudo nmap -sS -p22,80,443 10.10.10.5
sudo nmap -sS --top-ports 10 10.10.10.5
sudo nmap -F 10.10.10.5
```

### 2.6 Controlling Speed (Timing and Performance)

#### Timing Templates `-T0` to `-T5`

| Template | Name | Use case |
|---|---|---|
| `-T0` | **paranoid** | extremely slow (minutes between packets), for evading IDS |
| `-T1` | **sneaky** | slow, for evasion |
| `-T2` | **polite** | reduces load on the target |
| `-T3` | **normal** | **the default** |
| `-T4` | **aggressive** | fast. **Best** for CTFs and healthy networks |
| `-T5` | **insane** | fastest, but may give wrong results or drop connections |

Can be written by name or number: `-T4` or `-T aggressive`.

#### Fine-grained Options

| Option | Function |
|---|---|
| `--min-rate 100` | send **at least** 100 packets/sec |
| `--max-rate 50` | don't send **more than** 50 packets/sec |
| `--min-parallelism 100` | minimum number of concurrent probes |
| `--max-parallelism 1` | one probe at a time (slow and quiet) |
| `--max-retries 1` | number of retries |
| `--host-timeout 5m` | give up on a host after 5 minutes |

> **Practical tip:** `--min-rate 5000` with `-p-` cuts scanning all ports from minutes to seconds in lab environments. But on real networks it may drop packets or trigger IDS alerts.

### 2.7 Attacker and Defender Perspective

| | Attacker | Defender |
|---|---|---|
| **Choosing a type** | `-sS` is the fastest. `-sT` if they don't have root. `-sU` for often-forgotten services like SNMP and TFTP | Watches for lots of SYN packets without an ACK, or half-open connections, and lots of RST from one source |
| **Strongest defense** | | **Default Deny**: only open what you need, and monitor logs. And don't forget UDP! |

### 2.8 Answered Questions — Room 2

#### Theory Questions

| Question | Answer |
|---|---|
| How many port states does Nmap recognize? | **6** (open, closed, filtered, unfiltered, open\|filtered, closed\|filtered) |
| Which service uses UDP Port 53? | **DNS** |
| Which service uses TCP Port 22? | **SSH** |
| Which service uses TCP Port 80? | **HTTP** |
| What's the flag name meaning "Reset"? | **RST** |
| What's the flag meaning "Acknowledge"? | **ACK** |
| What's the flag that starts a connection? | **SYN** |
| TCP Connect Scan option | `-sT` |
| TCP SYN Scan option | `-sS` |
| UDP Scan option | `-sU` |
| Which needs root: `-sT` or `-sS`? | `-sS` (needs Raw Packets) |
| Expected reply from a closed TCP port on SYN? | **RST** (usually RST/ACK) |
| Expected reply from a closed UDP port? | **ICMP Port Unreachable** |
| Option to scan the top 100 ports | `-F` |
| Option to scan all ports | `-p-` |
| Option to scan the top 10 ports | `--top-ports 10` |
| `-T4` template name | **aggressive** |
| `-T5` template name | **insane** |
| `-T0` template name | **paranoid** |
| Option to set minimum send rate | `--min-rate` |
| Option to set maximum send rate | `--max-rate` |

#### Practical Questions (depend on your own Lab)

| What the question asks | Command | How to read the answer |
|---|---|---|
| Which ports are open? | `sudo nmap -sS -p- --open MACHINE_IP` | read the `PORT` column where state is `open` |
| Any open port in a given range? | `sudo nmap -sS -p1-1023 MACHINE_IP` | |
| What's the service on a given port? | `sudo nmap -sV -p PORT MACHINE_IP` | `SERVICE` and `VERSION` columns |
| Did a port show up as filtered? | `sudo nmap -sS MACHINE_IP` | look for `filtered` |
| Open UDP ports | `sudo nmap -sU --top-ports 100 MACHINE_IP` | look for `open` (and `open\|filtered`) |
| How many ports in a given state? | | read the `Not shown: N ...` line |

**Trick to see the reasoning:** add `--reason` to see why Nmap considered a port open or closed (e.g., `syn-ack` or `reset`).

---
<a id="room3"></a>

## 3) Room 3: Nmap Advanced Port Scans

**Goal:** advanced scans that exploit subtle details in the TCP RFC to evade some simple (Stateless) firewalls, plus Spoofing and Idle Scan techniques.

> **Important note before starting:** the scans in this Room (Null, FIN, Xmas) only work reliably against systems that strictly follow RFC 793. **Windows and Cisco systems often don't follow it** and reply RST to everything, so every port looks "closed" even if it's open. These scans succeed more often against older Unix/Linux systems.

### 3.1 The Golden Rule Behind Null, FIN, and Xmas

These three scans **never send a SYN at all**. Instead they send an "odd" packet that doesn't belong to any existing connection, relying on a rule from **RFC 793**:

> If a **closed** port receives a TCP packet without SYN, ACK, or RST set, it must reply with **RST**.
> If the port is **open**, it silently ignores the odd packet (**no reply**).

So:

| Result | State |
|---|---|
| A **RST** came back | **closed** |
| **No reply** | **open\|filtered** (can't distinguish open from blocked) |

### 3.2 Null Scan (Option `-sN`)

```bash
sudo nmap -sN 10.10.10.5
```

Sends a packet with **no flags set at all** (every bit = zero).

### 3.3 FIN Scan (Option `-sF`)

```bash
sudo nmap -sF 10.10.10.5
```

Sends a packet with **only the FIN flag** set. The idea: FIN means "end the connection," but there's no connection to begin with, so a compliant system should reply RST (if it strictly follows the RFC).

### 3.4 Xmas Scan (Option `-sX`)

```bash
sudo nmap -sX 10.10.10.5
```

Sends a packet with **FIN + PSH + URG** all set together. Named "Christmas" because the packet "lights up" with every flag, like a Christmas tree covered in lights.

**Quick comparison table:**

| Type | Option | Flags Sent |
|---|---|---|
| Null | `-sN` | (none) |
| FIN | `-sF` | FIN |
| Xmas | `-sX` | FIN, PSH, URG |

### 3.5 TCP Maimon Scan (Option `-sM`)

```bash
sudo nmap -sM 10.10.10.5
```

Sends **FIN/ACK** together. Discovered by researcher Uriel Maimon. On some old BSD systems this made the distinction between open and closed clear, but it's rarely useful against modern systems today.

### 3.6 TCP ACK Scan (Option `-sA`) — Not for Detecting "Open/Closed"!

```bash
sudo nmap -sA 10.10.10.5
```

This scan has a **completely different purpose** from the previous ones: it does **not** tell you whether a port is open — it's used to map out firewall rules (**Firewall Rule Mapping**).

| Reply | Conclusion |
|---|---|
| **RST** | **unfiltered** — the packet got through, there's no firewall blocking this port (but we still don't know open vs. closed!) |
| **No reply / ICMP error** | **filtered** — something is blocking the packet |

**When to use it?** To determine whether a firewall is **Stateful** (tracks connection state, so it blocks an ACK with no prior SYN) or **Stateless** (only filters by port number, so it lets any ACK through).

### 3.7 Custom Scan (Option `--scanflags`)

```bash
sudo nmap --scanflags SYNACK 10.10.10.5
sudo nmap --scanflags SYNFINPSHURG 10.10.10.5
```

Lets you **build your own custom combination of flags**, to test firewall behaviors not covered by the built-in scan types.

### 3.8 Spoofing and Decoys

#### IP Spoofing (Option `-S`)

```bash
sudo nmap -S SPOOFED_IP -e eth0 -Pn 10.10.10.5
```

Forges the source IP address. **The problem:** replies go to the spoofed IP, not to you, so you lose visibility into the result unless you're sniffing on a network that allows it (mostly used for education rather than practical use against the internet).

#### Decoys (Option `-D`)

```bash
sudo nmap -D decoy1,decoy2,ME,decoy3 10.10.10.5
sudo nmap -D RND:10 10.10.10.5          # 10 random addresses
```

Sends the real scan mixed in with fake scans that appear to come from other addresses (Decoys), making it hard for a Firewall/IDS to identify which IP is the real source. `ME` marks where your real address sits in the list.

#### MAC Spoofing (Option `--spoof-mac`)

```bash
sudo nmap --spoof-mac 00:11:22:33:44:55 10.10.10.5
sudo nmap --spoof-mac Apple 10.10.10.5     # a random vendor
sudo nmap --spoof-mac 0 10.10.10.5         # fully random
```

Changes the source MAC address (only useful on the local network, since MAC doesn't cross Routers).

#### Source Port Spoofing (Option `-g` or `--source-port`)

```bash
sudo nmap -g 53 10.10.10.5
```

Makes the scan appear to come from a well-known, trusted port (like 53 for DNS), to fool simple firewalls that trust traffic coming from that port.

### 3.9 Idle (Zombie) Scan — Option `-sI`

```bash
sudo nmap -sI ZOMBIE_IP:PORT TARGET_IP
```

The cleverest technique in this Room for total stealth. It needs a third, "sleeping" device (**Zombie**) with a **predictable, sequential IPID** and no traffic of its own at that moment.

**The idea in three steps:**

```
Step 1: Send a SYN/ACK to the Zombie, record its current IPID from the reply (RST)
          Attacker --SYN/ACK--> Zombie
          Attacker <---RST------ Zombie   (IPID = X)

Step 2: Send a "spoofed" SYN packet to the target, claiming it comes from the Zombie
          Attacker --SYN (spoofed source = Zombie)--> Target

Step 3: Send another SYN/ACK to the Zombie and compare the new IPID
          Attacker --SYN/ACK--> Zombie
          Attacker <---RST------ Zombie   (IPID = ?)
```

**The explanation:**
- If the target's port is **open**: the target sends a SYN/ACK to the Zombie (since it thinks the request came from it), and the Zombie automatically replies with RST, which **increases the IPID by 2** (one extra packet the Zombie sent).
- If the target's port is **closed**: the target sends RST directly to the Zombie, and the Zombie **ignores it** (it doesn't reply to an unsolicited RST), so the IPID **only increases by 1** (the natural increment from our own step).

**Why is it so powerful?** Because the target never sees your real address at all — only the Zombie's. But it needs a suitable Zombie (an old system with Sequential IPID), which is rare today in modern systems that use randomized IPID.

### 3.10 Attacker and Defender Perspective

| | Attacker | Defender |
|---|---|---|
| **Null/FIN/Xmas** | Trying to bypass simple Stateless firewalls | A modern **Stateful** firewall blocks these regardless of flags |
| **ACK Scan** | Used to understand firewall structure before choosing an approach | Watch for ACK packets arriving with no prior SYN connection |
| **Decoys/Spoofing** | Spreads out logs and complicates attribution | Compare packet timing across all IPs — the real one tends to be consistent in timing |
| **Idle Scan** | Fully hides identity behind an innocent machine | Watch for any device being used as a Zombie (unprompted SYN/ACK traffic from a strange IP) |

### 3.11 Answered Questions — Room 3

#### Theory Questions

| Question | Answer |
|---|---|
| Null Scan option | `-sN` |
| FIN Scan option | `-sF` |
| Xmas Scan option | `-sX` |
| Maimon Scan option | `-sM` |
| ACK Scan option | `-sA` |
| What flags are in a Xmas Scan? | **FIN, PSH, URG** |
| What flags are in a Null Scan? | **None (zero flags)** |
| Expected reply from a closed port in Null/FIN/Xmas (per RFC 793)? | **RST** |
| Expected reply from an open port in Null/FIN/Xmas? | **No reply** (state: open\|filtered) |
| Which OS commonly doesn't follow RFC 793, breaking these scans? | **Windows** (and Cisco network devices) |
| What's the main purpose of TCP ACK Scan? | **Mapping firewall rules (determining unfiltered/filtered)**, not open/closed |
| Option to build custom flags | `--scanflags` |
| Option to spoof the source IP | `-S` |
| Option to use fake Decoy addresses | `-D` |
| Option to use random IPs as Decoys | `-D RND:N` |
| Option to spoof the MAC address | `--spoof-mac` |
| Option to spoof the Source Port | `-g` or `--source-port` |
| Idle / Zombie Scan option | `-sI` |
| Which field does Idle Scan rely on to determine the result? | **IP Identification (IPID)** |
| In Idle Scan, how much does the Zombie's IPID increase if the target's port is open? | **2** |
| In Idle Scan, how much does the Zombie's IPID increase if the target's port is closed? | **1** |
| What's the essential requirement for the Zombie host? | **Sequential/Incremental IPID** and idle (not sending other traffic) |

#### Practical Questions (depend on your own Lab)

| What the question asks | Command | How to read the answer |
|---|---|---|
| Null Scan result on the target | `sudo nmap -sN MACHINE_IP` | compare it against a normal `-sS` result; note the difference between `open\|filtered` and `closed` |
| Xmas Scan result | `sudo nmap -sX MACHINE_IP` | same approach |
| ACK Scan result (filtered/unfiltered) | `sudo nmap -sA MACHINE_IP` | STATE column: `unfiltered` or `filtered` |
| Can the Lab's firewall be bypassed with FIN Scan? | compare `nmap -sS` to `nmap -sF` on the same port | if the port shows "open/not blocked" in FIN but was blocked in SYN, the evasion worked |

---
<a id="room4"></a>

## 4) Room 4: Nmap Post Port Scans

**Goal:** now that we know which ports are open, we need deeper info: exactly which service, what OS, any known vulnerabilities, and how to save it all in an organized way.

### 4.1 Service and Version Detection (Option `-sV`)

```bash
nmap -sV 10.10.10.5
nmap -sV --version-intensity 9 10.10.10.5    # deeper, slower probing (0-9)
nmap -sV --version-light 10.10.10.5          # fast, light probing (low intensity)
nmap -sV --version-all 10.10.10.5            # try every probe (very thorough)
```

**How does it work under the hood?** The port number alone isn't proof of the service (someone could run SSH on port 8080, for example). Nmap sends a series of **Probes** (queries) and compares the responses against a massive database called **`nmap-service-probes`**, trying to extract:

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
```

- Product name
- Version number — the **most important detail** for linking to known vulnerabilities (CVEs)
- Sometimes the host OS running the service

### 4.2 OS Detection (Option `-O`)

```bash
sudo nmap -O 10.10.10.5
sudo nmap -O --osscan-guess 10.10.10.5     # guess the closest match even if uncertain
sudo nmap -O --osscan-limit 10.10.10.5     # only if at least one open and one closed port exist
```

**How does it work under the hood?** Requires **root**. Sends a set of very low-level tests (default TTL, TCP option ordering, IPID behavior, TCP Window Size...) and compares them against a database called **`nmap-os-db`** which holds fingerprints of thousands of systems.

```
Running: Linux 5.X
OS CPE: cpe:/o:linux:linux_kernel:5
OS details: Linux 5.0 - 5.4
Network Distance: 2 hops
```

> **Warning:** OS Detection needs **at least one open port and one closed port** to work accurately; otherwise it returns `OS detection performed. No exact OS matches...`.

### 4.3 Nmap Scripting Engine (NSE)

The most powerful feature in Nmap — a scripting engine written in **Lua** that extends the tool from a simple port scanner into an enumeration tool, and even a basic exploitation tool.

#### Script Categories

| Category | Function |
|---|---|
| **auth** | tests authentication mechanisms (trying to bypass them) |
| **broadcast** | discovers devices via Broadcast on the network |
| **brute** | brute-force attacks against credentials |
| **default** | the set that runs automatically with `-sC` |
| **discovery** | extracts extra information about the target and network |
| **dos** | tests susceptibility to denial-of-service attacks (be careful!) |
| **exploit** | attempts to actively exploit a vulnerability |
| **external** | sends data to external services (e.g., VirusTotal) |
| **fuzzer** | sends random data to discover bugs |
| **intrusive** | may cause harm or get detected — only use with explicit authorization |
| **malware** | detects Backdoors or Malware |
| **safe** | safe and won't affect the target |
| **version** | helps with version discovery |
| **vuln** | searches for and reports known vulnerabilities |

#### Ways to Run Them

```bash
nmap -sC 10.10.10.5                          # default scripts
nmap --script=default 10.10.10.5             # same effect as -sC
nmap --script=vuln 10.10.10.5                # a whole category
nmap --script=ftp-anon 10.10.10.5            # a specific named script
nmap --script=ftp-anon,http-title 10.10.10.5 # multiple scripts
nmap --script="http-*" 10.10.10.5            # all scripts starting with http
nmap --script=default,vuln 10.10.10.5        # combine categories
nmap --script-args user=admin,pass=admin 10.10.10.5   # pass arguments to a script
```

```bash
# update the script database
sudo nmap --script-updatedb

# location of scripts on disk (typically on Linux)
/usr/share/nmap/scripts/
```

> **Security warning:** the `exploit`, `dos`, `brute`, and `intrusive` categories can crash a service or lock accounts after repeated failed attempts. **Only use them in environments where you have explicit authorization** (like your own Lab).

### 4.4 The Traditional All-in-One Scan (Option `-A`)

```bash
sudo nmap -A 10.10.10.5
```

A shortcut that enables together: `-sV` (version detection) + `-O` (OS detection) + `-sC` (default scripts) + Traceroute. Comprehensive and powerful, but **slow and very noisy**, so it's best suited for labs rather than tests requiring stealth.

### 4.5 Saving Results (Output Formats)

| Option | Format | Use Case |
|---|---|---|
| `-oN file.txt` | Normal | exactly what you see on screen |
| `-oG file.txt` | Grepable | one line per target, easy to filter with `grep`/`awk` (old but quick) |
| `-oX file.xml` | XML | for automated processing and converting into an HTML report |
| `-oA basename` | **All three at once** | produces `basename.nmap`, `.gnmap`, and `.xml` in one go |

```bash
sudo nmap -sS -sV -oA scan_results 10.10.10.5
# produces: scan_results.nmap / scan_results.gnmap / scan_results.xml

# convert XML into a shareable HTML report
xsltproc scan_results.xml -o scan_report.html
```

**Why always save results?** In real engagements (and long CTFs) you scan dozens of machines; saving results gives you a reference instead of re-scanning, and allows automated processing later (such as feeding them into other tools).

### 4.6 Attacker and Defender Perspective

| | Attacker | Defender |
|---|---|---|
| **After finding open ports** | Runs `-sV` and `-sC` to pin down exact versions and link them to specific CVEs (via `searchsploit`, for example) | Watches for distinctive NSE traffic (unusual queries across multiple protocols from the same IP) |
| **OS Detection** | Determines which exploits are relevant (Windows has different vulnerabilities than Linux) | Hides or alters TTL/Banner to confuse scanners (honeypots do the opposite: deceive the attacker with a fake system) |
| **Documenting results** | Keeps `-oA` output from each stage; builds a pentest report from it | Analyzes IDS logs to see exactly what the attacker scanned for |

### 4.7 Answered Questions — Room 4

#### Theory Questions

| Question | Answer |
|---|---|
| Option for service and version detection | `-sV` |
| Option for OS detection | `-O` |
| Option to run default scripts | `-sC` |
| What's equivalent to `-sC`? | `--script=default` |
| Option for the all-in-one scan | `-A` |
| What does `-A` include? | **Service Version + OS Detection + Default Scripts + Traceroute** |
| The language NSE scripts are written in | **Lua** |
| The category that searches for known vulnerabilities | **vuln** |
| The category of safe, non-intrusive scripts | **safe** |
| The category for password-guessing/brute-force scripts | **brute** |
| The category for actively attempting exploitation | **exploit** |
| Database of OS fingerprints | **nmap-os-db** |
| Database of service fingerprints | **nmap-service-probes** |
| Option to save output in normal format | `-oN` |
| Option to save output in Grepable format | `-oG` |
| Option to save output in XML format | `-oX` |
| Option to save all three formats at once | `-oA` |
| Option to pass arguments to an NSE script | `--script-args` |
| Command to update the script database | `nmap --script-updatedb` |

#### Practical Questions (depend on your own Lab)

| What the question asks | Command | How to read the answer |
|---|---|---|
| Service version on a given port | `nmap -sV -p PORT MACHINE_IP` | `VERSION` column |
| Guessed operating system | `sudo nmap -O MACHINE_IP` | `Running:` or `OS details:` line |
| Result of a specific vuln script | `nmap --script=vuln -p PORT MACHINE_IP` | shows a CVE or vulnerability if found, under the port's name |
| HTTP home page title | `nmap --script=http-title -p 80 MACHINE_IP` | `http-title:` line |
| A full scan and saving it | `sudo nmap -A -oA full_scan MACHINE_IP` | check the `full_scan.nmap` file |

---
<a id="recap"></a>

## 5) Topic Transition Recap: Full Review and Workflow

This final section of the module ties everything together. The idea is that you're now able to build a full **Scanning Methodology** instead of just remembering isolated commands.

### 5.1 The Full Methodology, Step by Step

```bash
# ===== STAGE 1: Discover live hosts =====
sudo nmap -sn 10.10.10.0/24 -oN hosts_up.txt

# ===== STAGE 2: Quick scan of every TCP port =====
sudo nmap -sS -p- --min-rate 5000 -T4 -oN all_ports.txt 10.10.10.5

# ===== STAGE 3: Deep scan on open ports only (faster and more accurate) =====
# (pull the port numbers from the previous step's result, e.g.: 22,80,445)
sudo nmap -sS -sV -sC -p22,80,445 -oN deep_scan.txt 10.10.10.5

# ===== STAGE 4 (optional): UDP scan for top ports =====
sudo nmap -sU --top-ports 20 -oN udp_scan.txt 10.10.10.5

# ===== STAGE 5: OS detection + full scan when needed =====
sudo nmap -O -oN os_scan.txt 10.10.10.5

# ===== STAGE 6: Targeted scripts based on discovered service =====
nmap --script="ftp-*" -p21 10.10.10.5
nmap --script=vuln -p80,443 10.10.10.5

# ===== STAGE 7: Document everything =====
sudo nmap -A -oA final_report 10.10.10.5
```

> **Why does this order matter?** Scanning all ports (`-p-`) quickly first, then going deeper only on the ones that are open with `-sV -sC` = huge time savings. Running `-A -p-` from the start on a full network might take hours.

### 5.2 Mental Map: Which Option Do I Use When?

| I want to... | Use |
|---|---|
| Find live hosts on my local network (most accurate) | `sudo nmap -PR -sn <subnet>` |
| Find live hosts over the internet (remote) | `sudo nmap -PE -PS443 -sn <range>` |
| Scan every TCP port quickly | `sudo nmap -sS -p- --min-rate 5000` |
| Evade a simple firewall | try `-sF` / `-sN` / `-sX` and compare |
| Learn how accurate a firewall is (stateful or not) | `sudo nmap -sA` |
| Get the exact version of a service | `nmap -sV -p<port>` |
| Determine the OS | `sudo nmap -O` |
| Automatically search for known vulnerabilities | `nmap --script=vuln` |
| Hide my identity completely | `sudo nmap -sI <zombie>:<port>` |
| Document everything for a report | `-oA basename` |

### 5.3 A Full Real-World Scenario

Suppose you start scanning a new CTF machine (`10.10.10.50`) from scratch — here's your thought process in sequence:

1. **"Is the host up?"** -> `ping -c1 10.10.10.50` or `nmap -sn`.
2. **"What's open?"** -> `sudo nmap -p- --min-rate 5000 -T4 10.10.10.50` (fast and comprehensive).
3. **"I see ports 22, 80, 445. What are the details?"** -> `sudo nmap -sC -sV -p22,80,445 10.10.10.50`.
4. **"Port 80 = Apache 2.4.41. Any known vulnerability?"** -> `nmap --script=vuln -p80 10.10.10.50` or `searchsploit apache 2.4.41`.
5. **"Port 445 means SMB. Does it allow anonymous access?"** -> `nmap --script=smb-enum-shares,smb-os-discovery -p445 10.10.10.50`.
6. **"Did I miss anything in UDP?"** -> `sudo nmap -sU --top-ports 20 10.10.10.50`.
7. **"Document everything"** -> re-run the key steps with `-oA` and save to your project folder.

This is exactly what "Topic Transition" means: moving from knowing individual commands to **thinking like a penetration tester**, building each decision on the result of the previous step.

---

<a id="cheatsheet"></a>

## 6) Cheat Sheet: Every Command on One Page

### Host Discovery
```bash
nmap -sL -n 10.10.10.0/24                # show targets only (no scan)
sudo nmap -sn -PR 10.10.10.0/24          # ARP (local network, most accurate)
sudo nmap -sn -PE 10.10.10.0/24          # ICMP Echo
sudo nmap -sn -PP 10.10.10.0/24          # ICMP Timestamp
sudo nmap -sn -PM 10.10.10.0/24          # ICMP Address Mask
sudo nmap -sn -PS22,80,443 10.10.10.0/24 # TCP SYN Ping
sudo nmap -sn -PA80 10.10.10.0/24        # TCP ACK Ping
sudo nmap -sn -PU53 10.10.10.0/24        # UDP Ping
```

### Basic Port Scanning
```bash
nmap -sT 10.10.10.5                      # TCP Connect (no root needed)
sudo nmap -sS 10.10.10.5                 # TCP SYN (default with root)
sudo nmap -sU 10.10.10.5                 # UDP Scan
sudo nmap -sS -p- 10.10.10.5             # all 65535 ports
sudo nmap -F 10.10.10.5                  # top 100 ports
nmap --top-ports 20 10.10.10.5           # top 20 ports
sudo nmap -sS -p- --min-rate 5000 -T4 10.10.10.5   # very fast
```

### Advanced / Evasion Scans
```bash
sudo nmap -sN 10.10.10.5                 # Null Scan
sudo nmap -sF 10.10.10.5                 # FIN Scan
sudo nmap -sX 10.10.10.5                 # Xmas Scan
sudo nmap -sM 10.10.10.5                 # Maimon Scan
sudo nmap -sA 10.10.10.5                 # ACK Scan (firewall mapping)
sudo nmap --scanflags SYNFIN 10.10.10.5  # custom flags
sudo nmap -D RND:5 10.10.10.5            # random Decoys
sudo nmap --spoof-mac 0 10.10.10.5       # random MAC
sudo nmap -sI zombie_ip:port target_ip   # Idle (Zombie) Scan
```

### Service/OS Detection and Scripts
```bash
nmap -sV 10.10.10.5                      # version detection
sudo nmap -O 10.10.10.5                  # OS detection
nmap -sC 10.10.10.5                      # default scripts
nmap --script=vuln 10.10.10.5            # search for vulnerabilities
nmap --script=ftp-anon -p21 10.10.10.5   # a specific script
sudo nmap -A 10.10.10.5                  # everything at once
```

### Output and Speed Control
```bash
sudo nmap -sS -sV -oA results 10.10.10.5 # save in all formats
nmap -T4 10.10.10.5                      # fast (best for CTFs)
nmap --min-rate 5000 10.10.10.5          # high send rate
nmap -n 10.10.10.5                       # no DNS lookup (faster)
nmap --reason 10.10.10.5                 # show the reason for each state
nmap --open 10.10.10.5                   # show only open ports
```

---

<a id="troubleshooting"></a>

## 7) Common Mistakes and How to Fix Them

| Problem | Likely Cause | Fix |
|---|---|---|
| `Host seems down` even though you're sure it's up | A Firewall is blocking ICMP Ping | Add `-Pn` to skip discovery and scan directly |
| Results are very slow | Scanning all ports with default settings | Use `--min-rate 5000 -T4` or scan `--top-ports` first |
| `-sS` doesn't work / a permissions-related message | Running without root | Add `sudo` before the command |
| UDP ports are all `open\|filtered` | Normal UDP behavior (no reply doesn't confirm anything) | Add `-sV` to send payloads that trigger a real reply |
| Null/FIN/Xmas report everything as "closed" or "open" indiscriminately | The target is Windows and doesn't follow RFC 793 | These scans are unreliable against Windows; use `-sS` instead |
| `-O` returns "No exact OS matches" | Not enough open/closed port combination to compare against | Scan more ports, or use `--osscan-guess` |
| Results differ between runs | Unstable network or IDS interference | Add `--max-retries 2` and slow down with `-T2` |
| `--script=vuln` is very slow | Trying many scripts against many ports | Narrow it down to specific ports with `-p` instead of scanning everything |

---

<a id="resources"></a>

## 8) References and Learning Resources

- **Official Nmap Reference Guide:** `https://nmap.org/book/man.html`
- **The full official Nmap book, free online:** `https://nmap.org/book/toc.html`
- **Official NSE script database:** `https://nmap.org/nsedoc/`
- **Official training site for legal practice:** `scanme.nmap.org`
- **TryHackMe — Nmap module:** the five Rooms this guide covers
- **RFC 793 (the original TCP specification):** reference for understanding why Null/FIN/Xmas scans work the way they do
- **searchsploit (from Exploit-DB):** for linking `-sV` results to known vulnerabilities
- **HackTricks — Pentesting Network section:** an excellent reference for practical Nmap commands with real scenarios

---

<a id="glossary"></a>

## 9) Glossary

| Term | Explanation |
|---|---|
| **Host Discovery** | the phase of identifying live hosts on the network before scanning ports |
| **Port Scanning** | sending packets to each port to determine its state (open/closed/filtered) |
| **Three-Way Handshake** | the process of starting a TCP connection: SYN -> SYN/ACK -> ACK |
| **Stateful Firewall** | tracks the full connection state (knows an ACK with no prior SYN is suspicious) |
| **Stateless Firewall** | filters only by port/IP number without tracking connection state |
| **IDS / IPS** | systems that detect/prevent intrusions by watching for suspicious traffic |
| **NSE** | Nmap Scripting Engine, the scripting engine built in Lua |
| **CVE** | a globally standardized identifier for a known security vulnerability |
| **Zombie Host** | an idle device used as an intermediary in an Idle Scan attack |
| **IPID** | a unique identifier for each IP packet, used to track sequencing in an Idle Scan |
| **Banner Grabbing** | extracting service information from the welcome message (Banner) sent upon connection |
| **Enumeration** | gathering detailed information about a service or system after discovering it |

---

### Final Word

Nmap is a massive tool, and this guide covers everything you need to start strong and understand **why** each command works, not just **how** to type it. Practice on `scanme.nmap.org` and on your own TryHackMe/HackTheBox machines, and always remember: **understanding > memorizing**.

**Good luck on your penetration testing journey 🛡️**
