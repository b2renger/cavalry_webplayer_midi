# Cavalry MIDI Player

Play a Cavalry `.cv` scene in the browser and drive its **Control Centre**
attributes live — either by hand with on-screen controls, or from a MIDI
controller (e.g. Akai MIDIMIX) with MIDI Learn. Ranges, types and names are
read straight from the scene file, so every control maps to the right values.

---

## What's in this folder

```
cavalry-midi-player/
├── index.html      ← the player + MIDI-learn UI (ready to go)
├── serve.py        ← local server with correct WASM headers
├── wasm-lib/       ← Cavalry Web Player runtime (bundled, v2.6.0)
│   ├── CavalryWasm.js
│   ├── CavalryWasm.wasm
│   ├── CavalryWasm.data
│   └── version.json
└── README.md       ← this file
```

The Cavalry Web Player runtime is already included in `wasm-lib/`, so the
project runs as-is — just start the server (below) and load a scene.

## Updating the runtime  (optional)

The Web Player is Scene Group's beta software. To swap in a newer build:

1. Download the latest package from the Cavalry docs:
   https://cavalry.studio/docs/web-player/  →  "Download the Cavalry Web Player"

2. Replace the files in `wasm-lib/` with the new
   `CavalryWasm.js` / `CavalryWasm.wasm` / `CavalryWasm.data`.

   `index.html` loads the runtime from `wasm-lib/` via these constants near
   the top of its script:

   ```js
   const WASM_JS   = "/wasm-lib/CavalryWasm.js";   // the module entry
   const WASM_LIB  = "/wasm-lib";                  // folder with .wasm + helpers
   ```

   If a future package renames the module or moves the `.wasm`, update these
   two constants to match.

## Run it

```bash
cd cavalry-midi-player
python3 serve.py
```

This opens http://localhost:8000 in your browser.

- Use **Chrome or Edge** — they have the most reliable Web MIDI support.
- `localhost` is a secure context, so Web MIDI works without HTTPS.
- On first use the browser asks permission to access MIDI devices — allow it.

## Use it

1. Click **Load scene** and choose your exported Cavalry scene (`.cv`, or the
   same JSON saved as `.txt`/`.json`).
   - In Cavalry, add the attributes you want to control to the **Control
     Centre** before exporting, and save to scene version ≥ 1.0.55
     (Cavalry 2.5.0+). Only Control Centre attributes appear in the list.
2. Each Control Centre attribute gets a row on the right with a **live manual
   control** — a slider + number box, per-axis sliders for `position`/vectors,
   an on/off switch for booleans, and a clickable **palette strip** for
   colour-array selectors. Drag anything and the stage updates immediately.
3. To bind a hardware control, click the small **`learn`** chip next to a
   slider, then move a fader/knob (or press a pad). The chip shows the binding
   (e.g. `CC 16·ch1`). Faders drive the value continuously; a pad on a numeric
   row toggles between min/max, and on a switch row it toggles on/off.
4. **Remap a binding.** Once a control is learned, a `↳ remap` row appears under
   it: set the **min / max** the fader should span (defaults to the attribute's
   full range, but you can narrow it — e.g. bind a fader to `BEND` but limit it
   to ±45), flip direction with **⇅ invert**, and remove the mapping with **✕**.
   Click **apply** to push the new range to the renderer immediately, using the
   fader's last position (otherwise the change takes effect on the next move).
   Re-learning a parameter keeps its custom range; it just moves to the new
   control. Output is always clamped to the attribute's real limits.
5. Bind transport to pads via the **MIDI pads →** row under the stage
   (play / play-pause / stop / restart); each bound pad gets a **✕** to clear.
6. Hit **▶** (or your bound pad) and perform. Use the scrubber to seek.
7. **Export the current frame** with **⤓ SVG** / **⤓ PNG** (top bar). Files are
   saved at the scene's resolution as `<scene>_f<frame>.svg/.png`.

The ranges, value types, and names shown come from the scene file itself: the
app parses the `.cv` JSON and reads each attribute's `varType`, its
`definitionOverrides` (`hardMin`/`hardMax`/`softMin`/`softMax`), and its
`niceName`, so a fader maps to the *correct* real-world range (e.g. an
`arrayIndex` selector becomes a 0…7 integer stepper, not a 0…1 float).

## Notes / gotchas

- Manual controls work **without** MIDI — Web MIDI just adds hardware binding.
  (Web MIDI needs Chrome/Edge over `localhost` or HTTPS.)
- After any change the app calls `player.render(surface)` explicitly — the
  Web Player has no auto dirty-flag, so a value only shows once it re-renders.
- **Geometry attributes need a scene rebuild.** Verified with Playwright (see
  `test/`): in this runtime (v2.6.0 beta) an in-place `setAttribute` changes an
  attribute's stored value (`getAttribute` reflects it) but the runtime never
  re-cooks **geometry** nodes — deformers, distributions, the duplicator,
  look-at targets — on render, re-seek, or even playback. Only material/colour
  attributes redraw live. The only thing that re-cooks geometry is loading the
  scene, so on any change the app bakes the current Control Centre values into
  the scene JSON and re-runs `MakeWithPath` (~28 ms), **throttled** (~45 ms
  paused, ~220 ms while playing) so dragging a fader doesn't reload on every
  event. Colours/materials still update instantly via the live `setAttribute`;
  geometry catches up at the rebuild cadence. Geometry is smoothest to tweak
  while paused. If a future runtime re-cooks on `setAttribute`, the rebuild
  becomes redundant but harmless.
- Discovery is hybrid: the runtime's `getControlCentreAttributes()` is the
  source of truth for *which* attributes exist; the scene JSON supplies the
  rich metadata (type/range/name); the runtime fills any gaps. If the file
  isn't JSON, the app falls back to runtime-only discovery.
- The Web Player is **beta**; if something misbehaves, check the
  [API reference](https://cavalry.studio/docs/web-player/api/).
- **SVG export is a raster snapshot.** The Web Player is a raster renderer — it
  can import SVG but has no vector export. The **⤓ SVG** button captures the
  rendered frame at full resolution and wraps it (as a PNG) inside an `.svg`, so
  it's a valid `.svg` file but not editable vector. For true editable vector
  SVG, export from the Cavalry **desktop** app.
- Audio is not yet supported by the Web Player.

## Links

- Web Player getting started: https://cavalry.studio/docs/web-player/
- Web Player API reference:    https://cavalry.studio/docs/web-player/api/
- Demos:                       https://cavalry.studio/docs/web-player/demos/
