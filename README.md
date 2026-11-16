# JKWeather

Astronomically driven day/night and weather system for Unreal Engine 5.8.
One world subsystem simulates the atmosphere; one actor renders it. Everything
is coupled: barometric pressure leads clouds, wind and events; humidity, dew
point and surface temperature drive fog, frost, wetness and snow; latitude and
date drive the sun, the moon and the seasons.

**Version 1.7.0 — UE 5.8 — code plugin (C++ source included) — requires the
Niagara plugin (engine built-in).**

## What's included

- Solar model shared by clock and disc: spherical declination, apparent-horizon
  sunrise/sunset (refraction + solar radius), Bennett refraction on the visible
  disc. `GetSolarAltitudeDegrees / GetSolarAzimuthDegrees / GetSolarDeclinationDegrees`.
- Moon with real synodic phases (lit-disc billboard), sidereal starfield,
  natural solar and lunar **eclipses**.
- Full atmosphere: pressure with tendency, humidity + dew point, feels-like
  (wind chill + Steadman), UV, 16-wind rose with Perlin gusts, convection
  (Espy LCL) with thermals and heat shimmer, aurora at high latitudes.
- **Seven precipitation types** (drizzle, rain, storm, snow, sleet, freezing
  rain, hail) + blizzard, with camera-following GPU Niagara VFX, wind-drift
  compensation and an under-roof occlusion sensor for visuals *and* audio.
- Electrical storms: IC/CG strikes, distance-delayed thunder (crack or
  rumble), storm-cell drift, `OnLightningStrike` delegate.
- **Five weather events**: fog, heat wave, cold wave, storm, dust haze
  (calima) — natural generation plus console forcing.
- **Multiplayer out of the box**: server-authoritative replication (~2 Hz,
  delta-compressed; the full state is about 2 KB and travels whole only on
  join), smooth client extrapolation of the solar clock, lightning as an
  unreliable multicast (a lost strike never stalls the session), zero setup. Details and a test checklist in
  `Docs/multiplayer_guide.md`.
- Climate bands by latitude with smooth transitions, southern-hemisphere
  season flip, and per-band **rain frequency**: annual precipitation shifts
  the cloud-to-rain thresholds (deserts rain rarely *and* lightly; rainforests
  often *and* hard).
- **Altitude-aware snow**: two simulated altitude bands; it can snow and
  accumulate on the peaks while the valley gets rain. Query with
  `GetSnowDepthAtZ / GetSnowDepthAtLocation`; the snow line is published to
  materials as the `SnowLineZ` MPC channel.
- Surface layer: wetness with drying, puddles, dawn dew, ground frost and
  freezing-rain glaze, radiation fog, snow with degree-hour melting and
  rain-on-snow.
- **Climate memory (new in 1.5)**: a quarterly ENSO/PDO oscillator gives the
  weather El Niño / La Niña events on a realistic 2-7 year rhythm inside
  multi-decade epochs. Read it with `GetEnsoIndex` / `GetPdoIndex` /
  `GetEnsoPhaseText`; the ocean heat (`GetEnsoOceanHeat`) is the system's
  predictor — build fallible NPC forecasters on it. `SetClimateAnomaly` adds a
  manual layer for scripted droughts or cursed biomes.
- **Reproducible weather (new in 1.5)**: `WeatherSeed` in the ini — 0 rolls a
  fresh world; any other value makes fronts, events, lightning and the whole
  climate chronicle a function of the seed and of the time settings. The
  server simulates in fixed steps of game time (`FixedStepGameHours`, 0.01 h
  by default) whatever the framerate: two dedicated servers at 30 and 500
  ticks per second, with the same seed, wrote the same 1,947-day diary line
  for line. The gusts you see run on real time on purpose and never feed the
  simulation or the diary (its daily maximum gust is the day's peak sustained
  wind × 1.5), so a speed change (`Weather.Speed`) applied at a different
  moment changes only the on-screen gusts. The resolved seed travels in the
  save.
