# README

Some experiments looking into the behavior of AWS Network ACLs and Security Groups.

I was interested if they sent any signal that packets were rejected (e.g. ICMP or TCP RST) or if they just silently dropped packets.

See [this article](http://www.chiark.greenend.org.uk/~peterb/network/drop-vs-reject) for a nice discussion of DROP vs REJECT, and [this article](https://serverfault.com/questions/157375/reject-vs-drop-when-using-iptables) for a lot of confusing opinions. :)


## Experimental design

I set up 2 AWS hosts in 2 different subnets of the same VPC. I used `t2.nano`s running Ubuntu 18.04. I ran in region `us-east-1` using AZs `us-east-1a` and `us-east-1b`. The subnets used the same network ACL association and security group for the test.

Between each step, I rolled back any changes I made to the network (iptables, security groups, ACLs) to block traffic to make sure requests succeeded again before going on to the next experient.

Each experiment looked like the following. I happened to be running in 4 tmux panes.

```bash
# pane 1, host 1 (172.31.26.130)
# Sends an http request to host 2
curl -s 172.31.43.42:8000 >/dev/null; echo $?

# pane 2, host 1 (172.31.26.130)
# Capture tcp dump of all traffic to the host and dump into a file
tcpdump host 172.31.43.42 -w my-capture-file.txt

# pane 3, host 2 (172.31.43.42)
# Run simple http server on port 8000
python3 -m http.server 8000

# pane 4, host 2 (172.31.43.42)

# Misc management activities with iptables, for example:
# List
iptables -L --line-numbers
# Delete numbered rule
# Rules were cleared after each experiment
iptables -D INPUT 1
# Add rule
iptables -A INPUT -p tcp --dport 8000 -j REJECT
```

### TCP dumps for various network configurations

## Successful request

This is a normal happy HTTP request. All other requests are disrupted in some ways.

```
15:51:02.749070 IP 172.31.26.130.57082 > 172.31.43.42.irdmi: Flags [S], seq 2123019698, win 62727, options [mss 8961,sackOK,TS val 3227103255 ecr 0,nop,wscale 6], length 0
15:51:02.750982 IP 172.31.43.42.irdmi > 172.31.26.130.57082: Flags [S.], seq 1270824101, ack 2123019699, win 62643, options [mss 8961,sackOK,TS val 1878577998 ecr 3227103255,nop,wscale 6], length 0
15:51:02.751000 IP 172.31.26.130.57082 > 172.31.43.42.irdmi: Flags [.], ack 1, win 981, options [nop,nop,TS val 3227103257 ecr 1878577998], length 0
15:51:02.751052 IP 172.31.26.130.57082 > 172.31.43.42.irdmi: Flags [P.], seq 1:82, ack 1, win 981, options [nop,nop,TS val 3227103257 ecr 1878577998], length 81
15:51:02.752622 IP 172.31.43.42.irdmi > 172.31.26.130.57082: Flags [.], ack 82, win 978, options [nop,nop,TS val 1878578000 ecr 3227103257], length 0
15:51:02.753397 IP 172.31.43.42.irdmi > 172.31.26.130.57082: Flags [P.], seq 1:155, ack 82, win 978, options [nop,nop,TS val 1878578001 ecr 3227103257], length 154
15:51:02.753404 IP 172.31.26.130.57082 > 172.31.43.42.irdmi: Flags [.], ack 155, win 979, options [nop,nop,TS val 3227103259 ecr 1878578001], length 0
15:51:02.753558 IP 172.31.43.42.irdmi > 172.31.26.130.57082: Flags [FP.], seq 155:861, ack 82, win 978, options [nop,nop,TS val 1878578001 ecr 3227103257], length 706
15:51:02.753575 IP 172.31.26.130.57082 > 172.31.43.42.irdmi: Flags [.], ack 862, win 968, options [nop,nop,TS val 3227103259 ecr 1878578001], length 0
15:51:02.753602 IP 172.31.26.130.57082 > 172.31.43.42.irdmi: Flags [F.], seq 82, ack 862, win 968, options [nop,nop,TS val 3227103259 ecr 1878578001], length 0
15:51:02.754979 IP 172.31.43.42.irdmi > 172.31.26.130.57082: Flags [.], ack 83, win 978, options [nop,nop,TS val 1878578003 ecr 3227103259], length 0
```

## iptables REJECT rule

Added a rule to reject traffic with: `iptables -A INPUT -p tcp --dport 8000 -j REJECT`.
The connection closes quickly because of the ICMP packet sent back by iptables on host 2.

```
15:58:53.211907 IP 172.31.26.130.57094 > 172.31.43.42.irdmi: Flags [S], seq 2407464667, win 62727, options [mss 8961,sackOK,TS val 3227573718 ecr 0,nop,wscale 6], length 0
15:58:53.213473 IP 172.31.43.42 > 172.31.26.130: ICMP 172.31.43.42 tcp port irdmi unreachable, length 68
```

## iptables DROP rule

Added a rule to reject traffic with: `iptables -A INPUT -p tcp --dport 8000 -j DROP`.
You can see tcp sending 3 sequential packets waiting for an ack (with time stretching out due to TCP backoff) until curl times out.

```
15:55:05.891140 IP 172.31.26.130.57088 > 172.31.43.42.irdmi: Flags [S], seq 500455406, win 62727, options [mss 8961,sackOK,TS val 3227346397 ecr 0,nop,wscale 6], length 0
15:55:06.914758 IP 172.31.26.130.57088 > 172.31.43.42.irdmi: Flags [S], seq 500455406, win 62727, options [mss 8961,sackOK,TS val 3227347421 ecr 0,nop,wscale 6], length 0
15:55:08.930755 IP 172.31.26.130.57088 > 172.31.43.42.irdmi: Flags [S], seq 500455406, win 62727, options [mss 8961,sackOK,TS val 3227349437 ecr 0,nop,wscale 6], length 0
```

## iptables reject-with tcp-reset

Added a rule to reject traffic with: `iptables -A INPUT -p tcp --dport 8000 -j REJECT --reject-with tcp-reset`.
The connection closes quickly because of the ACK/RST packet sent back by iptables on host 2.

```
16:00:06.460948 IP 172.31.26.130.57096 > 172.31.43.42.irdmi: Flags [S], seq 1896170312, win 62727, options [mss 8961,sackOK,TS val 3227646967 ecr 0,nop,wscale 6], length 0
16:00:06.462325 IP 172.31.43.42.irdmi > 172.31.26.130.57096: Flags [R.], seq 0, ack 1896170313, win 0, length 0
```

## Security group deny

We removed the security group setting allowing all TCP traffic to other nodes of the same security group.
The only other security group rules were allowing ssh from a set of subnets associated with our office VPN, so nodes in the subnset could no longer communicate with each other.
This looks identical to iptables in `DROP` mode. No response is sent and curl times out.

[security group configuration](security_group_setup.png)

```
16:06:58.070235 IP 172.31.26.130.57102 > 172.31.43.42.irdmi: Flags [S], seq 1307417946, win 62727, options [mss 8961,sackOK,TS val 3228058576 ecr 0,nop,wscale 6], length 0
16:06:59.074759 IP 172.31.26.130.57102 > 172.31.43.42.irdmi: Flags [S], seq 1307417946, win 62727, options [mss 8961,sackOK,TS val 3228059581 ecr 0,nop,wscale 6], length 0
16:07:01.090781 IP 172.31.26.130.57102 > 172.31.43.42.irdmi: Flags [S], seq 1307417946, win 62727, options [mss 8961,sackOK,TS val 3228061597 ecr 0,nop,wscale 6], length 0
```

## Network ACL deny

We added a network ACL rule to drop all incoming traffic on port 8000.
This looks identical to iptables in `DROP` mode. No response is sent and curl times out.

[network acl configuration](network_ack_setup.png)

```
16:12:46.538826 IP 172.31.26.130.57114 > 172.31.43.42.irdmi: Flags [S], seq 3061520091, win 62727, options [mss 8961,sackOK,TS val 3228407045 ecr 0,nop,wscale 6], length 0
16:12:47.554780 IP 172.31.26.130.57114 > 172.31.43.42.irdmi: Flags [S], seq 3061520091, win 62727, options [mss 8961,sackOK,TS val 3228408061 ecr 0,nop,wscale 6], length 0
16:12:49.570760 IP 172.31.26.130.57114 > 172.31.43.42.irdmi: Flags [S], seq 3061520091, win 62727, options [mss 8961,sackOK,TS val 3228410077 ecr 0,nop,wscale 6], length 0
```


