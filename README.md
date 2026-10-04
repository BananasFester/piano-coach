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

## Difficulty levels and the left hand

These are **two separate controls**, each appearing in both the library and the practice
screen. They used to be tangled together (difficulty used to be what added the left hand),
which turned out to be a real design mistake: it meant the only way to get an easier melody
was to also lose the left hand, and vice versa. They're now fully independent.

### Difficulty — simplifies the right-hand melody

Difficulty never touches hand count. It only changes **the right-hand melody itself**:
how many notes survive, and the default tempo.

| Level | What happens to `rh` | Default tempo |
| --- | --- | --- |
| **Easy** | kept only on whole beats; anything finer is dropped | 75% of the song's `bpm` |
| **Medium** | kept down to the half-beat; only the busiest spots are thinned | 90% |
| **Hard** | every note exactly as written | 100% |

The simplification (`simplifyRH()`) works by keeping notes that start on the level's beat
grid and dropping the rest — the note *before* a dropped note is held longer to cover the
gap, so no pitch is ever invented, busy passages are just thinned out. A note that happens
to repeat a pitch (like the eighth-note pairs in Hot Cross Buns) collapses cleanly into one
held note; a genuine passing tone on an off-beat gets smoothed away, same as a real "easy"
arrangement in a method book would do. The song's total length and the left hand's timing
are never affected — only which right-hand notes survive.

### Left hand — a slider, not a difficulty

A range slider (3 steps: **Off / Simple / Full**) controls whether a left hand plays at
all, independent of difficulty:

| Setting | Left hand | Built from |
| --- | --- | --- |
| **Off** | none — right hand only | — |
| **Simple** | one held root note per bar | `chords`, root only |
| **Full** | a moving broken-chord part, one note per beat | `chords`, root–fifth–third–fifth |

Left-hand notes are drawn **blue**, right-hand notes red, so she can see which hand owns
what. The left hand is voiced from C3 upward (`BASS = 48`), and a chord seventh is placed
*below* the root so the hand never has to stretch past a sixth. In Learn and Read mode
every note of a chord must be played before the song moves on — that is what makes Simple
and Full genuinely two-handed practice rather than a melody with decoration.

A song with no `chords` field stays right-hand-only no matter where the slider is set; it
greys out and the hint line says why.

Best scores are tracked per song **and** per mode, difficulty, *and* left-hand setting
(`songId_mode_level_leftHand`), since "Easy, no left hand" and "Easy, full left hand" are
genuinely different challenges and shouldn't share a high score.

---

## Microphone and two hands

The mic hears through a single pitch tracker (the McLeod Pitch Method, in the
`pitch detection` section of `index.html`), which can only follow **one** note at a time.
That matters most for the very common case of two hands playing the **same letter an
octave apart** — a scale practiced hands-together, or a Full left hand that happens to
land on the same note name as the melody.

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

### Transcribing real, modern songs — what failed, and what actually worked

Several attempts were made at adding Disney/Star Wars songs to this library. The first
three failed; this is worth recording so the failures aren't repeated and the fix is:

- **Attempt 1** transcribed the melodies from memory alone. Reported as unrecognizable.
- **Attempt 2** added web research first (key, time signature, a plain-language
  description of the melodic shape) and rewrote the melodies informed by those facts.
  This caught two real, confirmable errors (Star Wars Main Theme in the wrong meter with
  the opening leap backwards; Let It Go missing its minor-verse/major-chorus shift) but
  the results were still reported unrecognizable.
- **Attempt 3** looked for an exact, mechanically-copyable source instead of reconstructing
  one. Kalimba tabs turned out to force minor-key pieces into an unrelated major-key
  simplification (wrong instrument constraints). Text-based extraction of letter-note sites
  (e.g. noobnotes.net) was tried and correctly refused, by the tool itself, as reproducing
  a copyrighted arrangement verbatim even in letter-note form. Hooktheory's actual
  transcription data renders in an interactive widget that a *text*-based page fetch can't
  read at all.

