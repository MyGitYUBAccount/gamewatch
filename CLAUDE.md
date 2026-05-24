# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

GameWatch is a fully client-side static site for parents to monitor their kids' Roblox activity. It is two standalone HTML files — there is no build system, no package manager, no dependencies, no tests, and no backend.

- `index.html` — marketing landing page (hero, features, pricing, FAQ).
- `app.html` — the entire app: dashboard, child profiles, polling loop, notifications. All CSS and JS are inline.

## Running locally

There is nothing to install, build, lint, or test. Serve the directory with any static server and open the pages in a browser:

```
python3 -m http.server 8000
# then visit http://localhost:8000/         (marketing)
# or       http://localhost:8000/app.html   (the app)
```

Opening the files via `file://` mostly works, but `Notification.requestPermission()` is unreliable on some browsers without an HTTP origin.

## Data model

All state lives in browser `localStorage` under three keys (`app.html:249-253`):

- `gw_children` — array of `{ id, name, accounts: [{ game, username, userId }], status, sessionStart, lastSeen }`.
- `gw_logs` — most recent 50 notification events, newest first (capped in `sendNotification`, `app.html:290`).
- `gw_pro` — string `"1"` if the user is "Pro". This is the only Pro check anywhere in the app; there is no server.

`save()` (`app.html:256`) re-serializes children + logs after every mutation. There is no migration logic, so changing the stored shape will silently break existing users' data.

## The polling loop

`startPolling` (`app.html:350`) runs `pollAll` immediately and then every `isPro ? 10000 : 30000` ms. `pollAll` (`app.html:308`):

1. Collects every `(childId, userId)` pair whose `account.game === 'roblox'` (other games are skipped entirely).
2. Calls `checkRobloxPresence` with all userIds in one batch.
3. For each result, diffs the new `userPresenceType` against `prevStatuses[childId + '-roblox']` and fires `sendNotification` **only on transitions**: offline→online, online→offline, or any→in-game. Steady states do not notify.
4. Sets `child.sessionStart` on going online, clears it and sets `lastSeen` on going offline. The live session timer in the UI is driven by a separate 1s `setInterval` that only re-renders if any child has an active `sessionStart` (`app.html:523`).

## Roblox API surface

Two public endpoints, both CORS-friendly, both called from the browser (`app.html:262-282`):

- `POST https://users.roblox.com/v1/usernames/users` — resolves a username to a numeric `userId`. Called from `addChild` when a child is added (`app.html:483`).
- `POST https://presence.roblox.com/v1/presence/users` — returns `userPresences[]` with a `userPresenceType` integer.

The `STATUS` map at `app.html:242` is the source of truth for those integers: `0`=Offline, `1`=Online, `2`=In-Game, `3`=In Studio. The UI label, color, and dot color all come from this map.

## Multi-game caveat

The `GAMES` object (`app.html:235`) and the add-form `<select>` (`app.html:179-185`) list Roblox, Fortnite, Minecraft, Apex, Valorant — but **only Roblox is actually implemented**. A user can pick any of the others and the entry will save to `localStorage` without a `userId`, but `pollAll`'s filter (`a.game === 'roblox' && a.userId`) means it will never be checked and will never notify. The landing page and FAQ acknowledge this ("rolling out through 2026").

To add a real second game: write a `resolve<Game>User` and `check<Game>Presence` pair following the Roblox functions, extend the filter in `pollAll`, and extend `addChild`'s username-resolution branch (`app.html:481`).

## Pro / Stripe gotchas

- "Pro" gating is purely client-side: `isPro` (`app.html:253`) reads `gw_pro` from localStorage. The only enforced gate is the 2-child cap in `addChild` (`app.html:478`). Anyone can flip `localStorage.setItem('gw_pro','1')` in devtools.
- The Stripe checkout URL `https://buy.stripe.com/aFaeV63C22i18H04C80Fi00` is hardcoded in **both** files, each with a `<!-- REPLACE THIS HREF WITH YOUR STRIPE PAYMENT LINK -->` comment above it (`index.html:499`, `app.html:226`). When changing the payment link, update both copies.

## Routing assumption

`index.html` links to the dashboard with absolute `/app.html` paths (`index.html:372`, `:484`, `:541`). The site therefore assumes deployment at a domain root; serving from a sub-path will break those links.
