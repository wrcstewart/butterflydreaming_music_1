# ButterflyDreaming Music Module — Specification v0.1

## Overview

This document is the source of truth for the first exemplar media module for the
ButterflyDreaming platform. It describes two HTML files — a harness and a music
module — that together demonstrate the BD media module protocol. Claude Code should
read this document in full before writing any code and refer back to it throughout.

---

## Context

ButterflyDreaming is a dyadic encounter platform where two anonymous users
collaboratively explore and weave symbolic text nodes. The platform is documented
at https://butterflydreaming.info.

Media modules are self-contained HTML files that can be triggered from within the
platform to render a text node as sound, image, or other media. They communicate
with the main platform via the browser postMessage API. The text node content is
the single source of truth — all rendering parameters are encoded within it.

This exemplar uses ABC music notation as the text format and plays it back using
real acoustic recorder samples via the Tone.js Sampler.

---

## Project Structure

The project folder is called `butterflydreaming_music_1` and contains:

```
butterflydreaming_music_1/
  MUSIC_MODULE_SPEC.md        ← this file
  harness.html                ← simulates the main platform (to be created)
  music_module.html           ← the media module (to be created)
  bass-recorder/              ← sample files (already present)
    bass_recorder_F#2.mp3
    bass_recorder_C3.mp3
    bass_recorder_G#3.mp3
    bass_recorder_E4.mp3
```

---

## Sample Files

The bass recorder samples are real acoustic recordings from the Versilian Community
Sample Library (VCSL), licensed CC0 (public domain). They are MP3 files at 192kbps
stereo, converted from the original WAV masters.

The four anchor notes and their Tone.Sampler pitch mappings are:

| Filename                  | Tone.Sampler key |
|---------------------------|-----------------|
| bass_recorder_F#2.mp3     | "F#2"           |
| bass_recorder_C3.mp3      | "C3"            |
| bass_recorder_G#3.mp3     | "G#3"           |
| bass_recorder_E4.mp3      | "E4"            |

Tone.Sampler will automatically pitch-shift between these anchor points to cover
all notes in the ABC score.

---

## Libraries

Both HTML files should load all libraries from CDN — no npm, no build step.

| Library   | Purpose                              | CDN |
|-----------|--------------------------------------|-----|
| Tone.js   | Audio engine and Sampler             | https://cdnjs.cloudflare.com/ajax/libs/tone/14.7.77/Tone.js |
| abcjs     | ABC notation parser and renderer     | https://cdnjs.cloudflare.com/ajax/libs/abcjs/6.2.2/abcjs-basic-min.js |

Note: abcjs is used here only for ABC parsing to extract the note sequence.
Visual score rendering is NOT required in this exemplar.

---

## The postMessage Protocol

This is the communication contract between the harness and the music module.
All messages are plain JavaScript objects posted via window.postMessage.

### Harness → Module

```javascript
// Load and play a new ABC score
{
  type: "BD_INIT",
  payload: {
    text: "X:1\nT:Dream\nM:4/4\nL:1/4\nK:Amin\n|A2BC DEFG|\n"
  }
}

// Stop playback
{
  type: "BD_STOP"
}
```

### Module → Harness

```javascript
// Module has loaded successfully and is ready
{ type: "BD_READY" }

// User has edited the ABC text in the module and pressed Send Back
{
  type: "BD_UPDATE",
  payload: {
    text: "X:1\nT:Dream\nM:4/4\nL:1/4\nK:Amin\n|A2BC DEFG|\n"
  }
}

// An error occurred
{
  type: "BD_ERROR",
  payload: { message: "Failed to parse ABC notation" }
}
```

During development, the origin check should use "*". A comment in the code
should note that this should be tightened to "https://butterflydreaming.info"
in production.

---

## File 1: harness.html

### Purpose
Simulates the main ButterflyDreaming platform for development and testing purposes.
This is NOT part of the final platform — it is a development tool that demonstrates
how the main platform will interact with media modules.

### Layout
A clean, simple single-page layout with two columns:

**Left column — Controls:**
- A label: "ABC Notation"
- A `<textarea>` (id: `abc-input`) pre-populated with a short test ABC score
  (see Default ABC Score below)
