# CLAUDE.md

Guidance for Claude (and humans) working on this repo. Keep it current: tick off **Roadmap** tasks, update **Status**, **Decisions** and **Changelog** whenever something meaningful changes. When a phase changes how the code works, update **How the page works** in the same commit.

## What this is

A single-page "which bins go out this week?" checker for **Croydon Council kerbside collections**, currently for the **Wednesday A** round only, plus a matching iCalendar file. No build step, no framework, no dependencies beyond Google Fonts. Hosted on GitHub Pages.

- [index.html](index.html): the whole app (HTML, CSS and JS in one file, about 400 lines).
- [croydon-bins-wednesday-a.ics](croydon-bins-wednesday-a.ics): calendar file with the same dates, each with an alarm at 8pm the evening before. Not yet linked from the page.
- [README.md](README.md): placeholder.

Run it by opening `index.html` in a browser. There is nothing to install.

## Status

- **Current focus:** planning the multi-schedule rebuild (see **Roadmap**). Waiting on Croydon Council for every round's timetable.
- Timetable source: Croydon 2025-2026 timetable (ref "Kerbside - Wednesday A"), **3 Dec 2025 to 25 Nov 2026**. The page shows "Timetable ended" after that date, so **the 2026-2027 timetables must be live before 25 Nov 2026**.
- Garden waste dates are **estimated**: every other Thursday, anchored on the council-confirmed 1 Oct 2026. They are not part of the published timetable and the page says so.
- Hosting: GitHub Pages at the default `github.io` URL (add the exact URL here). No custom domain, no analytics.

## Roadmap

Written 2026-09-30, before the council's data arrives. Phases are ordered by dependency. **Phases 0–2 don't need the council's data** and can start now, using Wednesday A as the first schedule. Review items from the 2026-09-30 code review are tagged `R1`–`R18` (listed further down) and assigned to the phase where they fit.

**Hard deadline:** the 2026-2027 timetables must be live before **25 Nov 2026**. If the council's data is late, do Phase 3 for Wednesday A alone first, so existing users aren't left with "Timetable ended".

### Phase 0: Decisions and groundwork (no council data needed)

These choices affect later phases, so settle them first.

- [ ] **Decide on the domain before launching the PWA (Phase 5).** Installed apps, saved schedule choices and analytics history all belong to the site's address (its origin). Moving from `*.github.io` to a custom domain later would break every installed app and wipe every saved choice. Recommendation: buy a custom domain now (e.g. `croydonbins.uk`) and point it at the current host. That also lets us change host later without users noticing.
- [ ] **Hosting decision.** See **Hosting and analytics options** below. Recommendation: stay on GitHub Pages on the custom domain.
- [ ] **Analytics provider decision.** See below. Recommendation: GoatCounter.
- [ ] Add `.gitattributes` with `*.ics text eol=crlf` (R4).
- [ ] Rewrite README.md: what the site is, link to the live site, how to update data.

### Phase 1: Central data file and multi-schedule support (can start now with Wednesday A)

Goal: all schedule data lives in one file. The page and the ICS files are both produced from it, and adding a round means adding data only.

- [ ] Design `data/schedules.json` (draft schema below) and move the Wednesday A data into it: `RAW`, `EXTRA`, garden, `ITEMS`, bin metadata.
- [ ] Page loads the data with `fetch()` instead of hardcoded constants. Show a clear message if loading fails.
- [ ] Derive everything that's currently hardcoded from the data: timetable start/end, calendar range (`FIRST`/`LAST`), footer text, "Timetable ended" copy, year labels (R2, R7).
- [ ] Remove the Wednesday assumptions: the `wedcol` class, the `i===2` column highlight and "Wednesday" in the copy should all come from the selected schedule's collection day (R16).
- [ ] Add `scripts/validate.mjs`, a Node script with no dependencies that checks the data before it goes live: valid dates, no duplicates, each date falls on the schedule's weekday unless it has a "moved" note, weeks alternate A/B, dates in range, every bin key exists. Typing dates in from council PDFs is error-prone, so this is the main safety net.
- [ ] Decide how the validator runs. Recommendation: a GitHub Action on every push, so bad data can't be deployed.

