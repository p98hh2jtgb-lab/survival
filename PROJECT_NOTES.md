# CreatorPack (HUD PRO) – BeamNG mod · Compact project summary

Latest build: `CreatorPack_v5.9.3.zip` (repo root, branch `arena/01a0e290-survival`)
Link: https://github.com/p98hh2jtgb-lab/survival/raw/arena/01a0e290-survival/CreatorPack_v5.9.3.zip
Work flow: unzip latest zip -> edit -> re-zip as new version -> delete old zip -> commit/push. Reply to user in Albanian (Kosovo).

## Structure
- `lua/ge/extensions/creatorpack.lua` – damage/cost engine, guihooks events (`pfhud.action`, `creatorpack.cost`, `creatorpack.autophoto`), `pfhudAutoPhoto`.
- `lua/ge/extensions/core/input/actions/pfhud_actions.json` – bindings (PASS K, FAIL L, next/prev, toggles, reset survival, auto photo).
- `ui/modules/apps/CreatorPack/app.js` (MOD_VERSION), `app.html` (CSS+panel), fonts (Bangers, BebasNeue-Regular.ttf), `emoji/` (Fluent 3D PNG), `sounds/` (hit, bighit, critical, dead mp3).

## Features done
- Survival Chance HUD + reaction phrase under it: 23 styles × 6 levels (safe 76-100, caution 51-75, danger 26-50, severe 6-25, extreme 1-5, dead 0), ~2100 phrases, shuffle-bag (no repeats), rotation default 3000ms (slider 1-8s). Default pack streamer.
- Font: Bebas Neue CLEAN default (v5.7), 8-shadow black outline, no glow. "✨ CLEAN" preset button. % on baseline (v5.9.1). Label↔% gap slider `survivalLabelGap` (default 0.12em).
- Meme SFX (4 slots, XHR+WebAudio, 1.2s cooldown).
- Circles: `circlesClean` default true (no names, no damage %, no red ring).
- Auto photo (v5.9.x): button/Ctrl+1/binding; gets car's official preview image from game, flood-fill background removal (tolerance slider 8-70, auto-retries lower), crops, sets imageScale 0.62. v5.9.3 uses direct `bngApi.engineLua(expr, callback)`; 6s timeout message.

## User rules
- No red flash/TOTALED stamp, soft glow only, no -X% popup, no vignette.
- Don't replace/overlay the phrase; no dialogue/talking-car ideas.
- Banned phrases: WRECKED, SEND IT, PRAY, DOOMED, END OF THE ROAD, THAT'S IT, RUN OVER, BACK TO THE GARAGE.
- Deliver as GitHub raw link.

## Open issue
Auto photo never returned in v5.9.2 (stuck "duke marrë foton…"). Suspect old CreatorPack zips in mods folder. Waiting for user test of v5.9.3 (message ✓ / ✗ / "Loja s'u përgjigj").
