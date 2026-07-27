# Output Quality Checklist

**Purpose:** Verify a trip brief is honest, complete, and safe to present. Run at the end of `tasks/search.md`, before printing anything.

Every item is independently pass/fail. Any failure blocks the output until fixed.

---

## Cost honesty

- [ ] The full three-layer stack is shown: fare, taxes and port fees, gratuities
- [ ] The actor's `price_per_night` was **not** surfaced; true per-night was recomputed
- [ ] Gratuities were added from `data/gratuities-by-line.md`, or explicitly marked unknown for that line
- [ ] Gratuities were **not** guessed for a line absent from the table
- [ ] Gratuity and tax figures are labelled as estimates where they came from the static table, **with the table's last-reviewed date shown**
- [ ] `obc_amount` was subtracted where present
- [ ] **No null treated as zero.** A record with `taxes_and_fees: null` was not allowed to outrank records with reported taxes
- [ ] **No zero taken at face value on a mass-market line.** `taxes_and_fees: 0` was treated as missing and estimated, unless the line genuinely bundles taxes
- [ ] Any estimated tax figure is labelled as estimated, not reported
- [ ] The advertised-vs-real line is present
- [ ] Per-person and total costs are distinguished, not conflated
- [ ] If the user is solo, the single supplement was stated

## Flights

- [ ] Round-trip flights priced for every briefed sailing (unless the user is driving)
- [ ] **Final ranking is on cruise + flight combined**, not cruise alone
- [ ] Outbound arrives **before 12:00** on sailing day, or is a day-before flight with a hotel night costed in
- [ ] Outbound **never** arrives after 14:00 on sailing day
- [ ] Return departs **after 15:00** on disembarkation day, never before 11:00
- [ ] Port transfer time subtracted before judging the embarkation buffer
- [ ] Flight times shown in the brief, not just the price
- [ ] Any cheaper window-violating fare was named and explicitly rejected
- [ ] One-way sailings used a multi-city search, not a round trip from the wrong airport
- [ ] Sailings beyond ~11 months got a labelled estimate range, not an invented fare

## One-way handling

- [ ] Every sailing where `embark_port !== disembark_port` is identified as one-way
- [ ] A flight home is priced for each, or the total is explicitly marked incomplete
- [ ] Flight prices are labelled as live-scraped at run time, and as subject to change before booking
- [ ] Flight departure is disembark date + 1 or later, never same-day
- [ ] Port-to-airport transfers over an hour are flagged
- [ ] Where no cached fare exists, a labelled range was given rather than an invented number

## Correctness

- [ ] Every amount converted to USD before ranking (fare, taxes, OBC, cabin fares)
- [ ] Conversion divided by the rate, not multiplied (GBP and EUR amounts must grow in USD)
- [ ] Each field converted using its own currency tag, not the record's primary currency
- [ ] FX rate and its date stated in the output
- [ ] Source currency shown alongside USD on the top pick
- [ ] If the FX call failed, results grouped by currency rather than ranked across them
- [ ] `sold_out` results filtered out
- [ ] Night band applied (default 5-21), so a 32-night voyage cannot take the top slot
- [ ] Ranking used true per-night, not fare and not the actor's value
- [ ] `maxResults` set to 5; slicing used rather than one broad query
- [ ] Results deduped on `id` across slices

## Coverage disclosure

- [ ] Slice count is stated
- [ ] The cheapest-5-per-slice limit is disclosed
- [ ] No claim of exhaustive coverage anywhere in the output
- [ ] If zero results cleared the budget, the actual floor is reported instead of padded results

## Cost control

- [ ] Actual run spend is reported
- [ ] Slice count is proportionate to the query; no runaway fan-out
- [ ] If a run failed, the failure was disclosed rather than hidden

## Compliance

- [ ] No data pulled from Cruise Critic (blocks ClaudeBot by name, explicit ToS scraping ban)
- [ ] No data pulled from Vacations To Go (robots.txt Disallow, login wall)
- [ ] No hidden-city, Skiplagged, or fuel-dump itinerary auto-generated or recommended
- [ ] If asked about hidden-city, the risk disclosure was given: airlines detect it, ban accounts, void miles

## Booking links

- [ ] **Every presented sailing has a clickable link.** No row, briefed or tabled, says "no link available"
- [ ] Feed `booking_url` / `detail_url` validated against the record (date, ship, nights, port) before use
- [ ] Any link discarded for a param mismatch was disclosed, not silently dropped
- [ ] Constructed search URLs are accompanied by the search terms: port, date, nights, ship, target fare
- [ ] No constructed deep link was presented without confirming it loads
- [ ] MSC and Cunard links were presented normally, not withheld for returning 403 to bots

## Presentation

- [ ] Top pick leads with the cost stack, not a bare total
- [ ] The "Watch:" line names a specific catch, not a generic caveat
- [ ] Exactly three sailings briefed; the rest sit in the ranked table
- [ ] Balcony upsell flagged only when the delta is genuinely small
- [ ] Exactly one next action offered, not a menu
