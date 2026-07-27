# True Cost Math

The ranking logic. Needs no API, costs nothing, and is the reason the output is worth reading.

---

## Core Concepts

### The three-layer cost stack

Cruise pricing hides money in two places, and booking sites show you one of them at most.

```
LAYER 1   Base fare              advertised everywhere
LAYER 2   Taxes & port fees      in the data, buried on most sites
LAYER 3   Gratuities             in NO data source, $16-25/pp/day
```

A £389 cruise for two, 7 nights:

```
Fare                £389.50
Taxes & port fees    £92.00      +24%
Gratuities          £252.00      +65%
                    ────────
True total          £733.50      88% above the advertised number
```

Surfacing layer 3 is not secret access. It's arithmetic nobody performs.

### The formula

```
true_total     = price_amount
               + taxes_and_fees
               + (gratuity_rate × nights × passengers)
               - obc_amount
               + flight_home          (one-way sailings only)

true_per_night = true_total / nights
```

Rank on `true_per_night`, per person at double occupancy.

### Never use the actor's `price_per_night`

**Verified live 2026-07-25.** `cruise-data-feed` returns `price_per_night` computed from the fare alone:

```
Royal Caribbean Grandeur, Colón → San Juan, 7 nights
  actor price_per_night   £55.64    = 389.50 / 7
  with taxes              £68.79    = (389.50 + 92) / 7
```

24% understated before gratuities are even considered. Always recompute.

### Null is not zero

**Found in live testing 2026-07-25.** The NCL Ravenna → Philadelphia sailing returned `taxes_and_fees: null`. Treated as zero, it ranked **first** at $68.75/night, ahead of every sailing whose taxes were actually reported.

A missing field is not a free cruise. Treating null as zero systematically promotes records with incomplete data to the top of the ranking, which is the worst possible failure mode: the least trustworthy result wins.

**Rules:**
1. `taxes_and_fees: null` → the total is **incomplete**. Say so.
2. **Do not rank an incomplete record above complete ones.** Either exclude it from the ranking, or rank it using a typical-tax estimate and mark it clearly.
3. Typical port taxes run 15-25% of fare on mass-market lines. If estimating, use 20% and label it: `taxes estimated, not reported by source`.
4. Applying that to the NCL example: $634 fare + ~$127 estimated tax + $260 gratuities = **$1,021, or $78.54/night**, which drops it from first to second.

Same rule for any null in the stack: `nights`, `price_amount`, `obc_amount`. A null means unknown, and unknown must never be rendered as favorable.

### Zero is more dangerous than null

**Found in live testing 2026-07-25.** Five Carnival Miami → Bahamas sailings all returned `taxes_and_fees: 0`. Carnival charges roughly $100-150 in port taxes on a 5-night Bahamas itinerary. Zero is not real.

A literal `0` is worse than `null` because it passes a null check and reads as reported data.

**Rule:** on a mass-market line, treat `taxes_and_fees == 0` as **missing, not free**, unless the line genuinely bundles it (Virgin Voyages, P&O, Marella, and most luxury lines do). Estimate at 20% of fare and label it.

Applied to the Carnival example:

```
Fare                    $216
Taxes (est. 20%)         $43   ← reported as 0, not credible
Gratuities (5 × $16)     $80
                        ─────
True total              $339      $67.80/night

Reported per-night would have been $43.20. A 57% understatement.
```

**Lines that legitimately report zero taxes:** Virgin Voyages, P&O Cruises UK, Marella, and premium/luxury lines with all-inclusive fares. For everyone else, zero means the scraper didn't find the field.

### Bound the trip length

**Found in live testing.** Unbounded per-night ranking rewards absurdly long trips. A 32-night sailing at $40/night will outrank a 7-night at $55, but almost nobody has a month free.

| Situation | Band |
|---|---|
| User stated a length | ±3 nights around it |
| User stated a window but no length | **5-21 nights** default |
| User explicitly wants a long voyage | Honor it, and say the ranking is length-driven |

Any ranking on an unbounded ratio has this failure mode. Bound the denominator.

### Currency: everything is USD

Result sets mix GBP and EUR, and `price_currency` and `taxes_currency` are separate fields that can differ on the same record. **Convert every amount to USD before ranking.** See `frameworks/currency.md` for the rate source and the inversion trap.

Show the source currency alongside USD on the top pick, so a user checking the cruise line's own UK or EU site sees the number they'll recognize.

### Occupancy

Cruise fares are quoted **per person at double occupancy**. Solo travellers pay a single supplement, commonly 150-200% of the per-person rate, and it appears in no headline price.

If the user is travelling alone, say this explicitly. It roughly doubles the trip cost and is the single most common cruise-pricing surprise.

### Cabin tiers

`fares.cabin_class` exposes per-tier pricing. The interior-to-balcony delta is often small in absolute terms, and on a crossing with many sea days a balcony changes the trip materially. Flag it when the delta is under about $15/night; it's the highest-value upsell in cruising and worth naming.

---

## Examples

**Full stack with the flight home:**

```
Barcelona → Miami · MSC Seaview · 14 nights · 2 passengers

Fare (2 pax)              $1,240
Taxes & port fees           $186
Gratuities (14 × $14.50 × 2) $406
Onboard credit              -$100
Flight MIA → EWR, one-way    $97
                          ───────
TRUE TOTAL                $1,829
TRUE PER NIGHT            $65.32 pp

Advertised: "from $620pp". Real: $914pp.
```

**Length band doing its job:**

```
WITHOUT BAND          32n Pacific crossing   $41/night   ← wins, useless
WITH 5-21 BAND        14n Barcelona→Miami    $65/night   ← wins, bookable
```

---

## Anti-Patterns

| Anti-pattern | Why it breaks | Do instead |
|---|---|---|
| Using the actor's `price_per_night` | Excludes taxes, 24% understated | Recompute |
| Omitting gratuities | Largest hidden cost, up to 65% of fare | Add from the static table |
| Guessing gratuities for an unlisted line | Fabricated precision | Say unknown |
| Ranking across mixed currencies | £55 sorts below €69 but is worth more | Normalize first |
| Unbounded per-night ranking | 32-night voyages dominate | Apply the 5-21 band |
| Quoting per-person as trip cost | Fares are per person, double occupancy | State both |
| Ignoring the single supplement for solo travellers | Roughly doubles the real cost | Say it explicitly |
| Ranking one-way sailings without the flight home | The cheapest cruise is often the worst trip | Pair first |
| Hiding the math | The arithmetic is the value | Show every layer |
