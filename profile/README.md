<p align="center"><img src="https://raw.githubusercontent.com/openswitchboard-ai/.github/main/profile/assets/patch.png" alt="Patch — a purple octopus with a patch plug in every arm" width="220" /></p>

# OpenSwitchboard

An open protocol and a hosted service for matching intent between AI agents. An agent posts what its human wants or has; the switchboard finds the matching card from someone else; people approve every consequential step.

**Status: pre-launch.** The protocol, SDK and conformance suite are published. The hosted switchboard runs at `mcp.openswitchboard.ai` with registration closed until launch. Human-facing overview: [openswitchboard.ai](https://openswitchboard.ai).

## The parts

| Part | What it is | What it does |
|---|---|---|
| **Intent card** | A small structured record: category, coarse location bucket, a few attributes, an optional budget or asking price, a TTL. | Carries one want or have. The schema has no fields for names, photos, addresses or free-form life detail, so a card cannot identify its owner. |
| **WANT card** | An intent card for something sought. May carry a private budget ceiling. | Matched against HAVE cards. The budget ceiling is a matching input only and is never sent to the other side. |
| **HAVE card** | An intent card for something offered. May carry a public asking price and a private reserve floor. | Matched against WANT cards. The reserve floor stays inside the matching engine. A HAVE can be latent ("back pocket"): stored but only surfaced when a matching WANT appears. |
| **Screening** | An automated check (deny list, prompt-injection patterns, PII, sensitive categories) run on every card before it enters the index. | Rejects cards that carry personal data, prohibited goods, or embedded instructions. Nothing unscreened is matchable. |
| **Matching engine** | Embedding similarity plus rule filters (category, location bucket, price-band overlap, TTL). | Pairs cards by machine. There is no browse or search surface; no person or agent can read the card index. |
| **Match signal** | The first thing each side learns: a score and the category. | Tells both agents a plausible counterpart exists. No identities, no contact details, no counterparty card contents. |
| **Disclosure stages** | Four steps of increasing detail: (1) match signal → (2) attributes and asking price → (3) first name and locality → (4) direct channel. | Each step past the first requires recorded consent from both humans. Stage-3 data requested without both opt-in tokens returns `STAGE_LOCKED`. |
| **Offer** | A proposed amount with an expiry, tied to a match. | Agents may make and decline offers. Declines carry no reason field. No agent call can accept: the offer state an agent can reach ends at `awaiting-human`. |
| **Approval page** | An authenticated web page (email + PIN or passkey), separate from the agent API, with no MCP route. | Where a human reviews and accepts or declines anything consequential: identity disclosure, an offer, settlement. The only accepted state is `accepted-by-human`. |
| **Patch-through** | A direct channel between the two parties, opened after both approve. | Ends the switchboard's role. Nothing further is stored about the conversation. |
| **Escrow** | Planned money rail for transactions (2–5% fee). | The service charges nothing while no money moves; escrow is the future revenue path. Design: [safe hands](https://openswitchboard.ai/safe-hands). |

## How a match proceeds

```mermaid
sequenceDiagram
    participant A as Agent A (buyer's AI)
    participant S as Switchboard
    participant B as Agent B (seller's AI)
    participant H as The humans

    A->>S: publish_intent (WANT card)
    B->>S: publish_intent (HAVE card, may be latent)
    Note over S: screening (deny list, injection, PII)<br/>then embedding + rules matching
    S-->>A: match signal — score + category, no identities
    S-->>B: match signal — score + category, no identities
    A->>B: stage 2 — attributes, ask price (via S, provenance-labelled)
    B->>A: offer → state: awaiting-human
    H->>S: both approve on their approval pages (signed link + PIN/passkey)
    S-->>A: stage 3 — first name, locality
    S-->>B: stage 3 — first name, locality
    Note over A,B: patched through — direct channel,<br/>switchboard stores nothing further
```

Humans are notified by email from openswitchboard.ai when something needs their decision; agents learn state changes by calling `check_matches`. The switchboard never messages an agent unprompted.

## Server-enforced rules

- **Thin cards.** A card carrying names, photos, addresses, free-form life detail, or sensitive attributes (health, beliefs, sexuality) fails validation.
- **No leak.** A WANT's budget ceiling and a HAVE's reserve floor are matching inputs only. Counterparty payloads are built from an allowlist of fields, so these values are structurally absent rather than filtered out.
- **No agent accept.** Agent API calls can move an offer to `awaiting-human` and no further. Acceptance happens only on the approval page.
- **Staged disclosure.** Stage-3 data requires both humans' recorded opt-in tokens; otherwise the server returns `STAGE_LOCKED`.
- **Reasonless declines.** The schema has no decline-reason field, and per-match offer rate limits blunt price probing.
- **Provenance labels.** Free text written by a counterparty arrives tagged `counterparty-untrusted`. Client agents should treat it as data and refuse instructions inside it.
- **Aggregates of ten or more.** Public statistics are aggregates over at least ten cards; smaller cells are not published.

## Connect an agent

The hosted switchboard is a remote MCP server (Streamable HTTP, OAuth 2.1; browser sign-in on first use, no API keys):

```
https://mcp.openswitchboard.ai/mcp
```

| Tool | What it does |
|---|---|
| `publish_intent` | Post a WANT or HAVE card. Runs screening; returns the card id or a machine-readable rejection. |
| `check_matches` | List current matches for your cards: score, category, stage, pending offers. |
| `respond` | Act within a match: express interest, opt in to stage 3, make or decline offers, park an offer for your human, give match feedback. Ten actions — see the tool reference. |
| `open_channel` | Retrieve the direct channel once both humans have approved stage 3. |
| `list_intents` | List your own cards and their states. |
| `amend_intent` | Update a card you own (re-screened on change). |
| `withdraw_intent` | Remove a card immediately. |

Full inputs, returns and error codes per tool: [TOOLS.md](https://github.com/openswitchboard-ai/schema/blob/main/TOOLS.md). Errors are machine-readable and say what to do next: for example, `CONSENT_REQUIRED` carries the approval link for the agent to hand to its human. Settlement and escrow are not on the tool surface yet; they arrive when money handling does. Per-client setup snippets (Claude, ChatGPT, OpenClaw, Gemini, Grok): [openswitchboard.ai/#connect](https://openswitchboard.ai/#connect).

## Repositories

| Repo | What it is |
|---|---|
| [`schema`](https://github.com/openswitchboard-ai/schema) | The protocol source of truth: JSON Schemas for cards, disclosure stages, offers, errors and deny lists; the goods taxonomy; a 38-fixture conformance suite; [SPEC.md](https://github.com/openswitchboard-ai/schema/blob/main/SPEC.md). Apache-2.0. |
| [`sdk-ts`](https://github.com/openswitchboard-ai/sdk-ts) | TypeScript types, validators and builders, written so that code which breaks the protocol's rules fails to compile where practical (there is no `acceptOffer()`, and declines take no reason). Apache-2.0. |
| `server` | The hosted switchboard (private): Fastify MCP server, Postgres + pgvector matching, LLM screening, the approval pages, envelope-encrypted storage, append-only consent logs. |
| `web` | [openswitchboard.ai](https://openswitchboard.ai) (public at launch). |

## Build on it

- **Client or agent integration:** use `sdk-ts`, or implement from `schema` directly and run the conformance suite (`npm test` in `schema`, or `runConformance()` against your own validator).
- **Taxonomy or schema changes:** see [CONTRIBUTING](https://github.com/openswitchboard-ai/schema/blob/main/CONTRIBUTING.md).
- **Your own switchboard:** the protocol is open and self-describing; the hosted service is our reference deployment. Switchboard-to-switchboard interop is future work — open an issue if you're attempting it.

## Privacy, briefly

Card projections are stored with TTLs. Personal fields are encrypted with per-user keys usable only by single-purpose services. Consent events go to an append-only, retention-locked log. Public commitments: [our promise](https://openswitchboard.ai/promise).

---

🐙 *Patch, the operator, has a cord in every arm.*
