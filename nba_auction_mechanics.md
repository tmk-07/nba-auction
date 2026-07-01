# NBA Auction — Mechanics Breakdown

This document explains exactly how the three core systems work as implemented in `nba_auction_mvp.html`.

---

## 1. Talent Pool Construction

### Player tiers
Every real player in the master list (46 players) belongs to one of 5 tiers, each with a **base value**:

| Tier | Base Value | Base Hype (stars) |
|---|---|---|
| Superstar | 95 | 5 |
| Star | 72 | 4 |
| Starter | 50 | 3 |
| Role | 30 | 2 |
| Bench | 15 | 1 |

### Pool profile (talent density)
At the start of every game, one **profile** is chosen at random. Each profile specifies how many players to pull from each tier — this is what makes some drafts stacked with stars and others flat:

| Profile | Superstar | Star | Starter | Role | Bench |
|---|---|---|---|---|---|
| Superteam Showcase | 3 | 3 | 2 | 1 | 1 |
| Balanced Class | 1 | 2 | 3 | 2 | 2 |
| Flat & Even | 0 | 1 | 4 | 3 | 2 |
| Top Heavy & Thin | 2 | 1 | 1 | 3 | 3 |
| Deep Sleeper Class | 0 | 2 | 2 | 3 | 3 |

For each tier, that many players are randomly sampled (no repeats) from the master list for that tier, then the whole 10-player set is shuffled into the **reveal order** — the sequence players will come up for bidding, one at a time. Neither the human nor the bot ever sees this order in advance.

### Hidden true value (the "sleeper/bust" mechanic)
Each selected player gets a hidden `trueValue`, which is what actually decides the draft winner at the end:

```
trueValue = round(tierBaseValue × randomFactor)
randomFactor = random between 0.65 and 1.35
```

So a Bench-tier player (base 15) can land anywhere from ~10 to ~20, while a Superstar (base 95) can land anywhere from ~62 to ~128. This is the scouting-uncertainty layer — a player's tier gives you a rough idea of quality, but the exact number is never shown and has real spread, so a "lesser" player can occasionally out-value a "better" one.

### Visible hype (stars)
```
hype = clamp(tierBaseHype ± 1 [25% chance], min 1, max 5)
```
Hype stars are the only star-power signal shown on the card. They mostly track tier but get a random ±1 nudge a quarter of the time, so hype is a strong hint, not a guarantee, of true value.

### Visible stats (PPG / RPG / APG / DEF)
These are **not randomized** — they're fixed, illustrative stat lines hardcoded per real player (approximate, for gameplay feel, not live season data). DEF is a 1–10 defensive rating based on real defensive reputation, independent of scoring tier (e.g. Jrue Holiday and Herbert Jones grade high despite modest counting stats).

---

## 2. Bot Strategy — How It Decides What to Bid

### The bot never sees hidden true value
This is the core fairness rule: the bot estimates a player's worth from the exact same visible information the human sees (hype + PPG + RPG + APG), never from the hidden `trueValue`. Both sides are guessing.

### Step 1 — Bot's raw estimate
```
estimate = hype×16 + ppg×1.4 + rpg×1.1 + apg×1.3
estimate ×= persona.starBias   (only applied if hype ≥ 4 stars)
estimate ×= (1 + random between -persona.noise and +persona.noise)
```

### Step 2 — Persona (randomized each game, one of 4)

| Persona | starBias | budgetDiscipline | noise | Effect |
|---|---|---|---|---|
| Star Chaser | 1.18 | 1.00 | ±15% | Overvalues 4–5 star hype players, otherwise standard budgeting |
| Value Hunter | 0.92 | 0.82 | ±12% | Slightly undervalues hype, and spends more conservatively overall |
| Balanced Builder | 1.00 | 1.00 | ±12% | No bias, moderate randomness — a "fair" opponent |
| Sleeper Believer | 1.00 | 0.95 | ±30% | No systematic bias, but very high estimate volatility — wildly under/overbids at random |

`noise` is applied fresh on **every single bid decision** during an auction, not once per player — so the bot's willingness can wobble bid-to-bid, not just player-to-player.

### Step 3 — Willingness and hard cap
```
willingness = round(clamp(estimate / 11, 1, 15) × persona.budgetDiscipline)
maxBid(side) = budget[side] - (slots_remaining[side] - 1)
             = 0 if slots_remaining[side] is 0
cap = min(willingness, maxBid('bot'))
```
`maxBid` is the hard budget-safety rule for **either side**: it always keeps at least $1 reserved for every other roster slot still open, so nobody can bid themselves out of being able to fill their roster. `willingness` is the bot's personal ceiling based on perceived value; `cap` is whichever is lower.

