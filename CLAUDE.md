# CLAUDE.md — Номерний Слот 🎰🇺🇦

Context for Claude Code sessions on this project. Read this first.

## What this is
A 2D mobile game: spin a slot to "win" Ukrainian state license plates; a valuation
system scores each plate's "beauty" and prices it (50 ₴ … 5 000 000 ₴). Sell plates or
keep them in a collection. Target: publish on **Google Play first**. The owner communicates
in **Ukrainian** — reply in Ukrainian.

## Architecture (important)
- **Single source of truth: `index.html`** — vanilla HTML + CSS + JS, no framework, no build step.
  Graphics are drawn with DOM/CSS; confetti via Canvas; sound via Web Audio API; everything inline.
- Why single-file: it doubles as (a) the Capacitor `webDir` payload and (b) the shareable browser demo.
- **Native app = Capacitor** wraps `index.html` in an Android WebView → real `.aab` for Google Play.
  We write JS, *not* Kotlin. Native features (ads, billing) come as Capacitor plugins.
- **No backend yet.** State persists in `localStorage` (`ns_*` keys). A server (Firebase is the planned
  path) is only needed later for a real online leaderboard, cloud save, purchase-receipt validation,
  and server-authoritative economy / anti-cheat.

## Files
```
index.html             ← THE GAME (edit this; everything lives here)
www/index.html         ← generated copy for Capacitor (npm run copy:web)
artifact.html          ← generated browser-demo (skeleton stripped from index.html)
capacitor.config.json  ← appId com.numberslot.game, webDir "www"
package.json           ← scripts: copy:web / add:android / sync / open:android
README.md              ← full Android build + Play publishing guide
ROADMAP.md             ← prioritized future plans
CLAUDE.md              ← this file
```

## Workflow after editing index.html (ALWAYS do this)
Run from the project dir (Git Bash / `Bash` tool):
```bash
cp index.html www/index.html
sed -n '/^<body>$/,/^<\/body>$/p' index.html | sed '1d;$d' > artifact.html   # strip skeleton for the demo
sed -n '/^<script>$/,/^<\/script>$/p' index.html | sed '1d;$d' > "$TEMP/game.js" && node --check "$TEMP/game.js"  # syntax check
```
Then **republish the demo** with the Artifact tool using file `artifact.html` and the SAME url
(keeps the link stable): https://claude.ai/code/artifact/e3ec0fdb-7195-46fd-8eff-40fcdb4b1e05
Open locally for the owner: PowerShell `Start-Process "<path>\index.html"`.
Note: the artifact file has NO `<!doctype>/<html>/<head>/<body>` — those come from the publish skeleton.
So in `index.html` keep `<style>` INSIDE `<body>` (the sed strip relies on `^<body>$` / `^</body>$`
and `^<script>$` / `^</script>$` each being on their own line).

## Code map (inside index.html `<script>`)
- `I18N` + `t(path,params)` — 3 languages (uk default / en / ru). Dynamic strings use `t()`;
  reasons are stored as `{k, x}` codes and translated at render via `reasonText()`.
  - **Additive i18n:** new UI strings live in `I18N_EXT` (new top-level keys only), merged via
    `Object.assign` so the big base `I18N` is left untouched. Data-bound labels (regions, assets,
    achievements, special plates) carry their own inline `{uk,en,ru}` instead of i18n keys.
- `REGION{}` — authentic 27 Ukrainian region codes (АА Kyiv … СН Sevastopol) with `w` (spawn weight,
  population-based) and `f` (prestige price factor). `pickRegion()` weighted-picks a code (~12% custom/
  vanity random letters); `regionFactor()` feeds the normal price; `regionLabel()` shows it in the verdict.
- `evaluate(region,digits,series)` — the valuation. Returns `{price, tier, reasons, special?}`.
  - `SPECIAL[]` + `specialFor()` — **exception plates** with fixed prices (ВОР=3M, Агент 007=1.2M,
    ТАТО/МАМА=700k). Checked first; a match overrides normal scoring. Extend by adding `{test, price, tier, label}`.
  - Normal price now also multiplies by `regionFactor(region)`.
- **Progression** (section “Профіль · прогрес · майно”): `xp`+`levelInfo()`/`addXp()` (level-up toast);
  `ACH[]`+`checkAch()` (~10 achievements w/ rewards, checked after spin/sell/keep/buy/daily);
  `claimDaily()` (once/day: +300 ₴ +2 spins). Stats tracked: `totalEarned`,`bestPrice`,`jackpotCount`,`seenSpecials`.
- **Profile** (`renderProfile`): editable `nick` (used in leaderboard), `avatar` picker (`AVATARS`),
  level+XP bar, stats grid, achievements grid, owned assets, equipped background header.
- **Assets shop** (`ASSETS[]`, `renderAssets`, `buyAsset`, `equipBg`): spend ₴ on houses/cars/backgrounds
  (money sink). Shop screen has Currency/Assets sub-tabs (`shopTab`, `setShopTab`).
  - Normal scoring: strongest digit pattern (`put`) × letter multiplier, `0.94–1.06` variance,
    clamped `[50, MAX_PRICE=5_000_000]`, `roundNice()`. tier 0–4 → `RANK[]` colors.
- Slot: `spin()` (handles auto-sell of pending plate, free-spin/cost, `FORCE`/`RIG` cheats, reel anim)
  → `finishSpin()` → `countUp()` (price rolls up) → `celebrate(tier)` (confetti/shake/flash/vibrate/sound).
- Verdict render: `verdictResultHTML(animate)` is the single builder; `renderGameVerdict()` re-renders it
  on language change (so rank + reasons re-translate live).
- Screens/nav: `switchScreen()`, `renderScreen()`, `renderNav()` — 5 tabs: Game / Collection / Leaders / Shop / Profile.
- Collection: `sellCurrent` / `keepCurrent` / `sellFromCollection` / `sellAll`.
- Leaderboard: `genBots()` (99 seeded local bots, persisted) + `leaderRows(tab)`; tabs money / spins.
- Shop: `watchAd()` (+2 free spins, simulated), `buyPack()` (currency, simulated). DEMO stubs only.
- DEV cheats (console, desktop): `luck(true/false)`, `maxwin()`, `vor()`. **Strip before Play release.**

## Hard rules / constraints
- **No hate symbols or extremist content — ever.** 1488 (neo-Nazi), SS, Z-war symbols, etc. are banned
  by Google Play (Hate speech) and would risk the whole app + developer account. Declined on purpose.
- **Monetization stays LIGHT** — the game must remain fully playable for free. Rewarded video → +2 spins;
  optional currency purchase. No pay-to-win gates.
- Keep it **single-file**; don't introduce a build step/framework without a real reason.
- Economy anchors: start 1000 ₴, spin 100 ₴, trash = 50 ₴, top `АА 1111 АА` = 5 000 000 ₴.

## Env
Windows 11, PowerShell primary + Bash tool. Node v24 / npm 11 / git present. Java 8 only — the signed
`.aab` build needs **Android Studio** (it bundles JDK 17). See README.md for the full build/publish steps.
