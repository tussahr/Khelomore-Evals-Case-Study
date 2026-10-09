> Public copy: venue names are replaced with Venue A–E, and localities and contact details are removed.

# KheloMore Venue Assistant: Requirements

| | |
|---|---|
| **Owner** | Tushar Kumar |
| **Status** | **v1.5, 2026-10-09.** Assistant model: Sonnet 5.5, prompt v2.2. Previously v1.4: Rules R14–R19 added from the error analysis. Previously v1.3, 2026-10-08: Updated after the first error analysis: KheloMore platform policies added, labelling standard defined. |
| **Last updated** | 2026-10-07 |

---

## 1. Problem

KheloMore has **2,000+ venues**. Users land on a venue page to book a slot, but often hesitate because of unanswered questions:

- What's the cancellation policy? Is cancellation allowed at all?
- What's the refund policy?
- Can I reschedule?
- What amenities does this venue have (parking, washrooms, changing rooms, equipment)?
- Which metro station is nearest? How do I get there from where I am?

Most answers already exist somewhere: on the venue page. But they're hard to find, so users either leave without booking or raise support tickets.

## 2. Solution

An **"Ask about this venue"** button on every venue page. Tapping it opens an AI assistant that:

1. Shows **4–5 suggested questions** the user can tap for an instant answer
2. Shows a **short hint** of what the assistant can help with
3. Lets the user **ask anything in their own words and keep chatting**
4. Answers using **only** this venue's page data (there is no KheloMore-wide policy; all T&C are venue-specific)

## 3. Goal of this project

Build a prototype of the assistant **and an evaluation** that measures its quality, finds its failure modes and proves that improvements work. The eval is what decides whether the assistant is ready to go live on khelomore.com.

## 4. Users

| User | Context | What they need |
|---|---|---|
| First-time booker | New to KheloMore and this venue | Simple, clear answers and reassurance |
| Regular player | Knows the app, has a specific doubt | A fast, precise answer to an edge case |
| Hesitant or frustrated user | Worried about losing money, past bad experience | Correct policy, empathy, a clear route to support |

## 5. User experience

```
Venue page
   └── [ Ask about this venue ] button
          └── Assistant screen
                ├── Hint: "I can help with cancellation, refunds, rescheduling,
                │          amenities and venue rules for <Venue name>."
                ├── Suggested questions (tap to ask)
                │     • What is the cancellation policy?
                │     • Will I get a refund if I cancel?
                │     • Can I reschedule my booking?
                │     • What amenities are available here?
                │     • What are the venue rules?
                └── Free-text box → back-and-forth conversation
```

## 6. Scope

**In scope (v1)**
- One assistant, scoped to **one venue per conversation** (the page it was opened from)
- Multi-turn conversation
- Topics: cancellation, refunds, rescheduling, amenities, venue rules, venue address, general booking process
- Knowledge: that venue's page data + KheloMore platform policies (refund timeline, refundable amounts, injury cover, timings and lights)
- Language: English

**Out of scope (v1)**
- Access to the user's account or bookings. The assistant cannot book, cancel, reschedule or refund anything.
- **Live slot availability and live prices.** The assistant directs users to the slot picker on the page.
- **Directions.** If asked how to get there, the assistant shares the venue's full address only. It gives no routes, metro stations or travel times (§11, Q8).
- Comparing or recommending other venues
- Languages other than English

### 6.1 Venue data

**Source (prototype):** [data/venues.csv](../data/venues.csv), exported from KheloMore's venue database on 2026-10-07. It has 5 venues across 3 cities ([locality], [locality], [locality]).
**Fields:** Venue ID, Name, Full Address, Amenities, Highlights, Sports, Cancellation Policy, Reschedulable, Rescheduling Policy, Terms & Conditions, Things To Remember.
**Source (production):** the same fields, fetched live from the venue database each time a chat opens. New and updated venues work automatically.

**Findings from the data:**

