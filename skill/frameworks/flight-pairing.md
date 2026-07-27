# Flight Pairing

The step no cruise site performs. Cruise sites don't sell flights; flight sites don't know about ships. Assembling the two is the only genuinely novel output this skill produces.

**Always round-trip. Always priced. Always folded into the final ranking.**

---

## Core Concepts

### Why this dominates the ranking

**Verified live 2026-07-25:** `EWR → MIA`, Sep 20-25, 2 adults, nonstop United: **$478**.

That is *more* than the $414 all-in MSC Seaside sailing it pairs with. **The flight was 54% of the trip.**

A ranking that ignores the flight isn't slightly off, it answers a different question. A $394 cruise from Miami and a $733 cruise from Port Canaveral look 2x apart until flights land, then it's $872 vs $1,211.

**The final ranking is cruise + flight, never cruise alone.** The only exception is a user who says they're driving to the port.

### Why one-way sailings are cheap

Cruise lines move ships between seasons: Caribbean to Mediterranean in spring, Alaska to Mexico in autumn. Those crossings are one-way, unpopular, and dumped at $40-70 a night.

The reason nobody books them is the reason they're cheap: **you land in the wrong country.** For these, the return leg flies from a *different* airport, so it's a multi-city search, not a round trip.

### Data source: Apify, and only Apify

Flights come from one place: a Google Flights scraper actor on Apify. There is no free sweep layer, no cached-fare API, and no second token. The same Apify credit that pays for the cruise feed pays for the flights, which is why the cost quote in `tasks/search.md` covers both.

**Primary actor: `memo23/google-flights-scraper`.** Measured: 4.6s runtime, $0.01 per query with `maxItems: 5`.

**Fallback chain.** Only if the primary errors or returns 0 items on a live residential proxy. The date param name differs per actor, and a wrong name does not error — it silently searches a default date about 30 days out. Check this table before switching:

| Actor | Date param | Cost | Notes |
|---|---|---|---|
| `memo23/google-flights-scraper` | `departDate` | $1/1k + $0.005 start | **Primary.** 99.3% success. |
| `automation-lab/google-flights-scraper` | `departDate` | $0.005 + $0.00023/flight | First fallback. Simple schema. |
| `kaix/google-flights-scraper` | **`departureDate`** | $0.0001/flight | Cheapest, but the param name changes. Read it twice. |

**Always name the actor that produced the fare** in the brief when it wasn't the primary. A user comparing against Google Flights directly needs to know which scraper they're reconciling against.

```json
{ "origin": "EWR", "destination": "MIA",
  "departDate": "2026-09-20", "returnDate": "2026-09-25",
  "adults": 2, "cabinClass": "economy", "currency": "USD", "maxItems": 5 }
```

- **Residential proxy is ON by default.** Do not override it. Google returns empty results to datacenter IPs, and the failure is silent: `SUCCEEDED` with 0 items in ~5s. If you see that, the proxy is off.
- **Param is `departDate`, not `departureDate`.** `kaix` uses `departureDate`. Wrong name silently searches a default date ~30 days out.
- **Always cap `maxItems: 5`.** Default 100 costs $0.105 instead of $0.01.
- `origin` accepts comma-separated metro lists: `"EWR,JFK,LGA"`. One query, three airports.

### Useful output fields

| Field | Why |
|---|---|
| `price-usd` | Already USD |
| `bucket` | Google's own tag, `best` or `other`. Free quality signal. |
| `depart-time`, `arrive-time` | **Critical** — drives the safety windows below |
| `carrier-name`, `stops`, `duration-minutes` | Brief content |
| `booking-token` | Deep-link construction |

---

## The safety windows — these are not optional

Missing embarkation means losing the entire trip. Missing a return flight means an unplanned hotel night. Both are avoidable with time-of-day selection, and neither costs extra.

### Outbound: early morning, arrive with daylight to spare

Ships board mid-afternoon and sail early evening (typically 16:00-18:00).

| Preference | Rule |
|---|---|
| **Best** | Early-AM flight on sailing day, **arriving before 12:00** |
| **Safest** | Any flight the **day before**, plus a hotel night in the total |
| **Never** | Arrival after 14:00 on sailing day |

An early-AM arrival gives 4-6 hours of buffer against delays without paying for a hotel. That is usually both the cheaper and the safer choice, which is why it's the default.

If the cheapest fare arrives after 14:00 on sailing day, **do not silently recommend it.** Show the early-AM option even if it costs more, name the price difference, and let the user choose.

### Return: late in the day, well after disembarkation

Disembarkation runs 07:00-10:00, and customs plus baggage adds time.

