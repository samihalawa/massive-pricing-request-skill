---
name: massive-pricing-request-skill
description: Use when the user wants to canvass many providers, companies, or independent professionals to get a CLOSED, specific price quote on anything — a software project, a trade job (plumbing/repairs), a house move, private lessons, or any other product/project/service — either to research market pricing (competitive intelligence) or to personally procure a service. Sends one individual email per recipient (never BCC/bulk), the message explicitly demands a closed price, and a later pass parses replies into a price comparison. Default city Madrid, default sender samihalawaster@gmail.com via Gmail SMTP.
metadata:
  author: samihalawa
  version: "1.0.0"
  license: MIT
---

# Massive Pricing Request Skill

## Purpose

Get real, comparable, CLOSED price numbers from many providers at once, for any category — not just software. One template mechanism, category-specific slots. Two use modes share the same mechanics: pricing *research* (to set your own prices) and personal *procurement* (you need the service yourself).

Reference implementation this skill generalizes: a 104-company Spanish software-dev outreach for a "Farm Management Software" closed-quote request, sent individually via Gmail SMTP, replies later pulled and parsed via IMAP into a price comparison (quotes ranged €40K–€1.5M across responders). Same mechanics apply to "necesito un fontanero en Madrid", "quiero presupuestos de mudanza", or "busco profesor de portugués y su precio por hora".

## Mode Resolver

Classify the request before acting:

- **Market research mode**: user wants to know what others charge for a category of work (to price their own services, or just to understand the market). Recipients are providers/competitors in that space. Tone: "we are evaluating vendors for a project" (can be a real or representative project brief).
- **Personal procurement mode**: user actually needs the service themselves right now (a plumber, a mover, a tutor, a contractor). Recipients are providers who could do the actual job. Tone: same closed-price demand, but the brief is the user's real situation (address, dates, exact job).

Both modes use the identical send/track/parse mechanics below. Only the brief and the recipient search differ.

## Category Definition (always resolve first)

Before discovery, nail down with the user (or from context, if unattended) — do not invent silently on consequential specifics like budget ceilings, but reasonable assumptions for location/scope are fine to state and proceed:

1. **What** exactly is being priced (be as specific as the reference project brief was for software: functional requirements, technical metrics, scope boundaries).
2. **Where** — default city is **Madrid** unless the user names another city or comunidad autónoma. If the user wants a *national average* or cross-region comparison, fan the same campaign out to multiple cities/regions and label each recipient with its region so the final comparison can be grouped by region.
3. **Category-specific info to request in the reply** — this is what changes per vertical:
   - Software/dev project → final price or tight range, MVP vs. full-version timeline, tech stack, 1-2 prior relevant projects.
   - Trades (plumbing, electrics, repairs) → closed price for the described job, materials included/excluded, warranty period, earliest availability.
   - Moving/mudanza → closed price based on inventory + distance + floor/elevator, insurance coverage, availability dates.
   - Tutoring/lessons → price per hour (and per-pack if they offer one), format (online/in-person), availability, relevant credentials/experience.
   - Anything else → improvise an analogous 3-5 item info block: price, timeline/availability, what's included, relevant credentials/experience.

## Identity & Sender Rules (hard rule)

