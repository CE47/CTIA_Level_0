# Northbridge Manufacturing — BEC / Email Account Takeover CTI Exercise

A hands-on **Cyber Threat Intelligence (CTI) / Log Analysis** classroom exercise built around a
realistic **Business Email Compromise (BEC)** scenario.

You are a junior threat analyst at **Northbridge Manufacturing Ltd** (`northbridge-mfg.com`), a
~27-person manufacturer in Manchester. Management noticed the CFO's mailbox "acted strange" in
early August and handed you ~3 months of logs. Your job: explain exactly what happened, when, to
whom, why nobody noticed earlier, and how to stop it — **by correlating the evidence across three
different log sources, not by listening to rumors.**

> This is synthetic training data. All IPs, addresses, domains, mailboxes and invoice numbers are
> fictional. No real systems, users or funds are involved.

---

## Repository contents

```
├── README.md              ← you are here
├── guided_worksheet.md    ← ultra-guided version (zero-experience students)
├── student_worksheet.md   ← 35-question version (some prior experience)
├── m365_audit.log          M365 Unified Audit + Azure AD sign-ins (one JSON record per line)
├── exchange_msgtrack.log   Exchange 2019 Message Tracking (CSV, #Fields: header)
└── corp_proxy.log          Corporate web-proxy (Squid-style) access log
```

| File | Format | What it records |
|---|---|---|
| `m365_audit.log` | **JSONL** (one JSON object per line) | Sign-ins (`RecordType` 15), Exchange admin/rule ops (`RecordType` 1), mailbox reads/sends (`RecordType` 4) |
| `exchange_msgtrack.log` | **CSV** with a `#Fields:` header | Every mailflow event: RECEIVE, SUBMIT, DELIVER, REDIRECT, AGENT, DROP, FAIL |
| `corp_proxy.log` | **Squid extended log** | Staff web requests (GET) and HTTPS tunnels (CONNECT), including policy blocks |

The **answer key is intentionally not included** — it is held by the instructor GCN.

---

## Two versions of the exercise

This repository ships **two worksheets** covering the same scenario and evidence. Pick one per
student — do not give them both (they lead to the same target).

| Version | File | For students who… | Style |
|---|---|---|---|
| **Guided** | `guided_worksheet.md` | have **zero experience** — never read a log, know only a few Linux commands | Ultra step-by-step: it teaches `pwd`/`ls`/`grep` first, then runs every investigation step as a copy-paste command with "write your finding here" boxes. Intended to run from `~/Desktop/bec-email-cti` on Kali Linux. |
| **Standard** | `student_worksheet.md` | have **a bit of background** — comfortable with grep/basics, know roughly what a log line or an IP is | 35 questions in 5 parts; they hunt with their own commands and answer analytically. |

Both end with the same deliverable (an incident report), same logs, same story; the guided
version just adds scaffolding.

---

## How to use

1. Pick the matching worksheet above (`guided_worksheet.md` or `student_worksheet.md`).
2. Work through it in order — for the guided version, that means Sections 0→11; for the standard
   version, Parts 1→5.
3. Open the logs in any editor and grep freely; the whole exercise works with plain-text tools.
4. All timestamps are **UTC** (Manchester is UTC+1 in the summer). Keep one clock when correlating.

Suggested first steps:

```
# who signs in (and in what states)?
grep -o '"RecordType":[0-9]*' m365_audit.log | sort | uniq -c

# which reporter host dominates?
awk -F',' '{print $9}' exchange_msgtrack.log | sort | uniq -c | sort -rn | head

# what does the proxy actually block / allow?
awk '{print $4, $7}' corp_proxy.log | sort | uniq -c | sort -rn | head
```

---

## Learning objectives

- Read **JSONL (M365 audit)**, **CSV (Exchange message tracking)**, and **Squid proxy** formats
- Build a per-user **baseline** before hunting (hours, IPs, apps, MFA state)
- Toggle **noise/false positives** vs real malice (traveller, night-owl, DLP drop, compliance rule)
- Follow a full **BEC kill chain**: phish → AITM credential theft → credential stuffing → account
  takeover → mailbox read → invisible inbox **forwarding rule** → fake remittance → wire fraud
- Detect **how one field (`original-client-ip`) and one audit op (`New-InboxRule`) tie the whole
  chain together**
- Write SIEM-ready detection rules and an executive incident report

---

## Getting help during the exercise

- Searching for the *file formats* (e.g. "M365 unified audit JSONL", "Exchange message tracking
  fields", "Squid access.log format") is expected and encouraged.
- Do **not** search the internet for the fictional IPs/domains/mailboxes in these logs — they are
  made up; nothing will be found, the evidence is all on disk.
- Stuck? Re-read Part 2 (baseline). Every "weird" thing in Part 3 is only weird compared to the
  June–July shape of that same user.

---

## Disclaimer & license

Synthetic data for education. All trademarks belong to their owners.
