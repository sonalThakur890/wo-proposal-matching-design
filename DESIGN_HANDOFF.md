# WO-to-Proposal Matching (Tier 2) — Qualify Ticket UI

Design handoff for the **Link Proposal** enhancement inside the Control Center **Qualify Ticket (QT)** task. This packages the interactive prototype and every decision behind it so it can be implemented and version-controlled.

- **Prototype:** `qt_proposal_match_all_states.html` (single self-contained file — open in a browser; use the dark bar at the top to switch states).
- **Requirement:** Confluence — *Requirements: WO-to-Proposal Matching Agent (Tier 2)*, FUL space, page `4363583489`.
- **Owning surfaces:** `dmg.cc-tasks.ui` / `dmg.fulfilment.ui` (new QT task type: **Link Proposal**). Intelligence produced by IRIS (`dmg.fulfilment-intelligence.iris`); link/approve/attribution handled by `ticket-service` + `proposal-service`.

---

## What this is

When a customer approves a DMG proposal, it often arrives as a **brand-new work order with no reference to the original proposal**, so we lose attribution and can't measure proposal→approval conversion. The Tier 2 agent runs during ticket qualification, detects when an incoming WO likely came from a submitted proposal, and surfaces it to the NAT inside the existing Qualify Ticket panel.

This design covers the **NAT-facing UI** for every outcome of that match, as an **inline block inside the Qualify Ticket panel** (no new page/modal), rendered in the current DMG design system.

---

## Match signals (per requirement, strongest → weakest)

1. **NTE proximity** — WO NTE vs proposal customer price (strongest)
2. **Service line match** — same trade (Plumbing, HVAC, …)
3. **Scope similarity** — semantic compare of WO description vs proposal title/scope
4. **Timing** — recency of proposal submission

The UI surfaces these as compact chips (✓ green = strong, ≈ amber = partial). Every match screen shows the same parameter set: **Same store · Same trade · NTE proximity · Scope match**.

---

## States (5 outcomes → switcher in the prototype)

| State | Trigger (from doc) | NAT UI |
|---|---|---|
| **Auto-linked** | Property + service + NTE within 1% | Proposal already linked, qualification pre-filled & locked. NAT reviews → **Confirm & qualify** or **Unlink**. |
| **Confirm (single)** | Property + ≥1 signal, not all 3 | One suggested proposal → **Link & qualify** or **Not a match**. |
| **Choose (multiple)** | Several proposals score medium+ | Ranked, radio-select list → pick one → **Link & qualify**, or **None of these**. |
| **Low** | Property match only | No task surfaces; logged for training. Normal qualification. |
| **No match** | No submitted proposals in lookback | Nothing; normal qualification. |

---

## Behavior rules (apply to all match screens)

1. **Qualification actions are gated.** While a match decision is pending, the QT action buttons (Update Ticket and Continue, Update NTE, Save & Next, Qualification Done/Ready for Jobs) are **greyed/disabled**. They enable only when the NAT rejects the match (Unlink / Not a match / None of these) and goes manual.
2. **Link / Confirm → auto-qualify.** A confirmation step first states the outcome:
   > We'll mark **PRP-XXXXXX** approved and auto-qualify this ticket. A job will be created and posted to the preferred provider — **[Provider]**. If they decline or aren't available, it moves to the job board.
   - If the proposal has **no preferred provider**, the job posts to the **job board** (message adjusts).
   - On confirm: proposal → **Approved**, attribution written, ticket qualified, **task status flips In Progress → Done**.
3. **Dismiss → manual qualification.** Rejecting the match enables the normal QT form; nothing is linked.
4. **Customer NTE is NOT copied from the proposal.** The proposal is the source of truth for **scope/service/type of work** (pre-filled + locked), but **Customer NTE keeps the work order's own value** and stays **editable** — the customer may send a WO with a different/updated price they intend to keep.
5. **Reason capture is only on Auto-link → Unlink**, as **free text (up to 150 words)** — because unlinking overrides an automated decision and feeds model training. Single/Multiple dismiss is one-click, no reason needed.
6. **View proposal** link on every proposal, showing the originating ticket (proposal was submitted on a different WO), e.g. `View proposal ↗ · submitted on TKT-260401-3391`.

### Auto-link Unlink specifics
- Inline confirm warns the pre-filled qualification will be cleared and the ticket becomes standard.
- On unlink: locked fields reset & unlocked, proposal returns to its previous ticket, **Customer NTE is preserved** (not zeroed), override + free-text reason logged.

---

## Copy (final strings)

- **Titles:** Auto-linked → `Auto-linked proposal`; Single → `Did this come from a proposal?`; Multiple → `Did this come from one of these proposals?`
- **Shared say-line skeleton:** *"This work order looks like it came from a proposal your team already sent…"* + type-specific action clause.
- **Confidence chip:** `PLEASE REVIEW` (auto), `LIKELY MATCH` (single/multiple).
- **Outcomes block (2 lines, every screen):** ✓ primary (link → qualify + job) · ↩ secondary (dismiss/unlink → manual).

---

## Design system notes

- Built to the current Control Center look: DMG navy top bar (`#123A5E`) with orange accent (`#EE7127`), Poppins headings + Inter body, orange step accent bars, underline inputs, navy radios/buttons.
- Maps to **dmgKit** (Material UI based): match card ≈ `Card`, chips ≈ `Chip` (severity), buttons ≈ `Button`/`PrimaryButton`, selectable list ≈ `RadioCard`, confirm boxes ≈ inline `Alert`.
- ⚠️ **Colors/fonts are approximated** from a product screenshot; exact dmgKit theme tokens (hex/spacing/radii) are **not exposed via MCP**. Before building, swap the CSS `:root` variables for the real dmgKit token export for pixel fidelity.

---

## Implementation notes (for engineers)

- **IRIS** returns a structured `proposal_match`: `action` (`auto_link | confirm | multi_confirm | log_only | no_match`), candidate list (id, confidence, rationale, signals), and target proposal id for auto_link/confirm.
- **ticket-service:** add WO **NTE** to the IRIS qualification payload; read `proposal_match`; for auto_link execute the link + emit event + write attribution; for confirm/multi_confirm create the **Link Proposal** task pre-populated; log-only/no-match continue.
- **proposal-service:** on link-confirmed event → proposal `Submitted → Approved` + write attribution; signal preferred provider (from ERQ) to assignment.
- **Attribution/approval fire on the NAT's confirm**, not on auto-link, so **Unlink is a cheap, reversible rollback** (proposal returns to prior ticket).

---

## Repo suggestion

```
design/wo-proposal-matching/
├── qt_proposal_match_all_states.html   # interactive prototype (all 5 states)
└── DESIGN_HANDOFF.md                   # this doc
```

Prototype is dependency-free except Google Fonts (Poppins/Inter) via CDN — no build step. Open the HTML directly to review.
