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
42-degree bow. It needs a **low Sun** (altitude between 2 and 42 degrees) and a
shower that has **just stopped**: it shows for `VentanaArcoirisSegundos` (120
real seconds by default) after the precipitation ends, under a **broken sky**
(cover below about 80 %) with the Sun's disc clear. Intensity eases in and out
smoothly. Key settings: `bArcoiris`, `ArcoirisIntensidadMax`,
`VentanaArcoirisSegundos`.

### 22-degree halo
A ring 22 degrees around the dominant luminary — the Sun by day, the Moon by
night — with a sharp, slightly reddish inner edge, a white fade outward and no
added light inside, as a real ice-crystal halo looks.

- **When it appears.** A thin cirrus veil makes it. Whether there is a veil
  on a given day comes from the climate band's `CirrusFraction` (0.30 by
  default; deserts have more).
- **When it does not.** It fades under dense low cover and dies while it
  precipitates.
- **How bright it is.** It follows the light of its source:
  `HaloIntensidadMax` is the fraction of the sun's or moon's light that the
  veil sends into the ring (0.02 by default; 0.01 is faint and 0.05 vivid).
  So it is as visible against the noon sky as against the moonlit sky, and
  fainter under a crescent moon.

Key settings: `bHalo22`, `HaloIntensidadMax`. `Weather.SkyTest 2` forces it
for captures.

### Shooting stars
Brief streaks on **clear nights** (cloud cover below 0.25, well after dusk).
Frequency is `FugacesPorMinuto` (average per minute). Each streak lasts a
fraction of a second and travels across a random patch of sky. Key settings:
`bEstrellasFugaces`, `FugacesPorMinuto`.

### Dust haze — *calima* (`Weather.Event 5`)
A Saharan-style dust event. Thermodynamically it warms the air (about +3 C),
drops humidity and biases toward anticyclonic pressure. Visually it tints the
fog ochre (`CalimaTinte`), adds haze density (`CalimaNieblaExtra`) and adds aerosol
optical depth (`DustHazeOpticalDepth`, default 0.8 at full strength; Saharan
intrusions over Iberia measure 0.5-1.5) so the sky turns milky-ochre and the
Sun is veiled. It also drives the same volumetric-fog
extinction ramp as the Fog event (`ExtincionVolumetricaMax`, see the fog
guide's god-ray note below), so a dense dust storm shows visible sunbeams
too, not just a flat tint. Trigger it like any other event: `Weather.Event 5`.

An optional Niagara system (**Dust System**, empty = `/JKWeather/VFX/NS_Dust`)
adds airborne dust particles near eye level (`AlturaEmisorCalima`, much lower
than the precipitation emitter — dust hangs in suspension, it doesn't fall),
driven by `GetDustHaze01()` and the local wind. Same degrade-gracefully rule
as the thermal particles: without the asset, dust haze stays a sky/fog effect
with no ground-level particle. **The asset itself is not shipped yet** — it
needs the same by-hand authoring in the Niagara editor as the other
precipitation systems (see `precipitation_vfx_guide.md`); the C++ side is
ready and wired to the contract below.

Brief for authoring `NS_Dust`: soft, large, low-opacity tan/ochre sprites
(much bigger and dimmer than a raindrop or snowflake — this is haze, not
grit), spawned in a wide flattened volume near ground/eye level rather than
falling from above, drifting mostly horizontally with `WindVelocity`, very
low or no gravity. Same User Parameter contract as the other systems:
`Intensity` (0-1), `SpawnRate` (part/s), `WindVelocity` (cm/s, world space),
`Gust` (0-1).

### Ambient turbidity (every day, not just dust haze)
Dust haze above models the Saharan *extreme*; **Ambient Turbidity**
(`bTurbidezAmbiental`) models the ordinary aerosol that is always in the air,
continuously, without forcing an event. It is declared in **physical units**:
the aerosol optical depth at 550 nm (AOD), the quantity an AERONET sun
photometer measures.

- **From the climate band (default, `bAerosolFromClimateBand`):** every band
  declares a seasonal background AOD (`Winter/Spring/Summer/AutumnAerosolOpticalDepth`,
  same four anchors and cosine ease as the 850 hPa dew point). Factory values
  are AERONET medians next to each band's analogue site, from 0.05-0.1 in
  clean mid-latitudes to 0.3-0.5 in the Sahel and Arabia. The humidity
  reference is the band's normal humidity for the day (`GetDeclaredHumidityPct`),
  so on a typical day the sky carries exactly the declared AOD.
- Manual mode (no profile, or your own site data): `BackgroundAerosolOpticalDepth`
  (0.10, Caceres, Spain: AERONET 2005-2012, tau500 = 0.11 with Angstrom 1.38)
  and `AerosolReferenceHumidity` below.
- Humidity growth (Hanel): AOD x ((1-RH)/(1-RHref))^-gamma, with
  `AerosolHygroscopicGamma` (0.5) and `AerosolReferenceHumidity` (0.63). Pick
  the reference at which the growth factor equals its site mean, so the
  declared AOD is also the delivered mean (the growth is convex: the
  arithmetic mean humidity would bias it about +4%).
- `WindDustOpticalDepth` (0.05) is added at full wind.
- The total is turned into the sky atmosphere's exponential Mie layer:
  scattering = omega x AOD / H, absorption = (1 - omega) x AOD / H, with
  `AerosolSingleScatteringAlbedo` omega = 0.92 and H = the atmosphere's Mie
  scale height (1.2 km).

Golden hour needs no help: the sky atmosphere integrates the long slant path
at low Sun angles, which is what reddens and softens the Sun.
`TurbidezBoostHorizonte` is an artistic multiplier near the horizon and
defaults to **1 (physical)**. With turbidity off, the level's own atmosphere
is left as it was and only dust haze is added.

*Before 1.x (Oct 2026)* the turbidity was unitless Mie added on top
(`TurbidezPorHumedad` 4.0, `TurbidezPorViento` 1.5, `CalimaMieFactor` 8). At
60% humidity that meant an AOD of about 1.2: a white sky and no visible cast
shadows on an ordinary clear day. Those properties are retired (`_DEPRECATED`):
values saved in levels still load and are ignored, and Blueprints that set
them must move to the properties above.

It does **not** drive the volumetric-fog god-ray ramp (that stays opt-in via
the Fog event and dust haze only): turning volumetric fog on every ordinary
humid day has a real GPU cost that this feature does not spend on your
behalf.

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