- A button: "Send to Player" — sends BD_INIT to the iframe
- A status area showing the last message received from the module

**Right column — Player:**
- An `<iframe>` (id: `music-module`) pointing to `music_module.html`
- Should be large enough to show the module UI comfortably — suggest 600x400px

### Behaviour

1. On page load, the textarea is pre-populated with the default ABC score
2. When "Send to Player" is clicked:
   - Read the current text from the textarea
   - Post a BD_INIT message to the iframe
3. Listen for postMessage events from the iframe:
   - BD_READY → update status area: "Module ready"
   - BD_UPDATE → update the textarea with the new text from payload.text
     and update status area: "Received update from module"
   - BD_ERROR → update status area: "Error: [message]"

### Default ABC Score

```
X:1
T:Dream Fragment
M:4/4
L:1/8
Q:1/4=60
K:Amin
|A2 B2 c2 d2|e4 c4|B2 A2 G2 F2|E8|
|A2 c2 e2 c2|A4 E4|F2 G2 A2 B2|c8|
```

This score is in A minor, at a slow tempo (60bpm), within the range of the bass
recorder samples. It should produce a recognisable melodic phrase.

---

## File 2: music_module.html

### Purpose
The media module itself. A self-contained HTML file that receives ABC notation
via postMessage, parses it, and plays it through the Tone.js Sampler using the
bass recorder samples. This file is the reference implementation for third-party
media module developers.

### Layout
A clean, minimal single-page layout:

- A status line at the top showing current state
  (e.g. "Loading samples...", "Ready", "Playing", "Stopped")
- A Play / Pause button (disabled until samples are loaded)
- A Stop button
- A label: "ABC Notation (editable)"
- A `<textarea>` (id: `abc-display`) showing the current ABC text —
  editable by the user
- A "Send Back" button — posts BD_UPDATE to the parent window with the
  current textarea contents

### Initialisation Sequence

1. On page load, immediately post BD_READY to parent — this tells the harness
   the iframe has loaded (even before samples are ready)
2. Begin loading Tone.Sampler with the four bass recorder samples:
   ```javascript
   const sampler = new Tone.Sampler({
     urls: {
       "F#2": "bass_recorder_F#2.mp3",
       "C3":  "bass_recorder_C3.mp3",
       "G#3": "bass_recorder_G#3.mp3",
       "E4":  "bass_recorder_E4.mp3"
     },
     baseUrl: "./bass-recorder/",
     onload: () => {
       // update status to "Ready"
       // enable Play button
     }
   }).toDestination()
   ```
3. Update status to "Loading samples..."
4. Listen for postMessage events from the parent window

### Message Handling

**BD_INIT received:**
1. Store the ABC text
2. Display it in the textarea
3. Parse the ABC using abcjs to extract the note sequence
4. If parsing fails, post BD_ERROR and update status
5. If parsing succeeds, update status to "Ready — press Play"
6. Do NOT auto-play — wait for the user to press Play

**BD_STOP received:**
1. Stop any current playback
2. Update status to "Stopped"

### ABC Parsing and Playback

Use abcjs to parse the ABC string:

```javascript
const parsed = ABCJS.parseOnly(abcText)
```