- **Two-layer wind audio (new in 1.5)**: a broadband turbulence rumble
  everywhere, and an aeolian howl that requires both a strong flow and nearby
  obstacles (an upwind trace fan from the camera) — an empty field is honestly
  quiet. Per-band **prevailing winds** (trades, westerlies, polar easterlies)
  make cloud drift and rain slant read coherently day after day.
- Persistence (auto-save slot, state v11, backward compatible with v1-v10),
  30-day weather diary, medieval-flavoured calendar, 45 console commands,
  in-game tuning panel (`Weather.Panel`), 17 material parameters published
  every tick.

## Installation

1. Copy `JKWeather` into your project's `Plugins/` folder.
2. Open the project (accept the compile prompt — C++ source ships with the
   plugin) or build from your IDE.
3. Enable **JKWeather** in Edit -> Plugins if it is not already enabled.

### Supported engine versions

| Engine | Status |
|---|---|
| **UE 5.8** | Supported. This is the version the plugin is built and tested against; `JKWeather.uplugin` declares `EngineVersion 5.8.0`. |
| UE 5.4 - 5.7 | Not tested. The plugin ships full C++ source and uses no 5.8-only API that we know of, so it will most likely compile after Unreal rewrites the `.uplugin` version on open — but you are on your own, and the automation suite has never been run there. |
| UE 5.3 and older | Not supported. `BuildSettingsVersion.Latest` and `EngineIncludeOrderVersion.Latest` in the target rules assume a modern include order. |
| UE 4.x | Not supported. |

Platforms: **Win64** (`PlatformAllowList` in the `.uplugin`). It is the only
platform the plugin has been built and tested on. Mac, Linux, consoles and
mobile are not claimed and have never been built; the code uses no
Windows-only API that we know of, so add them to the allow list at your own
risk if you need them.

### Content ships pre-generated

Every runtime asset — `M_Moon` and `M_Starfield`, `MPC_Clouds` with all 17
channels, the SkyFX and precipitation materials, the demo materials and the
audio one-shots and loops — ships pre-built inside the plugin's `Content/`
folder. There is nothing to generate on first run: drop the renderer in a
level and press Play.

The Python utilities under `Scripts/` are development tools and, with one
exception, live in the source repository rather than the distributed package.
The exception is the two provenance generators (`generar_audio_fuente.py`,
`generar_texturas_fuente.py`), which ship with the plugin: they document how
the source art was synthesized and can regenerate an equivalent, fully
original set from a seed (see the AI DISCLOSURE in `LICENSE.txt`). Use them
only to regenerate assets from scratch. If the renderer ever finds a piece
missing, the startup log names it and states the fix
(`[WeatherRenderer] No ...` lines).

**Known issue:** closing the editor immediately after running one of the
`Scripts/` utilities from the Output Log can crash it. It is an editor-only
DX quirk of the Python setup flow, not a runtime/game issue — nothing under
`Scripts/` loads or runs outside the editor. If you run one of these scripts,
save your work and restart the editor before continuing rather than closing
it directly afterward.

## Demo map

Open **Plugins → JKWeather Content → Demo → Map_JKWeatherDemo**: a small
dehesa with holm oaks and granite, lit only by the JKWeatherRenderer. Select
the renderer and change *Weather | Editor Preview* to see each sky without
pressing Play, or press Play to run the full simulation. The ground uses
`M_GroundDemo`, so it gets wet, frosted and snowy with the weather. The oak
leaves do not react to wind or snow; `MF_WindBend` and `MF_ApplySnow` show how
to add that to your own foliage.

**The oaks need Nanite Foliage**, which UE 5.8 ships off. Without it they
render leafless, in the editor and packaged. Add to your project's
`Config/DefaultEngine.ini`:

```ini
[/Script/Engine.RendererSettings]
r.Nanite.Foliage=1
```

The first load of the demo map then builds the tree assemblies once (about
16 minutes measured), and they stay cached.

