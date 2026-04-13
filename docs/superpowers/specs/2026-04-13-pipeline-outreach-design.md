# Chicago Multifamily Pipeline — Outreach Design

**Date:** 2026-04-13
**Status:** Complete — ready for implementation
**Parent spec:** `2026-04-09-chicago-multifamily-pipeline-design.md`

---

## Overview

This spec defines the outreach subsystem: how the pipeline turns scored parcels into personalized outreach, manages a multi-touch sequence across channels, and tracks responses. The core principle: **every message sounds human, speaks to the owner's interests, and asks for a conversation — not a transaction.**

---

## Contact Enrichment

### Provider

**REISkip** — pay-per-match skip tracing. $0.15 per successful match, no subscription, no minimum commitment.

### Workflow

1. After scoring, select top parcels for outreach in the Flask UI
2. For LLC-owned parcels, run the IL SOS scraper first to get the registered agent name
3. Export selected parcels as CSV from the UI (owner name, property address, mailing address; registered agent name for LLCs)
4. Upload CSV to REISkip manually
5. Download results CSV with phone numbers and emails
6. Import results back into the pipeline — stored in the `contacts` table

### What Gets Stored

| Field | Source |
|---|---|
| Owner name | Assessor records |
| Phone number(s) | REISkip |
| Email address(es) | REISkip |
| Mailing address | Assessor records / REISkip |
| `source` | `reiskip` |
| `role` | `owner` (or `registered_agent` for LLCs) |

### Volume and Cost

- ~20-50 records per wave
- ~$3-7.50 per wave
- No API integration — CSV round-trip takes ~5 minutes at this volume. Revisit if volume grows significantly.

---

## Outreach Positioning

All outreach is from you as a **prospective buyer/developer who lives in the neighborhood.** Not representing a third party. No broker language. No "we" — always "I."

### License Constraint

You don't hold a broker's license. All outreach positions you as a principal (prospective buyer), not an intermediary. You're reaching out to explore whether the owner would consider selling to you directly.

### Broker Route Exception

When the pipeline identifies a property that is currently listed with a broker, it is flagged as "listed" in the UI. The outreach sequence is skipped entirely. You pass the listing info to your developer partner manually.

---

## Messaging Strategy

### Five Criteria

Every outreach message — email, letter, postcard, phone script — must satisfy all five:

1. **Sound human** — conversational, first-person, no corporate jargon or template feel. Plain text. Short sentences. The reader should feel like a person wrote this for them, not that a system generated it.
2. **Speak to their interests** — Dale Carnegie: frame around what the owner wants (relief, certainty, value realization, simplicity), not what you want. Use loss aversion framing carefully (Kahneman/Tversky). Never lead with "I want to buy your property."
3. **Local connection** — mention living in the area. Reference the specific neighborhood. Establish that you're a neighbor, not a faceless investor.
4. **Hook** — personalized to the parcel's strongest scoring signal, framed around the owner's situation using the yes-yes technique (see below).
5. **CTA = in-person meeting** — every message ends with a low-pressure ask to meet. Coffee, a walk-through, a conversation. Never "are you interested in selling?"

### Yes-Yes Framing (Dale Carnegie)

Open with 1-2 statements the owner will naturally agree with, building psychological momentum toward the ask. The hook signal determines which yes-yes chain to use.

**Hook selection logic:** The system picks the strongest scoring signal per parcel and frames it around the owner's lived experience:

- **High FAR gap** → "You've probably noticed how much new construction is happening on your block. A lot of owners in your position are realizing their property is worth more than they thought. I'd love to buy you a coffee and share what I'm seeing."
- **Long hold + rising taxes** → "After owning a property for 20+ years, you know the neighborhood better than anyone. You've also probably noticed your tax bill climbing. I'd love to sit down and talk about what options might make sense for you."
- **Neighborhood momentum** → "Your block has changed a lot in the last few years. Properties like yours don't come along very often. I'd love to meet up and share what I'm seeing in the area."
- **Deferred maintenance** → "Older buildings take real work to maintain — you know that better than anyone. At some point every owner weighs whether it still makes sense. If you've ever had that thought, I'd welcome the chance to talk."
- **Absentee owner** → "Managing a property from out of the area takes a lot of trust and effort. There comes a point where most owners start thinking about whether it's still worth it. If that sounds familiar, I'd love to meet and hear what you're thinking."

These are framing patterns, not hardcoded copy. Claude API receives the signal type + owner data and drafts around this structure, varying the language naturally per parcel.

### Research-Backed Messaging Guidelines

Based on academic and industry research informing the messaging approach:

