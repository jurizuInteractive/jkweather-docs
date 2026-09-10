# JKWeather — FAQ

Answers to the questions that come up most often, and to the traps that are easy
to fall into. If your question is not here, open an
[issue](https://github.com/jurizuInteractive/jkweather-docs/issues) — the answer
gets added to this page.

---

### The clock runs far faster (or slower) than I set

`DayLengthRealMinutes` in `DefaultGame.ini`, section
`[/Script/JKWeather.WeatherSubsystem]`. Note it is **real minutes per in-game
day**, not the reverse: `240` means a full day takes four real hours.

Almost everything else scales off that clock — weather event duration, lightning
cadence, wetness and snow integration — so a value far from the 5-minute default
changes the feel of the whole simulation. That is intended.

### A weather event lasts much longer than the seconds I asked for

`Weather.Event 4 180` does not mean 180 real seconds. The number is **seconds on
the reference clock** — a 5-minute day at speed 1 — which is 14.4 *game hours*.
At `DayLengthRealMinutes=240` and `Weather.Speed 1`, those 14.4 game hours are
about 2.4 real hours.

This is deliberate: before 1.6.0 the same call produced a duration that varied by
a factor of 288 depending on the length of the day. Now it always occupies the
same slice of a day.

### Same `WeatherSeed`, different weather

Two things to check first: both machines must be on **1.6.0 or later** (before
that a latch in the pressure walk burned the random stream once per frame, so
frame rate changed the outcome), and the seed has to be set before the first
tick — it is read at `Initialize`.

Note that only the **authoritative** side rolls the seeded stream. On a network
client, hail timing is drawn from a separate, non-seeded stream on purpose, so
hail can still be seen locally without shifting the simulation stream.

### It rains but I see no particles / hear no audio

The renderer needs the assets assigned. Check the log for `[WeatherRenderer]`
lines saying a system or sound was not found — the plugin degrades quietly and
tells you exactly what is missing. `Scripts/importar_audio_clima.py` imports the
audio; the material and VFX scripts under `Scripts/` regenerate the rest.

Also check the under-roof sensor: if the camera is under geometry that blocks
`ECC_Visibility`, precipitation VFX and audio are suppressed on purpose. A glass
skylight and a dense tree canopy both count as a roof.

### Daylight blows out to pure white

Enable **Project Settings → Rendering → Default Settings → "Extend default
luminance range in Auto Exposure settings"**, then restart the editor. The plugin
emits real lux values; without the extended range the auto-exposure clips. The
renderer warns about this in the log on startup.

### The console commands do not exist in my packaged build

Correct, and intended: all 34 `Weather.*` commands live under
`#if !UE_BUILD_SHIPPING`. Use a Development build to drive them.

### On a client, the storm HUD reads zero / hail never falls

Fixed in 1.6.0. Before that, electrical activity and the time-since-strike
counter only advanced on the authority, so any HUD bound to them read a flat
zero on clients — and since hail is gated on electrical activity, a client could
never render it.

Still true by design: derived values that are **not** replicated (precipitation
intensity and type among them) are recomputed per client, so they can differ by a
moment between machines. Authoritative gameplay must read them **server-side**
and replicate the result — see `Docs/multiplayer_guide.md`.

### Weather settings revert after loading a save

Fixed in 1.6.0 (save format v8). Six runtime settings — maritime factor, yearly
rain scale, ENSO regional sign, the manual anomaly, longitude and timezone — did
not travel in the save state, so applying a climate preset and reloading left a
half-applied climate.

Saves older than v8 deliberately do **not** overwrite those values on load, so an
old save cannot silently wipe a preset your project applied.

### The moon is in the wrong half of the sky

Fixed in 1.6.0. The lunar transit was computed with an inverted sign; outside a
full moon the error reached twelve hours. The default phase offset of 12 masked
it, because a full moon is symmetric about ±12 — so it only showed once the
synodic cycle moved off the default.