## Quick start A — self-contained sky (recommended)

1. Drop a **JKWeatherRenderer** actor into your level.
2. That's it. With **Create Own Sky** left on (the default) the actor owns
   and drives its own complete rig: sun and moon directional lights, sky
   light, sky atmosphere, volumetric clouds, height fog, wind source, star
   sphere, moon billboard, SkyFX and lightning light.
3. **Remove or disable any pre-existing sky actors** (Directional Light,
   SkyLight, SkyAtmosphere, VolumetricCloud, ExponentialHeightFog, sky
   spheres) — otherwise you will see two suns and doubled fog. A fresh empty
   level plus the renderer is the cleanest start.
4. The sky already runs in the viewport (see *Editor preview*). Press Play for
   the full simulation. `Weather.Panel` opens the live tuning panel.

## Quick start B — integrate with your existing sky

Keep your own lighting setup and let the system drive it:

1. On the renderer actor turn **Create Own Sky** off.
2. Assign your actors in **Details -> Weather|References**:

| Details panel | Drives |
|---|---|
| **Sun Light** | your sun DirectionalLight (position, lux, colour) |
| **Moon Light** | your moon DirectionalLight |
| **Sky Light** | your SkyLight (day/night floor, lightning pulse) |
| **Sky Atmosphere** | your SkyAtmosphere actor (Mie boost for dust haze) |
| **Height Fog** | your ExponentialHeightFog (event + radiation fog) |
| **Volumetric Cloud** | your VolumetricCloud actor (coverage MID) |
| **Star Sphere** | your star dome mesh |
| **Moon Billboard** | your moon mesh (tag `LunaBillboard` also works) |
| **Wind Source** | your WindDirectionalSource (foliage / cloth) |

Anything left unassigned is searched for in the level, and skipped if absent.

## Recommended project setting: auto exposure

The renderer drives **physically real light values** — 100,000 lux from the sun
at noon, 0.1 from a full moon. Unreal's default auto-exposure range is built for
non-physical values and cannot span that, so the renderer's own post process
widens the range when your project has not. A blank project therefore works
straight away.

If you tune exposure yourself, enable **Project Settings → Rendering → Default Settings →
"Extend default luminance range in Auto Exposure settings"**, then restart the
editor, so that exposure values in the editor read as EV100. In your Post Process Volume, keep Metering Mode on *Auto Exposure
Histogram* and leave Min/Max EV100 at a range rather than pinned to one value
(a fixed exposure cannot accommodate a real day-night cycle).

The plugin logs a note at startup if it is off.

With Lumen, a bright noon can also show Unreal's red viewport message *"Cached
lighting in Lumen and real-time sky capture lighting is going to be clipped"*.
That is the engine's default cached-lighting range ending at EV 12. To cover the
full physical range, add `r.EyeAdaptation.CachedLightingPreExposure=8` under
`[/Script/Engine.RendererSettings]` in `Config/DefaultEngine.ini`.

## Editor preview

The sky works in the level viewport without pressing Play. Select the
`JKWeatherRenderer` and use **Weather | Editor Preview**: *Preview Hour*,
*Preview Day Of Year* and *Preview Weather*. *Time Of Day Override* wins over
*Preview Hour*, so scrubbing it in Sequencer moves the viewport sky.

- It only touches the sky while it runs: lights, fog and atmosphere get their
  values back when you turn it off, before every save and before Play. Edits you
  make to those components during the preview are kept.
- Sky, light, fog and exposure run the game code. Precipitation particles, audio,
  lightning and sky FX (rainbow, halo, aurora, shooting stars) only run in Play.
- With your own sky, a sun that is not **Movable** will not follow the hour.
- A SkyAtmosphere set to *Planet Center At Component Transform* keeps the preview
  off, so a curved-world sky stays as your game configured it.

## Configuration

