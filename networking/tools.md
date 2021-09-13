# General Networking Tools
This is an area for notes about general networking tools and utilities, with quick notes and example commands

## Basic Network Commands
- Standard network interface details: `ip addr show` and `ifconfig` (obsolete)
- Network Manager Cli: `nmcli` - shows DNS details and default route. `nmcli device show` will provide more comprehensive list 
  - List IPv4 IP address and gateway: `nmcli device show | grep 172 | egrep '^IP4.G|^IP4.A'`
- Route information: `ip route show` - this will show the default route
- Detailed route tabl: `routel`
- Show IP neighbours (ARP table): `ip n`
- Force `ping` through specific interface: `ping -I eth1 1.1.1.1`

## DNS
- Dig (part of the `bind-utils` package for Centos)
  - Show the query/answer of a DNS request: `dig -4 www.linuxacademy.com`
  - Show the query/answer plus trace: `dig -4 www.linuxacademy.com +trace`
  
## DHCP
**Note:** commands may vary depending on *nix flavour in use but you can usually tab-complete from `/var/lib/dhc->` to find the right one. 

- DHCP request and offers are layer 2 broadcasts (similar to ARP requests) and need a DHCP client.
- DHCP server uses UDP 67 and client uses UDP 68
- Lookup the DHCP host: `cat /var/log/messages | grep "DHCPOFFER"` (or `cat /var/log/* | grep "DHCPOFFER"` if messages logs are rotating more frequently than DHCP offers).
- View lease information: `cat /var/lib/dhclient/dhclient--eth0.lease`
- View listening client: `sudo ss -luntp | grep dhclient` (switches: "listening", "udp", "numeric (don't resolve)", "tcp", "processes")

### Release IP address
- Release IPL `sudo dhclient -r`
- Restart DHCP Client: `sudo dhclient`

## Setting Static IP
- Check connection number using `nmcli`
- Run `sudo nmcli connection modify System\ ens5 ipv4.method manual ipv4.address <NEW_IP/CIDR> ipv4.gateway <GW_IP>>`. This won't change anything, need to restart the interface
- Bring interface down and up: `sudo nmcli connection down System\ ens5` and `sudo nmcli connection up System\ ens5` (can also use `systemctl restart network`)
- Add DNS server to interface properties: `sudo nmcli connection modify System\ ens5 ipv4.dns <DNS_IP>` (followed by interface restart as above)
- Change it back to DHCP: `sudo nmcli connection modify System\ ens5 ipv4.method auto ipv4.address "" ipv4.dns "" ipv4.gateway ""` (followed by interface restart as above)

### Adding additional IPs
- `sudo nmcli connection modify System\ ens5 ipv4.address <IP_ADD/CIDR>, <IP_ADD/CIDR>`. This will add 2 new IPs. They will rely upon GW/DNS settings from previous settings (whether it be DHCP or manual) in order to work. Assuming these settings are all correct, the new IPs will be pingable.
- Bring interface down and up: `sudo nmcli connection down System\ ens5` and `sudo nmcli connection up System\ ens5`

## Bonding
Assuming you have 2 cards setup already. 

- Create bond: `sudo nmcli connection add type bond con-name bond0 ifname bond0 mode active-backup ip4 <NEW_IP_ADD/CIDR`
- View newly created interface: `nmcli c`.
- Add interfaces to bond: `sudo nmcli connection add type bond-slave ifname ens5 master bond0` and `sudo nmcli connection add type bond-slave -ifname ens6 master bond0`.
- Bring the new bond-slave interfaces up: `sudo nmcli connection up bond-slave-ens5` and `sudo nmcli connection up bond-slave-ens6`
- Check the details of the bond interface: `nmcli devices show bond0` - notice that the gateway is missing.
- Add gateway to bond0 interface: `sudo nm connection modify bond0 ipv4.gateway <GW_IP>`
- Restart the bond interface: `sudo nmcli connection down bond0 && sudo nmcli connection up bond0`

## Teaming
- Need to use `teamd` so install: `yum install -y teamd bash-completion` and reload profile to enable `bash-completion`
- View network connections: `nmcli c`
- If you want to delete the connections to the interfaces using NetworkManager: `sudo nmcli connection delete Name\ of\ connection\ 1`
- Create team connection: `sudo nmcli connection add type team con-name Team0 ifname team0 team.config '{"runner":{"name": "activebackup"}, "link_watch": {"name": "ethtool"}}'`
- **Note:** Check teamd's example configs: `cat /usr/share/doc/teamd-<version>/example_configs/<list_of_configs>`
- Add slave interfaces: `sudo nmcli connection add type team-slave con-name slave1 ifname eth1 master team0` and a second slave interface: `sudo nmcli connection add type team-slave con-name slave2 ifname eth2 master team0`
- Add settings to the team interface: `sudo nmcli connection modify Team0 ipv4.addresses <IP_ADD/CIDR> ipv4.gateway <GW_IP> ipv4.method manual`
- Confirm settings are expected: `nmcli connection show | grep ipv4`
- Bring slaves up, then team: `sudo nmcli con up slave1 && sudo nmcli con up slave2` and then the team interface: `sudo nmcli con up Team0`
- Check state of team interface using teamd: `teamdctl team0 state`
- If we wanted to use round robin runner, create the team interface in the same way but don't specify a team.config. Round robin is the default. Confirm this using the above `teamdctl` command. 

## Routing
- Show routing table: `ip r`. You can also use `nmcli` and check the listed routes there.
- Flush routing table: `ip route flush 123.123.123.123` or `ip r f 123.123.123.123`
- Prohibit traffic to IP: `ip route add prohibit 1.1.1.1`. This will result in a ping failing with: 

