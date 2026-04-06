# Ear Tuner — Project Spec & Handoff Document

## Overview

**Ear Tuner** is a single-file PWA (`ear-tuner.html`) for iPhone/mobile that trains aural acuity — the ability to distinguish which of two notes is slightly higher or lower in pitch. It is designed to be used eyes-free (while driving, practicing, etc.) via tap and swipe gestures.

**Author:** Casey Mullen (`casey.mullen@gmail.com`)  
**Feedback email:** `eartunerapp@gmail.com` (constructed via JS at runtime — never put as plain text in HTML or Cloudflare will obfuscate it)  
**GitHub:** caseymullen  
**Output file:** `ear-tuner.html` (single self-contained file, no build step)

-----

## Visual Design

- **Font:** `Inconsolata` (body/labels) + `Nunito` (bold values)
- **Listening background:** `#4d1903` (dark orange)
- **Retest background:** `#0d2a1a` (dark green, formerly review mode color)
- **Settings background:** `#1a1a1a`
- **Info background:** `#f5efe6` (warm off-white)
- **Vignette:** `radial-gradient(ellipse at center, transparent 25%, rgba(0,0,0,0.55) 100%)`
- Safe area vars: `--safe-top`, `--safe-bot`, `--safe-l`, `--safe-r`
- `setBg(color)` syncs bg-fill, body, html, and meta-theme-color for Safari edge-to-edge

-----

## Architecture

Single HTML file. All JS inline in one `<script>` block. No frameworks, no build.

### Key constants

```js
const CENTS_SEQ = [100, 50, 25, 20, 15, 10, 7, 6, 5, 4.5, 4.0, 3.5, 3.0, 2.5, 2.0, 1.5, 1.0, 0.5];
const MAX_CENTS = 100;
const NOTE_GAP  = 0.42; // seconds between the two notes
const DUR_STEPS = [0.8, 1.2, 1.6, 2.0, 2.2, 2.8, 3.5, 4.5]; // note durations
const ALL_NOTES = /* A1–G7, MIDI 33–103, octave computed as Math.floor(midi/12)-1 */
```

### Default settings (localStorage key: `vio4-settings`)

```js
{
  lowestNote: 22,      // G3 (MIDI 55) — index into ALL_NOTES
  highestNote: 53,     // D6 (MIDI 86)
  startCentsIdx: 5,    // 10¢ — index into CENTS_SEQ
  noteDurIdx: 3,       // 2.0s
  attack: 1,           // 'Med'
  decay: 1,            // 'Med'
  soundIdx: 0,         // Violin (sample)
  testsPerRound: 3,
  shakeToReplay: false
}
```

### Stats (localStorage key: `vio4-stats`)

Per note name: `{ bestCents, lastFailureCents, attempts }`

- `bestCents`: lowest cents successfully completed a full round at
- `lastFailureCents`: most recent failure cents value
- Regression = `lastFailureCents >= bestCents`

### Logging (localStorage keys: `vio4-log`, `vio4-logging`)

- `logEvent(msg)` — writes `[timestamp] msg` to `vio4-log`, capped at 200 entries (circular)
- Enabled via Settings toggle; survives restarts
- Instrumented events: `visibilitychange`, `pageshow`, `playBothNotes` (with `audioCtx.state`), `startRound`, `roundSuccess`, `roundFail`, `retestStart`, `retestExit`

-----

## State Machine

### States

|State               |Description                                     |
|--------------------|------------------------------------------------|
|**START**           |Start button visible, no audio, no note selected|
|**LISTENING**       |Round in progress, awaiting user answer         |
|**LISTENING-FAILED**|Round failed, Continue visible, tap replays     |
|**RETEST**          |Retesting a specific note from the scores table |
|**RETEST-FAILED**   |Retest round failed, Retry+Continue visible     |
|**RETEST-ENDING**   |Retest done after ≥1 success, Retry+Back visible|
|**INFO**            |Info/scores overlay open                        |

### Transitions

**START**

- Start pressed / tap area → **LISTENING**
- Info button → **INFO** (returns to START)

**LISTENING**

