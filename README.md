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
























