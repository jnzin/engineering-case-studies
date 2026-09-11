# Three Consumer Apps, Three Mobile Architectures · Solo Developer

## TL;DR
Shipped three consumer apps to both the App Store and Google Play as a solo developer, roughly two weeks each from empty repo to store submission, deliberately building each one on a different mobile architecture: a **Capacitor** wrapper around a Next.js app, a **Flutter** game, and **two native clients** (SwiftUI and Jetpack Compose) over one shared API. **Six live store listings.** The strongest cleared **1,000+ downloads on Google Play and over 1,100 on the App Store**, holds **4.9 stars on Android (32 ratings) and 4.96 on iOS (25)**, and sustains about 1,250 ranked players and ~37 games a day.

## Context
I run a one-person engineering practice, building web products and the data systems behind them.

Framework comparisons never cover the parts that actually cost time, so I answered the mobile architecture question by shipping: one product on each of the three mainstream approaches, each taken all the way to both stores. All three are public and still running; two of them monetize.

| App | Architecture | Live since |
|---|---|---|
| **Álbum 2026** (sticker album collection manager with a camera scanner) | Next.js + Capacitor | May 2026 |
| **Oito a Zero** (squad-building simulation game) | Flutter | Jun 2026 |
| **epilogg** (series, film and anime tracker) | Native SwiftUI + native Kotlin/Compose over a shared API | Jul 2026 |

## Constraints
- **Solo.** No designer, no QA, no second developer, on all three.
- **About two weeks each** from empty repo to store submission. The first two were tied to a demand window (an album collecting season and a tournament) that would close whether or not I was ready.
- **Budget: the $99/year Apple fee and the $25 Google one-time fee**, plus free service tiers. **Zero ad spend**, so distribution had to be organic.
- **Two review processes**, with a different risk profile on each store.
- Each app had to monetize, or at least carry the infrastructure to.

## Options considered

**1. Web wrapped in a native shell (Capacitor).**
Fastest path by a wide margin if web code already exists, and the web build stays a product in its own right. The cost lands entirely at the hardware boundary: the moment the app needs the camera, the filesystem, or anything running at native speed, you are writing platform code anyway, and now you are writing it as a plugin instead of as an app. Bundle size also suffers, since you ship a runtime and your assets.

**2. Cross-platform framework with its own renderer (Flutter).**
One codebase covering iOS, Android and web, with full control of every pixel and reliable 60fps animation, which the WebView approach cannot promise. Nothing looks native on either platform, which is a real cost for a utility app and no cost at all for a game. Dart also makes it natural to isolate logic in a UI-free package, which turned out to matter more than I expected.

**3. Two native codebases over a shared backend (SwiftUI + Kotlin/Compose).**
Genuinely platform-correct UX, immediate access to every new OS feature, and the smallest binaries. The obvious cost is writing every screen twice. The non-obvious benefit is that it forces the domain logic out of the client and onto the server, because duplicating it twice is intolerable, and you end up with a cleaner architecture than you would have chosen deliberately.

## Decision and rationale
No single winner. The rule I came out with:

- **Capacitor when the product is essentially a data UI** and a web version is valuable on its own. It stops being the right answer at the first serious hardware requirement.
- **Flutter when the app *is* the product**: custom rendering, animation, game feel, and logic worth testing headlessly.
- **Native twice when the UX should feel like the platform** and the domain logic already lives on a server, which makes the clients thin enough that writing them twice is affordable.

The deciding question is not "which framework is best". It is **where the product's complexity lives**: in the pixels, in the hardware, or on the server.

## Implementation highlights

- **Capacitor's real limit is the hardware boundary, and I hit it head on.** The scanner app reads printed sticker codes from the live camera. Tesseract.js inside the WebView worked on still images and read garbage from a live feed, so I moved to native OCR: **ML Kit on Android**. On iOS, ML Kit ships CocoaPods-only and would not build under SPM, so I wrote **a Swift plugin over Apple's Vision framework**, testing four orientations per frame. Capacitor does not auto-discover plugins declared in the app target, so it had to be registered through a `CAPBridgeViewController` subclass wired into the storyboard. Dropping Tesseract took the app from **38MB to about 19MB**. This is the clearest evidence I have for the rule above: the web wrapper saved weeks on 90% of the app and gave all of it back on the remaining 10%.

- **OCR-tolerant matching that does not guess.** Character recognition reliably confuses `5/S`, `O/0`, `1/I/L`, `8/B`, `2/Z`, `6/G`, `7/T`. Rather than fuzzy-match and risk marking the wrong sticker, I collapse confusable pairs on **both** the scanned string and the catalog entry, then accept the match only if it resolves **uniquely**. Zero collisions across the 994-code catalog. An earlier version normalized globally (`O→0`, `I→1`) and silently broke every country code containing those letters, so KOR, CIV and COL could never match. Narrowing the normalization to a symmetric, collision-checked comparison fixed the class of bug, not just the instance.

- **Separating the game engine from the UI turned a design argument into a measurement.** The Flutter game keeps its simulation in a pure-Dart package with no UI and no Flutter dependency, which means it runs headlessly. When the difficulty felt wrong, I did not tune it by feel: I ran **3.7 million simulated matches** through Monte Carlo and adjusted the squad-advantage constant from `0.055` to `0.11`, which put a perfect campaign at roughly 8-9% in the best case and under 1% for weak squads. The architectural choice paid off as an empirical one.

