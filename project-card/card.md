---
oneliner: Side-by-side WebGL demo of seven parallax-mapping shaders, from normal maps to screen-space displacement
tags: [webgl, glsl, shader, parallax-occlusion-mapping, displacement, normal-mapping, silhouette, heightmap, real-time-rendering, stone]
stack: [WebGL2, GLSL, vanilla JS, single-file HTML]
generated: 2026-09-06
commit: c3155fc
placeholder: false
---
Seven ways of faking surface relief on a flat polygon are rendered on the same
height field with a draggable split-screen wipe, so normal mapping, naive
parallax, offset limiting, steep parallax, POM, silhouette POM and screen-space
displacement can be compared under the same camera and light. Geometry (tower,
cube, wall), material, depth, sample count, self-shadowing and a step-cost
heatmap are all live controls, which is what makes the failure cases — grazing
angle smearing, terraced steps, a silhouette that stays a perfect cylinder —
visible on demand. Working and hosted on GitHub Pages; the whole thing is one
index.html plus a base64-embedded photogrammetry scan, no build step.
