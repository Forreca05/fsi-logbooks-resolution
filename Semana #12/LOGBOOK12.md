# Packet Sniffing and Spoofing Lab

## Setup

The lab environment was configured using Docker containers to simulate a network of three machines. To allow the attacker container to sniff traffic from other nodes, it was set to "host mode." We identified the bridge interface created by Docker to ensure our Scapy scripts targeted the correct network.

```bash
# Build the container images
$ dcbuild

# Start the containers in detached mode
$ dcup -d

# Identify the network interface 
$ ifconfig
# Identified interface: br-7f69ed284fb2 (IP: 10.9.0.1)
```
![Interface
_name](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/4fe3953e7b625b151aa230d14acd27028fee17a8/Semana%20%2312/Images/interface_name.png)
---

## Lab Task Set 1: Scapy (Sniffing and Spoofing)

### Using Scapy

To familiarize ourselves with the tool before performing specific attacks, we used Scapy to construct a simple IP object and inspect its default fields.
```bash
scapy
a = IP()
a.show()
```
![using scapy](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/4fe3953e7b625b151aa230d14acd27028fee17a8/Semana%20%2312/Images/using_scappy.png)
## Task 1.1: Sniffing Packets

### Task 1.1A - Root vs. User Privileges

#### Objective
The goal was to verify that packet sniffing requires administrative privileges due to the nature of raw socket access.

#### Execution
Since the containers are silent by default, we first generated traffic by pinging 8.8.8.8 from hostA. Simultaneously, we ran the following Scapy script on the attacker machine to capture these ICMP packets:


```python
#!/usr/bin/env python3
from scapy.all import *

def print_pkt(pkt):
    pkt.show()


pkt = sniff(iface='br-7f69ed284fb2', filter='icmp', prn=print_pkt)
```

#### Result
* **Running as regular user:** The script failed with a permission error.
* **Running with sudo:** The script successfully captured packets.
* **Packet Layers Analysis:** The pkt.show() output detailed the encapsulated layers:

    1. Ethernet: Showing MAC addresses of the container and gateway.

    2. IP: Displaying Source (10.9.0.5) and Destination (8.8.8.8) IPs.

    3. ICMP: Identifying the type (Echo Request) and code.

* **Explanation:** Sniffing requires "Raw Sockets," which allow an application to interact directly with the network layer. To prevent unauthorized users from eavesdropping on network traffic, operating systems restrict this capability to the root user.

![no_root_privilege](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/4fe3953e7b625b151aa230d14acd27028fee17a8/Semana%20%2312/Images/without_root_privilege.png)

### Task 1.1B - BPF Filters

#### Objective
To apply Berkeley Packet Filter (BPF) syntax to capture specific subsets of network traffic.

#### Execution
We tested three different filters by modifying the `sniff` function:

1. **ICMP Packets only:**
   ```python
   pkt = sniff(iface='br-7f69ed284fb2', filter='icmp', prn=print_pkt)
   ```
![icmp_packets](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/4fe3953e7b625b151aa230d14acd27028fee17a8/Semana%20%2312/Images/icmp_packets.png)
2. **TCP from 10.9.0.5 to port 23:**
   ```python
   pkt = sniff(iface='br-7f69ed284fb2', filter='tcp and src host 10.9.0.5 and dst port 23', prn=print_pkt)
   ```
![tcp_packets](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/4fe3953e7b625b151aa230d14acd27028fee17a8/Semana%20%2312/Images/tcp_packets.png)
3. **Subnet 128.230.0.0/16:**
   ```python
   pkt = sniff(iface='br-7f69ed284fb2', filter='net 128.230.0.0/16', prn=print_pkt)
   ```
![particular_subnet](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/4fe3953e7b625b151aa230d14acd27028fee17a8/Semana%20%2312/Images/particular_subnet.png)
#### Result
The sniffer successfully filtered the packets, showing only the traffic that matched our BPF expressions.

---

## Task 1.2: Spoofing ICMP Packets

### Objective
To forge an IP packet and send it to an arbitrary destination.

### Execution
We constructed an IP packet with an ICMP layer. We used `ls(a)` to inspect the fields and sent the packet to the destination `1.2.3.4`.

```python
#!/usr/bin/python3
from scapy.all import *
a = IP()
a.dst = '1.2.3.4'
b = ICMP()
p = a/b
ls(a)
send(p, iface='br-7f69ed284fb2')
```

### Result
The packet was successfully injected into the network. Wireshark confirmed that the ICMP Echo Request was sent to `1.2.3.4` through the bridge interface as intended.
![task1.2](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/4fe3953e7b625b151aa230d14acd27028fee17a8/Semana%20%2312/Images/task1.2.png)

---

## Task 1.3: Traceroute

### Objective
To implement a custom Traceroute tool using Scapy to discover the path to a remote destination (`8.8.8.8`).

