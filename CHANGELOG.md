# JKWeather — Changelog

## 1.6.0

Multiplayer, and the audit pass that had to happen before it could ship.

Server-authoritative replication with zero setup is the headline feature, and
the store listing sells it — so it is cut as a release here rather than left
under "Unreleased", where a buyer checking the changelog against the listing
would have concluded the feature was not in the build they bought.

Alongside it: packaging, a documentation sync, one long-requested config knob
and type-safe public getters — plus a line-by-line audit that found the knob had
never been wired to the ini, that seeded runs were not reproducible, that event
durations varied by a factor of 288 with the length of the day, and that three
separate pieces of the simulation were dead on network clients. Those are fixed
below and each one is written up with what it broke and why.

Behaviour changes you will see: dust haze humidity, cloud layer altitude, and
**event durations at any `DayLengthRealMinutes` other than the factory 5**.
Save format moves to **v8**: event timings become game hours, and the six
runtime climate knobs finally travel with the state. v1-v7 saves load fine —
event progress is preserved exactly, and settings a pre-v8 save never carried
are left as configured rather than overwritten.

### Added

- **Multiplayer (v7): server-authoritative replication, zero setup.** The
  server's subsystem auto-spawns a transient `AJKWeatherReplicator`
  (`bAlwaysRelevant`, ~2 Hz) that delta-replicates the full
  `FJKWeatherState` (~130 B) plus the server clock
  (`TimeMultiplier`/`DayLengthRealMinutes`). Clients become network slaves:
  they never advance the calendar, roll events, fire lightning, write the
  diary or touch save slots — instead they extrapolate `SolarHour` with the
  replicated clock, blend each tick toward the latest snapshot (solar hour
  and wind bearing blend **circularly**, so midnight/north never swing the
  long way), and keep running the shipping simulation pipeline chasing the
  replicated `*Target`s, which converges to the server by construction.
  Lightning travels as a **reliable multicast** of `FJKLightningStrike`
  (same bolt, same distance-delayed thunder everywhere). `OnNewDay`,
  `OnSeasonChanged`, `OnWeatherEventStarted/Ended` and `OnLightningStrike`
  fire on clients from snapshot diffs; event *endings* are declared only by
  the authority (clients hold the fade-out edge). Every public mutator and
  the persistence API are gated server-side (`JK_SOLO_AUTORIDAD`): on a
  client they warn and no-op, so a client console cannot grief the session;
  new `HasWeatherAuthority()` (BlueprintPure) exposes the side you are on.
  Late joiners sync on their first snapshot (which also aligns the client
  RNG with the server seeds). Standalone is untouched. New
  `Docs/multiplayer_guide.md` with the architecture and a 9-point network
  test checklist; new automation test `JKWeather.Net.BlendNetState`.
- `DayLengthRealMinutes` ini key (`[/Script/JKWeather.WeatherSubsystem]`):
  real minutes per in-game day at speed 1 (default 5, clamped 0.5-1440).
  Until now the only lever was `Weather.Speed`; a 60-minute day is now
  `DayLengthRealMinutes=60` instead of `Weather.Speed 0.083`.
- `Docs/MIGRACION_JKWeather.md`: the pre-release rename tables (MPC channels,
  ini keys, moon material parameters) referenced from `WeatherSubsystem.h`,
  now actually shipped.
- Automation test seed (`Source/JKWeather/Private/Tests/JKWeatherTests.cpp`):
  calibration-anchor sky (day 81 sunrise/sunset/declination), equation-of-time
  extremes, save-state round trip and v1-default regression, calendar
  validation, no-snow `SnowLineZ` sentinel, and `static_assert`s that pin the
  enum values the internal ints still rely on.
- Repo/process tooling (source repository only, not packaged):
  `Scripts/preflight_release.py` (version/count/packaging drift checks — now
  also validates the reference ini against the header in **both** directions:
  a key documented in the ini with no matching `UPROPERTY(Config)` is a FAIL,
  because that is exactly how `DayLengthRealMinutes` shipped documented and
  dead; a `UPROPERTY(Config)` the ini never mentions is a WARN),
  `Scripts/medir_clima.py` (in-editor measurement bench: samples the subsystem
  during PIE and dumps a CSV, used to validate the 29-Aug fixes against
  numbers rather than impressions),
  `.gitignore`, `.gitattributes`, `.editorconfig`,
  and `DevDocs/` with the language policy + CoreRedirects rename plan.

### Added
- **Fog you can actually get lost in (FOG-01, layer 1).** The Fog event used
  to deliver haze, not fog: the density ceiling was 25-100x below what it
  takes to lose visibility, and the knob asked for an abstract density when
  what anyone actually wants is a distance. The event now drives density from
  a target visibility in metres (`Fog Event Visibility (m)`, default 20),
  derived through Koschmieder's law - extinction is inversely proportional to
  visibility - with a one-time `Visibility Calibration` constant for world
  scale. It also drives volumetric fog extinction, which is what makes dense
  fog read as a body of air rather than a distance tint: on the plugin's own
  fog it switches Volumetric Fog on, on YOUR fog actor it only modulates what
  you already enabled. `Max Density` went from 0.04 to 5.0 (no effect without
  a fog event: base density is already capped at 0.015 before the event is
  added). The old behaviour is one toggle away: turn off `Use Target
  Visibility` and `Fog Event Density` works exactly as before.
  New command `Weather.FogVisibility <m>` for the calibration loop.
  Still to come: the fog layer rising during an event (layer 2) and a real
  wind-driven fog bank with structure (layer 3).
- `stat JKWeather`: a profiling group of the plugin's own, splitting the cost
  of the simulation (subsystem tick) from the presentation (renderer tick),
  with separate counters for the sky-occlusion traces and the precipitation
  VFX/audio pass, plus a retained-history-days counter. Until now the whole
  plugin hid inside STATGROUP_Tickables and generic actor ticking, where a
  buyer could not budget it.
- Sequencer support with no editor module and no custom track: the renderer
  exposes an `Interp`-marked `Time Of Day Override`, so a JKWeatherRenderer
  added to a Level Sequence can key the hour of day like any other float.
  Any value >= 0 pins the solar hour every frame (which also freezes the
  clock for the shot); -1 releases it. Backed by a new `SetSolarHourQuiet()`
  so a sunrise sweep does not emit one log line per frame.
- `ApplyClimatePreset()` and `Weather.Preset <biome>`: one-click landscapes
  (temperate, mediterranean, desert, tropical, oceanic, continental,
  arctic) that move latitude, maritime factor, yearly rain scale and the
  regional ENSO sign to a coherent combination. The latitude bands already
  drove all of this internally; now a buyer does not need to know which
  latitude means which landscape. Longitude and timezone are left alone:
  the site's civil clock stays yours.
- Localisable text layer: every player-facing getter now has an FText twin
  built with `NSLOCTEXT` in the "JKWeather" namespace, so buyers can
  translate the weather through Unreal's standard Gather Text pipeline
  without forking the plugin - `GetSeasonDisplayText`,
  `GetDayPhaseDisplayText`, `GetPrecipitationTypeDisplayText`,
  `GetActiveEventDisplayText`, `GetWindCardinalDisplayText`,
  `GetEnsoPhaseDisplayText` and `GetDayRecordDisplayText`. The diary twin
  composes its sentence with `FText::Format` and `FText::Join`, so word
  order and number formatting are decided per language rather than by the
  code. The existing `FString` getters are untouched and keep working; they
  are what the logs, console commands and debug HUD use, in English on
  purpose.

