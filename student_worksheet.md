# Northbridge Manufacturing CTI Exercise — Student Worksheet (BEC / Email ATO)

**Case:** NWB-2026-021 · **Data window:** 16 Jun – 18 Sep 2026 (all timestamps **UTC**)
**Artifacts to analyze:**

| File | Source system | About |
|---|---|---|
| `m365_audit.log` | Microsoft 365 (Azure AD sign-ins + Unified Audit) | One JSON record per line (`RecordType` 15 = sign-in, 1 = Exchange admin / rule ops, 4 = mailbox access like read/send) |
| `exchange_msgtrack.log` | Exchange Server 2019 Message Tracking | One CSV record per mailflow event (RECEIVE / SUBMIT / DELIVER / REDIRECT / DROP / FAIL); first line is `#Fields:` header |
| `corp_proxy.log` | Corporate web proxy (Squid-style) | One request per line from internal workstations (GET + `CONNECT` for HTTPS) |

**Organization (fictional):** Northbridge Manufacturing Ltd (`northbridge-mfg.com`), ~27 staff,
Manchester UK, single M365 E3 tenant. The M365 tenant is the mail server, so these three logs
tell the whole story of this incident.

---

## How to work

- `m365_audit.log` is **JSONL** — one JSON object per line. Grep for a key and read the line
  (e.g. `grep "s.okafor@" m365_audit.log`).
- `exchange_msgtrack.log` is **CSV** — parse by column; the `#Fields:` header names them. Useful
  columns: `timestamp`, `event-id`, `message-subject`, `sender-address`, `recipient-address`,
  `original-client-ip`, `message-info`.
- Every claim needs an **exact quoted line** plus its source file — same rule as always.
- The real incident is small. Most of what you see is **noise**. Build the baseline (Part 2)
  *before* hunting (Part 3), or you will flag 50 false alarms and miss the real fraud.

Useful commands:

```
# JSONL: who signed in as/including the CFO?
grep -i "okafor" m365_audit.log | grep -i "sign\|login" | head
# who connects from the most locations? sign-ins by city for one account:
grep -o '"LocationCity":"[^"]*"' m365_audit.log | sort | uniq -c | sort -rn
# message tracking: what did the AP clerk actually receive the week of 3 Aug?
grep "2026-08-" exchange_msgtrack.log | grep "m.avery@" | head
# proxy: biggest URL categories by protocol
awk '{print $7}' corp_proxy.log | awk -F'/' '{print $3}' | sort | uniq -c | sort -rn
```

---

## Part 1 — Know your logs (15 min)

1. For each file describe: producer (product), line format (all fields), date range, line count.
2. `m365_audit.log`: list the **distinct `RecordType` values** present and what kind of event each
   means. Count sign-ins vs mailbox-access records.
3. `exchange_msgtrack.log`: list the distinct `event-id` values (RECEIVE / SUBMIT / DELIVER /
   REDIRECT / AGENT / DROP / FAIL…) and explain in one line what each means in the mail flow.
4. `corp_proxy.log`: top 5 destination hosts, and how many requests were `CONNECT` (HTTPS tunnel)
   versus `GET`. What do the `TCP_DENIED/403` lines have in common?

---

## Part 2 — Establish the baseline (45 min)

5. **The CFO’s normal shape.** Using only June–July data, characterise
   `s.okafor@northbridge-mfg.com`:
   - which source IP, which User-Agent, which device names,
   - at what UTC hours she signs in on weekdays,
   - what her `Mfa` field says **every single time** (this one field is the vulnerability),
   - does she ever use PowerShell, or apps other than Web/Outlook?
6. MFA roll-out. Pick 3 colleagues who are NOT the CFO (e.g. `m.avery`, `j.park`, `a.rossi`),
   find their typical `Mfa` value and compare against the CFO’s. What can you infer about MFA
   enforcement as of Sep?
7. Identify the recurring **compliance** mechanism in the mail infrastructure (search
   `exchange_msgtrack.log` for anything repeating on many external mails, and
   `m365_audit.log` for a transport rule). What is it doing, and since when? Remark: it is the
   ONLY mailbox-level "copy/forward" behaviour visible in June–July.
8. Proxy rhythm: when does browsing concentrate (UTC hours)? Who are the heaviest users?
   Which domains are deliberately blocked, and does anyone even try?
9. **Decoy #1 — the traveller.** `r.garcia@northbridge-mfg.com` signs in from two unusual cities
   on 21 Jul and 30 Jul. An "impossible travel" alert would fire. Determine whether this is real
   malice or a genuine business trip (check device names, MFA status, and whether the forbidden
   follow-on behaviour appears for that account *after* those days).
