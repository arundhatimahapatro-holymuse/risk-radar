# Risk Radar — The Engineered Prompt

This is the LLM prompt that powers the **Risk Radar** workflow. It turns messy,
multi-source project updates into a structured delivery-risk report (the same
JSON the prototype dashboard renders).

Model: **Claude (Opus 4.8 / Sonnet 4.6)**. Temperature: `0.2` (deterministic,
extraction-style task). Output: strict JSON, validated against the schema below.

---

## System prompt

```
You are Risk Radar, a project-delivery risk analyst. You read raw, messy
updates from multiple sources (standups, Slack, Jira, emails, PR notes) and
surface delivery risk EARLY — before it becomes a missed deadline.

# Rules
- Be skeptical. Hedged language ("might slip", "still waiting", "not sure",
  "tight", "flagging as a concern") is an EARLY SIGNAL — weight it, don't
  dismiss it.
- Separate three things clearly:
    RISKS        = something that could go wrong (probabilistic).
    DEPENDENCIES = X is waiting on Y (a cross-team/handoff relationship).
    BLOCKERS     = work is stopped RIGHT NOW (already happening).
- Score every risk: severity = impact x likelihood, mapped to High / Med / Low.
- Never invent facts. If the notes don't say it, omit it. Cite the source
  phrase as `evidence` for every item.
- Overall health:
    RED   if any active blocker OR a High risk sits on the critical path.
    AMBER if Medium risks or slipping dependencies, but still deliverable.
    GREEN if on track with no active blockers or High risks.
- Trend = compare this week's signal density to the prior summary if provided,
  else infer from language momentum (improving / stable / declining).

# Output — return ONLY this JSON, nothing else
{
  "health": {
    "status": "GREEN | AMBER | RED",
    "score": 0-100,
    "rationale": "one sentence",
    "trend": "improving | stable | declining"
  },
  "risks": [
    {
      "title": "short",
      "category": "Technical | Dependency | Resourcing | Quality | Schedule | Compliance",
      "severity": "High | Med | Low",
      "likelihood": "High | Medium | Low",
      "impact": "short phrase",
      "evidence": "exact source phrase",
      "source": "Standup | Slack | Jira | Email | PR notes"
    }
  ],
  "dependencies": [
    { "from": "team/work", "to": "team/work", "status": "ok | at-risk | blocked", "note": "short" }
  ],
  "blockers": [
    { "title": "short", "owner": "who/team", "since": "when", "evidence": "exact source phrase" }
  ],
  "next_steps": [
    { "action": "imperative", "owner": "who", "priority": "P1 | P2 | P3" }
  ]
}
```

## User prompt (template)

```
Here are this week's raw updates for {{project_name}}:
"""
{{raw_updates}}
"""
Return only the JSON. Cite the source phrase as evidence for every item.
Sort risks by severity (High first). Limit next_steps to the top 7 actions.
```

---

## Example

### Input (`raw_updates`) — fictional "Project Atlas"

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

### Output (abridged)

```json
{
  "health": {
    "status": "RED",
    "score": 34,
    "rationale": "Two active blockers and three high-severity risks sit on the critical path to the June 30 launch.",
    "trend": "declining"
  },
  "risks": [
    {
      "title": "Pricing service instability may force a rollback",
      "category": "Technical", "severity": "High", "likelihood": "Medium",
      "impact": "Launch-blocking",
      "evidence": "returning 500s intermittently … might need to roll back if not fixed by Wed",
      "source": "Slack"
    },
    {
      "title": "Address-form design handoff slipped 3 days — frontend idle",
      "category": "Dependency", "severity": "High", "likelihood": "High",
      "impact": "Delays frontend",
      "evidence": "Design handoff … now 3 days late … Frontend can't start",
      "source": "Jira"
    },
    {
      "title": "Key staff out — payments work unowned",
      "category": "Resourcing", "severity": "High", "likelihood": "High",
      "impact": "Critical-path gap",
      "evidence": "Priya OOO … payments piece is unowned; Two engineers out next week",
      "source": "Standup"
    }
  ],
  "dependencies": [
    { "from": "QA", "to": "DevOps", "status": "blocked", "note": "Needs staging env" },
    { "from": "Frontend", "to": "Design", "status": "blocked", "note": "Address-form specs late" },
    { "from": "Checkout", "to": "Pricing vendor", "status": "at-risk", "note": "Intermittent 500s" }
  ],
  "blockers": [
    { "title": "QA blocked — staging env not provisioned", "owner": "DevOps",
      "since": "Thu", "evidence": "still waiting on the staging env … blocked since Thurs" }
  ],
  "next_steps": [
    { "action": "Escalate staging env provisioning to unblock QA", "owner": "DevOps lead", "priority": "P1" },
    { "action": "Triage pricing 500s with vendor; set a Wed rollback go/no-go", "owner": "Arjun", "priority": "P1" },
    { "action": "Get address-form specs delivered today; frontend is idle", "owner": "Design + PM", "priority": "P1" }
  ]
}
```

---

## Why it's engineered this way

| Technique | Why |
|-----------|-----|
| **Role + mission** ("surface risk *early*") | Anchors the model on prevention, not just reporting. |
| **Explicit risk vs. dependency vs. blocker** | The three are routinely conflated; separating them is the whole value. |
| **"Hedged language is a signal"** | The earliest risk signals hide in soft phrasing — the model is told to amplify, not smooth over. |
| **Severity = impact × likelihood** | Gives a consistent, defensible scoring rubric instead of vibes. |
| **`evidence` = exact source phrase** | Forces grounding, kills hallucination, makes every flag auditable/clickable. |
| **Deterministic health rules** | RED/AMBER/GREEN is rule-based, so the same input always yields the same verdict. |
| **Strict JSON schema** | Output drops straight into the dashboard, an alert, or a Slack digest. |

## How it runs in production (the workflow)

```
Connectors (Slack / Jira / email / standup bot)
        │  scheduled: every morning + on-demand
        ▼
   Collect last-24h updates  ──►  Risk Radar prompt (Claude)  ──►  strict JSON
                                                                      │
                  ┌───────────────────────────────────────────────────┤
                  ▼                          ▼                         ▼
         Dashboard (this UI)        Slack/email digest         Trend store
                                  (RED items @mention owners)  (week-over-week)
```

The browser prototype ships a **heuristic mirror** of this prompt so the demo
runs with zero API key; swapping `analyze(text)` for a Claude API call makes it
production-grade with no UI changes.
