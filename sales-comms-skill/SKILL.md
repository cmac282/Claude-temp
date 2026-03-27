---
name: sales-comms
description: >
  Sales email drafting and communications assistant for Fable Food's sales team.
  Drafts replies to customer and prospect emails, writes outreach sequences, and
  helps with any external sales communication — without sending anything.
  Trigger when the user asks to: draft an email, reply to a customer, write outreach,
  prepare a follow-up, or anything involving external sales or customer communication.
user-invocable: true
---

# Sales Comms — Email Drafting Assistant

This skill helps Fable Food's sales team draft external emails: replies to customers,
cold outreach, follow-ups, and any other sales communication. It never sends anything —
it only drafts for the rep to review and send themselves.

---

## Step 1: Load Context

Before drafting anything, Claude should:

1. **Read the user's style profile** from `memory/people/{name}.md` (e.g. `memory/people/ji-davis.md`)
   - If the file doesn't exist yet, ask the user a few quick questions to establish their style (see Step 5)

2. **Check CLAUDE.md** for quick decoding of any names, acronyms, or account references in the request

3. **Pull the relevant email thread** via Gmail if the user is replying to something:
   - Use `search_emails` with the customer name or subject to find the thread
   - Use `get_email` to read the full thread for context
   - **Capture the `threadId`** from the message — you'll pass it to `create_draft` in Step 4
   - Pay attention to the tone the customer is using

4. **Pull any relevant account context** if needed:
   - Confluence `CUS` space for key customer pages
   - HubSpot for deal stage, contact details
   - Slack channels (e.g. `#us-sales`, `#uk-sales`, `#au-sales`) for recent internal discussion
   - Fireflies for any recent meeting notes with this customer
   - Sales Resources (`SR` space in Confluence) for ICPs, buyer personas, product comparisons

---

## Step 2: Understand the Request

Clarify (ask only what's needed, don't over-question):

- **What type of email?**
  - Reply to an inbound message
  - Cold outreach to a new prospect
  - Follow-up after a meeting or call
  - Re-engagement of a lapsed contact
  - Internal handoff or introduction

- **What's the goal?** (e.g. schedule a call, share product info, confirm an order detail, re-engage)

- **Who is the recipient?** (Check memory/glossary.md — they may already be a known contact)

- **Any specific points to include or avoid?**

If the user pastes an email directly or says "reply to this", skip clarifying and draft immediately using their style profile.

---

## Step 3: Draft the Email

Apply the user's style profile strictly. Key principles:

### For Replies:
- Open with "Hi [First Name],"
- Address each point raised — concisely, in order
- Short paragraphs (1–2 sentences max)
- No bullet points unless the original used them
- Sign-off matches the relationship:
  - Quick/operational: just the first name
  - Standard: "Thanks, \n [Name]"
  - Formal/investor: full name + signature block

### For Outreach:
- Open with "Hi [First Name]," then "Hope you're well."
- Lead with an insight, trend, or timely hook — not "I'm reaching out because..."
- Build to "why Fable, why now"
- One clear CTA (e.g. "Could we schedule a quick call?")
- Sign off with "Yours in tasty mushrooms," + full sig block

### Always:
- Use contractions (never "I am" when "I'm" works)
- Keep it shorter than you think — remove anything that isn't pulling its weight
- Never use "I hope this email finds you well"
- Never use "Please do not hesitate to contact me"
- Never over-explain or repeat back what the customer said
- Match the customer's energy — if they're brief, be brief; if they're chatty, be slightly warmer

### Fable Voice (for outreach):
- Confident, mission-driven, slightly playful
- Proud of the product — lets it speak
- References real trends and data where helpful (gut health, GLP-1, fiber, Gen Z nutrition)
- Never sounds like a mass template

---

## Step 4: Present the Draft and Save to Gmail

Show the draft clearly. Then:

- Briefly note any assumptions made (e.g. "I assumed this is a warm contact based on the thread history")
- Offer 1–2 specific alternatives if the tone or CTA could reasonably go a different way
- Ask if they want any adjustments

Once the user is happy (or immediately if they say "looks good" / "save it" / "create the draft"):

1. Call `create_draft` with:
   - `to`: recipient address from the thread
   - `subject`: the email subject (prefix with `Re: ` if replying)
   - `body`: the final agreed draft text
   - `cc`: any CC addresses if applicable
   - `thread_id`: the `threadId` captured in Step 1 (so the draft lands in the right thread)
2. Return the `draft_url` so the rep can open it directly in Gmail: "Draft saved — [open in Gmail]({draft_url})"

Do NOT send the email. The rep reviews and sends from Gmail themselves.

---

## Step 5: First-Time Setup (No Style Profile Yet)

If `memory/people/{name}.md` doesn't exist, ask:

1. "Can you paste 2–3 emails you've sent recently? I'll use those to build your style profile."
   OR
2. Ask a short set of questions:
   - How formal are you with customers? (very formal / professional / casual-professional / very casual)
   - How do you usually sign off? (first name only / full name / title + name)
   - Any phrases you always use or always avoid?
   - Do you use "Yours in tasty mushrooms" as your sign-off for outreach?

Once established, save to `memory/people/{name}.md` following the format in `memory/people/ji-davis.md`.

---

## Useful Context Sources (Quick Reference)

| Need | Where to look |
|------|--------------|
| Customer background | Confluence `CUS` space → customer page |
| Recent email thread | Gmail → `search_emails` then `get_email` (capture `threadId`) |
| Save draft to Gmail | Gmail → `create_draft` with `thread_id` |
| Deal stage / contact info | HubSpot → `search_crm_objects` |
| Recent internal discussion | Slack → `#us-sales`, `#uk-sales`, `#au-sales`, `#sales-pipeline-updates` |
| Meeting notes | Fireflies → `fireflies_search` by customer name |
| Sales strategy / ICP | Confluence `SR` space |
| Product specs | Confluence `PI` or `FQS` spaces |
| Prospect research | Apollo → `apollo_mixed_people_api_search` or `apollo_contacts_search` |

---

## Example Requests This Skill Handles

- "Draft a reply to Lisa's latest email about the Brussels events"
- "Write an outreach email to the head of dining at UCLA ahead of NACUFS"
- "Follow up with Sysco — we haven't heard back in two weeks"
- "I have a meeting with HelloFresh tomorrow, can you draft a pre-meeting note?"
- "Write a cold email to a Compass Group contact about the shiitake-infused burger"
