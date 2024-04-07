+++
title = "monitor network connections on linux"
summary = "How to monitor network connections on linux server using ss command"
date = 2019-02-05T17:54:00Z   
tags = ["linux", "network", "monitor"]
canonicalURL = "https://blog.skbali.com/2019/02/ss-command-to-monitor-network-connections-on-linux/"
ShowReadingTime = true
draft = false
+++

`netstat` was one of the first commands I used on a Linux server to monitor network connections between servers. However, on a busy server `netstat` can be a bit slow to capture and display all the information. There is another newer command 'ss' that is much faster than netstat and uses similar command line options to show network information.

`ss` is used to dump socket statistics and as per its man page, it can display more TCP and state information than other tools. With the help of this command you can easily debug network issues and get a good idea of how connections are being handled and what ports are being used.

In this post, I am going to review some of the most common options used with ss to monitor network connections.

## Command Options`


### 1 No options
When no option is passed, `ss` will display a list of all open non-listening sockets (e.g. TCP/UNIX/UDP) that have established connections.

### 2 tcp only
To show tcp connections only, pass the `-t` option.

```bash
ss -t
State              Recv-Q         Send-Q                             Local Address:Port                               Peer Address:Port          Process
ESTAB              0              0                                 192.168.50.100:54520                            192.168.50.100:9100
CLOSE-WAIT         32             0                                 192.168.50.100:38900                            128.95.160.156:https
ESTAB              0              0                                 192.168.50.100:ssh                               192.168.50.47:58798
ESTAB              0              0                                      127.0.0.1:56994                                 127.0.0.1:9090
ESTAB              0              0                                 192.168.50.100:41338                            192.168.50.100:3000
ESTAB              0              0                                 192.168.50.100:55634                             44.214.109.97:9100
ESTAB              0              0                        [::ffff:192.168.50.100]:9100                    [::ffff:192.168.50.100]:54520
ESTAB              0              0                             [::ffff:127.0.0.1]:9090                         [::ffff:127.0.0.1]:56994
ESTAB              0              0                        [::ffff:192.168.50.100]:3000                    [::ffff:192.168.50.100]:41338
```
### 3 Do not resolve service name
Passing the `-n` option avoids name resolution and makes the command complete faster. Notice the ports are listed as numeric instead of `http` or `https`.

```bash
ss -tn
State              Recv-Q         Send-Q                             Local Address:Port                               Peer Address:Port          Process
ESTAB              0              0                                 192.168.50.100:54520                            192.168.50.100:9100
CLOSE-WAIT         32             0                                 192.168.50.100:38900                            128.95.160.156:443
ESTAB              0              0                                 192.168.50.100:22                                192.168.50.47:58798
ESTAB              0              0                                      127.0.0.1:56994                                 127.0.0.1:9090
ESTAB              0              0                                 192.168.50.100:41338                            192.168.50.100:3000
ESTAB              0              0                                 192.168.50.100:55634                             44.214.109.97:9100
ESTAB              0              0                        [::ffff:192.168.50.100]:9100                    [::ffff:192.168.50.100]:54520
ESTAB              0              0                             [::ffff:127.0.0.1]:9090                         [::ffff:127.0.0.1]:56994
ESTAB              0              0                        [::ffff:192.168.50.100]:3000                    [::ffff:192.168.50.100]:41338
```

### 4 Listening sockets 
Passing the `-l` option displays only the listening sockets. This is useful for checking if your service that listens on a specific port is running.
```bash
s -lt
State             Recv-Q             Send-Q                         Local Address:Port                           Peer Address:Port            Process
LISTEN            0                  4096                           127.0.0.53%lo:domain                              0.0.0.0:*
LISTEN            0                  999                                127.0.0.1:31416                               0.0.0.0:*
LISTEN            0                  128                                127.0.0.1:ipp                                 0.0.0.0:*
LISTEN            0                  100                                  0.0.0.0:5433                                0.0.0.0:*
LISTEN            0                  128                                  0.0.0.0:ssh                                 0.0.0.0:*
LISTEN            0                  128                                    [::1]:ipp                                    [::]:*
LISTEN            0                  4096                                       *:9090                                      *:*
LISTEN            0                  4096                                       *:9100                                      *:*
LISTEN            0                  100                                     [::]:5433                                   [::]:*
LISTEN            0                  128                                     [::]:ssh                                    [::]:*
LISTEN            0                  4096                                       *:3000                                      *:*
```