| # | Finding | Impact |
|---|---|---|
| D1 | **Cancellation windows vary a lot by venue:** 2h, 4h, 24h, 48h, plus a 3-tier 50% refund at one venue | General answers would be wrong. Answers must come from venue data. |
| D2 | ~~Conflicting data at Venue D~~ **Resolved:** Venue D's T&C has a **badminton-specific** refund rule (100% at 48h+, 50% at 24–48h, 0% under 24h). The Cancellation Policy field applies to other sports. Not a conflict. | The assistant must apply sport-specific rules. R2 (real conflicts) has no example in the current data, so it will be tested with a synthetic case. |
| D3 | ~~Rescheduling policy mostly missing~~ **Resolved 2026-10-08:** the owner supplied `Reschedulable` and `Rescheduling Policy` columns. Venue A and Venue C: not reschedulable. Venue B: 18h. Venue D: 24h. Venue E: 4h. | v1 conversations ran on the old data; v2 uses the complete data |
| D4 | **No nearest metro, city field, map link or operating hours** | The "how do I get there?" suggested question can't be answered well |
| D5 | **Policy text is spread across 3 columns** (Cancellation Policy, T&C, Things To Remember), and the columns are used inconsistently between venues | The assistant must read all three |
| D6 | **One venue lists a direct phone number and a WhatsApp group** (Venue D), including "contact the facility for monthly bulk bookings" | Business decision: should the assistant send users to book directly with the venue? |
| D7 | **Highlights are empty for 2 of 5 venues** | Good sparse test cases for invented details |
| D8 | ~~Possible contradiction at Venue A~~ **Resolved:** "Free Paddles" in Highlights means paddles are free. "Rental Equipment" in Amenities covers other gear. | The assistant must use Highlights as a source of facts, not just marketing text. |

## 7. Assistant behaviour requirements