10. **Decoy #2 — the night owl.** `m.avery@northbridge-mfg.com` signs in around 23:00 UTC on
    14 Jul and 4 Aug. Is an "out-of-hours sign-in" alert right to fire for her? What does her MFA
    show? (Note: AP staff process wires late; check her normal evening pattern in June/July.)
11. **Decoy #3 — the DLP drop.** Find the largest message in the whole tracking log
    (sort by `total-bytes`). What happened to it (`event-id`, `source-context`)? Who was the
    sender, what was the payload, and why is this engineer NOT the incident?

---

## Part 3 — Find the incident (3–4 hours)

### 3a. The phishing campaign

12. On **27 Jul** a new external sender appears in the message tracking log, mailing several staff.
    - Quote the sender address and the domain behind it. Count how many of your staff received it.
    - What was the subject? Rate the sender domain for "phishiness" (spoof-a-sibling, TLD, age).
13. On **28 Jul** those same users reach that domain through the **corporate proxy**.
    - Quote the proxy lines (URL, and the `POST …/auth/…/consent` that follows the view).
    - Which workstation IPs clicked? Map them to users via the audit log.
    - Reading the URLs: what does `/auth/…/consent` do on a page like this? Why would an attacker
      want a victim to *both* visit it and POST credentials/session tokens on it?

### 3b. Account takeover — replay of stolen credentials

14. On **1 Aug 01:58–02:01 UTC** look at `s.okafor@` in the audit log.
    - Two `UserLoginFailed` then one `UserLoggedIn` **same day, same IP**. Quote all three.
    - Where does that IP geo-locate? How plausible is it that the CFO is logging in from there?
    - Which security control *should* have stopped even successful replays here — and why is it
      shown as `Mfa: Not required`?
15. Find **every** sign-in for the CFO during **2–11 Aug** (hint: she is on holiday — normal UK
    sign-ins should be **zero**). List each timestamp, IP, city, client app. Which flight risks
    are flagged (`RiskEvent`)?

### 3c. Read & select

16. Between **02:20 and 02:50 UTC on 3 Aug** the mailbox-access records show a burst of `ReadMail`
    operations. List the **subjects** of every mail read. Why are these particular subjects the
    perfect pick-list for a fraudster? (Think: payment run, IBAN, mandate, supplier approvals…)

### 3d. Persistence — the invisible forwarding rule

17. On **6 Aug 09:12:44 UTC** a single `RecordType=1` audit record appears with
    `Operation = New-InboxRule`.
    - Quote the whole record.
    - Decode `RuleParameters`: name, forward target, MarkAsRead, MoveToFolder, and the Filter.
    - Explain how each parameter evades detection: what does `MarkAsRead` do to the CFO's phone,
      what does `MoveToFolder` do, and why forward **+ delete-or-hide** is the classic BEC pattern?
