---
layout: post
title: "Iptables basic configuration and saving tutorial"
date: 2015-01-04 22:15
comments: true
categories:
  - ubuntu
  - iptables
  - configuration
  - basics
  - firewall
  - cheatsheet
  - tutorial
---
To list existing rules type the following command(as root):

```
% iptables -L
```  

If it wasn't previously configured, output should be:

```
Chain INPUT (policy ACCEPT)
target     prot opt source               destination

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
```

Which, in short words, means "accept everything" or that firewall isn't active.<br/>
Let's configure firewall to accept only the traffic we need and then save it.

<!-- more -->

Basic commands:<br/>
`-I` - `Insert` rule(it would be first)<br/>
`-A` - `Append` rule(it would be last)<br/>
`-D` - `Delete` rule<br/>
`-p` - Connection `protocol`(`tcp`/`udp`/`icmp`/`all`)<br/>
`--dport` - Destination `port`<br/>
`-j` - Action(`ACCEPT`/`REJECT`/`DROP`/`LOG`)
`-P` - Policy
`-m` - Allow filter rules to match based on connection state

### Accessible ports configuration

First of all, you want to allow `ssh` access, which is `tcp` port `22`
```
% iptables -A INPUT -p tcp --dport 22 -j ACCEPT
% iptables -L
Chain INPUT (policy ACCEPT)
target     prot opt source               destination
ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:ssh

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
```

To accept `http` traffic, you need to open `tcp` port `80`
```
% iptables -A INPUT -p tcp --dport 80 -j ACCEPT
% iptables -L
Chain INPUT (policy ACCEPT)
target     prot opt source               destination
ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:ssh
ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:http

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
```

To permit the return traffic(you will be unable to use `apt-get` or `aptitude` without this):

```
% iptables -I INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
% iptables -L
Chain INPUT (policy ACCEPT)
target     prot opt source               destination
ACCEPT     all  --  anywhere             anywhere             state RELATED,ESTABLISHED
ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:ssh
ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:http

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
```

If you have incoming `ftp` traffic, open `tcp` ports `20` and `21`:
```
% iptables -A INPUT -p tcp --dport 20 -j ACCEPT
% iptables -A INPUT -p tcp --dport 21 -j ACCEPT
% iptables -L
Chain INPUT (policy ACCEPT)
target     prot opt source               destination
ACCEPT     all  --  anywhere             anywhere             state RELATED,ESTABLISHED
ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:ssh
ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:http
ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:ftp-data
ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:ftp

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
```

To accept range of ports(for example if ftp is configured to use passive mode with pasv_max_port=10100 pasv_min_port=10090):
```
% iptables -A INPUT -p tcp --dport 10090:10100 -j ACCEPT
% iptables -L
Chain INPUT (policy ACCEPT)
target     prot opt source               destination
ACCEPT     all  --  anywhere             anywhere             state RELATED,ESTABLISHED
ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:ssh
ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:http
ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:ftp-data
ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:ftp
ACCEPT     tcp  --  anywhere             anywhere             tcp dpts:10090:10100

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
```

### How to set drop policy

Now we can either block all other ports by appending corresponding rule:

```
% iptables -A INPUT -j DROP
% iptables -L
Chain INPUT (policy ACCEPT)
target     prot opt source               destination
ACCEPT     all  --  anywhere             anywhere             state RELATED,ESTABLISHED
ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:ssh
ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:http
ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:ftp-data
ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:ftp
ACCEPT     tcp  --  anywhere             anywhere             tcp dpts:10090:10100
DROP       all  --  anywhere             anywhere

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
```

Or(which is much better and correct way) by specifying `DROP` policy:

```
% iptables -P INPUT DROP
% iptables -L
Chain INPUT (policy DROP)
target     prot opt source               destination
ACCEPT     all  --  anywhere             anywhere             state RELATED,ESTABLISHED
ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:ssh
ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:http
ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:ftp-data
ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:ftp
ACCEPT     tcp  --  anywhere             anywhere             tcp dpts:10090:10100

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
```
Now in `Chain INPUT` we list accessible ports, all other connections will be dropped.

To remove the rule you can use `-D` with insertion line or specify rule number:
```
% iptables -D INPUT -p tcp --dport 10090:10100 -j ACCEPT
OR
% iptables -D INPUT 6

% iptables -L
Chain INPUT (policy DROP)
target     prot opt source               destination
ACCEPT     all  --  anywhere             anywhere             state RELATED,ESTABLISHED
ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:ssh
ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:http
ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:ftp-data
ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:ftp

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
```

### How to save iptables

After first reboot all changes will gone, this is intended behavior since you can specify `DROP policy` or add `DROP all` rule in clean `Chain INPUT` which will log you out and permanently forbid server access.

If you are satisfied with your rules and want them to be applied automatically after reboot, the best way is to install `iptables-persistent` package with

```
aptitude install iptables-persistent
```

and then save the rules to special file, for `IPv4` and `IPv6` correspondingly:

```
% iptables-save > /etc/iptables/rules.v4
% ip6tables-save > /etc/iptables/rules.v6
```

Useful links:<br/>
[Ubuntu documentation: IptablesHowTo](https://help.ubuntu.com/community/IptablesHowTo)<br/>
[Ubuntu forum: iptables howto/tips](http://ubuntuforums.org/showthread.php?t=159661)<br/>
