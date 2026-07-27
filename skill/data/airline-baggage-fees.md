# Airline Baggage Fees

Used by `frameworks/true-cost-math.md` to convert a headline fare into a true all-in cost.

> ⚠️ **These figures are estimates and must be verified before public release.**
> Baggage pricing changes several times a year, varies by route and market, and is often cheaper prepaid online than at the airport. Last reviewed: **2026-07-25**.
> When the skill uses these numbers, it must label them as estimates. When a verified fee is available from the Apify response, that value wins.

---

## US carriers — one-way, economy, prepaid online

| Airline | Code | Carry-on (basic) | 1st checked | 2nd checked | Notes |
|---|---|---|---|---|---|
| Frontier | F9 | **$45-75** | $45-75 | $60-85 | Nearly everything is an add-on. Headline fare is close to a deposit. |
| Spirit | NK | **$40-70** | $40-70 | $55-80 | Same model. Cheapest at booking, most expensive at the gate. |
| Basic economy, legacy | AA/DL/UA | **Often excluded** | $35-40 | $45-50 | Personal item only on many basic fares |
| American | AA | Included (non-basic) | $40 | $45 | |
| Delta | DL | Included (non-basic) | $35 | $45 | |
| United | UA | Included (non-basic) | $40 | $50 | |
| JetBlue | B6 | Included | $45 | $60 | Blue Basic restricts carry-on |
| Southwest | WN | Included | **Free** | **Free** | Two free checked bags, still unusual |
| Alaska | AS | Included | $35 | $45 | |
| Hawaiian | HA | Included | $30 | $40 | |
| Allegiant | G4 | $25-60 | $25-50 | $35-65 | Add-on model |
| Sun Country | SY | $30-50 | $30-50 | $40-60 | Add-on model |

## European low-cost

| Airline | Code | Cabin bag | Checked | Notes |
|---|---|---|---|---|
| Ryanair | FR | Small bag free, larger **€20-40** | €25-60 | Strict sizers, gate fees are punitive |
| Wizz Air | W6 | Small free, larger €25-45 | €30-60 | Same model |
| easyJet | U2 | Small free, larger £25-45 | £25-55 | |
| Vueling | VY | Small free, larger €20-40 | €25-50 | |
| Norwegian | DY | Varies by fare | €25-55 | |
| TAP Air Portugal | TP | Usually included | Varies | Discount fares may exclude |
| Level | — | Varies | Varies | Long-haul low-cost, verify per fare |

## Transatlantic, standard economy

Most legacy carriers include a carry-on and one checked bag on transatlantic economy. Basic-economy transatlantic fares often do **not**. Always confirm from the fare class rather than the route.

---

## How to apply

1. Read the fare class from the Apify response.
2. If the class includes baggage → no addition. Note it as included.
3. If the class excludes it → add the round-trip cost, `fee × 2`.
4. If the class is unknown → **do not guess.** Say `fare class unknown, bag cost not included` and let the user decide.
5. Always show the arithmetic:

```
$318 headline + $120 carry-on (2 × $60) = $438 true
```

---

## Verification before release

- [ ] Spot-check every US carrier row against its current published fee page
- [ ] Confirm Southwest's two-free-bags policy still stands
- [ ] Confirm whether legacy basic economy still excludes carry-on
- [ ] Re-check European low-cost cabin-bag rules, which changed under EU pressure
- [ ] Add a `last-verified` date to this file and surface it in the skill's output
