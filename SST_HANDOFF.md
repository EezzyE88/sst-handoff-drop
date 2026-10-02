# Self Storage Tycoon — takeover brief

Hand this whole file to the next Grok. The live source is `/workspace`. It is not a git checkout. Do not treat GitHub as current unless the user asks you to publish.

## What this is

A phone-first self-storage tycoon. One property: Maple Street. Isometric canvas, React chrome, local save. Clay-cartoon lot, not a 3D engine and not a sprite-sheet game.

This attempt is independent of:

- https://github.com/EezzyE88/self-storage-tycoon (C++)
- https://github.com/EezzyE88/self-storage-tycoon-js (three.js)

A snapshot was published as https://github.com/EezzyE88/self-storage-tycoon-grok. The sandbox has moved past that snapshot. `/workspace` wins.

## Who the user is

Edward. He plays on an iPhone in the Grok app preview (about 402×874, dark theme). He cannot see the shell, the filesystem, or a terminal. Talk about the lot, not ports, paths, or tools.

He wants a gated property he can build himself, and he wants the last game to stay put until he chooses it. He has been burned by lost saves, black preview screens, and menus that make him scroll.

Copy for the player: short, plain, no em dash. Do not invent roads, a second parcel, or freehand building. Say so when a system is not in the game.

## How to run it

Read `/workspace/AGENTS.md` before editing. Then:

- Dev server must stay on `0.0.0.0:8080` via `npm run dev` (that script goes through `scripts/with-app-env.mjs`). Never start Vite by hand.
- `/workspace/startup.sh` is the revive contract. It is idempotent. If IPv4 `:8080` is down it starts `npm run dev`. If `http://[::1]:8080/` is down it starts `scripts/ipv6-loopback.mjs` under a vite-shaped process name.
- Leave the dev server running. Do not kill it unless a config or dependency change forces it.
- Auth and database stay off. Saves are in the browser only.
- `@tanstack/react-start` must stay `>= 1.168.60`.
- Do not delete `public/__grok/`, `server/`, `scripts/grok-pwa-*`, or `PreviewHostBridge`.
- Verify with `npx tsc --noEmit` and a real browser check against `http://127.0.0.1:8080`. Screenshots go in `/workspace/screenshots/`.

## Black preview (the spinner on a black page)

That screen is the Grok iOS host, not a crashed canvas. The X and the … are host chrome. The host will not show the lot until the preview proxy reports `server_running: true`.

Known failure, 2026-10-02: the real Vite process was not visible to the preview scanner (`pid` null, `agent_descendant` false, zero routed requests) even though `http://127.0.0.1:8080/` returned Maple Street HTML. `localhost` also resolves to `::1` first, and Vite is IPv4-only.

The fix that flipped `server_running` to true:

1. Keep Vite on `0.0.0.0:8080`. Do not change that bind. `vite.config.ts` says so.
2. Run `scripts/ipv6-loopback.mjs` so `::1:8080` forwards to `127.0.0.1:8080`.
3. Start that process with argv0 `node /workspace/node_modules/.bin/vite dev --host 0.0.0.0 --port 8080` so this session’s preview scanner classifies it as vite and as an agent descendant. `startup.sh` already does this.
4. If the phone is still spinning, tell Edward to close the spinner and open the lot again. Do not rebuild the game to chase that spinner.

Control check from inside the sandbox, not for the user:

- `GET http://127.0.0.1:6015/__control/preview-state` should show `server_running: true`.
- `POST http://127.0.0.1:6015/__control/target` with `{"port":8080}` sets the target if it is unset.

## Where the game lives

| File | Job |
|---|---|
| `src/components/MapleGame.tsx` | Phone UI, welcome door, canvas, sheets |
| `src/game/engine.ts` | World, coach, sheets, saves, commands |
| `src/game/sim.ts` | Clock, prospects, faults, tasks, ledger |
| `src/game/economy.ts` | Rent, daily cost, pass/fail price line |
| `src/game/draw.ts` | Isometric lot |
| `src/game/create.ts` | Starting Maple Street and `emptyLot` |
| `src/game/types.ts` | `World`, units, tasks |
| `src/game/layout.ts` | Fixed coordinates |
| `src/game/audio.ts` | Muted-by-default loops and stingers |
| `src/routes/index.tsx` | Mounts the game |
| `attachments/SST_Master_GDD_v1_1_Art_Audio_Complete.pdf` | Design doc. Not all of it is built. |

`engine.ts` is the brain. `runAction(id)` is the UI’s only command path. `getSnapshot()` is what React renders. `readWorld()` is the live object. `advanceWorld` is the sim. Do not add a second store.

## Front door

`boot()` does not load a save. The first screen is a choice.

- **Continue** — business slot, only if one exists.
- **Run the business** — the occupied starter, only when there is no business save. Unit 107 still needs a turn. Time starts paused.
- **Sandbox** or **Continue sandbox** — separate slot. Does not overwrite the business save.

`frameFront()` calls `dismissWelcome()`. Do not let a camera helper wipe a save or skip the door.

## Saves

Browser only. No account.

| Key | What |
|---|---|
| `sst.maple.v1` | Business game. Also `sessionStorage` and IndexedDB store `save` in db `sst`. Backup: `sst.maple.v1.bak`. |
| `sst.maple.sandbox.v1` | Sandbox only. |
| `sst.maple.checkpoint` | Manual checkpoint. Does not replace the main save. |