Simulation defaults are read from **your project's** `Config/DefaultGame.ini`,
section `[/Script/JKWeather.WeatherSubsystem]` — starting date, latitude,
civil-time offset and DST, persistence, day length (`DayLengthRealMinutes`:
real minutes per in-game day at speed 1; default 5, clamped 0.5-1440 —
`Weather.Speed` multiplies on top), the fine physics (lapse rate, `SeaLevelZ`,
`HighSnowBandAltitudeCm` for the high snow band, sleet/hail thresholds,
drought scale), and the v6 climate layer (`WeatherSeed` for reproducible
weather, `bEnableEnso`, `EnsoRegionalSign` and the six coupling scales).

**Autosave and fast clocks.** With `bSaveOnDayChange` the plugin writes a
checkpoint on every day rollover (asynchronously, off the game thread). That
checkpoint is deliberately **skipped while `Weather.Speed` is above 10x**: at
those speeds a day rolls over every few seconds, and serialising that often is
visible as hitching. The honest trade-off is that if the process dies while you
run accelerated, you lose every diary day since the last real save — the
session-close checkpoint, or a manual `Weather.Save`. At normal speed this
never applies.

The on-screen debug HUD (**Show Debug HUD**, non-Shipping builds only) ships
**off** by default on new renderer actors; enable it per actor in Details →
Weather|Debug when calibrating.

**That section does not exist until you add it.** Copy the block from
`Config/JKWeather_DefaultGame.ini` (inside the plugin) into your project's
`Config/DefaultGame.ini`. Its values reproduce the C++ defaults exactly, so
pasting it changes no behaviour — it just makes the simulation editable without
recompiling. Until you do, the plugin runs on the C++ defaults and says so once
in the log at startup.

**Factory anchor.** Out of the box the site is latitude 0, longitude 0, UTC,
and a new game starts on day 81 at 12:00 — exact solar noon with the sun all
but overhead (89.25°): a known-truth sky (sunrise 05:56:40, sunset 18:03:20
with the apparent horizon; exactly 06:00/18:00 with `Horizon Elevation = 0`) to
calibrate exposure, sky and materials against before you move the world to
its real coordinates with the ini, `SetSiteCoordinates()` or `Weather.Site`.
The equation of time ships disabled so that anchor holds; enable it for the
real analemma drift.

The model's **equinox** is day 79, not day 81. Since the solar geometry gained
the equation of the centre — the Earth's orbit is an ellipse, so the sun's true
ecliptic longitude runs up to 1.9° ahead of or behind the mean one — day 79 is
where the declination crosses zero, and that is the real 20th of March. The
plugin's calendar is now aligned with the Earth's instead of two days behind it.
Day 81 stays the start day because it is also the default of the save struct;
ask for day 79 if you want the sun exactly over the equator.

Renderer look-and-feel (volumes, intensities, halo, star brightness, eclipse
toggle, refraction) is per-actor in Details, and most of it is also live in
`Weather.Panel`.

Climate bands: the 7 built-in Earth bands (equatorial to polar) are active
**out of the box** — with no `DefaultClimateProfile` in the ini the subsystem
auto-instantiates the factory Earth profile at startup, so latitude drives the
climate from the first Play, and the factory profile inherits the civil time
derived from your coordinates instead of overriding it. To customise, create a
`JKClimateProfile` Data Asset (it comes pre-filled with the Earth bands) and
point `DefaultClimateProfile` at it in the ini — an authored profile's
civil-time settings then override the ini's. The classic single-band Iberia
climate of the original game remains as a legacy preset (`Weather.Profile
off`). Each band sets temperatures, cloud percentages,
wind, pressure variability, `AnnualPrecipMM` (rate **and** frequency) and
event multipliers. `Weather.Band` prints the active band including its
cloud-to-precipitation thresholds.

## Console commands (non-Shipping builds)

