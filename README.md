# Cat Clicker

A clicker-training marker that lives on a phone home screen. One file, no
dependencies, no build step, no app store.

**Live:** https://11prateek.github.io/cat-clicker/

Tap anywhere on the screen to fire a click. The whole screen is the button so
you can use it without looking, which matters when you're watching the cat
instead of the phone.

---

## Installing it

Open the URL **in Safari** on iOS, tap the share button, then **Add to Home
Screen**.

This step isn't optional polish. Two things only work from the home screen icon
and not from a Safari tab:

- the fullscreen, chrome-free launch
- reliable audio when the ringer switch is off

If someone opens the link inside Instagram, Slack or Gmail's built-in browser,
the Add to Home Screen option won't appear. They need to hit "Open in Safari"
first.

---

## How it works

### Everything is one file

`index.html` contains the markup, the styles and the synthesizer. Opening it by
double-clicking works offline on a desktop. There's nothing to install and
nothing to compile.

### The click is synthesized, not recorded

There's no audio file. The waveform is generated in JavaScript at page load and
written into an `AudioBuffer`. A tap then just starts a buffer source, which is
the lowest-latency path the Web Audio API offers.

This matters because in clicker training the marker has to land within a
fraction of a second of the behaviour. Loading and decoding an MP3 would add
startup lag on the first play; a pre-baked buffer doesn't.

### What the sound is made of

A box clicker is a bent metal tongue. You press it and it snaps — that's one
sound. You release and it springs back — that's the second. The gap between
them is however long your thumb is down.

`build()` models exactly that:

```
snap 1  at 0 ms      pitch HZ          amplitude 1.00   decay 8 ms
snap 2  at GAP       pitch HZ × 0.78   amplitude 0.58   decay 6 ms
```

The release is lower and quieter than the press because it happens with less
force and different geometry.

Each snap is a 35 ms burst of white noise with an exponential decay envelope,
run through three parallel filters:

| layer   | filter             | weight | what it contributes            |
| ------- | ------------------ | ------ | ------------------------------ |
| `body`  | bandpass, Q 8      | 1.00   | the fundamental — the pitch    |
| `upper` | bandpass ×2.4, Q 10| 0.32   | inharmonic partial — the metal |
| `edge`  | highpass ×3        | 0.16   | the leading tick               |

The 2.4× ratio is deliberately not a musical interval. Struck metal produces
inharmonic partials, and that inharmonicity is a lot of what makes something
sound like an object rather than a note.

Removing either of the upper layers measurably changes the output (−3 dB and
−6.6 dB respectively), so all three are earning their place.

### The DSP

`resonate()` is a single biquad section that designs its own coefficients and
applies them in one pass:

```
y[n] = b0·x[n] + b1·x[n-1] + b2·x[n-2] − a1·y[n-1] − a2·y[n-2]
```

Bandpass and highpass share every line of the setup maths and differ only in
the three numerator terms, so they're one function with a `type` argument.
Frequencies get clamped below Nyquist inside `resonate()`, which means callers
can ask for any frequency without guarding.

After both snaps are summed the buffer is normalized to a peak of 0.95. This
keeps loudness constant if you retune `HZ`, since a bandpass filter's gain
varies with its centre frequency.

### Why a double-click needs a real gap

Below roughly 20 ms the ear fuses two transients into a single smeared event.
At `GAP = 0.08` the two snaps are 80 ms apart, which reads clearly as "ck-ck".

The audible tail ends around 87 ms. `DUR` only needs to satisfy
`DUR ≥ GAP + 0.035` — currently 0.25, which leaves generous slack.

---

## The iOS workarounds

Most of the non-obvious code exists because mobile Safari is hostile to audio.
Each of these fixes a symptom that actually showed up in testing.

**The mute switch.** By default Web Audio is silenced by the ringer switch.
`navigator.audioSession.type = 'playback'` overrides that. Without it the app
appears broken to anyone whose phone is on silent — which, for a 6am training
session, is most people. Requires iOS 16.4+.

**Context built at load, not on first tap.** Constructing an `AudioContext`
spins up audio hardware and takes real time. Doing it lazily inside the first
tap handler put that cost directly on the critical path of the first click.

**Waiting for `resume()`.** iOS starts the context suspended and won't let it
run without a user gesture. `resume()` is asynchronous, so firing a buffer
immediately after calling it schedules sound into a context that isn't running
yet. The code waits for the promise instead.

