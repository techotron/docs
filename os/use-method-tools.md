Source: http://www.brendangregg.com/Articles/Netflix_Linux_Perf_Analysis_60s.pdf

USE Method (Utilisation, Saturation, Errors)

**Note:** Some tools require sysstat package installed:

`rpm -qa | grep sysstat`

### Uptime

`uptime` will provide the load (sum) averages in 1, 5 and 15 minute constants. The numbers include processes wanting to run on a CPU as well as processes blocked in uninterruptible I/O (usually disk I/O). 
They show the running thread (task) demand on the system plus waiting threads. You can also use `cat /proc/loadavg` to see these stats.

If the values are higher than your CPU count, then you might have a performance problem. To check CPU info: `cat /proc/cpuinfo`

### dmesg | tail

`dmesg -e | tail` will display the last 10 lines from the message buffer of the kernel (if there are any). It typically contains the messages produced by the device drivers. Look for errors that can cause performance issues.

`-e` will display the times in realtime and is a bit more human readable.

### vmstat 1

`vmstat 1` - short for virtual memory stat will print a summary of key server statistics on each line for (1) second, eg:

```bash
procs -----------memory---------- ---swap-- -----io---- --system-- -----cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 0  0      0 122008 148020 941064    0    0     3    54   54  121  4  1 95  0  0
 0  0      0 121992 148020 941088    0    0     0     0  391  975  0  0 100  0  0
 0  0      0 121992 148020 941096    0    0     0    60  391  957  0  0 100  0  0
 0  0      0 121992 148020 941096    0    0     0    56  381  936  0  0 100  0  0
 0  0      0 121992 148020 941096    0    0     0     0  360  924  0  0 100  0  0
```

Columns to check:
- **r**: Number of processes running on CPU and waiting for a turn. A value greater than CPU count is saturation.
- **free**: Free memory in kilobytes.
- **si**, **so**: Swap-ins and swap-outs. If this are not zero (ie, swap space is being used), you're out of memory.
- **us**, **sy**, **id**, **wa**, **st**: User time (application), system time (kernel), idle, wait I/O and stolen time. The system is busy if `us` + `sy` is high. A constant degree of `wa` points towards a disk bottleneck (when `id` is high because tasks are waiting for disk I/O). System time is necessary for I/O processing, an average of 20% might be worth exploring further.

### mpstat -P ALL 1

`mpstat -P ALL 1` - prints the CPU time breakdown per CPU. Use this to check for an imbalance. A single hot CPU can be evidence of a single-threaded application. Example output: 

```bash
14:36:16     CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest   %idle
14:36:17     all   14.29    0.00    3.06    0.00    0.00    0.00    0.00    0.00   82.65
14:36:17       0   14.29    0.00    3.06    0.00    0.00    0.00    0.00    0.00   82.65
```

### pidstat 1

`pidstat 1` - prints a rolling summary per process. The CPU column is the percentage per CPU, so a value of 380.00 would indicate the process was consuming almost 4 CPUs.

Example output:

```bash
14:38:01          PID    %usr %system  %guest    %CPU   CPU  Command
14:38:02            7    0.00    0.91    0.00    0.91     0  rcu_sched
14:38:02         2599    0.00    0.91    0.00    0.91     0  god
14:38:02         4466    0.91    0.00    0.00    0.91     0  aws

14:38:02          PID    %usr %system  %guest    %CPU   CPU  Command
14:38:03         4466    1.03    0.00    0.00    1.03     0  aws
14:38:03        30026    3.09    0.00    0.00    3.09     0  java
```

### iostat -xz 1

`iostat -xz 1` - workload and performance for block devices. Columns to watch:

Example output: 

```bash
avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           3.98    0.23    0.59    0.17    0.30   94.72

Device:         rrqm/s   wrqm/s     r/s     w/s   rsec/s   wsec/s avgrq-sz avgqu-sz   await  svctm  %util
xvda              0.00     4.42    0.19    3.72     5.27   107.74    28.93     0.01    3.62   0.52   0.20
```

- **r/s**, **w/s**, **rkB/s**, **wkB/s**: The delivered reads, writes, read Kbytes and write Kbytes per second. This can be used for workload characterisation. 
- **await**: The average time for the I/O in milliseconds. This is the time that the application suffers as it includes time queued and time being serviced. Larger than average times can be an evidence of saturation.
- **avgqu-sz**: The average number of requests issued to the device. Greater than 1 typically indicates saturation. 
- **%util**: Device utilisation. The time each second that the device was doing work. Values greater than 60% typically lead to poor performance.

**Note:** If the device is a logical disk fronting many back-end disks then 100% utilisation may mean that some I/O is being processed 100% of the time so it doing necessarily indicate saturation.

