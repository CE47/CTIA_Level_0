# Northbridge Manufacturing — GUIDED WORKSHEET for Absolute Beginners

**You have never read a log file. You know a few Linux commands. That's enough.**

This worksheet walks you through an investigation **one small step at a time**. Do every step in
order. Copy each command exactly, press Enter, look at the output, and write your finding in the
"Write down" boxes. There are no wrong answers if you follow the steps — the log files contain
everything you need.

**Where are the files?** The folder is called `bec-email-cti` and it is on your **Desktop**.
Every command below assumes you are inside that folder. If a command fails with
`No such file or directory`, your current folder is wrong — read **Section 1** again.

> All times in these logs are **UTC**. UTC is a single world clock (London time in winter).
> Keep all your notes in UTC so your timeline lines up.

---

## Section 0 — Meet the company in the story

You are helping a small UK manufacturer called **Northbridge Manufacturing**
(`northbridge-mfg.com`, about 27 people, Manchester). They use Microsoft 365 for email.
In early August 2026 the CFO's email account did things she never did. Management wants answers.

You will investigate by reading **three** files. Each file is a different "camera":

| File | What it records | Like a… |
|---|---|---|
| `m365_audit.log` | Who logged into the email system, when, from where | Security camera at the front door |
| `exchange_msgtrack.log` | Every email that traveled through the mail server | Postal-sorting-office record |
| `corp_proxy.log` | Web browsing by office computers (what websites staff visited) | Camera on the office internet line |

---

## Section 1 — Your toolbox (10 minutes)

Open a terminal (the black box). You need four tools. Try each:

```
pwd
```
Shows **where you are** (`pwd` = "print working directory"). The answer is your home folder, like `/root`.

```
ls
```
Lists the files in the current folder ("ls" = list). You should see `Desktop`.

```
cd Desktop/bec-email-cti
```
**cd** = "change directory". Now you entered the exercise folder.
Run `ls` again — you should see the three log files and this worksheet.

Run `pwd` — it should now end with `/bec-email-cti`.

> **TROUBLE?** If `ls` doesn't show the three `.log` files, you are outside the folder.
> Type `cd` alone (takes you home), then `ls`, then `cd Desktop/bec-email-cti`.

**Your final tool — `grep`.** The star of the whole exercise.

`grep` searches *inside* files for text. Try this:

```
grep "rossi" m365_audit.log | head
```

You just told grep: *find every line that contains "rossi" and show me the first 10.*
The `|` symbol (called "pipe") sends the output of one command into the next.
`head` shows the first 10 lines. `head -5` shows 5.

**Write down (practice):** how many times does the word "okafor" appear in `m365_audit.log`?

```
grep -c "okafor" m365_audit.log
```
(`-c` = count. You'll get a big number — that's normal.)

---

## Section 2 — What am I looking at? (15 minutes)

You must be able to *recognize* the shape of each file before you hunt. Open the first 3 lines
of each:

```
head -3 m365_audit.log
head -3 exchange_msgtrack.log
head -3 corp_proxy.log
```

**File 1 — `m365_audit.log` (JSON format).**
Each line is one event written as a `name:value` list wrapped in curly braces `{ }`.

```
{"CreationTime":"2026-07-21T07:40:00Z","RecordType":15,"UserId":"r.garcia@northbridge-mfg.com", ...}
```

Learn the important names (call them **fields**):

| Field | It means |
|---|---|
| `CreationTime` | When it happened (UTC) |
| `RecordType` | What kind of event (15 = sign-in, 1 = mailbox/rule action, 4 = reading/sending mail) |
| `UserId` | The account taking the action |
| `ClientIP` | The internet address that did it |
| `LocationCity` | Where that internet address is located on a map |
| `Mfa` | Whether multi-factor (second password / phone confirmation) was used |

**File 2 — `exchange_msgtrack.log` (CSV format = "comma separated values").**
One email event per line. The first line `#Fields:` is the list of column names — print it:

```
head -1 exchange_msgtrack.log
```

Find these columns in any row:

| Column | It means |
|---|---|
| `timestamp` | When the mail event happened |
| `event-id` | What happened (SUBMIT = mail created / sent; DELIVER = mail delivered to a mailbox; REDIRECT = a rule changed it; DROP = mail blocked) |
| `sender-address` | Who sent it |
| `recipient-address` | Who received it |
| `message-subject` | The subject line |
| `original-client-ip` | The real address of the machine that clicked "Send" — **super important later** |

