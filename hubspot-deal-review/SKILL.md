---
name: hubspot-deal-review
description: Weekly HubSpot deal health review for the Fablefood sales team. Finds deals with data quality issues (stale stages, missing amounts, overdue close dates, no owner) and walks the user through fixing each one interactively using popup panels. Use when asked to "run the deal review", "check deal health", or on the weekly schedule.
---

# HubSpot Deal Review

This skill runs a structured review of the Fablefood HubSpot pipeline, finds deals with problems, and uses interactive prompts to fix them one by one — without the rep having to open HubSpot.

---

## When to invoke

- User says "run the deal review" or "weekly deal check"
- Triggered by the weekly schedule
- User says "check which deals have issues"

---

## Review process

### Step 1 — Find problematic deals

Search HubSpot for deals in each of these problem categories. Run all searches before presenting anything to the user.

**Categories to check:**
1. **Overdue** — `dealstage` is not `closedwon` or `closedlost`, and `closedate` is in the past
2. **Missing amount** — `amount` is null or 0 for deals not in early stages
3. **Stale** — `hs_lastmodifieddate` is more than 30 days ago and stage is not closed
4. **No owner** — `hubspot_owner_id` is null

Search using `search_deals` with relevant keywords, then filter the results by the criteria above. Group issues by deal — one deal can have multiple problems.

### Step 2 — Present the summary

Before asking about individual deals, give the user a brief headline:

> "Found [N] deals with issues:
> - [N] overdue
> - [N] missing amount
> - [N] stale (30+ days no activity)
> - [N] no owner
>
> Ready to work through them?"

Wait for confirmation before proceeding.

### Step 3 — Fix each deal interactively

For each deal with issues, use `AskUserQuestion` to present a panel. Do NOT ask about multiple deals in one panel — one at a time.

**Panel format for each deal:**

Title: `[Deal Name] — [Issue Type]`

Body:
```
Deal: [dealname]
Stage: [dealstage]  |  Amount: $[amount]  |  Close date: [closedate]
Owner: [owner name or "unassigned"]
HubSpot: [url]

Issues found:
• [list each issue on its own line]

What would you like to do?
```

Options to present (use `AskUserQuestion` with these as selectable choices where possible):
- Update the close date
- Update the stage
- Update the amount
- Assign an owner
- Flag this as a data quality issue (posts to Slack)
- Skip this deal
- Stop review

### Step 4 — Apply fixes

Based on the user's response, call the appropriate tool:
- Close date / stage / amount / owner → `update_deal`
- Flag → `flag_issue`
- Skip → move to next deal
- Stop → end the review

After each fix, confirm what was updated and show the HubSpot link.

### Step 5 — Summary

When all deals have been reviewed, show a summary:
```
Deal review complete.
Fixed: [N] deals
Flagged: [N] issues
Skipped: [N] deals

All changes are logged in the audit trail.
```

---

## Important rules

- Never update more than one field per `update_deal` call — do separate calls if multiple fields need fixing
- Always show the HubSpot URL with the deal name so the rep can verify
- Never guess at values — if the user's answer is ambiguous, ask a follow-up before writing
- If a deal has both missing amount AND overdue close date, treat them as two separate questions in the same panel
- If the user says "flag it" without specifying an issue, ask them to describe the problem before calling `flag_issue`

---

## Tools used

From the HubSpot MCP proxy:
- `search_deals` — find candidate deals
- `get_deal` — get full deal details
- `update_deal` — fix stage, amount, close date, owner
- `flag_issue` — post data quality issue to Slack

Built-in Claude Code tools:
- `AskUserQuestion` — interactive panel for each deal