| Preference | Rule |
|---|---|
| **Best** | Departure **after 15:00** on disembarkation day |
| **Acceptable** | After 12:00, if the airport is under 30 minutes from the pier |
| **Never** | Before 11:00 |

A late departure also tends to be cheaper than the mid-morning business peak, so the safe choice is again usually the cheap one.

### Applying the windows

1. Pull 5 itineraries per leg.
2. **Filter by the windows first**, then take the cheapest survivor.
3. If nothing survives, report the cheapest compliant option even if it's pricier, and say why the cheaper one was rejected: `$60 cheaper but lands 15:40, ship sails 17:00 — too tight.`
4. Never present a window-violating flight as the recommendation.

---

## Which airports

- **Outbound destination:** nearest airport to `embark_port` — see `data/airport-codes.md`
- **Return origin:** nearest airport to `disembark_port`. Same as outbound for a round-trip sailing; **different for a one-way**
- **Home airport:** ask once during the interview. Use the metro list when the user names a city (`"EWR,JFK,LGA"`)

For one-way sailings, use `multiCityLegs`:
```
["EWR-MIA-2026-09-20", "BCN-EWR-2026-10-05"]
```

---

## When to run it

**Only on the top 3-5 cruise candidates, after cruise-only ranking.** Pricing all 18 costs $0.18 and 90 seconds for data that gets discarded.

Run the 3-5 queries **in parallel**. Added cost: ~$0.03-0.05. Added time: ~10 seconds.

Then **re-rank on the combined total.** The cruise-only order is scratch work; the flight-inclusive order is the answer.

### Beyond ~11 months out

Google Flights publishes about 11 months ahead. Cruise inventory is published much further out, so a sailing can be real while its flight is not yet priceable. Beyond that horizon, **do not invent a number**:

> Flight EWR → BCN, April 2027. Too far out for fare data. Transatlantic round-trips typically run $500-800. Budget $650 and re-check around December.

State cruise and flight as separate numbers, and mark the combined figure provisional.

---

## Examples

**Reordering in action:**

```
MSC Seaside · Miami · 4n
  Cruise all-in        $414
  EWR→MIA rt (2 pax)   $478   ← larger than the cruise
  TOTAL                $892   = $111.50/night

Carnival Glory · Port Canaveral · 4n
  Cruise all-in        $733
  EWR→MCO rt (2 pax)   $310
  TOTAL              $1,043   = $130.38/night
```

The cheap cruise still wins, but the gap collapses from 77% to 17%.

**Safety windows applied:**

```
EWR → MIA, Sep 20 (ship sails 17:00)
  07:00 → 10:12  $478  ✅ recommended, 7h buffer
  11:25 → 14:41  $485  ⚠️  2h buffer, tight with bags
  15:38 → 18:52  $478  ❌ ship has sailed
  20:37 → 23:52  $478  ❌ ship has sailed

Three of five itineraries are unusable at any price.
```

**Port-to-airport traps** (full table in `data/airport-codes.md`):

```
Civitavecchia → FCO, 90 min      Seward         → ANC, 2.5 hrs
Southampton   → LHR, 90 min      Colón          → PTY, 90 min
Cape Liberty  → EWR, 20 min      Port Canaveral → MCO, 50 min
```

Flag transfers over an hour, and subtract them from the buffer. A 10:12 arrival at MCO is a 11:02 arrival at the pier.

---

## Anti-Patterns

| Anti-pattern | Why it breaks | Do instead |
|---|---|---|
| Ranking cruises without flights | The flight can exceed the cruise | Price the top 3-5, then re-rank on the combined total |
| Recommending the cheapest flight regardless of time | Three of five itineraries can't make the ship | Filter by the safety windows first |
| Arrival after 14:00 on sailing day | One delay and the ship leaves | Early AM, or the day before plus a hotel |
| Return before 11:00 | Disembarkation isn't finished | After 15:00 preferred |
| Ignoring transfer time in the buffer | MCO is 50 minutes from Port Canaveral | Subtract it before judging the buffer |
| Overriding the residential proxy | Silent `SUCCEEDED` with 0 items | Leave the default alone |
| `departureDate` on `memo23` | Silently searches a default date | It's `departDate` here. Check the fallback table per actor. |
| Silently retrying or swapping actors | User cannot reconcile the fare against anything | Escalate the chain openly, name the actor used |
| Leaving `maxItems` at 100 | $0.105 instead of $0.01 | Cap at 5 |
| Round-trip search for a one-way sailing | You'd fly home from the wrong city | Use `multiCityLegs` |
| Pricing flights for all candidates | 18 × $0.01 and 90s for discarded data | Top 3-5 only, in parallel |
| Inventing a fare beyond data range | Fabricated precision | Labelled range, marked provisional |
