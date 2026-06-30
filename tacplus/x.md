# AAA with TACACS+ Configuration Guide

AAA (Authentication, Authorization, Accounting) is a security framework for controlling and auditing users access to network devices. It answers three distinct questions:

- Authentication: Who are you? Verify identity before granting any access.
- Authorization: What you are allowed to do? Constrain commands and privilege levels per users.
- Accounting: What did you do? Log activity for auditing, forensics, and compliance. 

### Common protocols that implement this framework include:

- RADIUS (Remote Authentication Dial-In User Service)
- TACACS+ (Terminal Access Controller Access Control System Plus)

| | RADIUS | TACACS+ |
|---|---|---|
| Transport | UDP (auth 1812, acct 1813) | TCP (49) |
| Encryption | Password attribute only | Entire packet body |
| AAA model | Combines authN + authZ | Separates all three independently |
| Command-level control | Limited | Per-command, per-privilege-level |
| Typical use | Network access (802.1X, VPN, Wi-Fi) | Router/switch/firewall admin |
| Origin | Open standard (RFC 2865) | Cisco-developed (RFC 8907) |

## Lab Topology

![alt text](image-1.png)

## Cisco IOS / IOS-XE

[Cisco IOS-XE TACACS+ Configuration Guide](
https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9200/software/release/17-17/configuration_guide/sec/b_1717_sec_9200_cg/configuring_tacacs_.html)

[Cisco IOS-XE AAA Command Reference](
https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-18/command_reference/b_1718_9300_cr/security_commands.html)

### Prerequisites (connectivity + SSH)

```
hostname spectre
ip domain-name phantomx.local

crypto key generate rsa modulus 2048

ip ssh version 2
ip ssh time-out 60
ip ssh authentication-retries 3

interface GigabitEthernet 0/0
    ip address 192.168.1.1 255.255.255.0
    no shutdown
```

Verify Reachability:

```
ping 192.168.1.10
ping 192.168.1.11
```

### Create a local fallback user

```
username diana privilege 15 secret moonfall
enable secret lost_dream
```

### Enable AAA, define key and TACACS+ servers

```
aaa new-model

tacacs server TAC1
    address ipv4 192.168.1.10
    key sewer_mermaid

tacacs server TAC2
    address ipv4 192.168.1.11
    key sewer_mermaid
```

You can also set a global key with `tacac-server key <key>`, but remember that the per-server key (if present) always overrides the global one:

```
tacac-server key sewer_mermaid
```

### Put your servers in a group

```
aaa group server tacacs+ TAC_GROUP
    server name TAC1
    server name TAC2
    ip tacacs source-interface GigabitEthernet 0/0
```

