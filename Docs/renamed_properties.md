# Renamed properties (1.8)

Before 1.8, most public properties of `AJKWeatherRenderer` and two fields of
`UJKWeatherSaveGame` had Spanish C++ names. They now have English names, taken
from the display names the editor already showed, so the Details panel and
Blueprint nodes look the same as before.

**Saved content keeps its values.** `Config/DefaultJKWeather.ini` carries a
`[CoreRedirects]` entry for every rename, so levels, Blueprints and save games
saved with an older version load their values into the new names. Keep that
file while you have content saved with a version before 1.8.

**C++ code must use the new names.** If your code reads or writes these
properties, replace the old identifiers using the table below.

## `UJKWeatherSaveGame`

| Old | New |
|---|---|
| `Estado` | `State` |
| `GuardadoUTC` | `SavedAtUtc` |

## `AJKWeatherReplicator` (internal, not saved)

| Old | New |
|---|---|
| `Estado` | `ReplicatedState` |
| `OnRep_Estado` | `OnRep_ReplicatedState` |
| `MulticastRayo` | `MulticastLightning` |

## `AJKWeatherRenderer`

| Old | New | Editor name | Category |
|---|---|---|---|
| `bCrearCieloPropio` | `bCreateOwnSky` | Create Own Sky | Weather / Custom Sky |
| `SolOverride` | `SunLightOverride` | Sun Light | Weather / References |
| `LunaOverride` | `MoonLightOverride` | Moon Light | Weather / References |
| `AtmosferaOverride` | `SkyAtmosphereOverride` | Sky Atmosphere | Weather / References |
| `NieblaOverride` | `HeightFogOverride` | Height Fog | Weather / References |
| `NubesOverride` | `VolumetricCloudOverride` | Volumetric Cloud | Weather / References |
| `EstrellasOverride` | `StarSphereOverride` | Star Sphere | Weather / References |
| `LunaBillboardOverride` | `MoonBillboardOverride` | Moon Billboard | Weather / References |
| `VientoOverride` | `WindSourceOverride` | Wind Source | Weather / References |
| `MaterialLuna` | `MoonMaterial` | Moon Material | Weather / References |
| `MaterialEstrellas` | `StarfieldMaterial` | Starfield Material | Weather / References |
| `MaterialArcoiris` | `RainbowMaterial` | Rainbow Material | Weather / References |
| `MaterialHalo` | `HaloMaterial` | Halo Material | Weather / References |
| `MaterialFugaz` | `ShootingStarMaterial` | Shooting Star Material | Weather / References |
| `MaterialAurora` | `AuroraMaterial` | Aurora Material | Weather / References |
| `MaterialShimmer` | `HeatShimmerMaterial` | Heat Shimmer Material | Weather / References |
| `SistemaLluvia` | `RainSystem` | Rain System | Weather / Precipitation |
| `SistemaNieve` | `SnowSystem` | Snow System | Weather / Precipitation |
| `SistemaGranizo` | `HailSystem` | Hail System | Weather / Precipitation |
| `bVFXPrecipitacion` | `bPrecipitationVFX` | Precipitation VFX | Weather / Precipitation |
| `AlturaEmisorSobreCamara` | `EmitterHeightAboveCamera` | Emitter Height Above Camera | Weather / Precipitation |
| `TasaMaxLluvia` | `MaxRainSpawnRate` | Max Rain Spawn Rate | Weather / Precipitation |
| `TasaMaxNieve` | `MaxSnowSpawnRate` | Max Snow Spawn Rate | Weather / Precipitation |
| `TasaMaxGranizo` | `MaxHailSpawnRate` | Max Hail Spawn Rate | Weather / Precipitation |
| `DerivaVientoVFX` | `PrecipitationWindDrift` | Wind Drift | Weather / Precipitation |
| `SegundosCompensacionDeriva` | `DriftCompensationSeconds` | Drift Compensation | Weather / Precipitation |
| `VelocidadCaidaLluviaCms` | `RainFallSpeedCms` | Rain Fall Speed (cm/s) | Weather / Precipitation |
| `VelocidadCaidaNieveCms` | `SnowFallSpeedCms` | Snow Fall Speed (cm/s) | Weather / Precipitation |
| `VelocidadCaidaGranizoCms` | `HailFallSpeedCms` | Hail Fall Speed (cm/s) | Weather / Precipitation |
| `AlturaEmisorNieveCms` | `SnowEmitterHeightCm` | Snow Emitter Height (cm) | Weather / Precipitation |
| `bAtenuarBajoTecho` | `bAttenuateUnderCover` | Attenuate Under Cover | Weather / Precipitation |
| `OclusionRayosTecho` | `CoverTraceRays` | Cover Trace Rays | Weather / Precipitation |
| `OclusionConoGrados` | `CoverTraceConeDeg` | Cover Trace Cone | Weather / Precipitation |
| `VelocidadTransicionTecho` | `CoverTransitionSpeed` | Cover Transition Speed | Weather / Precipitation |
| `IntensidadSolMediodia` | `SunIntensityAtNoon` | Sun Intensity at Noon | Weather / Sun & Moon |
| `IntensidadLunaLlena` | `FullMoonIntensity` | Full Moon Intensity | Weather / Sun & Moon |
| `DesfaseLunaHoras` | `MoonPhaseOffsetHours` | Moon Phase Offset | Weather / Sun & Moon |
| `PeriodoLunarDias` | `SynodicPeriodDays` | Synodic Period | Weather / Sun & Moon |
| `bRefraccionAtmosferica` | `bAtmosphericRefraction` | Atmospheric Refraction | Weather / Sun & Moon |
| `InclinacionOrbitaLunaGrados` | `LunarOrbitInclinationDeg` | Lunar Orbit Inclination | Weather / Sun & Moon |
| `PeriodoNodalDias` | `NodalPeriodDays` | Nodal Regression Period | Weather / Sun & Moon |
| `AmplitudPerigeoApogeo` | `PerigeeApogeeAmplitude` | Perigee/Apogee Amplitude | Weather / Sun & Moon |
| `InclinacionLunaGrados` | `MoonAzimuthCorrectionDeg` | Moon Azimuth Correction | Weather / Sun & Moon |
| `DistanciaLunaBillboard` | `MoonBillboardDistance` | Moon Billboard Distance | Weather / Sun & Moon |
| `TamanoLunaBillboard` | `MoonBillboardScale` | Moon Billboard Scale | Weather / Sun & Moon |
| `BrilloLuna` | `MoonDiscBrightness` | Moon Disc Brightness | Weather / Sun & Moon |
| `OrientacionGajoGrados` | `TerminatorOrientationDeg` | Terminator Orientation | Weather / Sun & Moon |
| `bBrilloSegunFase` | `bBrightnessFollowsPhase` | Brightness Follows Phase | Weather / Sun & Moon |
| `PisoFaseLuna` | `NewMoonLightFloor` | New Moon Light Floor | Weather / Sun & Moon |
| `FranjaCrepusculoGrados` | `TwilightBandDeg` | Twilight Band | Weather / Sun & Moon |
| `ExponenteAmanecer` | `SunriseCurve` | Sunrise Curve | Weather / Sun & Moon |
| `SolAtenuacionNubes` | `SunCloudAttenuation` | Cloud Attenuation | Weather / Sun |
| `SolFactorLuxCubierto` | `SunOvercastLuxFactor` | Overcast Lux Factor | Weather / Sun |
| `SkyFactorCubierto` | `SkyLightOvercastFactor` | Overcast Sky Light Factor | Weather / Sun |
| `bCompensacionExposicionClima` | `bWeatherExposureCompensation` | Weather Exposure Compensation | Weather / Exposure |
| `ExposicionCubiertoEV` | `OvercastExposureDropEV` | Overcast Exposure Drop (EV) | Weather / Exposure |
| `ExposicionPrecipitacionEV` | `PrecipitationExposureDropEV` | Precipitation Exposure Drop (EV) | Weather / Exposure |
| `bBalanceBlancosLuzDia` | `bDaylightWhiteBalance` | Daylight White Balance | Weather / Exposure |
| `BalanceBlancosK` | `DaylightWhiteBalanceK` | Daylight White Balance (K) | Weather / Exposure |
| `SolAnguloFuenteDespejado` | `SunSourceAngleClear` | Source Angle (Clear) | Weather / Sun |
| `SolAnguloFuenteCubierto` | `SunSourceAngleOvercast` | Source Angle (Overcast) | Weather / Sun |
| `InclinacionSolGrados` | `SunAzimuthCorrectionDeg` | Sun Azimuth Correction | Weather / Sun & Moon |
| `TemperaturaSolK` | `SunTemperatureK` | Sun Colour Temperature | Weather / Sun & Moon |
| `ColorLuna` | `MoonLightTint` | Moon Light Tint | Weather / Sun & Moon |
| `VelocidadInterpLuces` | `LightInterpolationSpeed` | Light Interpolation Speed | Weather / Sun & Moon |
| `FactorCieloNoche` | `NightSkyLuminance` | Night Sky Luminance | Weather / Sky |
| `BrilloEstrellas` | `StarBrightness` | Star Brightness | Weather / Sky |
| `TitileoEstrellas` | `StarTwinkle` | Star Twinkle | Weather / Sky |
| `IntensidadSkyDia` | `SkyLightIntensityDay` | Sky Light Intensity (Day) | Weather / Sky |
| `IntensidadSkyNoche` | `SkyLightFloor` | Sky Light Floor | Weather / Sky |
| `BrilloNocheLunaLlena` | `FullMoonNightLevel` | Full Moon Night Level | Weather / Sky |
| `BrilloNocheLunaNueva` | `MoonlessNightLevel` | Moonless Night Level | Weather / Sky |
| `GradosFadeEstrellas` | `StarFadeInDepthDeg` | Star Fade-in Depth | Weather / Sky |
| `CoberturaDespejado` | `CloudCoverageClear` | Coverage (Clear) | Weather / Clouds |
| `CoberturaCubierto` | `CloudCoverageOvercast` | Coverage (Overcast) | Weather / Clouds |
| `VelocidadCambioNubes` | `CloudCoverageChangeSpeed` | Coverage Change Speed | Weather / Clouds |
| `AltitudNubesDespejadoKm` | `CloudLayerAltitudeClearKm` | Layer Altitude (Clear) | Weather / Clouds |
| `AltitudNubesCubiertoKm` | `CloudLayerAltitudeOvercastKm` | Layer Altitude (Overcast) | Weather / Clouds |
| `GrosorNubesDespejadoKm` | `CloudLayerThicknessClearKm` | Layer Thickness (Clear) | Weather / Clouds |
| `GrosorNubesCubiertoKm` | `CloudLayerThicknessOvercastKm` | Layer Thickness (Overcast) | Weather / Clouds |
| `NieblaFactorHumedad` | `FogFromHumidity` | Fog from Humidity | Weather / Atmosphere |
| `NieveCmParaCubrir` | `SnowDepthForFullCoverCm` | Snow Depth for Full Cover | Weather / Atmosphere |
| `bRayosVisuales` | `bLightningFlashes` | Lightning Flashes | Weather / Storm |
| `bTruenoAudio` | `bThunderAudio` | Thunder Audio | Weather / Storm |
| `SonidoTruenoCercano` | `NearThunderSound` | Near Thunder Sound | Weather / Storm |
| `SonidoTruenoLejano` | `FarThunderSound` | Far Thunder Sound | Weather / Storm |
| `DistanciaTruenoCercanoM` | `NearThunderDistanceM` | Near Thunder Distance | Weather / Storm |
| `VolumenTrueno` | `ThunderVolume` | Thunder Volume | Weather / Storm |
| `DestelloEscala` | `FlashScale` | Flash Scale | Weather / Storm |
| `DestelloSkyIC` | `SkyFlashIntraCloud` | Sky Flash (Intra-cloud) | Weather / Storm |
| `DestelloSkyCG` | `SkyFlashCloudToGround` | Sky Flash (Cloud-to-ground) | Weather / Storm |
| `DestelloLuzCG` | `StrikePointLight` | Strike Point Light | Weather / Storm |
| `DistanciaVisualMaxM` | `MaxVisualDistanceM` | Max Visual Distance | Weather / Storm |
| `bArcoiris` | `bRainbow` | Rainbow | Weather / SkyFX |
| `ArcoirisIntensidadMax` | `RainbowMaxIntensity` | Rainbow Max Intensity | Weather / SkyFX |
| `HaloIntensidadMax` | `HaloMaxIntensity` | Halo Max Intensity | Weather / SkyFX |
| `bEstrellasFugaces` | `bShootingStars` | Shooting Stars | Weather / SkyFX |
| `FugacesPorMinuto` | `ShootingStarsPerMinute` | Shooting Stars per Minute | Weather / SkyFX |
| `VentanaArcoirisSegundos` | `RainbowWindowSeconds` | Rainbow Window (s) | Weather / SkyFX |
| `DistanciaFXCieloM` | `SkyFXDistanceM` | Sky FX Distance | Weather / SkyFX |
| `ToleranciaNodoEclipseGrados` | `EclipseNodeToleranceDeg` | Eclipse Node Tolerance (deg) | Weather / SkyFX |
| `ToleranciaSicigiaEclipseGrados` | `EclipseSyzygyToleranceDeg` | Eclipse Syzygy Tolerance (deg) | Weather / SkyFX |
| `CalimaTinte` | `DustHazeTint` | Dust Haze Tint | Weather / SkyFX |
| `CalimaAbsorcionEspectral` | `DustSpectralAbsorption` | Dust Spectral Absorption | Weather / SkyFX |
| `CalimaColorCielo` | `DustHazeSkyColor` | Dust Haze Sky Color | Weather / SkyFX |
| `CalimaColorCieloFuerza` | `DustHazeSkyColorStrength` | Dust Haze Sky Color Strength | Weather / SkyFX |
| `CalimaNieblaExtra` | `DustHazeExtraFog` | Dust Haze Extra Fog | Weather / SkyFX |
| `SistemaCalima` | `DustSystem` | Dust System | Weather / SkyFX |
| `TasaMaxCalima` | `MaxDustSpawnRate` | Max Dust Spawn Rate | Weather / SkyFX |
| `AlturaEmisorCalima` | `DustEmitterHeightAboveCamera` | Dust Emitter Height Above Camera | Weather / SkyFX |
| `bTurbidezAmbiental` | `bAmbientTurbidity` | Ambient Turbidity | Weather / SkyFX |
| `TurbidezBoostHorizonte` | `HorizonGlowBoost` | Horizon Glow Boost | Weather / SkyFX |
| `VentiscaVientoVFXExtra` | `BlizzardWindMultiplier` | Blizzard Wind Multiplier | Weather / SkyFX |
| `VentiscaNieblaExtra` | `BlizzardExtraFog` | Blizzard Extra Fog | Weather / SkyFX |
| `bNieblaDeValle` | `bValleyFog` | Valley Fog | Weather / SkyFX |
| `ValleFalloffFactor` | `ValleyFogFalloff` | Valley Fog Falloff | Weather / SkyFX |
| `bTermicas` | `bThermals` | Thermals | Weather / SkyFX |
| `TasaMaxTermica` | `MaxThermalSpawnRate` | Max Thermal Spawn Rate | Weather / SkyFX |
| `SistemaTermica` | `ThermalSystem` | Thermal System | Weather / SkyFX |
| `bLuzCenicienta` | `bEarthshine` | Earthshine | Weather / SkyFX |
| `LuzCenicientaMax` | `EarthshineMaxIntensity` | Earthshine Max Intensity | Weather / SkyFX |
| `AuroraIntensidadMax` | `AuroraMaxIntensity` | Aurora Max Intensity | Weather / SkyFX |
| `SonidoLluviaLoop` | `RainLoopSound` | Rain Loop Sound | Weather / Precipitation |
| `SonidoGranizoLoop` | `HailLoopSound` | Hail Loop Sound | Weather / Precipitation |
| `VolumenLluvia` | `RainVolume` | Rain Volume | Weather / Precipitation |
| `VolumenGranizo` | `HailVolume` | Hail Volume | Weather / Precipitation |
| `LluviaFiltroTechoHz` | `RainLowpassUnderCoverHz` | Rain Lowpass Under Cover | Weather / Precipitation |
| `SonidoVientoLoop` | `WindHowlSound` | Wind Howl Sound | Weather / Wind |
| `VolumenViento` | `WindVolume` | Wind Volume | Weather / Wind |
| `VientoAudibleDesdeKmh` | `RumbleAudibleAboveKmh` | Rumble Audible Above | Weather / Wind |
| `VientoVolumenMaxKmh` | `RumbleFullVolumeKmh` | Rumble Full Volume Speed | Weather / Wind |
| `SonidoVientoRumor` | `RumbleSound` | Rumble Sound | Weather / Wind |
| `ClaseSonidoClima` | `WeatherSoundClass` | Sound Class | Weather / Audio |
| `ConcurrenciaTruenos` | `ThunderConcurrency` | Thunder Concurrency | Weather / Audio |
| `VolumenVientoRumor` | `RumbleVolume` | Rumble Volume | Weather / Wind |
| `AullidoDesdeKmh` | `HowlAudibleAboveKmh` | Howl Audible Above | Weather / Wind |
| `AullidoMaxKmh` | `HowlFullVolumeKmh` | Howl Full Volume Speed | Weather / Wind |
| `bAullidoRequiereObstaculos` | `bHowlRequiresObstacles` | Howl Requires Obstacles | Weather / Wind |
| `SondaObstaculosDistM` | `ObstacleProbeDistanceM` | Obstacle Probe Distance (m) | Weather / Wind |
| `SondaObstaculosRayos` | `ObstacleProbeRays` | Obstacle Probe Rays | Weather / Wind |
| `VientoFiltroTechoHz` | `WindLowpassUnderCoverHz` | Wind Lowpass Under Cover | Weather / Wind |
| `NieblaDensidadBase` | `FogBaseDensity` | Base Density | Weather / Fog |
| `NieblaExtraMadrugada` | `FogExtraDensityAtNight` | Extra Density at Night | Weather / Fog |
| `NieblaFactorNubosidad` | `FogDensityFromCloudCover` | Density from Cloud Cover | Weather / Fog |
| `NieblaUmbralCalorC` | `FogHeatThresholdC` | Heat Threshold | Weather / Fog |
| `NieblaFactorCalor` | `FogHeatDensityFactor` | Heat Density Factor | Weather / Fog |
| `bNieblaPorVisibilidad` | `bFogUseTargetVisibility` | Use Target Visibility | Weather / Fog |
| `VisibilidadNieblaEventoM` | `FogEventVisibilityM` | Fog Event Visibility (m) | Weather / Fog |
| `CalibracionVisibilidad` | `FogVisibilityCalibration` | Visibility Calibration | Weather / Fog |
| `bControlarNieblaVolumetrica` | `bDriveVolumetricFog` | Drive Volumetric Fog | Weather / Fog |
| `ExtincionVolumetricaMax` | `VolumetricExtinctionAtFullFog` | Volumetric Extinction At Full Fog | Weather / Fog |
| `NieblaDensidadEvento` | `FogEventDensityLegacy` | Fog Event Density (legacy) | Weather / Fog |
| `NieblaDensidadMaxima` | `FogMaxDensity` | Max Density | Weather / Fog |
| `ColorNieblaNoche` | `FogNightColor` | Night Colour | Weather / Fog |
| `ColorNieblaDia` | `FogDayColor` | Day Colour | Weather / Fog |
| `VelocidadVientoMaxKmh` | `WindSpeedAtFullStrengthKmh` | Wind Speed at Full Strength | Weather / Wind |
| `EscalaFuerzaViento` | `WindStrengthScale` | Wind Strength Scale | Weather / Wind |
| `EscalaVelocidadComponente` | `WindSpeedScale` | Wind Speed Scale | Weather / Wind |
| `bMostrarDebug` | `bShowDebugHUD` | Show Debug HUD | Weather / Debug |

The renderer's lightning handler `OnRayoGenerado` is now `HandleLightningStrike`
(internal; bound to the subsystem's `OnLightningStrike`).
