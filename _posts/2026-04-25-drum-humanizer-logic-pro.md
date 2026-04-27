---
layout: post
title: Humanizing MIDI Drums in Logic Pro with Scripter
tags:
 - Music
 - Logic Pro
 - MIDI
 - JavaScript
 - Scripter
---

MIDI drums can sound stiff and mechanical, and doing it all manually is tedious and time-consuming. So I wrote a Logic Pro Scripter script (with the help of Claude AI) to handle the humanization automatically, and I'm sharing it here in full detail. 

Disclaimer : this is an experiment, I don't have enough perspective yet to tell if it does the job musically speaking, but I wanted to at least share the technical aspect of it. I occasionally work on **[Dabula](https://dabula-music.bandcamp.com/album/dabula)** — my post-rock/doom/stoner project. I will definitely test this technique for the next album.

When you program a drum beat in a DAW, the result is often too perfect — every hit lands exactly on the grid, every velocity is the same. Real drummers don't play like that. They accent certain beats, ghost others, rush slightly here, drag slightly there. This script replicates all of this behavior in real time, non-destructively, with a handful of adjustable parameters.

![Logic Pro Scripter — drum humanizer parameters (UI in French)](/images/logic-sc.png)

---

## What the Script Does

The script applies four layers of human-like variation to any MIDI drum track:

1. **Random velocity variation** — no two hits feel exactly the same
2. **Random timing offsets** — notes land slightly before or after the grid
3. **Beat-aware velocity boost** — downbeats hit harder than upbeats
4. **Hi-hat and ride alternation** — every other stroke is softer, mimicking a real player's wrist motion

Let's go through each section of the code and understand why it's built the way it is.

---

## MIDI Note Map

```javascript
var HIHAT_CLOSED  = 42;
var HIHAT_OPEN    = 46;
var HIHAT_PEDAL   = 44;
var RIDE_BOW      = 51;
var RIDE_BELL     = 53;
var RIDE_EDGE     = 59;
```

MIDI uses numbers (0–127) to identify pitches. Drum kits follow the General MIDI (GM) standard, where each drum sound is mapped to a specific number. By naming these constants at the top of the script, we avoid using "magic numbers" buried in the logic below — if your drum kit uses a different mapping, you only need to change it in one place.

> ⚠️ If your kit is not GM-compliant, open the Piano Roll, click a hi-hat note, and check its pitch number. Update these variables accordingly.

---

## Alternation Counters

```javascript
var hihatCounter = 0;
var rideCounter  = 0;
```

To alternate between a loud and a soft stroke, the script needs to remember which stroke it last played. These two counters track that independently for the hi-hat and the ride. They are global variables, meaning they persist across every note event during playback.

They are reset to zero in the `Reset()` function (covered later), so the alternation always starts on the strong beat when you press play.

---

## Plugin Parameters

```javascript
var PluginParameters = [
  { name: "Velocity Random", type: "lin", minValue: 0, maxValue: 30, defaultValue: 12, unit: "±" },
  { name: "Timing Random",   type: "lin", minValue: 0, maxValue: 30, defaultValue: 10, unit: "ticks" },
  { name: "Strong Beat Boost",  type: "lin", minValue: 0, maxValue: 30, defaultValue: 15, unit: "vel" },
  { name: "Weak Beat Cut", type: "lin", minValue: 0, maxValue: 15, defaultValue: 5,  unit: "vel" },
  { name: "Hihat Strong Accent",  type: "lin", minValue: 60, maxValue: 127, defaultValue: 95, unit: "vel" },
  { name: "Hihat Weak Accent", type: "lin", minValue: 20, maxValue: 90, defaultValue: 60, unit: "vel" },
  { name: "Ride Strong Accent",   type: "lin", minValue: 60, maxValue: 127, defaultValue: 90, unit: "vel" },
  { name: "Ride Weak Accent", type: "lin", minValue: 20, maxValue: 90, defaultValue: 55, unit: "vel" }
];
```

Logic's Scripter reads this array and automatically generates a graphical interface with sliders. Each parameter becomes a knob or slider that can be adjusted in real time during playback — no need to re-run the script.

Here is what each parameter controls:

| Parameter | Description | Recommended Range |
|---|---|---|
| `Velocity Random` | Random ± variation applied to all non-hihat/ride notes | 8–15 |
| `Timing Random` | Maximum timing offset in milliseconds | 5–15 |
| `Strong Beat Boost` | Velocity added on beats 1 and 3 | 12–20 |
| `Weak Beat Cut` | Velocity subtracted on beats 2 and 4 | 3–8 |
| `Hihat Strong Accent` | Target velocity for strong hi-hat strokes | 85–100 |
| `Hihat Weak Accent` | Target velocity for weak hi-hat strokes | 50–70 |
| `Ride Strong Accent` | Target velocity for strong ride strokes | 80–95 |
| `Ride Weak Accent` | Target velocity for weak ride strokes | 45–65 |

