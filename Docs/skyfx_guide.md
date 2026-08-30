# SkyFX guide — optical phenomena and weather extremes

This layer adds the "rare and beautiful" moments on top of the base weather:
rainbows, halos, shooting stars, eclipses, dust haze (calima), blizzards, valley
fog, and yearly precipitation scaling. Everything is driven by the same
astronomically-correct clock and the weather data the subsystem already
computes — no extra bookkeeping on your side.

All of it lives on `AJKWeatherRenderer` under the **Weather|SkyFX** category,
and reads from `UWeatherSubsystem`. Features degrade gracefully: if an optical
material is missing, that effect simply stays off (no crash, no error spam).

---

## 1. One-time setup: the optical materials

The rainbow, halo and shooting star are drawn on camera-facing billboards that
the renderer spawns automatically. They need three unlit, additive materials in
`/JKWeather/SkyFX`. Generate them once from the editor:

```
Output Log > (dropdown) Cmd: Python
py "C:/path/to/plugin/Scripts/crear_materiales_cielo.py"
```

This creates `M_Rainbow`, `M_Halo22` and `M_ShootingStar`. The script is
idempotent: it skips any material that already exists, so your tweaks survive
re-runs (delete a material in the Content Browser to regenerate it). You do **not**
assign these anywhere — the renderer loads them by path at BeginPlay.

Material parameters the renderer drives each frame:

| Parameter | Materials | Meaning |
|---|---|---|
| `Intensidad` | all | effect brightness (0 = off) |
| `Progreso` | shooting star | 0..1, position of the streak head |
| `RadioGrados` | rainbow, halo | present for reference; the ring is baked to match the billboard size the renderer uses |

Dust haze, blizzards, valley fog and eclipses need **no** new materials — they
modulate the atmosphere, height fog and the existing Sun/Moon.

---

## 2. The phenomena

### Rainbow
Appears in the **antisolar** direction (opposite the Sun) as the classic
42-degree bow. It requires a **low Sun** (altitude between 2 and 42 degrees), a
recently **wet surface** (`GetSurfaceWetness > 0.35`, i.e. just after rain) and
a reasonably **clear, bright Sun** (fades with cloud cover). Intensity eases in
and out smoothly. Key settings: `bArcoiris`, `ArcoirisIntensidadMax`.

### 22-degree halo
A thin bright ring around the dominant luminary — the Sun by day, the Moon by
night. It needs **thin cirrus** (moderate cloud cover, roughly 0.15–0.55) and
**high humidity**. Key settings: `bHalo22`, `HaloIntensidadMax`.

### Shooting stars
Brief streaks on **clear nights** (cloud cover below 0.25, well after dusk).
Frequency is `FugacesPorMinuto` (average per minute). Each streak lasts a
fraction of a second and travels across a random patch of sky. Key settings:
`bEstrellasFugaces`, `FugacesPorMinuto`.

### Dust haze — *calima* (`Weather.Event 5`)
A Saharan-style dust event. Thermodynamically it warms the air (about +3 C),
drops humidity and biases toward anticyclonic pressure. Visually it tints the
fog ochre (`CalimaTinte`), adds haze density (`CalimaNieblaExtra`) and raises the
atmosphere's Mie scattering (`CalimaMieFactor`) so the sky turns
milky-ochre and the Sun is veiled. Trigger it like any other event:
`Weather.Event 5`.

### Blizzard — *ventisca* (emergent)
There is **no blizzard command**: it emerges naturally when it is **snowing**
and the **wind is strong** (between `VentiscaVientoMinKmh` and
`VentiscaVientoMaxKmh` on the subsystem, 25–55 km/h by default). The snow VFX
wind is boosted (`VentiscaVientoVFXExtra`, flakes go near-horizontal) and extra
fog reduces visibility (`VentiscaNieblaExtra`). To see it: set a cold, snowy
state and raise the wind (e.g. via a storm or by pushing the pressure gradient).

### Valley fog
At **dawn** (roughly 04:00–09:00), when a dense fog bank is present, the fog's
height falloff is raised (`ValleFalloffFactor`) so the bank hugs the ground and
pools in low areas, then lifts as the morning warms. Key setting: `bNieblaDeValle`.

