# {{MACHINE_NAME}} — Hack The Box

> **Writeup status:** {{DRAFT / COMPLETE}} · **Publication eligibility:** {{UNVERIFIED / VERIFIED RETIRED}}
>
> Publish machine solutions only after confirming that HTB permits publication. Active machine writeups, hints, and spoilers must stay private. An expired season does not establish retirement. Never include flags, passwords, hashes of credentials, tokens, private keys, VPN files, or personal information.

## Overview

| Field | Details |
| --- | --- |
| Machine | {{MACHINE_NAME}} |
| Platform | Hack The Box |
| Operating system | {{Linux / Windows}} |
| Difficulty | {{Easy / Medium / Hard / Insane}} |
| Machine page | {{HTB_MACHINE_URL}} |
| Author | [Timow2025](https://github.com/Timow2025) |
| Completed on | {{YYYY-MM-DD}} |
| Published on | {{YYYY-MM-DD}} |
| Retirement verified on | {{YYYY-MM-DD — source: machine page}} |
| Key techniques | {{TECHNIQUE_1}}, {{TECHNIQUE_2}}, {{TECHNIQUE_3}} |

{{Write two to four sentences describing the exposed attack surface, the main weaknesses, and the final outcome. Do not claim steps or results that you did not verify.}}

**Attack path**

```text
{{Exposed service}} → {{Initial foothold}} → {{User context}} → {{Root / Administrator / SYSTEM}}
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

{{Describe the HTB lab assignment, the authorized target, and when the exercise took place. Confirm that the documented activity stayed within the assigned lab scope.}}

- **In scope:** {{ASSIGNED_MACHINE_AND_SERVICES}}
- **Out of scope:** {{OTHER_SYSTEMS_AND_EXCLUDED_ACTIVITIES}}
- **Environment:** {{ATTACKER_OS_AND_RELEVANT_TOOL_VERSIONS}}
- **Target notation:** `TARGET_IP`, `TARGET_HOST`, and `ATTACKER_IP` represent lab addresses; replace them locally when running commands.
- **Evidence handling:** {{Explain which identifiers and sensitive values were removed or replaced with consistent labels such as [REDACTED_PASSWORD_1].}}

This writeup documents an authorized lab exercise for educational purposes. It does not grant permission to test other systems.

## Recon

### Nmap Enumeration

**Objective:** {{What did you need to learn about the target?}}

```bash
# Paste the exact Nmap command(s) used, with sanitized target values.
# Include relevant options and the reason for using them in the explanation below.
```

```text
{{PASTE_RELEVANT_SANITIZED_NMAP_OUTPUT}}
```

| Port / Protocol | Service | Version / Evidence | Interpretation / Next Step |
| --- | --- | --- | --- |
| {{PORT/PROTO}} | {{SERVICE}} | {{OBSERVED_VERSION_OR_UNKNOWN}} | {{WHY_THIS_MATTERS}} |

**Analysis:** {{Separate confirmed observations from version-based guesses. Explain which service you prioritized and why. Record scan limitations if relevant.}}

### Web Enumeration

{{If no web service exists, retain this subsection and state “Not applicable — no web service identified.”}}

- **Entry points:** {{SCHEME_HOST_PORT_AND_VIRTUAL_HOSTS}}
- **Application / technology:** {{OBSERVED_TECHNOLOGY_AND_EVIDENCE}}
- **Interesting paths or functionality:** {{PATHS_ENDPOINTS_AND_FEATURES}}
- **Authentication / access controls:** {{OBSERVATIONS_WITHOUT_SECRETS}}

```bash
# Paste sanitized web enumeration commands, if used.
# Record the wordlist and relevant filtering options for reproducibility.
```

```text
{{PASTE_RELEVANT_SANITIZED_HTTP_OR_ENUMERATION_OUTPUT}}
```

**Analysis:** {{Explain what the response or behavior revealed and which hypothesis you tested next.}}

<!-- Add an image only after creating the referenced file. Example:
![Web enumeration showing the relevant application behavior](images/01-web-enumeration.png)
*Figure 1 — Explain what the screenshot proves; redact sensitive values before saving.*
-->

### Other Service Enumeration

{{Document relevant SMB, LDAP, DNS, FTP, or other enumeration. Use command → evidence → interpretation. If unnecessary, state “Not applicable” and explain briefly.}}

## Initial Access

**Finding:** {{NAME_OF_WEAKNESS}}

**Preconditions:** {{ACCESS_OR_CONFIGURATION_REQUIRED}}

**Reasoning:** {{Connect recon evidence to the chosen approach. Include a CVE only if applicability was verified.}}

### Reproduction

1. {{FIRST_ACTION_AND_PURPOSE}}
2. {{SECOND_ACTION_AND_PURPOSE}}
3. {{ACTION_THAT_ESTABLISHED_THE_FOOTHOLD}}

```bash
# Paste sanitized commands used to obtain initial access.
# Use descriptive placeholders for sensitive values.
```

```text
{{PASTE_MINIMAL_SANITIZED_RESULT}}
```

**Verification:** {{Record the resulting identity, host context, and access level. Use identity output or another non-secret proof; do not show flag contents.}}

**Outcome:** {{Describe the foothold and any constraints.}}

## User Escalation

{{Use this section for moving from the initial foothold to the intended user account, including lateral movement. If initial access already provided that account, state “Not applicable — initial access already established the required user context,” and reference the evidence above.}}

**Starting context:** {{INITIAL_ACCOUNT_AND_PRIVILEGES}}

**Finding and evidence:** {{OBSERVED_WEAKNESS_AND_WHY_IT_ENABLED_THE_TRANSITION}}

### Reproduction

1. {{ENUMERATION_ACTION_AND_RESULT}}
2. {{ACTION_TO_REACH_THE_USER_CONTEXT}}

```bash
# Paste sanitized commands, or replace the fence language with powershell as needed.
```

```text
{{PASTE_SANITIZED_IDENTITY_OR_ACCESS_VERIFICATION}}
```

**Outcome:** {{RESULTING_USER_AND_ACCESS_LEVEL — no flag values}}

## Root Escalation

{{For Windows, document escalation to Administrator or SYSTEM and identify which context was actually obtained. Do not treat these identities as interchangeable.}}

**Starting context:** {{CURRENT_USER_AND_PRIVILEGES}}

**Finding:** {{MISCONFIGURATION_OR_VULNERABILITY}}

**Evidence and reasoning:** {{Show what established exploitability and explain why the technique worked.}}

### Reproduction

1. {{RELEVANT_ENUMERATION_AND_FINDING}}
2. {{ESCALATION_ACTION_AND_PURPOSE}}
3. {{VERIFICATION_OF_ELEVATED_CONTEXT}}

```bash
# Paste sanitized escalation and identity verification commands.
```

```text
{{PASTE_SANITIZED_PROOF_OF_ELEVATED_IDENTITY — omit flags and secrets}}
```

**Outcome:** {{EXACT_FINAL_IDENTITY_AND_PRIVILEGES}}

**Lab cleanup:** {{List temporary artifacts removed or the instance reset performed, if applicable.}}

## Remediation

{{Map each recommendation to a weakness demonstrated above. Prefer specific configuration, permission, or patch changes over generic advice.}}

| Finding | Impact | Recommended Fix | Validation |
| --- | --- | --- | --- |
| {{INITIAL_ACCESS_WEAKNESS}} | {{DEMONSTRATED_IMPACT}} | {{SPECIFIC_FIX}} | {{HOW_TO_VERIFY_THE_FIX}} |
| {{USER_TRANSITION_WEAKNESS_OR_NA}} | {{IMPACT_OR_NA}} | {{FIX_OR_NA}} | {{VALIDATION_OR_NA}} |
| {{PRIVILEGE_ESCALATION_WEAKNESS}} | {{DEMONSTRATED_IMPACT}} | {{SPECIFIC_FIX}} | {{HOW_TO_VERIFY_THE_FIX}} |

**Detection opportunities:** {{Relevant logs, suspicious events, or monitoring ideas tied to this attack path.}}

## Lessons Learned

- **Technical insight:** {{WHAT_YOU_LEARNED_AND_WHY_IT_MATTERS}}
- **What did not work:** {{A_FAILED_APPROACH_AND_THE_EVIDENCE_THAT_RULED_IT_OUT}}
- **Process improvement:** {{WHAT_YOU_WOULD_CHANGE_NEXT_TIME}}
- **Further practice:** {{A_SPECIFIC_SKILL_TO_REVISIT}}
- **Assistance used:** {{Hints, walkthroughs, or other assistance and the steps they informed; write “None” if completed independently.}}

## References

{{Replace these placeholders with descriptive Markdown links. Cite sources near the relevant claims as well. Prefer vendor documentation and original advisories.}}

- {{HTB_MACHINE_PAGE_LINK}}
- {{RELEVANT_TOOL_DOCUMENTATION_LINKS}}
- {{VENDOR_ADVISORY_OR_CVE_LINK_IF_APPLICABLE}}
- {{CREDIT_FOR_ANY_EXTERNAL_TECHNIQUE_CODE_OR_WALKTHROUGH}}
- [HTB publication guidelines](https://help.hackthebox.com/en/articles/5188925-streaming-writeups-walkthrough-guidelines)

<!-- Before publication: remove unused placeholders and authoring comments, verify image paths,
confirm publication eligibility again, and inspect the full Git diff and images for secrets. -->
