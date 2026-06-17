# Dummy dataset — "Project Atlas" (fictional)

Self-created, fully fictional data for the Risk Radar prototype. No real,
confidential, or proprietary information. Names, teams, and tickets are invented.

**Project:** Atlas — Mobile Checkout Revamp
**Target launch:** June 30
**Sources:** daily standup, Slack `#atlas-eng`, Jira, PM weekly email, PR notes

---

## Week of June 15 — raw updates (this is the demo input)

```
[STANDUP · Mon] Checkout API integration is basically done but QA hasn't
started — still waiting on the staging env from DevOps, blocked since Thurs.
Priya OOO this week so the payments piece is unowned right now.

[SLACK #atlas-eng] "guys the new pricing service is returning 500s intermittently,
not sure if it's us or the vendor. might need to roll back if not fixed by Wed" — Arjun

[JIRA ATL-204] Design handoff for the address form slipped again, now 3 days late.
Frontend can't start the form work until specs land. Depends on Design team.

[EMAIL · PM weekly] Overall we're tracking to the June 30 launch but it's tight.
Legal review of the new T&Cs hasn't been scheduled yet — flagging as a concern.
Load testing still not done. Two engineers out next week.

[PR NOTES] Merged the cart refactor but test coverage dropped to 61%,
a few flaky tests on payment retries. Tech debt piling up on the webhook handler.
```

### Expected radar read (RED)
- **Health:** RED · ~34/100 · declining
- **Blockers (2):** QA blocked on staging env; pricing service 500s
- **Risks (7):** pricing rollback, design-handoff slip, staff out/unowned
  payments, unscheduled legal review, no load testing, dropped coverage, tight date
- **Dependencies (5):** QA→DevOps (blocked), Frontend→Design (blocked),
  Checkout→Pricing vendor (at-risk), Launch→Legal (at-risk), Payments→owner (at-risk)

---

## Bonus: a "GREEN week" you can paste to see the radar flip

```
[STANDUP] Checkout API done and QA passing on staging. Payments owned by Priya, on track.
[SLACK] Pricing service stable all week, no errors. Vendor confirmed.
[JIRA] Address-form specs delivered, frontend started and ahead of schedule.
[EMAIL · PM weekly] Comfortably tracking to June 30. Legal review booked for Thursday.
Load testing passed at 2x expected peak. Full team in next week.
[PR NOTES] Coverage back up to 84%, flaky tests fixed, webhook handler refactored.
```

Paste this into the dashboard's input box and re-run to watch health move toward GREEN.
