# Latitude-band climate (v1.3)

This version moves the regional configuration out of the code. The original
development climate used to live hardcoded in `WeatherSubsystem`; now it lives
in an editable **DataAsset**, and the subsystem resolves the parameters by
**latitude**, interpolating between bands. **Since 1.4 the factory Earth
profile (7 bands) is auto-instantiated at startup when the ini brings none**,
so latitude drives the climate out of the box — and the factory profile
inherits the civil time derived from your site coordinates instead of
overriding it. The classic single-band Iberia climate remains as a legacy
preset: `Weather.Profile off` or `SetClimateProfile(null)`.

---

## 1. What a band is and what it configures

A band (`FJKClimateBand`) holds the **input numbers** of a climate belt. The
key idea: **you only configure what the simulation needs as input**; everything
else *emerges* from the physical couplings and **is left untouched**.

**Inputs (configured on the band):**

| Group | Fields |
|---|---|
| Air temperature | `WinterTempC/SpringTempC/SummerTempC/AutumnTempC` + `ThermalAmplitudeC` (day/night swing) |
| Pressure | `PressureMinHpa`, `PressureMaxHpa`, `PressureVariability` (front frequency), `SummerPressureStability` |
| Humidity | `WinterHumidityPct/SpringHumidityPct/SummerHumidityPct/AutumnHumidityPct` + `DailyHumidityAmplitudePct` |
| Wind | `WindBaseKmh`, `WindMaxKmh` |
| Cloud cover | `WinterCloudPct/SpringCloudPct/SummerCloudPct/AutumnCloudPct` |
| Precipitation | `AnnualPrecipMM` (scales the *intensity* mm/h of each episode) |
| Events | `FogProbMult`, `HeatWaveProbMult`, `ColdWaveProbMult`, `StormProbMult` |

**Outputs (NOT configured; validated against your table):** feels-like
temperature, dew point, UV index, surface temperature, frost, physical fog,
precipitation type/intensity. They are a consequence of the inputs.

> Example: there is no "feels-like" field. The equatorial band (≈28 °C, ≈82 % RH)
> produces ~40 °C of feels-like **by itself**, through Steadman's apparent
> temperature. The desert (≈37 °C, ≈13 % RH) gives a feels-like *close* to the
> real one because the dry air does not inflate it. If you set feels-like by
> hand, you would break the very coupling that makes the system believable.

### Why per-season cloud cover and not "seasonal concentration"

Instead of asking you for an `AnnualPrecipMM` + an abstract distribution, the band
defines **4 explicit cloud-cover values** (one per season). The reason: in this
engine **precipitation emerges from cloud cover** (drizzle > 65 %, rain > 75 %,
storm > 90 %). Cloud cover is therefore the real knob for *when and how often* it
rains. With it you control the seasonal pattern directly and legibly;
`AnnualPrecipMM` is left only to scale the *intensity* of each episode (mm/h), not
its frequency.

---

## 2. Create a profile

1. **Content Browser → Miscellaneous → Data Asset → `DehesaClimateProfile`**.
2. The asset is **born with the 7 bands of Earth** already filled (equatorial →
   polar) per the reference table. Edit them or add your own.
3. Bands sort themselves by `CenterLatitudeDegrees`; between two centers the
   subsystem **interpolates** (it does not jump): at 32° you get a
   Mediterranean/tropical blend, not a step.

To reproduce your current game with no surprises, create `DA_Climate_Iberia` with
**a single band** copied from `IberiaBand()` (T 6/16/30/14, amplitude 12,
P 995–1030, H 80/62/38/70, wind 5–40, clouds 55/35/15/50). With a single band,
any latitude returns those numbers.

---

## 3. Assign the profile

**Via .ini** (startup), in `DefaultGame.ini`:

```ini
[/Script/JKWeather.WeatherSubsystem]
DefaultClimateProfile=/Game/Climate/DA_Climate_Earth.DA_Climate_Earth
LatitudeDegrees=39.0
```

**At runtime** (Blueprint or C++):

```cpp
Weather->SetClimateProfile(MyProfile);          // smooth transition
Weather->SetClimateProfile(MyProfile, true);    // instant (setup/load)
Weather->SetLatitudeDegrees(-34.6f);            // Buenos Aires: southern hemisphere
```

When you assign a profile, its **civil-time** settings (`CivilTimeOffset` —
which then overrides the offset derived from the ini's longitude/timezone —,
`bUseDaylightSaving`, daylight-saving window) **override** those from the .ini.
For a fantasy world, leave the offset at 0: solar noon falls at 12:00.

---

## 4. Southern hemisphere

Just use a **negative latitude**. The solar geometry already supported it
(`cos ω = −tan φ·tan δ`); the climate layer flips the seasons by shifting the
"climate day" by 182 days. In January, at latitude −34, it is **summer**
(`GetSeason()` returns `Summer`) even though the calendar still reads *January*:
the date is the date; only the season flips. Daylight saving lands in the
southern window (Oct–Apr) automatically.

---

## 5. Transitions

Changing latitude or profile does **not** produce an on-screen jump. The new band
is set as a *target* and the current one blends toward it over
`BandTransitionHours` (**game** hours, so `Weather.Speed` also accelerates the
transition). Useful for long journeys (a ship sailing down in latitude sees the
climate change gradually) or for a dynamic latitude tied to the player's position.

---

## 6. Console commands (non-Shipping builds)

| Command | Effect |
|---|---|
| `Weather.Latitude 45` | Set latitude to 45° N and re-resolve the band (smooth transition) |
| `Weather.Latitude -60` | 60° S (southern hemisphere) |
| `Weather.Latitude` | Print current latitude and band |
| `Weather.Band` | Dump every parameter of the current band to the log |

Combine them with the existing `Weather.SetDay`, `Weather.SetHour` and
`Weather.Speed` to jump across the year and see each season of each band quickly.

---

## 7. Validate against your table

To check that a band reproduces your reference table:

1. `Weather.Latitude <band center>` and `Weather.Band` to lock the values.
2. `Weather.SetDay` to a day in each season (e.g. 15 winter, 219 summer) and
   `Weather.SetHour` to early morning (~5) and mid-afternoon (~15).
3. Read `GetAtmosphere()` (or the debug panel): the **inputs** should match your
   table; the **derived** values (feels-like, dew point, UV) should fall in the
   range your table predicts. If a derived value goes out of range, adjust the
   *input* that feeds it (e.g. tropical feels-like rises with humidity, not with
   a field of its own).

---

## 8. Known limits

- **Polar bands (> 60°):** the midnight sun and the polar night are
  **approximated** with a day of perpetual twilight (duration clamped to
  0.35–23.65 h) so the day/night phases don't break. It is not the exact
  astronomical behavior of the polar circle; it is a playable approximation. If
  your game truly lives in the Arctic, that phase logic would need separate work
  (marked with `TODO` in `CalculateDayLength`).
- **Hurricanes / violent monsoon (tropical > 120 km/h):** out of scope for this
  layer. They are a new event, not a band parameter; the band's sustained wind
  does rise, but there is no tropical-cyclone system.
- **Save compatibility:** the state goes up to **v4** (adds latitude). Saves
  v1–v3 load fine: since they carry no latitude, the configured one is kept
  (ini / `SetLatitudeDegrees`) and the band is resolved from it.
