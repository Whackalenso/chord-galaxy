# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the Project

No build step. Open `index.html` directly in a browser, or serve it locally:

```bash
python3 -m http.server 8080
# then open http://localhost:8080
```

Click the screen to start audio (required by browser autoplay policy), then navigate with WASD/arrow keys or click tiles.

## Architecture

Single-file app (`index.html`, ~750 lines). All CSS, HTML, and JS are inline — no external dependencies, no bundler.

### Core Systems

**Hex Grid** — Axial coordinates `(q, r)`. `hexToPixel` / `pixelToHex` for conversion. `HEX_SIZE=200` (center-to-vertex). Six directions in `HEX_DIRS`.

**Deterministic Generation** — `coordHash(q, r)` produces a stable 0–1 value for any coordinate, enabling infinite procedural generation that's always identical for the same coords. `coordRand(q, r, idx)` provides a sequence of values per tile.

**Tile Map** — `hexMap` (Map) stores generated tiles lazily. Each tile: `{q, r, notes[5], px, py, noteLabel, seedType, isSeed, color, landmarkKey}`. `ensureTile(q, r)` generates on demand; `ensureArea(q, r)` pre-generates 2-ring neighborhood.

**Landmark System** — `getLandmarkType(q, r)` classifies tiles as tonic (~4%), parMinor (~3%), secDom (~2.5%), neighbor (~2%), or null. Landmarks anchor fixed chord voicings; non-landmark tiles trace a greedy mutation path from their nearest landmark via BFS (`findNearestLandmark`).

**Chord Mutation** — `pickMutation` moves one voice ±1 or ±2 semitones. Scored by hex direction + deterministic randomness. Voice range: MIDI 36–84 across 5 voices.

**Harmonic Analysis** — `detectPattern(cur, nb)` classifies transitions as `resolve`, `tension`, `brighten`, or `darken`. Priority order: suspension resolution → tritone resolution → tritone creation → half-step resolution → consonance delta → lateral motion.

**Audio** — Web Audio API. 5 voices (sine + detuned triangle per voice). Convolver reverb (55% dry / 45% wet). Glide via `linearRampToValueAtTime` over 0.4s.

**Rendering** — Canvas 2D, camera-follows-car. Border weight scales with note-diff count. Neighbor tiles show colored dots + voice-motion labels. Minimap in top-right.

### Landmark Types & Colors
| Type | Ring color | Label | Source chords |
|------|-----------|-------|---------------|
| tonic | gold `#ddc878` | I | C major voicings |
| parMinor | blue `#8fa4c8` | pm | Cm, Fm, Ab, Eb, Bb |
| secDom | orange `#d4845a` | V/ | D7, A7, E7, B7, G7 |
| neighbor | green `#8cc87a` | N | G, F, Am |

## Known Limitations

- Chord naming was removed (auto-identifier produced wrong names); HUD shows raw note names instead.
- Territory seam transitions between two landmarks' regions can be abrupt (hard white borders mark these).
- Tile color (consonance-based) is acknowledged as not very perceptually meaningful.