**The inaudible keep-alive loop.** iOS tears down the audio output route when
nothing has played for a while, and rebuilding it on the next tap is audible as
lag. A looping buffer at amplitude `1e-7` holds the route open. This was the
cause of the intermittent "sometimes it lags when I open it" behaviour.

**Re-resuming on `visibilitychange`.** Returning from the lock screen or app
switcher leaves the context suspended again.

**Wake lock with a `release` listener.** The screen shouldn't sleep mid-session.
iOS drops the lock when you background the app, so the code listens for the
`release` event and clears its reference — otherwise it would hold a stale
sentinel and never re-acquire one.

---

## Input gating

Accidental double-clicks are worse than they look. Three separate problems:

1. **Overlapping audio clips.** One click peaks at 0.95 × 0.85 ≈ 0.81. Two
   overlapping sum past 1.0 and the output hard-clips, which on a sharp
   transient sounds like a crackle rather than two clicks.
2. **Finger bounce.** One intended press registering as two touches.
3. **Nervous repeat tapping**, when you're not sure the first one landed.

Two gates handle these:

**Multi-touch rejection** uses `e.isPrimary`. A second finger or a resting palm
never clicks. This is stateless on purpose — counting active pointers is
fragile, because a `pointerup` lost when a finger slides off-screen leaves the
count stuck above zero and the app dead forever.

**A 300 ms lockout** after each successful click. Accidental bounces land
30–120 ms apart; deliberate repeat clicking rarely goes faster than three per
second (330 ms). Those barely overlap, which is what makes the window viable.

**Only a successful click resets the clock.** If blocked taps reset it too,
panic tapping would extend the lockout indefinitely and the app would go dead
in your hand.

### The feedback is deliberately asymmetric

| event         | what you see                               |
| ------------- | ------------------------------------------ |
| click fired   | pupil snaps to a narrow slit, springs back |
| tap suppressed| whole eye dims for a beat, pupil unmoved   |

Absence of the pupil snap is the real signal. The dim just confirms the screen
registered your finger at all.

This matters because **swallowing a click you meant is worse than letting a
double through**. If the app silently eats a click, you think you marked the
behaviour, the cat gets nothing, and you've taught it that the marker doesn't
reliably predict a treat — which is the one thing that destroys a conditioned
marker. Making suppression visible means you'd notice it happening.

---

## Tuning

All constants sit at the top of the script block.

| constant  | value  | what it does                                        |
| --------- | ------ | --------------------------------------------------- |
| `HZ`      | 3500   | Pitch of the snap, in Hz. Roughly where a real box clicker sits. |
| `GAP`     | 0.08   | Seconds between press and release. Keep above 0.02. |
| `DUR`     | 0.25   | Buffer length. Must be at least `GAP + 0.035`.      |
| `VOL`     | 0.85   | Master gain. Linear amplitude, not perceived loudness — 0.5 is about −6 dB, not "half". Above 1.0 clips. |
| `LOCKOUT` | 300    | Milliseconds of deafness after a click.             |

### Finding the right pitch for your cat

Sweep `HZ` while she's relaxed and watch for an ear flick or head turn. Cats
hear well into the ultrasonic, but a phone speaker rolls off hard past about
15 kHz, so you're limited to the top of human hearing anyway.

Once you find a pitch she orients to, **lock it and don't touch it again**. A
conditioned marker works because it's identical every single time.

---

## Deploying changes

The repo is served by GitHub Pages from `main` / root. Edit `index.html`,
commit, and Pages rebuilds in about a minute. The Actions tab shows when
`pages-build-deployment` finishes.

**iOS caches aggressively.** After a deploy, the home screen icon may still run
the old version. In rough order of effort:

1. Force-quit and relaunch.
2. Settings → Apps → Safari → Clear History and Website Data. Home screen apps
   share Safari's cache.
3. Delete the icon and re-add it. Always works.

Adding a version query string when you re-add it (`?v=2`) sidesteps this
entirely — iOS treats it as a different URL. Bump the number each deploy.

---

## Constraints worth knowing

- **The repo must be public.** GitHub Pages on the free tier won't serve a
  private repo.
- **The file must be named `index.html`**, or Pages won't serve it at the
  directory root.
- **The first tap after a cold launch is always marginally slower.** iOS won't
  let any audio start without a user gesture, and that gesture is your first
  tap. Tap once against your palm before starting a session.
- **`audioSession` needs iOS 16.4+.** On older versions the ringer switch
  silences the app.