### Changed
- **The debug HUD now ships OFF, and can be toggled from Blueprint.** `Show
  Debug HUD` defaulted to `true` on every new renderer actor while the README
  promised, in as many words, that it "ships **off** by default on new renderer
  actors". Fifteen lines of yellow text over the buyer's scene was the first
  thing they saw after following Quick start A, with the manual telling them it
  was off. It was also `EditAnywhere` only, so a game that wanted a "debug mode"
  key could not implement one; it is now `BlueprintReadWrite` too.
- **Contract fix (behaviour change):** `FJKAtmosphere.PhysicalFog01` now
  carries the RAW physical fog, exactly as its documentation always claimed,
  and the merged total (physical fog combined with the scheduled Fog event)
  moved to a new `FogDensity01` field alongside it. Until now the snapshot
  silently returned the merged value under the "physical" name, so the two
  could never be told apart. If you were reading `PhysicalFog01` as the
  total, switch to `FogDensity01`; `GetPhysicalFogRaw()` and
  `GetFogDensity01()` are unchanged.
- Every buyer-facing string is now English, per the project's own language
  policy: all 34 console command descriptions, the ~90 runtime log lines, the
  debug HUD (including moon phase names), the weather diary entry text
  (`GetDayRecordText`) and the ENSO phase modifiers. Previously an English
  HUD line could read "March 22, Year 1 | Spring (dia 81)" and the diary
  answered in Spanish while its sibling getters answered in English.
  Spanish stays in the source comments, which are internal.
- The built-in medieval Castilian calendar preset no longer mixes languages:
  its date formats used a literal "Year" inside 13th-century Castilian
  ("A XII dias andados del mes de Otubre, Year III"); they now read
  "anno {Year}".

### Documentation
- **Integrator contracts that were only true in the authors' heads are now
  written down**, next to the thing they constrain: the **no-reentrancy**
  rule for `OnLightningStrike` and its five sibling delegates (your handler
  runs mid-tick; queue mutations for the next frame instead); the full set of
  side effects of `SetDayOfYear` (it also cancels the *pending* event, resets
  `LastEventDay`, and deliberately does **not** archive the half-finished day);
  the day-checkpoint that is skipped above 10x speed and exactly how much
  diary you can lose if the process dies; the `ECC_Visibility` channel of the
  under-roof sensor (glass skylights and dense foliage both count as a roof)
  together with a trace budget; and `SeaLevelZ` as the altitude reference to
  set when your terrain does not live at Z=0.
- **Reduced flashes is now a documented accessibility feature.** `Flash Scale`
  always existed but was sold as a tuning knob; it is the photosensitivity
  control, it covers all three flash consumers at once, and the README now
  says so and asks you to wire it into your own options menu.
- **Supported engine and platform matrix** in the README: 5.8 tested, 5.4-5.7
  untested but likely to compile from source, 5.3 and older unsupported,
  Win64/Mac/Linux only.
- Sequencer's `Time Of Day Override` now documents what a high `Weather.Speed`
  does around it: the pinned hour always wins, but the rest of the weather
  keeps running, so a long take can go from clear to overcast at a frozen
  clock, and day rollovers still fire underneath.
- A note in the automation suite explaining that the "Condition failed" lines
  in the log are the engine's own self-tests sharing the batch, not JKWeather
  failures.

### Removed
- Two stale entries from the README's *Known limitations*, both of which the
  code had already outgrown: "**No network replication**" (contradicted by the
  multiplayer bullet three lines above it, and by the plugin description) and
  the claim that polar day/night is approximated by clamping day length to
  0.35-23.65 h — that clamp was removed when the solar model was consolidated,
  and the day-state getters now derive from elevation, so both regimes are
  modelled honestly. Replaced with what the code actually does.

### Added
- **A public support channel, and a preflight gate that keeps the studio's
  identity separate from the author's.** `SupportURL` was empty — a buyer had
  nowhere to write — and the README's Support section listed no contact at all.
  Support now lives in the documentation repository's **GitHub Issues**, chosen
  over a chat server on purpose: an answer written there is indexed and found by
  whoever hits the same thing next, which is what turns support into
  documentation instead of repeated work. The bug form makes the build
  configuration and the network mode **required** fields, because the 34 console
  commands do not exist in Shipping builds and several derived values are
  recomputed locally on a non-authoritative client — a report missing those two
  usually costs a round trip. The README says what to include
  (engine version, platform, the `LogJKWeather` lines, and the one-block dump
  `Weather.Sensors` produces), because a report without those costs a round
  trip. The new preflight check refuses any personal identifier in the URLs the
  editor's plugin browser shows, or in the files `FilterPlugin.ini` actually
  ships — and it flagged, on its first run, that `DocsURL` still pointed at a
  personal GitHub account. It also knows that a Discord invite defaults to
  expiring after seven days, and warns to make it permanent: a dead support link
  on a store page is worse than an empty field.
- **Two more preflight gates, both for build configurations the plugin had never
  been compiled in.** The first looks for symbols that only exist under
  `#if !UE_BUILD_SHIPPING` / `WITH_EDITOR` guards but are *used* only from
  inside them — a file-scope `static` in that position compiles away in Shipping
  and then trips `unused-function` under warnings-as-errors. It also flags
  headers whose conditional blocks the matching `.cpp` does not mirror. Every
  build of this plugin so far has been Development Editor, and eight guarded
  blocks (one of 459 lines, holding all 34 console commands) had never been
  excluded for real. The check is static analysis: it does not replace packaging
  a Shipping build once, but it catches the cheap regression afterwards.
  The second reports whether the host project has a `Server` target at all —
  without one the dedicated-server binary cannot be built, so the renderer's
  `IsNetMode(NM_DedicatedServer)` guard cannot be exercised. The multiplayer
  guide's checklist now spells out how to build and run that binary, and what
  the server log must and must not contain.

### Fixed
- **Runtime climate settings survive a save and reach network clients.** Six
  values could be changed at runtime — maritime factor, yearly rain scale, ENSO
  regional sign, the manual climate anomaly, longitude and timezone — and none
  of them were in `FJKWeatherState`. Apply the *tropical* preset (latitude 8,
  maritime 0.65, rain x1.6, ENSO sign +1), autosave, reload: the latitude came
  back (it has travelled since v4) but maritime damping and rainfall reverted to
  the ini's values, leaving a half-applied climate. A `SetClimateAnomaly` set by
  a story event vanished the same way. And the subsystem's own header claimed
  those knobs "only tune the client's local prediction and the next snapshot
  corrects it" — which cannot be true of values no snapshot carries. All six are
  now stored, restored and replicated as **save format v8**; the derived
  civil-time offset is recomputed from longitude and timezone on load with the
  same formula `SetSiteCoordinates` uses.
  Saves older than v8 deliberately do **not** overwrite the configured values —
  loading an old game must not silently wipe a preset the project had applied —
  and a new test, `JKWeather.SaveState.V8RuntimeKnobs`, locks both halves of
  that contract. `FJKClimateAnomaly`'s own fields gained the `SaveGame` flag:
  the save archive filters on it recursively, so without it the nested anomaly
  would have serialised empty.