**File 3 — `corp_proxy.log` (plain web log).**
One line per web request. The important part is the URL near the middle and the workstation
address at the left (such as `10.20.1.12`). `CONNECT` means "open an encrypted tunnel to..." —
normal for office mail/Teams.

---

## Section 3 — Mission A: learn a normal user (20 minutes)

You can't spot a fake log-in until you know what a *real* one looks like.

The **CFO** is `s.okafor@northbridge-mfg.com`. Work with only her sign-ins
(a sign-in is a line whose `RecordType` is `15` — put the pattern in *single* quotes so the
shell doesn't eat the inner double quotes):

```
grep "okafor" m365_audit.log | grep '"RecordType":15' | head -20
```

Look at 10–20 lines. **Write down:**
- Her normal `ClientIP` (write the number you see over and over): ____________
- Her normal `LocationCity`: ____________
- Her normal `UserAgent` (the "Mozilla/... Edg" string): ____________
- The value of the `Mfa` field on her lines: ____________

Now pick any **other** person, for example `j.park@northbridge-mfg.com`:

```
grep "j.park" m365_audit.log | head -10
```

Compare their `Mfa` value to the CFO's. **Write down:** what big difference do you notice about
the CFO's `Mfa` compared to j.park's?

> This difference is the **weak point** that the whole attack used. Keep it in mind.

---

## Section 4 — Mission B: who is logging in from strange places? (20 minutes)

Good analysts ask: *does anyone sign in from a place they don't belong?*

Make a **list of all cities** that appear in the audit log, most common first:

```
grep -o '"LocationCity":"[^"]*"' m365_audit.log | sort | uniq -c | sort -rn
```

**Write down:**
- The city that appears thousands of times (the whole office's normal city): ____________
- Any cities that appear only a handful of times (these are rare = interesting).

Now check WHO logged in from each rare city. Example, for Paris:

```
grep "Paris, FR" m365_audit.log | grep -o '"UserId":"[^"]*"'
```
```
grep "Bucharest, RO" m365_audit.log | grep -o '"UserId":"[^"]*"'
```
```
grep "Kiev, UA" m365_audit.log | grep -o '"UserId":"[^"]*"'
```

**Write down:** which user logs in from `Paris, FR` and `Amsterdam, NL`?
Which user shows strange activity from `Bucharest, RO` and `Kiev, UA`?

> One of these users is innocent (a real business trip). The other one is the story.
> You'll separate them in Section 7.

---

## Section 5 — Mission C: find the email that started it (25 minutes)

An attack usually starts with a **phishing email** - a fake email that tricks someone.
Find which outside sender suddenly mailed several people inside the company.

Search the message-tracking file for the phish domain. The attacker mailed from a look-alike
domain that ends in `-ms.xyz` (a "do not trust this" zone just for this exercise):

```
grep "docview-ms.xyz" exchange_msgtrack.log | head -3
```

Confirm the column names first so you know where to look:
```
head -1 exchange_msgtrack.log
```

On the matched lines, find the column **`sender-address`**.
**Write down the exact sender address**: ____________
(It should end in `@docview-ms.xyz`.)

Now count how many of the company's staff received it around 27 July:

```
grep "2026-07-27" exchange_msgtrack.log | grep "docview-ms.xyz" | grep "DELIVER"
```

`DELIVER` rows show the mail actually landing in someone's inbox.
**Write down:** how many `DELIVER` rows do you count? ____________

Now the interesting part. The next day (28 July), office computers visited that same website
through the office **proxy**. Search the web log for the domain:

```
grep "docview-ms.xyz" corp_proxy.log
```

**Write down:**
- How many lines? (add `| wc -l` to count) ____________
- Notice the pattern: a `GET` (viewing a page) followed by a `POST` to an address ending in
  `/consent`. A "consent" page is where the website asks "log in with Microsoft?" — this kind of
  fake login page is called an **AITM phishing page**. It grabs your password *and* your session.

> What happened: on 27 July the attacker mailed the company; on 28 July people clicked.
> Only ONE of those clickers later shows the strange foreign log-ins from Section 4.

---

## Section 6 — Mission D: follow the attacker's days (40 minutes)

We know the story starts ~1 August. Let's look at 1–11 August for the CFO's account.
Print her sign-ins for that window (times UTC):

```
grep "okafor" m365_audit.log | grep '"RecordType":15' | grep "2026-08-0"
```

**Write down** (in a small table: date / time / ClientIP / city / Mfa):

1. **1 Aug** — what IP / city? Did log-ins fail before succeeding? (Look for `UserLoginFailed`
   lines followed by `UserLoggedIn`.) ____________
2. **3 Aug** — what IP / city / time? ____________
3. **Every day from 2 to 11 Aug** — is the CFO *ever* logging in from Manchester? (She is on
   holiday. If you see Manchester during those days, treat it as the attacker!) ____________

Now look at what happened to her **mailbox** on 3 August. Reading someone else's mail shows up as
a line whose `RecordType` is `4`, with an `Operation` of `ReadMail`. Count them for 3 August:

```
grep "okafor" m365_audit.log | grep "2026-08-03" | grep "ReadMail"
```

`head` the list and **write down 3 of the mail subjects** she "read":

- ____________
- ____________
- ____________

> Those subjects are financial (invoices, IBAN, payment approvals). The attacker is reading
> the company's payment pipeline to plan the fraud.

---

## Section 7 — Mission E: the invisible forwarding rule (30 minutes)

Attackers who get into a mailbox usually leave a silent **forwarding rule**: "copy every
important email to my own address, and mark it as read so the victim doesn't notice."

Rules being created show up as a line whose `RecordType` is `1`, with an `Operation` of
`New-InboxRule`. Search for it:

```
grep "New-InboxRule" m365_audit.log
```

**Write down:**
- The date/time: ____________
- The `RuleName` (write it exactly): ____________
- In `RuleParameters`, find the `ForwardTo` value — that's the **burn email address**
  (attacker's copy box). Write it: ____________
- Also in `RuleParameters`: `MarkAsRead` (true = hides mail from the victim's phone) and
  `MoveToFolder` (true = hides it from the inbox list). What folder does it move mail into? ____________

Now prove the rule **works**. Search the message-tracking file for that burn address:

```
grep "northbridge.inv.2476@gmail.com" exchange_msgtrack.log
```

Look at the **`event-id`** and **`message-subject`** columns.
**Write down the subjects** of the emails the attacker secretly received copies of:

- ____________
- ____________
- ____________

(The emails whose subjects are about invoice "INV-88912" are the fraud emails. We'll see
them next.)

---

## Section 8 — Mission F: the fake invoice (30 minutes)

On **6 August at ~15:02** someone "sent" an email FROM the CFO. Look at the SUBMIT row:

```
grep "2026-08-06" exchange_msgtrack.log | grep "SUBMIT" | grep "INV-88912"
```

Now compare two things on that one line:
1. **`sender-address`** — who does it claim sent the email? ____________
2. **`original-client-ip`** — the *real* address that clicked Send. Write it: ____________

**Now compare with a normal email.** Any normal employee sending from OWA has their own IP in
that column. Grab one innocent July example:

```
grep "2026-07-" exchange_msgtrack.log | grep "SUBMIT" | grep "SOURCE:OWA" | head -5
```

Write the `original-client-ip` values of those innocent rows: ____________

> Same "SUBMIT", same method — but the fake invoice's `original-client-ip` equals the attacker's
> IP from Section 6, NOT the CFO's IP. That is how a message-tracking log proves *who really
> pressed Send*.

Now check the audit log — the same minute shows the act of "sending as" someone else
(`SendAs` = pretending to be the mailbox owner):

```
grep "SendAs" m365_audit.log | tail -3
```

**Write down:** who did the system record as acting (`UserId`), and what `ClientIP` did it
record? ____________

---

## Section 9 — Mission G: discovery and cleanup (15 minutes)

On **11 August** the CFO returns from holiday and (probably) sees things are wrong.
Redo the Section 4 city-list and look for a return to normal:

```
grep "okafor" m365_audit.log | grep '"RecordType":15' | grep "2026-08-11"
```

**Write down:** which IP / city is the CFO using now (should be back to her normal Manchester IP)? ____________

IT steps in. Look for these three cleanup actions by the IT user `j.park`:

```
grep "j.park" m365_audit.log | grep "SetUserPassword\|RevokeAllUserSessions\|Remove-InboxRule"
```

**Write down the 3 actions IT took (and times):** ____________

Finally, confirm the attacker is gone: are there any more `Kiev` or `Bucharest` log-ins
**after** 11 August?

```
grep "Kiev, UA\|Bucharest, RO" m365_audit.log | grep "2026-08-1[2-9]\|2026-09-"
```

**Write down what you found (probably nothing — that's a good answer).** ____________

---

## Section 10 — Bonus missions: the decoys (20 minutes)

Real investigations contain **false alarms** that look scary but are nothing. Clear these three
so you can honestly say "I checked".

**A. The traveller.** `r.garcia` logged in from Paris and Amsterdam (Section 4). Show that this
is a genuine trip and not an attack: check her `Mfa` value and `Device` name on those days.

```
grep "r.garcia" m365_audit.log | grep "Paris, FR\|Amsterdam, NL"
```

**Write down:** MFA value = ____________ · Device = ____________
(The clinching check: her `Mfa` says `MFA succeeded` — she fully passed the second login
check on those days. A thief who only had her password could not do that. Compare with the
strange log-ins from Section 4: do *they* ever show `MFA succeeded`? No — that's the easiest
way to tell a real trip from an account takeover.)

**B. The night owl.** `m.avery` (accounts payable) signs in around 23:00 UTC. Check her MFA and
IP. Also recall: AP clerks process late payments — a 23:00 sign-in by her is normal.

```
grep "m.avery" m365_audit.log | grep "23:0"
```

**Write down:** MFA = ____________ · IP = ____________ (should be her normal one).

**C. The blocked file.** On 14 August an engineer `t.hughes` tried to email a large password
protected file. Find what the mail server did to it:

```
grep "hughes" exchange_msgtrack.log | grep "HX3"
```

Look at the `event-id` of the row with `DROP`. **Write down:** event = ____________
That's the mail server's Data Loss Prevention stopping a file it can't inspect — a safety
feature doing its job, NOT the attacker.

---

## Section 11 — Write your report (30 minutes)

Use the "My findings" template below and fill it in — this becomes your **incident report**.

### My findings

1. **Which normal user I studied and her normal IP / city / sign-in hours** (Section 3):
2. **The weakness I noticed in her MFA** (Section 3):
3. **Strange sign-in cities and which user had each** (Section 4):
4. **The phishing email**: sender, date, and how many staff received it (Section 5):
5. **The "consent" phishing page** did what? (Section 5):
6. **Attacker log-ins on 1 Aug and 3 Aug**: IPs and cities (Section 6):
7. **Subjects the attacker read on 3 Aug** (Section 6):
8. **The forwarding rule**: name, ForwardTo address, MarkAsRead, MoveToFolder (Section 7):
9. **The fake invoice**: who it claimed to be from vs `original-client-ip` (Section 8):
10. **The `SendAs` record** and its ClientIP (Section 8):
11. **IT's 3 cleanup actions on 11 Aug** (Section 9):
12. **After 11 Aug — any more attacker sign-ins?** (Section 9):
13. **The three decoys and why they are innocent** (Section 10):
14. **In my own words, the whole story in 5 sentences** (link: phish → stolen login → fake
    sign-ins → reading payment emails → secret forwarding → fake invoice → money loss →
    caught → cleaned):

---

## Command cheat sheet (keep this open)

```
pwd                     # where am I
ls                      # what's in this folder
cd Desktop/bec-email-cti # go into the exercise folder
head -N file            # first N lines of a file
grep "word" file        # lines containing word
grep -c "word" file     # count of matching lines
grep "word" file | head  # first few matches
cat file | more         # page through a long file (press q to quit)
wc -l file              # number of lines in a file
```

## Troubleshooting

| Symptom | Fix |
|---|---|
| `No such file or directory` | You are in the wrong folder. Type `pwd`. Then `cd Desktop/bec-email-cti`. |
| `command not found: grep` | Very unusual on Kali — ask your instructor. |
| Output is huge | Add `\| head -20` at the end (pipe + head). |
| Question marks / boxes in output | The log files are UTF-8 text; this is just your terminal font for quotes — ignore it. |
| "Permission denied" | You are not in the right directory or file — check `ls -la`. |

## Glossary

- **IP address** — the internet "postcode" of a computer. `208.67.222.222`.
- **JSON / CSV / log** — ways of storing text data; you learned to read all three today.
- **UTC** — one world clock; all these logs use it.
- **Phishing** — a fake email that tricks a person into typing their password on a fake page.
- **AITM phishing** — a fake login page that steals the password *and* the browser session.
- **Account takeover (ATO)** — attacker logs in as the victim using stolen credentials.
- **Forwarding rule** — an email rule that silently copies/sends emails elsewhere, and can
  mark them read and hide them in a folder.
- **SendAs** — acted as the mailbox owner; the record proves a "send" happened from an account.
- **MFA** — a second check (phone code / app) before login. "MFA not required" = password alone
  was enough.
- **IOC** — "Indicator of Compromise": anything you can point to and say *"this is attacker
  stuff"* (an IP, a domain, an email address).

---

**When you finish:** hand in your "My findings" section. 