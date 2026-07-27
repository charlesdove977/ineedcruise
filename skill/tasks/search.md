<purpose>
Turn a loose sentence about wanting to travel cheaply into a small set of real cruises ranked by honest all-in cost per night, with the flight home priced when the sailing is one-way.
</purpose>

<user-story>
As someone who wants to travel a long way for very little money, I want to describe my constraints in plain language and get back a few genuinely cheap sailings with the real total after taxes and gratuities, so that I can book knowing the number won't move at checkout.
</user-story>

<when-to-use>
- User describes a trip in any form: region, month, budget, or just "somewhere cheap"
- User wants to know whether a cruise fare they found is actually good
- Entry point routes here via /ineedcruise
</when-to-use>

<context>
@frameworks/cruise-data-feed.md
@frameworks/currency.md
@frameworks/true-cost-math.md
@frameworks/flight-pairing.md
@data/gratuities-by-line.md
@data/airport-codes.md
@templates/trip-brief.md
@checklists/output-quality.md
</context>

<steps>

<step name="check_access" priority="first">
Verify Apify access before doing anything else.

1. Apify is reachable two ways. **Check both:**
   - MCP tools (`mcp__apify__call-actor`) available in this session — this is the primary path and needs no env var
   - `APIFY_TOKEN` in the environment, for the CLI path

   ```bash
   test -n "$APIFY_TOKEN" && echo "env token: present" || echo "env token: absent"
   ```

   **An absent env var does not mean no access.** If the MCP tools are available, Full access is live. Do not force a degraded mode on the env check alone.

2. If neither path works, route to `tasks/setup.md`.

3. Apify is the **only** external dependency. It drives both legs: `vulnv/cruise-data-feed` for sailings and `memo23/google-flights-scraper` for flights. There is no second token to configure and no free fallback tier.
</step>

<step name="parse_intent">
Convert the user's sentence into a query object. Do not present a menu.

```
{ region|ANY, embarkPort|ANY, windowStart, windowEnd,
  minNights, maxNights, maxPrice|null, oneWayOnly|any, passengers }
```

| User says | Parse to |
|---|---|
| "cheap cruise out of Florida this winter" | `embarkPort=Miami,Fort Lauderdale, window=Dec-Feb, nights=5-14` |
| "repositioning crossing to Europe, two weeks, under $1500" | `region=Repositioning, roundTrip=false, nights=12-16, maxPrice=1500` |
| "somewhere warm, cheap as possible" | `region=Caribbean, nights=5-14, sort=price` |
| "I want to get to Europe cheap" | `region=Transatlantic + Repositioning, roundTrip=false` |

Rules:
- **Infer aggressively, then confirm once in the next step.** Not naming a destination is the point of this skill, so never interrogate the user for a destination. But do fill gaps that change what gets searched: embark port or region, rough window, night range, budget, passenger count, and **the home airport they'd fly from** (flights are part of the ranking, so a missing origin makes the result wrong, not just thin).
- Ask follow-ups **here**, before any paid call. Questions are free; slices are not.
- **Always set `minNights`/`maxNights`.** Default 5-21. Unbounded per-night ranking rewards absurdly long sailings.
- Default window if unstated: next 3-12 months. Cruise inventory is published much further out than flight cache data, so a longer window is fine here.
- Default `passengers` to 2. Cruise fares are quoted per person at double occupancy, and solo travellers pay a 150-200% single supplement that headline pricing never shows. If the user is solo, say this.

**Wait for the user's answer if you asked a clarifying question.**
</step>

<step name="confirm_plan_and_cost">
**Every slice costs $0.10 of the user's money. Confirm before spending it.**

Show the parsed query, the slices you intend to run, and the total cost. Then stop.

```
Here's what I'll search:

  Departing     Miami, Fort Lauderdale
  Window        Dec 2026 – Feb 2027
  Length        5–14 nights
  Budget        under $800 all-in pp
  Travelers     2

  Flying from   EWR/JFK/LGA
  Slices (5):   port:Miami · port:Fort Lauderdale · region:Caribbean
                region:Repositioning · region:Bahamas

  Cost          ~$0.50    (5 × $0.10 Apify minimum)
              + ~$0.04    (round-trip flights for the top 4)
                ─────
                ~$0.54    ·  about 60 seconds

  Returns       cheapest 5 per slice, ranked in USD on
                cruise + flight combined, all-in

Run it? Or adjust the ports, window, or budget first.
```

**Ask for the home airport here if you don't have it.** Flights are part of the ranking, so a missing origin makes the whole result wrong rather than merely incomplete. If the user is driving to the port, say so and skip the flight leg.