Another interesting variation of this is to see both listening and non-listening sockets.
```bash
ss -tan
State              Recv-Q         Send-Q                             Local Address:Port                               Peer Address:Port          Process
LISTEN             0              4096                               127.0.0.53%lo:53                                      0.0.0.0:*
LISTEN             0              999                                    127.0.0.1:31416                                   0.0.0.0:*
LISTEN             0              128                                    127.0.0.1:631                                     0.0.0.0:*
LISTEN             0              100                                      0.0.0.0:5433                                    0.0.0.0:*
LISTEN             0              128                                      0.0.0.0:22                                      0.0.0.0:*
ESTAB              0              0                                 192.168.50.100:54520                            192.168.50.100:9100
CLOSE-WAIT         32             0                                 192.168.50.100:38900                            128.95.160.156:443
ESTAB              0              0                                 192.168.50.100:22                                192.168.50.47:58798
ESTAB              0              0                                      127.0.0.1:56994                                 127.0.0.1:9090
ESTAB              0              0                                 192.168.50.100:41338                            192.168.50.100:3000
ESTAB              0              0                                 192.168.50.100:55634                             44.214.109.97:9100
LISTEN             0              128                                        [::1]:631                                        [::]:*
LISTEN             0              4096                                           *:9090                                          *:*
LISTEN             0              4096                                           *:9100                                          *:*
LISTEN             0              100                                         [::]:5433                                       [::]:*
LISTEN             0              128                                         [::]:22                                         [::]:*
LISTEN             0              4096                                           *:3000                                          *:*
ESTAB              0              0                        [::ffff:192.168.50.100]:9100                    [::ffff:192.168.50.100]:54520
ESTAB              0              0                             [::ffff:127.0.0.1]:9090                         [::ffff:127.0.0.1]:56994
ESTAB              0              0                        [::ffff:192.168.50.100]:3000                    [::ffff:192.168.50.100]:41338
```

### 5 Process name and process id
To show the process name and pid pass the `-p` option
```bash
ss -pt
State           Recv-Q      Send-Q                      Local Address:Port                         Peer Address:Port       Process
ESTAB           0           0                          192.168.50.100:54520                      192.168.50.100:9100
CLOSE-WAIT      32          0                          192.168.50.100:38900                      128.95.160.156:https
ESTAB           0           0                          192.168.50.100:ssh                         192.168.50.47:58798
ESTAB           0           0                          192.168.50.100:46364                       91.189.91.124:https       users:(("wget",pid=99845,fd=4))
ESTAB           0           0                               127.0.0.1:56994                           127.0.0.1:9090
ESTAB           0           0                          192.168.50.100:41338                      192.168.50.100:3000
ESTAB           0           0                          192.168.50.100:55634                       44.214.109.97:9100
ESTAB           0           0                 [::ffff:192.168.50.100]:9100              [::ffff:192.168.50.100]:54520
ESTAB           0           0                      [::ffff:127.0.0.1]:9090                   [::ffff:127.0.0.1]:56994
ESTAB           0           0                 [::ffff:192.168.50.100]:3000              [::ffff:192.168.50.100]:41338
```

### 6 Filter by connection state
ss allows you to filter output by connection state. Below is a way to filter and display only the ESTABLISHED connections.

```bash
ss -tn state established
Recv-Q            Send-Q                                 Local Address:Port                                     Peer Address:Port             Process
0                 0                                     192.168.50.100:54520                                  192.168.50.100:9100
0                 0                                     192.168.50.100:38870                                 185.199.111.133:443
0                 0                                     192.168.50.100:43986                                  34.120.177.193:443
0                 0                                     192.168.50.100:39460                                  128.95.160.157:443
0                 0                                     192.168.50.100:22                                      192.168.50.47:58798
0                 0                                          127.0.0.1:56994                                       127.0.0.1:9090
0                 0                                     192.168.50.100:41338                                  192.168.50.100:3000
0                 0                                     192.168.50.100:55634                                   44.214.109.97:9100
0                 0                            [::ffff:192.168.50.100]:9100                          [::ffff:192.168.50.100]:54520
0                 0                                 [::ffff:127.0.0.1]:9090                               [::ffff:127.0.0.1]:56994
0                 0                            [::ffff:192.168.50.100]:3000                          [::ffff:192.168.50.100]:41338
```

