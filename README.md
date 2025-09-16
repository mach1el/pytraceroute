# pytraceroute ![VERSION](https://img.shields.io/badge/version-1.0-violet.svg)

![Python](https://img.shields.io/badge/python-3670A0?style=for-the-badge&logo=python&logoColor=ffdd54)

This Python Traceroute Tool is a simple and effective network utility that traces the path data packets take from your machine to a specified target host. It provides a detailed breakdown of each hop along the route, helping users diagnose network performance issues, identify potential bottlenecks, and analyze the connectivity between different points in the network.

- UDP probes (fallback/no-root)
- TCP SYN-style probes (non-blocking connect) — useful when ICMP/UDP are filtered
- Pipelined TTL windows for faster traces (parallelization across TTLs)
- Async reverse DNS so name lookups don't block probe collection
- Stronger reply matching by parsing ICMP payloads (matches ICMP/UDP probes to originating probe)
- Cross-platform privilege checks (Unix + Windows) and graceful fallbacks

---

## Quick start

### Run from the command line

```bash
# ICMP mode (recommended — requires root/admin on most OSes)
sudo python3 pytraceroute.py example.com

# Force UDP mode (no root required)
python3 pytraceroute.py --udp example.com
```

### Example with options

ICMP mode (recommended when possible — requires admin/root on Unix or Administrator on Windows):
```bash
sudo python3 pytraceroute.py --icmp example.com
```

Force UDP probes (no root needed):
```bash
python3 pytraceroute.py --udp example.com
```

TCP probes (non-blocking connect) — useful when ICMP/UDP are filtered by firewalls:
```bash
python3 pytraceroute.py --tcp --tcp-port 443 example.com
```

Pipeline several TTLs in parallel (faster traces; default pipeline=4):
```bash
python3 pytraceroute.py --pipeline 6 --icmp example.com
```

Fastest (no DNS, immediate IPs only):
```bash
python3 pytraceroute.py --no-dns example.com
```

Default behavior: async reverse DNS is enabled (names printed later as `(resolved)`), ICMP is attempted if running as admin, otherwise UDP/TCP used as requested.

---

## Command-line options

```
usage: pytraceroute_enhanced.py [options] host

Options:
  -h, --help            show this help message and exit
  -m, --max-hops       maximum hops (default: 30)
  -q, --probes         probes per hop (default: 3)
  -w, --wait           per-window timeout seconds (default: 2.0)
  --udp                force UDP probes (do not use ICMP echo)
  --icmp               prefer ICMP probes (requires admin privileges)
  --tcp                enable TCP SYN/connect probes (non-blocking)
  --tcp-port           destination TCP port for TCP probes (default: 80)
  --pipeline           number of TTLs to pipeline (window size, default: 4)
  --select-chunk       select() time-slice in seconds (default 0.15)
  --no-dns             do not perform reverse DNS lookups (fastest)
  --no-async-dns       disable async DNS (use blocking DNS or none if --no-dns)
```

---

## Output format

The tracer prints one line per hop. Example output:

```
Traceroute to example.com (93.184.216.34), max hops 30, probes 3, timeout 2.0s
 1  router.example.net (192.0.2.1) 0.995s
 2  isp-gw.example.net (198.51.100.1) 1.321s
 3  * * *
 4  93.184.216.34 (93.184.216.34) 12.123s
Destination reached.
```

- `* * *` means no response was received for that hop within the timeout.
- When a hop responds, the resolver will attempt a reverse DNS lookup; if it fails the numeric IP is shown.

---

## Permissions & behavior

- **ICMP mode**: If you run the script as root/administrator, it will attempt to use ICMP raw sockets (classic traceroute behavior using ICMP Echo). This gives better matching of replies in many environments.
- **UDP fallback**: If raw sockets are unavailable (non-root), the tool automatically falls back to UDP probes so you can still trace without admin rights. You can also force UDP mode with `--udp`.
- **Windows note**: `os.geteuid()` is Unix-only. On Windows run as Administrator when using ICMP; the script attempts a sensible fallback if raw sockets are not permitted.

## Programmatic usage

You can import and run the tracer from Python (useful for embedding into collector tools):

```python
from pytraceroute import Tracer

t = Tracer('example.com', max_hops=20, probes=3, timeout=2.0, use_icmp=False)  # use_icmp=False forces UDP
t.run()
```

This lets you capture output or adapt the `Tracer` class to return structured data for logging or dashboards.

---

# IP Header Packet Overview

An IP (Internet Protocol) header packet is a crucial part of data communication over a network. It contains essential information required for routing and delivering packets from the source to the destination.

## Key Components of an IP Header

1. **Version (4 bits):**  
   Indicates the IP version being used. IPv4 has a value of `4`, while IPv6 has a value of `6`.

2. **Header Length (4 bits):**  
   Specifies the length of the IP header in 32-bit words. This field helps determine where the actual data begins in the packet.

3. **Type of Service (ToS) / Differentiated Services (8 bits):**  
   This field is used for quality of service (QoS) settings, allowing for prioritization of packets based on their importance.

4. **Total Length (16 bits):**  
   Defines the total length of the IP packet, including both the header and the data, measured in bytes.

5. **Identification (16 bits):**  
   A unique identifier for each packet, used to help in reassembling fragmented packets.

6. **Flags (3 bits):**  
   Controls fragmentation. This includes flags like "Don't Fragment" (DF) and "More Fragments" (MF).

7. **Fragment Offset (13 bits):**  
   Indicates the position of a fragment in the original packet. This is important when a packet is divided into smaller pieces for transmission.

8. **Time to Live (TTL) (8 bits):**  
   Specifies the maximum number of hops a packet can take before being discarded. This prevents packets from circulating indefinitely.

9. **Protocol (8 bits):**  
   Indicates the protocol used in the data portion of the IP packet, such as TCP (`6`), UDP (`17`), or ICMP (`1`).

10. **Header Checksum (16 bits):**  
    A checksum used for error-checking the header to ensure its integrity.

11. **Source IP Address (32 bits):**  
    The IP address of the sender.

12. **Destination IP Address (32 bits):**  
    The IP address of the recipient.

13. **Options (variable length):**  
    An optional field used for network testing, debugging, and security purposes.

14. **Padding (variable length):**  
    Extra bytes added to ensure the header's length is a multiple of 32 bits.

## Importance of the IP Header

The IP header plays a critical role in ensuring that data is delivered correctly across a network. It provides all the necessary information to route packets, handle errors, and manage fragmentation, making it a fundamental element of IP-based communication.

---

# ICMP Packet Overview

The Internet Control Message Protocol (ICMP) is a critical component of the Internet Protocol suite, primarily used for diagnostic and error-reporting purposes in networking. ICMP packets are typically used to relay information about network issues, connectivity, and path conditions.

## Key Components of an ICMP Packet

1. **Type (8 bits):**  
   Specifies the type of ICMP message. Some common types include:
   - `0`: Echo Reply (used in ping responses)
   - `3`: Destination Unreachable
   - `8`: Echo Request (used in ping requests)
   - `11`: Time Exceeded (used in traceroute)

2. **Code (8 bits):**  
   Provides additional context for the ICMP type. For example:
   - For `Type 3` (Destination Unreachable), the code might indicate whether the destination network or host is unreachable.

3. **Checksum (16 bits):**  
   A checksum used to verify the integrity of the ICMP header and data. It's computed by taking the one's complement of the one's complement sum of the ICMP message.

4. **Identifier (16 bits, optional):**  
   Used to match requests with replies. This field is often used in Echo Request and Echo Reply messages (like in the `ping` command) to associate each reply with the correct request.

5. **Sequence Number (16 bits, optional):**  
   A sequence number that helps to match responses to requests and detect lost packets. It’s typically incremented with each Echo Request.

6. **Data (variable length):**  
   The data section is optional and can contain additional information, such as timestamps, additional error information, or padding data for testing purposes.

## Common ICMP Message Types

### 1. **Echo Request and Echo Reply**
   - **Type 8 (Echo Request):** Sent by a source to check connectivity with a destination (used in `ping`).
   - **Type 0 (Echo Reply):** Sent in response to an Echo Request, indicating that the destination is reachable.

### 2. **Destination Unreachable**
   - **Type 3:** Indicates that a packet could not be delivered to its destination. The `Code` field provides additional details, such as whether the failure was due to the network, the host, or a specific protocol.

### 3. **Time Exceeded**
   - **Type 11:** Used in `traceroute` operations, this message is sent when a packet's Time to Live (TTL) reaches zero before reaching its destination, indicating the path's progress.

## Importance of ICMP Packets

ICMP is integral to network diagnostics and error reporting. Tools like `ping` and `traceroute` rely heavily on ICMP to provide insights into network performance, detect issues, and troubleshoot connectivity problems. ICMP messages are not used for data transport like TCP or UDP, but they provide valuable control and error-reporting functions that ensure robust network communication.

## Security Considerations

While ICMP is useful for network management, it can also be exploited in network attacks, such as ICMP flood attacks (a type of Denial of Service attack). As a result, some networks restrict or filter ICMP traffic to protect against such threats.

---

# CHANGE LOG

Enhanced traceroute (ICMP/UDP) - improved logic and safety

Key improvements over a simple raw-socket traceroute:
- clear separation of ICMP probe building and response parsing
- configurable probes per hop, timeout, and max hops
- graceful fallback: if not running as root, uses UDP-based probes (no raw socket required)
- batch probing per hop using select() to wait for responses to multiple probes
- improved argument parsing and helpful CLI output
Note: Running ICMP mode requires root/administrator privileges on most OSes.

# License
[![GitHub License](https://img.shields.io/github/license/mach1el/pytraceroute?style=for-the-badge&color=orange)](https://github.com/mach1el/pytraceroute/blob/master/LICENSE)