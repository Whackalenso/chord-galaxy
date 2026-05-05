# Chord Drive — Project Context & Design History

## What This Is

An interactive web experience where you explore an infinite procedural hex grid of musical chords. Each hex tile is a chord. Moving between tiles plays chords with smooth voice leading. The goal is to create an experience where you discover beautiful, expressive chord progressions by navigating a space — not by programming them.

**Current state:** Single HTML file (`chord-drive.html`) with Canvas rendering, Web Audio API synth, procedural generation, click-to-navigate + WASD driving.

---

## Inspirations

- **VCV Rack Galaxy module** (AmalgamatedHarmonics) — treats harmonic content as a navigable space
- **Telepathic Instruments Orchid** — instrument with built-in chords you can play easily
- **Teenage Engineering PO-20** — fun built-in chord playback
- **User's vibe-coded prototype** with 6 sliders (dissonance, tension, space, etc.) that generated chords by parameter

---

## Design Evolution (key decisions made in this chat)

### 1. Started as a driving game with biomes
Originally: car drives through 2D terrain with "mood" biomes (sad, beautiful, scary). Chords voice-lead as you cross regions.

### 2. Mood biomes → Functional categories
**Problem:** Labeling chords by mood (serene, melancholy) was inaccurate — a minor chord can feel many ways depending on context. Chords don't have fixed emotions, they have fixed *relationships*.

**Solution:** Switched to functional categories: Open (sus/quartal), Rooted (triads), Color (extensions), Tension (diminished/altered). These describe *what the chord is*, not how it should feel. Emotion emerges from the player's journey between categories.

### 3. Continuous biomes → Hex tiles
**Problem:** Continuous gradient between biomes was ambiguous — chords are discrete, not continuous. The center of pie-slice biomes was unclear.

**Solution:** Each chord is its own hexagonal tile. Biomes became organic regions of hex tiles. Sharp boundaries between biome types (especially tension→resolution) were intentionally designed.

### 4. Two-axis system (bright↔dark, simple↔complex)
Informed by the user's chord theory document. Instead of discrete biomes, organized chords along two perceptual axes:
- **Horizontal:** Bright (lydian) ↔ Dark (locrian)  
- **Vertical:** Simple (triads) ↔ Complex (dense extensions)

### 5. Single key center
All chords constrained to key of C (including parallel minor, harmonic minor, melodic minor borrowings). This ensures every transition sounds intentional — no accidental key clashes.

### 6. Mutation-based generation (current system)
**The breakthrough:** Instead of placing pre-defined chords, start from a seed chord (C major) and grow outward. Each neighboring tile differs by exactly ONE note (half step or whole step mutation). Voice leading is structural, not simulated.

### 7. Multiple key centers via landmarks
Expanded beyond single-key by adding seed types:
- **Tonic (I)** — C major in various voicings
- **Parallel minor (pm)** — Cm, Fm, Ab, Eb, Bb
- **Secondary dominants (V/)** — D7, A7, E7, B7, G7
- **Neighbor keys (N)** — G major, F major, Am
- **Tritone substitutions (bII)** — Db7, computed adjacent to specific tonics

### 8. Infinite procedural generation (current)
Map generates lazily as you explore. No pre-defined grid. Landmarks placed via deterministic coordinate hashing. Each non-landmark tile traces ancestry to nearest landmark via mutation chain.

---

## Current Architecture

### Hex Grid
- `HEX_SIZE = 200` (center to vertex)
- Axial coordinates (q, r)
- `hexToPixel()` / `pixelToHex()` for conversion
- `hexCorners()` for rendering
- 6 directions: `[[1,0],[1,-1],[0,-1],[-1,0],[-1,1],[0,1]]`

### Tile Generation (`ensureTile(q, r)`)
1. **Check if landmark** — `getLandmarkType(q,r)` uses `coordHash()` (deterministic hash of coordinates) to decide if a tile is a tonic (~4%), parallel minor (~3%), secondary dominant (~2.5%), or neighbor key (~2%)
2. **Check if tritone sub** — if adjacent to a tonic that's flagged for a tri sub, and this tile is the chosen neighbor direction, generate a Db7 voicing optimized for voice leading to that tonic
3. **Otherwise: trace to nearest landmark** — BFS outward to find nearest landmark, trace a greedy path from landmark to this tile, generate each step by single-note mutation
4. Each tile stores: `{q, r, notes[5], px, py, noteLabel, seedType, isSeed, color, landmarkKey}`

### Chord Mutation (`pickMutation`)
- For each of 5 voices, try moving ±1 or ±2 semitones
- Filter out moves that go out of range (MIDI 36-84) or create duplicate pitches
- Score mutations based on hex direction (right→prefer upward, down→prefer half-steps) + deterministic randomness from target coordinates
- Pick from top 3 candidates

### Landmark Ownership
- Each tile stores `landmarkKey` — the coordinate key of the landmark it traces ancestry to
- Landmarks own themselves
- Used for territory visualization

