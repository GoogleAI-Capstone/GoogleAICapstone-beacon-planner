# Beacon Planner

A browser-based tool that recommends where to install BLE beacons on a building floor plan, then lets a person adjust the result by hand.

Part of the **Google–Ajou AI Convergence Capstone Design** project (team Brain, Ajou University), where I lead the team and oversee the whole project. The project builds indoor positioning with BLE beacons for **Lotte Mall Gwanggyo**, a partner site of the capstone.

There is no backend. The calculation engine runs in the browser (in a Web Worker), so the app deploys as static files to Vercel, Netlify or GitHub Pages with no server cost.

## Why I built it

Beacon-based positioning only works if the beacons are placed well, and the mall floor plans are large: B1 to 4F, hundreds of walls per floor. Placing beacons by hand is slow, and it is hard to tell whether a layout is good until you have installed it. Beacons also cost money, so "just add more" is not an answer.

I wanted to answer three questions before anyone climbs a ladder with a beacon:

1. Where should the beacons go?
2. How many do we need, and what will it cost?
3. Can a person still override the algorithm where it is wrong (a tenant unit, an escalator landing that the floor plan mislabels)?

## What it does

- Loads the Lotte Mall Gwanggyo floor plan (B1–4F) that is bundled with the app.
- Computes a beacon layout per floor and reports coverage, HDOP and cost.
- Lets you drag, add, delete and **lock** beacons, then re-evaluate the layout.
- Exports beacons, RF settings, options, metrics and cost as one JSON file.

### Two placement modes

The mall is treated differently above and below ground, because that is how the positioning problem differs.

| Mode | Used for | Goal |
|---|---|---|
| **Positioning** | Underground floors (no GPS) | At least 3 beacons visible at every point, with good geometry (low HDOP), so trilateration works |
| **Checkpoint** | Above-ground floors (GPS gives x/y) | Beacons only answer "which floor am I on?", so they go at elevator, escalator and stair landings |

The positioning objective is not plain signal coverage. It is the area where **3 or more beacons are visible and HDOP is acceptable**, because three nearly collinear beacons give a diverging position fix. Placement uses a greedy algorithm, the standard approach for max-coverage problems, with a (1 − 1/e) ≈ 63% approximation guarantee.

Using checkpoint mode above ground cuts the total from 488 beacons to 98.

## How it was built

- **React 18 + Vite**, no backend.
- `src/engine/` holds the calculation: `geometry.js` (floor plan, walls, visibility), `rf.js` (path-loss model and RF settings), `planner.js` (greedy placement for both modes), `planner.worker.js` and `client.js` (run it off the main thread).
- `src/FloorCanvas.jsx` draws the floor plan and beacons and handles drag / add / delete.
- The engine is a JavaScript port of the team's Unity C# prototype. `tools/verify.js` runs both against the same floor plans: wall counts and evaluation points match exactly, and beacon counts agree within ±3 (greedy selection breaks ties differently in floating point).

## How to use it

Requires Node.js 18 or later.

```bash
npm install
npm run dev        # http://localhost:5173
npm run verify     # compare the engine with the Unity C# reference values
npm run build      # static build in dist/
```

In the app:

1. Set the beacon specs (model, unit price, Tx power, 1 m reference RSSI, path-loss exponent, threshold).
2. Pick a floor. The floor plan is already loaded.
3. Press **Calculate placement** (about 3 seconds; the UI stays responsive).
4. Adjust: drag to move, Shift+click to add, Delete to remove, or lock a beacon. Locked beacons are kept on recalculation. Beacons you move or add are locked automatically.
5. Press **Re-evaluate** after manual edits to refresh coverage and cost, then **Save JSON**.

Deploying is the standard Vite flow (build command `npm run build`, output directory `dist`). For GitHub Pages, set `base` in `vite.config.js`.

## Results so far

Recommended configuration: positioning mode on B1, checkpoint mode on 1F–4F, at 15,000 KRW per beacon.

| Floor | Mode | Beacons | Metric |
|---|---|---|---|
| B1 | Positioning | 66 | 82.3% triple coverage, mean HDOP 0.90 |
| 1F | Checkpoint | 10 | 20 vertical-transport points |
| 2F | Checkpoint | 8 | 16 vertical-transport points |
| 3F | Checkpoint | 8 | 15 vertical-transport points |
| 4F | Checkpoint | 6 | 9 vertical-transport points |
| **Total** | | **98** | **1,470,000 KRW** (vs 7,320,000 KRW for 488 beacons) |

B1 triple coverage is 82.3% because tenant interiors are excluded from placement; turning that option on brings it to about 95%.

## Known limits

- **RF parameters change the result completely.** Moving the threshold from −80 dBm to −85 dBm took triple coverage from 6–34% to about 95%. The values are literature values for indoor 2.4 GHz, not measurements from the mall, so treat the numbers as relative comparisons until calibrated. The calibration procedure is: mount one beacon at 3 m, record RSSI at 1, 2, 3, 5, 8, 12 and 20 m for 30 s each, and fit `RSSI = A − 10·n·log10(d)`.
- **Floor detection can fail on 1F.** Only 97.7% of points are classified correctly (48 wrong points), because dense B1 beacons can be heard through the slab from a spot on 1F far from any checkpoint. The app should keep the current floor and change it only when a checkpoint beacon is passed.
