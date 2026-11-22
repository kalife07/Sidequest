# How Sidequest works, concept by concept

> **For:** the team, the pitch, Devpost and judges' "how does it…?" questions.
> **Owner:** Anthony. Every rule and number below is taken from the code on `main` (Sun Nov 22). File paths are given so anyone can check.
> Short version for the stage: [`docs/demo.md`](demo.md).

---

## 0. The whole thing in one picture

```
 WHO            Google Calendars (times only) · postal code → home · interests · friends
 WHERE          60 curated places · ~4,000 city events · 55,000-business registry · 2,545 OSM venues
 CONDITIONS     hourly weather · travel time per person per place
        │
        ▼
 FEED ENGINE (plain code, no AI, ~200 ms)
   1 time grid → 2 hard rules → 3 score (7 factors) → 4 learning → 5 order + variety → 6 cards
        │                                                                   │
        ▼                                                                   ▼
 CLAUDE: one-line caption per card (never ranks)              SEARCH: words → filters → same engine
        │
        ▼
 PLAN: Let's go → invite → live banner on friends' phones → I'm in → confirmed → Google Calendar
       → Book it · live location · memory photos → Memories
```

Every card answers **what** (a place or event), **when** (a real time slot) and **who** (friends free for the whole visit).

---

## 1. Where places come from, and how each one is labelled

There are **four kinds of places**. Each becomes the same `Place` object (`types/domain.js`): name, category, tags, price level (1–4), coordinates, opening hours per weekday, typical visit length, indoor/outdoor, photo. What differs is how much we **know** about it.

### 1a. The 60 curated places (the main feed)
`data/places.json`, hand-picked by Adel. Montréal cafés, restaurants, bars, parks, viewpoints and activities (Jean-Talon market, the Biodôme, Mont-Royal lookouts…).

| Field | How it's set |
|---|---|
| **Category** | one of `cafe, restaurant, bar, park, viewpoint, activity` (`event` is only for city events) |
| **Tags** | from a fixed vocabulary of 15 interests (`coffee, cheap-eats, views, outdoors, bars, live-music, art, museums, markets, brunch, iconic, free, late-night, cocktails, nature`) plus a cuisine tag for restaurants (`portuguese`, `indian`…) |
| **Price** | `$`–`$$$$` (1–4) |
| **Hours** | verified per weekday, several intervals allowed (lunch + dinner); a close earlier than the open = closes after midnight (bars) |
| **Visit length** | `typicalDurationMin` (60 min for a café, 90–120 for a museum or a bar) |
| **Indoor** | true/false (parks and viewpoints are outdoor: they disappear in the rain) |
| **Photo** | one real photo per place (`public/places/<id>.jpg`) |
| **Contacts** | verified website / booking link / phone in `lib/booking/contacts.js`, for **Book it** |

These are the only places with **verified hours**, so they rank first (see §5).

### 1b. City events (Montréal open data)
`public.events`: ~4,000 events from montreal.ca, with exact times and the official picture read from each event page. `lib/data/events.js` + `lib/data/eventRules.js`:

- **Kept:** music, theatre, cinema, dance, exhibitions, fairs & markets, comedy, circus, spoken word, games, sports & outdoors. **Dropped:** kids-only and seniors-only events, online events, events with no location or no exact start.
- **Near you:** within **8 km** of your home, the nearest 40 (about 150 fit this week).
- **Label:** category `event`; tags from the event type (Musique → `live-music`, Exposition → `art, museums`…) + `free` if free + `outdoors` if outside; price `$` if free, else `$$`; the official photo (or a fallback picture); **hours = the event's own window** (a 7:15 PM concert is "open" 7:00–9:30 PM that day only); a multi-day exhibition uses its daily hours (or 11:00–17:00).
- The card shows **"Starts 7:15 PM"** (exact start) and **Buy tickets / event page**. Other dates of the same show become that card's "other times" instead of separate cards.

### 1c. Registry restaurants (search only)
The city's food-establishment registry: **54,937 businesses**, of which ~5,700 are open restaurants. It has name, address and coordinates, but **no cuisine, hours, price or photo**. So:

