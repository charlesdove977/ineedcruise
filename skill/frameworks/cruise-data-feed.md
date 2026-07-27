# Cruise Data Feed

The primary data source. **Read before writing any cruise query.** Everything here was verified live on 2026-07-25 against real runs, not read off the docs.

Actor: `vulnv/cruise-data-feed` · $0.005 per result · 60+ cruise lines

---

## Core Concepts

### Why cruise and not flights

There is no Google Flights for cruises. Every official cruise API (Sabre, Travelport, Traveltek, Revelex, Odysseus, Widgety) is gated behind IATA/CLIA accreditation or enterprise sales, and Amadeus shut its self-service portal on 2026-07-17. That gate is why no free consumer tool does this, and why it is worth building.

### The hard cap, and why it doesn't matter

**Every run returns exactly 5 results**, regardless of `maxResults`. Verified across three queries with `maxResults` of 12, 25, and 40. This is the actor's optional `apiKey` gate.

The workaround is the architecture: **slice, don't sweep.**

Because `sort: "price"` is honored, those 5 results are the *cheapest 5 for that slice*. Issue narrow queries across the query space and each returns a genuine floor:

```
region=Caribbean        → cheapest 5 Caribbean
region=Repositioning    → cheapest 5 repositioning
region=Alaska           → cheapest 5 Alaska
embarkPort=Miami        → cheapest 5 from Miami
departureDate=2026-11   → cheapest 5 in November
```

Ten slices = 50 real candidates = $0.25. That is the sweep.

### Input schema

| Field | Notes |
|---|---|
| `cruiseLines` | Array, enum of 50 lines |
| `region` | Substring match. **`"Repositioning"` is a native value.** Also Caribbean, Alaska, Mediterranean, Northern Europe, Pacific Northwest, Transatlantic |
| `embarkPort` | Substring match: "Miami", "Barcelona", "Southampton" |
| `departureDateFrom` / `departureDateTo` | `YYYY-MM-DD` |
| `minNights` / `maxNights` | **The trip-length band, native.** Always set it. |
| `minPrice` / `maxPrice` | In the cruise's own currency, which is not always USD. See currency trap below. |
| `roundTrip` | `"any"` / `"true"` / `"false"`. **`"false"` is the repositioning filter.** |
| `sort` | `price`, `-price`, `departure_date`, `-departure_date`. **Always `price`.** |
| `maxResults` | Capped at 5 regardless. Set it to 5 and stop paying for the illusion. |

### Output fields that matter

| Field | Why |
|---|---|
| `price_amount` / `price_currency` | Base fare. **Not the total.** |
| `taxes_and_fees` / `taxes_currency` | **The differentiator.** Separate from fare. |
| `price_per_night` | ⚠️ **Excludes taxes.** Recompute it. See below. |
| `nights` | Ranking denominator |
| `embark_port` / `disembark_port` | Differing values mean one-way, which means a flight home |
| `region` | Includes `"Repositioning"` |
| `obc_amount` | Onboard credit, effectively a discount |
| `sold_out` | Filter these out |
| `fares.cabin_class` / `fares.price_amount` | Per-cabin pricing. Interior vs balcony deltas are often small. |
| `booking_url` | Link out |
| `itinerary.*` | Day-by-day ports, sea days, overnights |

---

## Verified Behavior

### `price_per_night` understates by the tax amount

Live example, Royal Caribbean *Grandeur of the Seas*, Colón → San Juan, 7 nights:

```
price_amount        £389.50
taxes_and_fees       £92.00
price_per_night      £55.64   ← 389.50 / 7, fare only
TRUE per night       £68.79   ← (389.50 + 92) / 7
```

**24% understated.** Never surface the actor's `price_per_night`. Always recompute:

```
true_per_night = (price_amount + taxes_and_fees) / nights
```

Then add gratuities from `data/gratuities-by-line.md`, which no source exposes.

### Currency is mixed and unnormalized

A single result set returned GBP and EUR side by side. `price_currency` and `taxes_currency` are separate fields and are not guaranteed to match each other.

**Convert everything to one currency before ranking.** Comparing £55.64 against €69.00 as raw numbers produces a wrong ordering.

### Coverage is thin per slice

A 10-month window with `minNights: 7` and `roundTrip: false` returned 5 results, which is the cap. Whether that is 5-of-5 or 5-of-500 is invisible from the response.

**Never claim completeness.** The honest framing is "cheapest 5 found in this slice," and the number of slices searched should be stated.

### Low adoption, real risk

6 total users, 3 monthly at time of writing. 100% success rate, but a single maintainer with almost no usage is a real bus-factor risk. If it breaks, `sercul/*` per-line scrapers are the fallback, though most lack `taxes_and_fees`.

---

## Examples

**Repositioning sweep, the signature query:**

```json
{ "roundTrip": "false", "region": "Repositioning", "sort": "price",
  "minNights": 7, "maxNights": 21, "maxResults": 5,
  "departureDateFrom": "2026-09-01", "departureDateTo": "2027-06-30" }
```

**Slice fan-out, 8 calls, ~$0.20:**

```
region=Repositioning · region=Caribbean · region=Alaska ·
region=Mediterranean · region=Transatlantic ·
embarkPort=Miami · embarkPort=Barcelona · embarkPort=Fort Lauderdale
```

Dedupe on `id`, convert currencies, recompute true per-night, rank.

**Verified live result:** Royal Caribbean, Colón → San Juan, 2027-04-24, 7 nights, £389.50 + £92 = £481.50 all-in, £68.79/night, one-way.

---

## Anti-Patterns

| Anti-pattern | Why it breaks | Do instead |
|---|---|---|
| Setting `maxResults` above 5 | You pay for a cap you cannot exceed | Set 5, slice more |
| Using `price_per_night` directly | Understates by the tax amount, ~24% in testing | Recompute with taxes |
| Ranking across mixed currencies | £55 sorts below €69 but is worth more | Normalize first |
| One broad query | 5 results for the whole world | Slice by region, port, month |
| Claiming complete coverage | The cap is invisible in the response | Say "cheapest 5 per slice, N slices searched" |
| Ignoring `sold_out` | Ranks unbookable cruises first | Filter them |
| Forgetting gratuities | $16-23/person/night, larger than taxes on short sailings | Add from the static table |
| Ignoring `disembark_port` | Misses that a one-way needs a flight home | Compare against `embark_port`; if they differ, price the flight |
