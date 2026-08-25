# Volumetric Smoke — Compute Shader Ray Marching

> Real-time volumetric smoke rendered entirely on the GPU: a voxel grid drives a two-stage ray
> march, with animated Worley noise breaking up the silhouette.

**Engine:** Unity · **Languages:** C#, HLSL (compute) · **Built:** 2024

---

## Overview

Smoke is represented as an occupancy grid of voxels living in a `ComputeBuffer`, and rendered
by marching a ray per pixel through that volume in a compute shader. There is no particle
system and no billboard geometry — the smoke is evaluated analytically per pixel and composited
back over the frame.

The pipeline is:

```
Voxeliser.cs    →  builds/expands the voxel occupancy grid on a ComputeBuffer
CS_Noise        →  generates animated Worley noise into a RWTexture2D
CSRaytrace      →  marches rays through the grid, accumulating density
Unlit/Combiner  →  composites the smoke buffer over the camera image
```

---

## Ray Marching

### Camera ray reconstruction

Each thread rebuilds its own world-space ray from its pixel coordinate — UV to normalised
device coordinates, through the inverse projection matrix into view space, then into world
space by the camera-to-world matrix:

```hlsl
float2 uv    = id.xy / float2(_TextureWidth, _TextureHeight);
float4 ndcCS = float4(uv.x * 2 - 1, uv.y * 2 - 1, 0, 1);
float3 viewDir   = mul(_invProjectionMatrix, ndcCS).xyz;
float3 rayVector = normalize(mul(_CamToWorldMatrix, float4(viewDir, 0)).xyz);
```

### Two-stage march with empty-space skipping

The expensive part of volumetric rendering is sampling density in places that have none, so the
march runs in two phases.

**Stage one** steps cheaply along the ray doing nothing but occupancy lookups, until it either
enters a smoke voxel or exceeds the maximum ray length:

```hlsl
while (smokeValue <= 0 && length < _RayMaxLength) {
    length += stepSize;
    positionWS = origin + length * rayVector;
    smokeValue = getSmokeVoxel(positionWS);
}
```

**Stage two** only begins once the ray is inside the volume, and accumulates density over a
bounded number of samples. Rays that never intersect the smoke pay only for the cheap
occupancy test — the density function is never evaluated for them.

Voxel lookup flattens 3D coordinates into the linear buffer as
`z * (X * Y) + y * X + x`, with an out-of-bounds guard returning `-1` so the marcher can
distinguish "outside the volume" from "empty voxel".

### Density

Density falls off with normalised distance from the smoke origin, softened by `smoothstep` and
perturbed by the noise texture, which is what stops the smoke reading as a hard analytic sphere.

---

## Procedural Noise on the GPU

`CS_Noise` generates **Worley (cellular) noise** directly into a `RWTexture2D` rather than
loading an authored texture. For each pixel it searches the surrounding 5×5 cell neighbourhood,
hashes each cell to a feature point, and keeps the minimum distance:

```hlsl
float2 cell   = floor(uv) + float2(i, j);
float2 kPoint = cell + frac(sin(dot(cell, float2(12.9898, 78.233))) * 43758.5453);
float  dist   = distance(uv, kPoint);
if (dist < minDist) { minDist = dist; bestPoint = kPoint; }
```

Scrolling the sample coordinates by `_Time` animates the field, so the smoke's internal detail
churns rather than sitting static.

---

## Voxel Debug Visualisation

`Voxeliser` can render the occupancy grid directly as instanced cubes, using a
`ComputeBuffer` of `IndirectArguments` so every voxel is drawn in a single indirect draw call
rather than one call per cube. Smoke expansion over time is driven by an `AnimationCurve`, so
the growth easing is authored in the Inspector instead of hardcoded.

---

## Known Issues

### The noise is sampled in screen space — this is the main bug

`sampleDensity` indexes the noise texture by the ray's **screen UV** rather than by the sampled
world position:

```hlsl
float noiseValue = _NoiseTexture[uint2(uv.x * _NoiseResolution.x, uv.y * _NoiseResolution.y)].r;
```

Because `uv` is constant for the whole ray, every sample along that ray receives the *same*
noise value, and the pattern is pinned to the screen rather than to the volume. The visible
result is that detail **swims across the smoke as the camera moves**, instead of being attached
to it — and the noise adds no variation along the ray's depth at all.

The fix is to sample by world position, so the field is genuinely three-dimensional and
camera-independent:

```hlsl
float noiseValue = sampleNoise3D(samplePoint);   // not screen uv
```

### Others

- **Stage two ignores the volume bounds.** Once density accumulation starts it runs a fixed 40
  steps regardless of whether the ray has left the voxel grid, so smoke can bleed past the
  volume it is supposed to occupy.
- **`col` is reassigned every iteration** of the accumulation loop when it only needs writing
  once after it.
- **The Y bounds test is asymmetric.** `getSmokeVoxel` tests X and Z symmetrically against
  ±extent, but Y against `-extent.y + extent.y` (i.e. `0`) and `extent.y + extent.y`. It works
  because the volume is anchored at ground level, but it reads as a mistake rather than intent.
- **`float length` shadows the HLSL intrinsic `length()`** inside the marching kernel.
- Smoke is rendered to a reduced-resolution target and composited back up, which is the right
  strategy for volumetrics, but there is no upsampling filter — edges are blocky under motion.

---

## What I'd Do Differently

Beyond the bugs above: the density function is a sphere falloff, so this renders a *smoke ball*
rather than smoke. Driving density from the voxel grid itself — with the grid advected by a
simple velocity field — would make it behave like a volume instead of a shaded primitive. The
occupancy buffer is already the right data structure for that; it just isn't being used for
anything except empty-space skipping.
