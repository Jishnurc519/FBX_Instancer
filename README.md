# FBX_Instancer

A TouchDesigner component for rendering large numbers of GPU-instanced, animated FBX/Mixamo characters using the POP (Point Operator) family. Instead of running a separate skeletal animation per character, animation data is baked into textures once and then sampled per-instance on the GPU via a GLSL Copy POP — making it possible to drive hundreds of independently animated, independently colored/scaled/timed instances in real time.

## Requirements

- TouchDesigner **2025.x** with the **POP (Point Operator)** family enabled. POPs are part of TouchDesigner's experimental operator set, so make sure your build has them available before loading the component.

## What it does

- **Bakes skeletal animation to texture** — converts FBX/Mixamo bone transforms into a texture-based representation so animation can be sampled per-vertex/per-instance in a shader, rather than relying on per-character CPU-driven rigs.
- **Assigns per-instance attributes** — each instance gets its own scale, color, animation offset, and playback speed, so a crowd doesn't read as a single mesh copy-pasted everywhere.
- **Validates the import pipeline automatically** — the component continuously checks that the Import SELECTs referencing the FBX source are in sync with each other, so mismatched rigs or stale references get caught instead of silently breaking the bake.
- **Samples and instances via a GLSL Copy POP** — the actual placement/animation lookup happens in a custom GLSL Copy POP, which is what allows this to scale to large instance counts while staying on the GPU.

## Inputs

The component has four physical inputs:

| Input | Purpose |
|---|---|
| **1** | POP input defining what to instance on — the point cloud whose points each become one character instance. |
| **2** | AnimOffset map — supplies the per-instance animation phase offset. |
| **3** | Scale map — supplies the per-instance scale value (combined with the global **Scale** parameter). |
| **4** | Color map — supplies the per-instance color. |

These feed the `AnimOffset`, `ScalerTex`, and `Color` template attributes described above; input 1 is the base point cloud they all get attached to before reaching the GLSL Copy POP.

## Component parameters

Exposed on the `.tox`'s custom parameter page:

- **Importparent** — path to the imported FBX COMP (e.g. `/project1/Walking`). The Import Select operators inside the component watch this and re-sync automatically when it changes.
- **Speed** — global animation playback rate, fed into the bake's `uPercent` phase.
- **Scale** — global instance scale, multiplied against each instance's own `scalerTex` attribute.

## Per-instance template attributes

The GLSL Copy POP reads these from the **template point cloud** you feed it — one point per instance. They need to exist on the template input *and* be registered on the GLSL Copy POP's **Create Attributes** page, or the shader will read garbage/uninitialized values (this will show up as flicker or instability):

| Attribute | Type | Purpose |
|---|---|---|
| `N` | vec3 | Per-instance orientation/normal, used to build the rotation matrix each instance is placed with. |

Any custom uniform you add to the shader (e.g. pre-transform controls) must likewise be added to the GLSL Copy POP's own parameter page — uniforms that exist only in the GLSL code but aren't declared as POP parameters will read uninitialized memory and cause flicker.

## How the deformation works

- Bone transforms are baked **straight from TouchDesigner's evaluated `worldTransform`** per frame, not recomposed manually from translate/rotate/scale channels — this avoids rotation-order and pre-transform mismatches that would otherwise show up as twisted limbs.
- The bake is stored as a texture (`texelFetch`-based lookup, not filtered sampling) so the GLSL Copy POP can pull the correct bone matrix for any instance/frame combination on the GPU.
- At copy time, the shader computes a `SkinnedDeformMat` for the current instance's phase, applies it to the template's base position/normal, then applies per-instance rotation (from `N`), scale (`ScalerTex` × global Scale), and writes the result to `P`/`N`, plus `Color`.

## Setup

1. Drop `FBX_Instancer.tox` into your network.
2. Export your character from Mixamo as FBX **with skin**, import it into TouchDesigner, and set **Importparent** to that imported FBX COMP.
3. The Import Selects re-sync automatically; the deformation bake runs (this can take a few seconds up to roughly a minute depending on bone/frame count — watch the textport for completion).
4. Build a template point cloud (one point per instance) carrying `N`, `AnimOffset`, `ScalerTex`, and `Color`, and feed it into the GLSL Copy POP's template input.
5. Tune **Speed** and **Scale** to taste.

## Notes

- Designed around Mixamo-rigged FBX files (`mixamorig_`-prefixed bones); other rigs may work but haven't been tested.
- POPs are an experimental operator family — exact parameter/function names can shift between TouchDesigner builds. If something that used to compile suddenly doesn't, check whether the build's POP implementation changed before assuming the `.tox` is broken.
- If you see flickering instances, check first for any uniform or attribute used in the shader that isn't also explicitly declared on the GLSL Copy POP's parameter/attribute pages.

## License

MIT — see [LICENSE](LICENSE) for the full text.
