# Verisphere surface glossary and display standard

**Status:** normative for first-party surfaces (web app, browser extension); recommended
for third-party integrations. Layout and interaction may differ per surface; **words,
numbers, and colors must not.**

## 1. Names

| Term | Use exactly | Never |
|---|---|---|
| The protocol / company / product | **Verisphere** | VeriSphere, Verity (as a product name) |
| Token | **VSP** | $VSP in UI copy, "VSP tokens" (redundant after first use) |
| The score | **Verity Score**, abbreviated **VS** | Truth score, trust score, veracity score |
| A stakeable statement | **claim** | post (internal id only), sentence (UI-side only) |
| Backing a claim as true | **support** | upvote, agree, back, stake for |
| Backing a claim as false | **challenge** | downvote, disagree, dispute, stake against |
| Money at risk on a claim | **stake** (noun/verb) | bet, wager, position (finance sense) |
| Relationship between claims | **link** — a *supporting* link or a *challenging* link | edge, citation |
| Removing a stake | **withdraw** | unstake, exit, sell |
| Fee-free transaction path | **gasless** ("no gas needed") | free, sponsored, relayed (internal) |

Identifiers stay as the protocol names them (`evs`, `verity_score`, `postId`) — the table
governs *copy shown to people*, not code.

## 2. Verity Score display

- Scale is **−100 … +100**. Show the sign always (`+42`, `−17`, `0`), no decimals in chips,
  one decimal in detail views.
- **Effective** VS is what users see by default. Label the base (local) score only in detail
  views, as "base".
- Color comes from the shared `vsColor` ramp (frontend `src/ui/vsColor.ts`, mirrored in the
  extension `src/shared/vsColor.ts`): support-green through neutral to challenge-red.
  Do not re-derive colors per surface.
- A claim with no stake shows **"no stake yet"**, never `0` (0 is a real, contested score).

## 3. Amounts

- VSP: up to 4 decimals in inputs, 2 in summaries, thousands separators; unit after the
  number (`1,250.00 VSP`). Never scientific notation.
- USDC: 2 decimals. Prices as `$1.1085/VSP`.
- Percentages: one decimal (`+4.2%`); price impact as `~9.1%`.

## 4. Actions and buttons

- Primary verbs: **Support**, **Challenge**, **Stake**, **Withdraw**, **Buy**, **Sell**.
- Creating a claim from text: **"Create & Challenge"** / **"Create & Support"** (never
  "register", "submit", "mint").
- Wallet: **Connect** / **Disconnect**; show the truncated address `0x99…1810` in the pill.
- Progress steps use the same three labels everywhere:
  *Create claim on-chain → Stake → Insert into article* (extension: *… → Show on page*).

## 5. Disclosures (verbatim, every surface that shows a score)

> Verisphere scores are staked market positions, not authoritative fact rulings.

And on any trade surface:

> Executes on the public pool from your own wallet — the site never holds your funds.

## 6. Design tokens

Brand accent indigo `#4f46e5`; support/challenge colors and the VS ramp are exported from
the frontend tokens and copied verbatim into the extension (`src/shared/tokens.ts`). A
surface that wants a different palette should still map support/challenge/VS to the shared
semantic colors.

## 7. Change control

Edits to this file accompany the code change that needs them, in the same PR window.
Third-party surfaces are asked to follow §1–§2 and §5 exactly and may diverge elsewhere.