Time & calendar: `Weather.SetDay` `Weather.SetHour` `Weather.Speed`
`Weather.Date` `Weather.Latitude` `Weather.Site` `Weather.Profile`
`Weather.Preset <biome>` — State & info: `Weather.Atmosphere`
`Weather.Sensors` `Weather.Band` `Weather.Enso` `Weather.History` — Persistence:
`Weather.Save` `Weather.Load` `Weather.DeleteSave` `Weather.Reset [day]` —
Forcing / testing: `Weather.Event 0-5` `Weather.Pressure` `Weather.Humidity`
`Weather.Snow` `Weather.Drought` `Weather.Pasture` `Weather.ClearSky` `Weather.Lightning [ic|cg]`
`Weather.Thermals` `Weather.Aurora` `Weather.VFXTest 0-4` `Weather.SkyTest 0-5`
(includes solar/lunar eclipse) `Weather.DebugPrecipitationVFX 0|1` (logs why a
precipitation curtain is or is not visible: intensity, cover, live particles) — Look: `Weather.NightSkyDarkness`
`Weather.NightSkyLight` `Weather.NightExposure` `Weather.TwilightLux` `Weather.TwilightSkyLight` `Weather.StarBrightness` `Weather.StarTwinkle`
`Weather.FogVisibility <m>`
`Weather.Panel` — Climate authoring (run headless with `-ExecCmds`; they
simulate years in seconds and print `[Bench]`-style rows to the log):
`Weather.Bench` `Weather.BenchStorm` `Weather.BenchProfile` `Weather.BenchIberia`
`Weather.Calibrate` `Weather.FitSeasonPrecip`.

Overrides stay forced until released (no argument, `0`, or a negative value
depending on the command — each help text says which). Stopping PIE clears all
overrides.

## Blueprint / C++ API highlights

`UWeatherSubsystem` (world subsystem, `GetWorld()->GetSubsystem<UWeatherSubsystem>()`):
time and calendar getters; atmosphere getters (temperature, feels-like,
pressure + tendency, humidity, dew point, UV, wind with gusts); solar
geometry (`GetSolarAltitudeDegrees(bApparent)`, azimuth, declination, sunrise
and sunset, day phase); precipitation (type, intensity, rate mm/h);
altitude-aware queries (`GetTemperatureAt`, `GetSnowDepthAtZ`,
`GetSnowDepthAtLocation`, `GetSnowLineZ`, `GetThermalTopZ`); events, band and
history; full state save/load (`FJKWeatherState`, v11); network authority
(`HasWeatherAuthority`).

### Curved worlds (planets)

JKWeather assumes a flat world by default. For a spherical world, your game
tells the subsystem where the observer stands; nothing else changes:

- `SetLocalFrame(FQuat)`: local East/North/Up frame to world at the
  observer (local axes: North = +Y, East = -X, Up = +Z). Sun, moon, stars,
  optical FX, wind, precipitation emitters, lightning and the under-roof
  sensor all follow it.
- `SetSphericalAltitude(PlanetCenter, SeaLevelRadiusCm)`: altitude becomes
  the distance to the planet centre minus the sea-level radius
  (`GetTemperatureAt`, `GetSnowDepthAtLocation`, `GetAltitudeZ`).
- `SetVisualVerticalScale(Scale)`: visual-only compression of the volumetric
  cloud layer for small planets (the meteorology keeps real kilometres).
- Feed latitude/longitude with `SetSiteCoordinates` as the observer moves.

With the defaults (identity frame, scale 1, spherical altitude off) the
plugin behaves exactly as in a flat world. The shipped `NS_Rain`, `NS_Snow`
and `NS_Hail` read `User.LocalFrame` for their fall direction; custom
precipitation systems should do the same (see
`Docs/precipitation_vfx_guide.md`). Known gaps: Unreal's Exponential Height
Fog is a flat Z plane, and the `SnowLineZ` / thermal-top MPC channels are
world Z values.

