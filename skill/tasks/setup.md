<purpose>
Get the user from zero to a working cruise search, with the required access explained honestly, including what it actually costs per run.
</purpose>

<user-story>
As someone who just installed this skill, I want to know which access I need, which is optional, and what a search costs, so that I can run one immediately without an unexplained error or a surprise bill.
</user-story>

<when-to-use>
- First run, when Apify is unreachable
- User runs `/ineedcruise setup`
- User wants to add flight pairing after running cruise-only
- Access stopped working
</when-to-use>

<steps>

<step name="explain_what_is_needed" priority="first">
Set expectations before asking for anything.

> This needs exactly one thing: **Apify**.
>
> Apify drives both halves of the search, the cruise feed and the flight pricing. Free tier gives $5/month in credits, no card required. Apify enforces a **$0.10 minimum per run**, and a search runs 5-8 cruise slices plus 3-5 flight lookups, so budget **$0.55-0.85 per search**. That's roughly **6-9 searches a month** on the free tier.
>
> There is no second token, no free tier fallback, and no way to run this for nothing. Every official cruise API is gated behind travel-agency accreditation, which is exactly why no free consumer tool does this.

Confirm they want to proceed.

**Wait for response.**
</step>

<step name="apify_access">
Apify is reachable two ways. **Check the MCP path first**, since many users already have it.

1. If `mcp__apify__call-actor` is available in this session, access is already live. Say so and skip to verification. **Do not send someone to sign up for something they already have.**

2. Otherwise:
   - Sign up at **apify.com**. Free tier, $5/month credits, no card required.
   - **Settings → Integrations → API tokens**, copy it.
   - Export:
     ```bash
     echo 'export APIFY_TOKEN="your_token_here"' >> ~/.zshrc
     source ~/.zshrc
     ```
   - Restart Claude Code so the variable reaches the process.

**Never paste a token into chat.** It persists in the transcript and in shell history. If one has already been pasted, rotate it before using it for anything real.

**Wait for confirmation.**
</step>

<step name="verify_apify">
Prove it works. Do not accept "I set it up" as verification.

Run `vulnv/cruise-data-feed` with a small query:

```json
{ "region": "Repositioning", "sort": "price", "maxResults": 5,
  "minNights": 7, "maxNights": 21 }
```

| Result | Meaning | Fix |
|---|---|---|
| 5 items with `taxes_and_fees` populated | Working | Continue |
| Auth error | Token wrong or not exported | Re-check export, restart session |
| 0 items | Filters too narrow | Widen the window |
| Actor not found | Actor was removed | Fall back to `sercul/*` per-line scrapers, and warn that most lack `taxes_and_fees` |

Cost of this check: **$0.10** (Apify's per-run minimum, even for 5 results).

Note for the user: **5 results per run is a hard cap**, not a thin market. The skill works around it by slicing the query space. See `frameworks/cruise-data-feed.md`.
</step>

<step name="verify_flights">
The same Apify access prices the flights. Prove that half works too, because a cruise ranking without flights answers a different question.

Run `memo23/google-flights-scraper` with a near-term round trip:

```json
{ "origin": "EWR", "destination": "MIA",
  "departDate": "{today+45d}", "returnDate": "{today+50d}",
  "adults": 1, "currency": "USD", "maxItems": 5 }
```

| Result | Meaning | Fix |
|---|---|---|
| 5 itineraries with `price-usd` and times | Working | Continue |
| `SUCCEEDED` with 0 items in ~5s | Residential proxy was overridden | Restore the proxy default and rerun |
| Fares for a date ~30 days out you didn't ask for | Wrong date param name | It's `departDate` on this actor |
| Actor error | Actor unavailable | Escalate per the fallback table in `frameworks/flight-pairing.md` |

Cost of this check: **~$0.01**.
</step>

<step name="confirm_and_handoff">
Report the final state plainly:

```
I Need Cruise configured.

  Apify   {✅ verified via MCP | ✅ verified via token | ⬜ missing — required}
            cruise feed    vulnv/cruise-data-feed
            flights        memo23/google-flights-scraper

Cost per search: ~$0.55-0.85
  5-8 cruise slices at Apify's $0.10 per-run floor   $0.50-0.80
  3-5 flight lookups at ~$0.01                       $0.03-0.05

Output currency: USD
Gratuity rates: static estimates, last reviewed 2026-07-25

Try: /ineedcruise "cheapest repositioning cruise this spring"
```
</step>

</steps>

<output>
Verified access:
- Apify via MCP or `APIFY_TOKEN` — the only requirement, verified twice: a cruise run returning `taxes_and_fees`, and a flight run returning priced itineraries

No files written. Nothing stored in the skill directory.
</output>

<acceptance-criteria>
- [ ] Per-search cost stated up front, before any signup, including Apify's $0.10 per-run floor
- [ ] Apify stated as the only dependency; no second token implied anywhere
- [ ] MCP path checked before sending the user to sign up for Apify
- [ ] Token exported to shell profile, never pasted into chat
- [ ] Warning given about transcript and shell-history exposure
- [ ] Cruise feed verified by a live run that returned `taxes_and_fees`
- [ ] Flight actor verified by a live run that returned priced itineraries with times
- [ ] The 5-result cap explained as a cap, not a thin market
- [ ] Gratuity rates disclosed as static estimates with their last-reviewed date
- [ ] Final state reported unambiguously
- [ ] User handed a runnable example command
</acceptance-criteria>