- **A network client no longer consumes the seeded RNG.** `WeatherSeed` promises
  "same seed, same history", and the store listing sells reproducibility as the
  simulation argument — but two client-side paths drew from `WeatherRng` every
  tick, shifting the client's stream away from the server's. Both are closed,
  and they needed different answers:
  *The pressure front redraw* is now authority-only. `PressureTarget` travels in
  `FJKWeatherState` and the snapshot snaps it, so a locally drawn front was
  overwritten anyway — it was wasted work as well as a broken promise.
  *The hail dice* could not simply be gated: precipitation type is not
  replicated, so a client that never rolls would **never see hail**, undoing the
  client-side hail that the electrical-activity fix had just made possible. The
  streams are separated instead — the authority rolls `WeatherRng`, so
  server-side hail stays a function of `WeatherSeed`; the client rolls the
  global `FMath` stream. Client hail falls at slightly different moments per
  client, which is what the multiplayer guide already documents for every
  non-replicated derived value.
  Every remaining `WeatherRng` consumer was audited: clock advance, lightning
  scheduler, strike generation, event roll and event activation all sit behind
  the authority branch or an authority guard. **No client path touches the
  seeded stream any more.**
- **Lightning cadence now runs on the game clock.** The strike scheduler
  integrated in real seconds while the storm event feeding it runs in game
  hours. The mismatch predates this release but the event time-base fix
  *widened* it: the event now scales with both `TimeMultiplier` and
  `DayLengthRealMinutes`, and the scheduler scaled with neither. Measured on the
  bench without looking for it — a day record reading `rain 99.0 mm (24.0 h) |
  14 strike(s)`: twenty-four game hours of thunderstorm rain and fourteen bolts,
  exactly the number that fits in the ~100 *real* seconds the run lasted.
  The strike interval, the initial wait and the storm cell's drift are now kept
  in game hours, converted from their authored values on the same reference
  clock the event durations use — so cadence at factory settings is identical to
  before, and correct at every other day length. `GetSecondsSinceLastStrike()`
  and the thunder delay deliberately stay in **real** seconds: the speed of
  sound and the flash decay are physics at the player's ear, not simulated time.
- **Lightning multicast is `Unreliable`.** A flash and a thunderclap are
  cosmetic; losing one breaks nothing, but a burst of reliable multicasts during
  a storm can fill the channel's reliable buffer and cause hitching or a client
  disconnect. It now degrades where it should.
- **`SetDayOfYear` actually schedules the destination day's event.** It set the
  once-per-day lock to -1 with a comment saying the destination day "can roll a
  fresh event", but `TryGenerateEvent()` was only ever called from the day
  rollover, so the -1 sat inert and a day you jumped to got no natural weather
  event until the *next* rollover. The contract is now honoured rather than
  merely documented.
- **The save-version test is no longer tautological.** `SaveState.RoundTrip`
  asserted "version rewritten as current" while feeding in the current version,
  so it passed whether `GetWeatherState` stamps the version or merely preserves
  it — two different contracts. The round-trip assertion stays, and the stamping
  contract is now proven separately by feeding a v3 state and requiring v7 out.
- **The moon rises on the right side of the night again.** The lunar transit was
  computed as `solar noon − phase offset`. The moon's ecliptic longitude is
  built as `sun longitude + elongation`, so it sits **east** of the sun as the
  offset grows — and a body east of the sun culminates *later*, not earlier. The
  transit is `solar noon + offset`.
  The default hid it: at `Moon Phase Offset = 12` (full moon) both signs give
  midnight, because a full moon is symmetric about ±12. But the synodic cycle
  runs by default, so within a few in-game days the offset leaves 12 and the
  error appears — **twelve hours of it at the quarters**: a first quarter
  transited at 06:00 instead of 18:00, putting the moon in the wrong half of the
  night and contradicting the property's own tooltip ("12 = FULL moon rising at
  dusk; 6 = first quarter"). Only the hour angle was wrong; declination, the lit
  fraction of the disc and the azimuth were already correct, so there was no
  second inversion cancelling it out.
- **Volumetric clouds can now fully clear and fully close.** The material bias
  was fed `CoberturaActual * 2 − 1`. That value is not a 0–1 alpha: it is a
  coverage value clamped to `[Clear Coverage, Overcast Coverage]`, 0.1–0.9 by
  default, so the bias only ever travelled **−0.8 … +0.8** while both the
  comment above it and the header of `Scripts/crear_material_nubes.py` document
  the contract as `Cloud_GlobalCoverage (-1 clear .. +1 overcast)`. The correct
  alpha is `CoberturaNorm`, which normalises that range — the same fix that had
  already been applied to the cloud layer altitude and thickness eleven lines
  above, and missed here.
- **The Blueprint nodes speak English.** UHT turns the `//` block above a
  `UFUNCTION` into the node's tooltip, and 39 Blueprint-exposed functions had
  that block in Spanish — including the ones the README sells by name. A
  designer hovering *Get Sky Occlusion At* read *"SENSOR DE OCLUSION: fraccion
  de cielo tapado sobre un punto…"*. The pattern was already invented inside
  this plugin: the renderer carries explicit `meta = (ToolTip = "…")` on ~90
  properties, so the maintainer's Spanish comment stays put and the buyer reads
  English. The same treatment is now applied across the subsystem, and several
  tooltips were rewritten rather than translated — `GetSkyOcclusionAt` now
  states its trace channel, its cost and the player-pawn exclusion in the node
  itself, and `GetSnowLineZ` carries the material contract that used to live
  only in a code comment.
- **The seven factory climate bands are English and translatable.** They shipped
  as `Ecuatorial`, `Desertica`, `Mediterranea`, `Templada fria`… built with
  `FText::FromString`. Two defects in one: they reach the buyer through
  `GetClimateBandName()` — a `BlueprintPure` returning `FText`, the obvious node
  for a weather-station HUD — and through `Weather.Band` / `Weather.Latitude`;
  and `FText::FromString` is **not picked up by Gather Text**, so the header
  promised localization and delivered a culture-invariant Spanish literal.
  They are now `LOCTEXT` in the `JKWeather` namespace, the same one the seasons
  and events already use, so they enter Unreal's localization pipeline and can
  be translated without recompiling. Nothing compares against these strings, so
  the rename is presentation-only.
- **The last Spanish strings are gone from buyer-facing output.** The 1.5.0
  changelog claimed "every buyer-facing string is now English"; ~14 lines had
  survived it, including one that **switched language mid-sentence** inside the
  auto-exposure warning — the very warning the README devotes a section to
  ("…Auto Exposure settings\" y reinicia el editor"). The rest were the
  renderer's actor-discovery block that the README tells buyers to read (`Sol
  encontrado`, `Luna encontrada`, `SkyLight encontrada`, …), the `SENSORS`
  header and site/ENSO console output, the `Weather.VFXTest` mode names, the
  audio-loading label for hail, and `A TIERRA` for cloud-to-ground strikes. An
  undocumented Spanish alias for the `Weather.Lightning` argument (`tierra`)
  was removed as well. The medieval Castilian calendar preset stays Spanish —
  that is its whole point — and is now marked `// JK_ES_OK`.
  `Scripts/preflight_release.py` gained two checks that keep all of this from
  quietly aging again: one greps Spanish stopwords inside `TEXT("…")` across
  `Source/`, the other flags any Blueprint-exposed `UFUNCTION` whose comment
  block is Spanish and that carries no explicit `ToolTip`. The second one
  immediately caught a thirty-ninth function the manual sweep had missed.
