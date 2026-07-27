# Currency Normalization

**All output is USD.** The cruise feed returns source currency with no conversion option, and a single result set mixes GBP and EUR. Ranking raw numbers across currencies produces a wrong order.

---

## Core Concepts

### The problem, verified live

One `cruise-data-feed` run returned:

```
Royal Caribbean   £541.50 GBP
MSC               €483.00 EUR
Norwegian         €557.00 EUR
```

`£541.50` sorts *below* `€557.00` as a raw number, but is worth **more** ($721 vs $634). Sorting without converting inverts the ranking.

Worse: `price_currency` and `taxes_currency` are **separate fields** and are not guaranteed to match on the same record. Convert each independently.

### Rate source

**Primary — frankfurter.dev.** Free, no key, no rate limit, ECB reference rates.

⚠️ **Fetch with `curl`, not Python `urllib`.** Verified live: `urllib.request.urlopen` gets **HTTP 403** because it sends no user-agent. `curl` works. Fetch to a file with curl, then parse.

```bash
curl -s "https://api.frankfurter.dev/v1/latest?base=USD&symbols=GBP,EUR,CAD,AUD"
```

```json
{"amount":1.0,"base":"USD","date":"2026-07-24",
 "rates":{"AUD":1.431,"CAD":1.4086,"EUR":0.87897,"GBP":0.75053}}
```

**Fallback — open.er-api.com.** Also free, no key, and reports its own freshness.

```bash
curl -s "https://open.er-api.com/v6/latest/USD"
```

Both agreed to four decimal places when checked on 2026-07-25, which is a good sign for either.

### Direction of the math

The APIs return **USD → foreign**. You need the inverse.

```
usd_amount = foreign_amount / rate

£541.50 / 0.75053 = $721.49
€557.00 / 0.87897 = $633.70
```

Getting this backwards produces numbers that look plausible and are wrong by the square of the rate. Sanity-check against a known anchor: **£1 is worth more than $1; €1 is worth more than $1.** If a GBP figure converts to a smaller USD number, the division is inverted.

### Rules

1. **Fetch rates once per run.** Cache for the session. ECB publishes daily; per-record calls are wasted.
2. **Convert every monetary field**: `price_amount`, `taxes_and_fees`, `obc_amount`, and per-cabin `fares.price_amount`.
3. **Convert using each field's own currency tag.** Do not assume `taxes_currency` matches `price_currency`.
4. **Convert before ranking, never after.**
5. **State the rate and its date** in the coverage note. Cruise fares are booked weeks out and rates move.
6. **Show the original alongside USD** on the top pick. A user comparing against the cruise line's own UK site needs to see the number they'll recognize.
7. **If both rate APIs fail**, present in source currency, say conversion was unavailable, and **do not rank across currencies**. Group by currency instead.

### Rounding

Round USD to whole dollars for display. Keep full precision for ranking. Rounding before comparison can flip adjacent candidates.

---

## Examples

**Fetch and invert:**

```bash
RATES=$(curl -s "https://api.frankfurter.dev/v1/latest?base=USD&symbols=GBP,EUR,CAD,AUD")
# GBP 0.75053 → multiply GBP amounts by 1/0.75053 = 1.3324
```

**Applied to a real record:**

```
Celebrity Constellation, San Juan → Barcelona, 13 nights

  Source              USD
  £541.50   fare      $721
  £90.00    taxes     $120
            ─────────────
            total     $841
            per night  $65 pp

  (rate: GBP 0.75053, ECB 2026-07-24)
```

**Cross-currency ranking, before and after:**

```
RAW (wrong)              CONVERTED (correct)
€557.00  ← ranks first   $634  ← ranks first
£541.50                  $721
```

The raw order is inverted; the two records swap once converted.

---

## Anti-Patterns

| Anti-pattern | Why it breaks | Do instead |
|---|---|---|
| Ranking before converting | £541 sorts below €557 but is worth more | Convert first |
| Multiplying instead of dividing | Wrong by the square of the rate | `usd = foreign / rate` |
| Assuming `taxes_currency` matches `price_currency` | Separate fields, can differ | Convert each independently |
| Hardcoding rates | Goes stale silently, and cruise fares book weeks out | Fetch per run |
| Fetching rates per record | Wasted calls, ECB updates daily | Fetch once, cache |
| Omitting the rate and date | User cannot reconcile against the cruise line's site | State both |
| Dropping the source currency entirely | User sees $721 where the site says £541.50 | Show both on the top pick |
| Ranking across currencies when the FX call failed | Silent wrong ordering | Group by currency, disclose |