Rules:
- **Always show the cost before running.** A user on Apify's free $5/month tier gets 6-10 searches. They should know what each one costs.
- **Always include a `region: "Repositioning"` slice when the user is price-led.** That is where the outliers live, and it is the slice most likely to surprise them.
- If the user's ask implies more than 8 slices, say so and propose a narrower set rather than silently running a $1+ search.
- If they adjust, re-quote and confirm again.

**Wait for explicit approval before proceeding. Do not spend on an assumption.**
</step>

<step name="slice_and_sweep">
Query the cruise feed. This is the paid step, so shape it deliberately.

**The actor returns exactly 5 results per run regardless of `maxResults`** (see `frameworks/cruise-data-feed.md`). Because `sort: "price"` is honored, those 5 are the cheapest 5 *for that slice*. So slice the query space instead of issuing one broad call.

1. Build 5-10 slices from the parsed intent. Vary one axis at a time:

   | Axis | Example slices |
   |---|---|
   | Region | Repositioning, Caribbean, Transatlantic, Mediterranean, Alaska |
   | Embark port | Miami, Fort Lauderdale, Barcelona, Southampton |
   | Month | one slice per month in the window |

   Always include a `region: "Repositioning"` slice when the user is price-led. That is where the outliers live.

2. Run each slice with `sort: "price"`, `maxResults: 5`, and the night band applied.

3. Dedupe on `id`.

4. Drop anything with `sold_out: true`.

5. **Convert everything to USD.** Follow `frameworks/currency.md`. Result sets mix GBP and EUR, and `price_currency` and `taxes_currency` can differ on the same record. Fetch rates once per run from `api.frankfurter.dev`, convert every monetary field, and convert **before** ranking. All output is USD.

**Cost:** **$0.10 per slice minimum.** Apify enforces a $0.10 floor on `maxTotalChargeUsd` per run, so a 5-slice search is ~$0.50 and an 8-slice search is ~$0.80, regardless of how few results come back. Choose slices deliberately, and state the slice count and the actual spend in the output.

If nothing clears the user's budget: say so and report the actual floor. "Nothing under $800 all-in; cheapest found is $940 for 11 nights, Barcelona to Miami." Never pad.
</step>

<step name="compute_true_cost">
Apply `frameworks/true-cost-math.md` to every candidate. This is the product.

```
true_total     = price_amount + taxes_and_fees + gratuities
true_per_night = true_total / nights
```

- **Never use the actor's `price_per_night`.** It excludes taxes and understated by 24% in live testing.
- Gratuities come from `data/gratuities-by-line.md`. If the line isn't listed, say gratuities are unknown for that line rather than guessing.
- If the line bundles gratuities, say so. It's a selling point, not a blank.
- Subtract `obc_amount` where present; onboard credit is a real discount.

Rank on `true_per_night`.
</step>

<step name="pair_flights">
**Every sailing needs a round-trip flight priced, not just one-way sailings.** Follow `frameworks/flight-pairing.md`. Skip only if the user says they're driving to the port.

Measured live: `EWR → MIA` round-trip for 2 was **$478**, larger than the $414 cruise it paired with. This step routinely changes which sailing wins.

1. **Take the top 3-5** from the cruise-only ranking. Do not price flights for all candidates; that's $0.18 and 90 seconds for data you discard.

2. **Resolve airports** via `data/airport-codes.md`:
   - Outbound destination = nearest airport to `embark_port`
   - Return origin = nearest airport to `disembark_port` (same for round-trip sailings, **different for one-way**)
   - Home airport = the user's, as a metro list where sensible (`"EWR,JFK,LGA"`)

3. **Run the 3-5 queries in parallel** with `memo23/google-flights-scraper`. This is the only flight source; there is no free sweep layer.
   ```json
   { "origin": "EWR,JFK,LGA", "destination": "MIA",
     "departDate": "{sail_date}", "returnDate": "{disembark_date}",
     "adults": {passengers}, "currency": "USD", "maxItems": 5 }
   ```
   - `departDate`, not `departureDate`. The param name is actor-specific; see the fallback table in `frameworks/flight-pairing.md` before switching actors.
   - `maxItems: 5` — the default of 100 costs 10x
   - Leave the residential proxy default alone, or Google returns 0 items silently
   - One-way sailings use `multiCityLegs`, not `returnDate`

   If `memo23` errors or returns 0 items on a live proxy, escalate through the fallback chain in `frameworks/flight-pairing.md` and **say which actor produced the numbers**. Never silently retry.