### Harmonic Analysis (`detectPattern`)
Priority-ordered pattern detection between two chords:
1. **Suspension resolution** — 4th→3rd or 2nd→3rd against a held note
2. **Tritone resolution** — tritone collapsing to consonant interval
3. **Creating tritone** — flagged as tension
4. **Half-step resolution** — half-step interval resolving to wider interval
5. **Creating half-step dissonance** — flagged as tension
6. **General consonance comparison** — using weighted interval scores (P5=9, M3/m3=6-7, P4=8, m2=1, tritone=1.5)
7. **Lateral motion** — brighten (voice up) or darken (voice down)

Returns: `{type, voiceIdx, delta, strength, label}`
Types: `'resolve'`, `'tension'`, `'brighten'`, `'darken'`

### Consonance Scoring
```
INTERVAL_CONSONANCE = {
  0:10 (unison), 1:1 (m2), 2:3 (M2), 3:6 (m3), 4:7 (M3), 
  5:8 (P4), 6:1.5 (tritone), 7:9 (P5), 8:6 (m6), 9:7 (M6), 
  10:3 (m7), 11:2 (M7)
}
```
Average across all note pairs in the chord.

### Tile Colors
Derived from consonance score. More consonant = warmer gold, more dissonant = cooler blue. This is acknowledged as imperfect — the user noted colors often feel inaccurate.

### Border Rendering
- **Hard white borders** where adjacent tiles differ by 2+ notes (jarring jumps)
- **Faint borders** where tiles differ by 1 note (smooth voice leading)
- Thickness scales with note difference count

### Audio
- Web Audio API
- 5 voices, each = sine oscillator + detuned triangle oscillator
- Convolver reverb (45% wet / 55% dry)
- `linearRampToValueAtTime` for glide (0.4s)
- Voice leading: each voice moves to its target MIDI note independently
- No octave jumping in playback — voices move to the nearest pitch

### Neighbor Indicators
When on a tile, the 6 adjacent tiles show:
- Colored dot (blue=resolve, red=tension, gold=brighten, purple=darken)
- Text label of the relationship type
- Which voice moves and direction (e.g., "E↑F")

### Seed Type Visual Markers
Each landmark tile has a colored ring and label:
- Gold ring + "I" = tonic
- Blue ring + "pm" = parallel minor
- Orange ring + "V/" = secondary dominant
- Green ring + "N" = neighbor key
- Pink ring + "bII" = tritone sub

### Navigation
- **Click** any visible hex tile to teleport there
- **WASD / Arrow keys** to drive (car physics with acceleration, friction, turning)
- Minimap in top-right shows explored area with landmark dots

---

## Known Issues & Areas for Improvement

### Chord naming was removed
The auto-identifier (`identifyChord`) was producing wrong names (e.g., labeling Cmaj7 as having Ab). The HUD now shows raw note names instead. Proper chord naming could be revisited.

### Colors need work
The consonance-based coloring produces a narrow range that doesn't feel informative. Options discussed:
- Color by interval content (fifths vs thirds vs seconds)
- Color by distance from origin
- Drop color coding entirely, let indicators do the work

### Voice leading across territory seams
Tiles at the boundary between two landmarks' territories may differ by 2+ notes. Hard borders now visualize this, but the *musical* experience of crossing a seam is still abrupt. Potential fix: generate "bridge" chords at seams that share notes with both sides.

### Indicator accuracy
The resolve/tension detection is much improved (proper consonance scoring, suspension detection, tritone resolution) but still imperfect. The user noted some cases still feel wrong.

### Web MIDI output
Discussed but not implemented. The architecture supports it — Web MIDI API can send to virtual MIDI ports (IAC Driver on macOS, loopMIDI on Windows) for routing to DAWs/synths. Each tile change would send note-on/off events.

### The user's chord theory document
The user shared a Google Doc about chord theory (couldn't be fetched directly but was discussed). Key insights incorporated:
- Chords exist on a spectrum, not in fixed categories
- The "brightness" spectrum maps to modes (lydian→locrian)
- Every chord can relate to one of three diminished 7th families (Barry Harris concept)
- Lao Tzu-inspired hierarchy: 1 (tonic) → 2 (major/minor) → 3 (T/SD/D) → 10,000 things
- Traditional tonic/subdominant/dominant was evaluated and rejected as too relational/ambiguous for this context

---

## File Structure

Single file: `chord-drive.html` (~760 lines)
- All CSS inline in `<style>`
- All JS inline in `<script>`
- No external dependencies
- No build step

---

## Potential Next Steps (discussed but not built)

1. **Web MIDI output** — send chords to external synths/DAWs
2. **Better tile colors** — more perceptually meaningful color mapping
3. **Bridge chords at territory seams** — smoother transitions between landmark regions
4. **Chord naming** — smarter identifier or alternative labeling
5. **Sound design** — richer synth (more oscillator types, filter, envelope) or sample-based playback
6. **Record/playback** — record your path and replay the chord progression
7. **Share routes** — export a sequence of hex coordinates as a "composition"
8. **More landmark types** — augmented chords, quartal voicings, cluster seeds
9. **Touch/mobile support** — tap to navigate on mobile devices
10. **Visual polish** — the user is design-focused; the current aesthetic is functional but minimal
