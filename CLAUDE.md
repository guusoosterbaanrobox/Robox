# Robox (Robot Wars)

A dice-battle robot-building game. Single-file React + Three.js app, shipped as one standalone `index.html`.

## Files
- `robot-wars.jsx` is the SOURCE OF TRUTH. One ~5,300-line file (~2.7 MB; much of that is embedded base64 images). Do not read it whole: use Grep, then Read with offset/limit.
- `index.html` is a BUILD OUTPUT: a small HTML shell (head, manifest, `<div id="root"></div>`, then one `<script>`) with the minified bundle inlined.

## Build (how index.html is produced)
Dependencies: `esbuild`, `react@19`, `react-dom@19`, `three@0.128`.
1. `entry.jsx`:
   ```jsx
   import React from "react";
   import { createRoot } from "react-dom/client";
   import RobotWars from "./robot-wars.jsx";
   createRoot(document.getElementById("root")).render(<RobotWars />);
   ```
2. `esbuild entry.jsx --bundle --minify --format=iife --loader:.jsx=jsx --jsx=automatic --outfile=bundle.js`
3. Take the shell = everything in the existing `index.html` up to and including `<div id="root"></div>\n<script>\n`, append `bundle.js`, then `\n</script>\n</body>\n</html>\n`.
TODO: turn this into a proper build script, e.g. `npm run build`, and keep the shell as its own file.

## Game rules (as implemented)
- Phase 1, Workshop: players draft parts (head, body, feet; 6 colors) from a blind bag into 3 bays.
- Phase 2, Refinement: each turn the player draws from the bag or takes from the table, then swaps the part into a bay slot (the displaced part goes to the table). A player can declare War, which triggers a 5-part War Surge draft. A Final Chance turn follows a declaration. A "golden" robot (see `findGoldenBay`) wins instantly.
- Phase 3, War: the bays are matched up and compared with animated dice rolls; ties are resolved with `resolveTie`.

## Game modes (`mode` state in the main `RobotWars` component)
- `"ai"` is Single Player: classic bag/table draw against the AI. Must stay unchanged.
- `"local"` is Multiplayer pass-and-play: two humans on one device. It is the only mode where `displayWs` follows the active player and the bay fade (`fadingWs`, `triggerBayFade`) applies.
- `"advanced"` is Single Player Advanced: same as `"ai"`, but Phase 2 turns open with a d6 roll for BOTH the player and the AI:
  - 1 or 4 is Peek & Pick: reveal the top 3 bag parts, choose one, place it as a normal draw.
  - 2 or 5 is Mirror Swap: the active side offers one of its bay parts; the other side (the AI picks its most expendable part) offers one back; the two are traded.
  - 3 or 6 is Call Shot: name a color and type; if the opponent has it anywhere in a bay it must be handed over in exchange for one of the caller's parts, otherwise the turn is wasted.
  - Faces 4-6 currently repeat 1-3. The user may later define distinct effects.

## Advanced-mode code map (grep these)
State: `d6Roll`, `advancedAbility`, `advancedStep`, `peekOptions`, `pendingExchange`.
Constants: `ADVANCED_ABILITY_LABEL`, `advancedAbilityForRoll`.
Functions: `startAdvancedMatch`, `rollAdvancedDie` (player), `runAiAdvancedTurn` (AI, called from the AI-turn effect), `startPeek`, `applyPeekedPart`, `choosePeekOption`, `startMirrorSwap`, `resolveMirrorSwapOffer`, `startCallShot`, `resolveCallShotQuery`, `chooseCallShotQuery`, `completePartExchange`, `handleAdvancedBayClick`, `finishAdvancedTurn`, `resetAdvancedState`, `aiPickExpendablePart`.
UI: menu entry in `MenuPanel` (`onSinglePlayerAdvanced`); "ROLL D6" button replaces the bag button in Phase 2; overlays for the `peekChoose`, `swapChooseOffer`, `swapChooseReceive`, `callChooseType` and `callChooseGive` steps.

## Gotchas
- `displayWs = mode === "local" ? activeWs : playerWs`. In single-player modes the player always sees their own bay, never the AI's.
- Status text "Waiting for the AI…" must cover both `"ai"` and `"advanced"`. Any `mode === "ai"` check needs a matching `"advanced"` check.
- The AI-turn effect runs for `"ai"` and `"advanced"` only.
- Any new per-turn state must also be reset in `resetForNewMatch` and `resetGame`.
- Not yet verified by hand: Golden Robot win mid-exchange, and Final Chance turns in advanced mode.
- Headless Playwright testing works for smoke tests. The bay slot buttons use the class `ring-1 ring-white/70`; the bag button uses `pick-bag`.