Basic filter usage as shown in the help is like this:
```bash
       FILTER := [ state STATE-FILTER ] [ EXPRESSION ]
       STATE-FILTER := {all|connected|synchronized|bucket|big|TCP-STATES}
         TCP-STATES := {established|syn-sent|syn-recv|fin-wait-{1,2}|time-wait|closed|close-wait|last-ack|listening|closing}
          connected := {established|syn-sent|syn-recv|fin-wait-{1,2}|time-wait|close-wait|last-ack|closing}
       synchronized := {established|syn-recv|fin-wait-{1,2}|time-wait|close-wait|last-ack|closing}
             bucket := {syn-recv|time-wait}
                big := {established|syn-sent|fin-wait-{1,2}|closed|close-wait|last-ack|listening|closing}
```

some valida filter examples are:
```bash
ss -tn state connected
State              Recv-Q         Send-Q                             Local Address:Port                               Peer Address:Port          Process
ESTAB              0              0                                 192.168.50.100:54520                            192.168.50.100:9100
CLOSE-WAIT         32             0                                 192.168.50.100:45676                            128.95.160.156:443
LAST-ACK           0              32                                192.168.50.100:38900                            128.95.160.156:443
ESTAB              0              0                                 192.168.50.100:22                                192.168.50.47:58798
ESTAB              0              0                                      127.0.0.1:56994                                 127.0.0.1:9090
ESTAB              0              0                                 192.168.50.100:41338                            192.168.50.100:3000
ESTAB              0              0                                 192.168.50.100:55634                             44.214.109.97:9100
ESTAB              0              0                        [::ffff:192.168.50.100]:9100                    [::ffff:192.168.50.100]:54520
ESTAB              0              0                             [::ffff:127.0.0.1]:9090                         [::ffff:127.0.0.1]:56994
ESTAB              0              0                        [::ffff:192.168.50.100]:3000                    [::ffff:192.168.50.100]:41338
```

### 7 Show connections from a specific ip address
`ss` allows you to filter and display connections by ip addresses. Let's say you wanted to see how many connections you had to an address, you could run a command like this
```bash
ss -tn dst 192.168.50.100
State          Recv-Q          Send-Q                             Local Address:Port                                Peer Address:Port           Process
ESTAB          0               0                                 192.168.50.100:54520                             192.168.50.100:9100
ESTAB          0               0                                 192.168.50.100:41338                             192.168.50.100:3000
ESTAB          0               0                        [::ffff:192.168.50.100]:9100                     [::ffff:192.168.50.100]:54520
ESTAB          0               0                        [::ffff:192.168.50.100]:3000                     [::ffff:192.168.50.100]:41338
```

You can also use CIDR notation like `ss -tn dst 192.168.86.0/24`

### 8 Show connections for a specific port
Let us say you wanted to see all outbound connections to port `9100`
```bash
ss -tn dport = :9100
State            Recv-Q            Send-Q                          Local Address:Port                            Peer Address:Port            Process
ESTAB            0                 0                              192.168.50.100:54520                         192.168.50.100:9100
ESTAB            0                 0                              192.168.50.100:55634                          44.214.109.97:9100
```

`ss` also allows you to query and display by local port. Say you want to list all connections higher than port 50000. 
I used the `-H` option to suppress header and sorted the output to show the connections higher than 50000.
```bash
s -Htn sport gt :50000
ESTAB               0                    0                                  192.168.50.100:54520                               192.168.50.100:9100
ESTAB               0                    0                                       127.0.0.1:56994                                    127.0.0.1:9090
ESTAB               0                    0                                  192.168.50.100:55634                                44.214.109.97:9100
```

### 9 Combine filters

You can also combine filters to perform more advanced queries. Here I combined local port and destination address
```bash
s -Htn sport gt :10000 dst 192.168.50.0/24
ESTAB               0                    0                                  192.168.50.100:54520                               192.168.50.100:9100
ESTAB               0                    0                                  192.168.50.100:41338                               192.168.50.100:3000
```

### 10 Monitor the output
Using the Linux watch command you can setup a monitor to see continuous output on your terminal. Below I refresh the output every four seconds and display all HTTPS connection information
```bash
watch -n 4 ss -tn dst :443
```

## Conclusion
In the above examples I showed some of the most common commands that I normally use when I need to inspect network connection and states on a Linux server. 
The ability to add multiple filters and output the information fast makes it a very useful tool to debug and understand network connections and states.

## Further reading
[ss man page](https://man7.org/linux/man-pages/man8/ss.8.html)