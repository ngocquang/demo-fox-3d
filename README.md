# Forest Pond: real-time three.js scene

A photoreal forest-pond scene in a **single self-contained HTML file**
(`index.html`, three.js r169 as ES modules from jsDelivr, no build step). The
lighting follows a red-fox reference photo: deep shade under a heavy canopy,
hard sunbeams breaking through in dappled patches, a warm rim of light on the
subject, dark earth and exposed roots in front, bright back-lit leaves at the
edges.

Everything is generated procedurally at start-up: PBR textures are baked on the
GPU at 2048 px, leaf and litter atlases are painted on a canvas, and the trees,
rocks, terrain, fish and fox are built from code. The only network requests are
three.js itself, the Helvetiker font for the "ZPS" text, and an optional HDRI.
Each of these has a fallback.

## Run it

Serve the folder with any static server and open it in a WebGL2 browser:

```sh
npx serve .            # or: python3 -m http.server
# then open http://localhost:3000 (or :8000)
```

Opening the file directly (`file://`) also works in Chromium-based browsers.

| Key | Action |
| --- | --- |
| `F` | Toggle cinematic ⇄ free fly |
| `W A S D` + mouse (click to lock the pointer) | Fly |
| `Shift` | Sprint |
| `Space` / `E`, `C` / `Q` | Up / down (no collision with the water, so you can dive at will) |
| `M` | Sound (procedural ambience, low-passed underwater) |
| `P` | Pause |
| `H` | Hide the UI |

The lil-gui panel has controls for time of day (sun elevation and azimuth),
exposure and auto exposure, wind, water murkiness, wave strength, caustics,
fish count, depth of field, render scale, and an A/B toggle for every post
effect.

URL parameters:

| Parameter | Effect |
| --- | --- |
| `?t=<sec>` | Start time in the 90 s loop |
| `?free` | Start in free-fly mode |
| `?tex=1024` | PBR texture size |
| `?shadow=2048` | Sun shadow map size |
| `?scale=0.75` | Internal render scale (disables auto scaling) |
| `?nohdri` | Skip the HDRI and use the procedural environment |
| `?hdri=<url>` | Load a custom equirect `.hdr` |
| `?seed=<n>` | Regenerate the forest with another seed |

## What's inside

- **Renderer.** HDR half-float scene target with a float depth texture, ACES
  Filmic tone mapping at exposure 0.9, sRGB output, and a PCFSoft 4096² sun
  shadow map fitted around the pond. Antialiasing is SMAA. The pixel ratio is
  `min(devicePixelRatio, 2)`, and an optional auto render scale holds 60 fps.
- **Lighting.** A low (35°) warm sun comes from behind-left, which gives the fox
  its rim light. Fill comes from a hemisphere light plus IBL: a PMREM HDRI when
  reachable, otherwise a procedural sky-and-canopy environment. The dappled
  light is real canopy shadow from about 60k alpha-tested leaf-cluster cards,
  which sway in the wind in both the visible and the shadow pass. About 15% of
  the ground gets direct sun.
- **Post-processing** (EffectComposer):
  1. Depth-only GTAO at half resolution.
  2. Shadow-map ray-marched volumetric light, which works above and below the
     water.
  3. Radial god rays through the canopy occlusion mask.
  4. Bloom.
  5. Gather bokeh depth of field. It autofocuses on the look-at target, or on
     the nearest fish underwater.
  6. GPU auto exposure, then ACES.
  7. SMAA.
  8. Colour grade: teal shadows, warm highlights, contrast, vignette and 2%
     grain, plus underwater wobble, chromatic aberration and lens droplets.
- **Water.**
  - A 256² grid displaced by 4 Gerstner waves, FBM and ring ripples (fish
    gulps, bubbles, camera splashes), with two scrolling normal maps on top.
  - Half-resolution planar reflection from an oblique-clipped mirror camera.
  - Refraction from a copy of the opaque frame plus linear depth, with
    Beer–Lambert absorption (red dies first).
  - Schlick Fresnel (IOR 1.33), a GGX sun glint with sparkle, a shoreline
    foam/scum line and a pollen film.
  - Caustics projected along the sun direction onto everything underwater, but
    only inside sunlit patches.
  - From below: total internal reflection and a refracted Snell's window.
- **Life.** 12 koi, 4 trout and a school of 30 minnows, with a vertex-shader
  spine wave and physical materials (clearcoat, sheen, iridescence). They run
  boids with obstacle avoidance, startle when you come within 2 m, hover when
  idle, and gulp at the surface. There are also dust motes that only light up
  inside sunbeams, silt, bubble streams and floating leaves.
- **Forest.** Recursive trees (beech, oak, alder) with root flares, bark
  displacement on the near LOD, 3 LOD levels and instanced impostor billboards
  with a hazy forest wall for the far forest. The floor has triplanar mossy
  rocks with parallax occlusion, exposed roots, ferns, grass and thousands of
  leaf-litter cards.
- **Fox.** A low-poly red fox stands on the far bank in its own sunbeam, with
  bevelled, extruded **ZPS** text on its head.

The header comment in `index.html` lists where to swap in real textures, an
HDRI, or GLTF assets (search for `ASSET HOOK`).