- **Electrical activity is no longer dead on clients.** `CalculateLightning`
  sat entirely inside the Tick's authority branch, and it was the only place
  that ever assigned `ElectricalActivity` and the only place that advanced
  `SecondsSinceStrike`. On a client, therefore, `GetElectricalActivity()`
  returned a flat **0 forever** and `GetSecondsSinceLastStrike()` stayed pinned
  at the 0.0 the strike multicast writes — "a bolt just fell", permanently. Both
  are public `BlueprintPure` nodes with no server-only warning, so any storm HUD
  bound to them lied on every client. The second-order effect was worse:
  `CalculatePrecipitation` gates hail on `GetElectricalActivity() > 0.55`, so a
  **client could never render hail** while the server logged it in the diary.
  The cell-maturity half now runs on both sides as `UpdateElectricalActivity()`;
  only the strike *scheduler* — which draws from `WeatherRng` and multicasts —
  stays server-side. Nothing new is replicated: maturity is derived from cloud
  cover, storm precipitation, the active event and pressure, all of which the
  client already has and all of which converge. Server behaviour is unchanged.
- **The client's ENSO anomaly no longer freezes on join.** The indices
  replicate and blend correctly, so the getters looked right — but
  `RecomputeEnsoAnomaly()`, which turns them into the `AnomalyTotal` the
  simulation actually consumes, was only called from the authority branch of
  `AdvanceClock`. On a client it ran exactly twice (`Initialize` and the first
  snapshot) and then the derived anomaly was stale for the rest of the session.
  Cross a quarter boundary into La Niña (`PrecipIntensityScale` 0.7) and the
  client stayed at 1.0: **~43 % more rain on the client than on the server**,
  forever. It is now recomputed at the end of the client blend tick — pure
  arithmetic, no RNG, no allocation.
- **Seven public mutators were missing their authority guard.** 20 of the 27
  were gated; these were not: `SetPrecipitationYearScale`, `SetMaritimeFactor01`,
  `SetEnsoEnabled`, `SetClimateAnomaly`, `ForceEnsoIndex`, `SetWeatherState` and
  `ResetWeather`. Two hang off console commands with no other guard —
  `Weather.Drought` and `Weather.Enso` — and `multiplayer_guide.md` claimed in
  as many words that those knobs "only shape the client's local prediction and
  the next snapshot corrects it" and that "a client console cannot grief the
  session". **Both sentences were false**, because `PrecipitationIntensity` and
  `PrecipitationType` are not in `FJKWeatherState` and no snapshot corrected
  them. Concretely: two clients in one storm, client A types `Weather.Drought 0`,
  and A's rain is gone for the rest of the session — no particles, no rain
  audio, no wet ground, clean lines of sight — while B is still in the downpour.
  In a shooter that is a competitive advantage typed into a console. All seven
  are now gated; `SetWeatherState` became a gated wrapper over a private
  `ApplyWeatherStateInternal` so the client's first-snapshot sync (which must
  run on a client) and the already-gated load/reset paths still work. The guide
  has been corrected and its checklist now tests these specific commands.
- **Weather events now last the same slice of a day at any day length.**
  `ProcessWeatherEvent` advanced its clock with `DeltaTime * TimeMultiplier`.
  Every other integration in the file goes through `GetGameHoursPerSecond()`,
  which folds in **both** `TimeMultiplier` *and* `DayLengthRealMinutes`; this one
  folded in only the first. So the fraction of a game day an event occupied was
  inversely proportional to the length of the day — and the ini offers all four
  lengths as first-class options:

  | `DayLengthRealMinutes` | A 180 s event actually lasted |
  |---|---|
  | 5 (factory) | **14.4 game hours** — 60 % of the day |
  | 60 | 1.2 h |
  | 240 | 0.3 h |
  | 1440 (real time 1:1) | **0.05 h = 3 game minutes** |

  The same "storm" ranged from half a day to a three-minute shower: a factor of
  288 across the ini's own range. Worse, the storm's own cloud ramp (`case 4`)
  already integrated in game hours, so an event's intensity and its cloud cover
  were running on two different clocks.
  `EventDuration`, `EventElapsed` and both fades are now kept in **game hours**
  and advanced with `GetGameHoursPerSecond()`. The public API does not change
  shape: `ForceWeatherEvent` / `Weather.Event` still take **seconds**, now
  defined on the *reference clock* — a 5-minute day at speed 1 — so `180` means
  what it always meant at factory settings (14.4 game hours) and now means the
  same thing at every day length. The natural generator's historical
  `FRandRange(60, 300)` roll was converted once to `FRandRange(4.8, 24.0)`: same
  single draw, so seeded runs keep their exact stream position.
  **Save format v7.** v1-v6 saves are converted on load with the same constant;
  since all four fields scale together, an in-flight event resumes at exactly
  the same progress and only its total length changes. New automation test
  `JKWeather.SaveState.V7EventHours` locks that invariant.
- **The pressure latch no longer voids `WeatherSeed`, and the storm barometer
  finally recovers.** The three weather events wrote straight onto
  `PressureTarget` with monotone ratchets (`Max`/`Min`). That target is
  integrated state, not something recomputed each tick, and it is persisted in
  the save. Two consequences, both serious:
  *(a)* `Min()` is a ratchet, so as a storm faded out and `1013 - 21*I` climbed
  back toward 1013 the rising value was discarded — the target stayed at the
  deepest pressure reached and the barometer **kept falling after the storm was
  over**, which fed ~+25 cloud points back in and started rain hours after it
  had cleared. The comment and `atmosphere_guide.md` had promised the opposite
  ("recovers as the front passes") since 1.2.
  *(b)* Worse and silent: the walk draws a new target whenever
  `|P - PressureTarget| < 0.6`. With the heat-wave target latched at 1024 and
  the whole draw range below it (Mediterranean summer draws in [1010.7, 1021.4];
  Iberia in [1005.9, 1023.7]), `Max()` returned 1024 *every time*, so the
  condition re-armed on the next frame and `WeatherRng.FRandRange` was called
  **once per frame** for the rest of the event — roughly 7,000 draws in a 180 s
  event at 60 fps, 3,500 at 30. Every front, event and lightning bolt afterwards
  came out of a different point in the stream, so two machines with the same
  seed and different frame rates diverged completely. `WeatherSeed` is sold as
  "same seed, same history of fronts, events, El Niños and La Niñas"; it was
  not. (The ENSO/PDO layer was always genuinely reproducible — it hashes
  `(seed, year, quarter)` into a throwaway stream — so only half the promise
  was broken.)
  The event offset now lives in `GetPressureTargetWithEvent()`, applied to a
  copy at the point of use with the same `Max`/`Min` semantics, so behaviour at
  peak intensity is byte-identical to before while `PressureTarget` keeps the
  front the walk drew. The pressure chases the effective target; the redraw test
  still measures against the walk's own target, which during an event sits far
  away — so it does not fire, and the RNG is not burned.
