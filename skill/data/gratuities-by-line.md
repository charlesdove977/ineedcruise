# Gratuities & Daily Service Charges by Cruise Line

The cost no source exposes. Used by `frameworks/true-cost-math.md`.

> ⚠️ **These are estimates, and the skill must say so in every brief it prints.** Lines adjust rates annually and they differ by cabin tier and region. Last reviewed: **2026-07-25**.
>
> **Binding rule:** any output using this table carries the line `Gratuity rates are estimates, last reviewed 2026-07-25; confirm at booking.` If a fare explicitly bundles gratuities, that wins over this table. If a line is absent from the table, say unknown rather than guessing.

---

## Why this table exists

Gratuities are the largest hidden cost on a mass-market cruise and appear in **no** data source checked during research, including `cruise-data-feed`, which does expose taxes and port fees.

On a 7-night sailing at $18/person/day, that is **$126 per person**, often 25-30% of a discounted fare. A $389 cruise is really $515 before anyone books a flight.

---

## Mass-market — billed separately, per person per day

| Line | Standard cabin | Suite | Notes |
|---|---|---|---|
| Royal Caribbean | $18.00 | $20.50 | Auto-billed daily |
| Carnival | $16.00 | $18.00 | |
| Norwegian (NCL) | $20.00 | $25.00 | Often waived under "Free at Sea" promos |
| Princess | $17.00 | $19.00 | Called "crew appreciation" |
| Celebrity | $18.00 | $23.00 | Frequently bundled in "Always Included" fares |
| MSC | $14.50 | $16.50 | Lower than US lines; varies by market |
| Holland America | $17.00 | $19.00 | |
| Disney | $16.00 | $16.00 | Suggested, not auto-billed |
| Costa | €10.00 | €12.00 | EUR |
| P&O Cruises (UK) | **£0** | £0 | Included in fare |
| Marella | **£0** | £0 | Included |
| Virgin Voyages | **$0** | $0 | Included, no auto-grat |

## Premium and luxury — usually included

| Line | Gratuities |
|---|---|
| Viking | Included |
| Regent Seven Seas | Included |
| Silversea | Included |
| Seabourn | Included |
| Explora Journeys | Included |
| Oceania | Included on most fares |
| Azamara | Included |
| Ritz-Carlton Yacht Collection | Included |

## River cruises

Usually **not** included on European river lines. Budget €12-18/person/day for AmaWaterways, Uniworld, Avalon, Emerald, CroisiEurope. Scenic and Tauck include them.

---

## How to apply

```
gratuities = daily_rate × nights × passengers

true_total     = price_amount + taxes_and_fees + gratuities
true_per_night = true_total / nights
```

Worked example, Royal Caribbean, 7 nights, 2 passengers:

```
Fare (2 pax)        £389.50
Taxes & fees         £92.00
Gratuities          £252.00   (7 × $18 × 2, converted)
                    ────────
True total          £733.50
True per night       £52.39 pp
```

The actor reported `price_per_night: 55.64`. The real figure per person is £52.39, but the **total** is £733.50 against a £389.50 headline. That gap is the entire pitch.

**Rules:**
1. Line in the "included" list → add nothing, say gratuities included. It's a selling point.
2. Line not in this table → say `gratuities not known for this line, not included` and do not guess.
3. Fare explicitly bundles them (NCL Free at Sea, Celebrity Always Included) → trust the fare, note the bundle.
4. Always show the arithmetic. The math is the value.

---

## Maintenance

Rates are published by the lines and change roughly once a year, usually with a price rise. Re-check them and bump the last-reviewed date above when any of these is true:

- [ ] It has been more than 12 months since the last review
- [ ] A user reports a real bill that differs from the table by more than $2/person/day
- [ ] A line moves between the "billed separately" and "included" tables (Virgin, P&O, Marella, and NCL's Free at Sea promo are the volatile ones)

Until then the disclaimer carries the risk: every brief states the date and tells the user to confirm at booking.