**Draft schema** (to finalise once we see the council's format):

```jsonc
{
  "meta": { "council": "Croydon", "updated": "2026-10-01", "source": "Croydon Council timetables 2025-26 & 2026-27" },
  "bins":  { "food": { "name": "Food", "long": "Food caddy", "colour": "food" }, ... },
  "weeks": { "A": ["food", "green"], "B": ["food", "blue", "black"] },
  "schedules": [
    {
      "id": "wed-a",                 // stable forever: used in saved choices, URLs, ICS filenames and UIDs
      "label": "Wednesday A",
      "councilRef": "Kerbside - Wednesday A",
      "weekday": 3,
      "collections": [["2025-12-03", "B"], ["2026-01-02", "B", "Moved to Friday for New Year"], ...],
      "garden": { "dates": [...], "estimated": true },   // or a shared garden round id; depends on council answer
      "extras": { "2026-01-06": "Christmas tree collections start" }
    }
  ],
  "items": [ { "name": "Teabags", "bin": "food", "aliases": ["tea bag", "tea bags"] }, ... ]
}
```

Store explicit dates per schedule rather than a rule like "every other Wednesday". Bank-holiday moves differ by round and year, and the validator checks the pattern anyway.

### Phase 2: Schedule picker with a remembered choice (depends on Phase 1)

- [ ] First visit: a "Choose your collection round" screen listing every schedule, with a link to the council's "find your collection day" lookup for people who don't know their round.
- [ ] Save the choice in `localStorage` (wrapped in try/catch) and restore it on later visits. Show "Your round: Wednesday A · Change" near the top.
- [ ] Also put the choice in the URL (`?round=wed-a`), so links can be shared, bookmarked and installed with the round included. URL beats saved choice beats picker.
- [ ] **iOS issue to test:** home-screen apps on iOS don't share storage with Safari, so a round chosen in Safari won't carry over into the installed app. Putting the round in the URL before the user installs should fix this, because iOS installs the current URL. Check on a real iPhone.
- [ ] Privacy: saving the round only stores something the user explicitly chose, so it doesn't need a consent banner. Mention it in the privacy note (Phase 6).
- [ ] Deep-link format: once live, a link like `?round=wed-a` can be shared by the council, community groups and so on.

### Phase 3: Council data drop-in and ICS files (blocked on council data)

- [ ] Enter every round's 2025-26 and 2026-27 timetable into `schedules.json`. Validator must pass.
- [ ] Add `scripts/build-ics.mjs` to produce `calendars/<id>.ics` for every schedule from the data file: CRLF line endings, lines folded at 75 characters, stable UIDs. Commit the generated files so the site stays build-free. The GitHub Action can check they're up to date.
- [ ] Keep the existing UID format for `wed-a` (`YYYY-MM-DD@croydon-bins-wed-a`) so current subscribers don't get duplicate events. Keep or redirect the old `croydon-bins-wednesday-a.ics` path.
- [ ] Add "Add to calendar" to the page for the selected round: a download link plus a `webcal://` subscribe link, with short instructions for Google, Apple and Outlook (R3). Subscribing is better than a one-off import because new timetables then reach people automatically.
- [ ] Fix the garden date estimates: use confirmed dates from the council if available, and otherwise leave out bank holidays like 25 Dec (R6).

### Phase 4: Review fixes (can be done alongside Phases 1–2)

- [ ] R5: re-render when the tab becomes visible again and at midnight, so a tab left open doesn't show yesterday's answer.
- [ ] R8: `<noscript>` message.
- [ ] R9: replace the calendar's invalid `role="grid"` with a real `<table>`.
- [ ] R10: narrow the `aria-live` region to the headline and date.
- [ ] R11: remove the redundant `aria-label`s on prev/next.
- [ ] R12–R14: search improvements. Add aliases (tea bag, yogurt, pizza box, foil, cling film), expand the item list from the council's A–Z guide, add a "not collected at kerbside" category (e.g. electricals, with a pointer to the tip), and simplify the filter.
- [ ] R17: show one-off events (Christmas trees) in "Coming up".

### Phase 5: PWA (installable, works offline)

Do this after the domain decision (Phase 0) and ideally after Phase 2, so the installed app opens on the user's own round.

- [ ] `manifest.webmanifest`: name "Croydon Bin Day" (or similar), short_name, `start_url: "./"`, `scope`, `display: "standalone"`, `theme_color`/`background_color` taken from the existing colours (`--bg`/`--ink`), icons.
- [ ] **Icons:** one master SVG based on the existing bin illustration, exported as:
  - `icon-192.png`, `icon-512.png` (`purpose: any`)
  - `icon-maskable-512.png` (`purpose: maskable`, artwork within the central 80% safe zone)
  - `apple-touch-icon.png` 180×180 (no transparency)
  - `favicon.svg` plus `favicon-32.png` fallback
  - Optional: `og-image.png` (1200×630) for link previews when shared.
  Keep the master SVG in `assets/icons/src/`. We need a way to export PNGs (design tool, or a small script); decide in this phase.
- [ ] Service worker `sw.js`:
  - pre-cache the app shell (HTML, CSS/JS if split, icons, self-hosted fonts);
  - fetch `schedules.json` from the network first and fall back to the cached copy, so timetable updates arrive promptly but the app still works offline;
  - a version constant that's bumped on every release, with old caches cleaned up.
- [ ] Self-host the two fonts (woff2, Latin subset) instead of Google Fonts. This gives offline support and privacy, and removes a third-party request (R15).
- [ ] Install button:
  - Chrome/Edge/Android: catch `beforeinstallprompt` and show our own "Install app" button;
  - iOS Safari: there's no install prompt, so show a one-off hint: "Tap Share, then Add to Home Screen";
  - hide both when already installed (`display-mode: standalone`), and allow dismissing (remember in `localStorage`).
- [ ] Test on Android Chrome, iOS Safari and desktop Chrome/Edge. Run the Lighthouse PWA/installability checks.
- [ ] Consider splitting CSS/JS out of `index.html` at this point, since the service worker caches them separately anyway (R18). Still no build step.

### Phase 6: Analytics

- [ ] Add the chosen provider (recommendation: GoatCounter; see options below). Just one small script, with no cookies.
- [ ] Custom events worth tracking:
  - `round-selected` (with round id): usage per round;
  - `install-accepted` / `install-dismissed`;
  - `ics-download` / `ics-subscribe` (with round id);
  - `search-no-result` (with the search term): the best signal for which items to add to the list;
  - also track whether the page runs as an installed app or in a browser tab.
- [ ] Short privacy note in the footer or on a `/privacy` page: what's collected, no cookies, the round is stored on the device only.
- [ ] Check the page still works when analytics is blocked. Ad blockers will stop some of it; that's expected.

### Later ideas (not scheduled)

- Push reminders ("Bins out tonight") from the PWA. These need a server to send them (e.g. Cloudflare Workers) and are the main reason we might one day leave GitHub Pages. The ICS alarms cover this for now.
- Postcode → round lookup, if the council can provide the data.
- Other councils, if the data format is generic enough.

## Hosting and analytics options

**Hosting (recommendation: stay on GitHub Pages, on a custom domain).**

| | GitHub Pages | Cloudflare Pages | Netlify |
|---|---|---|---|
| Cost | Free | Free | Free tier |
| Custom domain + HTTPS | Yes | Yes | Yes |
| Custom headers (caching) | No | Yes (`_headers`) | Yes |
| Redirects | No (only meta refresh) | Yes | Yes |
| Server-side functions | No | Workers | Functions |
| Built-in analytics | No | Cloudflare Web Analytics | Paid |
| Fits current workflow | Already set up, deploys on push | Connects to the same repo | Connects to the same repo |

GitHub Pages covers everything in Phases 0–6. It can't set custom headers, but its fixed 10-minute cache is fine for a service worker (browsers check the SW file for updates themselves). Move to Cloudflare Pages only if we need redirects, headers or server-side code (e.g. push reminders). With a custom domain that move is invisible to users.

**Analytics (recommendation: GoatCounter).** Only cookieless tools are listed, which should mean no cookie consent banner under UK GDPR/PECR. Check the current ICO guidance before launch.

| | Cost | Custom events | Notes |
|---|---|---|---|
| **GoatCounter** | Free for non-commercial | Yes | Open source, tiny script, simple dashboard. |
| Cloudflare Web Analytics | Free | No | Pageviews only, so it can't show round choice or failed searches. |
| Plausible | Paid (~£9/month) | Yes | Polished; can self-host. |
| Umami Cloud | Free tier | Yes | Good alternative to GoatCounter. |
| Google Analytics | Free | Yes | Uses cookies, so it needs a consent banner. Overkill here. |

## Questions for Croydon Council

Check the council's reply against these; the answers shape Phases 1–3.

1. A full list of kerbside rounds and their official names/refs (e.g. "Kerbside - Wednesday A").
2. Timetables for **2025-26 and 2026-27** for every round, ideally as CSV or a spreadsheet rather than PDF.
3. Do garden waste rounds follow the kerbside rounds, or are they separate? Confirmed garden dates, including bank-holiday changes.
4. How residents find their round (a lookup URL we can link to). Is postcode/address → round data available?
5. Do flats or communal bins have different schedules we should exclude or handle?
6. When is each year's timetable normally published? This sets our yearly update deadline.
7. Is it OK to use and display their data this way? Any attribution wording they want?

## Decisions

Record decisions here with the date, so they aren't re-argued later.

- _(none yet: domain, hosting and analytics decisions pending in Phase 0)_

## How the page works (index.html)

_Describes the current single-schedule version. Rewrite this section when Phase 1 lands._

Everything runs in one IIFE at the bottom of the file, from hardcoded data:

| Constant | Purpose |
|---|---|
| `BINS` | Bin metadata: short name, long description, CSS colour variable. |
| `ORDER` | Display order of the four kerbside bins. |
| `A`, `B` | The two alternating weeks: A = food + green, B = food + blue + black. |
| `RAW` | The timetable: `[isoDate, A or B, optionalNote]`. A note marks a moved collection (e.g. bank holidays). |
| `EXTRA` | One-off calendar flags keyed by ISO date (currently Christmas tree collections). |
| `GARDEN` | Generated list of garden waste dates, every 14 days from the anchor, clipped to 1 Dec 2025 to 30 Nov 2026. |
| `ITEMS` | "Which bin" search data: `[item, binKey or null, optionalTip]`. `null` = bag on top of recycling. |
| `FIRST`, `LAST` | Month range the calendar can navigate between. |

Render functions, all called once on load:

- `renderHero()`: headline ("Tomorrow", "This Wednesday"…), instruction, moved-date badge, and the kerb illustration (bins out vs. faded). On collection day it keeps today's collection shown all day and offers a toggle to see next week's.
- `renderGarden()`: garden waste card with the next estimated date.
- `renderUpcoming()`: next 8 collections, main and garden merged.
- `renderCal()`: month grid with coloured bars per bin, prev/next navigation.
- `renderResults()`: live search over `ITEMS`.

Dates are built with `new Date(y, m, d)` (local midnight) and compared by timestamp. `key(d)` produces `YYYY-MM-DD`. Day differences use `Math.round(ms / DAY)`, which absorbs the 23/25-hour days around BST changes. Keep this pattern; don't parse ISO strings with `new Date("2026-01-01")` (that's UTC and shifts the day).