---

## Utility Functions

### randomize()

```javascript
function randomize(range) {
  return Math.round((Math.random() * 2 - 1) * range);
}
```

`Math.random()` produces a number between 0 and 1. Multiplying by 2 and subtracting 1 shifts the range to −1 → +1, giving us a positive or negative random value. Multiplying by `range` scales it to the desired spread. The result is a random integer anywhere between `-range` and `+range`.

This is the core of all humanization in the script — randomness that is bounded and controllable.

### clampVelocity()

```javascript
function clampVelocity(v) {
  return Math.max(1, Math.min(127, Math.round(v)));
}
```

MIDI velocity must stay between 1 and 127. Without clamping, adding a boost to an already-loud note could produce an invalid value (e.g., 140), causing unpredictable behavior in the instrument plugin. This function ensures the value is always safe, rounded to an integer, and never silent (0 would mute the note).

---

## Beat-Aware Velocity Boost

```javascript
function getBeatBoost(beatPos) {
  var ticksPerBeat = 960;
  var ticksPerBar  = ticksPerBeat * 4;
  var posInBar     = (beatPos * 960) % ticksPerBar;

  if (posInBar < ticksPerBeat)          return GetParameter("Strong Beat Boost");
  else if (posInBar < ticksPerBeat * 2) return -GetParameter("Weak Beat Cut");
  else if (posInBar < ticksPerBeat * 3) return Math.round(GetParameter("Strong Beat Boost") * 0.6);
  else                                  return -GetParameter("Weak Beat Cut");
}
```

In a 4/4 bar, human drummers naturally accent beats 1 and 3 (the downbeats) and play lighter on beats 2 and 4. This function reproduces that natural hierarchy.