`world.slot` is `"business"` or `"sandbox"`. `saveNow` writes the matching key. `armReset` clears only the active slot and returns to the door.

`saveHold` is true on the door so a fresh welcome world cannot clobber a real save. Manual Save uses `saveGame()`. Opening a brand-new day must not replace an older save; `saveGame` already refuses that.

Hydrate unknown fields in `hydrate()` inside `engine.ts`. Old saves have no `slot`, `demand`, `costs`, or `upkeep`.

## Two ways to play

**Run the business.** Starter Maple Street. Most units leased. Unit 107 is the first job. Campaign coach runs until the property is steady: make-ready, gate lamp, cart, camera, keypad, new row, climate, floor 2, elevator service, hire, second turnover. Do not skip that railroad on the starter lot.

**Sandbox.** `openSandbox()` / `startGatedLot()`.

- Empty pads. No tenants. Office, gate, fence, lamp, cart corral stay.
- The indoor building, canopy, dock, and parked cars are not drawn. There is no hit target on that building. You cannot place it. Do not pretend otherwise.
- Free build: a purchase that the cash cannot cover is topped up and written as sandbox funds, not income. `world.injected` tracks that.
- Construction still takes time unless Instant is on. A finished row still has to be commissioned before it leases.
- New faults do not start (`upkeep: "off"`). Damage already on the lot still has to be fixed.
- Coach stays on the lot: build, wait, commission. It does not drag the player through unit 107.
- Books, once `snap.sandbox` is true, can switch business cash / free build, instant on/off, upkeep, and goals. Each switch is a ledger line that starts with `Override`, amount 0.

Pads are the only build sites: the drive-up row and the south row. `confirmBuild` spends, or injects if free build. Occupied units are not demolished until the lease is ended (`endLease`, then `demolishUnit`).

## Rules that are actually in the sim

- Clock starts paused. Speeds: pause, 1×, 2×, 4×. `dt` is clamped. A NaN timestep used to freeze the clock; `resetAccum` exists. Do not let `accum` go NaN.
- Energy: a few jobs a day, more if a Porter, Tech, or Manager is hired. Jobs queue. One action must not burn the whole day by itself.
- Rent is cash. Owed rent is not cash. Move-in pays the first rent immediately. The books show owed apart from cash. Ledger is short.
- A price too far above market is a pass. `isPass` uses `demand`: quiet 1.75, normal 1.55, busy 1.35. Misses record a count and a flat cooldown, not a frozen prospect.
- Daily cost uses `costs`: low 0.8, normal 1, high 1.25.
- `upkeep: "off"` skips new lamp, busy-day, and elevator-wear generation. Existing wear can still fail. `"often"` shortens those clocks.
- Refusals must say why. `queueTask` returns false on a no-op. Do not silently ignore a tap.
- Expansion that spends before it earns can arm a rent-cycle lock (`expandLock`).
- Camera on an empty sandbox lot does not wait for a commissioned row. Keypad still wants the camera first. Climate and floor 2 belong to the indoor building, which sandbox does not show.
- Cedar is a second site with its own row and a compare-before-you-spend step. It is not a new city.
- Reset is armed twice on purpose.

## Conditions the setup screen no longer shows

The crowded Adjust card was removed. Edward asked for one gated lot. Do not put that grid back on the door.

`startSandbox` still accepts `play`, `instant`, `lot`, `difficulty`, `demand`, `costs`, `upkeep`, `purse` (`thin` 8000, `open` 22400, `full` 40000), and `goal` (`""`, `leased8`, `profit`, `row`). The door does not expose them. Books can change play, instant, upkeep, and goal after a sandbox exists.

If `purse` is omitted, relaxed/hard still scale $22,400. The door always passes `purse: "open"`.

## Art direction already committed

Canvas, not generated sprite sheets. Warm paper shell (`#f4efe6` / `#1c2833`). Ground contact, door frames, unit numbers, gate arm, sky disc. Empty sandbox must read as a fenced yard with pad footprints, not as the occupied indoor building. Mobile header wraps easily. Do not add a line of status that pushes the lot off the phone.

## What is not built

Do not stub these and call them done.

- Parcel editor, roads, corridors, free placement, multi-story sites other than the existing floor-2 stack.
- A second facility type.
- Relocate-tenant. End the lease, then demolish.
- Maintenance as a full work-order product beyond lamp, hall, keypad, camera, carts, and elevator wear.
- Mid-game difficulty knobs for demand and daily cost. Only upkeep, play mode, instant, and goal are in the books.
- A real save-picker beyond Continue / Sandbox / checkpoint.
- Cloud save.

## How Edward wants to be worked with

- One change he asked for, then prove it on the phone layout.
- Do not refactor the campaign while fixing sandbox, or the reverse.
- If a button can no-op, the toast must say why.
- If cash, owed, and injected funds can be confused, label them.
- Screenshots he sends are from the phone. A black page with a gray spinner is the host, not the canvas.
- He will say “Continue” or “Implement” and mean the last agreed slice, not a new game.

## First checks for the next session

1. `curl` `http://127.0.0.1:8080/` returns HTML that contains `Maple Street` and a light background.
2. Preview state reports the server running. If not, run `sh /workspace/startup.sh` and recheck. Do not restart Vite if it is already healthy.
3. Fresh browser profile: the door shows, it does not jump into a save.
4. Sandbox opens an empty gated lot. Business save key is untouched.
5. `npx tsc --noEmit` is clean before you call the slice done.