### Styling conventions

- Colours are CSS custom properties on `:root`, with a dark palette under `prefers-color-scheme: dark` and an explicit `[data-theme]` override. Add new colours in all three places.
- Fonts: Bricolage Grotesque (display) and Atkinson Hyperlegible (body, chosen for legibility).
- Mobile breakpoint at 480px. Motion is gated behind `prefers-reduced-motion: no-preference`.
- UK English throughout ("colour", "yoghurt", en-GB date formatting). Plain, direct instructions ("Out by 6am").

## The .ics file

- 52 main collections + 26 garden + 1 Christmas tree event = 79 `VEVENT`s. All-day events (`VALUE=DATE`), `TRANSP:TRANSPARENT`, `VALARM` at `-PT4H` (8pm the day before).
- UIDs are stable (`YYYY-MM-DD@croydon-bins-wed-a`, `garden-YYYY-MM-DD@…`) so re-importing updates rather than duplicates. Don't change the UID scheme.
- It was made outside this repo and is edited by hand, and **it duplicates the data in `index.html`** (as of 2026-09-30 they match exactly). Phase 3 replaces it with generated files.

## Updating the timetable (yearly task)

Current manual process, until Phase 1 lands. After that it becomes: add dates to `schedules.json`, run the validator, run the ICS build, commit.

