# Network Performance Tuning
Dumping some notes from some network performance tuning:

These were parameters we added to a `docker run` command but can be changed as per standard sysctls

|Parameter|Description|Notes|
|---|---|---|
|--ulimit nofile=65535:65535|Increase max open file descriptors|Default is: 1024:4096|
|--sysctl net.core.somaxconn=65535|Increases max number of connections queued up for a socket.|Default is 128|
|--sysctl net.ipv4.tcp_fin_timeout=1|Change how long to wait for the final fin packet.|Default is 60. Lots of advise suggesting not to change this unless needed to aid in a DoS attack.|
|--sysctl net.ipv4.ip_local_port_range="9000 65535"|Sets the port range the interface can use. Increasing this allows a greater number of ports to be used in connections to the upstream service|Default is: 32768   60999|
|--sysctl net.ipv4.tcp_rmem="4096 87380 8388608"|Min/default/max size of TCP receive buffer|Default is: 4096    87380   6291456|
|--sysctl net.ipv4.tcp_wmem="40960 873800 83886080"|Min/default/max size of TCP send buffer|Default is: 4096    20480   4194304|
|--sysctl net.ipv4.tcp_mem="428430 571270 856860"|TCP will regulate it's memory in various ways depending on these limits (low, pressure, high)|Default is: 42843   57127   85686|
|--sysctl net.ipv4.tcp_max_orphans=655360|Max number of TCP sockets which aren't attached to a user file handle (aka orphans). Used to prevent simple DoS attacks. Increasing this holds the risk of increasing mem usage (~64kb per orphaned connection)|Default is: 16384|
