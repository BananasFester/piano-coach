# Piano Coach

A single-file practice app for a beginner pianist. It listens to the piano through the
phone's microphone, shows the next note to play, and keeps score. Everything — markup,
styles, songs and code — lives in **`index.html`**. There is no build step, no
dependencies and no server code: open the file and it runs.

---

## Running it

| How | Microphone? | Notes |
| --- | --- | --- |
| Double-click `index.html` | No | `file://` is not a secure context, so the browser blocks the mic. The on-screen keyboard still works, and the app says so in a banner. |
| `http://localhost` | Yes | Localhost counts as secure. Any static server works, e.g. `npx serve .` |
| GitHub Pages / any https host | Yes | This is how it is meant to be used — phone on the music stand. |

A quick local server without installing anything:

```bash
node -e "const h=require('http'),f=require('fs');h.createServer((q,s)=>{s.writeHead(200,{'Content-Type':'text/html'});s.end(f.readFileSync('index.html'))}).listen(8777)"
# then open http://localhost:8777
```

---

## The four modes

| Mode | What happens | Scored on |
| --- | --- | --- |
| **Learn** | The song waits on each note until it is played correctly. The next key glows on the keyboard. | notes played vs. wrong keys |
| **Play** | The song keeps the beat after a 4-beat count-in. Notes fall onto the keyboard and must be hit at the red line. | Perfect / Good / Missed / Wrong |
| **Read** | **No keyboard at all.** The notes are drawn as real sheet music on a staff, and it waits like Learn mode. This is for learning note names and book reading. | notes read vs. wrong answers |
| **Listen** | Plays the tune so she knows how it is supposed to sound. | not scored |

### Read mode details

* Right hand is drawn on the treble staff, left hand on the bass staff (grand staff on
  Medium and Hard). Note heads, stems, flags, sharps and ledger lines are all drawn, so
  middle C really does sit on its own ledger line below the treble staff.
* The page scrolls so the note she owes stays about one beat in from the left, under a
  faint red marker.
* Two ways to answer: **play it on the piano** (mic), or **tap the letter** on the pad
  below the staff. The pad has C D E F G A B plus a `♯` modifier that applies to the next
  tap. A pad answer is resolved to the octave nearest the printed note, so naming the
  letter correctly is enough.
* Letter names under the note heads are **off by default** — that is the point of the
  mode. Turn them on in Settings ▸ *Letter names in Read mode* while she is starting out.
  The "Next note" readout in the HUD shows `?` while they are off.

---

## Difficulty levels

The selector appears both in the library and on the practice screen. It does not change
the melody — it changes **how much of the left hand comes along**:

| Level | Left hand | Built from |
| --- | --- | --- |
| **Easy** | none — right hand only | `rh` |
| **Medium** | one held root note per bar | `chords`, root only |
| **Hard** | a moving broken-chord part, one note per beat | `chords`, root–fifth–third–fifth |

Left-hand notes are drawn **blue**, right-hand notes red, so she can see which hand owns
what. The left hand is voiced from C3 upward (`BASS = 48`), and a chord seventh is placed
*below* the root so the hand never has to stretch past a sixth.

In Learn and Read mode every note of a chord must be played before the song moves on —
that is what makes Medium and Hard genuinely two-handed practice rather than a melody with
decoration.

A song with no `chords` field stays right-hand-only at every level; the Medium and Hard
buttons grey out and the hint line says why.

---

## Microphone and two hands

The mic hears through a single pitch tracker (the McLeod Pitch Method, in the
`pitch detection` section of `index.html`), which can only follow **one** note at a time.
That matters most for the very common case of two hands playing the **same letter an
octave apart** — a scale practiced hands-together, or a Hard-mode left hand that happens
to land on the same note name as the melody.

