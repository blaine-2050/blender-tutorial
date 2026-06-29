# Blender + 3D Math: Building a Game World from Zero
### A 10-Hour Tutorial for Beginners on Apple Silicon

> **Philosophy.** Every Blender button does math. This tutorial teaches both simultaneously — you'll build a dark fantasy outdoor environment fit for a game, and understand *why* everything works the way it does. By the end, you'll be as comfortable talking to a graphics programmer as you are clicking in Blender's viewport.

---

## Before You Begin

**You need:**
- Blender 4.x — download free at [blender.org](https://blender.org). Use the macOS Apple Silicon (arm64) build.
- Claude desktop app with the **blender-mcp** add-on (see Hour 0 below).
- About 10 hours, split across as many sessions as you like. Each hour is self-contained.

**What you'll build:**
A moody outdoor environment — a ruined stone tower on a cliff above a river, lit at dusk. Suitable as a game level or cinematic render.

---

## Hour 0 — Setup (30 min, not counted in the 10)

### 0.1 Install Blender on M5 Mac

1. Go to [blender.org/download](https://blender.org/download).
2. Choose **macOS – Apple Silicon**.
3. Drag Blender.app to `/Applications`.
4. First launch: right-click → Open (bypasses Gatekeeper).

M5 Macs run Blender exceptionally well via Metal GPU acceleration. In Blender's preferences (`Cmd+,` → System), set **Cycles Render Devices** to **Metal** and enable your GPU.

### 0.2 Install blender-mcp

`blender-mcp` lets Claude send Python commands directly into your running Blender session. It's the fastest way to iterate — describe what you want, Claude builds it, you adjust visually.

**Step 1 — Install the `uv` package manager:**

```bash
brew install uv
```

**Step 2 — Install the Blender addon:**

1. Download `addon.py` from [github.com/ahujasid/blender-mcp](https://github.com/ahujasid/blender-mcp).
2. In Blender: `Edit → Preferences → Add-ons → Install…`
3. Select the downloaded `addon.py` file.
4. Enable the add-on by checking the box next to **"Interface: Blender MCP"**.
5. Press `N` in the 3D Viewport — you'll see a **BlenderMCP** tab in the sidebar.
6. Click **Connect to Claude**. Blender now listens on `localhost:9876`.

**Step 3 — Connect Claude:**

*Claude Code (CLI):*
```bash
claude mcp add blender uvx blender-mcp
```

*Claude Desktop:* Go to `Claude → Settings → Developer → Edit Config` and add:
```json
{
    "mcpServers": {
        "blender": {
            "command": "uvx",
            "args": ["blender-mcp"]
        }
    }
}
```

**Test it:** Ask Claude: *"Add a UV sphere at the origin with radius 2."* It should appear in your Blender viewport.

### 0.3 Blender Controls Cheat Sheet

| Action | Shortcut |
|---|---|
| Orbit viewport | Middle-mouse drag |
| Pan | Shift + middle-mouse |
| Zoom | Scroll wheel |
| Numpad 1/3/7 | Front/Right/Top view |
| Numpad 5 | Toggle ortho/perspective |
| G / R / S | Grab / Rotate / Scale |
| G then X/Y/Z | Constrain to axis |
| Tab | Toggle Edit/Object mode |
| Shift+A | Add object |
| X | Delete |
| Ctrl+Z | Undo |
| F12 | Render |
| N | Toggle properties sidebar |

---

## Hour 1 — Space, Points, and Vectors

### What You'll Build
Add the ground plane and understand exactly what "3D space" means mathematically.

---

### 1.1 The Math: What Is 3D Space?

Blender's viewport is a window into **ℝ³** — the set of all triples of real numbers (x, y, z). Every object, vertex, and light is a *point* in this space.

**A point:** A location. Written as `P = (x, y, z)`. The origin is `(0, 0, 0)`.

**A vector:** A direction and magnitude. Written as `v = (vx, vy, vz)`. Vectors don't have position — they describe *displacement*.

The difference between two points is a vector:
```
v = B - A = (Bx - Ax, By - Ay, Bz - Az)
```

**Blender's coordinate convention:**
- X → right
- Y → forward (into screen in front view)
- Z → up

This matters because game engines often use Y-up or Z-forward differently. You'll handle this at export time.

### 1.2 Vector Length (Magnitude)

The length of a vector is the 3D version of the Pythagorean theorem:

```
|v| = √(vx² + vy² + vz²)
```

**Unit vectors** have length 1. To normalize any vector:
```
v̂ = v / |v|
```

Unit vectors describe *pure direction*. Blender uses them constantly — surface normals (which way a face points), light directions, camera orientation.

### 1.3 Blender: Your First Objects

Open Blender. Delete the default cube: click it, press `X`, confirm.

**Add the ground:**
1. `Shift+A → Mesh → Plane`
2. With it selected, press `S`, type `20`, Enter — scales it to 20×20 meters.
3. In the Properties panel (right side, green triangle icon), set Location to `(0, 0, 0)`.

**What just happened mathematically:** You created 4 vertices at `(±10, ±10, 0)` and scaled them — a linear transformation (more on this in Hour 3).

**Add a reference sphere (delete later):**
1. `Shift+A → Mesh → UV Sphere`
2. In the pop-up (bottom left), set Segments=32, Rings=16, Radius=1.
3. Move it to Z=1: press `G`, `Z`, `1`, Enter.

This sphere sits on your ground plane. Notice how its bottom touches Z=0 exactly because its center is at Z=1 and radius is 1. Coordinates are physical.

### 1.4 The Math: Dot Product

Two vectors. One operation. Endless use in 3D graphics:

```
a · b = ax·bx + ay·by + az·bz = |a||b|cos(θ)
```

Where θ is the angle between them. This means:
- `a · b = 0` → vectors are **perpendicular**
- `a · b > 0` → angle < 90° (roughly same direction)
- `a · b < 0` → angle > 90° (roughly opposite)

**Where Blender uses it:** Lighting. The brightness of a surface point equals `max(0, N̂ · L̂)` where N̂ is the surface normal and L̂ is the direction toward the light. Face the light directly → dot product = 1 → full brightness. Face away → negative → 0 (dark). This is **Lambert's Law**, the foundation of all 3D shading.

### 1.5 The Math: Cross Product

```
a × b = (ay·bz - az·by,  az·bx - ax·bz,  ax·by - ay·bx)
```

Result: a new vector **perpendicular to both a and b**. Length = `|a||b|sin(θ)`.

**Where Blender uses it:** Computing face normals. Given a triangle with vertices A, B, C:
```
edge1 = B - A
edge2 = C - A
normal = normalize(edge1 × edge2)
```

This is exactly what Blender does internally when you add a face. The normal determines which side is "outside" — critical for lighting, backface culling, and rendering.

**Try it:** In Edit Mode (`Tab`), enable **Face Normals** overlay (the Overlays dropdown → Normals → Faces). You'll see blue lines poking out of every face — those are the normals, computed from cross products.

### 1.6 Claude MCP Practice

Ask Claude:
> *"In my Blender scene, add a plane at origin, scale it 20 units, then add 5 smooth rocks (ico-spheres, radius 0.3–0.8, random) scattered between -8 and 8 on X and Y, sitting on Z=0."*

Watch it build your scattered rocks. This is the workflow: describe → Claude scripts → you refine visually.

---

## Hour 2 — Meshes: Vertices, Edges, and Faces

### What You'll Build
The base cliff shape — a rough, angular rock formation.

---

### 2.1 The Math: What Is a Mesh?

A **mesh** is a piecewise-linear approximation of a surface. It consists of:

- **Vertices (V):** Points in ℝ³.
- **Edges (E):** Line segments connecting two vertices.
- **Faces (F):** Polygons bounded by edges. Blender supports n-gons (any number of sides), but triangles (tris) and quads (4-sided) are most common.

**Euler's formula** for a closed polyhedron:
```
V - E + F = 2
```

A cube: 8 vertices, 12 edges, 6 faces → 8 - 12 + 6 = 2. ✓

Triangles are the GPU's native primitive — every quad gets split into 2 triangles before rendering. Blender shows quad counts; the GPU sees triangle counts.

### 2.2 Edit Mode Deep Dive

Select your ground plane. Press `Tab` to enter **Edit Mode**.

The header shows three selection modes:
- **Vertex mode** (1): click individual points
- **Edge mode** (2): click edges
- **Face mode** (3): click faces

Press `A` to select all. Press `Alt+A` to deselect.

**Key operations:**

`E` — **Extrude.** Extends geometry outward, creating new connected faces.

`Ctrl+R` — **Loop Cut.** Adds an edge loop parallel to existing geometry. Scroll to increase cuts.

`I` — **Inset Faces.** Creates a smaller face inside a selected face.

`Ctrl+B` — **Bevel.** Rounds off hard edges by adding geometry.

### 2.3 Build the Cliff Base

1. Delete everything and start fresh: `A`, `X`, Delete All.
2. `Shift+A → Mesh → Cube`
3. Scale it: `S`, `X`, `4`, Enter. `S`, `Y`, `3`, Enter. `S`, `Z`, `2`, Enter.
   You now have a 8×6×4 meter block.
4. Enter Edit Mode (`Tab`), select all (`A`).
5. Right-click → **Subdivide**, set Cuts to 3. This adds geometry to work with.
6. Switch to **Vertex mode** (1).
7. Enable **Proportional Editing** (O key — a circle appears around your cursor when moving).
8. Randomly grab clusters of vertices and displace them slightly to create an irregular cliff surface. Use `G` + axis to move, scroll to control the influence radius.

The goal is a rough, angular mass — not smooth. This will become the cliff your tower sits on.

### 2.4 The Math: Normals and Manifold Geometry

Every face has a **normal vector** pointing outward. Normals affect:
- Lighting (which side gets illuminated)
- Backface culling (the GPU skips faces pointing away from camera)
- Physics (collision detection uses normals to compute bounce direction)

**Manifold geometry:** A mesh is *manifold* (watertight) if every edge is shared by exactly 2 faces, and faces around each vertex form a disk. Non-manifold geometry causes problems in rendering, boolean operations, and game engines.

Check for issues: **Mesh → Clean Up → Merge by Distance** (removes duplicate verts), **Select → Select All by Trait → Non Manifold** (highlights problem geometry).

### 2.5 Smooth vs. Flat Shading

Right-click your cliff → **Shade Smooth**. The normals are interpolated across edges using **Phong shading** — a weighted average of adjacent face normals at each vertex, creating the illusion of a smooth surface even with low-poly geometry.

For a cliff you want **Shade Flat** (hard faceted look) or a mix: right-click → **Shade Smooth By Angle** and enter ~30° when prompted (Blender 4.1+). This smooths only edges below the threshold, leaving sharp cliff faces hard.

**The math:** Phong interpolation computes `N_vertex = normalize(Σ N_face)` for each vertex, then interpolates across the face using **barycentric coordinates** — more on those in Hour 5.

---

## Hour 3 — Transformations and Matrices

### What You'll Build
Position the cliff correctly, add the tower base, understand why transforms compose the way they do.

---

### 3.1 The Math: Linear Transformations

In 3D, three fundamental transforms are all **linear maps** — they can be written as matrix multiplications.

**Scale** by (sx, sy, sz):
```
S = | sx  0   0  |
    |  0  sy  0  |
    |  0   0  sz |
```

**Rotation** around Z by angle θ:
```
Rz = | cos θ  -sin θ  0 |
     | sin θ   cos θ  0 |
     |   0       0    1 |
```

To transform a point: `P' = M · P` (matrix times column vector).

**Composition:** Applying scale then rotation is just matrix multiplication:
```
M = R · S
```
Order matters! `R · S ≠ S · R` in general. Blender applies Scale → Rotation → Translation (SRT order, also called TRS in reverse).

### 3.2 Homogeneous Coordinates

Translation isn't linear — you can't write "add a constant vector" as a matrix multiply in 3D. The solution: add a 4th coordinate w=1, making every point `(x, y, z, 1)`.

Now translation by (tx, ty, tz) is:
```
T = | 1  0  0  tx |
    | 0  1  0  ty |
    | 0  0  1  tz |
    | 0  0  0   1 |
```

This is why 3D transforms are **4×4 matrices**. Every object in Blender has a 4×4 **world matrix** that encodes its full transform. See it: select an object, open the Python console (`Scripting` workspace), type `bpy.context.active_object.matrix_world`.

The full transform matrix is `M = T · R · S`.

### 3.3 Blender: Object Transform Panel

Select any object. Press `N` (or look at the right sidebar) → **Item** tab. You see:
- **Location:** The translation vector (tx, ty, tz)
- **Rotation:** Euler angles in degrees (Blender uses XYZ Euler by default)
- **Scale:** The scale factors

**Apply transforms** (`Ctrl+A → All Transforms`) bakes the matrix into the vertex positions and resets Location to origin, Rotation to 0, Scale to 1. Do this before exporting to a game engine — engines often expect clean transforms.

### 3.4 Build and Position the Tower

With your cliff selected, position it: Location Z = `-1` (partially buried), wherever looks good on XY.

Now add the tower base:
1. `Shift+A → Mesh → Cylinder`, Vertices=8, Radius=2, Depth=6.
2. Move it to sit on top of your cliff. Use `G`, `Z` to lift it.
3. In Edit Mode, select the top face. `E` to extrude upward — make the tower taller.
4. Loop cut (`Ctrl+R`) to add horizontal bands. Select some faces and scale inward to create window ledges.

**Tower shape goal:** Octagonal (8-sided) base, 15m tall, crumbling top (delete some top faces and leave the edges jagged).

### 3.5 The Math: Euler Angles and Gimbal Lock

Blender's default rotation mode uses **Euler angles** — three rotations applied in sequence (X, then Y, then Z). The problem: if one rotation aligns two axes, you lose a degree of freedom. This is **gimbal lock**.

For animation and physics, **quaternions** are better. A quaternion `q = (w, x, y, z)` encodes any rotation without singularities. Switch an object to Quaternion rotation: Properties → Object → Rotation mode → Quaternion.

The math: a unit quaternion represents rotation by angle θ around axis (ux, uy, uz):
```
q = (cos(θ/2), ux·sin(θ/2), uy·sin(θ/2), uz·sin(θ/2))
```

Blender uses quaternions internally and converts for display.

---

## Hour 4 — Sculpting and Organic Detail

### What You'll Build
Add organic detail to the cliff and ruined tower using Blender's sculpt tools.

---

### 4.1 The Math: Displacement Along Normals

Sculpting moves vertices along their **normal direction** by a brush-weighted amount:
```
P' = P + w(P) · ε · N̂(P)
```
Where `w(P)` is the brush weight at point P (based on distance from brush center, falloff curve), ε is strength, and N̂ is the local normal.

**Falloff curves** are typically smooth — Gaussian `w = e^(-d²/r²)` or polynomial. This ensures no hard seams.

### 4.2 Sculpt Mode Workflow

Select the cliff, switch to **Sculpt Mode** (mode dropdown or Ctrl+Tab).

Key brushes:
- **Draw (X):** Push surface out
- **Draw Sharp:** Sharper ridges
- **Grab (G):** Pull large chunks
- **Flatten (Shift+F):** Flatten to average plane — great for cliff faces
- **Crease (Shift+C):** Sharp cracks and crevices
- **Smooth (Shift):** Hold Shift with any brush to smooth

**Settings:** Set Dyntopo on (Dynamic Topology — it subdivides where you sculpt, adding detail only where needed). Constant Detail ~10-12 works well for a cliff.

Workflow for the cliff:
1. **Grab** to pull out cliff face overhangs and general shape
2. **Flatten** large planar faces
3. **Crease** to add vertical crack lines
4. **Draw Sharp** for rock stratification lines (horizontal banding)

For the tower:
1. Switch to Object Mode, select tower.
2. Sculpt Mode → **Crease** along mortar lines (between stones).
3. **Grab** to push out individual stones slightly.

### 4.3 Remeshing

Heavy sculpting can create uneven mesh distribution. **Remesh** to clean up:
- Header → **Remesh** → Voxel Size 0.1m → Remesh.

This replaces your mesh with a uniform voxelization — mathematically it's sampling the implicit surface at a regular grid and running **Marching Cubes** to extract a triangle mesh.

---

## Hour 5 — Materials and Shaders

### What You'll Build
Stone, moss, cliff rock, and dark sky materials using Blender's node shader system.

---

### 5.1 The Math: Light and Surfaces

When light hits a surface, three things happen:
- **Absorption:** Some wavelengths absorbed (creates color)
- **Reflection:** Light bounces off
- **Transmission:** Light passes through (glass, skin)

The **BRDF** (Bidirectional Reflectance Distribution Function) `f(ωi, ωo)` describes how much light incoming from direction ωi reflects toward direction ωo. All real materials have a BRDF.

**Lambertian BRDF** (perfectly diffuse, matte):
```
f = albedo / π
```
Constant — light scatters equally in all directions. The color you see is just the surface color.

**Specular BRDF** (glossy/mirror-like):
```
f ∝ D(h) · G(ωi, ωo) · F(ωi, h) / (4 · (n·ωi) · (n·ωo))
```
Where h is the half-vector, D is the Normal Distribution Function, G is geometric shadowing, F is Fresnel. This is **GGX/Cook-Torrance**, the model Blender's Principled BSDF uses.

### 5.2 The Principled BSDF

Blender's **Principled BSDF** node is a physically based shader. Key parameters:

| Parameter | Meaning | Cliff Rock |
|---|---|---|
| **Base Color** | Albedo (diffuse color) | Dark grey #2a2a2a |
| **Roughness** | 0=mirror, 1=matte | 0.9 |
| **Metallic** | 0=dielectric, 1=conductor | 0 |
| **Normal** | Bump/normal map input | From texture |
| **Subsurface** | Light scattering beneath surface | 0 |

### 5.3 Build the Stone Material

1. Select the tower. Go to the **Material Properties** (sphere icon, right panel).
2. Click **New**. Name it "Stone_Tower".
3. Switch to the **Shader Editor**.

Build this node network:
```
[Texture Coordinate] → UV → [Noise Texture] → [ColorRamp] → Principled Base Color
[Texture Coordinate] → Object → [Musgrave Texture] → [Bump] → Principled Normal
```

**Step by step:**
1. `Shift+A → Input → Texture Coordinate`
2. `Shift+A → Texture → Noise Texture`, plug UV → Vector, Scale=8, Detail=16
3. `Shift+A → Converter → ColorRamp`. Left stop: dark grey #1a1a1a, right: #3d3535. Plug → Principled Base Color.
4. `Shift+A → Texture → Musgrave Texture`, Scale=20.
5. `Shift+A → Vector → Bump`, plug Musgrave → Height, Strength=0.3. Plug → Principled Normal.
6. Set Principled Roughness=0.85, Metallic=0.

**Moss accent:** Add a second Principled BSDF (Base Color #2d4a1e, green), Mix Shader, use Noise Texture → ColorRamp (B&W) as the mix Factor for patchy moss coverage.

### 5.4 UV Unwrapping

Edit Mode → select all → `U → Smart UV Project` — auto-unwraps based on face angles.

**What UV unwrapping does mathematically:** Finds a mapping `f: Surface → [0,1]²` that minimizes distortion. Perfect unwrapping is impossible for closed surfaces (same problem as flat world maps). Blender uses conformal (angle-preserving) mappings where possible.

### 5.5 The Math: Barycentric Coordinates

When the GPU shades a pixel inside a triangle, it interpolates UV coordinates and normals from the three vertices using **barycentric coordinates**:

For point P inside triangle ABC:
```
P = α·A + β·B + γ·C,   where α + β + γ = 1, all ≥ 0
```

The weights (α, β, γ) are proportional to sub-triangle areas. Any vertex attribute interpolates the same way:
```
UV_P = α·UV_A + β·UV_B + γ·UV_C
```

This is what makes smooth shading and texture mapping work.

---

## Hour 6 — Lighting

### What You'll Build
A dramatic dusk lighting setup — warm low sun, cool blue sky fill, subtle god rays.

---

### 6.1 The Math: Radiometry

| Quantity | Unit | Meaning |
|---|---|---|
| **Radiant Flux** | Watts (W) | Total power emitted |
| **Irradiance** | W/m² | Power per unit area received |
| **Luminous Flux** | Lumens (lm) | Flux weighted by human eye sensitivity |

**Inverse square law:**
```
E = Φ / (4π · d²)
```
Double the distance → quarter the brightness. Baked into Blender's point and spot lights. Directional lights (sun) don't follow this — they're effectively at infinity.

### 6.2 Set Up Dusk Lighting

**Sun light:**
1. `Shift+A → Light → Sun`. Rotation X=80°, Z=45° (low angle). Energy=3 W/m². Color: #FF8C3A.

**Sky (world lighting):**
1. World Properties → Use Nodes.
2. `Shift+A → Texture → Sky Texture`. Type=Nishita, Sun Elevation=5°. Plug → Background Color.

**The Nishita sky model** simulates Rayleigh scattering (blue sky) and Mie scattering (haze). At low sun angles, light travels through more atmosphere → red/orange sunset.

**Rim light:** Area light behind the cliff, color #4A7ABF (cool blue), Energy=20W. Separates dark cliff from dark background — a classic cinematography trick.

### 6.3 The Math: Color Spaces

Colors in Blender live in **Linear** space. Your monitor uses **sRGB** (gamma ≈ 2.2):
```
C_display = C_linear^(1/γ),  γ ≈ 2.2
```

Blender defaults to **Filmic** tone mapping, which handles HDR scenes correctly. Color math is done linearly; display conversion happens last.

---

## Hour 7 — Cameras and Projection

### What You'll Build
A cinematic camera with depth of field. Understand the math that turns 3D into 2D.

---

### 7.1 The Math: The View Pipeline

Every frame, each vertex goes through:
```
World Space → [View Matrix] → Camera Space → [Projection Matrix] → Clip Space → Screen Space
```

**View Matrix:** Transforms world so camera is at origin, looking down -Z. It's the inverse of the camera's world matrix.

**Projection Matrix (perspective):**
```
P = | f/aspect   0         0                        0              |
    |   0         f         0                        0              |
    |   0         0   -(far+near)/(far-near)   -2·far·near/(far-near) |
    |   0         0        -1                        0              |
```

Where `f = 1/tan(FOV/2)`. The perspective divide `(x/w, y/w)` makes distant objects appear smaller.

### 7.2 Field of View and Focal Length

```
FOV = 2 · arctan(sensor_width / (2 · focal_length))
```

Blender default sensor width = 36mm.

| Focal Length | FOV | Character |
|---|---|---|
| 24mm | 84° | Wide, environmental |
| 50mm | 47° | "Natural" human eye |
| 85mm | 29° | Telephoto, compressed depth |

For a game world establishing shot: **24mm**. For dramatic tower close-up: **85mm**.

### 7.3 Add the Camera

1. `Shift+A → Camera`. Press `Numpad 0` to look through it.
2. Frame your shot, then `Ctrl+Alt+Numpad 0` to snap camera to view.
3. Camera Properties: Focal Length=24mm, Clip Start=0.1m, Clip End=500m.
4. Depth of Field: enable, set Focus Distance to tower, F-Stop=2.8.

### 7.4 The Z-Buffer

For every pixel, the GPU keeps a **depth buffer** storing the z-coordinate of the closest fragment. New fragments farther away are discarded.

**Z-fighting** happens when two surfaces are nearly coplanar at distance — caused by limited buffer precision. Fix: increase Clip Start.

---

## Hour 8 — Geometry Nodes and Procedural Generation

### What You'll Build
A procedurally scattered forest and a winding river.

---

### 8.1 The Math: Noise and Randomness

**Noise functions** (Perlin, Simplex, Worley) are smooth, spatially coherent random fields. **Fractal noise** sums multiple frequencies:

```
fractal(x) = Σ (amplitude^i) * noise(x * frequency^i),  i = 0..octaves
```

This is exactly what Blender's **Noise Texture** and **Musgrave Texture** compute.

### 8.2 Geometry Nodes — Scatter Trees

1. Select ground plane → Add **Geometry Nodes** modifier → New.
2. Build:
```
[Group Input] → [Distribute Points on Faces] (Density=0.3, Poisson Disk)
    → [Instance on Points] ← [Object Info: Tree_Simple]
        → [Rotate Instances] ← [Random Value: Z rotation 0–6.28]
            → [Scale Instances] ← [Random Value: 0.7–1.3]
                → [Realize Instances] → [Group Output]
```

**Poisson Disk distribution** ensures points are never closer than a minimum distance — more natural than pure random. Implemented via Bridson's algorithm (O(n) time).

### 8.3 Procedural River

1. `Shift+A → Curve → Bezier Curve`. Shape control points to wind through the valley.
2. Add Geometry Nodes modifier:
```
[Curve Input] → [Resample Curve: 200] → [Curve to Mesh]
    ← Profile: [Curve Circle: radius=3]
        → [Set Material: Water] → [Group Output]
```

**Bezier curve math:**
```
B(t) = (1-t)³P0 + 3(1-t)²tP1 + 3(1-t)t²P2 + t³P3,   t ∈ [0,1]
```

The derivative B'(t) gives the tangent vector — Blender uses this to orient the profile curve correctly along the path.

---

## Hour 9 — Rendering with Cycles

### What You'll Build
A final render with realistic lighting.

---

### 9.1 The Math: Path Tracing

Blender's **Cycles** renderer solves the **rendering equation** (Kajiya 1986):

```
Lo(x, ωo) = Le(x, ωo) + ∫ f(x, ωi, ωo) · Li(x, ωi) · (n̂ · ωi) dωi
```

The integral is approximated via **Monte Carlo sampling**: shoot rays from camera → bounce randomly through scene → average the results. This is one **path**. Average thousands per pixel → convergence. Noise = variance of the estimator, proportional to 1/√N.

### 9.2 Render Settings for M5 Mac

- Engine: **Cycles**, Device: **GPU Compute → Metal**
- Render Samples: **512**, Viewport Samples: **64**
- Denoising: **OpenImageDenoise** (neural denoiser, effectively free quality boost)
- Max Bounces: 4 total for outdoor scenes (diminishing returns beyond that)

### 9.3 Compositing

In the **Compositor** workspace:

- **Glare** (`Filter → Glare`, Type=Fog Glow, Threshold=1.0) — god rays from bright areas
- **Color Balance** — Lift toward blue (shadows), Gain toward orange (highlights). Classic cinematic "teal and orange."
- **Vignette** — Ellipse Mask → Blur → Mix to darken edges.

The Color Balance node implements the **ASC CDL** standard:
```
out = (in × slope + offset) ^ power
```

---

## Hour 10 — Claude MCP Workflow and Game Export

### What You'll Build
Refine with Claude's help, then export for Unity or Godot.

---

### 10.1 The Claude MCP Workflow

With `blender-mcp` running, describe changes in natural language — Claude executes them as Python/bpy scripts.

**Example prompts:**

> *"Select all stone objects and set their material roughness to 0.85."*

> *"Add 20 instances of 'Rock_Small' scattered randomly on the ground plane between -10 and 10, with random Z rotation and 0.5–1.5x scale."*

> *"Create an edge-split modifier on the Tower object, split angle 35 degrees."*

> *"Set up a sun lamp at elevation 8 degrees, azimuth 225 degrees, energy 2.5, color temperature 3200K."*

> *"Bake ambient occlusion to a 2048px texture for the cliff object."*

**Tip:** Ask Claude to explain the script it ran — great way to learn `bpy`.

### 10.2 The Math: Rasterization vs. Path Tracing

Game engines use **rasterization** (not path tracing) for real-time performance:

1. **Vertex shader:** Multiply each vertex by MVP matrix `(Model × View × Projection)` → clip-space position.
2. **Rasterization:** Determine which pixels each triangle covers.
3. **Fragment shader:** Evaluate simplified PBR (GGX + Lambert) per pixel.

Game engines get ~16ms per frame (60fps); Cycles can take minutes. The tradeoff is accuracy for speed.

**Lightmapping:** Bake Cycles' accurate light into textures, use them in the game engine at zero runtime cost. Standard pipeline for static environments.

### 10.3 Export Pipeline

1. Apply transforms: `Ctrl+A → All Transforms`
2. Clean up: **Mesh → Clean Up → Merge by Distance**
3. `File → Export → glTF 2.0 (.glb)`:
   - Format: GLB (single binary file)
   - Include: UVs, Normals, Materials
   - Transform: **Y Up** (Unity/Godot convention)

Drop the `.glb` into Godot or Unity — both auto-import it.

### 10.4 Baking Ambient Occlusion

**The math:**
```
AO(x) = (1/π) ∫_Ω V(x, ω) (n̂ · ω) dω
```
V=1 if direction ω is unoccluded, 0 if blocked. Darkens crevices where light can't reach.

**To bake:** Render Properties → Bake → Type: Ambient Occlusion. Requires a UV map and an unconnected Image Texture node selected in the material. In the game engine, multiply the AO texture with base color for free-looking crevice shadowing.

---

## Math Reference — Quick Lookup

### Vectors
```
Addition:      a + b = (ax+bx, ay+by, az+bz)
Scale:         c·a = (c·ax, c·ay, c·az)
Length:        |a| = √(ax² + ay² + az²)
Normalize:     â = a / |a|
Dot product:   a·b = ax·bx + ay·by + az·bz = |a||b|cosθ
Cross product: a×b = (ay·bz-az·by, az·bx-ax·bz, ax·by-ay·bx)
```

### Transformations (4×4 homogeneous)
```
Translation(tx,ty,tz):  T[0][3]=tx, T[1][3]=ty, T[2][3]=tz, diagonal=1
Scale(sx,sy,sz):        diagonal = sx,sy,sz,1
Rotation Z by θ:        [[cosθ,-sinθ,0,0],[sinθ,cosθ,0,0],[0,0,1,0],[0,0,0,1]]
Compose (S then R):     M = R · S
```

### Rendering
```
Lambert shading:    L = max(0, N̂·L̂) × albedo × light_color
Inverse square:     E = Φ / (4π·d²)
sRGB gamma:         C_display = C_linear^(1/2.2)
Perspective FOV:    FOV = 2·arctan(sensor_width / (2·focal_length))
Bezier curve:       B(t) = (1-t)³P0 + 3(1-t)²tP1 + 3(1-t)t²P2 + t³P3
```

---

## Next Steps

**Blender learning:**
- Blender Guru's Donut tutorial — procedural materials deep dive
- Ian Hubert's Lazy Tutorials — fast, cinematic techniques
- Simon Thommes on Geometry Nodes — procedural generation

**3D Math:**
- *3D Math Primer for Graphics and Game Development* — Dunn & Parberry (free online)
- *Real-Time Rendering* — Akenine-Möller et al
- *Physically Based Rendering* — pbrt.org (free online, the path-tracing bible)

**Game development:**
- Godot 4 docs (free, open source, Python-like GDScript)
- Unity Junior Programmer pathway

**With Claude MCP:** Once comfortable, ask Claude to generate Geometry Nodes networks, write custom GLSL shader nodes, or build Python scripts that procedurally generate entire level layouts. The combination of Blender's visual feedback and Claude's scripting turns you into a one-person production pipeline.

---

*Tutorial version 1.0 — Written for Blender 4.x on Apple Silicon, June 2026.*
