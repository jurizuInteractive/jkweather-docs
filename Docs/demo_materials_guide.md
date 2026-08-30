# Demo materials guide

The renderer publishes 17 live weather channels every tick (MPC `MPC_Clouds` +
a dynamic instance on your volumetric cloud material). This pack provides the
**consumers**: ready-made Material Functions and demo materials so every channel
is visible out of the box, and so you can wire the same effects into your own
materials in minutes.

## 1. Generating the pack (one-time, in the editor)

Run these from the Output Log (`Cmd` dropdown -> `Python`), in this order:

```
py "<plugin>/Scripts/agregar_viento_mpc.py"      # ensures all 14 MPC channels
py "<plugin>/Scripts/crear_materiales_demo.py"   # MF pack + M_GroundDemo
py "<plugin>/Scripts/crear_material_nubes.py"    # M_VolumetricClouds
```

Everything lands in `/JKWeather/Demo`. The scripts are skip-if-exists: delete an
asset in the Content Browser to regenerate it (your tweaks are never overwritten).

## 2. What each asset does

| Asset | Reads | Effect |
|---|---|---|
| `MF_ApplyWetness` | `GroundWetness` | Darkens albedo, drops roughness, grows puddles on horizontal surfaces (world-noise mask). |
| `MF_ApplySnow` | `SnowCover` | Blends a snow layer on upward-facing surfaces with an organic noise edge. |
| `MF_ApplyFrost` | `GroundFrost` | Subtle crystalline whitening on horizontals at cold dawns. |
| `MF_WindBend` | `Viento_Direccion`, `Viento_Fuerza` | World Position Offset sway for foliage (base anchored, top moves, per-position phase). |
| `M_GroundDemo` | the three surface MFs | Two-tone procedural ground with wetness -> frost -> snow chained. Drop it on a plane or landscape to see everything react. |
| `M_VolumetricClouds` | `Cloud_GlobalCoverage`, `Cloud_BottomZ/TopZ` (MID), `Viento_*`, `LightningFlash` (MPC) | Volume-domain cloud material: coverage changes cloud amount, clouds drift with the wind, and lightning lights them from inside. |

## 3. Wiring the MFs into your own materials

Each surface MF has the same signature — insert it between your existing nodes
and the final pins:

- Inputs: `BaseColor` (V3), `Roughness` (scalar)
- Outputs: `BaseColor`, `Roughness`

Recommended chain order: **Wetness -> Frost -> Snow** (snow covers everything).
`MF_WindBend` outputs `WPO`: plug it into World Position Offset of your foliage
material (input `Amplitude` in cm, default 12).

## 4. The volumetric cloud material

Assign `M_VolumetricClouds` to the `VolumetricCloud` component (Details ->
Material). The renderer wraps whatever material is assigned in a dynamic
instance and drives, every tick:

- `Cloud_GlobalCoverage` — -1 clear .. +1 overcast
- `Cloud_BottomZ` / `Cloud_TopZ` — layer limits in world cm (sea level assumed
  at Z=0), used for the vertical density profile

Material knobs you can tune on the asset: `CloudScale`, `DensitySharpness`,
`ExtinctionStrength`, `WindSpeed`, `FlashStrength`.

This is a functional demo (procedural 3D noise), not an AAA skyscape. To use
your own cloud material, keep the parameter names above and the renderer will
drive it identically.

## 5. Testing recipe (console)

```
Weather.VFXTest 1      # rain -> watch M_GroundDemo darken and puddle up
Weather.Snow 20        # snow depth -> snow layer grows
Weather.SetHour 7      # cold clear dawn -> frost on horizontals
Weather.Lightning      # with clouds in view -> inner flash
Weather.ClearSky / Weather.Event storm   # coverage swings the cloud amount
```

## 6. Notes

- The MPC channels the MFs read are guaranteed by `agregar_viento_mpc.py`; if a
  generation script aborts, run that one first.
- Surface MFs mask by `PixelNormalWS.Z`, so walls stay dry/clean — as they
  should.
- Regenerating: delete the asset, re-run the script. Never edit the generated
  graphs expecting re-runs to preserve them; the scripts skip existing assets
  precisely so your manual edits are safe.