Acoustically, that case is a dead end for pitch detection: a note and its own octave,
played together, produce a sound wave with *no periodicity the lower note doesn't already
have* — there is nothing left for a second detector to find, no matter how clever. So the
fix lives in the judging logic instead of in pitch detection: in `input()`, when a detected
note matches more than one still-owed note in the current chord (same letter, different
octave), **all of them are credited at once**, not just the first. This is what actually
makes two-hand octave practice (like the C major scale with both hands) work — it isn't
pretending to hear two notes, it's recognizing that one reading can't help but mean both.

This only fires for notes that share a letter name. Two hands playing genuinely
**different** notes together (e.g. a left-hand G under a right-hand E) are still only ever
tracked as one note, because a single pitch tracker truly cannot separate them reliably —
an earlier attempt at a "second pitch" heuristic was tested and dropped (see
`Verifying changes`) after it turned out to mistake a single note's own harmonics for a
second note. In practice this is rarely a problem: real hands are never struck in
perfect, microsecond sync, so the tracker usually catches both notes in quick succession
anyway. If a specific chord is consistently only half-registering, lowering **Mic
sensitivity** or slightly staggering the hands is the practical fix.

---

## Song format

Songs are plain objects in the `BUILTIN` array (see the `/* ---------- songs ---------- */`
banner in `index.html`).

```js
{id:'lightly', title:'Lightly Row', tier:'Beginner', bpm:92, bpb:4,
 rh:'G4 E4 E4:2 F4 D4 D4:2 C4 D4 E4 F4 G4 G4 G4:2 ...',
 chords:'C:4 G7:4 C:4 G7:4 C:4 G7:4 C:4 C:4'}
```

| Field | Meaning |
| --- | --- |
| `id` | Short unique string. **Never reuse or rename it** — saved best scores are keyed on it. |
| `title` | Shown in the library and the practice bar. |
| `tier` | Which shelf it sits on: `'Starter'`, `'Beginner'` or `'Intermediate'` (the `TIERS` array sets the order). This is about how tricky the *melody* is, and is separate from the Easy/Medium/Hard selector. |
| `bpm` | Beats per minute at 100% speed. |
| `bpb` | Beats per bar — 4 for 4/4, 3 for 3/4. Drives the beat lines, the metronome accent and bar lines in Read mode. |
| `rh` | The right-hand melody. |
| `chords` | Optional harmony track that builds the left hand. |

### Note syntax (`rh`)

* `C4` = middle C. Letter + optional `#`/`b` + octave number: `F#4`, `Bb3`, `G5`.
* Each note is **one beat** unless you add a duration: `G4:2` holds two beats,
  `C4:0.5` is half a beat, `G4:0.75` works too.
* `R` is a rest, and takes a duration the same way: `R:2`.
* Separate notes with spaces. `|` and `,` are also accepted as separators, so you can
  write bar lines in for readability: `C4 D4 E4 F4 | G4:4`.
* Range is A0–C8; anything outside that is rejected.

### Chord syntax (`chords`)

* A chord symbol followed by `:beats` — `C:4 G7:4 Am:2`. Without `:beats` it defaults to 4.
* Root is a letter plus optional `#`/`b`. Types: `` (major), `m`, `7`, `m7`, `maj7`,
  `dim`, `aug`, `sus4`, `sus2`, `6`, `5`.
* The track starts at beat 0 and runs in parallel with the melody. **Keep the total beats
  equal to the melody's total beats.** Anything past the end of the melody is dropped.
* One chord per bar is usually plenty; split a bar (`G:2 C:2`) when the melody changes
  harmony halfway through.

---

## Adding a song

1. Work out the melody in beats and write the `rh` string. Count the beats.
2. Write a `chords` string that adds up to the **same** number of beats.
3. Add the object to `BUILTIN` under the right `tier`, with a new `id`.
4. Check it (see *Verifying changes* below), then open the app and try all three levels.

For one-off or personal songs there is no need to touch the code at all — the
**+ Add your own song** button in the library takes a title, a bpm, the notes and an
optional chord track, and stores it in the browser under *My songs*.