4. **Apply the safety windows before picking a fare.** Filter first, then take the cheapest survivor:

   | Leg | Rule |
   |---|---|
   | Outbound | Early AM, **arriving before 12:00** on sailing day. Or any flight the day before, plus a hotel night in the total. **Never** arriving after 14:00. |
   | Return | **After 15:00** on disembarkation day. Never before 11:00. |

   Subtract port transfer time from the buffer: a 10:12 arrival at MCO is an 11:02 arrival at Port Canaveral.

   If the cheapest fare violates a window, **report the compliant one instead** and say why the cheaper was rejected: `$60 cheaper but lands 15:40, ship sails 17:00 — too tight.`

5. **Re-rank on the combined total.** The cruise-only ordering was scratch work; cruise + flight is the answer.

   ```
   combined_total     = cruise_true_total + flight_total (+ hotel if arriving day before)
   combined_per_night = combined_total / nights
   ```

6. **Beyond ~11 months out**, Google Flights has no data. Give a labelled range, keep cruise and flight as separate numbers, and mark the combined figure provisional. Never invent a fare.

**Added cost:** ~$0.03-0.05. **Added time:** ~10 seconds in parallel.
</step>

<step name="resolve_booking_links">
**Every sailing presented must carry a clickable link.** Follow `data/booking-links.md`.

For each candidate that will appear in the output:

1. If `booking_url` exists, **validate its query params against the record** (`sailDate` vs `departure_date`, `ship`, `nights`, `port`). A mismatch means the link prices a different cruise. Discard it and say why.
2. If `detail_url` exists and validates, use it. Itinerary-level links with no date param are safe; say the user picks their date.
3. Otherwise construct the line's search URL from the table, and **print the search terms alongside it**: port, date, nights, ship, target fare.
4. If the line isn't in the table, link its homepage with the same search terms.

**Never output a sailing without a link.** Step 3 or 4 always fires. "No link available" is not an acceptable result.

Note that MSC and Cunard return 403 to automated checks. That is bot protection, not a broken URL. Present them normally.
</step>

<step name="write_brief">
Produce output via `templates/trip-brief.md`.

1. **Top pick** — full brief: ship, line, route, dates, nights, the cost stack broken out, true per-night, the catch, booking link
2. **#2 and #3** — one line each
3. **Ranked table** — all candidates that cleared the filters, **each with a link**
4. **Coverage note** — how many slices were searched, and that each returns its cheapest 5
5. **Run cost**

Always show the arithmetic from headline to true cost. The math is the value; hiding it makes the skill look like every other booking site.

Run `checklists/output-quality.md` before presenting.

**Present the brief and wait for the user's reaction.**
</step>

<step name="offer_next">
Offer one concrete next action, not a menu:

- Top pick is one-way and the flight is expensive → offer to check the reverse crossing
- Everything clustered near the ceiling → offer to widen the window or the night band
- A repositioning slice returned the winner → offer the same route in the opposite season
- Balcony was only a few dollars a night more → say so; it's the highest-value upsell in cruising

</step>

</steps>

<output>
A trip brief printed conversationally:
- Top 3 sailings with the full cost stack broken out
- True all-in cost per night, gratuities included
- Flight home priced for one-way sailings
- Ranked table of all candidates
- Slice count, coverage caveat, and run cost

No files written. Stateless by design.
</output>

<acceptance-criteria>
- [ ] Apify access checked via MCP, not the env var alone
- [ ] Gaps resolved by asking **before** any paid call, never after
- [ ] Search plan and total cost shown, and explicit approval received, before the first Apify run
- [ ] A `region: "Repositioning"` slice was included whenever the user was price-led
- [ ] Night band always set (default 5-21)
- [ ] `maxResults` set to 5; slicing used instead of one broad query
- [ ] All amounts converted to USD before ranking, with the rate and its date stated
- [ ] `sold_out` results filtered out
- [ ] Actor's `price_per_night` never surfaced; true per-night recomputed with taxes
- [ ] Gratuities added from the table, or explicitly marked unknown
- [ ] **Every presented sailing carries a link** — feed link, validated; or a constructed search URL with its search terms
- [ ] Feed links were validated against the record before use, and mismatches disclosed
- [ ] **Round-trip flights priced for the top 3-5**, not just one-way sailings
- [ ] Final ranking is on cruise + flight combined, not cruise alone
- [ ] Outbound arrives before 12:00 on sailing day, or is a day-before flight with a hotel night costed in
- [ ] Return departs after 15:00 on disembarkation day
- [ ] Port transfer time subtracted from the embarkation buffer
- [ ] Any window-violating cheaper fare was named and explicitly rejected, not silently used
- [ ] `maxItems: 5` set and the residential proxy default left alone
- [ ] Zero-result case reports the actual floor instead of padding
- [ ] Coverage caveat states slice count and the cheapest-5-per-slice limit
- [ ] Run cost reported
- [ ] `checklists/output-quality.md` passed before presenting
</acceptance-criteria>