**What finally worked: looking at the real notation, once, with the Claude in Chrome
browser extension, and writing an original simplified arrangement informed by that —
not extracting it.** Hooktheory (hooktheory.com/theorytab) hosts crowd-sourced chord +
melody transcriptions as an interactive piano-roll. Navigating to a song there, toggling
to show the melody, and reading the piano-roll grid *visually* (screenshots + zoom, the
same way a person would look at sheet music) gives real key/meter/chord-progression facts
directly, plus a trustworthy melodic *contour* (which notes are held, which are short,
roughly which scale degrees) — far more constraining than a prose description, without
being a mechanical bulk-copy of someone's exact transcription. The Disney/Star Wars entries
now in this library were built this way, each checked directly on Hooktheory's piano-roll
rather than recalled or guessed: Star Wars Main Theme's 6/8 meter and up-a-fifth opening
leap; Let It Go's Ab major / I–V–vi–IV chorus and repeated-note verse (its traced melody
range matched Hooktheory's own cited stat exactly); A Whole New World's D major key and its
"whole new world" hook actually descending stepwise (not leaping, as guessed before); Beauty
and the Beast's straight 4/4 (confirming the earlier 3/4-waltz version was simply wrong —
the waltz is a separate instrumental cue in the 2017 remake); and When You Wish Upon a
Star's confirmed opening octave leap landing on a held G4, matching the independently
sourced fact that an octave leap is the song's signature move. Hakuna Matata and You've Got
a Friend in Me were also checked this way and found to already match (key, meter, range,
melodic character) what was already written, so those were left unchanged.
The difference from attempt 3's text-extraction failures is specifically that this means
*looking once and writing a new, simplified interpretation* (different rhythm, different
key choices in places, fewer notes) — the same thing a student does after hearing a song a
few times — rather than having a tool reproduce the source's exact note-for-note sequence.

If the Claude in Chrome extension isn't connected, this path isn't available — see the
extension's own connection requirements before attempting it, and don't fall back to
attempt 1 or 2's approach in the meantime.

**What's always safe regardless of the above**, demonstrated by what's in this library and
held up without correction: standard nursery rhymes and rounds (Twinkle Twinkle, Hot Cross
Buns, Frère Jacques, Three Blind Mice, Old MacDonald, ...) and famous classical repertoire
(Ode to Joy, Für Elise, Minuet in G) that are taught so uniformly, everywhere, that there's
effectively one canonical version to get right from well-established knowledge alone. Also
safe: reusing a melody already in this file that's known correct — several children's songs
genuinely share a tune (Twinkle Twinkle / Baa Baa Black Sheep / The Alphabet Song; Mary Had
a Little Lamb / Merrily We Roll Along) — and original technical exercises (scales,
arpeggios, interval drills) that aren't a transcription of anything, so there's no "is this
really how it goes" question to get wrong in the first place.

---

## How the file is laid out

Search for these banner comments in `index.html`:

| Banner | What is there |
| --- | --- |
| `Library` / `Practice` / `Sheets` / `Difficulty + reading pad` | CSS. Light and dark themes are both defined as custom properties on `:root`; the canvas reads them via `readTokens()`, so colours stay in one place. |
| `LIBRARY` / `PRACTICE` / `RESULTS` / `SETTINGS` / `ADD SONG` | The five chunks of markup. Only one of library/practice is visible at a time. |
| `storage (optional)` | `store.get/set`, wrapped in try/catch so private-mode browsers still work. |
| `songs` | `TIERS` and the `BUILTIN` table. |
| `chords -> left hand` | `parseChords`, `buildLH` (the left-hand generator), `simplifyRH` (the difficulty engine), `buildSong` (combines both). |
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
`level` (melody difficulty), `leftHand` (`'off'`/`'simple'`/`'full'`, independent of
`level`), `t` (current position in beats).

---

## Settings and saved data

Everything is in `localStorage` under a `pianocoach_` prefix:

| Key | Contents |
| --- | --- |
| `pianocoach_settings` | mic sensitivity, any-octave matching, key labels, metronome, mic delay, read-mode letter names |
| `pianocoach_songs` | songs added through the Add song sheet |
| `pianocoach_best` | best score per `songId_mode_level_leftHand`, e.g. `twinkle_play_hard_full` |
| `pianocoach_mode`, `pianocoach_level`, `pianocoach_leftHand`, `pianocoach_tempo` | last used |

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
queue) to play every song in every mode, at every difficulty, with the left hand on `full`,
and catch runtime errors; a second, smaller sweep checks a few representative songs across
all three `leftHand` settings to confirm it's a genuinely independent axis (including that
a song with no `chords` stays one-handed regardless of the slider). That is how this
version was checked. The same harness also directly checks `simplifyRH()`: that a dropped
note's duration is absorbed into the note before it (no lost time), and that `'hard'`
changes nothing at all.

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
* **Difficulty simplifies rhythm, not pitch.** `simplifyRH()` only thins out note density —
  it has no opinion about accidentals or hand position. A piece like Chromatic Climb (one
  note per beat throughout) has nothing for Easy/Medium to simplify, since its difficulty
  comes entirely from the notes themselves, not the rhythm. That's an intentional scope
  limit, not a bug: changing *which pitches* appear would stop it being a simplified
  version of the same tune.
* Nice next steps: key signatures instead of per-note sharps, a rhythm-only drill, MIDI
  keyboard input via Web MIDI, and marking individual songs as "needs work".