| ID | Requirement | Why it matters |
|---|---|---|
| R1 | Answers only from this venue's data **and the KheloMore platform policies** ([data/khelomore_policy.md](../data/khelomore_policy.md)) | These are the only sources of truth |
| R2 | **If a venue's data contradicts itself, the assistant treats that point as unavailable.** It says the information isn't available and shares the ticket link. It **never volunteers that the data conflicts** and never picks one version. **Exception (amended 2026-10-09):** if the *user* points out the conflict, the assistant may acknowledge it neutrally ("I can't confirm which applies for that timing") and share the ticket link. | Venue data has known conflicts (D2). Volunteering inconsistencies hurts trust; denying what the user can see sounds evasive. |
| R3 | Never invents policies, fees, timelines, amenities, metro stations or directions. If the info isn't available (e.g. a venue with no rescheduling policy), says so plainly and offers the ticket link. | A made-up refund rule or a non-existent parking lot destroys trust |
| R4 | Never claims to have done something ("I've cancelled your booking"). Explains how the user can do it. | No system access |
| R5 | Never guesses slot availability or price. Points to the slot picker on the page. | Live data isn't available to the assistant |
| R6 | Shares the **raise-a-ticket link** for payment failures, refund disputes, account-specific issues, complaints, or when the user asks for a human | These need a person or real data |
| R7 | Corrects false assumptions politely (e.g. "Since cancellation is free anytime…" when it isn't) | Agreeing with a false belief spreads misinformation |
| R8 | Asks a clarifying question when needed instead of guessing | Wrong guesses lead to wrong answers |
| R9 | Stays on this venue and KheloMore bookings. Politely declines off-topic requests. If asked about another venue, suggests opening that venue's page. | Keeps answers grounded and on-brand |
| R10 | Uses earlier turns correctly in a multi-turn chat (e.g. "what about on weekends?") and doesn't contradict its earlier answers | Multi-turn is a common failure point |
| R11 | Never asks for or repeats card numbers, CVV, OTPs or passwords | Security and fraud prevention |
| R12 | Friendly, concise, plain English. **≤ 120 words** per reply. Markdown (bold, bullets) is allowed because the chat UI renders it. | Mobile UX |
| R13 | **Never shares venue phone numbers, WhatsApp groups or other off-platform contacts**, even when they appear in venue data Saying "contact the venue directly" (e.g. for offline-only sports) is fine, as long as no contact details are given. | Keeps bookings and support on KheloMore (D6) |
| R14 | **Speaks as KheloMore** ("we", "our team"), not about KheloMore in the third person | The assistant represents KheloMore |
| R15 | **Never suggests booking outside the KheloMore app** (walk-ins, calling the venue) unless the venue data says a sport is offline-only | Keeps users and bookings on the platform |
| R16 | **Uses the ticket link only for booking-flow issues** (payments, cancellations, refunds, rescheduling, account problems). For other unanswered questions (e.g. amenity details), it says what information it has, without pushing a ticket. | Avoids needless tickets and support load |
| R17 | **No dead ends:** when handing off, says "we'll check and get back to you" and gives the link. Never leaves the user guessing what will happen. | Reassurance, not a dilemma |
| R18 | **If a venue isn't reschedulable or has no rescheduling policy:** say so, show the cancellation policy for context **without pushing the user to cancel**, and offer the ticket link to explore options. Never invent rescheduling steps or windows. | Owner rule from review |
| R19 | Uses correct sports terms (e.g. cricket and football are played on a **turf**, not a court) | Credibility |

Each requirement is also a **hypothesis about where the assistant might fail**. Error analysis will test them and may uncover failures we didn't predict.

### 7.1 KheloMore platform policies

Error analysis showed users ask about KheloMore-wide rules that no venue page covers. The owner confirmed these (full text in [data/khelomore_policy.md](../data/khelomore_policy.md)):

| Topic | Policy |
|---|---|
| Refund timing | Processed instantly by KheloMore; 2–10 days to reach the bank, depending on payment method |
| What's refundable | Convenience fee is non-refundable; slot price incl. taxes and insurance (if taken) is refundable per the venue's policy |
| Injury cover | Sports injury cover can be bought as an add-on on the booking review page; mention it whenever injuries come up |
| Timings and lights | If the venue is open and a slot is available, the user can book and play; if it's open after dark, lights are available |

Anything else about how KheloMore or its support works (e.g. ticket response times, where buttons are in the app) is **not known**. The assistant must say so and must not invent processes.

## 8. Evaluation requirements

| ID | Requirement |
|---|---|
| E1 | **Venue sample:** 5 real venues, chosen to vary in **how complete their page data is** (rich vs. sparse) and in sport and city. Sparse venues are where we expect the assistant to invent details. |
| E2 | **Test scenarios:** 40–60 scenarios built from dimensions: question topic × user type × query clarity × venue data richness. The owner reviews and approves 20 draft combinations before the rest are generated. |
| E3 | **Multi-turn tests:** each scenario is a 2–4 turn conversation. A simulated user (an LLM playing the user in that scenario) asks follow-up questions, so multi-turn failures (R10) show up. |
| E4 | **Traces:** every run saves the full conversation, venue, model, prompt version and timestamp, so results can be reproduced and compared |
| E5 | **Human review:** the owner (domain expert) labels every conversation pass/fail with a note on the first thing that went wrong |
| E6 | **Failure taxonomy:** notes are grouped into named failure modes, each with a count and a rate |
| E7 | **Automated checks:** at least one code check (e.g. word limit, ticket link present when required) and at least one LLM-judge check (e.g. "invented a fact not in the venue data") |
| E8 | **Judge validation:** the judge's verdicts are compared against the owner's labels and its agreement is reported. A judge that hasn't been validated isn't trusted. |
| E9 | **Improvement loop:** change the prompt (v1 → v2), rerun the same scenarios and report before vs. after per failure mode |

**Rule:** quality metrics are defined **after** reading the conversations, not before.

**Labelling standard (pass/fail), as applied by the owner:** a conversation **fails** only if a reply is **critical**: it would likely make the customer take a wrong action or lose money or a booking. Examples from the v1 labels: a wrong or invented refund, cancellation or rescheduling rule or step (seed-02, seed-03, seed-05); a wrong ticket URL (gen-24); pushing the user to book outside the app (gen-08); withholding a clear policy answer by claiming the rules conflict (gen-19). Everything else **passes**, with a note for non-critical issues (minor invented details, hedging, tone, missing extras).

**Synthetic data:** the Test Venue (synthetic) test venue ([data/venues_test.csv](../data/venues_test.csv)) is invented. It is used only in evaluation, never shown to real users, and is labelled as synthetic in all results.

## 9. Success criteria

**The project is complete when:**
- [ ] All E1–E9 deliverables exist in the repo
- [ ] The README presents the results as a PM report: problem, method, failure modes, fix, before/after, recommendation
- [ ] Code and data live in a **private** GitHub repo
- [ ] A public write-up shares **only the method and results**, with venue names anonymized and no raw venue data

**Proposed launch bar** (to be refined after error analysis):
- Invented facts (R3): **0** in the test set
- Guessed slot availability or price (R5): **0**
- Asked for sensitive data (R11): **0**
- Other failure-mode targets: set after the error analysis milestone (M4)

**Post-launch product metrics** (for the live rollout, not this project):
- Share of conversations resolved without a support ticket
- Booking conversion: venue-page sessions *with* the assistant vs. without
- Thumbs-up/down rate on answers

## 10. Technical approach & cost

| Decision | Choice | Reason |
|---|---|---|
| Language | Python 3.9 (already installed) | Simple, standard for evals |
| Assistant model | **Claude Sonnet 5.5** (`claude-sonnet-5-5`), prompt v2.2, thinking off, low effort. Chosen 2026-10-09 over Haiku 4.5 by eval: in v2, Haiku flagged for unsupported claims in 49% of replies vs Sonnet 21%, and Haiku made a critical reschedule-deadline error. | Quality over price; cost controlled by caching, short replies and stored answers |
| Judge & simulated-user model | Claude Sonnet 5.5 (`claude-sonnet-5-5`): $2 / $10 per million tokens | A stronger model grades a cheaper one |
| Knowledge delivery | The venue's data placed in the prompt for each conversation | Simplest approach. No search system is needed at this size. |
| Data format | JSONL files in the repo | Readable, git-diffable |
| Secrets | `ANTHROPIC_API_KEY` environment variable; `.env` is git-ignored | The key never enters the repo |

### Cost estimate: live usage at 400 users/day

**Assumptions per conversation:** about 2,000 tokens of context sent with every turn (instructions ~1,000 + venue data ~1,000), ~50-token user messages, ~150-token replies, and conversation history that grows with each turn.

| Scenario | Turns | Cost / conversation | Per day (400) | Per month |
|---|---|---|---|---|
| Light: taps 1–2 suggested questions | 2 | ~$0.006 | ~$2.30 | **~$70** |
| **Typical** | 3 | ~$0.009 | ~$3.60 | **~$110** |
| Heavy: long back-and-forth | 5 | ~$0.016 | ~$6.40 | **~$190** |

- About 75% of the cost is the ~2,000-token context re-sent on every turn. Output tokens make up the rest.
- Phase 2 (KheloMore feature docs) will add context, so costs will rise. Re-estimate then.
- Switching the assistant to Sonnet 5.5 would roughly **double** these numbers.
- **Cost levers for launch:**
  - **Pre-generate answers to the 5 suggested questions** for each venue (2,000 venues × 5 answers, regenerated when the venue page changes). Taps then cost nothing and give identical answers for every user. Only free-text questions call the model.
  - **Prompt caching:** within a conversation, turns 2+ can re-read the context at ~10% of the input price, if the context is ≥ 4,096 tokens (Haiku's minimum) and the user replies within 5 minutes. **At ~2,000 tokens in v1, caching won't apply.** It becomes relevant in Phase 2.
  - **Cap turns per conversation** (e.g. 10) to limit abuse.
- **Not included:** a maps/directions API, if added later (§11, Q8), and hosting.

### Cost estimate: this eval project

About 60 conversations × 3–4 turns, each run with the assistant, the simulated user and the judge, repeated for v1 and v2: **about $5–10 in total.**

### Planned repo structure
```
docs/requirements.md      ← this document
data/venues.csv           ← 5 sample venues (public venue-page fields only)
prompts/                  ← assistant, simulated-user and judge prompts, versioned
src/                      ← scripts: generate scenarios, run conversations, run evals
data/scenarios.jsonl      ← test set
results/                  ← traces, labels, eval reports per run
README.md                 ← the portfolio-facing report
```

## 11. Open decisions

| # | Question | Proposed default | Status |
|---|---|---|---|
| Q1 | Policy source | Venue page data only; there is no KheloMore-wide T&C | ✅ Decided |
| Q2 | Language | English | ✅ Decided |
| Q3 | Handoff | https://www.khelomore.com/sports-venues/raise-a-request | ✅ Decided |
| Q4 | Conversation style | Multi-turn | ✅ Decided |
| Q5 | Max reply length | 120 words | ✅ Decided |
| Q6 | Models | Haiku 4.5 assistant, Sonnet 5.5 judge | ✅ Decided |
| Q7 | When a venue's own data conflicts (D2), what should the assistant do? | Say the information isn't available and share the ticket link. Never mention the conflict. | ✅ Decided |
| Q8 | Directions | Drop the directions suggestion; share the address if asked | ✅ Decided |
| Q9 | Metro, map link and hours in the export (D4)? | Not needed for v1 | ✅ Decided |
| Q10 | Final list of suggested questions | Cancellation, refund, reschedule, amenities, venue rules (§5). Revisit after M4. | 🟡 Proposed |
| Q11 | Which 5 sample venues? | The 5 in `data/venues.csv` | ✅ Decided |
| Q12 | Rescheduling policy (D3) | Varies per venue; many venues have none. If missing, the assistant says so and shares the ticket link. | ✅ Decided |
| Q13 | Share venue phone numbers and WhatsApp links (D6)? | No (R13) | ✅ Decided |
| Q14 | Publishing | Private repo; publish only the method and results | ✅ Decided |
| Q15 | KheloMore-wide policy text | Refund timing, refundable amounts, injury cover, timings/lights confirmed (§7.1). Ticket response time and in-app steps are unknown. | ✅ Resolved |

## 12. Roadmap after this project

| Phase | Scope |
|---|---|
| **Phase 2** | Add KheloMore product docs (how Passes work, how hosted games work, etc.) so the assistant can answer platform-level questions. Extend the test set and re-run the eval. |
| Later | Directions/maps, Hindi/Hinglish, access to the user's own bookings |

## 13. Milestones

| # | Milestone | Output |
|---|---|---|
| M0 | Requirements locked | This doc at v1.0, §11 resolved |
| M1 | Assistant v1 | Policy + 5 venue files + prompt v1 + chat script |
| M2 | Test set | Approved dimensions & scenarios |
| M3 | Run v1 | Multi-turn conversation traces |
| M4 | Error analysis | Human labels + failure taxonomy with rates |
| M5 | Automated evals | Code check(s) + LLM judge, judge validated |
| M6 | Assistant v2 | Prompt fix targeting the top failure modes |
| M7 | Re-run & compare | Before/after table |
| M8 | Publish | Private GitHub repo + public write-up of method & results |

## 14. Decision log

| Date | Decision | By |
|---|---|---|
| 2026-10-07 | Domain: KheloMore venue-page assistant | Owner |
| 2026-10-07 | Calls the Claude API; key stored as an environment variable, never committed | Owner |
| 2026-10-07 | Knowledge = venue page; English; multi-turn; ≤120 words; ticket-link handoff | Owner |
| 2026-10-07 | Models: Haiku 4.5 (assistant), Sonnet 5.5 (judge) | Owner |
| 2026-10-07 | Self-contradicting venue data → KheloMore standard policy + ticket link | Owner |
| 2026-10-07 | No directions feature; address only. Reschedule policy is per venue; missing → ticket link. | Owner |
| 2026-10-07 | Never share venue phone numbers or off-platform contacts | Owner |
| 2026-10-07 | Repo private; publish only method & results | Owner |
| 2026-10-07 | **Requirements v1.0 locked** | Owner |
| 2026-10-07 | No KheloMore-wide T&C; ticket URL confirmed; KheloMore feature docs moved to Phase 2 | Owner |
| 2026-10-07 | Conflicting venue data → "information not available" + ticket link; never mention the conflict | Owner |
| 2026-10-07 | Venue D 50% rule is badminton-specific (not a conflict); "contact the venue directly" allowed without details; markdown allowed | Owner |
| 2026-10-07 | Venue A paddles are free (Highlights count as venue facts) | Owner |
| 2026-10-08 | Labelling standard: misleading/harmful reply = Fail | Owner |
| 2026-10-08 | KheloMore platform policies added (§7.1) | Owner |
| 2026-10-08 | Test Venue (synthetic) synthetic venue kept for evaluation only | Owner |
| 2026-10-08 | Rescheduling columns added to venue data; 4 scenarios' expected behaviour updated | Owner |
| 2026-10-09 | Error analysis complete (47/49): R14–R19 added; support response time and app navigation added to platform policy | Owner |
| 2026-10-09 | Labelling standard: Fail = critical responses only (owner's applied standard) | Owner |
| 2026-10-09 | Assistant model = Claude Sonnet 5.5 (prompt v2.2, enforced 120-word limit) | Owner |
| 2026-10-09 | R2 amended: neutral acknowledgement allowed when the user raises the conflict | Owner |
| 2026-10-09 | Founder sign-off to publish the case study naming KheloMore (venue names and customer data still excluded); code repo stays private | Owner |