- Correct answer, round incomplete → **LISTENING** (next attempt, same note/diff)
- Correct answer, round complete → **LISTENING** (same note, harder diff, chime plays)
- Wrong answer → **LISTENING-FAILED**
- Info button → **INFO**
- Scores table Retest button → **RETEST**

**LISTENING-FAILED**

- Continue / swipe right → **LISTENING** (new note if hasBest, else same note reset)
- Tap area / replay button → replays both notes (stays in LISTENING-FAILED)
- Tap individual note circle → replays that note only
- Info button → **INFO**

**RETEST**

- Correct answer, round incomplete → **RETEST** (next attempt)
- Correct answer, round complete → **RETEST** (same note, harder diff, chime plays)
- Wrong answer, first round → **RETEST-FAILED** (step easier)
- Wrong answer, no successes yet → **RETEST-FAILED** (step easier)
- Wrong answer, after ≥1 success → **RETEST-ENDING**
- Exit retest button / close button (top-right X) → **LISTENING**
- Info button → **INFO**

**RETEST-FAILED**

- Continue / swipe right → **RETEST** (same note, easier diff)
- Retry / swipe left → **RETEST** (same note, from bestCents)
- Tap area / replay button → replays both notes
- Exit retest button → **LISTENING**

**RETEST-ENDING** (≥1 success, then failure)

- Back / swipe right → **INFO** (scrolled to Scores table)
- Retry / swipe left → **RETEST** (fresh retest from bestCents)
- Tap area / replay button → replays both notes

**INFO**

- Close button (top or bottom) → returns to prior state
- Scores table Retest button → **RETEST**
- If no active round when closing → `startRound()` is called

-----

## Gesture Map (Eyes-Free Operation)

|Gesture                 |Effect                                                 |
|------------------------|-------------------------------------------------------|
|Tap lower half of screen|Replay both notes (always); or Start if on start screen|
|Tap left note circle    |Play note 1 only (when roundFailed)                    |
|Tap right note circle   |Play note 2 only (when roundFailed)                    |
|Swipe up                |“Second note is higher” (answer)                       |
|Swipe down              |“Second note is lower” (answer)                        |
|Swipe right             |Continue / Back (when visible)                         |
|Swipe left              |Retry (retest only, when Retry button visible)         |

-----

## Round Logic

### Note selection (listening mode)

1. Untested notes in range → picked with equal probability
1. All tested → weighted by `bestCents` (higher = worse = more likely); regressed notes get 1.25× weight
1. Regression = `lastFailureCents >= bestCents`

### On correct answer (full round complete)

- `bestCents` updated if improved
- `lastFailureCents` cleared if `cents >= lastFailureCents`
- `centsIdx++` (harder)
- Chime plays (`chimeSuccess()` — boxing bell at A4, 440Hz, 2.5s decay)
- `diff-value` label NOT updated until just before next round starts (after 3200ms delay)
- Message: “Good job! Trying X¢, next!” (listening) or “Good job! Going harder… Trying X¢, next!” (retest)

### On wrong answer

- `lastFailureCents` set to current cents
- `pendingCentsStep = -1` (applied on Continue)
- Message: “Wrong. The first/second note was higher.”
- In listening mode: if no `bestCents` for this note → stay on same note; else new note
- In retest: first round → try again easier; no successes → try again easier; ≥1 success → RETEST-ENDING

### Difficulty stepping

- `pendingCentsStep` is deferred — applied when user presses Continue/Back, not immediately
- This keeps the display showing the correct value during the failed-round state

-----

## Audio

- Web Audio API, `AudioContext` / `webkitAudioContext`
- `ensureAudio()` is **async** and **awaits** `audioCtx.resume()` — critical for iOS
- `playBothNotes()` is **async** and **awaits** `ensureAudio()` before scheduling
- Soundfonts from: `https://caseymullen.github.io/microbreaker/ear/sounds/`
- Default sound: Violin (sample), soundfont-player library (gleitz MusyngKite)

### Feedback sounds

- **Correct (per-attempt):** Two sine tones, C5→E5, 0.12s apart, 0.22s decay
- **Wrong:** Two triangle tones, A3→G3, 0.15s apart, 0.42s decay (falling)
- **Round success (chime):** Boxing bell at A4 (440Hz), inharmonic partials at 1×/2.756×/5.404×/8.933×, 2.5s decay. Called with `stopAllSounds()` first + 150ms gap.