- **`LatitudeDegrees` is clamped on the property, not just in the setters.**
  `SetLatitudeDegrees` and the save restore both clamped to -89..89, but the
  `UPROPERTY(Config)` had no `ClampMin`, so the ini could still set exactly 90:
  `cos(lat) = 0` pins the solar azimuth at 180 and degenerates the arc.
- **`DayLengthRealMinutes` now actually reaches the simulation.** The property
  carried only a `// Config` comment and no `UPROPERTY(Config)`, so the ini key
  documented in the README, in this changelog and in the shipped reference ini
  was dead: it never reached the clock, and there is no setter either, so the
  day length could not be changed by any route. (Which also means the clamp fix
  below was guarding a value the ini could never set.)
- **Dust haze no longer pins relative humidity to 15%.** The event dried the air
  by writing straight onto `RelativeHumidity`, but that variable is *integrated*,
  not recomputed: `CalculateAtmosphere` only interpolates it towards its target
  (~0.04 points per frame with the default 5-minute day at speed 1). Subtracting
  up to 18 points every tick floored it in three or four frames and kept it
  there for the whole event, ignoring intensity and both fades - and dragging
  the dew point and the physical fog down with it. The drying now happens on the
  humidity *target*, the same route the heat wave already used. Same class of
  bug as the storm-coverage fix in 1.3.1.
- **The volumetric cloud layer now reaches its configured altitudes.** The base
  altitude and thickness lerps used `CoberturaActual` as their alpha, but that
  is a material coverage value living in `[Coverage (Clear), Coverage
  (Overcast)]` (0.1-0.9 by default), not a 0-1 alpha. With the defaults the
  layer only ever swung between 4.7 and 2.3 km instead of the 5.0-2.0 km the
  tooltips promise, and since neither coverage field is clamped, a value above 1
  extrapolated the altitude out of range. Now uses `CoberturaNorm`.
- **Under-roof attenuation no longer dies with the precipitation VFX.** The roof
  sensor was updated inside `ActualizarVFXPrecipitacion`, which the tick only
  calls when `bVFXPrecipitacion` is on and which returns early when no Niagara
  component exists - yet the rain audio and the wind audio consume the same
  factor. With the VFX off, or simply without the `NS_Rain`/`NS_Snow`/`NS_Hail`
  assets, `bAtenuarBajoTecho` silently stopped working for sound: full-volume
  downpour and howling wind inside a cave. The sensor now lives in the tick.
- **`GetSkyOcclusionAt` no longer occludes on the viewer's own pawn.** The
  traces carried no ignored actors, so a first-person camera - which sits inside
  its own character capsule, on the Pawn profile, which blocks Visibility - read
  as "under a roof" under open sky. It now takes an optional `IgnoredActor` and,
  left null, ignores player 0's pawn; the renderer's wind obstacle probe already
  did this.
- The weather diary's plain-string line (`GetDayRecordText`) was missing the
  dust haze case, so a calima day printed with no event at all. Its localisable
  twin and the severity ranking both had it.
- `SetAuroraOverride` was missing the server-authority guard its sibling
  `SetConvectionOverride` carries.
- `SaveUserIndex` was absent from the shipped reference ini, which claims to
  reproduce the C++ defaults exactly.
- `DayLengthRealMinutes` is now actually clamped to the documented 0.5-1440
  range. Both the shipped reference ini and this changelog claimed it was
  "clamped in code", but no clamp existed at the site that drives the clock:
  `DayLengthRealMinutes=0` made `24/0` infinite and sent `SolarHour` to
  infinity on the first tick - which neither the speed ceiling nor the
  skipped-day valve can absorb, since `FloorToInt(inf)` is undefined.
- Two stale documentation claims in the public headers: `GetSurfaceWetness`
  and `GetSnowDepthCm` still named the pre-1.3 MPC channels
  (`Suelo_Mojado` / `Nieve_Acumulada`) instead of `GroundWetness` /
  `SnowCover`, and the renderer class comment named Details categories
  ("Clima|...") that no longer exist ("Weather|...").
- All non-ASCII bytes removed from the source (em dashes, a stray plus-minus
  sign and one tilde), enforcing the rule the project already stated for
  itself.
- Moon billboard, star sphere and optical sky FX now follow the CAMERA
  (Sequencer and spectator views included) instead of the pawn. With the
  pawn, cinematics showed lunar parallax against a hidden character, and
  moving more than the sphere radius (~20 km) from the anchor walked the
  player out of the starfield entirely; camera-centred is also the
  physically correct frame, since stars have no parallax.
- Weather audio can now be routed and capped: a `Sound Class` property feeds
  every weather sound (loops and thunder) into the host game's mixer, a
  `Thunder Concurrency` asset caps how many overlapping rumbles an active
  storm may stack, and silent loops release their voice after 5 s instead of
  playing at volume 0 forever (they restart seamlessly on the next audible
  frame).
- Save slots honour a configurable `SaveUserIndex` (console multi-profile
  certification; was hardcoded to user 0), the day-change checkpoint is now
  written asynchronously off the game thread (the shutdown checkpoint stays
  synchronous on purpose), the save wrapper version is stamped on write, and
  loading a save written by a NEWER plugin version is refused with
  instructions - suppressing this session's autosave so the newer file is
  never overwritten.
- The renderer disables itself on dedicated servers (no VFX, MIDs, traces or
  audio components server-side); the simulation subsystem still runs for
  authoritative gameplay queries.
- Time speed is now clamped to 0-1000 (`Weather.Speed` / `SetTimeSpeed`) and
  `AdvanceClock` gained a valve that compresses pathological multi-day jumps
  (long hitches at high speed) into a single `Weather.SetDay`-style day
  change. Before, an arbitrary multiplier could force millions of per-day
  rollovers in one frame and freeze the game thread.
- NaN guards on three renderer properties that accepted 0 from the editor or
  Blueprint: `Twilight Band` and `Star Fade-in Depth` (division by zero fed
  NaN into light intensities and rotations) and `Sunrise Curve` (exponent 0
  kept the sun on all night; negative values exploded to infinity).
  `ClampMin` metadata plus code-side floors, because Blueprint setters bypass
  property metadata.
- A loud error (log, plus on-screen in non-Shipping) when two
  JKWeatherRenderer actors are active in the same world: they fight over the
  MPC, lights, VFX and audio every frame, and the symptom ("it flickers") is
  the most expensive integration mistake to diagnose.
- `preflight_release.py`: source-art files with no provenance note in
  LICENSE.txt are now a FAIL (was WARN) - nothing ships without papers.
- Deleted the two orphan MP3 thunder placeholders from
  `Resources/SourceArt/Audio/` (replaced by the seeded WAVs in 1.5.0, but
  still in the tree with no provenance note). They were also a live trap:
  `importar_audio_clima.py` tries `.mp3` before `.wav`, so a re-import would
  have silently preferred the old undeclared thunder over the seeded one.
- `preflight_release.py` also FAILs when a provenance script cited by
  LICENSE.txt is missing from `Scripts/` (the AI-disclosure promise is only
  worth the generators backing it), and lost its UTF-8 BOM so the shebang
  works when the script is run directly.