- **Email length:** 50-125 words (Boomerang, 40M+ emails — highest response rates in this range)
- **Subject lines:** Include property address or neighborhood. Questions outperform statements by 10-20% (Woodpecker). 6-10 words optimal (Retention Science).
- **Personalization impact:** Personalized cold emails produce 32.7% higher response rates than generic templates (Backlinko, 12M emails). Property-specific references are the highest-impact personalization tier.
- **Plain text only:** Plain text emails get 2x higher reply rates than HTML in cold outreach (Lemlist). Also avoids spam filters.
- **Send timing:** Tuesday-Thursday, 9-11 AM local time (HubSpot, 20M+ emails)
- **Narrative transportation:** Personal story framing ("I live in the neighborhood, here's why I'm reaching out") outperforms transactional messaging (Green & Brock, 2000)
- **First follow-up is the highest-leverage message:** 40% higher reply rate than the initial email alone (Woodpecker)

---

## Outreach Sequence

### 7 Touches Over 30 Days

| Touch | Day | Channel | Method | Notes |
|---|---|---|---|---|
| 1 | 0 | Email | Automated (Gmail via Claude connector) | Personalized, strongest signal as hook, yes-yes framing |
| 2 | 3 | Email | Automated (Gmail via Claude connector) | Short follow-up, different signal/angle than Touch 1 |
| 3 | 7 | Phone | Manual (UI surfaces number + script) | Script follows same five messaging criteria, conversational not verbatim |
| 4 | 14 | Physical letter | Automated (Lob API) | Handwritten font on letter + envelope, home return address |
| 5 | 19 | Email | Automated (Gmail via Claude connector) | New hook or neighborhood momentum angle |
| 6 | 24 | Postcard | Automated (Lob API) | Standard 4x6, brief reminder + CTA |
| 7 | 30 | Email | Automated (Gmail via Claude connector) | Final "circling back, still interested" — warm close |

### Sequence Rules

- **No repeated messaging:** Each touch uses a different signal or angle. No one gets the same message twice.
- **Response stops the sequence:** If the owner responds at any point (any channel), the sequence stops automatically and the parcel moves to `responded` stage.
- **End of sequence:** If no response after Touch 7 (Day 30), the parcel moves to `dead` unless manually overridden.
- **Listed properties skip entirely:** Flagged as "listed" in the UI. No outreach sent. You pass to your developer partner manually.
- **Nothing sends without approval:** Every automated message (email, letter, postcard) is drafted, reviewed by you, and explicitly approved before sending.
- **Missing contact info:** If REISkip returns no email for a parcel, email touches are skipped and the sequence shifts to the next available channel (phone on Day 7, letter on Day 14). If no phone number either, the sequence starts at Touch 4 (letter). Mailing address is always available from assessor records, so physical mail is the fallback channel.

### Estimated Cost Per Parcel (Full Sequence)

| Item | Cost |
|---|---|
| Skip trace (REISkip) | ~$0.15 |
| Emails (5 touches via Gmail) | Free |
| Phone call (manual) | Free |
| Letter (Lob) | ~$1.20 |
| Postcard (Lob) | ~$0.50 |
| Claude API drafting (Haiku, ~7 calls) | ~$0.01 |
| **Total per parcel** | **~$1.86** |

---

## Message Drafting (Claude API)

### Batch Generation

When you click "Draft Outreach" for a wave of selected parcels, the system calls the Claude API once per parcel per touch. All drafts are generated upfront and stored in the DB. You review the full wave at once before approving anything.

### Model

**Haiku** — more than capable for 50-125 word personalized messages from structured prompts. Fractions of a cent per message. Use a smarter model only if drafts need complex reasoning, which outreach emails don't.

### Prompt Structure

Templates stored in `config/templates/` as prompt instruction files (not mail-merge templates). Each contains:

- The five messaging criteria as system instructions
- The yes-yes framing pattern for the relevant signal type
- Touch number context (first touch vs. follow-up vs. final — tone and approach shift)
- Which parcel/owner fields to reference

At draft time, the system injects parcel-specific data into the prompt:

