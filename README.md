# Parallax mapping vs. real displacement

Games fake surface detail. A brick wall is usually one flat polygon, and the
bumps are an illusion painted on by the shader. There is a family of techniques
for this, they cost wildly different amounts, and they fail in different ways.

This is an interactive demo of that family, side by side on the same surface, so
you can see exactly what each one buys and where each one breaks.

![Parallax occlusion mapping on the left, silhouette POM on the right — the left
limb is a clean curve, the right limb is broken by protruding
stones](docs/hero.jpg)

*Same height field, same camera, same light. Left: the relief is convincing, but
the outline is still a perfect cylinder — the illusion is confined to the inside
of the shape. Right: stones break the outline.*

## Try it

### **[→ Live demo](https://mfagerlund.github.io/parallax-mapping-demo/)**

Or clone the repo and open `index.html` directly — no server, no build step, no
network, no install.

- **Drag** to orbit, **wheel** to zoom.
- **Technique** picks the two shaders being compared; **drag the white handle**
  to move the split.
- **Compare presets** are shortcuts that set both dropdowns to a pairing that
  demonstrates something specific.

## The short version of the problem

You have a flat polygon and a **height map** — a greyscale image saying how far
in or out the real surface would be at each point.

The cheapest trick, **normal mapping**, uses that height map only to bend the
lighting. It looks like relief until the light or camera moves, because nothing
actually moved.

**Parallax mapping** goes further: it shifts the texture lookup based on your
viewing angle, so nearer bumps appear to slide over farther ones. Cheap, and
convincing head-on.

**Parallax occlusion mapping (POM)** walks a ray through the height field to find
the first thing it hits. Now bumps properly hide each other and cast into their
own crevices. This is what most games mean by "parallax."

All of the above share one hard limit: **they cannot change the outline.** The
shape's silhouette is its polygon edge, so relief can only ever happen *inside*
the shape. A cobbled cylinder stays a perfect cylinder in profile.

Beating that requires either **real displaced geometry** — actually subdividing
the mesh and moving vertices, which is what tessellation does — or a per-pixel
trick that discards fragments where the ray escapes the surface entirely, which
is what this demo calls **silhouette POM**.