Delegates (BlueprintAssignable): `OnNewDay`, `OnSeasonChanged`,
`OnWeatherEventStarted`, `OnWeatherEventEnded`, `OnPrecipitationChanged`,
`OnLightningStrike` (full strike data: azimuth, distance, power, IC/CG,
thunder delay). In multiplayer they all fire on clients too, driven by the
replicated state and the lightning multicast.

MPC channels written every tick to `MPC_Clouds` (read with a
CollectionParameter node): `NightFactor`, `CloudCoverage`,
`StarBrightness`, `StarTwinkle`, `WindDirection`, `WindStrength`,
`WindGust`, `PrecipitationIntensity`, `GroundWetness`, `SnowCover`,
`SnowLineZ`, `GroundFrost`, `FogDensity`, `RelativeHumidity`,
`LightningFlash`, `ThermalActivity`, `AuroraActivity`.

## Performance

GPU time of the weather passes, measured on an **NVIDIA GeForce RTX 5070 Ti**
at **1920 × 1080**, Development build of a test project with the demo map,
averaged over about 2,000-3,000 frames per scene with the engine's CSV profiler
(`r.GPUCsvStatsEnabled 1`):

| Scene | Weather passes | Whole frame (GPU) |
|---|---|---|
| Clear night | 0.8 ms | 3.4 ms |
| Snow, 98 % cloud | 1.0 ms | 3.9 ms |
| Dense fog | 1.4 ms | 4.2 ms |
| Clear, 13:00 | 1.4 ms | 4.3 ms |
| Rain | 1.6 ms | 4.5 ms |
| Storm | 2.3 ms | 5.1 ms |
| Dust haze, 87 % cloud | 2.5 ms | 5.3 ms |
| Dust haze, 48 % cloud | 3.6 ms | 6.5 ms |

"Weather passes" adds volumetric clouds (0.35-3.1 ms, almost all of the
variation), sky atmosphere (~0.2 ms), height and volumetric fog (~0.15-0.2 ms),
the sky-light capture (0.04-0.18 ms) and the precipitation particles with their
GPU simulation (under 0.05 ms). Volumetric clouds dominate, and they cost most
with broken cloud at a low sun. Sky atmosphere and fog would also run in a
project with Unreal's own sky. Numbers on other GPUs, resolutions and scalability
settings will differ; profile your own scene with `stat gpu`.

## Known limitations

- In multiplayer, the weather **diary/history lives on the server** (client
  `GetHistory()` is empty — query it server-side), state mutators are
  server-only by contract, and a client-side Sequencer `Time Of Day Override`
  is ignored: the clock belongs to the server. Cloud noise, aurora curtains
  and precipitation particles are per-client visual instances of the same
  replicated weather, like foliage wind.
- Polar day and polar night are modelled honestly: day length returns a true
  24 h or 0 h past the polar circle, and every day-state getter (`IsNight`,
  `GetDayPhase`, `SunIntensity`) is derived from solar **elevation**, so none
  of them degenerate. Check `HasSunriseToday()` before reading
  `GetSunriseHour` / `GetSunsetHour`, which carry no meaning on those days;
  `GetSolarRegime()` tells you which of the three regimes you are in. Right on
  the polar circle the regime can flip a day early or late, because the two
  culminations are geometric and the threshold is the apparent horizon (a
  refraction term under 0.6 degrees).
- Snow is simulated at two altitude bands and interpolated in between — it is
  a per-world model, not a per-pixel terrain field. The demo ground material
  uses the valley `SnowCover`; blend by height with the `SnowLineZ` channel in
  your own materials. The contract is **snow where `WorldPosition.Z >=
  SnowLineZ`** — e.g. `saturate((WorldPosition.Z - SnowLineZ) / 10000)`. With
  no snow in either band the channel returns a huge Z, so that expression
  correctly evaluates to zero everywhere; comparing the other way round would
  paint snow over the whole world.
