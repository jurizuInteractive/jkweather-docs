# Guide: Niagara precipitation systems (JKWeather)

The renderer manages two persistent `UNiagaraComponent`s attached to the camera
(rain and snow). It activates them, moves them, turns them off under cover and
pushes their parameters every tick. The Niagara systems are yours: any system that
respects the contract below works. Store them in the plugin Content:

    Plugins/JKWeather/Content/VFX/NS_Rain.uasset
    Plugins/JKWeather/Content/VFX/NS_Snow.uasset

(or assign them by hand on the actor: `Weather|Precipitation → SistemaLluvia/SistemaNieve`).

## User Parameters contract (written by the C++ every tick)

| Parameter     | Type   | Range  | Meaning                                              |
|---------------|--------|--------|------------------------------------------------------|
| Intensity     | float  | 0-1    | Effective intensity (ceiling and transitions included)|
| SpawnRate     | float  | 0-3000 | Particles/s already scaled (Intensity × TasaMaxLluvia)|
| WindVelocity  | Vector | cm/s   | World wind, GUSTS included                            |
| Gust          | float  | 0-1    | Raw gust factor (for audio/extra sway)               |

Create them in the **Parameters → User Parameters** tab of each system with those
exact names (in English). The system must use `User.SpawnRate` as its rate;
everything else is optional but recommended.

## NS_Rain (5 minutes from the Fountain template)

1. Content Browser → right click → FX → **Niagara System** → *New system from
   selected emitters* → **Fountain** → name it `NS_Rain`.
2. Create the 4 User Parameters from the table.
3. **Emitter Properties**: Sim Target = `GPUCompute Sim`; enable **Fixed Bounds**
   (≈ 4000 on each axis). Emitter State → Loop Behavior = `Infinite`.
4. **Spawn Rate**: link SpawnRate = `User.SpawnRate` (arrow ▾ → Link Inputs).
5. **Initialize Particle**:
   - Lifetime: Uniform 0.9 – 1.3
   - Sprite Size: Uniform X 4-7, **Y 40-90** (elongated drop)
   - Color: A = Dynamic Input *Multiply Float* → `User.Intensity` × 0.35
6. **Shape Location** (replaces the cone): Shape = Box/Plane, Box Size
   (3500, 3500, 100).
7. Delete the *Add Velocity in Cone* and add two **Add Velocity**:
   - (0, 0, -2400) with Uniform down to -2800 → fall
   - `User.WindVelocity` × 0.9 → drift (the wind tilts the rain by itself!)
8. Particle Update: **Drag** = 0.15; Gravity Force Z = -1500.
9. **Sprite Renderer**: Alignment = `Velocity Aligned` (the drop stretches in its
   fall direction; with wind the curtain really tilts). Material: a simple Unlit
   Translucent one (Emissive white × 0.6, soft radial mask) or the
   DefaultSpriteMaterial to start.

## NS_Snow (variation of the same skeleton)

Same as above, with these changes: Sprite Size Uniform 2-5 (X=Y), Lifetime 4-7 s,
fall (0, 0, -90 to -140), drift `User.WindVelocity` × 0.6, **Curl Noise Force**
(Strength ≈ 60, Frequency ≈ 0.6) for the flakes' dance, Drag 1.2, Alignment =
`Camera Facing`. Alpha = `User.Intensity` × 0.8.

## What the C++ does for you (don't duplicate it in the system)

- **Camera tracking** with drift compensation: the emitter shifts against the wind
  (`SegundosCompensacionDeriva`) so the curtain falls centered after drifting.
- **Under cover**: an upward trace every 0.15 s → `Intensity` fades to 0 under
  cover and returns on exit (`bAtenuarBajoTecho`, `VelocidadTransicionTecho`).
- **Transitions**: drizzle→rain→storm→nothing blend in ~1 s; on stop `Deactivate()`
  is used, so the live drops finish their fall (no cuts).
- **Types**: Drizzle/Rain/Storm feed NS_Rain with their intensity; Snow feeds
  NS_Snow. Snowfall now only happens below 0 °C (the climate decides it).

## Test without waiting for the weather

    Weather.VFXTest 1    ← rain forced to 100%
    Weather.VFXTest 2    ← snow forced to 100%
    Weather.VFXTest 0    ← back to the simulation
    Weather.Atmosphere    ← full report (mm/h and cm/h rates included)

Tunables on the actor (`Weather|Precipitation`): `TasaMaxLluvia` (3000),
`TasaMaxNieve` (1200), `AlturaEmisorSobreCamara` (900), `DerivaVientoVFX`,
`SegundosCompensacionDeriva` and the `bVFXPrecipitacion` switch.

## Cooking note (the same trap as the MPC and M_Moon)

A system loaded only by path can end up outside the buyer's package. Assigning
`SistemaLluvia`/`SistemaNieve` on the demo map's actor creates the hard reference
that guarantees its inclusion.

## Hail, sleet and freezing rain (types 5 / 6 / 7)

The subsystem already decides these on its own (sleet in the 0-2.5 C band,
freezing rain under ~1 C, hail bursts of a few minutes inside storms). The
renderer now maps them to particles and sound:

| Type | Particles | Sound |
|---|---|---|
| 5 Sleet | half rain + half snow | rain loop at half volume |
| 6 Hail | `NS_Hail` + 40% rain underneath | `S_HailLoop` + rain underneath |
| 7 Freezing rain | plain rain (the ice forms on contact: `GroundFrost` rises) | rain loop |

**`NS_Hail` is optional.** Duplicate `NS_Rain`, make the particles fewer,
larger, faster and whitish, keep the same User parameters (`Intensity`,
`SpawnRate`, `WindVelocity`, `Gust`) and save it as `/JKWeather/VFX/NS_Hail`.
The renderer picks it up automatically (or assign `SistemaGranizo` in Details).
Without it there is no warning: hail falls back to the rain system at full
blast, and the hail *sound* still plays, so the weather reads correctly anyway.

**Altitude swap (thermal lapse rate).** VFX and audio share one calculation
(`CalcularObjetivosPrecipitacion`) that samples `GetTemperatureAt()` at the
camera: rain arrives as silent snow where the local air is below 0 C, and snow
melts into audible rain above +2 C. Climb a mountain during a shower and watch
it whiten — and hush.

**Testing:** `Weather.VFXTest 3` forces hail, `4` forces sleet, `0` returns to
simulation. Natural hail: `Weather.Event storm` and wait for a burst
(`HailChancePerMinute`, ~2.5 min each).

**Audio.** `S_RainLoop` / `S_HailLoop` are imported by
`Scripts/importar_audio_clima.py` from `Resources/SourceArt/Audio/lluvia_loop.wav`
and `granizo_loop.wav` (procedural placeholders — drop your real WAVs over them
and re-run the script with `REEMPLAZAR = True`). Volume follows intensity; under
a roof the loop is attenuated and low-passed (`LluviaFiltroTechoHz`, 900 Hz by
default): the classic drumming-on-the-roof. Tunables in `Weather|Precipitation`:
`VolumenLluvia`, `VolumenGranizo`, `TasaMaxGranizo`.