Logic provides `event.beatPos` as a floating-point beat position. Multiplying by 960 converts it to ticks (Logic's internal resolution). Using the modulo operator (`%`) gives us the position within the current bar, which we then map to one of four zones:

- **Beat 1** → full boost (e.g., +15)
- **Beat 2** → slight reduction (e.g., −5)
- **Beat 3** → partial boost (60% of beat 1, e.g., +9) — the secondary accent
- **Beat 4** → slight reduction (e.g., −5)

This creates a natural push-and-pull feel without any manual editing.

---

## Hi-Hat Processing

```javascript
function processHihat(event) {
  var isAccent = (hihatCounter % 2 === 0);
  hihatCounter++;

  var baseVel   = isAccent
    ? GetParameter("Hihat Strong Accent")
    : GetParameter("Hihat Weak Accent");

  var velRandom = randomize(GetParameter("Velocity Random") * 0.5);
  event.velocity = clampVelocity(baseVel + velRandom);
  return event;
}
```

A real drummer playing eighth notes on the hi-hat doesn't hit every stroke with the same force. The wrist naturally accents the downstroke and ghosts the upstroke. This function replicates that motion.

The modulo check (`hihatCounter % 2 === 0`) returns `true` on even counts (0, 2, 4...) and `false` on odd ones, creating a perfect alternation. The random variation is halved (`* 0.5`) compared to the rest of the kit because hi-hat dynamics are more controlled than, say, a snare hit.

The same logic applies to `processRide()`, which is identical but uses its own counter and parameters.

---

## Main Event Handler

```javascript
function HandleMIDI(event) {
  if (event instanceof Note) {
    var pitch = event.pitch;
    var timingOffset = Math.abs(randomize(GetParameter("Timing Random")));

    var isHihat = (pitch === HIHAT_CLOSED || pitch === HIHAT_OPEN || pitch === HIHAT_PEDAL);
    var isRide  = (pitch === RIDE_BOW || pitch === RIDE_BELL || pitch === RIDE_EDGE);

    if (isHihat) {
      event = processHihat(event);
    } else if (isRide) {
      event = processRide(event);
    } else {
      var velRandom  = randomize(GetParameter("Velocity Random"));
      var beatBoost  = getBeatBoost(event.beatPos);
      event.velocity = clampVelocity(event.velocity + velRandom + beatBoost);
    }

    if (timingOffset > 0) {
      event.sendAfterMilliseconds(timingOffset);
    } else {
      event.send();
    }

  } else {
    event.send();
  }
}
```

`HandleMIDI` is the entry point called by Logic for every single MIDI event on the track. The script first checks if the event is a `Note` — other events like pitch bend or sustain pedal are passed through untouched.

For notes, the routing logic is simple:
- **Hi-hat pitches** → dedicated alternation function
- **Ride pitches** → dedicated alternation function
- **Everything else** (kick, snare, toms) → global randomization + beat boost

The timing offset uses `Math.abs()` to ensure it is always positive — `sendAfterMilliseconds` only accepts positive values. This introduces a slight forward delay (never early), which mimics the natural drag of a human player reacting to the beat.

---

## Transport Reset

```javascript
function Reset() {
  hihatCounter = 0;
  rideCounter  = 0;
}
```

Logic calls `Reset()` whenever the transport is stopped and restarted from the beginning. Without this, the alternation counters would keep their previous values, potentially starting on a weak stroke instead of a strong one every time you press play. Resetting ensures consistent, predictable behavior across multiple playback sessions.

---

## The Complete Script

```javascript
// ============================================
// DRUM HUMANIZER - Logic Pro Scripter
// Full humanization + Hi-hat & Ride alternation
// ============================================

var HIHAT_CLOSED  = 42;
var HIHAT_OPEN    = 46;
var HIHAT_PEDAL   = 44;
var RIDE_BOW      = 51;
var RIDE_BELL     = 53;
var RIDE_EDGE     = 59;

var hihatCounter = 0;
var rideCounter  = 0;

var PluginParameters = [
  { name: "Velocity Random",    type: "lin", minValue: 0,  maxValue: 30,  numberOfSteps: 30, defaultValue: 12, unit: "±" },
  { name: "Timing Random",      type: "lin", minValue: 0,  maxValue: 30,  numberOfSteps: 30, defaultValue: 10, unit: "ticks" },
  { name: "Strong Beat Boost",  type: "lin", minValue: 0,  maxValue: 30,  numberOfSteps: 30, defaultValue: 15, unit: "vel" },
  { name: "Weak Beat Cut",type: "lin", minValue: 0,  maxValue: 15,  numberOfSteps: 15, defaultValue: 5,  unit: "vel" },
  { name: "Hihat Strong Accent",  type: "lin", minValue: 60, maxValue: 127, numberOfSteps: 67, defaultValue: 95, unit: "vel" },
  { name: "Hihat Weak Accent",type: "lin", minValue: 20, maxValue: 90,  numberOfSteps: 70, defaultValue: 60, unit: "vel" },
  { name: "Ride Strong Accent",   type: "lin", minValue: 60, maxValue: 127, numberOfSteps: 67, defaultValue: 90, unit: "vel" },
  { name: "Ride Weak Accent", type: "lin", minValue: 20, maxValue: 90,  numberOfSteps: 70, defaultValue: 55, unit: "vel" }
];

function randomize(range) {
  return Math.round((Math.random() * 2 - 1) * range);
}

function clampVelocity(v) {
  return Math.max(1, Math.min(127, Math.round(v)));
}

function getBeatBoost(beatPos) {
  var ticksPerBeat = 960;
  var ticksPerBar  = ticksPerBeat * 4;
  var posInBar     = (beatPos * 960) % ticksPerBar;

  if (posInBar < ticksPerBeat)          return GetParameter("Strong Beat Boost");
  else if (posInBar < ticksPerBeat * 2) return -GetParameter("Weak Beat Cut");
  else if (posInBar < ticksPerBeat * 3) return Math.round(GetParameter("Strong Beat Boost") * 0.6);
  else                                  return -GetParameter("Weak Beat Cut");
}

function processHihat(event) {
  var isAccent = (hihatCounter % 2 === 0);
  hihatCounter++;
  var baseVel  = isAccent ? GetParameter("Hihat Strong Accent") : GetParameter("Hihat Weak Accent");
  event.velocity = clampVelocity(baseVel + randomize(GetParameter("Velocity Random") * 0.5));
  return event;
}

function processRide(event) {
  var isAccent = (rideCounter % 2 === 0);
  rideCounter++;
  var baseVel  = isAccent ? GetParameter("Ride Strong Accent") : GetParameter("Ride Weak Accent");
  event.velocity = clampVelocity(baseVel + randomize(GetParameter("Velocity Random") * 0.5));
  return event;
}

function HandleMIDI(event) {
  if (event instanceof Note) {
    var pitch        = event.pitch;
    var timingOffset = Math.abs(randomize(GetParameter("Timing Random")));
    var isHihat = (pitch === HIHAT_CLOSED || pitch === HIHAT_OPEN || pitch === HIHAT_PEDAL);
    var isRide  = (pitch === RIDE_BOW || pitch === RIDE_BELL || pitch === RIDE_EDGE);

    if (isHihat) {
      event = processHihat(event);
    } else if (isRide) {
      event = processRide(event);
    } else {
      event.velocity = clampVelocity(
        event.velocity + randomize(GetParameter("Velocity Random")) + getBeatBoost(event.beatPos)
      );
    }

    if (timingOffset > 0) {
      event.sendAfterMilliseconds(timingOffset);
    } else {
      event.send();
    }

  } else {
    event.send();
  }
}

function Reset() {
  hihatCounter = 0;
  rideCounter  = 0;
}
```
