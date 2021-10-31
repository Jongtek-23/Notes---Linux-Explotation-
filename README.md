# Notes---Linux-Explotation-

# 

### Remote Enumeration - Information Gathering - Commands

##### Enumerating NFS (Network File System)
```bash
nmap --script nfs-ls,nfs-showmount,nfs-statfs <IP target>
showmount -e <IP target>

mkdir -p /mnt/home/bob
mount -t nfs <NFS Server IP>:/home/bob /mnt/home/bob -o nolock
```

##### Enumerating Portmapper (rpcbind) ->  port 111/TCP
```bash
nmap --script rpc-grind,rpcinfo <IP target> -p 111
rpcinfo -p <IP target>
```

##### SAMBA -> SMB/CIFS 
- It provides print and file sharing services for windows clients within an environment
- SAMBA can be found listening on the usual "NetBIOS" ports
```bash
netbios-ns      137/tcp     # NETBIOS Name Service
netbios-ns      137/udp
netbios-dgm     138/tcp     # NETBIOS Datagram Service
netbios-dgm     138/udp
netbios-ssn     139/tcp     # NETBIOS Session Service
netbios-ssn     139/udp
microsoft-ds    445/tcp     # Microsoft Naked CIFS
```

- Using nmap:
`nmap -sT -sU -sV <IP target> -p135,137,138,139,445 --open`

- Enemarate SMB:
```bash
nmap --script smb-enum-shares <IP target>

smbclient -L <IP target>
smbmap -H <IP target>
```
- SMBMAP allows interact with the remote file system, searching file contents for specific strings, and even execiting commands.

- Connect SMB shares:
```bash
smbclient \\<IP target>\<name share smb>
```
- To mount SMB shares:
```bash
apt install cifs-utils

mkdir -p /mnt/www
mount -t cifs \\<IP target>\www /mnt/www
cd /mnt/www
ls -las
```
##### SAMBA - SMB User -> enumarate users over the SMB protocol
- Enumerate with `rpcclient`
```bash
for u in $(users.txt); do rpcclient -U "" <IP target> -N --command="lookupnames $u"; done | grep "User: 1"


The “User: 1” in the output indicates that the user exists on the remote system.
```
- Some useful `rpcclient` commands include "lookupsids," "netshareenum," "srvinfo" and "enumprivs" to mention several. 

- Enumerate with `enum4linux`
```bash
• Operating System
• Users
• Password policies
• Group membership
• Shares
• Domain/Workgroup Identification
```
- Command `enum4linux` against a SAMBA server:
```bash
enum4linux <IP target>
```
##### Enumerating SMTP Users

- Verbs of SMTP : `"HELO", "RCPT" or "MAIL"`

- Sent an email while directly connected to an email server via `telnet` or some other way :
```bash
telnet mail.server.site 25
```
- A large majority of mail servers in-use are Linux-based, we’ll be focusing on enumerating users from `Sendmail`, a popular Open-Source *NIX-based mail server. 

- Enumerate which options, "verbs" or "features" are enabled on an SMTP server -> 25/tcp
```bash
nmap --script smtp-commands 192.168.13.26 -p 25
```
- Direct enumeration options SMTP with `telnet` oe `netcat`, tp see which verbs are enebled on the mail server :
```bash
telnet <IP target> 25
>> help

nc <IP target> 25
>> help
```
- Using the verb `RCPT TO` -> who we’re sending the mail to, in this case, the user we’re enumerating.
- First , connect:
```bash
telnet <IP target> 25
```
- Second, `HELO` to a domain name :
```bash
> HELO tester.localdomain
```
- Third, we tell the server who the mail will be from with the "MAIL FROM":
```bash
> MAIL FROM: tester@tester.localdomain
```
- Fourth, enumerate potential users from the system with the "RCPT TO: <user@domain.com>" command:
```bash
> RCPT TO: root@server2.localdomain
> RCPT TO: jongtek@server2.localdomain
```
- Another feature we can use to enumerate users is the `EXPN` feature.
```bash
> EXPN root
> EXPN jongtek
```
-  `VRFY` , to enumerate users:
```bash
telnet <IP target> 25
> HELO foo
> VRFY root
> VRFY jongtek
```
- `smtp-user-enum` by pentestmonkey. A tool that automates the user enumeration for SMTP:
```bash
smtp-user-enum -M VRFY -U user.txt -t <IP target>
smtp-user-enum -M EXPN -u admin -t <IP target>
smtp-user-enum -M RCPT -U user.txt -T mail-servers-ips.txt
smtp-user-enum -M EXPN -D example.com -U user.txt -t <IP target>
```

### Local Enumeration - Information Gathering - Commands


#### Local Enumeration - Network Information

- Important questions:
```bash
• How is the exploited machine connected to the network?
• Is the machine multi-homed? Can we pivot to other hosts in other segments?
• Do we have unfettered outbound connectivity to the internet or is egress traffic limited to certain ports or protocols?
• Is there a firewall between me and other devices/machines?
• Is the machine I’m on, communicating with other hosts? If so, what is the purpose or function of the hosts that my current machine is communicating with?
• What protocols are being used that are originating from my actively exploited machine? Are we initiating FTP connections or other connections (SSH, etc.) to other machines?
• Are other machines initiating a connection with me? If so, can that traffic be intercepted or sniffed as cleartext in transit? Is it encrypted?
```
- `ifconfig` is used to get information regarding our current network interfaces. We want to know what our IP address is, and whether or not there are additional interfaces that we may be able to use as pivots to other network segments:
```bash
ifconfig -a

'-a' -> for a full listing of interfaces
```
- `route` is used to print our current network routes, which includes our gateway of course. Knowing what our static routes and gateway are can help us in case we need to manually configure our network interfaces, pivot to other network segments, or will come in handy if we decide to execute ARP-poisoning or other Man-In-The-Middle- style attacks.
```bash
route -n
```
- `traceroute` is used to know how many hops are between our compromised machine:
```bash
traceroute -n <IP target>
```
- DNS information:
```bash
cat /etc/resolv.conf

Question:
- What machine is resolving our DNS queries?
- Can we use it to exfiltrate data over a DNS tunnel?
- Is the DNS server vulnerable to any exploits?
- Is it an Active Directory controller?
```
- `ARP cache`. This information is useful when it comes down to determining who we’re communicating with, what’s being communicated, and whether that traffic or communication has any value to us from an exploitation perspective. I.E., credentials transmitted in cleartext, etc.
```bash
apr -en
```
- `netstat`
```bash
netstat -auntp
netstat -tulpn

This gives us:
• What other machines or devices we are currently connected to
• Which ports or services on other machines we are connected to
• What ports our current machine are listening on
• Are there other systems establishing connections with our current machine
```
- `netstat without netstat`:
```bash
cat /proc/net/tcp
cat /proc/net/udp
```
- `ss` , an alternative to 'netstat':
```bash
ss -twurp
```
- `Outbound Port Connectivity`. Check outbound firewall rules. Knowing this information will come in handy if and when we need to establish outbound connections to other systems we control for the purpose of maintaining access or exfiltrating data.
```bash
nmap -sT -p4444-4450 portquiz.net

- Considering using nmap’s --T (timing) option at a low value to stay under any internal IDS radar.
```
#### Local Enumeration - System Information






