A few library entries (Disney tunes, the two Star Wars themes) are **simplified personal
arrangements** of well-known melodies, not verbatim transcriptions — they're there for
this family's practice, not for publishing or distributing. Their titles say
"(simplified)" to keep that honest; keep that label on anything similar you add.

**When the song is a real, well-known piece, don't rely on memory alone for anything
beyond public-domain nursery rhymes and standard classical repertoire** (Twinkle Twinkle,
Für Elise, Minuet in G and the like are safe — they're extremely standard and were checked
against well-established knowledge of them). For anything more specific — a modern pop or
film song, a particular arrangement's chord changes — recalled-from-memory transcriptions
of these are genuinely unreliable in ways that are easy to miss until someone who knows the
song listens to it. The first pass at the Disney/Star Wars songs in this library had two
real errors this way: the Star Wars Main Theme was written in the wrong meter (4/4 instead
of the real 6/8) with the opening leap backwards, and Let It Go was missing the song's
entire defining feature — the shift from a minor verse into the relative major for the
chorus. Both were caught only when the person using the app said the songs "didn't sound
right," not by any check this repo can run on its own.

Before adding or trusting a transcription of a real song beyond simple public-domain
tunes, look up (don't just recall) at least: the **key**, the **time signature**, and a
plain-language description of the melody's shape for the part you're transcribing (rising
or falling, stepwise or leaping, where the memorable/structural moment is). Music-theory
analysis sites and Wikipedia-style articles are good for this — they describe the music
analytically rather than reproducing a copyrighted transcription outright, which also
keeps this closer to "an arrangement informed by analysis" than "a copy of someone else's
sheet music." Then write your own note sequence from that, rather than copying a found
tab/transcription verbatim. If you can't verify these basics, say so in the song's title or
a comment rather than shipping a guess as if it were checked.

---

## How the file is laid out

Search for these banner comments in `index.html`:

| Banner | What is there |
| --- | --- |
| `Library` / `Practice` / `Sheets` / `Difficulty + reading pad` | CSS. Light and dark themes are both defined as custom properties on `:root`; the canvas reads them via `readTokens()`, so colours stay in one place. |
| `LIBRARY` / `PRACTICE` / `RESULTS` / `SETTINGS` / `ADD SONG` | The five chunks of markup. Only one of library/practice is visible at a time. |
| `storage (optional)` | `store.get/set`, wrapped in try/catch so private-mode browsers still work. |
| `songs` | `TIERS` and the `BUILTIN` table. |
| `chords -> left hand` | `parseChords`, `buildLH`, `buildSong` — the difficulty engine. |
| `settings` / `state` | `settings` defaults and the single `S` state object. |
| `audio` | Oscillator-based piano tone and the metronome click. |
| `pitch detection` | The McLeod pitch method (`mpm`) plus `analyze()`, which decides when a note has actually been struck. The `/*MPM*/ … /*END*/` markers fence the algorithm itself. |
| `input handling` | `input(midi, src)` — the one funnel for mic notes, key taps and pad taps. Mode-specific judging lives here. |
| `keyboard layout` / `canvas` | `layoutKeys()` sizes the keyboard to the song's range; `draw()` renders falling notes + keyboard. |
| `Read mode: real notation` | `drawStaff()` and friends. `dia(midi)` maps a pitch to a diatonic step, which is what puts a note on the right line. |
| `main loop` | One `requestAnimationFrame` loop; Learn and Read freeze time on the next note, Play and Listen run the clock. |
| `HUD` / `flow` / `library` / `wiring` | DOM updates, mode and level switching, the song list, and all event listeners. |

Key state on `S`: `notes` (flat, sorted by start beat), `groups` (notes bundled by start
beat, so chords are one unit), `gi` (which group Learn/Read is waiting on), `mode`,
`level`, `t` (current position in beats).

---

## Settings and saved data

Everything is in `localStorage` under a `pianocoach_` prefix:

| Key | Contents |
| --- | --- |
| `pianocoach_settings` | mic sensitivity, any-octave matching, key labels, metronome, mic delay, read-mode letter names |
| `pianocoach_songs` | songs added through the Add song sheet |
| `pianocoach_best` | best score per `songId_mode_level`, e.g. `twinkle_play_hard` |
| `pianocoach_mode`, `pianocoach_level`, `pianocoach_tempo` | last used |

Two settings matter most when something feels wrong:

* **Mic sensitivity** — turn up if quiet notes are missed, down if room noise triggers notes.
* **Mic delay fix** — raise it if Play mode says "late" when she is on time.

---

## Verifying changes

There is no test framework, but the app's logic can be exercised from Node without a
browser, which is worth doing after touching songs or the engine.

**Song data** — pull the music functions out of the file and check every song:

```bash
node -e "
const fs=require('fs');
const app=fs.readFileSync('index.html','utf8').split('\r\n').join('\n').match(/<script>([\s\S]*)<\/script>/)[1];
const chunk=app.slice(app.indexOf('const NAMES='),app.indexOf('/* ---------- settings ---------- */'));
const m=new Function('localStorage',chunk+'\nreturn {BUILTIN,parseSong,parseChords,buildSong};')({getItem:()=>null,setItem:()=>{}});
for(const s of m.BUILTIN){
  const len=m.parseSong(s.rh).len, cb=m.parseChords(s.chords).reduce((t,c)=>t+c.dur,0);
  console.log((cb===len?'ok  ':'DIFF')+' '+s.id+'  melody '+len+'  chords '+cb);
}"
```

`DIFF` means the chord track and the melody disagree on length — fix it before shipping.

**Whole app** — the page script can also be run against a stubbed DOM and canvas (fake
`document.querySelector`, a proxy for the 2D context, a manual `requestAnimationFrame`
queue) to play every song in every mode at every level and catch runtime errors. That is
how this version was checked.

**Pitch detection** — `mpm()` can be called directly from Node with a synthetic waveform
(sum of a few sine harmonics, same shape as the app's own `tone()` synth) to check what it
reports for a given pitch or mix of pitches, without a mic or a browser. This is how the
two-hand fix was designed: a synthetic test of two notes an octave apart confirmed the
combined wave really does carry no extra information (so the fix belongs in the judging
logic, not the detector), and a synthetic test of a *single* note caught an earlier
"second pitch" heuristic mistaking that note's own harmonics for a second note, which is
why that heuristic isn't in here.

Always also open it in a real browser afterwards: the canvas rendering, touch targets and
microphone can only really be judged by eye and ear.

---

## Known limits and ideas

* **Monophonic ear.** The pitch detector hears one note at a time. Two hands on the
  **same letter** (an octave apart) are credited together regardless, since one reading
  can't mean anything else — see *Microphone and two hands* above. Two hands on
  **genuinely different** notes at the exact same instant are still only tracked as one;
  this is rarely an issue in practice since real playing is never perfectly
  synchronized, but a borderline chord can be nudged by lowering Mic sensitivity,
  staggering the hands slightly, or tapping the on-screen keys / Read mode's letter pad,
  which are unambiguous either way.
* **Rhythm is not graded in Read mode.** It waits for the right pitch and ignores timing.
  Note *values* are drawn (filled, hollow, flagged) but not judged.
* **Clef glyphs** come from the system font (`𝄞`, `𝄢`). If no font on the device has them
  the app detects it and falls back to the `RIGHT` / `LEFT` labels, which are drawn either
  way.
* **Generated left hands are formulaic by design** — a root per bar, or a broken chord per
  beat. A song can be given a hand-written left-hand part instead by extending
  `buildSong()`; the hook is deliberately small.
* Nice next steps: key signatures instead of per-note sharps, a rhythm-only drill, MIDI
  keyboard input via Web MIDI, and marking individual songs as "needs work".