- Reconstructed the two provenance generators the LICENSE cites
  (`Scripts/generar_audio_fuente.py`, `generar_texturas_fuente.py`): the
  2026-08-24 originals (seed 20260823) were lost without a backup. The
  reconstructions document the synthesis method, are measured against the
  shipped files (format, duration, per-band energy, loop seams, texture
  statistics) and regenerate an equivalent original set from a documented
  seed - not byte-identical copies. The LICENSE's AI-disclosure paragraph now
  says exactly that, and both scripts ship in the buyer package
  (`FilterPlugin.ini`), so the folder the buyer's LICENSE points at actually
  contains them. The README's stale `[WeatherRenderer] Sin ...` log reference
  was corrected to the English `No ...` lines while in there.
- The buyer package now includes the real `/CHANGELOG.md` (it was missing from
  `FilterPlugin.ini`, while the stale Spanish 1.3.1 patch notes shipped
  instead; those moved to `DevDocs/` in the source repo).
- `README.md` synced to 1.5.0: version banner, state v6, the full list of 34
  console commands (`Weather.Enso` was missing), and a "New in 1.5" summary
  of ENSO/PDO, `WeatherSeed` and the two-layer wind audio.
- `Docs/atmosphere_guide.md` published the pre-release Spanish MPC names
  (`Suelo_Mojado`, `Nieve_Acumulada`, `Humedad_Relativa`) as current — the
  exact silent-default trap the README warns about. Table corrected to
  `GroundWetness` / `SnowCover` / `RelativeHumidity`, with a pointer to the
  canonical 17-channel table.
- `Docs/demo_materials_guide.md` said 14 live channels; it is 17.
- `LICENSE.txt` rewritten against the real contents of `Resources/SourceArt/`
  (the old notice cited `Universe.jpg`, which no longer exists) with explicit
  per-file provenance fields to complete before submission.
- `GetFogDensity01`, `GetDustHaze01` and `GetBlizzard01` (public header)
  compared raw ints against magic numbers (`ActiveEvent == 1`, `== 5`,
  `PrecipitationType != 4`); they now compare through `EJKWeatherEvent` /
  `EJKPrecipitationType`.

### Changed
- `Show Debug HUD` (`bMostrarDebug`) now defaults to **off** on new renderer
  actors — a commercial default; the exposure warning and `Weather.Panel`
  already cover onboarding. Actors already placed keep their serialized value;
  flip it in Details > Weather|Debug if you relied on it.

## 1.5.0

Weather gains long-term memory. A quarterly ENSO/PDO oscillator gives the
climate El Niño / La Niña events on a realistic 2-7 year rhythm inside
multi-decade active and quiet epochs, and all simulation randomness moves to a
seeded, persisted stream — the same seed now tells the same weather story.
Alongside it, an audit pass on the physics: the wind howl no longer sounds in
an empty field, and five smaller correctness bugs are fixed.

### Added
- **ENSO / PDO climate oscillator.** Two recharge-discharge oscillators (the
  standard toy model of the real phenomenon) step once per calendar quarter:
  a fast ENSO index in ONI convention (>= +0.5 El Niño, <= -0.5 La Niña,
  ±1 moderate, ±1.5 strong, ±2 very strong) with a dominant ~5-year period,
  and a slow PDO (~50-year full cycle) that modulates its amplitude and bias.
  Niños are damped asymmetrically — sharp and short — while Niñas chain for
  two or three years, as they do in the Pacific.
- **Anomalies apply as baseline shifts, never as forced values.** The indices
  are converted daily into a `FJKClimateAnomaly` coupled by latitude (full
  strength in the tropics, a winter-only teleconnection lobe near 40°, almost
  nothing at the poles) and applied to the annual temperature base, the cloud
  target (rain **frequency**, through the existing thresholds), the **centre**
  of the pressure walk (the Southern Oscillation — fronts keep running),
  seasonal humidity, precipitation **intensity** and the storm weight of the
  daily event roulette. Since the whole simulation already reverts toward its
  own baselines, the anomaly enters once and the system self-regulates.
- **Seeded, reproducible simulation.** `WeatherSeed` in the ini: 0 rolls a
  fresh world (previous behaviour), any other value makes fronts, events,
  lightning and the climate chronicle reproducible. The resolved seed and the
  live stream state travel in the save, so each save keeps its own weather.
  The quarterly ENSO noise is a pure function of `(seed, year, quarter)`,
  which makes the centuries-long chronicle immune to framerate and to
  `Weather.SetDay` jumps. Renderer randomness (shooting stars, flash flicker,
  thunder pitch) deliberately stays on the global RNG: it is cosmetic, it
  never reaches the save, and sharing a stream with it would let framerate
  and graphics settings alter the simulation.
