# Orion — Hack The Box

> **Writeup status:** COMPLETE · **Publication eligibility:** VERIFIED RETIRED — [official HTB record](https://labs.hackthebox.com/achievement/machine/2473840/945), checked October 31, 2026
>
> This writeup follows the author's screenshots and explanation. It omits lab IP addresses, passwords, password hashes, application keys, flag values, and raw screenshots.

## Overview

| Field | Details |
| --- | --- |
| Machine | Orion |
| Platform | Hack The Box |
| Operating system | Linux; the SSH banner showed Ubuntu 22.04.5 LTS |
| Machine page | [Orion](https://app.hackthebox.com/machines/945) |
| Author | [Timow2025](https://github.com/Timow2025) |
| Completed on | 2026-10-31 — HTB showed “You have solved Orion!” |
| Published on | 2026-10-31 |
| Retirement verified on | 2026-10-31 — [official HTB achievement record](https://labs.hackthebox.com/achievement/machine/2473840/945) |
| Key techniques | Craft CMS pre-authentication RCE, application secret exposure, password hash cracking, SSH credential reuse, local Telnet authentication bypass |

An initial scan found SSH and HTTP. The Orion Telecom website led to a Craft CMS administration login that displayed version 5.6.16. The author used a Metasploit module for CVE-2025-32432 to gain code execution as `www-data`, then read Craft's `.env` file and accessed its local MySQL database. A recovered CMS password hash led to an SSH session as `adam`. From that session, a localhost-only GNU Inetutils Telnet service allowed a root login through CVE-2026-24061. The HTB completion dialog confirms the machine was solved; no credential or flag value is reproduced here.

**Attack path**

```text
Orion Telecom website → Craft CMS 5.6.16
    → CVE-2025-32432 → www-data shell
    → Craft .env → local MySQL database → CMS password hash
    → recovered password → SSH as adam
    → localhost Telnet / CVE-2026-24061 → root prompt
```

## Contents

- [Scope and Authorization](#scope-and-authorization)
- [Recon](#recon)
- [Initial Access](#initial-access)
- [User Escalation](#user-escalation)
- [Root Escalation](#root-escalation)
- [Remediation](#remediation)
- [Lessons Learned](#lessons-learned)
- [References](#references)

## Scope and Authorization

The author assessed the assigned Orion machine in a Hack The Box lab. The HTB completion screenshot is dated October 31, 2026; the supplied evidence shows the assigned machine and local attacker environment. The [official HTB record](https://labs.hackthebox.com/achievement/machine/2473840/945) marks Orion as retired.

- **In scope:** Orion's website, the resulting lab shell, and services on the assigned host.
- **Out of scope:** Other machines, unrelated infrastructure, and production systems.
- **Attacker environment:** Kali Linux with Nmap 7.99, Metasploit Framework, and John the Ripper; the latter two tool versions were not recorded.
- **Target notation:** `TARGET_IP` stands for the assigned lab IP. The lab hostname `orion.htb` is retained where it explains the sequence.
- **Evidence handling:** Screenshots containing `.env` values, the password hash, cracked password, lab addresses, and flag contents are kept out of this document. Commands are sanitized or described without secret values.

This writeup documents authorized lab activity only.

## Recon

### Network Enumeration

The author ran a service scan on October 30, 2026 (UTC). `-sV` requested service-version detection and `-T4` set the timing template. The real lab address is replaced with `TARGET_IP`.

```bash
nmap TARGET_IP -sV -T4
```

**Relevant sanitized output**

```text
Host is up (1.7s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.15
80/tcp open  http    nginx 1.18.0 (Ubuntu)
Service Info: OS: Linux
```

| Port | Observed service | Next step |
| --- | --- | --- |
| 22/tcp | OpenSSH 8.9p1 Ubuntu | Consider only after obtaining a valid account credential. |
| 80/tcp | nginx 1.18.0 (Ubuntu) | Inspect the website and its application. |

The screenshot covers Nmap's default 1,000 TCP ports: two open and 998 closed. It does not establish the state of unscanned TCP ports or UDP. Local socket enumeration after access later showed Telnet at `127.0.0.1:23`; that loopback service was not among the two externally reachable ports in this scan.

### Web Enumeration

The browser screenshot shows an **Orion Telecom** landing page with Home, Services, Solutions, About, and Contact navigation. A separate administration login identifies the application as **Craft CMS 5.6.16**. A test login on that page returned an invalid username or password message. The exact admin URL is not visible in the screenshot; the test credentials are intentionally omitted.

Craft's [security advisory](https://github.com/craftcms/cms/security/advisories/GHSA-f3gw-9ww9-jmc3) lists Craft 5.x through 5.6.16 as affected by CVE-2025-32432 and 5.6.17 as patched. The displayed version therefore supported testing that vulnerability. A version match alone would not prove exploitation; the resulting `www-data` shell is the stronger evidence below.

## Initial Access

**Finding:** Pre-authentication remote code execution in Craft CMS 5.6.16, consistent with CVE-2025-32432.

**Preconditions:** Reachability of the Craft CMS application. The supplied evidence does not show a valid CMS login.

The author ran the [Metasploit Craft CMS module](https://github.com/rapid7/metasploit-framework/blob/master/modules/exploits/linux/http/craftcms_preauth_rce_cve_2025_32432.rb). The screenshot shows the selected module and its `run` command, but not the target and payload option values, so those settings are not reconstructed here.

```text
msf exploit(linux/http/craftcms_preauth_rce_cve_2025_32432) > run
Leaked session.save_path: /var/lib/php/sessions
The target is vulnerable. Session path leaked.
Injecting stub & triggering payload...
```

The module's status output is its own assessment. The subsequent shell provided independent confirmation of execution in the web application context:

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/var/www/html/craft/web
```

**Outcome:** The author obtained a shell as `www-data`, not as the host's root user. The screenshot does not show the full Metasploit session setup or payload type.

## User Escalation

**Starting context:** Shell as `www-data` in the Craft application directory.

### Application Configuration and Database

The author listed `/var/www/html/craft` and read its `.env` file. The visible file contained a Craft application security key and local MySQL connection details, including database `orion` and database user `root`. Their values are not reproduced. The database `root` account is a MySQL identity; it does not mean that the `www-data` shell had operating-system root privileges.

Using the database password from `.env`, the author successfully queried the local MySQL server and saw the `orion` database. A query against the `users` table returned a CMS account named `admin` whose email identified `adam`, followed by that account's bcrypt password hash. The complete hash is omitted.

```sql
-- Sanitized representation of the observed database queries
DESCRIBE users;
SELECT id, username, email FROM users;
SELECT password FROM users WHERE username = 'admin';
```

### Password Recovery and SSH

The author tested the extracted bcrypt hash with John the Ripper and a local wordlist. John reported one recovered password. The hash and plaintext password are intentionally absent from this writeup. An SSH attempt as `admin` failed; a subsequent password-based SSH login as `adam` succeeded. The author's explanation links that login to the recovered password. The screenshots show the successful session, but not the password bytes entered at the SSH prompt.

```bash
# The local hash file contained the extracted CMS hash; its contents are not included.
john --format=bcrypt --wordlist=/usr/share/wordlists/rockyou.txt ~/orion-admin.hash
ssh adam@orion.htb
```

The `adam` shell showed:

```text
uid=1000(adam) gid=1000(adam) groups=1000(adam)
adam@orion:~$
```

The author read the user flag from `adam`'s home directory. Its value is omitted. `sudo -l` reported that `adam` could not run `sudo` on Orion, so the later root path did not use a sudo allowance.

**Outcome:** The author moved from the web service account to an SSH shell as `adam` by recovering and reusing the CMS credential, according to the supplied explanation.

## Root Escalation

**Starting context:** Interactive SSH shell as `adam` (`uid=1000`).

### Local Service Discovery

The author enumerated file capabilities, SUID binaries, timers, cron entries, running services, and listening sockets. Those listings do not show a successful escalation through capabilities, SUID, sudo, or cron. The relevant finding was a running `inetutils-inetd.service` and a Telnet listener bound to `127.0.0.1:23`. A local `telnetd` binary identified itself as GNU Inetutils 2.7.

The loopback address meant the service was not exposed through the host's external interface in the socket listing. Once the author had an `adam` shell on the host, that did not prevent connecting to the service locally.

### Telnet Authentication Bypass

The author ran the following command from the `adam` session:

```bash
USER='-f root' telnet -a 127.0.0.1 23
```

The [GNU Inetutils security advisory](https://www.openwall.com/lists/oss-security/2026/01/20/2) describes this authentication-bypass mechanism; the [Debian security tracker](https://security-tracker.debian.org/tracker/CVE-2026-24061) identifies it as CVE-2026-24061. `telnet -a` sends the client-side `USER` value for automatic login. A vulnerable `telnetd` passes that value unsanitized to `login`; the injected `-f root` option bypasses normal authentication. The advisory lists GNU Inetutils versions 1.9.3 through 2.7 as affected, consistent with the observed 2.7 version.

The resulting screenshot shows a `root@orion:~#` prompt and `root.txt` in the home directory. It also shows `cat root.txt` entered, but the captured frame stops before that command's output. No separate `id` command appears in the root session. The author's explanation identifies this as the root step, and the HTB completion dialog shows Orion solved; the flag value is not included.

**Outcome:** A root login session through the local Telnet service is supported by the root prompt and the documented bypass. The evidence does not show a separate UID verification command or a root flag value.

**Lab cleanup:** No cleanup steps or final state of the individual HTB instance were supplied. The machine's officially retired status is a separate fact from the state of the author's instance.

## Remediation

| Finding | Demonstrated impact | Recommended fix | Validation |
| --- | --- | --- | --- |
| Craft CMS 5.6.16 affected by CVE-2025-32432 | Pre-authentication code execution as `www-data` | Update Craft CMS to 5.6.17 or a later fixed release; review the host for payload artifacts and rotate the Craft security key and exposed credentials | Confirm the patched version and that the former exploit path no longer yields code execution |
| Craft's runtime could access a MySQL `root` credential | The web shell could query the application database and obtain a user password hash | Use an application-specific MySQL account with only required privileges; rotate the exposed database password and protect the `.env` file from unnecessary readers | The application works with a limited account; unrelated databases and privileged MySQL operations are denied |
| CMS credential recovered and reused for host access | A password from the CMS account led to SSH as `adam` | Require unique, strong credentials for CMS and operating-system accounts; rotate the affected password and review SSH access | The former CMS password no longer authenticates to SSH |
| GNU Inetutils Telnet 2.7 authentication bypass | A local `adam` session reached a root login without the normal password check | Disable Telnet if it is not required; otherwise apply a fixed inetutils package and review its login configuration | TCP 23 is closed when Telnet is removed, or the bypass attempt fails on a patched service |

The Craft update and credential-rotation recommendations follow [Craft's incident guidance](https://craftcms.com/knowledge-base/craft-cms-cve-2025-32432). The Telnet recommendation follows the [GNU Inetutils advisory](https://www.openwall.com/lists/oss-security/2026/01/20/2). Binding Telnet to localhost alone did not prevent a user already on the host from reaching it.

**Detection opportunities:** Review web requests for the Craft asset-transform exploit pattern, unexpected access to the Craft `.env` file, MySQL logins from the web service context, SSH logins as `adam`, and local Telnet logins that result in root access. Craft's guidance notes that suspicious requests to the relevant controller indicate probing but do not alone prove compromise.

## Lessons Learned

- **Technical insight:** A web RCE can expose application secrets and database access, even when it initially runs only as `www-data`. A local-only legacy service then became reachable from the user shell and supplied the final privilege transition.
- **What did not work:** A direct test login to the Craft admin page failed, SSH as `admin` failed, and `sudo -l` offered no route for `adam`.
- **Evidence discipline:** The Craft version and module check supported the CVE hypothesis; the `www-data` shell confirmed code execution. The root prompt supports the Telnet outcome, while a separate `id` for that session was not captured.
- **Further practice:** Review how services restricted to localhost can still affect host privilege boundaries, and how application database privileges and password reuse influence post-exploitation risk.
- **Assistance used:** The author used the public Metasploit module and John the Ripper. AI assistance organized the supplied evidence, checked primary sources, and drafted this writeup; the author performed the lab actions.

## References

- [Orion machine page](https://app.hackthebox.com/machines/945)
- [Official HTB retirement record](https://labs.hackthebox.com/achievement/machine/2473840/945)
- [HTB publication guidelines](https://help.hackthebox.com/en/articles/5188925-streaming-writeups-walkthrough-guidelines)
- [Nmap reference guide](https://nmap.org/book/man.html)
- [Craft CMS advisory for CVE-2025-32432](https://github.com/craftcms/cms/security/advisories/GHSA-f3gw-9ww9-jmc3)
- [Craft CMS incident guidance](https://craftcms.com/knowledge-base/craft-cms-cve-2025-32432)
- [Rapid7 Metasploit Craft CMS module](https://github.com/rapid7/metasploit-framework/blob/master/modules/exploits/linux/http/craftcms_preauth_rce_cve_2025_32432.rb)
- [GNU Inetutils Telnet security advisory](https://www.openwall.com/lists/oss-security/2026/01/20/2)
- [Debian security tracker: CVE-2026-24061](https://security-tracker.debian.org/tracker/CVE-2026-24061)
- [John the Ripper documentation](https://www.openwall.com/john/doc/)
