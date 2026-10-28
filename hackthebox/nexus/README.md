# Nexus — Hack The Box

> **Writeup status:** COMPLETE · **Publication eligibility:** VERIFIED RETIRED — [HTB retirement record](https://labs.hackthebox.com/achievement/machine/3484863/948), checked October 28, 2026
>
> This writeup describes an authorized Hack The Box lab machine. Flag values, credentials, tokens, private addresses, and unsanitized screenshots are excluded.

## Overview

| Field | Details |
| --- | --- |
| Machine | Nexus |
| Platform | Hack The Box |
| Operating system | Linux (Ubuntu) |
| Difficulty | Not independently verified |
| Machine page | [Nexus](https://app.hackthebox.com/machines/948) |
| Author | [Timow2025](https://github.com/Timow2025) |
| Completed on | Date not recorded in the available evidence |
| Published on | 2026-10-28 |
| Retirement verified on | 2026-10-28 — [official HTB retirement record](https://labs.hackthebox.com/achievement/machine/3484863/948) |
| Key techniques | Virtual host enumeration, exposed configuration files, installer abuse, PHP code execution, credential reuse, Git tree path traversal, privileged scheduled task abuse |

Nexus is a Linux machine exposing SSH and an nginx web server hosting multiple virtual hosts. Enumeration revealed a Git service and a Krayin CRM installation. A leftover installation endpoint allowed the creation of an administrative application account, while an unsafe attachment-handling mechanism enabled server-side PHP execution.

Credentials discovered through application configuration were subsequently reused to access a Linux account and the Gitea service. Finally, a privileged repository synchronization process processed a crafted Git tree with a traversal path. That path modified a root-managed cron configuration; the scheduled payload obtained the root flag without an interactive root shell.

**Attack path**

```text
Nmap and virtual host enumeration
    |
    v
git.nexus.htb / billing.nexus.htb
    |
    v
Exposed Krayin installation endpoint
    |
    v
Administrative account creation
    |
    v
PHP attachment upload and execution
    |
    v
Reverse shell (www-data)
    |
    v
Exposed .env credentials
    |
    v
Password reuse -> SSH as jones
    |
    v
Gitea access as jones
    |
    v
Crafted Git tree -> template-sync path traversal
    |
    v
Privileged cron file modification
    |
    v
Root-run cron payload reads the root flag
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

The author assessed the assigned Nexus machine in an authorized Hack The Box laboratory environment. The exact exercise date was not recorded in the available evidence.

- **In scope:** The assigned Nexus machine and its exposed services.
- **Out of scope:** Other HTB systems, unrelated hosts, and external infrastructure.
- **Attacker environment:** Kali Linux, Nmap 7.99, Gobuster 3.8.2, curl 8.22.0, Git, Python 3, and a custom Python reverse-shell listener.
- **Target notation:** `TARGET_IP`, `TARGET_HOST`, and `ATTACKER_IP` are placeholders for the laboratory addresses.
- **Evidence handling:** Credential values, tokens, flag contents, and real lab addresses are omitted. No screenshots are included.

This writeup documents authorized laboratory activity only.

## Recon

### Nmap Enumeration

**Objective:** Identify reachable services and establish the initial attack surface.

The first scan used Nmap service detection:

```bash
nmap TARGET_IP -sV
```

**Relevant output**

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.16
80/tcp open  http    nginx 1.24.0 (Ubuntu)

Service Info: OS: Linux
```

Nmap also reported 567 closed and 431 filtered TCP ports among the default 1,000 scanned ports.

| Port / Protocol | Service | Version / Evidence | Interpretation / Next Step |
| --- | --- | --- | --- |
| 22/tcp | SSH | OpenSSH 9.6p1 Ubuntu | Potential authenticated entry point |
| 80/tcp | HTTP | nginx 1.24.0 Ubuntu | Primary target for web enumeration |

**Analysis:** The web service was prioritized because it presented a broader application attack surface. The SSH service remained relevant as a potential entry point if valid credentials were discovered.

The scan did not cover all 65,535 TCP ports, so its results should not be interpreted as a complete service inventory.

### Web Enumeration

**Objective:** Discover additional applications through HTTP virtual host enumeration.

The following Gobuster command was executed:

```bash
gobuster vhost \
  -u http://nexus.htb/ \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  --append-domain
```

**Relevant results**

```text
billing.nexus.htb  Status: 302
  -> http://billing.nexus.htb/admin/login

git.nexus.htb      Status: 200
```

| Virtual Host | Observation | Next Step |
| --- | --- | --- |
| billing.nexus.htb | Redirect to an administrative login | Inspect the CRM application |
| git.nexus.htb | Accessible Git web interface | Inspect exposed repositories |

The enumeration was stopped after 346 of 4,989 candidates, so these findings do not represent a complete virtual host inventory.

**Analysis:** Both virtual hosts offered promising investigation paths. The Git service potentially exposed source code or configuration information, while the billing application exposed an administrative interface.

#### Git Repository Inspection

The accessible Git interface exposed a repository containing:

```text
.env
docker-compose.yml
documents
```

The repository displayed two commits and a single `main` branch.

The `.env` file and Docker Compose configuration were particularly relevant because they could reveal application deployment details, internal service names, and credentials.

Further investigation identified configuration information referencing credentials on the target system.

**Finding:** Sensitive application configuration was accessible through the Git service.

**Impact:** Disclosure of deployment details and credentials that could potentially be reused elsewhere.

The exposed information became relevant during the later user escalation phase.

### Other Service Enumeration

No additional network service enumeration was documented beyond the initial Nmap results and the identified web applications.

The Gitea service became important later because its authentication system and repository management functionality were accessible using credentials recovered during the assessment.

## Initial Access

**Finding:** Administrative Account Creation Through an Exposed Krayin Installer

**Preconditions:** Network access to the billing virtual host and its leftover installation endpoint.

**Reasoning:** Investigation of the billing application revealed an active debugging interface that referenced an installation endpoint. The installer remained accessible after the application had already been deployed.

### Reproduction

A preliminary request was sent to the installer endpoint:

```bash
curl -i \
  --resolve billing.nexus.htb:80:TARGET_IP \
  -H 'X-Requested-With: XMLHttpRequest' \
  -H 'Accept: application/json' \
  -H 'Content-Type: application/json' \
  --data '{}' \
  http://billing.nexus.htb/install/api/admin-config-setup
```

The server returned:

```text
HTTP/1.1 200 OK
Server: nginx/1.24.0 (Ubuntu)

1
```

The HTTP response alone did not prove that administrative account creation was possible. A subsequent request containing new administrator account details was accepted, and the newly created account successfully authenticated to the Krayin dashboard.

**Verification:** Successful login to the CRM dashboard with the newly created administrator account.

**Outcome:** Administrative control of the Krayin web application was obtained without access to a previously authorized administrator account.

### Authenticated PHP Attachment Upload

**Finding:** Server-Side PHP Execution Through a Web-Accessible Mail Attachment

**Preconditions:** Authenticated access to the Krayin administrative dashboard.

After obtaining application administrator access, the mail composition functionality was investigated.

A minimal PHP test file was prepared:

```php
<?php echo 'HTB_OK'; ?>
```

The file was attached to a message through the application's mail interface.

The application accepted the attachment and returned metadata indicating that the PHP file had been stored beneath a web-accessible path:

```text
/storage/emails/1/probe.php
```

The attachment metadata identified the content type as:

```text
text/x-php
```

Accessing the uploaded file through the web server produced:

```text
HTB_OK
```

**Analysis:** The observed output confirmed that the uploaded PHP test code was executed by the server rather than merely downloaded or displayed as source code.

This established a server-side code execution primitive through the authenticated attachment functionality.

### Reverse Shell

A PHP reverse-shell payload was subsequently uploaded using the same attachment mechanism.

The original payload used a TCP socket to connect back to a laboratory listener and launched an interactive shell process with its standard input, output, and error streams connected to that socket.

The callback address is represented below as `ATTACKER_IP`.

A custom Python listener was started on the attacking machine:

```bash
python rshell.py listen
```

After triggering the uploaded PHP file, the listener received an incoming connection from the target.

**Relevant output**

```text
connect to [ATTACKER_IP] from [TARGET_IP]
/bin/sh: 0: can't access tty; job control turned off
$
```

**Verification**

```bash
id
```

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

**Outcome:** A reverse shell was established as `www-data`, confirming operating-system command execution with the web application's service account privileges.

No interactive root shell was obtained at this stage.

## User Escalation

**Finding:** Credential Reuse Between Application Configuration and Linux/Gitea Accounts

**Starting context:** `www-data` (UID 33).

**Evidence and reasoning:** Earlier Git repository enumeration exposed application configuration details. Further investigation of the deployed `.env` configuration revealed a password associated with the `krayin` application user.

The possibility that this password had been reused for the Linux account `jones` was tested.

### Reproduction

The recovered password was used to authenticate over SSH:

```bash
ssh jones@TARGET_IP
```

The password itself is intentionally omitted.

**Verification**

```text
jones@nexus:~$
```

The successful SSH session confirmed that the recovered credential was accepted for the `jones` account.

The same credential also successfully authenticated to the Gitea service as `jones`.

**Outcome:** Access progressed from the restricted `www-data` context to an authenticated Linux session as `jones`, with additional access to Gitea repositories.

This demonstrated password reuse across application and infrastructure components.

## Root Escalation

**Finding:** Path Traversal in a Privileged Gitea Template Synchronization Job

**Starting context:** Authenticated Linux and Gitea access as `jones`.

### Discovery

During local enumeration, a Python-based repository synchronization process was identified.

The job was observed to run with root privileges on an approximately 60-second schedule.

Relevant configuration values included:

```python
GITEA_URL = "http://localhost:3000"
REPO_ROOT = "/var/lib/gitea/data/gitea-repositories"
STAGING_DIR = "/home/git/template-staging"
LOG_FILE = "/var/log/template-sync.log"
```

The implementation constructed repository paths using values derived from repository metadata:

```python
bare_path = os.path.join(
    REPO_ROOT,
    owner,
    "%s.git" % name
)
```

Another portion of the synchronization routine wrote retrieved repository content to a target file:

```python
with open(target, 'wb') as f:
    f.write(cat_result.stdout)
```

The script also modified file permissions according to the Git tree mode.

**Analysis:** The synchronization process crossed a critical trust boundary: content controlled through a lower-privileged Git account was processed by a root-executed filesystem synchronization task.

The security of this operation depended on whether repository paths and Git tree entries were validated before being converted into filesystem destinations.

### Reproduction

The `jones` account was able to create and modify repositories in Gitea.

A test repository was created, and a custom Git tree was constructed programmatically using Git plumbing commands.

The test used:

- `git hash-object` to create a blob containing the test payload.
- `git mktree` to construct Git tree objects.
- `git commit-tree` to create a commit from the crafted tree.
- `git push` to submit the resulting commit to the Gitea repository.

The crafted tree included parent-directory traversal components designed to escape the intended staging directory.

The privileged synchronization task subsequently processed the repository.

**Relevant log output**

```text
Found 1 template repo(s)
Syncing template: jones/test
synced: ../../../../../etc/cron.d/nexus-root
Template sync complete
```

The same behavior was observed in consecutive synchronization cycles.

**Verification:** The synchronization log showed that the crafted repository content was processed using a traversal path targeting a file outside the intended staging directory. The later flag retrieval confirmed the effect of the scheduled payload.

### Privileged Cron Execution

The payload targeted a cron configuration under:

```text
/etc/cron.d/nexus-root
```

Its purpose was to execute a command under the root account and write the contents of a root-protected flag file to a temporary location readable by the testing account.

After the synchronization and scheduled execution, the temporary output file contained the expected root flag.

The flag value is intentionally omitted.

**Outcome:** The crafted Git tree caused a privileged synchronization process to write outside its intended staging location. Cron then ran the payload as root and produced the root flag. No interactive root shell was established or required.

### Lab Cleanup

The author removed the temporary proof file during testing. A later missing-file message therefore reflected that removal, not a failed escalation. After both flags were submitted, the author's HTB instance was offline and no longer reachable; the HTB website showed it as Offline. That observation concerns the individual lab instance; the machine's retired status is documented separately above.

## Remediation

| Finding | Impact | Recommended Fix | Validation |
| --- | --- | --- | --- |
| Exposed installation endpoint | Unauthorized administrator account creation | Disable or remove the installer after deployment; enforce installation-state checks | Verify installer routes cannot create accounts after setup |
| Active debugging interface | Disclosure of internal application behavior | Disable production debugging and restrict diagnostic endpoints | Confirm debugging information is unavailable to unauthenticated users |
| Executable PHP attachments | Server-side code execution | Store uploads outside the web root, restrict file types, and disable script execution in upload directories | Verify PHP attachments cannot execute |
| Exposed configuration credentials | Disclosure of sensitive secrets | Remove secrets from repositories, rotate exposed credentials, and use a secrets-management solution | Review repository history and validate credential rotation |
| Credential reuse | Unauthorized access to multiple services | Use unique credentials for each service and account | Confirm compromised application credentials cannot authenticate to SSH or Gitea |
| Privileged template synchronization | Arbitrary privileged filesystem modification | Validate Git tree paths, reject traversal components, constrain filesystem destinations, and avoid running the synchronization process as root | Test crafted repository trees and confirm writes cannot escape the staging directory |
| Root cron configuration abuse | Root-level command execution | Restrict cron configuration writes to trusted administrators and monitor changes | Confirm unprivileged repository activity cannot modify cron files |

**Detection opportunities**

- Monitor unexpected requests to application installation endpoints.
- Alert on new administrator account creation outside approved administrative workflows.
- Detect executable file uploads to web-accessible directories.
- Monitor unusual outbound connections initiated by the web service account.
- Audit access to sensitive configuration files.
- Monitor SSH authentication and Gitea repository activity.
- Alert on unexpected changes to `/etc/cron.d/`.
- Review synchronization logs for paths containing `../` or destinations outside the approved staging directory.

## Lessons Learned

**Technical insight:** Nexus demonstrated how several independent weaknesses can be chained into a complete compromise. The initial web application weaknesses enabled code execution, exposed credentials allowed movement to a Linux account, and unsafe processing of attacker-controlled Git data by a privileged job enabled root-level execution.

**What did not work:** No substantive failed exploitation attempt was documented. An initial empty request to the Krayin installer established endpoint behavior but did not independently prove account creation. The missing temporary proof file observed later resulted from cleanup rather than an unsuccessful escalation.

**Process improvement:** Separating observations from hypotheses was particularly important. HTTP 200 responses, accessible configuration files, and successful repository synchronization were individually insufficient to prove exploitation until their effects were verified.

**Further practice:** Investigate Git object internals, path normalization, filesystem confinement, and safe design of privileged automation jobs.

**Assistance used:** AI assistance was used to organize the collected evidence, explain findings, and prepare the technical writeup. The machine enumeration, exploitation, and verification steps were performed by the author.

## References

- [Nexus machine page](https://app.hackthebox.com/machines/948)
- [Official HTB retirement record for Nexus](https://labs.hackthebox.com/achievement/machine/3484863/948)
- [HTB Streaming / Writeups / Walkthrough Guidelines](https://help.hackthebox.com/en/articles/5188925-streaming-writeups-walkthrough-guidelines)
- [Nmap Documentation](https://nmap.org/docs.html)
- [Gobuster](https://github.com/OJ/gobuster)
- [Git Documentation](https://git-scm.com/docs)
- [PHP: fsockopen](https://www.php.net/manual/en/function.fsockopen.php)
- [PHP: proc_open](https://www.php.net/manual/en/function.proc-open.php)
