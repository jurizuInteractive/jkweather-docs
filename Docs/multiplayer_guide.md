# Multiplayer guide (v7)

JKWeather is server-authoritative and needs **zero setup**: place the same
renderer, run the same code. In standalone nothing changes. In a networked
game (listen or dedicated server) the plugin does the rest by itself.

## How it works

**The server simulates; clients render.** On `OnWorldBeginPlay`, the server's
weather subsystem spawns an `AJKWeatherReplicator` (transient, hidden). Every
simulation tick the subsystem pushes its full `FJKWeatherState` into it; the
engine delta-replicates it at ~2 Hz. The whole struct is ~130 bytes, so the
bandwidth cost is noise. `bAlwaysRelevant` means late joiners get the state
the moment they connect - the first snapshot **is** their synchronization.

**Clients are network slaves that still feel alive.** A client does not
decide anything: it never advances the calendar, rolls weather events,
generates lightning, writes the diary or touches save slots. What it does do
is run the same simulation pipeline you ship for single player, chasing the
replicated `*Target` values (cloud target, wind target, pressure target), so
clouds drift, gusts gust and the barometer needle moves locally with real
dynamics. Between snapshots the client extrapolates `SolarHour` with the
server's replicated clock (`TimeMultiplier`, `DayLengthRealMinutes`), and each
tick it blends the continuous state toward the latest snapshot - the solar
hour and the wind bearing blend **circularly**, so crossing midnight or north
never swings the long way around.

**Lightning is an event, not state.** Each strike travels as a *reliable*
`NetMulticast` of `FJKLightningStrike`: every machine gets the same bearing,
distance and power, and schedules the same distance-delayed thunder. The
`OnLightningStrike` delegate fires on clients exactly like on the server.

**Delegates fire on clients.** `OnNewDay`, `OnSeasonChanged`,
`OnWeatherEventStarted/Ended` and `OnLightningStrike` are driven on clients
from snapshot transitions and the strike multicast; `OnPrecipitationChanged`
fires naturally from the client's own pipeline. One expected quirk: on join,
a client may receive one `OnSeasonChanged` as it lands on the server's date.

## The contract: mutate on the server

Every state mutator (`SetSolarHour`, `SetTimeSpeed`, `ForceWeatherEvent`,
`ApplyClimatePreset`, `SetLatitudeDegrees`, save/load/delete slot, and so on)
is **server-authoritative**: called on a client it logs a warning and does
nothing. Change the weather in server code (or the server's console) and it
arrives replicated. `HasWeatherAuthority()` (BlueprintPure) tells you which
side you are on.

**All 27 public mutators are gated, with no exceptions.** That includes the
tuning knobs (`SetPrecipitationYearScale`, `SetMaritimeFactor01`,
`SetEnsoEnabled`, `SetClimateAnomaly`, `ForceEnsoIndex`) and the two heavy
hammers, `SetWeatherState` and `ResetWeather`. An earlier version of this guide
said those knobs were "harmless by construction" because the next snapshot would
correct them. **That was wrong and it has been fixed.** `PrecipitationIntensity`
and `PrecipitationType` do not travel in `FJKWeatherState`, so no snapshot ever
corrected them: a client typing `Weather.Drought 0` cleared its own rain for the
rest of the session - no particles, no rain audio, no wet ground, clean lines of
sight - while everyone else stayed in the downpour. Do not rely on convergence
to make an ungated mutator safe; gate it.

Console commands go through the same mutators, so a client console cannot
grief the session: `Weather.Speed 1000` on a client is a warning, not a
frozen server.

## What is intentionally per-client (cosmetic)

Cloud shape noise, aurora curtains, precipitation particles and thermal
wisps are visual instances: every client sees the same *weather* (same
coverage, same event, same intensity, same sun) rendered through its own
particles, exactly like foliage wind. The weather **diary/history stays on
the server** - `GetHistory()` on a client returns its own (empty) log; query
it server-side if your gameplay needs it.

## Sequencer and time of day in multiplayer

The clock is the server's. `Time Of Day Override` on a client renderer is
ignored (silently - it is applied every frame); pin the hour on the server
if a synchronized cinematic needs it.

## Test checklist (run before shipping a networked title)

1. **PIE, listen server + 2 clients.** Same sun position, same clouds trend,
   same event on all three windows; thunder audible on all, once each.
2. **Dedicated server + 2 clients.** Same as above; confirm the server log
   shows the replicator spawn line and **no renderer VFX/audio server-side**.

   > A note on doing this properly. PIE's *Play As Client* launches a server
   > process from the **editor** binary, where Slate, the viewport and the
   > renderer all exist even when unused — so it does not really exercise the
   > `IsNetMode(NM_DedicatedServer)` guard that disables the renderer's tick.
   > For that you need a real server binary, which needs a `Server` target in
   > the host project:
   >
   > ```csharp
   > public class MyProjectServerTarget : TargetRules
   > {
   >     public MyProjectServerTarget(TargetInfo Target) : base(Target)
   >     {
   >         Type = TargetType.Server;
   >         DefaultBuildSettings = BuildSettingsVersion.Latest;
   >         IncludeOrderVersion = EngineIncludeOrderVersion.Latest;
   >         ExtraModuleNames.Add("MyProject");
   >     }
   > }
   > ```
   >
   > Package it with `RunUAT BuildCookRun ... -server -noclient`, launch with
   > `MyProjectServer.exe <Map> -log -port=7777`, and connect a client with
   > `open <ip>:7777`. In the server log you should see the weather subsystem
   > start and the replicator spawn, and **no** `[WeatherRenderer]` lines
   > creating Niagara systems, material instances or audio: the simulation runs
   > on the server, the *painting* must not.
3. **Late join.** Connect a client mid-storm: it must arrive already in the
   storm (no clear-sky flash), correct date, correct hour.
4. **Latency.** `Net PktLag=60`, then `120`, then `250` (plus
   `Net PktLagVariance=30`): the sun must stay smooth (extrapolation), clouds
   must not rubber-band, events must start/end once.
5. **Client mutation attempts.** From a client console: `Weather.Speed 50`,
   `Weather.SetHour 3`, `Weather.Event 4`, **`Weather.Drought 0`**,
   **`Weather.Enso 2`**, **`Weather.Reset`** - each must warn and change
   nothing. The last three are the ones that used to slip through.
8. **Storm HUD on a client.** In a storm, `GetElectricalActivity()` must climb
   on the client the same way it does on the server, and
   `GetSecondsSinceLastStrike()` must count up between bolts instead of sitting
   at 0. Hail must be able to fall on a client (it is gated on electrical
   activity > 0.55).
9. **Quarter boundary.** Cross an ENSO quarter with a client connected: its
   rain intensity must follow the server's new phase, not stay on the phase it
   joined with.
6. **Server clock changes.** On the server: `Weather.Speed 50`, then
   `Weather.SetHour 3` - clients must follow smoothly (hour slews the short
   way, no full-day spin).
7. **Save/load.** Only the server writes the slot; stop and relaunch the
   server: clients joining resume the saved weather.