Quality settings in shipping games routinely swap between these tiers, which is
why a "model quality" slider can change how a wall's edge looks, not just its
texture. Crimson Desert is one public example: it drops tessellation at Low and
[substitutes parallax](https://steamcommunity.com/app/3321460/discussions/0/805721252880897880/).
It runs on Pearl Abyss's proprietary
[BlackSpace Engine](https://80.lv/articles/pearl-abyss-demonstrated-crimson-desert-s-blackspace-engine-tech-advancements),
and nothing here is a claim about its internals.

## The six modes

| # | Mode | What it shows |
|---|---|---|
| 0 | Normal mapping only | Lighting-only relief. Flat the moment anything moves. |
| 1 | Parallax, naive | Divides by the view angle. Explodes at grazing angles. |
| 2 | Parallax + offset limiting | Welsh 2004. One deleted division; enormous improvement. |
| 3 | Steep parallax | Ray march without interpolation. Visible terracing. |
| 4 | Parallax occlusion mapping | Same march, interpolated. The usual production choice. |
| 5 | Silhouette POM | Marches in object space, so rays can miss entirely and cut the outline. |

### Normal mapping vs. POM

![Left half lit as a flat surface, right half with visible depth in the
joints](docs/normal.jpg)

Both halves use the identical normal map. The left only *shades* the relief; the
right displaces the texture lookup, so stones occlude each other and joints gain
real depth. The cheapest large win in the whole ladder.

### Why offset limiting exists

![Left half smeared into liquid streaks, right half solid stone](docs/grazing.jpg)

*Left: naive parallax. Right: offset-limited.* Viewed at a shallow angle, the
naive version divides by a number approaching zero, the texture offset runs away,
and the surface liquefies. The 2004 fix is to simply delete that division — which
is geometrically wrong and looks far better. That single change is the entire
difference between these two halves.

### Why nobody ships steep parallax

![Left half showing terraced steps across the stones, right half
smooth](docs/banding.jpg)

At 10 samples the un-interpolated march quantises the surface into visible
terraces. Interpolating between the last two samples removes them for
approximately no cost.

## What it costs

Measured on an Intel Iris Xe integrated GPU at 960×540, with the object filling
the frame — a deliberate worst case. Extra milliseconds per frame over plain
normal mapping:

| samples | 8 | 16 | 32 | 64 | 128 |
|---|---|---|---|---|---|
| Wall — offset-limited | +0.41 | +0.30 | +0.20 | +0.31 | +0.15 |
| Wall — POM | +0.55 | +0.78 | +1.01 | +1.75 | +2.89 |
| Wall — silhouette POM | +0.97 | +1.27 | +2.02 | +3.50 | +6.30 |
| Cylinder — POM | +2.1 | +2.2 | +2.7 | +3.6 | +6.5 |
| Cylinder — silhouette POM | +3.1 | +4.4 | +7.5 | +10.7 | +19.7 |

Offset limiting is effectively free and does not care about sample count. POM
scales linearly. **Silhouette POM costs roughly 2.2× POM on a flat wall and 3× on
a cylinder**, because rays near the outline cross a long slice of the surface
instead of plunging straight through it.

The **Step-cost heatmap** checkbox shows where the samples actually go:

![Heatmap showing blue on stone faces and red streaks along joints and stone
edges](docs/heat.jpg)

*POM on a wall at a shallow angle, 32 samples.* Blue means the ray hit almost
immediately — the tops of stones. Red means it crossed most of the depth first:
deep joints, and the lee side of every stone edge. Cost is wildly non-uniform, it
follows the height map rather than the screen, and it worsens as the view
flattens.

### The costs this table does not include

If you are deciding whether to use silhouette relief in a real engine, these
usually matter more than the numbers above, and this demo pays none of them:

- **Discarding fragments disables early depth rejection** for the whole draw
  call, often costing more than the ray march itself.
- **Correct depth requires the shader to write depth explicitly.** This demo
  never writes `gl_FragDepth`, so the depth buffer retains the *undisplaced
  hull's* depth — not absent depth, but confidently wrong depth. A silhouetted
  surface would intersect other objects incorrectly, and ambient occlusion, fog
  and depth of field would all read the wrong distance. Writing it disables
  further hardware optimisations.
- **Shadow maps must repeat the entire march**, or the shadow will not match the
  outline the camera sees — multiplied by the number of shadow cascades. This is
  usually where the technique dies in production.
- **Transitions are visible.** Fading the effect out with distance changes the
  object's outline, which reads far worse than fading detail.

Together these are much of the answer to *why do engines tessellate instead.*

### Rough guidance

- **Flat walls** — use POM. A silhouette technique only shows at building corners
  and window reveals, which are cheaper and better as real geometry.
- **Columns and pillars** — use geometry. A 24-sided pillar is 48 triangles;
  paying 3× pixel cost to avoid 48 triangles is a bad trade.
- **Ground** — use POM. Ground is flat, so there is no outline to break. What
  ground stresses is shallow viewing angles, where sample counts explode.

## What this is not

- **Not a new technique.** Everything here dates from 2004–2006 and is well
  documented elsewhere. The demo exists to make the comparison concrete and the
  costs measurable, not to propose anything.
- **Not equivalent to tessellation.** Real displacement creates geometry: it
  shadows correctly, writes correct depth, and survives every rendering pass.
  Mode 5 fakes the outline by discarding pixels, and reaches a similar look
  through a completely different mechanism with different failure modes.
- **Not generalisable as written.** Mode 5's ray march is exact only because the
  shapes here are a cylinder and a slab, which have closed-form intersections. It
  does not extend to arbitrary meshes — published approaches approximate each
  triangle with a curved patch instead.

## Controls worth knowing

**Depth** scales the height map. Push it high and you will reproduce the
"parallax slathered on everything" look that players complain about — the
artifacts are inherent, not a bug in the implementation.

**Number of samples** is the ray-march budget.

**Step view independence** mirrors a CryEngine parameter. At 0 every pixel gets
the same sample count. At 1 the count falls off for surfaces facing you directly,
so shallow-angle pixels — which travel furthest through the height map — keep the
full budget. Consequence worth knowing: at the default of 0.75, a
directly-facing surface gets only a quarter of the slider value.

## Licence and credits

Code is MIT — see [LICENSE](LICENSE).

The default material is a photogrammetry scan of a real rock wall:
**`rock_wall_10`** by Rob Tuytel, [Poly Haven](https://polyhaven.com), CC0. It is
public domain and *not* covered by the MIT grant above.

Two procedural materials are included as alternatives, generated in-page.

## Implementation notes

Skip unless you are reading the source.

**The march is a fixed-step sign sampler, not a robust root finder.** It steps at
a constant interval and tests only whether the field changed sign, then
interpolates between the last two samples. A feature thinner than one step can
sit entirely between two samples and be missed — which on the silhouette shows up
as a spurious notch, since a missed hit becomes a discarded pixel. Lower sample
counts make this more likely. Conservative alternatives exist (distance-bounded
marching, or a min-max mip pyramid) at real cost; this demo takes the simple path
on purpose, and the artifacts are part of what it is showing.

**Geometry.** The surface is modelled at the *top* of the height field, the
standard relief-mapping convention, so the rasterized hull is a correct outer
bound and no inflated shell is needed. A flat wall cannot grow a silhouette
because the rasterizer emits no fragments beyond its polygon edge — relief can
only bite inward.

**Height map precision.** Derived from the scan's 16-bit displacement,
contrast-stretched across its 0.5–99.5 percentile range into 8 bits, a 1.78×
precision gain. Normals are derived from the height map rather than shipping the
scan's own normal map, so both come from the same source data.

They do **not** agree in scale, though, and this is a real wart: the bump
strength is baked once at a fixed constant (`index.html:430`) and never sees the
live `Depth` or `Tiling` uniforms. Push Depth up and the parallax deepens while
the shading normals stay as steep as they were, so the two drift apart. Correct
would be deriving the normal in the shader from the same `uH` the march uses.

Being a baked height map it is also single-valued, so the real wall's undercuts
were flattened at capture time.

**Why the texture is a `.js` file.** Firefox gives every `file://` document a
unique opaque origin, so a sibling `.jpg` counts as cross-origin and tainting
blocks the texture upload. Data URIs stay same-origin. The restriction does not
apply to the hosted version, which is served over https, but it uses the same
data URI so there is only one code path. If `texture-data.js` is missing the demo
falls back to procedural stone and says so in the panel.

**Benchmarking.** `benchAll([{geom:1, mode:4, steps:32}, ...])` from the console.
It pauses the render loop, sizes the buffer once, and sweeps configurations in
alternating order reduced with `min()` — this GPU's clock sags under sustained
load, and a fixed-order sweep charges all the drift to whatever ran last.

**A gotcha worth generalising.** Two separate bugs here were safety clamps
quietly becoming the real value: a step-size floor derived from a far-too-generous
ray extent, and a `max(8.0, …)` sample clamp sitting at the bottom of a slider
that started at 8. In both cases a control appeared to work and did nothing.
Clamps belong outside the range the user can reach.

## Files

```
index.html         the entire demo — markup, CSS, GLSL, JS
texture-data.js    the scanned material, base64-embedded (2.3 MB)
docs/              screenshots used by this README
```
