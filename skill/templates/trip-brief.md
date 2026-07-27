# Trip Brief Template

The output format. Printed conversationally, not written to a file.

Design intent: the top pick must be **screenshot-able**. The cost stack is the hook, because nobody else shows it. If someone would paste this into a group chat, it worked.

---

## Template

```template
TOP PICK — {embark_port} → {disembark_port}, {nights} nights
{cruise_line} · {ship_name} · departs {departure_date}

  Fare ({passengers} pax)          ${fare}
  Taxes & port fees                ${taxes}
  Gratuities ({n} × ${rate} × {p}) ${gratuities}
  {obc_line}
                                   ────────
  CRUISE TOTAL                     ${cruise_total}

  Flights {home} ⇄ {port} ({p} pax) ${flight_total}
    Out  {out_date} {out_time} → {out_arrive}  {out_carrier}  {buffer_note}
    Back {ret_date} {ret_time} → {ret_arrive}  {ret_carrier}
  {hotel_line}
                                   ────────
  TRIP TOTAL                       ${trip_total}
  PER NIGHT                        ${per_night} pp

Advertised as "{advertised_claim}". Real cost {real_pp} pp.
{source_currency_line}

Watch: {the_catch}
{cabin_upsell_line}

→ Book: {booking_url}
  {search_terms_line}


#2  {route_2} · {nights_2}n · ${per_night_2}/night · {one_line_why}
#3  {route_3} · {nights_3}n · ${per_night_3}/night · {one_line_why}


ALL QUALIFYING SAILINGS

{ranked_table}

{coverage_note}
Run cost: {actual_spend}
```

---

## Field Documentation

| Field | Source | Notes |
|---|---|---|
| `embark_port` / `disembark_port` | Actor | Differing values mean one-way; say so |
| `fare` | `price_amount` | For the stated passenger count |
| `taxes` | `taxes_and_fees` | Separate field, always shown |
| `gratuities` | `data/gratuities-by-line.md` | Show the arithmetic: `14 × $14.50 × 2` |
| `obc_line` | `obc_amount` | Omit if null. Render as a negative. |
| `flight_line` | `frameworks/flight-pairing.md` | One-way sailings only. Omit for round-trips. |
| `true_total` | Computed | The number that matters |
| `per_night` | `true_total / nights` | Per person, double occupancy. **Never the actor's value.** |
| `advertised_claim` | Constructed | e.g. `from $620pp` — the gap is the point |
| `the_catch` | Judgment | One-way, sea days, transfer distance, unknown gratuities |
| `cabin_upsell_line` | `fares.*` | Only when the balcony delta is under ~$15/night |
| `source_currency_line` | `price_currency` | Shown only when the source wasn't USD, e.g. `Listed at £541.50 GBP · rate 0.75053, ECB 2026-07-24` |
| `cruise_total` | Computed | Fare + taxes + gratuities − OBC |
| `flight_total` | `frameworks/flight-pairing.md` | Round-trip, all passengers, window-compliant |
| `out_time` / `out_arrive` | Flight actor | Must arrive before 12:00 on sailing day |
| `buffer_note` | Computed | e.g. `7h before sailing` or `4h after transfer` |
| `ret_time` | Flight actor | Must depart after 15:00 on disembarkation day |
| `hotel_line` | Computed | Only when arriving the day before; show the added night |
| `trip_total` | Computed | **The ranking number.** Cruise + flights (+ hotel) |
| `booking_url` | `data/booking-links.md` | Validated feed link, or the line's search URL. **Never empty.** |
| `search_terms_line` | Record | Shown under a constructed URL: `Port {x} · {date} · {n} nights · {ship} · target ${fare}pp` |
| `coverage_note` | Run metadata | Slice count, the cheapest-5-per-slice caveat, and the FX rate date |
| `actual_spend` | Run accounting | Always present |

---

## Section Specifications

### Flights are part of the price, not a footnote
The trip total is cruise **plus** flights. Measured live: a $478 round-trip against a $414 cruise, so the flight was the larger number. Ranking on the cruise alone answers a different question than the user asked.

Show the flight times, not just the price. `07:00 → 10:12, 7h before sailing` is the line that tells someone the flight is actually usable. If a cheaper fare was rejected for landing too late, name it and say why.

### The cost stack
**Never collapse it.** Showing fare → taxes → gratuities → total is the single most valuable thing in the output, because it's the one thing no booking site does. Collapsing it to a total makes this look like every other cruise site.

### Advertised vs real
One line, always. `Advertised as "from $620pp". Real cost $914pp.` This is the sentence people screenshot.

### The catch
Name the actual problem, not a generic caveat:
- `One-way. Flight home from Miami is priced in above.`
- `9 of 14 days are sea days. Great for a crossing, dull if you want ports.`
- `Gratuities unknown for this line, not included in the total.`
- `Seward is 2.5 hours from Anchorage airport.`

### Links
Per `data/booking-links.md`. **Every row in every section gets one.**
- Feed link only if its params match the record
- Otherwise the line's search URL, **always with the search terms printed beneath it**
- If a feed link was discarded for a param mismatch, say so; a silent drop reads as "no link existed"

```
→ Book: https://www.msccruisesusa.com/find-a-cruise
  Port Miami · 2026-09-21 · 4 nights · MSC Seaside · target $124pp interior
```

### Coverage note
Always present. The 5-result cap is invisible in the data and must never be passed off as completeness. The gratuity rates are a static table, not live data, so their age belongs here too:

> Searched 8 slices; each returns its cheapest 5. Not an exhaustive scan of all sailings.
> FX: GBP 0.75053, ECB 2026-07-24. Gratuity rates are estimates, last reviewed 2026-07-25; lines adjust them annually, so confirm at booking.

### Run cost
One line, always. Users of a paid-per-query tool should never wonder what a search cost.

---

## No-flight-data variant

Flights come from Apify only. Two situations leave a sailing unpriced: the departure is beyond Google Flights' ~11-month horizon, or the flight actor failed. Both use this shape:

```template
  Flight home        NOT PRICED — {reason}
                     Barcelona → Newark, mid-April 2027
                     Budget $500-800, verify around December
                                   ────────
  CRUISE TOTAL       ${true_total}   (flight not included, total provisional)
```

`{reason}` is either `beyond fare-data horizon` or `flight lookup failed`. Name which. A user who thinks the number is complete will book on it.

**Never present a one-way sailing's total as complete without the flight.** The flight is the reason the cruise is cheap.

---

## Anti-Patterns

| Anti-pattern | Fix |
|---|---|
| Leading with the ranked table | Lead with the top pick; the table is the receipt |
| Collapsing the cost stack to one number | Show all three layers, always |
| Quoting the actor's `price_per_night` | Recompute with taxes and gratuities |
| Omitting the advertised-vs-real line | It's the most shareable line in the output |
| Presenting a one-way total without the flight | Mark it incomplete |
| Generic caveats ("prices may vary") | Name the specific catch |
| Ten options | Three briefed, rest in the table |
| Omitting the coverage note | The 5-cap is invisible; disclose it |
| Omitting run cost | Always state it |
| A sailing with no link | Fall through to the line's search URL; never "no link available" |
| A search URL with no search terms | Print port, date, nights, ship, target fare beneath it |
| Using a feed link whose date doesn't match the record | Validate params first; discard and disclose on mismatch |