- **Default FROM: `samihalawaster@gmail.com`, sent via Gmail SMTP** (`smtp.gmail.com:587`, STARTTLS). A personal-looking Gmail address reads as an individual/small buyer, which gets more honest, less-inflated pricing and more complete detail than a branded company domain does.
- **Never send this kind of campaign from `sami@samihalawa.com`** or any other address that reads as a branded/agency/big-company domain. Recipients who think they're quoting "a company" tend to either inflate the price or hold back real numbers, expecting a negotiation. This applies whether the campaign is for research or personal procurement.
- Credentials: read `SMTP_HOST`, `SMTP_PORT`, `SMTP_USER`, `SMTP_PASSWORD` from environment/`.env` if present; otherwise ask the user once for the samihalawaster@gmail.com app password (never invent or reuse an unrelated stored credential silently — confirm which mailbox's app password is being used before the first send). Do the same for `IMAP_HOST`/`IMAP_PORT`/`IMAP_USER`/`IMAP_PASSWORD` for the reply-check phase (same mailbox, same credentials, IMAP port 993).
- Never print or log the password in any report shown to the user.

## Workflow

1. **Define category** — resolve the What/Where/category-info-block above.
2. **Discover targets** — web search / directories / maps listings for providers matching the category + location. Build a deduplicated list of individual recipient emails (not generic `info@` only if a named contact is findable — named contacts get read more often). Log the list to a plain file (`recipients.txt`) before sending, one address per line, so a resend or a "who did we already contact" check is trivial.
3. **Draft the closed-price message** — Spanish by default for Spain-based targets (match the recipient's likely language; English if targeting non-Spanish markets). Structure:
   - Short opening stating what's being priced.
   - Explicit line that the requester evaluates by clarity and final price only, and is not doing sales calls as the primary channel (acknowledge in the closing that clarifying questions by email are fine — many good vendors will ask before quoting; that's expected, not a failure of the template).
   - The category-specific requirements block (bullet list of what's needed/expected).
   - The category-specific "information required in your reply" block (price/range, timeline or availability, what's included, relevant experience/credentials).
   - Sign-off as Sami Halawa, reachable at `samihalawaster@gmail.com`.
4. **Send individually** — one SMTP send per recipient, in a loop, never BCC/CC multiple recipients on one message. Reuse the pattern: open one SMTP connection, reconnect on `SMTPServerDisconnected`/`SMTPSenderRefused`, small delay (2-4s) between sends to avoid throttling/spam flags, log `OK`/`FAIL` per recipient to a `send.log`, print a final `DONE ok=X fail=Y total=Z` line.
5. **Track sends** — keep `recipients.txt` (who was targeted) and `send.log` (what actually went out) so a later "send to the rest" or "resend to non-repliers" never re-sends to someone already contacted, unless the user explicitly asks for a resend.
6. **Check replies** — on request (or after a reasonable wait, e.g. same day or next day), connect via IMAP to the sending mailbox, pull messages since the campaign's send date, and read full bodies (not just subjects) — subject lines alone are not enough to judge whether a real price was quoted.
7. **Classify every genuine reply** into exactly one bucket:
   - **Closed price given** — extract the number(s)/range, currency, what's included, timeline.
   - **Wants more info before quoting** — note what they're asking for (useful if the user wants to answer and get a second-round price).
   - **Declined / out of scope** — no price, not pursuing.
   - **Noise** — newsletters, security alerts, delivery bounces, unrelated automated mail — exclude from the comparison entirely, but don't hide that they exist if the user asks for a total inbox count.
8. **Report a price comparison** — for "closed price given" replies, list them sorted lowest-to-highest (or grouped by region if multi-region), with company name, price/range, timeline, and one line of what's included. Surface the "wants more info" group separately since those can become second-round closed quotes if the user answers them. Never blend estimated/vague figures into the same list as genuinely closed ones without flagging the difference.

## Hard Rules

- Individual sends only — no BCC, no mass single-message blasts. This was proven necessary in the reference campaign: individual sends got real, personalized, often very detailed closed-price replies; anything that reads as a mail-merge blast gets ignored or auto-filtered.
- The message must explicitly ask for a **closed/specific price**, not merely "a quote" — vague requests get vague ranges back.
- Default sender `samihalawaster@gmail.com`; never `sami@samihalawa.com` or another branded domain for this kind of price-comparison campaign.
- Default location Madrid; stay flexible to any city/comunidad the user names, and support fanning the same campaign across multiple regions when the user wants a national average.
- Always do the reply-check and classification pass as its own step — sending without ever coming back to read and parse the replies defeats the entire purpose of the skill.
- Never fabricate a price on behalf of a provider who didn't give one — "wants more info" and "declined" are not "closed price given".
- If SMTP sending from the intended account fails (auth error, provider block), diagnose before switching accounts: a handful of test sends to confirm whether it's a credentials issue or a provider-side block, then report the finding plainly rather than silently trying an unrelated mailbox.
