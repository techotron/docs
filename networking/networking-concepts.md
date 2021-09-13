# Networking Concepts

An area for detailing various networking concepts which don't make sense to include in the [tools](./tools.md) readme.

## Packet Flow

A high level description of the flow of a packet:

Example: Client sends request to Server using a hostname

1. DNS query to resolve hostname to IP
2. ARP request to find destination MAC of resolved IP OR Gateway (if outside network). If not in ARP table, broadcast is sent out.
3. TCP 3-way handshake. SYN -> SYN ACK -> ACK (unless using UDP which is connectionless)
4. Connection. Data will be sent back and forth.

## DHCP

1. Client broadcasts request
2. Server responds with an offer of an IP
3. Client officially requests the IP previously offered.
4. DHCP server acknowledges IP has been assigned.

## NIC Teaming and Bonding

### Teaming

- Support for IPv6 link monitoring
- Able to work with D-Bus and Unix Domain Sockets
- Load Balancing for LACP support (combine throughput)
- Leverages NetworkManager and associated tools.
- Doesn't make sense to use in environment where asymmetric routing is involved (eg Cloud environment like AWS).

#### Team Runners

Team Runners are the modes (round robin, active-backup, broadcast (all interfaces), loadbalance (active (decides route based on desisions) and passive (alternate ports, uses less RAM)) and LACP (802.3ad)

#### Link Watchers

Methods to check the ports are ok.

- ethtool: default, watches for link state changes (use this with LACP)
- arp_ping: monitors availability of MAC addresses using ARP
- nsna_ping: neighborhood advertisement and neighborhood soliciation from the IPv6 neighbor discovery are used to monitor neighbor's interface.

### Bonding

- Doesn't require teamd
- Works in virtual environment
