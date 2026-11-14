# Migration guide — pre-release names → JKWeather 1.3+

If you are coming from any pre-release build (or from the original development
project), three families of names changed before launch. **Old names are never
written or read anymore**: a material, ini or Blueprint still using one keeps
receiving the stale default forever, with no warning. This page is the
complete mapping.

## 1. MPC channels (`MPC_Clouds`)

The renderer writes **only** the English names. Fourteen channels were
renamed; run `Scripts/agregar_viento_mpc.py` in the editor — it adds any
missing new channel idempotently and reports every old name still living in
the asset, with its replacement.

| Old (pre-release) | New (1.3+) |
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

`ThermalActivity`, `AuroraActivity` and `SnowLineZ` are additions (they never
had an old name). `Luna_Brillo` and `Luna_DirSol` **no longer exist as MPC
channels**: the moon is driven through a material instance on the billboard —
parameters `DirSol`, `Brillo`, `EclipseLunar` and `Earthshine` on `M_Moon`.

## 2. Ini keys (`[/Script/JKWeather.WeatherSubsystem]`)

Two changes. First, the **section itself** was renamed when the module became
JKWeather: the engine no longer reads
`[/Script/DehesaWeatherSystem.WeatherSubsystem]` — move your keys to
`[/Script/JKWeather.WeatherSubsystem]`. Second, four keys were renamed; with
the old name the key is silently ignored and the C++ default applies:

| Old key | New key |
|---|---|
| `SlotAutoguardado` | `AutosaveSlot` |
| `AltitudBandaNieveCm` | `HighSnowBandAltitudeCm` |
| `VentiscaVientoMinKmh` | `BlizzardWindMinKmh` |
| `VentiscaVientoMaxKmh` | `BlizzardWindMaxKmh` |

## 3. C++/Blueprint API

The original Spanish public API was **removed** (not deprecated) in 1.3; the
current names are the English ones in `WeatherSubsystem.h`, and they are
direct one-to-one translations of the old ones (e.g. `ObtenerTemperatura` →
`GetTemperature`, `EstaLloviendo` → `IsRaining`). If a symbol from a
pre-release Blueprint fails to resolve, search `WeatherSubsystem.h` for the
English term — every getter is listed there with its documentation. String
getters were additionally replaced by enums where it mattered:
`GetPrecipitationType()` returns `EJKPrecipitationType`,
`GetActiveEvent()` returns `EJKWeatherEvent`, `GetSeason()` returns
`EJKSeason` — compare with `==` / switch nodes instead of matching text.
