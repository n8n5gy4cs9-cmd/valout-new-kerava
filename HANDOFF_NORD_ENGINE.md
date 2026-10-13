# HANDOFF for the AI working in nord-engine — integrate "Vallout 2: New Kerava" as an example

**Context.** This repo (`valout-new-kerava`) is a finished, playable game made with nord-engine. The owner will copy its files into nord-engine as `examples/vallout/`. Your job: make it a first-class example that follows nord-engine's own rules (AGENTS.md / HANDOFF.md), then verify and commit to `main`. Read nord-engine `AGENTS.md` first.

## What the copied folder looks like (as-is)
```
examples/vallout/
  index.html (redirect to game/)   README.md  PLAN.md  PRD.md  tasks.md  HANDOFF_NORD_ENGINE.md
  engine/           vendored COPY of nord-engine runtime/core + runtime/phaser + phaser.min.js   → DELETE (engine already has them)
  game/             the Nord project: nord.project.json, index.html, scenes/main.scene.json, scripts/valout.js + scripts/v/*.js, assets/, assets.md
  tools/            pixelart.mjs (copy, writes to game/), art/valout/*.mjs, gen_audio.mjs, serve.mjs, shots.mjs, smoke.mjs
  docs/             RULES.md, screenshots/
```
## Target layout (engine convention: sources in tools/examples/<id>/, generated into examples/<id>/)
Use id **`vallout`**.
1. `tools/examples/vallout/game.mjs` — modelled on `tools/examples/forgotten/game.mjs`: `project('vallout','Vallout 2: New Kerava','2d',{ w:640, h:480, pixelArt:true, bg:'#050403', physics:{engine:'none',gravity:[0,0,0]}, input })`, one entity `E('Valout2',{ s:['scripts/valout.js'] })`, `assets` list (all `game/assets/**`), `files: { 'assets.md', 'GUIDE.md' }`, `readme`.
2. Move `game/scripts/` → `tools/examples/vallout/scripts/`, `game/assets/` → `tools/examples/vallout/assets/`, `game/assets.md` + README → `tools/examples/vallout/docs/`.
3. Move `tools/art/valout/` → engine `tools/art/vallout/` (or keep id `valout` in the art folder name); the engine's `tools/pixelart.mjs` already writes to `tools/examples/<id>/` — **delete the copied `pixelart.mjs`** and make the art module export `id = 'vallout'`. Move `tools/gen_audio.mjs` → engine `tools/gen_vallout_audio.mjs` and change `OUT` to `tools/examples/vallout/assets/audio` (follow `gen_xkom_audio.mjs`).
4. Delete `engine/`, `game/index.html`, root `index.html`, `tools/serve.mjs` (engine has nord-serve). Engine-generated `index.html` uses `/runtime/core/index.js` (importmap `nord`), `/vendor/phaser/phaser.min.js`, `/runtime/phaser/index.js` — the game scripts only `import { NordScript } from 'nord'` and relative `./v/*.js`, so no code change is needed.
5. Run `node tools/gen_examples.mjs vallout` → produces `examples/vallout/`. Never hand-edit it.
6. Update `tools/shots.mjs` / `tools/smoke.mjs` → rename `tools/vallout-smoke.mjs` / `vallout-shots.mjs`: replace the local `./serve.mjs` import with the engine's serve approach (or `nord-serve`), and the URL `/game/` → `/examples/vallout/`. They need `window.__valout?.ready` (set by the script) and use internal hooks `window.__valout.ctx/screens`. Playwright path used: `/opt/node-tools/node_modules/playwright` (adjust to engine's e2e lib `tools/e2e-lib.mjs` if preferred).

## Bookkeeping the engine requires
- `node tools/tasks.mjs add M6 "Example game: Vallout 2 New Kerava"` then `done <ID>`; add CHANGELOG line; add to HANDOFF.md "Current status".
- `THIRD_PARTY.md`: (a) game rules ported from **fallout2-ce** (`src/combat.cc attackDetermineToHit`, `stat.cc`, `skill.cc`; license: Sustainable Use / see repo) — formulas only, no assets; (b) bitmap font `font.png` + `pixeltext.js` are copied from the engine's `forgotten` example. All art/audio is original. Mention sfall/falltergeist only as reference.
- `ASSETS.md` entry pointing at `examples/vallout/assets.md`; add to the library index: `node tools/build-library-index.mjs`; if `docs/manual` lists examples run `node tools/build-manual.mjs`.
- Optional: add a `template` block in `game.mjs` if a "isometric RPG" starter is wanted.

## How the game works (so you can fix things)
- `scripts/valout.js`: entry NordScript; builds shared `ctx` (services, state `G`), preloads sheets, input, update loop (`ps.events.on('update')`).
- `scripts/v/`: `data.js` (items/enemies/locations/quests/dialogue), `rules.js` (pure F2 rules), `path.js`, `mapgen.js` (pure, seeded), `play.js` (Area: iso render/entities/loot/death), `classic.js` (click + turn-based combat; mixin on Area.prototype), `modern.js` (WASD + mouse real-time; mixin), `hud.js`, `ui.js` (Layer toolkit), `screens.js` (title, create, world map, overlays, dialogue, wall minigame, endings).
- Sprite sheets are 'spritesheet' type with `*.meta.json` frame-name maps (`frames` name→index; actors have `rows`). Texture keys are the asset paths. Depth bands: world < 9500, HUD 12000, overlays 15000, cursor 20000.
- Pure modules (`rules/path/mapgen/data`) run in node (see how they were tested: generate every location, check NPC/exit reachability, to-hit numbers).

## Verify before ticking
1. `node tools/pixelart.mjs vallout` and audio generator re-run cleanly; `node tools/gen_examples.mjs vallout`.
2. Serve the engine, open `/examples/vallout/`; title → NEW GAME → intro → create → play. F10 toggles Classic/Modern.
3. Run the smoke test: expected `ALL PASS`, no console errors (WebGL "GPU stall" warnings in swiftshader are harmless).
4. `node tools/smoke-examples.mjs vallout` (engine's generic smoke) and the editor's project hub opens it (the main scene has one entity with the script; the project is script-driven, the scene is intentionally minimal).
## Known limitations / ideas
- Only headless-verified; never played by a human on real GPU; audio not listened to. Balance untuned (enemy damage is fairly high for a 33-HP start).
- Font is upper-case ASCII only (ä/ö transliterated). Dialogue text is English with Finnish flavour.
- No perks/traits/barter/hex grid. World map is drawn in code (no image asset).
- Modern mode aim assumes cursor points at chest height (+14 px offset in `aimVec`).
- Save is one slot via engine `save` (`valout2` key).