- Owner name, property address, neighborhood
- Strongest scoring signal(s) and their plain-English meaning
- Hold duration, property details, any relevant context from the parcel record
- Touch number and what prior touches said (so follow-ups don't repeat)

### Output Formats

- **Email (Touches 1, 2, 5, 7):** Plain text body + subject line. No HTML, no images.
- **Letter (Touch 4):** Plain text body formatted for Lob letter dimensions.
- **Postcard (Touch 6):** Front message (brief, ~30 words) + back content (name, phone, neighborhood reference).
- **Phone script (Touch 3):** Talking points and conversation starters, not a verbatim script. Displayed in the UI when the phone touch is due.

### Review Flow

1. Drafts appear in the UI grouped by wave
2. You read each draft, edit inline if needed, approve or reject
3. Approved emails send via Gmail (Claude connector integration); approved letters/postcards send via Lob API
4. All drafts (original + any edits) stored in DB for learning over time

---

## Gmail Integration

### Connection

Uses the existing **Claude connector integration** with your personal Gmail account. No separate OAuth setup needed.

### Sending Approach

- Send from your personal Gmail address
- Plain text only — no HTML templates, no images, no tracking pixels
- Volume: ~20-50 emails per wave, well under Gmail's 500/day limit for personal accounts
- Send timing: Tuesday-Thursday, 9-11 AM Central (research-backed optimal window)

### Deliverability

At 20-50 emails per wave, spam risk is minimal. If deliverability becomes an issue in the future, migrate to a dedicated Google Workspace account (~$7/month) with 2-4 weeks of warmup. For now, personal Gmail is sufficient.

### Spam Filter Avoidance

- Plain text (no HTML) avoids the primary spam trigger
- Low volume avoids rate-based filtering
- Personalized content (not identical across recipients) avoids pattern-based filtering
- No tracking links, no images, no attachments

---

## Lob API Integration

### Account

Standard Lob account, pay-per-piece pricing.

### Letter (Touch 4)

- **Font:** Handwritten-style font on both the letter and envelope
- **Return address:** Home address
- **Content:** Claude-drafted, approved by you, sent to Lob via API
- **Address validation:** Lob's built-in CASS verification runs before sending. If an address is undeliverable, the UI flags it and skips the touch.

### Postcard (Touch 6)

- **Size:** Standard 4x6
- **Front:** Brief personalized message (Claude-drafted, ~30 words)
- **Back:** Your name, phone number, neighborhood reference
- **Address validation:** Same CASS verification as letters

### Integration Flow

1. You approve the draft in the UI
2. System calls Lob API with recipient address, return address, and message content
3. Lob validates address, creates the mail piece, returns a tracking ID
4. Tracking ID and send status stored in the `outreach` table
5. Lob handles printing, stamping, and mailing

---

## Sequence Scheduler & UI

### Dashboard Prompt on App Load

Every time you open the Flask UI, it runs a query against the `outreach` table:

```
For each parcel where stage = 'outreach':
  Find the most recent outreach row → which touch, when sent
  Apply the 30-day sequence timing rules
  If next touch is due today or overdue → surface it
```

### "Due Today" Section

Appears at the top of the UI on load:

- Groups due items by channel: "3 emails ready to draft," "2 phone calls due," "1 letter ready to send"
- Click into any item to see the parcel detail panel + draft/approve the next touch
- Overdue items highlighted so nothing falls through the cracks

### Sequence Status Per Parcel

The detail panel (right column of the existing Review UI) shows:

- Where the parcel is in its 7-touch sequence (e.g., "Touch 3 of 7 — Phone call — Day 7")
- Timeline view: which touches have been sent, which are upcoming, which are overdue
- If the owner responds at any point, sequence stops and parcel moves to `responded`

### No Background Jobs

No cron, no background processes. The query runs on page load against `outreach.sent_date` + sequence timing rules defined in config. The Flask app is stateless — it reads the DB and shows what's due.

---

## Database Changes

All outreach data uses the tables already defined in the master spec (`contacts`, `outreach`, `waves`). The following additions are needed:

### `outreach` table — additional columns

| Column | Type | Notes |
|---|---|---|
| `touch_number` | int | Which touch in the sequence (1-7) |
| `draft_body` | text | Claude-generated draft (original) |
| `final_body` | text | What was actually sent (after edits) |
| `draft_subject` | text | Email subject line (null for mail/phone) |
| `lob_tracking_id` | text | Lob tracking ID (null for email/phone) |
| `lob_status` | text | Lob delivery status (null for email/phone) |

### `sequence_config` — stored in YAML, not DB

Sequence timing and channel mapping stored in `config/outreach.yaml`:

```yaml
sequence:
  - touch: 1
    day: 0
    channel: email
    template: templates/email_first_touch.md
  - touch: 2
    day: 3
    channel: email
    template: templates/email_follow_up_1.md
  - touch: 3
    day: 7
    channel: phone
    template: templates/phone_script.md
  - touch: 4
    day: 14
    channel: mail_letter
    template: templates/letter.md
  - touch: 5
    day: 19
    channel: email
    template: templates/email_follow_up_2.md
  - touch: 6
    day: 24
    channel: mail_postcard
    template: templates/postcard.md
  - touch: 7
    day: 30
    channel: email
    template: templates/email_final.md
```

Changing the sequence = edit this YAML. Adding a touch = add an entry. The scheduler reads this config to determine what's due.