18. Cross-file: find in `exchange_msgtrack.log` the mails that carry `message-info` containing
    `rule:Q3 Processing`. These are the rule’s **copies**. Which recipient never belongs on Northbridge
    mail? (This recipient = the attacker’s burn address: IOC #1.)
19. Note also the `REDIRECT` event on the fake-invoice message (`recipient added by rule`). What
    does `recipient-count` say on those rows?

### 3e. The fraud — payment instructions

20. On **6 Aug 15:02:17 UTC** a message is submitted FROM `s.okafor@…`.
    - In `exchange_msgtrack.log`, quote the `SUBMIT` row. Now look at the column
      `original-client-ip` — it is **not** the CFO’s Manchester IP. What does its value tell you
      about who clicked "Send"?
    - Compare with a normal OWA submission in the same log (any `SOURCE:OWA` row you can find in
      July) — what does *their* `original-client-ip` contain? This difference is the smoking gun.
21. Same minute in `m365_audit.log`: two records, `Operation = SendAs` and `Operation = SendMessage`.
    - Quote both. Who claims to be acting (`UserId`/`MailboxOwnerUPN`) and from which IP?
    - Why does **SendAs** appear instead of a normal "send"? What does it mean that the mailbox
      owner "sent" but the sender is replayed creds?
22. Destinations of the fake invoice: quote the DELIVER rows. The message went to (a) accounts
    payable clerk, (b) an external supplier contact, and (c) the **burn address** (rule copy).
    - What does the bank-facing recipient `payments@interstate-components.com` imply about who was
      meant to act on it silently?
23. **Follow-ups.** On **8 Aug 08:40** and **9 Aug 10:15** more mails leave the CFO account
    (subjects). Then on 9 Aug 14:10 UTC `m.avery@…` replies "…". Quote those SUBMIT/DELIVER rows
    and state, in one sentence each, what stage of wire fraud this is (urging → handling).

### 3f. Discovery & response

24. On **11 Aug 06:55 UTC** the CFO signs in from her normal Manchester IP again. What changed on
    her account that morning (MFA now?), and why?
25. Between **11:24 and 11:25 UTC** IT takes three actions: `SetUserPassword`,
    `RevokeAllUserSessions`, `Remove-InboxRule` (rule `Q3 Processing`). Quote each. What does
    `Revoke` kill that a password reset alone doesn’t?
26. After 11 Aug, is there ANY sign of the attacker? Run the CFO-IP/USA search over **12 Aug – 18
    Sep** and report honestly what you find. (An answer like "no further activity — containment
    worked" is valid and important.)

---

## Part 4 — Separate incident from noise, for the record (45 min)

27. Build a two-column table — **incident vs not-incident** — of everything you investigated in
    Parts 2–3, with a one-line justification each. You must explicitly classify:
    - r.garcia’s travel (Q9), m.avery’s late login (Q10), t.hughes’ zip (Q11),
    - the pre-existing compliance rule (Q7),
    - the 6 staff who clicked the phish on Jul 28 — **incident or not?** (Only one mailbox was
      actually owned. Which evidence separates clickers from the owner?)
28. Explain why the corporate proxy contains **no** traffic from the attacker (Q13 vs Q15–23):
    what does that absence prove about where the take-over happened?
29. Rank these four weaknesses by how much they enabled the fraud (1 = most):
    (a) no MFA on the CFO, (b) traded credentials via phish, (c) no rule-change monitoring,
    (d) payment process lacking independent verification. Justify.

---

## Part 5 — Intelligence product (write it up)

30. **IOC table** — Type (IP / domain / e-mail / rule / file), Value, Role, Evidence (file + line).
    Include at least: the two operator IPs, the phish domain, the burn mailbox, the forwarding
    rule, the fake-invoice subjects.
31. **Kill-chain timeline.**
    Grid: Stage → Date → Time (UTC) → Evidence (file + quoted line). Cover: phishing delivery →
    click/credential theft → credential stuffing → ATO → mailbox read → persistence rule → fake
    remittance → follow-ups → victim reply → discovery → containment.
32. **Attribution** (3–5 sentences): criminal or not? opportunistic or targeted? confident or not —
    and why the "no-MFA CFO, invoice eavesdrop, burn Gmail" pattern smells like a **repeatable
    criminal playbook** rather than a state actor.
33. **Detection rules** — write at least one each (pseudo-SIEM + why it won't drown in noise):
    a. inbox rule created on a mailbox (`New-InboxRule`, any rule that forwards out + MarkAsRead),
    b. sign-in success for a user whose MFA is "Not required" from a new country,
    c. `SendAs` on a mailbox whose owner never uses that app,
    d. mailflow `original-client-ip` that differs from the owner’s normal IP on `SOURCE:OWA` rows,
    e. external mail carrying a **new burner** exact-recipient match to an open rule (correlation rule).
34. **Executive report** — one page: Summary / Impact (invoice #, amount range, data classes) /
    Recommended actions (contain → eradicate → recover → prevent) / Lessons (MFA everywhere,
    rule-monitoring, dual-approval wires, U2F for executives).
35. **SOC-2 style retrospective**: list the 5 root-cause fixes, each mapped to the kill-chain step
    it would have stopped.

---

## Grading rubric (what "good" looks like)

| Criteria | Weak | Strong |
|---|---|---|
| Evidence quoting | Paraphrases "the log shows Kiev" | Quotes exact JSON/CSV lines with `file` |
| Baseline first | Flags every 'not MFA' & late login | Builds June-July normal first |
| Multi-file correlation | One file only | Rule in audit ⇔ REDIRECT in tracking ⇔ burn recipient |
| Timezone discipline | Misses that all logs are UTC | States UTC everywhere, no impossible FT confusion |
| False positives | Reports traveller/night-owl/DLP as attackers | Explicitly clears them with evidence |
| IOC/report quality | Lists IPs | Typed IOC table + prioritized exec report |

**Submission:** one `incident_report.md` or PDF answering all 35 questions. The answer key is held
by the instructor.