- Climate API: `GetEnsoIndex`, `GetPdoIndex`, `GetEnsoOceanHeat` (the ocean
  heat is the system's *predictor* — build fallible NPC forecasters on it),
  `GetEnsoPhaseText`, `ForceEnsoIndex`, `SetEnsoEnabled` / `IsEnsoEnabled`,
  and `SetClimateAnomaly` / `GetClimateAnomaly` — a manual anomaly layer for
  scripted droughts or cursed biomes that combines with the automatic one
  (offsets add, scales multiply).
- `Weather.Enso` console command: prints index, phase, ocean heat, PDO and
  layer state; with an argument forces the index (`Weather.Enso 2`).
- Ini keys: `WeatherSeed`, `bEnableEnso`, `EnsoRegionalSign` (+1 = El Niño
  brings rain to your site, Peru-style; -1 = drought, Australia-style) and the
  six per-index coupling scales.
- **Prevailing winds per climate band**: `PrevailingWindCompassDeg` +
  `WindAngleVarianceDeg` in `FJKClimateBand`, blended circularly across bands.
  The factory Earth profile now ships trade easterlies in the tropics,
  westerlies in the temperate belts and polar easterlies, so cloud drift,
  banners and rain slant read coherently day after day. Variance 180 means
  "no prevailing wind" (uniform), which is what authored pre-1.5 profiles and
  the legacy Iberia band get, so their behaviour is unchanged.
- `EnsoRegionalSignScale` per band (1 follows the global sign, -1 inverts it,
  0 mutes the rain/pressure signal; the thermal anomaly carries no sign), for
  multi-region worlds where the same Niño floods one coast and dries another.
- **Two-layer wind audio**: a broadband turbulence *rumble* with no
  requirements, and the *howl* proper, plus `Rumble Sound`, `Rumble Volume`,
  `Howl Audible Above`, `Howl Full Volume Speed`, `Howl Requires Obstacles`,
  the obstacle-probe distance and ray count, and `Wind Lowpass Under Cover`.
  No new audio asset is required: without one, the rumble reuses the howl
  loop pitched down with a fixed low-pass.
- `GetTimeSpeed()` (Blueprint/C++): reads the clock multiplier.
- Save state **v6**: oscillator indices, ocean heats and RNG seeds. Saves from
  v1-v5 load with a neutral phase and their background weather unchanged.

### Fixed
- **The wind howled all the time, everywhere.** Three separate faults. The
  threshold was 6 km/h — Beaufort 2, barely moving leaves — so the *howl*
  sample played almost permanently. There was no obstacle requirement, when a
  howl is by definition an aeolian tone: vortices shedding off cables,
  branches, edges and gaps, which is also why its pitch rises with speed. And
  it ignored the cover sensor, so a gale howled at full volume inside a cave.
  Now the rumble covers open ground (audible from 14 km/h, growing ~V²), the
  howl demands both a strong flow (28-70 km/h) and nearby geometry — a short
  horizontal upwind trace fan from the camera scales it, so an empty field is
  honestly quiet — and both layers muffle and low-pass under cover, exactly
  as the rain already did.
- The valley decided snowfall at `Temperature < 0` while the high band used
  `<= 0.5`: inconsistent between bands and against the physics, since snow
  reaches the ground above freezing. Both now use `<= 0.5` (the 0-2.5 sleet
  window still covers the wet transition).
- Feels-like temperature used the instantaneous gusty wind, so the HUD needle
  shivered with the ~1 s Perlin gust. Wind chill is defined on the sustained
  wind, and that is what it reads now.
- Pending thunder counted down in real seconds while the storm lived in
  compressed game time: at `Weather.Speed 50` the rumble arrived long after
  the storm had cleared. The countdown now scales with the clock.
- The aurora's slow variation was a fixed sine of day and year — the same
  repeating-waveform flaw the cloud cover had before 1.3. It now uses
  per-year seeded Perlin, the same recipe.
- Two one-shot warnings (the looping-thunder safeguard and the missing ini
  section) were function-level statics, so they fired once per editor process
  and stayed silent for every later PIE session. They are per-world now.

### Changed
- **Nocturnal calm.** The thermal component of wind speed now decays after
  sunset (the boundary layer decouples, which is why nights are still), while
  the synoptic terms — pressure gradient and depth of the low — blow the same
  at night, as they should.
- The renderer's `Wind Audible Above` and `Wind Max Volume Speed` are now the
  **rumble** thresholds, with new defaults of 14 and 55 km/h.
- Startup logs the climate layer state and the resolved seed, so a world whose
  weather changed after upgrading says why in its first log lines.

Save format moves to v6 (backward compatible; v1-v5 migrate silently and are
rewritten as v6). **The ENSO layer ships enabled**: existing worlds gain
interannual variability on recompile — set `bEnableEnso=False` in the ini for
pre-1.5 behaviour. A `JKWeatherRenderer` already placed in a level keeps its
serialized wind-audio values, so reset `Wind Audible Above` and `Wind Max
Volume Speed` on those actors or the rumble will start at breeze speeds.

## 1.4.0

The site is now set with real-world coordinates, and the factory default is a
neutral calibration anchor instead of the original development site.

### Added
- `LongitudeDegrees` (east positive) and `TimezoneHours` in the ini, with
  Blueprint getters: the base civil offset is **derived** from them
  (solar noon = 12:00 + timezone − longitude/15) instead of typed by hand.
  `CivilTimeOffset` is no longer a config key.
- `SetSiteCoordinates(Lat, Lon, TimezoneHours)` (Blueprint/C++) sets the whole
  site in one call, and the `Weather.Site lat lon tz` console command mirrors
  it (without arguments it prints the current site and solar noon).
- `MakeFactoryEarthProfile()` (C++): instantiates the built-in 7-band Earth
  profile **inheriting** the current civil time; used by startup and by
  `Weather.Profile default`.

### Changed
- **Neutral factory defaults**: latitude 0, longitude 0, UTC, new games start
  on day 81 at 12:00 — the model equinox with the sun at the zenith, a
  known-truth sky to calibrate against.
- Equation of time is now **off by default** so that anchor holds (12:00 =
  exact transit; with it on, day 81 transits at ~12:08). Enable
  `bUseEquationOfTime` for the real ±16/−14 min analemma drift.
- A climate profile still overrides the derived civil offset (unchanged
  contract); calling `SetSiteCoordinates` afterwards re-derives and wins.
- The Earth latitude bands are now active **out of the box**: with no
  `DefaultClimateProfile` in the ini, startup auto-instantiates the factory
  Earth profile, so latitude drives the climate from the first Play. The
  factory profile inherits the ini-derived civil time (authored profiles
  keep their override power). The classic Iberia band remains as a legacy
  preset via `Weather.Profile off` / `SetClimateProfile(null)`, and its
  display name drops the development-site qualifier. An unresolvable
  `DefaultClimateProfile` path now logs a warning and falls back to the
  Earth profile instead of silently landing on Iberia.

### Fixed
- `Weather.Profile default` reused a fixed object name on repeat runs
  (in-place object replacement working by accident); factory profiles now get
  unique instance names via `MakeUniqueObjectName`.

No save-format change (state stays v5). Pre-release note: inis using the old
`CivilTimeOffset=` key must switch to the coordinate pair.

## 1.3.1

Maintenance patch on 1.3.0. No public API or save-format changes (state stays
v5); levels and Blueprints are untouched. (Detailed Spanish dev notes for
this patch live in the source repository under `DevDocs/`.)

### Fixed
- Storm cloud closure is frame-rate independent: it integrates in game hours,
  scales with `Weather.Speed` and freezes on pause.
- Natural weather events start at a random hour of the day (scheduled at the
  day rollover) instead of always at midnight.
- `Weather.SetDay` / `Weather.SetHour` recalculate sun and temperature
  instantly — no stale `IsNight`/`GetDayPhase` on the same frame.
- The hail dice and burst timer no longer advance during a state restore.
- `GetSnowLineZ` no-snow sentinel uses explicit units (`SeaLevelZ` + 1e9 cm).
- Fog windows anchor to the real sunrise/sunset at any latitude and time zone
  (night fog follows `NightAlpha`; valley fog centres on real dawn).
- Reduced render-state churn on volumetric clouds and moon light (epsilon
  gating, same caching pattern as the atmosphere).
- Runtime-created star sphere and moon billboard get their search tags and
  are destroyed on `EndPlay` — no doubled sky after a second renderer or a
  re-stream.
- The tuning panel restores the host game's input mode and cursor on close.
- `crear_materiales_precipitacion.py`: particle alpha now comes from a Custom
  HLSL node (root cause of the "Failed to compile Material" on UE 5.8);
  `reparar_material_estrellas.py` migrated to the 1.3 MPC channel names.

### Added
- `GetWeatherHistoryRef()` (C++ only): history access without copying the
  array; `Weather.History` uses it.

## 1.3.0

### Fixed
- Dust haze (calima) event was unreachable: `Weather.Event 5` now works and
  heat waves under a stable anticyclone can turn into calima naturally.
- Unified time base: pressure, clouds, wind veer, humidity, convection, aurora
  and event duration now integrate in game hours. `Weather.Speed` accelerates
  the weather instead of changing its physics; pause freezes everything; the
  pressure tendency no longer dies at high speed.
- High-latitude clock: sunrise/sunset comparisons are wrap-safe, so `IsNight`,
  day phases and sun intensity work past ~62 degrees (where auroras live).
- Consolidated solar model: the clock and the visible disc share the same
  declination and an apparent-horizon (-0.833 deg) sunrise definition — the
  disc appears the same minute the light comes up (was ~4 min apart).
- Phantom snow: a hidden floor made snow accumulate with zero intensity.
- Thermal anchor follows civil noon instead of fixed 05:00/17:00.
- Weather no longer repeats identically every ~21 days (per-year Perlin).
- Save hitches at high time speed (day-rollover checkpoint skipped above x10).
- Config was inert: the `[/Script/JKWeather.WeatherSubsystem]` ini section now
  exists and all documented properties are editable.
- `MPC_Clouds` completed: all published channels exist in the asset
  (ThermalActivity, AuroraActivity, LightningFlash, FogDensity, GroundFrost,
  RelativeHumidity — and new SnowLineZ).
- Rain/snow/hail VFX no longer vanish when the camera turns (GPU fixed-bounds
  override keeps the camera inside the culling volume).
- Thunder could loop forever (looping source assets + fire-and-forget play):
  assets reimported as one-shots plus a runtime safeguard.
- Electrical activity scales with how deep the sky is into storm territory —
  a barely-stormy sky rumbles occasionally instead of firing at full cadence.
- 22-degree halo is now an occasional phenomenon: thin-veil-only window, dies
  with precipitation, per-day crystal-quality factor.
- Wind howl source audio re-synthesised procedurally (royalty-free): no more
  thunder-band rumble or human-voice character; rain/hail loop DC offsets
  removed.
- Weather diary only counts precipitation hours when measurable (>= 0.1 mm/h).
- Legacy Spanish text wrapper reported "none" during hail; diary now ranks
  sleet, freezing rain, hail and calima.
- Moon material was generated in the wrong place under the wrong name
  (`M_Luna` in `/Game/...`), which is not where the renderer looks
  (`/JKWeather/M_Moon`): outside the original development project the script
  appeared to succeed while the plugin kept reporting a missing moon
  billboard. It also declared only `DirSol` and `Brillo`, so the lunar-eclipse
  tint and the earthshine the renderer drives had nowhere to land (Unreal
  silently drops MID parameters that do not exist) — the advertised lunar
  eclipse could not show. Both parameters are now built into the material, and
  the script verifies at the end that all four exist.
- First-run instructions skipped the moon and starfield scripts entirely, so
  the self-contained sky came up without either. Both are now steps 3 and 4.
- Asset-generation scripts no longer depend on `/Game` paths from the original
  development project; the plugin mount point is the only path they need.
- The `[/Script/JKWeather.WeatherSubsystem]` ini section lives in the *project*,
  not the plugin, so installing the plugin alone left every documented setting
  inert with no indication why. The block now ships as
  `Config/JKWeather_DefaultGame.ini` and the subsystem logs a one-line pointer
  at startup when the section is absent.
- `Weather.DeleteSave` was undone by the shutdown checkpoint: deleting the slot
  and stopping play rewrote it, so the next session resumed anyway and testing
  a new `InitialDay` was impossible. Deleting the autosave slot now suppresses
  automatic saving for the rest of the session (an explicit `Weather.Save`
  re-enables it; `Weather.Reset` keeps saving, since it leaves a valid state).
- `SnowDepthAltaCm` was the only field of `FJKWeatherState` without the
  `SaveGame` flag, so the high snow band vanished silently for projects that
  serialize the struct with a save-game-filtered archive.
- The `SnowLineZ` material contract was documented with the comparison
  inverted, which would paint snow over the entire world when there is none.
  Snow is where `WorldPosition.Z >= SnowLineZ`.
- The renderer no longer keeps its own copy of the solar model: it asks the
  subsystem for altitude and azimuth, so the "change one formula, change the
  other" hazard is gone.
- One log category for the whole plugin (`LogJKWeather`) instead of a
  file-static category plus 27 lines going to `LogTemp`.

### Added
- Per-climate rain **frequency**: `AnnualPrecipMM` shifts the cloud-to-rain
  thresholds (log2 around the 700 mm reference; the default band is
  unchanged). `Weather.Band` prints the active thresholds.
- **Altitude-aware snow**: second simulated snow band (`HighSnowBandAltitudeCm`),
  `GetSnowDepthAtZ / GetSnowDepthAtLocation / GetSnowDepthAltaCm /
  GetSnowLineZ`, and a `SnowLineZ` MPC channel for height-based materials.
  Save state v5 (backward compatible; old saves load seamlessly).
- `Weather.Profile` console command: hot-swap climate profiles
  (`default` loads the built-in 7-band Earth profile; `off` returns to the
  classic single Iberia band).
- Canonical solar API: `GetSolarDeclinationDegrees`,
  `GetSolarAltitudeDegrees(bApparent)`, `GetSolarAzimuthDegrees`.

### Changed
- **Public types renamed**: the `Dehesa` prefix (the name of the project this
  system grew in) is gone. `ADehesaWeatherRenderer` is now
  **`AJKWeatherRenderer`**, and the same single-token substitution applies to
  every public type: `FJKWeatherState`, `FJKAtmosphere`, `FJKLightningStrike`,
  `FJKDayRecord`, `FJKClimateBand`, `UJKClimateProfile`, `UJKWeatherSaveGame`,
  the enums `EJKSeason` / `EJKWeatherEvent` / `EJKPrecipitationType` /
  `EJKDayPhase` / `EJKPressureTrend`, and the delegate types `FJKOn*`. Enum
  members, function names and property names are unchanged.
- **Auto-exposure requirement documented and detected.** The renderer emits
  physically real lux (100,000 at noon), which Unreal's default auto-exposure
  range cannot span: the image saturates to white as the sun climbs. Nothing in
  the plugin said so, and the symptom points nowhere near the cause. The README
  now covers it and the renderer warns once at startup if
  `r.DefaultFeature.AutoExposure.ExtendDefaultLuminanceRange` is off.
- **Every name a buyer sees is now in English.** All 123 public properties of
  the renderer carry a `DisplayName` and an English tooltip, so the Details
  panel reads in English without renaming the underlying variables (which would
  break level and Blueprint references). Four config keys were renamed:
  `SlotAutoguardado` to `AutosaveSlot`, `AltitudBandaNieveCm` to
  `HighSnowBandAltitudeCm`, and `VentiscaVientoMin/MaxKmh` to
  `BlizzardWindMin/MaxKmh` — update those four lines if you carry an ini over
  from a pre-release build, since an unrecognised key falls back to the C++
  default in silence.
- **All 17 MPC channels are now in English**, under one naming convention (flat,
  no category prefix): `NightFactor`, `CloudCoverage`, `StarBrightness`,
  `StarTwinkle`, `WindDirection`, `WindStrength`, `WindGust` and
  `PrecipitationIntensity` join the nine that already were. See the README for
  the full old-to-new table; `agregar_viento_mpc.py` reports any old name left
  in the asset.
- The renderer no longer falls back to `/Game/...` paths from the original
  development project when locating its content: the plugin mount point is the
  only path it uses.

### Removed
- The deprecated Spanish-named API (33 forwarders, including the Spanish text
  variants of phase/season/precipitation/event). Every one had a 1:1 English
  replacement already present, and this happened before first release, so no
  shipped project can be affected. Text now comes only from
  `GetDayPhaseText` / `GetSeasonText` / `GetPrecipitationTypeText` /
  `GetActiveEventText`, with the enums available for in-game localisation.
- `crear_material_estrellas.py` v2, which built the star dome in `/Game` with
  its collection parameters wired to a material parameter collection that no
  longer exists (result: a black sky). The working script, previously named
  `reparar_material_estrellas.py`, takes its place under that name.
- One-time migration scripts from the original project
  (`consolidar_assets_plugin.py`, `borrar_t_universo.py`,
  `migrar_a_jkweather.py`) — their migration is long done and their paths
  predate the JKWeather rename, so running them could only fail. The renamed
  MPC channels are listed in the README.
- The DehesaSoil experiment is no longer bundled (separate future product).

## 1.2.0
- Baseline public feature set (see README "What's included").
