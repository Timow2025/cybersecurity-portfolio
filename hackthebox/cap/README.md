# Cap — Hack The Box

> **Writeup status:** COMPLETE · **Publication eligibility:** VERIFIED RETIRED — [HTB retirement record](https://labs.hackthebox.com/achievement/machine/2120988/351), checked October 29, 2026
>
> This writeup documents only the steps supported by the author's screenshots and statements. It excludes credential values, flag contents, real lab IP addresses, and raw packet captures.

## Overview

| Field | Details |
| --- | --- |
| Machine | Cap |
| Platform | Hack The Box |
| Operating system | Linux |
| Difficulty | Easy |
| Machine page | [Cap](https://app.hackthebox.com/machines/351) |
| Author | [Timow2025](https://github.com/Timow2025) |
| Exercise evidence dated | 2026-10-29 |
| Completed on | 2026-10-29 — both flags submitted, per the author |
| Published on | 2026-10-29 |
| Retirement verified on | 2026-10-29 — [official HTB record](https://labs.hackthebox.com/achievement/machine/2120988/351) |
| Key techniques | Web enumeration, exposed packet capture, plaintext FTP credential disclosure, SSH access, Linux file capabilities |

Cap exposed FTP, SSH, and a Gunicorn web application. The web dashboard was accessible without authentication in the assigned lab and offered a downloadable packet capture. That capture showed a plaintext FTP password attempt for `nathan`; the author confirmed that the recovered password authenticated to SSH as that user. Local enumeration then revealed `CAP_SETUID` on the Python 3.8 interpreter, which the author used to open a shell with root UID. The author confirms that both flags were submitted, without disclosing their values.

**Attack path**

```text
Unauthenticated dashboard on port 80
    → downloadable security snapshot PCAP
    → FTP USER/PASS for nathan visible in cleartext
    → SSH shell as nathan
    → /usr/bin/python3.8 with CAP_SETUID
    → shell with UID 0
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

The author assessed the assigned Cap machine in an authorized Hack The Box laboratory session. The supplied scan and SSH screenshots are dated October 29, 2026. The documented activity is limited to the assigned machine and its exposed services.

- **In scope:** Cap and its FTP, SSH, and HTTP services in the HTB lab.
- **Out of scope:** Other machines, unrelated infrastructure, and production systems.
- **Attacker environment:** Kali Linux with Nmap 7.99 and Wireshark; the exact Wireshark version was not recorded.
- **Target notation:** `TARGET_IP` replaces the lab machine address in commands and output.
- **Evidence handling:** Passwords, flag values, and real IP addresses are omitted. No raw screenshots or PCAP are included; some of those files contain credentials or real IP addresses.

This writeup documents authorized laboratory activity only.

## Recon

### Nmap Enumeration

**Objective:** Identify reachable services and guide subsequent enumeration.

The author ran the following service scan. `-sV` requested version detection; `-Pn` skipped host discovery.

```bash
nmap TARGET_IP -sV -Pn
```

**Relevant sanitized output**

```text
Host is up.
Not shown: 997 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.2
80/tcp open  http    Gunicorn
Service Info: OSs: Unix, Linux
```

| Port / Protocol | Service | Observed evidence | Next step |
| --- | --- | --- | --- |
| 21/tcp | FTP | vsftpd 3.0.3 | Inspect whether the service exposes useful data or credentials |
| 22/tcp | SSH | OpenSSH 8.2p1 Ubuntu | Test only if a credential is obtained |
| 80/tcp | HTTP | Gunicorn | Inspect the web application |

The screenshot shows Nmap's default 1,000 TCP ports, not a full TCP or UDP survey. No inference is made about unscanned ports.

### Web Enumeration

Browsing the HTTP service on port 80 opened a dashboard without an authentication step, according to the author. The visible navigation offered `Security Snapshot (5 Second PCAP + Analysis)`, `IP Config`, and `Network Status`. The screenshot shows those menu entries, but does not establish what the latter two pages contained.

Opening **Security Snapshot** displayed a packet summary and a **Download** button. The page reported 72 total packets, including 69 IP packets and 69 TCP packets, with zero UDP packets. The author downloaded the capture and inspected it in Wireshark.

The visible FTP control stream contained:

```text
USER nathan
331 Please specify the password.
PASS [REDACTED_PASSWORD]
```

The `PASS` value was readable in the capture. The supplied Wireshark screenshot stops before the server response to `PASS`, so it does not prove that the FTP login succeeded. It does prove that an FTP password attempt was transmitted in cleartext and exposed through the download. FTP's `USER` and `PASS` exchange is known to expose credentials without a protected control channel; see [RFC 2228](https://www.rfc-editor.org/rfc/rfc2228.html).

**Analysis:** The unauthenticated dashboard made a sensitive packet capture available to a visitor. The captured password became a candidate for the SSH service identified earlier. The credential's validity for SSH was confirmed by the author in the next step.

### Other Service Enumeration

No additional direct FTP enumeration or successful FTP authentication was documented. The supplied evidence does not show content from `IP Config` or `Network Status`.

## Initial Access

**Finding:** A password exposed in a downloadable FTP capture authenticated to SSH as `nathan`.

**Preconditions:** Access to the unauthenticated web dashboard and the packet-capture download; SSH reachable on port 22.

**Reasoning:** The capture showed `USER nathan` followed by a plaintext `PASS` command. The author tested that password against the `nathan` SSH account.

### Reproduction

The exact SSH invocation was not visible in the screenshot. This is the equivalent sanitized form of the step the author described:

```bash
ssh nathan@TARGET_IP
```

The password is intentionally omitted. The resulting session displayed an Ubuntu login banner and the shell prompt:

```text
nathan@cap:~$
```

**Verification:** The prompt and the author's confirmation support an interactive SSH session as `nathan`. The screenshot does not include a separate `id` command for this session.

**Outcome:** Initial access to the operating system was obtained as `nathan`. This confirms the captured password was valid for SSH; it does not establish whether the FTP login attempt succeeded.

## User Escalation

Not applicable — initial access already provided the `nathan` user context. The next documented transition was from `nathan` to a shell with root UID.

## Root Escalation

**Starting context:** Interactive SSH shell as `nathan`.

**Finding:** The general-purpose Python 3.8 interpreter had the `cap_setuid` file capability.

### Discovery

The author enumerated file capabilities:

```bash
getcap -r / 2>/dev/null
```

**Relevant output**

```text
/usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip
```

`CAP_SETUID` permits changing process user IDs. Giving that capability to an interpreter allows code run through it to request UID 0, subject to normal capability execution rules. The `cap_net_bind_service` capability was also present, but was not needed for the demonstrated escalation. See [capabilities(7)](https://man7.org/linux/man-pages/man7/capabilities.7.html) and [setuid(2)](https://man7.org/linux/man-pages/man2/setuid.2.html).

### Reproduction and Verification

The author invoked the capability-bearing interpreter directly, changed UID to 0, and started Bash:

```bash
/usr/bin/python3.8 -c 'import os; os.setuid(0); os.system("/bin/bash")'
id
```

**Observed output**

```text
uid=0(root) gid=1001(nathan) groups=1001(nathan)
root@cap:~#
```

**Outcome:** An interactive shell with root UID was confirmed. The GID and supplementary group remained `nathan`, so this writeup does not describe the resulting process as having root group membership. The author confirms that both flags were submitted; no flag values are shown.

**Lab cleanup:** No manual cleanup steps were documented. After submitting both flags, the author observed that the individual HTB instance was offline. That observation is separate from the machine's officially retired status.

## Remediation

| Finding | Demonstrated impact | Recommended fix | Validation |
| --- | --- | --- | --- |
| Unauthenticated dashboard and PCAP download | Visitors could obtain a capture containing credentials | Enforce server-side authentication and per-request authorization on both the snapshot page and the download endpoint; limit capture retention and access | Anonymous requests to both endpoints are denied; authorized exports contain no credentials |
| Plaintext FTP credential in traffic | A password was readable in the PCAP | Replace plaintext FTP with a protected protocol or require TLS on the FTP control channel; rotate the exposed password | New captures contain no readable `PASS` value; the old password no longer works for SSH |
| Credential valid for SSH | Exposure of one captured password yielded a user shell | Use separate, unique credentials for services and accounts; rotate the affected SSH credential | A captured service credential cannot authenticate to SSH |
| `CAP_SETUID` on `/usr/bin/python3.8` | `nathan` used Python to launch a shell with UID 0 | Remove `CAP_SETUID` from the interpreter and avoid granting dangerous file capabilities to general-purpose runtimes; move any legitimate privileged action to a narrow, audited service | `getcap` no longer lists `cap_setuid` on Python, and a non-root `os.setuid(0)` test fails |

The web authorization recommendation follows the [OWASP Authorization Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html). For FTP control-channel protection, see [RFC 4217](https://www.rfc-editor.org/rfc/rfc4217.html). Changes to file capabilities should account for any legitimate need for the interpreter's other capability; `setcap -r` would remove all capabilities from that file, not just `CAP_SETUID` ([setcap(8)](https://man7.org/linux/man-pages/man8/setcap.8.html)).

**Detection opportunities:** Monitor anonymous snapshot downloads, unexpected exports of sensitive packet payloads, FTP credential use without transport protection, SSH logins to `nathan` from unexpected sources, and launches of the capability-bearing Python interpreter followed by UID 0 processes.

## Lessons Learned

- **Technical insight:** A short, downloadable packet capture can disclose an account password when a plaintext protocol is present. A separate file-capability misconfiguration then turned user access into root-UID execution.
- **What did not work:** No failed exploitation attempt was documented. The visible FTP `PASS` request alone was insufficient to establish successful FTP authentication; the later SSH session proved the captured password was valid for SSH.
- **Process improvement:** Preserve the distinction between a plausible capability-based escalation and a verified root shell. Here, `id` supplied the verification and also showed the group context remained `nathan`.
- **Further practice:** Review Linux file-capability inheritance and safer designs for collecting and sharing network captures.
- **Assistance used:** AI assistance was used to organize the evidence, check capability semantics, suggest a verification approach, and draft this writeup. The author performed the lab actions and supplied the screenshots.

## References

- [Cap machine page](https://app.hackthebox.com/machines/351)
- [Official HTB retirement record for Cap](https://labs.hackthebox.com/achievement/machine/2120988/351)
- [HTB publication guidelines](https://help.hackthebox.com/en/articles/5188925-streaming-writeups-walkthrough-guidelines)
- [Nmap documentation](https://nmap.org/docs.html)
- [RFC 2228 — FTP Security Extensions](https://www.rfc-editor.org/rfc/rfc2228.html)
- [RFC 4217 — Securing FTP with TLS](https://www.rfc-editor.org/rfc/rfc4217.html)
- [Linux capabilities(7)](https://man7.org/linux/man-pages/man7/capabilities.7.html)
- [Linux setuid(2)](https://man7.org/linux/man-pages/man2/setuid.2.html)
- [Linux setcap(8)](https://man7.org/linux/man-pages/man8/setcap.8.html)
- [Python 3.8 `os.setuid()`](https://docs.python.org/3.8/library/os.html#os.setuid)