Extract the notes from the parsed output. Each note has a pitch and duration.
Map ABC pitch names to Tone.js scientific pitch notation:
- ABC uses letter names with octave markers (, for lower, ' for higher)
- Tone.js uses C4 style notation
- Middle C in ABC is C (no modifier) = C4 in Tone.js

Build a sequence of { note, duration, time } objects and schedule them using
Tone.Transport and sampler.triggerAttackRelease().

Use Tone.Transport for timing so that Play/Pause/Stop work correctly:
- Play → Tone.Transport.start()
- Pause → Tone.Transport.pause()
- Stop → Tone.Transport.stop() and reset position to 0

When the sequence ends, update status to "Finished" and reset transport position.

### ABC Pitch to Tone.js Mapping

ABC middle octave (no modifier) maps to octave 4 in Tone.js:

| ABC | Tone.js |
|-----|---------|
| C   | C4      |
| D   | D4      |
| E   | E4      |
| c   | C5      |
| d   | D5      |
| C,  | C3      |
| D,  | D3      |
| C'' | C6      |

Sharps: ^C = C#4, Flats: _C = Db4

### Duration Mapping

ABC note lengths relative to L (unit note length):
- Default L:1/8 means one unit = eighth note = "8n" in Tone.js
- A2 = two units = quarter note = "4n"
- A4 = four units = half note = "2n"
- A/ = half unit = sixteenth note = "16n"

The tempo is set by the Q field in the ABC header (e.g. Q:1/4=60 means
60 quarter notes per minute). Set Tone.Transport.bpm.value accordingly.

---

## Visual Design

Both files should have a calm, minimal aesthetic consistent with ButterflyDreaming:

- Background: very dark (near black) — suggest #0a0a0f
- Text: off-white — suggest #e8e8e0
- Accent: a muted teal — suggest #4a9b8e
- Buttons: subtle, not garish — dark background with teal border and text
- Font: a clean serif or system font — suggest Georgia or system-ui
- No gradients, no shadows, no animation beyond functional feedback
- Textareas: dark background, monospace font for the ABC text

A brief comment at the top of each file should read:
"ButterflyDreaming Media Module — [filename] — BD Protocol v0.1"

---

## Error Handling

- If Tone.Sampler fails to load a sample file, log the error to console and
  post BD_ERROR with a descriptive message
- If ABC parsing produces no notes, post BD_ERROR: "No playable notes found"
- If the browser does not support Web Audio API, display a clear message in
  the status area: "Web Audio not supported in this browser"
- All postMessage handlers should be wrapped in try/catch

---

## Testing Checklist

Before considering this exemplar complete, verify:

- [ ] Samples load without errors in Chrome, Firefox and Safari
- [ ] BD_INIT received → ABC displayed in module textarea
- [ ] Play button plays the score using the bass recorder sound
- [ ] Pause button pauses mid-score and resumes from same position
- [ ] Stop button stops and resets to beginning
- [ ] Editing the module textarea and pressing Send Back updates the harness textarea
- [ ] Editing the harness textarea and pressing Send to Player replays with new score
- [ ] BD_ERROR displayed correctly in harness status for malformed ABC
- [ ] No console errors on page load

---

## What This Exemplar Demonstrates

For third-party module developers, this file shows:

1. How to receive BD_INIT and extract text content
2. How to post BD_READY on load
3. How to post BD_UPDATE when the user modifies content
4. How to post BD_ERROR on failure
5. How to use the bass recorder samples with Tone.Sampler
6. The expected visual register of a BD media module

A third-party developer building a different module (graphics, XR, different
instrument) need only follow the postMessage protocol. Everything else is their
creative freedom.

---

## Notes for Claude Code

- Do not use any framework (no React, no Vue) — vanilla HTML, CSS and JavaScript only
- Do not use npm or any build tool — CDN only
- Keep each file self-contained — no shared JS files between harness and module
- Use ES6+ JavaScript (const, let, arrow functions, async/await) throughout
- Add clear comments explaining each section, especially the postMessage handlers
- The bass-recorder folder is at the same level as the HTML files — use relative paths
- Test ABC score must be within the range F#2 to E4 (the range of the samples)
- Do not auto-play on BD_INIT — always wait for user to press Play

---
## Amendments to Spec v0.1

### Amendment 1 — Key Signature Handling
The ABC parser must honour the key signature declared in the K: field of the
ABC header. When using abcjs parsed output, use the note objects returned by
the parser directly — do not re-parse raw letter names. The abcjs parser
applies key signature accidentals to each note automatically in its output.
This ensures that for example K:D correctly makes all F, C and G notes sharp
without them needing explicit ^ markers in the score.

### Amendment 2 — Play/Pause/Stop Controls
Replace the separate Play and Pause buttons described earlier with:
- A single **Play/Pause toggle button** whose label changes depending on state:
  - Shows "Play" when stopped or paused
  - Shows "Pause" when playing
- A separate **Stop button** that stops playback and returns position to the
  beginning of the score
This is the standard audio player pattern and what users will expect.
*ButterflyDreaming — a reflective ecosystem — Media Module Spec v0.1*