-----

## UI Structure

```
#app (flex column, fixed inset)
  #phase-label          — top: "Listen" / "Re-testing [note]"
  #upper-half           — diff-value + "difference" label
  #circles-zone         — two .note-circle divs + #replay-btn + #start-btn
  #lower-half           — #tap-area (z-index 0) + #progress-row + #status-msg + #round-actions + #exit-retest-btn

Fixed elements:
  #swipe-hint           — bottom center, above corner buttons
  #retest-close-btn     — top right, shown in retest (green X circle)
  #info-btn             — bottom right (ⓘ)
  #settings-btn         — bottom left (gear)
```

### Progress indicators

- Each slot: `.prog-slot` with a grey placeholder circle (`rgba(255,255,255,0.08)`)
- Filled by `renderProgress()` with green ✓ or red ✗ SVG

### Round actions (shown when `roundFailed`)

- `#retry-action-btn` (blue) — retest only
- `#continue-action-btn` (dark) — normal and retest
- `#back-action-btn` (cream `#f5efe6`, black text) — retest-ending only

-----

## Info Page

- Collapsible text: first paragraph always visible; paragraphs 2–6 hidden behind “… more” button on phones (`min-width: 600px` breakpoint auto-expands)
- Sections: text → Average Score → Score Chart → Scores table → Reset All Scores → attribution
- Score chart: dynamic x-axis based on worst current score
- Scores table: sorted by frequency (ascending pitch), filtered to current note range, highlighting for regression (yellow) and no-best/has-failure (faint red)
- Retest button in table: green-tinted (`rgba(26,61,40,0.15)`)
- Email link constructed via JS (never in static HTML): `eartunerapp@gmail.com`

-----

## iOS-Specific Handling

- `setBg()` syncs all background surfaces for edge-to-edge color
- `pageshow` handler clears `#app` visibility (fixes BFCache restore showing info page)
- `showStartScreen()` resets ALL game state (fixes partial JS-heap restore showing wrong state)
- Orientation change clears inline font sizes on `.hint-text`
- `ensureAudio()` is async/await throughout — iOS requires gesture-originated resume

-----

## Pending Tasks

### Group 2 remaining polish

- Copy/Clear log buttons: use rounded rectangles (not ovals/pills)
- Both buttons show brief visual feedback on press (e.g. “Copied!” label change, or button flash)

### Group 3 — Audio focus-loss debugging (do in isolation)

**The bug:** After switching to another app and returning to Ear Tuner, sounds sometimes fail to play — not just interrupted sounds, but all new sounds too. Switching away and back a second time often fixes it. This suggests `AudioContext` stays in `suspended` state after the first return, and our `visibilitychange`/`pageshow` `resume()` calls are being ignored by iOS because they don’t originate from a user gesture.

**What’s already in place:**

- `ensureAudio()` is async and awaits `resume()`
- `playBothNotes()` awaits `ensureAudio()`
- `visibilitychange`, `pageshow`, `focus` all call `audioCtx.resume().catch(()=>{})`
- Logging now captures `audioCtx.state` at `visibilitychange`, `pageshow`, and `playBothNotes`

**Plan:**

1. Enable logging on the device
1. Reproduce the bug (switch apps, return, tap, hear silence)
1. Copy the log and paste it into a new chat session
1. Analyze: what is `audioCtx.state` at `playBothNotes`? Is it `suspended` or `running`? Does `visibilitychange` show `running` before `playBothNotes` is called?
1. Based on evidence, likely fix: call `audioCtx.resume()` directly inside the tap/touch handler (which IS a user gesture) rather than relying on visibility events. Something like:

```js
document.getElementById('tap-area').addEventListener('touchstart', async () => {
  if (audioCtx && audioCtx.state === 'suspended') await audioCtx.resume();
}, { passive: true });
```

1. May also need to add resume inside `handleTap`, swipe handlers, and note circle tap handlers — anywhere a user gesture triggers audio.

**Key insight:** iOS WebKit only allows `AudioContext.resume()` to succeed when called from within a user gesture event handler. Calling it from `visibilitychange` or `pageshow` may silently fail. The fix must move `resume()` into the actual touch/click handlers.