# Project File Walkthrough

The `cPacketSniffer` project is a network packet sniffer written in C, designed to capture and analyze network traffic using `libpcap`. Below is a breakdown of what each file does.

## Core Application
- **`sniffer.c`**: The main entry point. It parses command-line arguments (device selection, filters, mode), sets up the `libpcap` session, handles signal interrupts (Ctrl+C), and contains the `process_packet` callback loop that dispatches packets to specific handlers.
- **`sniffer.h`**: Header for the main application, defining constants and shared structures.
- **`Makefile`**: The build script. It defines how to compile the C files into the final `sniffer` executable, linking against the `pcap` library.

## Protocol Handlers (The "Onion" Layers)
These files wrap raw data into structured representations of network layers.
- **`datagram.c/h`**: The rawest layer. Wraps the captured byte array and provides methods to promote it to an Ethernet frame.
- **`ethernetframe.c/h`**: Represents the Data Link Layer (Layer 2). Parses Source/Dest MAC addresses and determines if the payload is IP or ARP.
- **`ippacket.c/h`**: Represents the Network Layer (Layer 3). Parses IPv4 headers (Source/Dest IP, Protocol) and determines if the payload is TCP, UDP, or ICMP.
- **`arppacket.c/h`**: Handles ARP (Address Resolution Protocol) packets. Used for resolving IP addresses to MAC addresses.
- **`icmppacket.c/h`**: Handles ICMP (Ping) packets. Used by the Ping Flood Detector.
- **`tcpsegment.c/h`**: Represents the Transport Layer (Layer 4) for TCP. Parses Ports, Flags (SYN, ACK), and Sequence numbers.
- **`udpsegment.c/h`**: Represents the Transport Layer (Layer 4) for UDP. Parses Ports and Length.

## Security & Analysis Modules
These files implement the specific logic for the "Apps" or "Security Tools" supported by the sniffer.
- **`pingflooddetector.c/h`**: Detects potential Ping Floods by analyzing the frequency and size of ICMP Echo Request packets targeting a specific host.
- **`tcpsessiontracker.c/h`**: Tracks TCP sessions. It matches IP/Port pairs to follow the state of a connection (Handshake, Data, Teardown).
- **`tftpsessiontracker.c/h`** & **`tftp.c/h`**: Specifically monitors TFTP (Trivial File Transfer Protocol) sessions, which run over UDP. It parses TFTP opcodes (Read, Write, Data, Ack).
- **`arppacket.c`** (Logic): Also contains logic for ARP Spoofing detection by checking for unsolicited ARP replies (Gratuitous ARP).

## Utilities & Data Structures
- **`generic-dict.c/h`**: A generic Hash Table implementation. Used by session trackers to store state (key-value pairs of ConnectionID -> SessionState).
- **`simple-set.c/h`**: A simple Set data structure. Used for storing unique items, like IP addresses in ARP requests.
- **`ipaddress.c/h`**: Helper functions to format and manipulate IP addresses (e.g., converting `struct in_addr` to string).
- **`macaddress.c/h`**: Helper functions to format and print MAC addresses (e.g., `00:11:22:33:44:55`).
- **`utils.c/h`**: General helper functions.

## Technical Deep Dive: How It Works Internally

### 1. The "Promotion" Pipeline (Object-Oriented C)
The project uses an interesting **Object-Oriented style in C**.
- **Encapsulation**: Structs (like `ethernetframe`) contain function pointers (`e->ether_type(e)`) acting as methods.
- **Inheritance-like Flow**:
    1.  `sniffer.c` captures raw bytes.
    2.  `datagram` wraps these bytes.
    3.  `datagram` creates an `ethernetframe` (Layer 2).
    4.  `ethernetframe` checks its type (e.g., IPv4) and creates an `ippacket` (Layer 3).
    5.  `ippacket` checks its protocol (e.g., TCP) and creates a `tcpsegment` (Layer 4).
This allows the code to "peel" the packet layers cleanly.

### 2. Data Structures
Custom data structures are implemented from scratch to manage state.

#### **Generic Dictionary (`generic-dict.c`)**
- **Type**: Hash Table.
- **Collision Resolution**: **Open Addressing with Quadratic Probing** (`currentPos += 2 * ++collisionNum - 1`).
- **Resizing**: Automatically rehashes when load factor > 0.5.
- **Usage**: Used by `tcpsessiontracker` to map a specific connection ID (String: `IP:Port + IP:Port`) to its current `SessionState`.

#### **Simple Set (`simple-set.c`)**
- **Type**: Dynamic Array (Vector-like).
- **Behavior**: Stores unique strings (e.g., IP addresses). Checks for existence before adding (`O(N)` search).
- **Usage**: Used by `arpspoof` detection to track which IPs have sent ARP Requests. If an ARP Reply comes from an IP that *didn't* request it, it might be a spoofing attack.

## Inputs Needed From You
To run this project, you need:
1.  **Linux Environment**: A Linux OS or **WSL (Windows Subsystem for Linux)**. The code handles raw sockets and includes headers like `<arpa/inet.h>` and `<sys/socket.h>` which are native to Linux.
2.  **Dependencies**: The `libpcap-dev` library must be installed.
    - Ubuntu/Debian: `sudo apt-get install libpcap-dev`
3.  **Permissions**: Packet sniffing requires `sudo` (root) privileges to access the network interface.

## Windows Compatibility
- **Direct Run**: **No.** It is likely **not** runnable directly on Windows without significant modification (porting to `Winsock` and `Npcap` SDK).
- **Recommended**: Run it inside **WSL**. This allows you to run the code "on Windows" but inside a Linux compatibility layer.
