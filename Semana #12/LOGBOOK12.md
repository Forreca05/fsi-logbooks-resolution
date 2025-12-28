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

---

## Lab Task Set 1: Scapy (Sniffing and Spoofing)

## Task 1.1: Sniffing Packets

### Task 1.1A - Root vs. User Privileges

#### Objective
The goal was to verify that packet sniffing requires administrative privileges due to the nature of raw socket access.

#### Execution
We utilized the following Scapy script to sniff ICMP packets:

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
* **Explanation:** Sniffing requires "Raw Sockets," which allow an application to interact directly with the network layer. To prevent unauthorized users from eavesdropping on network traffic, operating systems restrict this capability to the root user.

### Task 1.1B - BPF Filters

#### Objective
To apply Berkeley Packet Filter (BPF) syntax to capture specific subsets of network traffic.

#### Execution
We tested three different filters by modifying the `sniff` function:

1. **ICMP Packets only:**
   ```python
   pkt = sniff(iface='br-7f69ed284fb2', filter='icmp', prn=print_pkt)
   ```

2. **TCP from 10.9.0.5 to port 23:**
   ```python
   pkt = sniff(iface='br-7f69ed284fb2', filter='tcp and src host 10.9.0.5 and dst port 23', prn=print_pkt)
   ```

3. **Subnet 128.230.0.0/16:**
   ```python
   pkt = sniff(iface='br-7f69ed284fb2', filter='net 128.230.0.0/16', prn=print_pkt)
   ```

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