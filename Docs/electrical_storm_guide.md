# Guide: electrical storm and audio (v1.2)

The storm generates **electrical strikes** with the same two-layer split as the
rest of the plugin: `UWeatherSubsystem` produces the **data** for each strike and
`AJKWeatherRenderer` turns it into **light and sound**. None of this is saved:
the scheduler re-arms itself on load.

## How a strike is generated

While there is a storm (Storm event and/or storm-type precipitation), the
subsystem maintains a **cell** with electrical activity 0-1:

- The cell's **maturity** is the storm intensity (with its fade-in and fade-out).
  The higher the maturity, the more strikes per minute.
- **Pressure rules**: a deep low fires more (up to ~+90% activity at 975 hPa).
  Direct coupling with the v1.1 atmosphere layer.
- The cell is born on a random bearing and **drifts with the wind**.
- At the peak, a strike falls every ~3-10 s; at the edges, far fewer.

### Intra-cloud / cloud-to-ground variability

As in real cells, most strikes are **intra-cloud flashes** (the sky lights up from
within, no impact) and the share of **cloud-to-ground strikes** grows with
maturity:

| Cell maturity | Cloud-to-ground |
|---|---|
| Edges (entering/leaving) | ~12% |
| Storm peak | ~42% |

Cloud-to-ground strikes also fall somewhat closer and hit harder (power 0.55-1.0
versus 0.35-0.95 for the flashes).

## The data: FJKLightningStrike

Each strike is broadcast through the `OnLightningStrike` delegate with:

| Field | Meaning |
|---|---|
| `bCloudToGround` | `false` = intra-cloud flash, `true` = cloud-to-ground strike |
| `AzimuthDegrees` | Compass bearing 0-360 toward the strike (North = +Y, East = -X) |
| `DistanceMeters` | Distance to the observer (the cell approaches as it matures) |
| `Power` | 0-1: flash brightness and thunder volume |
| `ThunderDelaySeconds` | `DistanceMeters / 343`: how long the thunder takes to be heard |

Blueprint example: bind `OnLightningStrike` from the subsystem and, if
`bCloudToGround` and `DistanceMeters < 500`, set the grass on fire or spook the
cattle. World direction of the bearing: `(-sin(b), cos(b), 0)`.

## What the renderer does

- **Flash**: 1-3 pulses with exponential decay (leader + return strokes, like the
  real flicker). Applied at once to the SkyLight (contribution configurable per
  type) and to the MPC as `Cielo_Relampago` (0-1.5) so the cloud/sky material
  lights up from within on the flashes.
- **Cloud-to-ground strike**: additionally places a point light (300 m radius, no
  shadows) at the impact point: bearing + distance from the camera, with a
  vertical trace to the ground. Impacts farther than `DistanciaVisualMaxM` are
  pulled closer to that radius so their light can be seen.
- **Thunder**: scheduled with the data's delay (343 m/s). Closer than
  `DistanciaTruenoCercanoM` the near **crack** plays; beyond that, the far
  **rumble**. Volume by power and distance, random pitch so they don't sound
  cloned. 2D on purpose: thunder envelops, it isn't localized.
- **Wind howl**: a 2D loop whose volume and tone follow the gusted speed
  (`GetWindSpeedWithGust`). A ^1.5 curve: the breeze is barely audible, the gale
  howls and the gust "breathes" on top. It plays between `VientoAudibleDesdeKmh`
  and `VientoVolumenMaxKmh`, storm or not.

## Included audio and how to replace it

The plugin ships three decent procedural placeholders in
`Resources/SourceArt/Audio/` (44.1 kHz, 16-bit, stereo):

| WAV | Expected asset | Use |
|---|---|---|
| `trueno_cercano.wav` | `/JKWeather/Audio/S_ThunderNear` | Dry crack with body (7.5 s) |
| `trueno_lejano.wav` | `/JKWeather/Audio/S_ThunderFar` | Deep rolling rumble (11 s) |
| `viento_aullido_loop.wav` | `/JKWeather/Audio/S_WindHowl` | Seamless looping howl (10 s) |

**Import them once** by running `Scripts/importar_audio_clima.py` inside the editor
(same as the other scripts: Output Log in Python mode). The script is idempotent
and leaves the wind loop marked as looping.

To **replace** them you have two paths:

1. Assign your own `USoundBase` on the actor: Details > `Weather|Storm` (thunder)
   and `Weather|Wind` (loop). If the slot has something, the plugin auto-loads
   nothing. Any SoundWave or SoundCue works (a Cue with several random thunders is
   great).
2. Or replace the WAVs in `Resources/SourceArt/Audio`, delete the assets in
   `/JKWeather/Audio` and run the script again.

## Settings (Details and panel)

In `Weather|Storm`: `bRayosVisuales`, `bTruenoAudio`, sounds, `VolumenTrueno`,
`DistanciaTruenoCercanoM`, `DestelloEscala` (overall multiplier),
`DestelloSkyIC` / `DestelloSkyCG` (SkyLight contribution per type),
`DestelloLuzCG` (impact light) and `DistanciaVisualMaxM`.

In `Weather|Wind`: `SonidoVientoLoop`, `VolumenViento`, `VientoAudibleDesdeKmh`
and `VientoVolumenMaxKmh`.

The live panel (`Weather.Panel`) includes "Thunder volume", "Wind volume" and
"Lightning flash".

## Test it in 20 seconds

```
Weather.Event 4 300     <- force a 5-minute storm
Weather.Lightning            <- immediate strike, random type
Weather.Lightning cg         <- cloud-to-ground now (ic = intra-cloud)
Weather.Debug           <- HUD: line "Rayos: actividad ... | ultimo: ..."
```

`Weather.Event 0` ends the current event. The natural generator won't roll another
event on that same game day.