- **Cuisine is guessed from the name** (`lib/cuisines.js`, 19 cuisines): whole words like *indienne, tandoori, masala* → `indian`; *sushi, ramen, izakaya* → `japanese`; plus well-known names without a cuisine word. A restaurant whose name doesn't say its cuisine can't be found that way.
- **Only used when you search a cuisine** ("indian food tonight"): up to 40 within 6 km.
- **Label:** category `restaurant`, tag = the cuisine, **assumed hours 11:30–22:00**, price `$$`, a generic picture, and **`hoursVerified: false`** → **"Hours not verified · call ahead"**. The badge is shown in the place's details sheet and on the plan once you lock it in, **not on feed cards** (team choice: it cluttered the feed).

### 1d. OpenStreetMap venues (search only)
`lib/geo/venues.json`: 2,545 named venues (bowling, karaoke, climbing, escape rooms, cinemas, hotels…). Each **kind** maps to a category, tags, a typical visit and assumed hours (karaoke 18:00–02:00, bowling 11:00–23:00…). Also "hours not verified". Added only when a search asks for that kind of thing or that name.

---

## 2. People: who you are, who your friends are, when you're free

| Concept | How it works |
|---|---|
| **Account** | Google sign-in through Supabase Auth. First sign-in creates your user row and sends you to onboarding (calendar → home → interests → friends). |
| **Home** | your **postal code**, but only its first 3 characters are kept (e.g. `H2T`, a neighbourhood), geocoded once with OpenRouteService. Outside Québec/Montréal codes are refused. Your exact address is never stored. |
| **Interests** | 5+ chips from the 15 tags above. They are a *head start* for the interest score; your swipes then take over. |
| **Friends** | added from the Friends tab (search by **@username** or name). A friendship is stored **both ways** with a closeness 1–3. Today it's one-sided: adding someone doesn't need their acceptance (friend requests are on the "next" list). |
| **Busy times** | from **Google Calendar**, read at each sign-in for the next 14 days: **only start/end times**, never titles. The calendars you ticked in Google; all-day events, cancelled and declined events are ignored; class timetables marked "Free" still count as busy. |
| **Travel times** | for every person × curated place, from **OpenRouteService**: walking time; if any walk is over 25 min, that person's times switch to **bike time** (our stand-in for metro/BIXI, ORS has no transit). If routing fails: a straight-line estimate (12 km/h × 1.3 detour + 5 min) shown with **"~"**. Events, registry and venues always use the straight-line estimate ("~"). |
| **Weather** | Open-Meteo hourly forecast for Montréal (temperature, rain probability, icon), cached 30 min. For the demo a rainy afternoon can be staged with `FORCE_RAIN_DAY`. |

---

## 3. The feed engine, step 1: the time grid

`lib/engine/slots.js`. Instead of comparing calendar intervals, the next **7 days** are cut into **30-minute slots** from **09:00 to 23:00** (Montréal time), starting **at least 1 hour from now**.

For each slot and each person: **free or busy**. A slot is busy if any busy block touches it, **widened by a 15-minute buffer on both sides** (`ENGINE.bufferMin`), the time to get there and back. A class at 10:05 makes the 9:30–10:00 slot busy, so nothing is suggested to end at 10:00; a class ending at 11:00 makes 11:00–11:30 busy.

A **visit** = consecutive slots, as long as the place needs: `max(60 min, typical visit)` rounded up to 30 minutes, never across two days.

---

## 4. Step 2: hard rules (a visit is thrown out if any fails)

`lib/engine/candidates.js`, checked cheapest first, for every place × every possible start time (~10,000 visits per feed):

| Rule | Detail |
|---|---|
| **Search filters** | category, price cap, indoor-only, cuisine (only when you searched) |
| **You can get there** | your travel ≤ **35 min**. Friends over 35 min are simply not on that card. |
| **Open the whole visit** | every slot inside the opening hours (split lunch/dinner and after-midnight closings handled). No hours = closed. |
| **You're free the whole visit** | (with the 15-min buffer) |
| **Weather known** | no forecast for an hour → skipped (we never invent weather) |
| **No outdoor place in the rain** | rain probability over **60 %** during the visit → parks/viewpoints dropped, indoor places stay |
| **At least one friend free the whole visit** | the "who" of the card = every friend who is free for **all** of it and within 35 min. With a search like "with Adel", Adel must be one of them. |

---

## 5. Step 3: scoring. Why one restaurant beats another