> Do you want to ping broadcast? Then -b. If not, check your local firewall rules.

And a curl failing with:

> curl: (7) Failed to connect to 1.1.1.1: Permission denied

- Similar result would be to blackhole the IP: `ip route add blackhole 1.1.1.1`

Ping result:

> connect: Invalid argument

Curl result: 

> curl: (7) Failed to connect to 1.1.1.1: Invalid argument

- Same with `unreachable` - message back is 

> No route to host

- Remove route (such as the example above: `ip route del prohibit 1.1.1.1`
- These route changes will not persist a system restart (they would persist a network restart though). To create persistent routes, you'd need to add the route to `/etc/sysconfig/network-scripts/route-<INTERFACE_NAME>`.
- To create a route out through a specific interface: `ip route add 1.1.1.1 via <GW_IP> dev eth0`
- Create a route out through a specific gateway: `ip route add 1.1.1.1 via <GW_IP>`
- Create route for network: `ip route add 10.0.8.0/24 via <GW_IP>`

## Local Name Resolution
- Name Service Switch file: `/etc/nsswitch.conf`. This is used by `getent`, eg: `getent ahosts latoys.com`
- Check the ordering by the "hosts" field in the .conf file. By default, this is `files dns myhostname`. However, if you used one of the tools in the "bind-utils" package (eg `host latoys.com`), you'd find that it would use DNS and ignore the value of /etc/hosts.
- Use a public DNS server to resolve a host: Google: `dig @8.8.8.8 <HOSTNAME>` Cloud Flare: `dig @1.1.1.1 <HOSTNAME>`
- Get list of Nameservers for a domain: `dig NS <HOSTNAME>`. You can tell if the reponse is authoritative or not by the "AUTHORITY: N" (where "N" is a number. 0 means no authority).
- Using one of the NS in the response above, you can get an authoritative resolution for a host: `dig @<NAMESERVER_FROM_ABOVE> <HOSTNAME>`. Example response:

> flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 3, ADDITIONAL: 6

**Note:** The "qr aa rd" means: "query request", "authoritative answer" and "recursion desired"
**Note:** The rest means, 1 query requested, 1 answer given, 3 authoritative nameservers and an additional 6 non-auth NS.

So for `dig @ns3.memset.com snow-online.co.uk`

The answer was:

```bash
; <<>> DiG 9.10.6 <<>> @ns3.memset.com snow-online.co.uk
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 12940
;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 3, ADDITIONAL: 6
;; WARNING: recursion requested but not available

;; QUESTION SECTION:
;snow-online.co.uk.             IN      A

;; ANSWER SECTION:
snow-online.co.uk.      300     IN      A       92.222.3.151

;; AUTHORITY SECTION:
snow-online.co.uk.      7200    IN      NS      ns3.memset.com.
snow-online.co.uk.      7200    IN      NS      ns2.memset.com.
snow-online.co.uk.      7200    IN      NS      ns1.memset.com.

;; ADDITIONAL SECTION:
ns3.memset.com.         7200    IN      A       31.222.188.99
ns3.memset.com.         1800    IN      A       31.222.188.99
ns2.memset.com.         7200    IN      A       78.31.107.87
ns2.memset.com.         1800    IN      A       78.31.107.87
ns1.memset.com.         7200    IN      A       89.200.136.74
ns1.memset.com.         1800    IN      A       89.200.136.74

;; Query time: 29 msec
;; SERVER: 31.222.188.99#53(31.222.188.99)
;; WHEN: Tue Nov 19 22:49:52 GMT 2019
;; MSG SIZE  rcvd: 211
```

- Check how old the query cache is: `dig <HOSTNAME> +noall +answer`
- When we see `NXDOMAIN` in the "status:" section of a dig response, this means the domain does not exist.

## Firewalls
Todo

## tc (Traffic Control)
The following commands use `tc` and `netem` to simulate network issues on the host

### Generic commands
- Show list of rules currently applied: `sudo tc qdisc show`
- Add rule: `sudo tc qdisc add...`
- Change rule: `sudo tc qdisc change...`
- Delete rule: `sudo tc qdisc del...`

### Add/remove delay onto the "eno1" interface
```bash
sudo tc qdisc add dev eno1 root netem delay 1000ms
sudo tc qdisc del dev eno1 root netem delay 1000ms
```

**Note:** If a rule already exists, then s/add/change

### Add 100ms +/- 10ms network delay variation
```bash
sudo tc qdisc del dev eno1 root netem delay 100ms 10ms
sudo tc qdisc del dev eno1 root netem delay 100ms 10ms 25%
```

The second command adds a correlation which makes the variation a bit more random

### Use a delay distribution table
A network delay is typically less uniform than the above examples so you can use a distribution table to describe the delay.
```bash
sudo tc qdisc change dev eno1 root netem delay 100ms 1000ms distribution normal
```

The tables you can use reside in /usr/lib/tc
They are compiled by iproute2. It's possible to create your own based on experimental data and use that.

### Packet loss
```bash
sudo tc qdisc add dev eno1 root netem loss 0.5%
sudo tc qdisc add dev eno1 root netem loss 0.5% 25%
```

Same as before, the second command here adds a correlation to make it more random

## LSOF
List open ports on the host 
```bash
lsof -n -a -i4 -a -P
```

- `-n`: Don't resolve IPs to hostnames (faster)
- `-a`: AND operator
- `-i4`: Match IPv4 addresses
- `-P`: Don't resolve port numbers to network files (faster)
