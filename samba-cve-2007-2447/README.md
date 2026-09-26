# Samba "Username Map Script" Command Execution

**Author:** Miguel Bernath Liao
**Date:** 09/17/2026

---

## Environment & Scope

This exercise was performed in an isolated home lab for educational purposes, against a system I own and control, on a private network with no route to the public internet.

| | |
|---|---|
| **Attacker** | Kali Linux (`10.0.2.3`) |
| **Target** | Metasploitable 2 (`10.0.2.5`) — an intentionally vulnerable training VM |
| **Network** | Private, host-only VirtualBox network (`10.0.2.0/24`) |
| **Vulnerability** | CVE-2007-2447 — Samba `username map script` command execution |

---

## Discovery and Researching

I began by scanning the target machine, 10.0.2.5, for the ports commonly associated with Samba Services:

```bash
nmap -p 139,445 10.0.2.5
```

The scan returned the following results:

![Nmap scan results](images/01-nmap-scan.png)

Port 139 is commonly used for NetBIOS session services, while port 445 provides SMB directly over TCP. Because both ports were open, I investigated whether the target's SMB/Samba service contained any known vulnerabilities.

My research led me to **CVE-2007-2447**, known as the Samba "username map script" command-execution vulnerability. It affects certain Samba versions from 3.0.0 through 3.0.25rc3 when vulnerable configuration options are enabled. Improper handling of shell metacharacters can allow commands supplied through specially crafted usernames to be executed by the server. [Rapid7's vulnerability description](https://www.rapid7.com/db/vulnerabilities/cve-2007-2447/) identifies this as a high-severity vulnerability and associates it with the Metasploit usermap_script module.

It is important to note that finding ports 139 and 445 open does not prove that the system is vulnerable. The ports only indicate that an SMB-related service is reachable. The software version, configuration, and controlled vulnerability testing must also be considered.

## Performing the Attack

I started the Metasploit Framework from my Kali Linux machine:

![Starting the Metasploit Framework](images/02-metasploit-start.png)

I then selected the Samba username map script exploit:

![Selecting the exploit module](images/03-module-select.png)

Metasploit initially selected its default Unix reverse-shell payload. I configured the target address using, RHOSTS, I selected a Unix reverse-command-shell payload, and I set a LHOST to the IP address of my kali machine. Lastly, I launched the exploit.

![Configuring options](images/04-options-config-1.png)
![Configuring options](images/05-options-config-2.png)

Metasploit started a reverse TCP handler on 10.0.2.3:4444. The output then reported the shell was opened:

![Reverse handler and session opened](images/06-session-opened.png)

This message confirmed that the exploit successfully created a command-shell session between the Kali machine and the Metasploitable target.

To determine the privilege level of the shell, I ran:

![Running id in the shell](images/07-id-command.png)
![Output confirming root](images/08-id-output-root.png)

This confirmed that the attack was successful and that the resulting shell had root privileges. Root access represents a complete compromise of the target because it allows unrestricted control over its operating system, files, services, and user accounts.

## Protecting Against the Attack

The primary defense is to upgrade Samba to a supported version that is not vulnerable to CVE-2007-2447. Legacy Samba installations should not remain in production, especially when they are exposed to untrusted networks.

Additional protections include:

- Disable the username map script option if it is not required.
- Restrict access to TCP ports 139 and 445 using host and network firewalls.
- Permit SMB connections only from trusted systems and network segments.
- Do not expose SMB services directly to the public internet.
- Segment file servers from user workstations and other sensitive systems.
- Run Samba with the minimum privileges necessary instead of root whenever possible.
- Regularly scan systems for outdated software and insecure configurations.
- Monitor SMB authentication attempts and unusual outbound connections.
- Use intrusion-detection or endpoint-monitoring tools to identify unexpected command execution.
- Replace intentionally vulnerable systems such as Metasploitable with supported operating systems outside isolated training environments.

---

## References

- [Rapid7 — CVE-2007-2447](https://www.rapid7.com/db/vulnerabilities/cve-2007-2447/)
- [NVD — CVE-2007-2447](https://nvd.nist.gov/vuln/detail/CVE-2007-2447)