- **A ranking that had to be designed against its own players.** The daily challenge gives everyone the same pool of squads. In testing, with player ratings visible, everyone converged on the same optimal XI and the leaderboard degenerated into a tie. Hiding the ratings turned it into a recall game and restored a real distribution, which meant also neutralizing the sort order so the ranking could not leak through it. Scoring is validated server-side, and a scheduled function **closed the first monthly season autonomously in production**: 523 players ranked, podium recorded, titles granted to the winner's profile, counters reset, with no manual intervention.

- **Writing the client twice forced a better backend.** The tracker runs one Node + Hono + Drizzle + Postgres API with TMDB as the catalog source, cached locally, plus anime id mappings reconciled across TMDB, AniList and MAL. Both clients are thin. The migration importer accepts a competitor's GDPR export in two formats, one of them undocumented, by detecting the shape rather than trusting the file extension, skips the sensitive files inside the archive (tokens, device records), resolves series by TVDB id through TMDB's `/find` with a name-search fallback, and preserves original watch dates.

- **A native rating prompt measures the moment you place it in.** The game asks for a store rating once per install, after a player wins their first championship campaign, with the "already asked" flag written before the call rather than after, because the OS rate-limits the prompt and a silent retry would burn the one chance. The native dialog renders in place and the user answers without leaving the app, so what comes back is what they feel at that moment, good or bad. That is the honest reading of the mechanism, and it cuts both ways: the trigger point decides which users you sample and in what state, so the same app prompted after a win and after a defeat would not score the same. Shipping without a prompt, as the tracker did, is not a neutral choice either. It just means no signal at all.

- **Store review is a schedule input, not a formality.** The album app was **rejected four times by Apple** (intellectual property under 5.2.1, then business model under 2.1b, then App Tracking Transparency under 2.1) while Google approved the same build in one pass. The game cleared Apple in under 24 hours. Review risk correlates with what the app *claims*, not with how well it is built, and I now price that into timelines.

- **Monetization shipped, not deferred.** AdMob on two apps with `app-ads.txt` verification and store linking, a lifetime PRO tier through RevenueCat on the scanner, and in the game an earned-currency economy where a rewarded ad is one of the ways to earn, spent on cosmetics. Revenue is modest and I would not present it otherwise, but all of it is live, which is a different thing from having planned it.

## Results
- **Three apps, six live store listings**, all published solo, **about two weeks each** from empty repo to submission, with **zero marketing spend**. Every download was organic.
- **Oito a Zero** (Flutter): **1,000+ Google Play downloads and over 1,100 on the App Store**, **4.9 stars** (32 ratings) on Android and **4.96 stars** (25 ratings) on iOS, about **1,250 ranked players**, **~37 matches per day**, and a monthly season that closes itself.
- **Álbum 2026** (Capacitor): **1,000+ Google Play downloads**, 3.8 stars (11 ratings), and a small fraction of that on iOS. A 1,074-entry catalog with live-camera OCR, cloud backup on a shareable album code, AdMob plus a RevenueCat lifetime tier.
- **epilogg** (native x2): 5+ downloads on Google Play and 6 on the App Store. Native iOS and Android clients over one API, with achievements, shareable 1080x1920 stat cards and an importer for a competitor's data export.
- **Seasonal demand buys volume, not retention.** Both football apps cleared 1,000 Google Play downloads off the same tournament window, but the game holds roughly 510 active installs to the sticker app's 300, and sustains daily play on top of that. The one with a reason to come back every day is the one that kept people. The sticker app was, correctly, a seasonal tool: useful while the album was being filled and not after.

## What I'd do differently

- **Match the acquisition mechanism to the category, before writing code.** Two of these launched into a seasonal demand spike: for a few weeks a lot of people search for something specific and no incumbent owns the term, so the downloads arrive on their own. The third entered an evergreen category against established competitors, where demand is steady, and steady demand is already served. It has the cleanest architecture of the three and **five downloads**. The gap between 1,000 and 5 is not a marketing failure I could have closed with more effort after shipping. It is the difference between a category that hands you distribution and one where you have to buy it, and I picked the second without deciding how I would pay. The two weeks of build were not the mistake; spending them before answering that question was.

- **Ship the server-controlled surfaces in version one.** The game has an upgrade prompt that was never activated, because the config document it reads was never created, and its copy is baked into the client, so it cannot be changed without a release. The lesson generalizes past version gates: a force-upgrade block, a maintenance or outage notice, an announcement banner, a feature kill switch. Anything you might need to say or switch off later has to already exist in the build people have installed, and its text has to live on the server rather than in the binary. It is the one category of work that genuinely cannot be added after the fact, and it costs an afternoon before launch.

---

**Stack:** Swift / SwiftUI · Kotlin / Jetpack Compose · Flutter / Dart · Capacitor · Next.js · TypeScript · Node.js · Hono · Drizzle · PostgreSQL · Firebase (Firestore, Auth, Cloud Functions, FCM) · Google ML Kit · Apple Vision · AdMob · RevenueCat

**Live:**
Álbum 2026 · [App Store](https://apps.apple.com/br/app/scanner-figurinhas-%C3%A1lbum-2026/id6772548640) · [Google Play](https://play.google.com/store/apps/details?id=com.jonlabs.figs)
Oito a Zero · [App Store](https://apps.apple.com/br/app/oito-a-zero-futebol-7-a-0/id6784423053) · [Google Play](https://play.google.com/store/apps/details?id=com.jonlabs.cupgame)
epilogg · [App Store](https://apps.apple.com/br/app/epilogg/id6790479509) · [Google Play](https://play.google.com/store/apps/details?id=br.com.jonlabs.epilogg)