### Methodology
Traceroute works by sending packets with an increasing Time-To-Live (TTL).
* When a router receives a packet with TTL=1, it decrements it to 0, drops the packet, and returns an ICMP Type 11 (Time Exceeded).
* When the packet reaches the destination, it returns an ICMP Type 0 (Echo Reply).

### Execution
We implemented the following function to automate the discovery:

```python
#!/usr/bin/python3
from scapy.all import *

def tracer(dst, i = 30):
    a = IP()
    b = ICMP()
    a.dst = dst
    for ttl in range(1, i + 1):
        a.ttl = ttl
        pkt = a/b
        # sr1 sends and waits for the first response
        reply = sr1(pkt, timeout = 2)
        if reply is None:
            print(ttl, ": Request timed out")
        elif reply.type == 0:
            # Type 0 is Echo Reply: we reached the target
            print(ttl, ":", reply.src)
            print("Destination Reached")
            break
        elif reply.type == 11:
            # Type 11 is Time Exceeded: a router dropped the packet
            print(ttl, ":", reply.src)
        else:
            print(ttl, ": Unexpected reply type: ", reply.type)

tracer("8.8.8.8")
```

### Result
The script successfully traced the route hop-by-hop. We observed the IP addresses of intermediate routers (Type 11) until the destination server responded with a Type 0 packet, ending the loop.

![Task1.3](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/4fe3953e7b625b151aa230d14acd27028fee17a8/Semana%20%2312/Images/task1.3.png)

## Task 1.4: Sniffing and-then Spoofing
 
### Objective
To combine the previous techniques into a single automated program. The goal is to sniff for ICMP Echo Requests on the network and immediately send a spoofed Echo Reply, effectively making any target IP appear "alive" to the sender.
### Execution
We wrote a Python program using Scapy that sniffs for ICMP Echo Requests (Type 8) on the network interface and automatically crafts an Echo Reply (Type 0) swapping the source and destination IP addresses.
```python
#!/usr/bin/python3
from scapy.all import *

def spoof_icmp(pkt):
    # Check if the packet is an ICMP Echo Request (Type 8)
    if(pkt[ICMP].type == 8):
        ip = pkt[IP]
        icmp = pkt[ICMP]

        # Construct the reply:
        # 1. Swap Source and Destination IPs
        # 2. Set ICMP Type to 0 (Echo Reply)
        # 3. Copy ID and Sequence number to match the request
        # 4. Include the original payload (Raw layer)
        reply = IP(src=ip.dst, dst=ip.src) / ICMP(type=0, id=icmp.id, seq=icmp.seq) / pkt[Raw]

        send(reply)
prn=spoof_icmp
# Sniff for ICMP packets and trigger the spoof_icmp callback
pkt = sniff(iface='br-7f69ed284fb2', filter='icmp', prn=prn)
```
### Observations
We pinged three distinct types of IP addresses from the user container (10.9.0.5) while our sniffing-and-spoofing program was running on the attacker container (10.9.0.1). The results were as follows:

### Pinging 1.2.3.4 (Non-existing Internet IP)
**Result:** The ping command received a reply successfully.
**Analysis:** Since this IP address does not belong to the local network, the victim machine forwards the packet to its default gateway. Our sniffer captured this outgoing ICMP Echo Request and immediately injected a spoofed Echo Reply. As a result, the victim believes 1.2.3.4 is alive, even though it does not exist.
![ping 1.2.3.4](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/4fe3953e7b625b151aa230d14acd27028fee17a8/Semana%20%2312/Images/ping%201.2.3.4.png)

### Pinging 10.9.0.99 (Non-existing Local IP)
**Result:** The ping command failed to receive a reply ("Destination Host Unreachable"), even with our program running.
**Analysis:** Observing the Wireshark logs reveals a distinct pattern for this case. Instead of ICMP packets, we see repeated ARP Broadcast Requests asking "Who has 10.9.0.99?".
This happens because the target is on the local subnet. The OS tries to resolve the MAC address via ARP before creating the IP packet. Since the host 10.9.0.99 does not exist, no ARP Reply is received, and the OS never sends the ICMP packet. Consequently, our code (which filters for ICMP) never sees a trigger packet and sends no reply. To make this work, we would also need to spoof ARP replies.
![ping 10.9.0.99](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/4fe3953e7b625b151aa230d14acd27028fee17a8/Semana%20%2312/Images/ping%2010.0.9.99.png)

### Pinging 8.8.8.8 (Existing Internet IP)

**Result:** The ping command received replies, but Wireshark showed duplicate packets.
**Analysis:** In this case, two responses were generated:

1. One legitimate reply from the real Google DNS server (8.8.8.8).

2. One spoofed reply from our program confirming that our attack does not block legitimate traffic, it simply races against the real server to inject a packet. The victim machine receives both, often accepting the one that arrives first or displaying both as duplicates.

![ping 8.8.8.8](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/4fe3953e7b625b151aa230d14acd27028fee17a8/Semana%20%2312/Images/ping%208.8.8.8.png)