# Atmosphere layer guide (v1.1)

The atmosphere layer is not a loose list of getters: each variable lives in the
simulation and weighs on the others. This guide summarizes what it exposes, how
they couple to each other, which parameters it publishes to the MPC and how it
persists.

## Variables and getters (`UWeatherSubsystem`)

| Variable | Getter | Unit | Type |
|---|---|---|---|
| Air temperature | `GetTemperature()` | C | simulated (already existed) |
| Feels-like temperature | `GetFeelsLikeTemperature()` | C | derived |
| Relative humidity | `GetRelativeHumidity()` | % | simulated |
| Dew point | `GetDewPoint()` | C | derived (Magnus) |
| Barometric pressure | `GetBarometricPressure()` | hPa | simulated |
| Pressure tendency | `GetPressureTendency()` / `GetPressureTrend()` | hPa per game hour / enum | derived |
| Wind speed | `GetWindSpeed()` / `GetWindSpeedWithGust()` | km/h | simulated (already existed) |
| Wind direction | `GetWindDirection()` / `GetWindAngleDegrees()` / `GetWindCardinalText()` | vector / degrees / 16-point rose | simulated |
| UV index | `GetUVIndex()` | 0-11+ | derived |
| Precipitation rate | `GetPrecipitationRateMMH()` | mm/h | derived |
| Surface wetness | `GetSurfaceWetness()` | 0-1 | simulated |
| Accumulated snow | `GetSnowDepthCm()` | cm | simulated |
| Daily rain gauge | `GetRainTodayMM()` | mm | simulated |

`GetAtmosphere()` returns all of the above in a single `FJKAtmosphere` struct
(BlueprintPure): ideal for a weather-station HUD with a single node.

The wind rose points TOWARD where the wind blows, in the plugin's world frame
(North = +Y, East = -X, the same one the sky dome uses).

## Coupling map ("what weighs on what")

- **Pressure is the master variable.** Its slow random walk simulates fronts:
  - An anomaly below 1013 hPa pushes `CloudCoverTarget` up (a 993 low = +24 cloud
    cover); above 1013 it clears.
  - The **gradient** (tendency) and deep lows add wind; a high, still anticyclone
    = calm base breeze.
  - Below 1000 hPa the daily event probability rises x1.6 and fogs tend to turn
    into storms; above 1025 it halves and storms degenerate into anticyclonic fog.
  - Conversely, an **active storm deepens the low** (the barometer plunges with
    the fade-in and recovers as the front passes) and a **heatwave sustains an
    anticyclone**.
  - Playable consequence: watching the barometer fall WARNS of what's coming.
- **Humidity** has a seasonal base (dry summer, humid winter), a daily cycle in
  antiphase with temperature (peaks at dawn), saturates with rain and fog, and
  dries out with the heatwave. From it derive:
  - the **dew point** (Magnus),
  - the warm side of **feels-like** (apparent temperature with vapor pressure and
    wind; the cold side is wind chill, with a continuous blend between 8 and 12 C),
  - the **dawn dew**: with temperature pinned to the dew point, calm and no sun,
    the ground wakes up wet without having rained,
  - and the **renderer fog**: relative humidity scales the density (dry air 55% …
    saturated 145%, adjustable with `NieblaFactorHumedad`).
- **Precipitation** is translated to mm/h by type (drizzle <1, rain up to 8, storm
  12-30, snow in water equivalent) and from there:
  - it wets the ground (`GetSurfaceWetness`), which is later dried by sun, wind and
    dry air;
  - it accumulates **snow** in cm when it snows, which melts by degree-hour above
    0 C (faster with sun and with rain on snow) and whose meltwater also wets the
    ground;
  - it fills the **daily rain gauge**, which empties when the day changes.
- **The UV index** comes from the REAL solar elevation (latitude + declination +
  hour angle, with the same civil noon as the sun's arc) attenuated by cloud
  cover: ~10-11 at clear summer noon, ~2.5 in winter, ~30% of the value under an
  overcast sky.

## Time scale

The surface integrations (wetness, snow, rain gauge) and the barometric tendency
work in **game hours**, so `Weather.Speed` also accelerates the atmosphere. Since
the clock compresses time (a day = 5 real minutes by default),
`RainAccumulationScale` (0.12) compensates for that compression so a storm leaves
~15-35 mm and a drizzle 1-3 mm: plausible daily-report figures at any day length.

## Parameters published to the MPC (`MPC_Clouds`)

The renderer writes them every tick (the parameters must exist in the asset;
`Scripts/agregar_viento_mpc.py` adds them idempotently):

| Parameter | Range | Content |
|---|---|---|
| `GroundWetness` | 0-1 | surface wetness (puddles, roughness, darkened albedo) |
| `SnowCover` | 0-1 | snow cm / `NieveCmParaCubrir` (snow blending on terrain) |
| `RelativeHumidity` | 0-1 | subsystem RH (haze, condensation on glass, mold) |

These are the atmosphere-layer channels only; the renderer publishes **17
channels in total** — the canonical list is in the README ("MPC channel
names"), and the full mapping from every pre-release Spanish channel name is
in `Docs/MIGRACION_JKWeather.md`. A
material still reading an old name silently receives the asset's default
forever — run `Scripts/agregar_viento_mpc.py` to detect leftovers.

`NieveCmParaCubrir` (12 cm by default) is adjusted on the renderer actor,
category **Weather|Atmosphere**.

## Console and panel

- `Weather.Pressure <hPa>` sets the barometer (990 = low, 1030 = high);
  `Weather.Pressure 0` releases it.
- `Weather.Humidity <0-100>` sets the RH; a negative value releases it.
- `Weather.Snow <cm>` puts snow on the ground to test materials.
- `Weather.Atmosphere` prints the full report to the log.
- The panel (`Weather.Panel`) includes the **"Fog by humidity"** row.

## Persistence (state v2)

`FJKWeatherState` now saves pressure (value and target), humidity, wetness,
snow and daily rain, with `Version = 2`. v1 saves load without touching anything:
`SetWeatherState()` detects the old version and re-seeds the atmosphere with values
consistent with the saved season. Derived values (dew, feels-like, UV, tendency)
are never saved: they are recomputed on load.