1. Get the new timetable from croydon.gov.uk.
2. Add the new dates to `RAW` (and `EXTRA` for any one-offs). Mark moved dates with a note.
3. Extend the garden window in the `GARDEN` generator and move `LAST` in the calendar section.
4. Update hardcoded end-date text: the footer, the "Timetable ended" message in `renderHero()`, and the "2026-2027" references.
5. Update the `.ics` to match.
6. Update **Status** and **Changelog** here.

## Review backlog (code review 2026-09-30)

Overall the code is tidy and accessibility-minded, and it's correct for its current data: the ICS and HTML schedules agree, the date maths is safe across clock changes, and there's no XSS exposure because all `innerHTML` content is hardcoded (the search term is never injected). Each item is assigned to a roadmap phase.

| # | Issue | Phase |
|---|---|---|
| R1 | Timetable expires 25 Nov 2026 | Deadline, Phase 3 |
| R2 | Two sources of truth (HTML + ICS) | 1, 3 |
| R3 | ICS not linked from the page | 3 |
| R4 | ICS uses LF, RFC 5545 requires CRLF | 0, 3 |
| R5 | `today` computed once; stale if the tab stays open | 4 |
| R6 | Garden estimates include unlikely dates (25 Dec 2025), ignore bank holidays | 3 |
| R7 | Hardcoded year/end-date text scattered around | 1 |
| R8 | No `<noscript>` fallback | 4 |
| R9 | Calendar `role="grid"` is invalid ARIA | 4 |
| R10 | `aria-live` covers the whole hero including the toggle | 4 |
| R11 | Redundant `aria-label`s on prev/next | 4 |
| R12 | Search has no synonyms/alternative spellings | 4 |
| R13 | Search item list is thin (~37 items) | 4 |
| R14 | Redundant `includes` check in the search filter | 4 |
| R15 | Not installable or offline; Google Fonts dependency | 5 |
| R16 | "Wednesday" baked into CSS, JS and copy | 1 |
| R17 | Christmas tree event missing from "Coming up" | 4 |
| R18 | Single file will get unwieldy | 5 |

## Working agreements

- Keep the site itself build-free: plain HTML/CSS/JS served as-is. Node scripts in `scripts/` (validator, ICS generator) are dev tools only, with no npm dependencies unless justified here.
- Schedule `id`s are permanent once published: they're in users' saved choices, shared URLs, ICS filenames and calendar UIDs.
- Test date logic by temporarily overriding `now` (e.g. `const now = new Date(2026, 10, 24)`). Check the hero, garden card, upcoming list and calendar for at least: a collection day, the day before, a moved date, and after the timetable ends. Once there are several schedules, test at least two rounds on different weekdays.
- Check both light and dark mode, and a ~375px-wide viewport, after UI changes.
- Keep the council caveats (garden dates may change, check bin-status) whenever touching garden logic.

## Changelog

- **2026-09-30**: Code review; added CLAUDE.md; wrote the multi-schedule / PWA / analytics roadmap.
- **c2280b3**: Added garden waste (estimated fortnightly Thursdays) to the page and the .ics.
- **f50a6e0**: Initial page and .ics, built in Claude desktop.