- The under-roof sensor is a cone of line traces above the camera on the
  **`ECC_Visibility`** channel, so anything that blocks visibility counts as a
  roof: a large glass skylight shelters you from the rain, and so does dense
  foliage. Give those meshes their own channel if you need to tell them apart.
  Budget: `NumRays` traces (max 16) per call, uncached — the renderer's own
  use is ~33 traces/s; watch `stat JKWeather` if you call
  `GetSkyOcclusionAt` yourself per actor per frame.
- Month names use deliberate medieval Castilian spelling (Janero, Deziembre).

## Accessibility: reduced flashes

Lightning is short, high-contrast light — the pattern photosensitivity
guidelines warn about. The renderer's **Flash Scale** (`DestelloEscala`,
Details -> Weather|Storm, also live in `Weather.Panel`) is the single knob for
it, and it covers all three consumers at once (SkyLight pulse, the
`LightningFlash` MPC channel and the ground-strike point light):

| Flash Scale | Result |
|---|---|
| `1.0` | Full storm (default) |
| `0.3` | Flashes still readable, much gentler |
| `0.0` | No flash at all — thunder and rain still play, so the storm still reads |

Wire it to your own accessibility menu rather than leaving it in the actor's
Details panel. Thunder volume is separate (`VolumenTrueno`).

## MPC channel names

All 17 channels are in English. If you are coming from any pre-release build,
note that fourteen of them were renamed: the renderer writes **only** the new
names, and a material reading an old one silently receives the asset's default
value forever, with no warning.

| Old | New |
|---|---|
| `Cielo_FactorNoche` | `NightFactor` |
| `Nubes_Cobertura` | `CloudCoverage` |
| `Estrellas_Brillo` | `StarBrightness` |
| `Estrellas_Titileo` | `StarTwinkle` |
| `Viento_Direccion` | `WindDirection` |
| `Viento_Fuerza` | `WindStrength` |
| `Viento_Racha` | `WindGust` |
| `Precipitacion_Intensidad` | `PrecipitationIntensity` |
| `Suelo_Mojado` | `GroundWetness` |
| `Nieve_Acumulada` | `SnowCover` |
| `Humedad_Relativa` | `RelativeHumidity` |
| `Suelo_Escarcha` | `GroundFrost` |
| `Niebla_Fisica` | `FogDensity` |
| `Cielo_Relampago` | `LightningFlash` |

`Scripts/agregar_viento_mpc.py` reports any old name still living in the asset,
with its replacement, so run it and read the log before editing materials by
hand. `Luna_Brillo` and `Luna_DirSol` no longer exist: the moon is driven by a
material instance on the billboard (`DirSol`, `Brillo`, `EclipseLunar`,
`Earthshine` on `M_Moon`).

## License

Distributed exclusively through Fab; use is governed by the Fab EULA. See
`LICENSE.txt` for copyright and third-party notices.

## Support

**Bugs and questions:** https://github.com/jurizuInteractive/jkweather-docs/issues
**Private enquiries (licensing, commercial):** jurizu.interactive@gmail.com

Issues are the preferred channel, and not out of formality: the answer stays
searchable for whoever hits the same thing next. The bug form asks for the
engine version, the platform, the **build configuration** and the **network
mode** — those last two matter more than they look, because the 45 console
commands do not exist in Shipping builds and several derived values are
recomputed locally on a non-authoritative client.

Please include the relevant lines from `Saved/Logs/<Project>.log` — the plugin
logs under the `LogJKWeather` category — and the output of `Weather.Sensors`,
which dumps the whole atmospheric state in one block. With those, most reports
are diagnosable without a round trip.

Before writing, two things answer most questions on their own:

- **`CHANGELOG.md`** for version history and for the reasoning behind every
  behaviour change. Fixes are written up with what broke and why, not just what
  changed.
- **`Docs/`** for the seven subsystem guides (atmosphere, climate bands,
  electrical storms, precipitation VFX, sky FX, demo materials, multiplayer).
  The physics itself is documented as comments next to the code it describes.

If you hit something the docs do not cover, say so in the report — a gap in the
documentation is a bug too.
