# Airport & City Code Resolution

Three jobs: map a cruise disembark port to its nearest airport, turn what a human said into an IATA code, and expand a city into the metro airport list the flight actor accepts.

---

## Cruise port → airport

Used by `frameworks/flight-pairing.md`. **Cruise ports are frequently not the city with the airport.** Transfer time is a real cost; flag anything over an hour.

| Port | Airport | Transfer |
|---|---|---|
| Cape Liberty / Bayonne | EWR | 20 min |
| Miami | MIA | 20 min |
| Fort Lauderdale | FLL | 10 min |
| Port Canaveral | MCO | 50 min |
| Galveston | HOU / IAH | 60-90 min ⚠️ |
| New Orleans | MSY | 30 min |
| San Juan | SJU | 20 min |
| Seattle | SEA | 30 min |
| Vancouver | YVR | 30 min |
| **Seward** | ANC | **2.5 hours** ⚠️ |
| Whittier | ANC | 90 min ⚠️ |
| San Diego | SAN | 15 min |
| Los Angeles / Long Beach | LAX / LGB | 30-45 min |
| **Civitavecchia** | FCO (Rome) | **90 min** ⚠️ |
| Barcelona | BCN | 25 min |
| **Southampton** | LHR / LGW | **90 min** ⚠️ |
| Dover | LHR / LGW | 2 hours ⚠️ |
| Amsterdam | AMS | 30 min |
| Rotterdam | AMS | 60 min ⚠️ |
| Cherbourg | CDG (Paris) | 3.5 hours ⚠️ |
| Le Havre | CDG | 2.5 hours ⚠️ |
| Copenhagen | CPH | 20 min |
| Venice | VCE | 30 min |
| Athens / Piraeus | ATH | 45 min |
| Lisbon | LIS | 20 min |
| Málaga | AGP | 20 min |
| Palma | PMI | 20 min |
| Dubai | DXB | 30 min |
| Singapore | SIN | 30 min |
| Sydney | SYD | 30 min |
| **Colón** | PTY (Panama City) | **90 min** ⚠️ |
| Cartagena | CTG | 20 min |
| Buenos Aires | EZE | 60 min |
| Santiago / Valparaíso | SCL | 90 min ⚠️ |
| Rio de Janeiro | GIG | 40 min |

For ports not listed: resolve the nearest major airport from context and **state the assumption plus estimated transfer time** in the output.

---

---

## Metro groups

When the user names a city rather than an airport, search the whole metro in one query. `memo23/google-flights-scraper` accepts a comma-separated `origin`, so `"EWR,JFK,LGA"` costs the same as `"EWR"` and frequently finds a cheaper fare out of the airport they didn't think to check.

| Metro airports | City code |
|---|---|
| EWR, JFK, LGA | **NYC** |
| MCO, SFB | **ORL** |
| LHR, LGW, STN, LTN, LCY | **LON** |
| ORD, MDW | **CHI** |
| DCA, IAD, BWI | **WAS** |
| CDG, ORY, BVA | **PAR** |
| NRT, HND | **TYO** |
| FCO, CIA | **ROM** |
| MXP, LIN, BGY | **MIL** |
| SVO, DME, VKO | **MOW** |
| YYZ, YTZ | **YTO** |
| GRU, CGH, VCP | **SAO** |
| EZE, AEP | **BUE** |
| BCN | BCN (no city alias) |
| LAX | LAX (no city alias) |

**Rule:** query the full metro list, then present the specific airport back to the user. They care that it's Newark, not "NYC".

---

## US origin airports — quick reference

| City | Primary | Also |
|---|---|---|
| New York | JFK | EWR, LGA |
| Los Angeles | LAX | BUR, LGB, SNA, ONT |
| Chicago | ORD | MDW |
| Washington DC | IAD | DCA, BWI |
| San Francisco | SFO | OAK, SJC |
| Miami | MIA | FLL, PBI |
| Boston | BOS | PVD, MHT |
| Dallas | DFW | DAL |
| Houston | IAH | HOU |
| Seattle | SEA | — |
| Denver | DEN | — |
| Atlanta | ATL | — |
| Philadelphia | PHL | — |
| Phoenix | PHX | AZA |

**Charles's home region:** New Jersey → **EWR** primary, JFK and LGA as alternates, PHL as a wildcard for European routes.

---

## Multi-airport cities — disclosure rule

When a user names a city with several airports, search the primary and **say which one you used.** Offer the alternates only if a candidate is close to the budget ceiling, where a nearby-airport swap might clear it.

Never silently pick an airport and present it as "flights to London." LGW and LCY are not interchangeable to someone with a connection to make.

---

## Warm-destination filter

For "somewhere warm" queries, filter the sweep to these regions rather than asking the user to name a place:

**Caribbean:** SJU, MBJ, PUJ, NAS, AUA, CUR, GND, SXM, BGI, ANU, STT
**Mexico / Central America:** CUN, PVR, SJD, MEX, SJO, LIR, PTY, BZE
**US warm:** MCO, FLL, MIA, TPA, PHX, LAS, HNL, SAN
**Europe (shoulder season):** LIS, AGP, PMI, ATH, IST, AYT, FAO
**South America:** CTG, BOG, GYE, LIM, BUE

---

## Notes

- IATA codes are three letters, uppercase. Reject anything else before it reaches the API.
- `depart_date` accepts `YYYY-MM` (whole month) or `YYYY-MM-DD` (specific day). Month form is what the sweep uses.
- This table covers the common cases. For anything unlisted, resolve the code from context and state the assumption in the output.