High I/O isn't always an application issue. It depends if it's written to not be blocked by I/O latencies (eg by using read-ahead for reads and buffering for writes).

### free -m

`free -m`

Example output:

```bash
             total       used       free     shared    buffers     cached
Mem:          2003       1652        350          0         95        763
-/+ buffers/cache:        793       1210
Swap:            0          0          0
```

- **buffers**: Buffer cache, used for block device I/O (in Mbytes)
- **cached**: Page cache, used by file systems (in Mbytes)

It can lead to higher disk I/O if these are near zero. 

- **-/+ buffers/cache**: Linux uses free memory for the disk caches but can reclaim it quickly if applications need it. This line is the result of the used memory, minus the sum of memory allocated to buffers and cached.

### sar -n DEV 1

`sar -n DEV 1` - Check network interface throughput (rxkB/s and txkB/s) as a measure of workload. Get the interface speed with `ethtool INTNAME`. With Amazon EC2 instances, check [here](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/general-purpose-instances.html#general-purpose-network-performance) for the performance.

Example output:

```bash
16:53:51        IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   rxcmp/s   txcmp/s  rxmcst/s
16:53:52           lo      3.06      3.06      0.19      0.19      0.00      0.00      0.00
16:53:52         eth0      2.04      2.04      0.08      0.69      0.00      0.00      0.00
```

In this example, the recieve and transmit values are pretty unburdening.

### sar -n TCP,ETCP 1

`sar -n TCP,ETCP 1` is a summarised view of key TCP metrics.

Example output:

```bash
11:41:54     active/s passive/s    iseg/s    oseg/s
11:41:55         0.00      0.00     11.34     12.37

11:41:54     atmptf/s  estres/s retrans/s isegerr/s   orsts/s
11:41:55         0.00      0.00      0.00      0.00      0.00
```

- **active/s**: Number of locally-initiated TCP connections per second (via connect())
- **passive/s**: Number of remotely-initiated TCP connections per second (via accept())
- **retrans/s**: Number of TCP retransmits per second

The active and passive counts are a rough indication of server load. Passive ~new accepted connections. Active ~Downstream connections. Helpful to think of passive as inbound connections and active as outbound (but not always true).

Retransmits are a sign of network or server issue. Could be unrelieable network or overloaded server dropping packets.

### sar -n SOCK
List stats relating to sockets on the host

```bash
08:55:13 AM       LINUX RESTART

09:00:01 AM    totsck    tcpsck    udpsck    rawsck   ip-frag    tcp-tw
09:10:01 AM       261         9         4         0         0         0
09:20:01 AM       262         9         4         0         0         1
09:30:01 AM       259         7         4         0         0         0
09:40:01 AM       260         7         4         0         0         2
09:50:01 AM       260         7         4         0         0         2
10:00:01 AM       269        14         5         0         0     16141
10:10:01 AM       271        14         5         0         0     16156
10:20:01 AM       271        13         4         0         0     15585
10:30:01 AM       262         9         4         0         0     15842
10:40:01 AM       274        14         4         0         0     16108
10:50:01 AM       265         8         4         0         0      9065
11:00:01 AM       265         8         4         0         0         0
11:10:01 AM       265         8         4         0         0         0
11:20:01 AM       270        10         4         0         0         0
11:30:01 AM       270        10         4         0         0         0
11:40:01 AM       299        26         4         0         0     15646
11:50:01 AM       295        22         4         0         0     14960
12:00:01 PM       281        13         5         0         0     15800
12:10:01 PM       284        15         5         0         0     16384
12:20:01 PM       285        16         5         0         0     15817
12:30:01 PM       267         8         4         0         0         0
12:40:01 PM       272        10         4         0         0         0
Average:          271        12         4         0         0      7614
```

### ss -n -o state time-wait
`ss -n -o state time-wait` will list the number of connections in a TIME_WAIT state. This can be useful if there is a build up of connections in a TIME_WAIT state. Use with `| wc -l` to get a count of the connections (although this information can already be seen with `sar -n SOCK`)

The following `ss` command will list the unique quadruplets (sport, saddress, dport, daddress). This can be helpful to confirm if the number of TIME_WAIT connections marries up with the limitations of the TCP configuration

`ss -o state time-wait -tan 'dport = :80' | awk '{print $(NF-1)" "$(NF-2)}' | sed 's/:[^ ]*//g' | sort | uniq -c`

### top

`top` - useful as an overview of the previously mentioned command. If values look different from the previous commands - then might be an indication that load is variable. Ctrl-S to pause and Ctrl-Q to continue.