`lib/engine/scoring.js`. Every surviving visit gets **7 scores between 0 and 1**, weighted (`SCORE_WEIGHTS`). This is exactly the **"Why am I seeing this?"** sheet.

| Factor | Weight | Exact rule |
|---|---|---|
| **Interest** | 30 % | For each of the place's tags: your learned weight for that tag + **0.5 if it's one of your interests**; averaged over the tags, plus your learned weight for the place's **category**; squashed to 0–1 (`sigmoid(3 × total)`). A place matching nothing of yours ≈ 0.5. |
| **Friends** | 20 % | (sum of closeness of the friends free for the whole visit) ÷ (sum of closeness of all your friends). Whole group free = 1. |
| **Timing** | 15 % | best hours per category (Mhammad's `timing.js`): restaurants 11:30–13:30 and 18:00–20:30, cafés 8:00–11:30 and 14:00–17:00, bars 20:00–02:00, viewpoints around sunset (15:45–16:45 in late Nov), parks 11:00–15:00, activities 13:00–19:00. Inside = 1, then −0.5 per hour away (minimum 0.1). **Events always 1** (their time is part of what they are). |
| **Weather** | 10 % | indoor: 0.8 (1 when it's raining, it's the good choice). Outdoor: (1 − rain %) × warmth (−5 °C → 0, 20 °C → 1). |
| **Fairness** | 10 % | 1 − (the **longest** trip in the group ÷ 35 min). The person travelling the most decides, so nobody gets a 35-min trek while the others walk 5. Bike trips count **double** here (they're our metro stand-in: "15 min" by bike can be 3 km). |
| **Novelty** | 10 % | your last interaction with this place in the past 14 days: none → 1, saved → 0.6, skipped → 0.2, already planned → 0.1. |
| **Proximity** | 5 % | 1 − (your own trip ÷ 35 min), bike trips counted double. |

**Learned dislike:** if your learned weight for a category is ≤ −0.5 (2 skips, §6), every visit of that category is multiplied by **0.3**: it drops out of your top 10.

### A real example (Anthony's feed simulated at demo time, Sun 1:30 PM)
Anthony's interests: live-music, markets, coffee, views, cheap-eats, outdoors, nature. Friends: Oscar, Mohammad, Adel.

| | **Ma Poule Mouillée** (card #2) | **Beauty's Luncheonette** (card #46) |
|---|---|---|
| Tags / price | cheap-eats, portuguese · `$` | brunch, iconic · `$$` |
| When | Sun 7:00–8:00 PM | Sat 12:30–1:30 PM |
| Interest (30 %) | **0.68**: "cheap-eats" is one of his interests | 0.50: no tag matches him |
| Friends (20 %) | 1.00: all 3 free | 1.00: all 3 free |
| Timing (15 %) | 1.00: dinner hour | 1.00: lunch hour |
| Weather (10 %) | 0.80: indoor, dry | 1.00: indoor, rainy (good choice) |
| Fairness (10 %) | **0.34**: longest trip 23 min, walking | 0.14: a 15-min bike ride counts as 30 |
| Novelty (10 %) | 1.00 | 1.00 |
| Proximity (5 %) | **0.43**: he walks 10 min | 0.14: bike ride |
| **Total** | **0.789** | 0.721 |

Both are open, both have the whole group free, both at a perfect hour. Ma Poule Mouillée wins on **what Anthony likes** (cheap-eats) and on **distance** (walkable for him, shortest longest-trip for the group).

---

## 6. Step 4: learning from your swipes

`lib/engine/learning.js`, saved per user in `user_tag_weights`:

| Action | Category weight (`cat:bar`) | Tag weights |
|---|---|---|
| **Skip** | −0.25 | −0.25 **split** across the place's tags |
| **Save** | +0.15 | +0.15 split |
| **Let's go** | +0.30 | +0.30 split |

- All weights stay between −1 and +1.
- **2 skips of bars** → `cat:bar` = −0.5 → bars are suppressed (× 0.3). On the next refresh they're gone from the top 10 (the demo moment).
- The tag step is split, so skipping a bar that also serves coffee teaches "not bars" much more than "not coffee".
- A user's swipes are applied one at a time (server queue + the app sends them in order), so fast skipping never loses one.
- If you skip *everything*, everything ends up suppressed and nothing stands out any more (that's why the presenter's weights are reset before the demo).

---

## 7. Step 5: order, variety and the endless feed

`lib/engine/index.js`, `diversify.js`, `variety.js`.

1. **Order score** = the true score + adjustments that only change the order (the card still shows its true match %):
   - **+1 for curated places** (verified hours), so they always come before search-only places;
   - a **random shake of ±0.07** per place, different each time you open the app (a "visit"), so close calls swap places;
   - **−0.15 for places you looked at in your last visits** (the app remembers the last 30 places shown on screen).
2. **Best times per place:** for each place, its best visit on each day; keep its **2 best days**.
3. **Variety pass:** cards are picked one by one; a card loses 0.15 if its category appeared in the **previous 3 cards** and 0.06 if its day appeared in the **previous 2**. **A place never appears twice**, and a show with several dates appears once.
4. **"Also free" times:** each card carries up to **3 other good times** for the same place (another day, or 2+ hours apart), each passing every hard rule with its own list of free friends. Picking one in the invite sheet plans that time instead.
5. **Pages:** 10 cards per page; the next page loads when you're 5 cards from the end. **Pass 1** = each place's best time; **pass 2** = each place's other good day; **pass 3+** = the list again, reshuffled. The feed never ends.
6. **Tap Feed while on the feed** = a new visit (new shake, places you just saw move down) from the top. Swipe left/right to change tab.

Measured on our real data: a teammate's top 5 changes almost completely from one visit to the next (0.3 of 5 places repeat), and about half of the first 20 cards are walkable.

---

## 8. The caption (Claude)

`lib/ai/captions.js`, model **`claude-haiku-5-5`**:
- **One** Claude call per page of 10 cards, with only the card's facts (place, time, who, travel, weather).
- Rules in the prompt: at most 80 characters, only the given facts, no invented prices/menus/adjectives, never "cheap" (say "budget"), no emojis.
- The answer is checked (Zod schema); cached per card + friends.
- The first page waits **0.8 s** at most; if Claude is slower, a **template caption** goes out ("Antoine, Maya and Léa are free too · 9°C and cloudy") and Claude's version is ready for the next page and the next visit.
- **Claude never decides what you see or in which order.**

---

## 9. Search

`lib/ai/search.js`, `lib/ai/searchTerms.js`, `app/api/search/route.js`. **Plain code, no AI**: your words become filters, then the same engine runs.

| You type | Becomes |
|---|---|
| cheap, budget, student, free… | price ≤ `$` |
| a friend's name ("with Adel") | Adel must be free on every card |
| tonight / tomorrow / this weekend / Saturday / morning / afternoon / evening | a time range (tonight = 18:00–23:00) |
| coffee, drinks, park, view, museum, concert, expo… | categories (café, bar, park, viewpoint, activity, event) |
| indoor, if it rains | indoor only |
| indian, sushi, pizza… (19 cuisines) | the cuisine filter + matching **registry restaurants** near you |
| bowling, karaoke, climbing, cinema… | places whose name/tags match, incl. **OSM venues**; if none: "No bowling nearby · showing Activities" |
| a name ("Schwartz's") | name search over our places, the registry and venues |
| anything else | ignored, and the summary says so honestly ("Nothing called “xyz” nearby") |

The summary chips (e.g. *Budget · with Adel · Tomorrow night*) show exactly what was understood; ✕ clears the search.

---

## 10. Plans: from "Let's go" to the calendar

`lib/social/plans-store.js`, `app/api/plans/*`, `components/plans/*`.

### Who gets the invite
1. **Let's go** (swipe the orange bar) opens the invite sheet with **only the friends shown on that card**: your friends who are free for the whole visit and within 35 min. All are ticked; untick anyone. You can also pick one of the "also free" times.
2. **Send** → `POST /api/plans`. The server (never the browser) decides who you are; the invite list is de-duplicated and **you're removed from it**; **every invitee is re-checked free** (with the buffer) right now, so a stale feed can't invite someone who just got busy → "Maya isn't free then".
3. A plan row (status **pending**) + one member row per person (you = **in**, them = **invited**).
4. The server sends a **"plan changed" signal** (Supabase Realtime Broadcast) to the plan's channel and to **each member's own invite channel**. Only members get it, and the signal carries **no data**: each phone refetches through the API, which checks membership again.
5. On the invitee's phone: the **"Anthony invited you to …" banner** slides down on the feed (and a badge on the Plans tab) in **under a second**. If the live connection drops, the app checks every 2–3 s anyway.
6. Non-members can't open the plan (403), not even with the link.

### Statuses
| Event | Status |
|---|---|
| Plan sent | **pending** ("Waiting for replies") |
| **At least one** invitee says I'm in | **confirmed** |
| Every invitee says Can't make it | **cancelled** (final) |
| The only "in" changes to "out" | back to **pending** |
| Organizer cancels | **cancelled** (final), RSVPs then refused |

### Google Calendar
On **I'm in**, the plan is added to that member's **primary Google Calendar** (and to the organizer's when the plan becomes confirmed). Can't make it / cancelled → the event is removed. It uses the long-lived token saved at sign-in (scope `calendar.events`, server only), runs after the response so an RSVP never waits for Google, and simply skips people who didn't connect Google. The plan screen shows "Added to your Google Calendar ✓".

### "Start a sidequest now"
From My plans: a 1-hour plan starting in ~10 minutes. **Your own calendar is ignored** (you're saying you're free); your friends' calendars count, with the buffer. Only curated places, open the whole hour, within 35 min, not outdoors in the rain. Best = the most free friends, then the fairest longest trip, then your own trip. Then it's a normal invite.

### Book it
On a confirmed plan, the organizer gets a **booking type**: events → tickets (or nothing if free); a place with a booking link → online; restaurants → phone; cafés/bars → walk-in; parks/viewpoints → nothing to book. Plus party size (members in), a ready message in **French and English**, the links/phone (only verified ones, else Google Maps), and **Mark as requested** (everyone sees "Reservation requested"). The **booking assistant** (Claude) reads the place's **official website** (our verified list only) and reports the booking link or the site's "no reservations" sentence. Every answer is checked against the page: the link must be one of the page's real links, the quote must appear word for word. Sidequest never books anything itself.

### Live location (from 30 min before)
Opt-in, members only, from **30 min before to 30 min after** the start, at most one update every 15 s. Others see **status only**: *arrived* (within 150 m), *8 min away*, *running late* (ETA after the start) or *not sharing*. **Never coordinates.** Rows are deleted after the plan.

### Memory photos → Memories
In the first 15 minutes of a plan and around halfway, a banner says "📸 Take a memory photo"; members can add photos from 15 min before the start until 12 h after the end. Photos are resized in the phone (1280 px), checked on the server (real JPEG/PNG/WebP, ≤ 5 MB), stored in a **private** bucket and shown with links that expire after 1 hour, to members only. **Memories** (its own tab, camera icon) gathers every photo from all your plans: a story ring per outing, then month → outing → grid; tap an outing to open its plan.

---

## 11. Live updates

Supabase Realtime **Broadcast**: after any plan change, the server sends "plan X changed" to `plan:<id>` and `user:<memberId>:invites`. Phones listening on those channels refetch through the API. Measured with 3 phones: invite banner in **157 ms**, "confirmed" in **119 ms**, catch-up after going offline in **90 ms**. The signal has no data, so a stranger listening learns nothing; polling (2–3 s) is the backup.

---

## 12. Privacy and security

- Calendar: event **times only**, never titles. Location: opt-in, members only, 30 min around the plan, status/ETA only. Photos: private bucket, expiring links. Home: postal-code area only.
- **Row-level security on every table**; tested with real user tokens: nobody can read another user's busy times, swipes, weights, travel or plans they're not in; Google tokens, locations and photo rows are server-only.
- Identity always comes from the session, never from the request; every input is validated (Zod) → clean 4xx errors.
- No secrets in the browser code (checked on the built bundle).

---

## 13. Numbers for the pitch

| | |
|---|---|
| Curated places | 60 (12 restaurants, 12 activities, 10 bars, 9 cafés, 9 parks, 9 viewpoints) + ~150 city events this week |
| Search-only places | ~5,700 open registry restaurants (19 cuisines) + 2,545 OSM venues |
| Visits checked per feed | ~10,000, in ~200 ms |
| Score factors | 7 |
| Live updates | < 200 ms (3 phones measured) |
| Automated tests | 626, plus an overnight QA run: ~1,300 cards for 39 test users checked against the database, **0 rule violations** |