> The `source-interface` ensures packets leave from the IP your daemons expect (this will matter if your daemon's host stanza restricts by source IP)

### Step 3: Authentication

```
aaa authentication login default group TAC_GROUP local
aaa authentication enable default group TAC_GROUP enable
```
> `group TAC_GROUP` is tried first (both servers, in order), `local` is the fallbacks if the whole group is unreachable.

 The default lists get applied to the console and VTY lines automatically, so you don't strictly need to apply them under line vty / line con 0 unless you're using named lists.

**Named lists examples, if you want different behavior per line:**

 ```
aaa authentication login VTY_AUTH group TAC_GROUP local
aaa authentication login CONSOLE_AUTH local

line vty 0 4
    login authentication VTY_AUTH

aaa authorization console 
line con 0
    login authentication CONSOLE_AUTH
 ```

Test authentication: 

 ```
test aaa group TAC_GROUP riven valor legacy
```

### Authorization

```
aaa authorization exec default group TAC_GROUP local
aaa authorization commands 1 default group TAC_GROUP local
aaa authorization commands 15 default group TAC_GROUP local

```

### Accounting

```
aaa accounting exec default start-stop group TAC_GROUP
aaa accounting commands 1 default start-stop group TAC_GROUP
aaa accounting commands 15 default start-stop group TAC_GROUP
```

### Lines:

```
line vty 0 4
    exec-timeout 10 0
    logging synchronous
    transport input ssh

line vty 5 15
    transport input none

aaa authorization console
line console 0
    exec-timeout 5 0
    logging synchronous
```

## NX-OS Configuration

[Cisco Nexus AAA Configuration Guide](https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/106x/configuration/security/cisco-nexus-9000-series-nx-os-security-configuration-guide-release-106x/m-configuring-aaa.html)

[Cisco Nexus TACACS+ Configuration Guide:](https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/106x/configuration/security/cisco-nexus-9000-series-nx-os-security-configuration-guide-release-106x/m-configuring-tacacs.html)

### Prerequisites (Connectivity + SSH)

```
ssh key rsa 2048 force
feature ssh
ssh login-attempts 3
ssh login-gracetime 60

interface mgmt 0
    ip address 192.168.1.254/24
    vrf member management
```
> SSH should be enable by default on NX-OS, if not use `feature ssh`

> The default 1024-bit RSA is considered weak by modern standards, overwrite the existing 1024-bit key with a new 2048-bit one

Verify reachability to the AAA servers:

```
ping 192.168.1.10 vrf management

```

If you run into reachability problem with tacplus+ after wiping the node, consider checking the ARP table. (took me nearly 2 hours of debugging 😭)

```
show ip arp vrf management 
clear ip arp 192.168.1.10 vrf management
```

### Enable the tacacs+ feature, define key and servers

```
feature tacacs+

tacacs-server key sewer_mermaid
tacacs=server host 192.168.1.10
tacacs-server host 192.168.1.11
```


### Step 2: Define the server group

```
aaa group server tacacs+ TAC_GROUP
    server 192.168.1.10
    server 192.168.1.11
    use-vrf management
    source-interface mgmt0
    deadtime 5
```

> `deadtime` controls how long NX-OS will skip a TACACS+ server after it's been marked unresponsive/dead, before trying it again

### Step 3: Authentication

NX-OS-specific things here, login console is a separate, explicit method list — NX-OS doesn't silently fold console into default the way IOS does, so if you want the console line to use TACACS+ too, you state it.

```
aaa authentication login default group TAC_GROUP local
aaa authentication login error-enable
aaa authentication login ascii-authentication
```
> `error-enable` line is just for error messages, I recommend having it on duirng set up, you can always drop it later.

> By default, NX-OS will attempt to authenticate using `PAP`. You can either change your TACACS+ server config to match, or enable `ASCII authentication` on the switch instead.


### Authorization
NX-OS authorization doesn't have privilege-level buckets like IOS's commands 1 / commands 15. Instead, config-commands and commands apply to all configuration and exec commands respectively, and the actual permit/deny logic comes from RBAC roles the server hands back.

```
aaa authorization config-commands default group TAC_GROUP local
aaa authorization commands default group TAC_GROUP local
```

### Step 5: Accounting

```
aaa accounting default group TAC_GROUP
```

> NX-OS accounting is a single line covering both exec and command accounting

### Verify (IOS)

For debugging when something doesn't work, `debug tacacs` and `debug aaa authentication` on the router plus watching the daemon's log side by side will show you exactly where it breaks

```
test aaa group TAC_GROUP diana moonfall legacy
show tacacs
show aaa servers
debug tacacs 
debug aaa authentication
```

### NX

```
show running=config aaa
show running-config tacacs+
show tacacs-server
show tacacs-server groups
show aaa groups

show tacacs-server status
ping 192.168.1.x vrf management

show user-account
show users
```

in your ubuntu you need:

```
Host 192.168.1.1
    KexAlgorithms +diffie-hellman-group14-sha1,diffie-hellman-group-exchange-sha1
    HostKeyAlgorithms +ssh-rsa
    PubkeyAcceptedAlgorithms +ssh-rsa
```

For testing, shut down one link and see if another TAC server take over. 

Now shut down both and see if your local fall back work, wait for the time out, this will take a number of seconds, this is normal!

When tacacs+ server are up local account should not work. Local will only take over, if the TAC_GROUP doesn't respond. local is only consulted if the group itself failed to respond (timeout/unreachable)

> aaa authentication login default group TAC_GROUP local


Host 192.168.1.1
    KexAlgorithms +diffie-hellman-group14-sha1,diffie-hellman-group-exchange-sha1
    HostKeyAlgorithms +ssh-rsa
    PubkeyAcceptedAlgorithms +ssh-rsa