### Eclipses
Real solar and lunar eclipses, gated on the **already-simulated lunar nodes**.
A **solar** eclipse needs a new Moon near a node (the Moon crosses in front of
the Sun): the Sun's light collapses to a sliver. A **lunar** eclipse needs a
full Moon near a node (Earth's shadow falls on the Moon): the disc darkens and
the Moon light turns blood red. Thresholds are generous (this is a game, not an
ephemeris), so alignments happen a couple of times per "eclipse season" as the
node regresses over its 18.6-year cycle. Key setting: `bEclipses`. You can force
one for a screenshot with `Weather.SkyTest 4` / `5`.

---

## 3. Yearly precipitation and the maritime factor

- **Drought / wet year** — `Weather.Drought <scale>` multiplies all
  precipitation by a factor: `0.6` for a dry year, `1` normal, `1.4` for a wet
  year. Blueprint: `SetPrecipitationYearScale`.
- **Maritime factor** — `SetMaritimeFactor01(0..1)` softens the daily
  temperature swing (oceans buffer extremes) and raises baseline humidity, for
  coastal vs. continental feel. There is no console command; set it from
  Blueprint or C++ per location.
- **Pressure to hail** — inside storms, low barometric pressure increases hail
  probability (deeper lows carry more vigorous convection). This is automatic
  and scales with `Weather.Pressure`.

---

## 4. Commands (non-Shipping builds)

| Command | Effect |
|---|---|
| `Weather.Event 5 [sec]` | Force dust haze (calima) |
| `Weather.Drought <0.6-1.4>` | Scale the year's precipitation |
| `Weather.SkyTest 0` | Clear the optical override |
| `Weather.SkyTest 1` | Force the rainbow |
| `Weather.SkyTest 2` | Force the halo |
| `Weather.SkyTest 3` | Fire shooting stars |
| `Weather.SkyTest 4` | Force a solar eclipse |
| `Weather.SkyTest 5` | Force a lunar eclipse |

`Weather.SkyTest` only overrides the *visuals* for capture; it does not touch the
weather simulation. Set it back to `0` when you are done.

---

## 5. Tuning notes

- The rainbow and halo ring radii are baked into `M_Rainbow` / `M_Halo22` to
  match the billboard size the renderer uses (50-degree and 26-degree half-angles).
  If you resize the rings in the material, keep both in sync.
- Dust haze and valley fog capture the atmosphere's base Mie scattering and the
  fog's base height falloff once at runtime, then modulate around those bases, so
  they respect whatever you author on the components.
- Everything is off-by-default-safe: with `bEclipses`, `bArcoiris`, etc. disabled,
  or with the optical materials absent, the base weather is unchanged.


---

## 6. Thermals and convection (level 2)

Convection ties the dry/moist adiabatic idea to something visible: rising warm
air. The subsystem computes a **convective index** (`GetConvectiveIndex01`, 0-1)
and a **thermal top** (`GetThermalTopZ`, the cloud base / LCL where thermals cap
and cumulus form). The index rises with **surface heating** - strong Sun, clear
sky, light wind, and unsaturated (dry) air - and peaks in the early afternoon,
when the ground has banked the most heat. It is also nudged by **conditional
instability**: the environmental lapse rate compared with an estimated moist
adiabatic. It is read-only; it never feeds back into the simulation.

It reaches the world through three optional channels, each of which degrades to
"off" if its asset is missing:

1. **MPC channel `ThermalActivity`** (0-1), written every frame. Any material can
   read it - heat shimmer on your own shaders, extra foliage jitter, dust on hot
   ground. Run `Scripts/crear_material_termicas.py` once to add the channel.
2. **Rising-particle Niagara** (`NS_Thermal`, optional) at `/JKWeather/VFX/NS_Thermal`.
   The renderer spawns it as an updraft following the camera, scaling spawn rate
   by the index and passing `Intensity`, `SpawnRate`, `TopZ` and `WindVelocity`.
   Author it by hand (like the precipitation systems); the C++ side is ready.
3. **Heat-shimmer post-process** (`PP_HeatShimmer`, optional), built by
   `crear_material_termicas.py`. It reads `ThermalActivity` and ripples the scene,
   stronger toward the bottom of the frame; the renderer adds it as a global
   blendable automatically. This graph is experimental across UE versions - if it
   fails to build, the MPC channel is still there for your own effect.

**Soaring birds / glider lift** are not spawned by the plugin (that is content you
own), but the index and thermal top are exposed to Blueprint so you can drive
them yourself.

### Command

`Weather.Thermals <0-1>` forces the convective index for authoring; call it with
no argument to release back to the simulation.

### Settings

Subsystem (**Weather|Convection**): `ConvectionWindCutoffKmh`, `ConvectionLCLFactorM`.
Renderer (**Weather|SkyFX**): `bTermicas`, `TasaMaxTermica`.


---

## 7. Earthshine and aurora (level 2)

### Earthshine (*luz cenicienta*)
The "old Moon in the new Moon's arms": near new phase, sunlight bouncing off
Earth faintly lights the Moon's dark side. The renderer computes its strength
from the lunar elongation (strong near new, zero toward full) and writes it to
the Moon material as the scalar **`Earthshine`**. Settings (**Weather|SkyFX**):
`bLuzCenicienta`, `LuzCenicientaMax` (subtle, ~0.12).

Note: the visible fill needs the Moon material to read the `Earthshine`
parameter and add it to the unlit hemisphere. The bundled `M_Moon` may not do
this yet; if you regenerate or edit it, add an `Earthshine` scalar and a faint
blue-grey emissive on the side the terminator leaves dark. The C++ hook is
already there — it is a no-op until the material uses it.

### Aurora (borealis / australis)
A poleward glow that only makes sense at **high latitudes**. The subsystem
computes a **potential** (`GetAuroraActivity01`, 0-1) gated by latitude (rising
above 55 degrees), boosted near the **equinoxes**, with a slow pseudo-random
drift (there is no geomagnetic model). The renderer shows it only **at night**
with a **clear sky**, on a large billboard toward the pole (north in the northern
hemisphere, south in the southern), using the optional material **`M_Aurora`**
(built by `crear_materiales_cielo.py`: animated green-to-magenta curtains). If
`M_Aurora` is absent, the aurora stays off. It is also published to the MPC as
`AuroraActivity`. Settings (**Weather|SkyFX**): `bAurora`, `AuroraIntensidadMax`.

### Commands

`Weather.Aurora <0-1>` forces the aurora potential for authoring (still only
visible at night with a clear sky); no argument releases it. Earthshine has no
command - it follows the Moon phase.
