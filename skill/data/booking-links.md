# Booking Links

**Every sailing presented must carry a clickable link. No exceptions.** A result the user can't act on isn't a result.

The feed's `booking_url` is unreliable in two different ways, so this file defines a fallback chain that always produces something that works.

---

## The two failure modes, both verified live 2026-07-25

**1. Null.** MSC and Margaritaville records returned `booking_url`, `detail_url`, and `image_url` all null. That scraper captures pricing for those lines but not links.

**2. Present but wrong.** Royal Caribbean records carried a `booking_url` whose `sailDate` param said `2026-10-29` while the record's `departure_date` said `2026-09-03`. Two different Harmony sailings shared an identical `groupId` and link. Clicking it prices a different cruise than the one that was ranked.

**A wrong link is worse than no link.** It quotes a price the user then blames you for.

---

## The fallback chain

```
1. booking_url present AND validates  →  use it
2. detail_url present AND validates   →  use it
3. neither, or validation fails       →  construct the line's search URL
4. line not in the table below        →  link the line's homepage,
                                          state port + date + ship + nights
```

**Never emit "no link available." Step 3 or 4 always fires.**

### Validation, step 1 and 2

Before using a link from the feed, check its query params against the record:

| Param seen | Must match |
|---|---|
| `sailDate`, `date`, `departureDate` | `departure_date` |
| `ship`, `shipCode` | `ship_code` |
| `duration`, `nights` | `nights` |
| `port`, `embarkPort` | `embark_port` |

Any mismatch → **discard the link, fall through to step 3.** Say the feed's link was discarded and why.

If the link carries no checkable params (Carnival's itinerary-level `detail_url` is like this: route in the path, no date), it's safe to use. Say it lands on the itinerary and the user picks the date.

---

## Search URLs by line

HTTP status verified 2026-07-25 with a browser user-agent.

| Line | Search URL | Status |
|---|---|---|
| Carnival | `https://www.carnival.com/cruise-search` | 200 |
| Royal Caribbean | `https://www.royalcaribbean.com/cruises` | 200 |
| Celebrity | `https://www.celebritycruises.com/cruises` | 200 |
| Norwegian (NCL) | `https://www.ncl.com/vacations` | 200 |
| Princess | `https://www.princess.com/en-us/cruise-search` | 200 |
| Holland America | `https://www.hollandamerica.com/en/us/find-a-cruise` | 200 |
| Costa | `https://www.costacruises.com/cruises.html` | 200 |
| Margaritaville at Sea | `https://www.margaritavilleatsea.com` | 200 |
| Disney | `https://disneycruise.disney.go.com/cruises-destinations` | 302 → ok |
| MSC | `https://www.msccruisesusa.com/find-a-cruise` | 403 to bots, **fine in a browser** |
| Cunard | `https://www.cunard.com/en-us/find-a-cruise` | 403 to bots, **fine in a browser** |
| Virgin Voyages | `https://www.virginvoyages.com` | homepage; `/voyages` and `/sailings` both 404 |

**403 is bot protection, not a broken URL.** Those pages load normally for a human. Do not withhold them.

### Present a constructed link with its search terms

A search URL without the terms to type is half a link:

```
→ https://www.msccruisesusa.com/find-a-cruise
  Port Miami · 2026-09-21 · 4 nights · MSC Seaside · target $124pp interior
```

---

## Deep-link patterns worth trying

Only where the feed gave nothing and the pattern is known-stable. **Confirm the page loads before presenting it**; a constructed deep link that 404s is the same failure as a wrong one.

| Line | Pattern |
|---|---|
| Carnival | `carnival.com/cruise-search?dest={code}&numadults=2&duration={n}` |
| Royal Caribbean | `royalcaribbean.com/cruises?departurePorts={PORT}&sailDate={YYYY-MM}` |
| Celebrity | `celebritycruises.com/cruises?departurePorts={PORT}` |

Never hand over a constructed deep link that hasn't been checked. Fall back to the plain search URL instead.

---

## Affiliate

There is no cruise affiliate program offering a data feed that a solo builder can join. CruiseDirect is link-only at 3% via CJ, and it ships no fare data.

If a CJ/CruiseDirect relationship is set up later, wrap the final URL in the tracking link **at presentation time only**. Never let affiliate wrapping change which sailing ranks first.

---

## Anti-Patterns

| Anti-pattern | Why it breaks | Do instead |
|---|---|---|
| "No link available" | The user can't act; the result is worthless | Fall through to the search URL |
| Using `booking_url` unvalidated | RCI's sailDate didn't match the record | Validate params first |
| Presenting a 403 URL as broken | Bot protection, loads fine for humans | Present it, note nothing |
| Constructing a deep link without checking it | A 404 is as bad as a wrong link | Verify, or use the plain search URL |
| A search URL with no search terms | User lands on a blank form | Always print port, date, ship, nights |
| Silently dropping a discarded link | The user thinks none existed | Say it was discarded and why |