### Step 4 — Raise or pass, each turn
On its turn, the bot considers raising by $1:
```
nextBid = currentBid + 1
if nextBid > cap → bot passes immediately
```
If it can afford to raise, it doesn't always push through — it has a chance to back off early even within its own cap, to simulate hesitation/bluffing rather than always bidding to the exact limit:
```
pushThrough probability = clamp(0.9 - (nextBid / cap) × 0.4, 0.35, 0.9)
```
In plain terms: early in the bidding (nextBid far below cap) the bot pushes through ~90% of the time; as the price climbs toward its own ceiling, that probability drops toward ~35–50%, even if it could technically still afford more. This is what makes the bot feel like it's "cooling off" rather than robotically bidding to a hard line every time.

If the random roll fails, or the bid exceeds `cap`, the bot passes and the auction resolves to the human at the last bid.

---

## 3. Talent Score & Pentagon

At game end, true values are never shown directly. Instead the game computes:

### A) Positional value adjustment (applies before anything else)
Each player was auto-slotted into a position (G/G/F/F/C) as they were drafted — into their natural position if open, otherwise the nearest open slot. The penalty is based on distance along the line G(0) – F(1) – C(2):

| Distance from natural position | Penalty | Multiplier |
|---|---|---|
| 0 (natural) | none | ×1.00 |
| 1 (e.g. G↔F or F↔C) | small | ×0.92 |
| 2 (G↔C) | larger | ×0.78 |

```
adjustedValue(player) = round(trueValue × multiplier)
```
This `adjustedValue` — not raw `trueValue` — is what feeds both the Talent Score and (indirectly) team strength.

### B) Overall Talent Score (0–100)
```
top5Raw = sum of the 5 highest trueValues among all 10 players in this draft
teamAdjustedTotal = sum of adjustedValue across a team's 5 players
TalentScore = clamp(round(teamAdjustedTotal / top5Raw × 100), 0, 100)
```
`top5Raw` is the theoretical best possible 5-player haul from *this specific pool* (position-penalty-free). Scaling against it — rather than a fixed number — keeps scores meaningful whether the pool was stacked or flat: getting 80/100 in a weak "Flat & Even" class and getting 80/100 in a "Superteam Showcase" class both mean roughly the same thing — you got about 80% of the best possible talent available that game. The winner is whichever team has the higher `teamAdjustedTotal` (the displayed rounded score is just for presentation; ties are only declared if the rounded scores match).

### C) Pentagon — 5 axes, each scaled 0–100
For 4 of the 5 axes, the team's raw sum on that stat is scaled against the sum of the **top 5 players in the whole pool** on that same stat (same "best possible" logic as Talent Score):

| Axis | Team value | Scaled against |
|---|---|---|
| Scoring | sum of team PPG | top 5 pool PPG values, summed |
| Rebounding | sum of team RPG | top 5 pool RPG values, summed |
| Playmaking | sum of team APG | top 5 pool APG values, summed |
| Defense | sum of team DEF (1–10 each) | top 5 pool DEF values, summed |
| Star Power | sum of team hype (1–5 each) | fixed max of 25 (5 players × 5 stars) |

```
axisScore = clamp(round(teamStatSum / maxStatSum × 100), 0, 100)
```

Star Power is the one exception — it's scaled against a fixed ceiling (25) rather than the pool's actual top 5, since hype is already bounded 1–5 by design regardless of pool profile.

Both teams' 5 axis scores are plotted as two overlapping polygons on the same radar chart, giving a visual "team shape" comparison (e.g. a defense-and-rebounding team vs. a scoring-and-playmaking team) separate from the single headline Talent Score number.

---

## Quick reference — what's hidden vs. visible

| Data | Visible during draft? | Used by bot? | Shown at end? |
|---|---|---|---|
| PPG / RPG / APG / DEF | ✅ Yes | ✅ Yes | ✅ Yes (via pentagon axes) |
| Hype (stars) | ✅ Yes | ✅ Yes | ✅ Yes (Star Power axis) |
| trueValue | ❌ Never | ❌ Never | ❌ Never shown directly (only via derived Talent Score) |
| adjustedValue | ❌ Never | ❌ Never | ❌ Never shown directly (only via derived Talent Score) |
