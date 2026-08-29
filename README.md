# Airzooka flow sandbox

An axisymmetric (r–z) incompressible flow simulator for the barrel in
`OpenSCAD/airzooker.scad`, running entirely in WebGL2. Open `index.html` in a
browser — no build step, no server needed.

## Why axisymmetric rather than plain 2D

The device has circular rotational symmetry, so a slice through the axis is
enough — but the *slice must know it is a slice*. A planar 2D solver produces a
counter-rotating vortex **pair**, which propagates and decays quite differently
from a vortex **ring**. This solver works in cylindrical coordinates with the
`1/r` terms carried through the divergence, the pressure Laplacian and the
viscous stress, so rings form, pinch off and travel with roughly the right
dynamics.

## What the model says about the printed design

With the values in the `.scad` file — 25 mm bore, 100 mm barrel, **5 mm exit
radius** — and an 80 mm stroke:

| quantity | value |
|---|---|
| swept volume | 157 mL |
| exit : bore area ratio | 25 : 1 |
| ejected slug length | 2000 mm |
| **slug L/D** | **200** |
| mean exit speed at peak push | ~78 m/s (Mach 0.23) |

A vortex ring rolls up from the shear layer at the orifice lip and **pinches off
at L/D ≈ 4** (the "formation number" — Gharib, Rambod & Shariff 1998). Anything
ejected after that cannot join the ring; it becomes a trailing jet, which mixes
with still air and stalls within a few diameters. At L/D = 200 essentially the
entire shot is trailing jet.

Running it confirms this. At the same instant of flight:

- **5 mm exit:** one long, thin, straight shear tube from the muzzle to the far
  edge of the domain. No pinch-off, no ring, and a large recirculation *inside*
  the barrel as air is dragged back around the piston.
- **18.5 mm exit (L/D ≈ 4):** a compact, detached vortex ring that has separated
  cleanly from its trailing shear layer and is propagating as a coherent blob.

For a 25 mm bore and 80 mm stroke, the exit radius that lands on L/D = 4 is

```
r_exit = cbrt(R_bore² · stroke / 8) = cbrt(25² · 80 / 8) ≈ 18.4 mm
```

i.e. roughly **three quarters of the bore diameter**, not a fifth of it. That is
why commercial airzookas have a mouth almost as wide as the barrel. The sandbox
recomputes this number live as "exit R for L/D=4" whenever you change the bore or
the stroke.

Two secondary effects worth playing with:

- **Piston clearance.** The default 0.5 mm print tolerance opens ~4 % of the bore
  area. Because the pressure behind a small orifice is high, a surprising amount
  of the stroke escapes backwards instead of out of the muzzle.
- **Exit speed.** 78 m/s is Mach 0.23 and rising fast as the orifice shrinks. Past
  Mach 0.3 the incompressible solver is no longer valid, and a real printed nozzle
  would start choking and dumping the energy into heat and noise.

## Solver

- Staggered MAC grid; `u_r` and `u_z` on their own faces, pressure and tracer at
  cell centres. Faces are stored in one `(Nz+1)×(Nr+1)` RGBA32F texture.
- Semi-Lagrangian advection with an RK2 backtrace, sampling each velocity
  component at its own staggered location.
- Pressure projection by Jacobi iteration on the cylindrical Poisson equation,
  with `r`-weighted radial coefficients. The axis coefficient vanishes naturally
  at `r = 0`, so the symmetry boundary needs no special case. Neumann at solids,
  `p = 0` at the open boundaries.
- Explicit viscous diffusion using the axisymmetric Laplacian, including the
  `-u_r/r²` term.
- Smagorinsky sub-grid viscosity, standing in for the turbulent mixing that
  circular symmetry excludes.
- Optional vorticity confinement to counteract numerical diffusion of ring cores.
- Barrel, orifice plate, optional internal taper and the moving piston are
  evaluated analytically per fragment rather than stored as a mask, so geometry
  sliders are exact and free.
- Adaptive timestep from a GPU max-reduction of `|u|`. Readbacks go through a
  pixel-pack buffer and a fence — a synchronous `readPixels` cost ~10 ms of stall
  per frame against a 0.15 ms solver step.

### Accuracy notes

Jacobi converges slowly and this problem is strongly constrained by the piston.
A convergence check at 15 ms of flight (64 radial cells, 5 mm exit):

| pressure iterations | peak &#124;u&#124; |
|---|---|
| 40 | 72.6 m/s |
| 160 | 82.8 m/s |
| 640 | 84.9 m/s |

The default of 120 sits within a few percent of converged. Raise it if a result
looks marginal.

## Backends

The simulator runs on **WebGPU compute** where it is available and falls back to the original
**WebGL2 fragment-shader** path otherwise. The HUD names the live backend. Both produce
**bit-identical** results: driven with the same timestep sequence for 500 steps, the two
paths agreed to a maximum absolute difference of exactly 0.

## Performance

The WebGL2 path is **render-pass bound, not pixel bound**, which is unintuitive and worth
knowing. Measured on an Apple M1 through ANGLE's Metal backend:

| per pass | µs |
|---|---|
| shading, same framebuffer | 3.8 |
| + framebuffer switch | ~65 |
| + texture rebind | ~93 |

The switch cost is essentially independent of texture size and format — R32F, RGBA8 and a
64×64 target all land within a factor of 1.5. So cost is **steps × (iterations + 6) × ~70 µs**,
and the pixel count barely enters. Two consequences:

- **Halving the resolution does not quarter the run time.** It only helps by allowing a larger
  timestep, so the gain is roughly linear, not quadratic. This surprises people.
- **Pressure iterations are the main dial**, since each one is a whole render pass.

A WebGPU **compute dispatch costs ~4 µs against that ~65 µs**, which is the entire reason for
the second backend.

### What was worth doing

Common to both backends:

- Solid-boundary enforcement folded into the `forces` and `proj` kernels, removing two passes
  per step.
- Fewer pressure iterations, paid for with a smaller CFL number. At 15 ms of flight,
  `cfl 0.5 / 30 iterations` reads **83.6** m/s against `cfl 1.0 / 120 iterations` at **79.5** —
  better *and* 36 % fewer passes.
- **Axial cell stretching.** The jet is long and thin and the timestep is set by the axial
  velocity, so axial cells can be ~2× the radial size: peak 42.6 → 43.0 m/s for 1.7× the speed.
- A **directional CFL**, `dt = cfl / (u_z/dz + u_r/dr)`, without which stretching buys nothing.
- Steps per frame **auto-paced** to a frame-time budget.

WebGL2 only: `invalidateFramebuffer` before each pass, worth ~16 %.

WebGPU only:

- Every dispatch for every step of a frame goes into **one command encoder, submitted once**,
  with per-step uniforms addressed by dynamic offset.
- A `prepare` kernel folds geometry, boundary conditions and `1/den` into four coefficients
  and a scaled right-hand side, once per step. The Jacobi inner loop then costs one `dot`
  instead of five `solidAt` evaluations — **32.5 → 14.4 µs per iteration**.

### A tiling optimisation that did not work

The obvious WebGPU win looks like workgroup shared memory: load a tile plus halo, run several
Jacobi iterations with barriers, write once, and cut dispatches fourfold. Built and measured,
it was **1.69× slower** than the naive one-iteration-per-dispatch kernel (17.1 vs 10.1 µs per
iteration).

The reasoning was wrong because a dispatch costs only ~4 µs here, so there is nothing to
amortise, while the halo makes each workgroup compute 2.25× the cells it keeps. The simple
kernel is memory-bandwidth bound, which is the right place to be. It was deleted.

### Where it ended up

Simulating the as-printed 5 mm case:

| | ms of flight per second | a 150 ms shot |
|---|---|---|
| original WebGL2 settings | 0.55 | 272 s |
| tuned WebGL2, Balanced | 2.5 | 60 s |
| **WebGPU, Balanced** | **16.0** | **9.4 s** |
| **WebGPU, Draft** | **63.2** | **2.4 s** |
| a ring design (18.5 mm exit) | faster than real time | instant |

**29× on the default, 115× on Draft.** The small orifice remains the expensive case, because
the timestep scales with `1/u_max` and `u_max` scales with `1/r_exit²`. Any design worth
printing runs effectively instantly.

Below about 30 pressure iterations the solve stops enforcing mass conservation and diverges
outright; the slider is clamped there and a guard catches it.



## Accuracy: what is *not* converged

Worth being blunt about. Re-running the same case at 64, 112 and 160 radial cells:

| radial cells | 5 mm exit, peak at target | 18.5 mm exit, peak at target |
|---|---|---|
| 64 | 93.7 m/s | 6.8 m/s |
| 112 | 42.6 m/s | 5.4 m/s |
| 160 | 104.6 m/s | 9.0 m/s |

That is **not monotonic and not converged** — a factor of two, wandering. The shear layer off
a sharp orifice lip is chaotic at these resolutions, and the result depends on where the grid
happens to cut it.

So: the **analytic** numbers (L/D, area ratio, exit speed, the optimal exit radius) are exact
and are what the design conclusion rests on. The **structural** result — a clean detached ring
versus a straight shear tube that never pinches off — is robust and reproduces at every
resolution. The **measured velocities at the target plane are good to about a factor of two**
and should only ever be used to rank designs, never quoted.

## Colour legend

Every field is drawn with a labelled legend in the corner of the viewport: the colour ramp,
the quantity, its units and the numeric range. The scale is whatever the shader actually used
that frame, including the colour-gain slider, so it never drifts from what is on screen.

| field | units |
|---|---|
| tracer | fraction, 0–1 |
| speed \|u\| | m/s |
| vorticity ω_θ | 1/ms |
| pressure | Pa relative to ambient (compressible) or m²/s² kinematic (incompressible) |
| axial velocity u_z | m/s |
| Mach number | dimensionless, with a white contour at M = 1 |
| schlieren | relative \|∇ρ\| |

The two pressure fields genuinely differ: the incompressible solver works at ρ = 1 and its
pressure is kinematic, so labelling both "Pa" would have been wrong.

## Video export

Exports a WebM to disk. The interactive solver chooses its timestep adaptively, which is right
on screen and wrong for video — frames would represent unequal slices of time and the motion
would be subtly, invisibly wrong. **Export therefore runs on a fixed schedule: every video
frame advances exactly the same simulated interval**, subdivided into as many equal sub-steps
as stability requires. The sub-step count varies between frames; the frame interval never
does. Verified: 30 exported frames, all frame intervals exactly 0.2000000 ms.

**The colour scale is fixed for the whole clip.** On screen the range tracks the flow, which
is what you want while exploring; in a video it means a colour does not signify the same thing
from one frame to the next, and the numbers under the legend visibly crawl. Export therefore
runs the shot once first to find the peak, fixes the range from that, then records. It costs
roughly double, and can be turned off — in which case the range is frozen at whatever the
current frame shows, still constant but probably badly chosen.

Frames carry a burned-in clock, the configuration, the colour legend and a physical scale bar.
These sit in bands **above and below** the image rather than on top of it, so nothing obscures
the flow and no semi-transparent backing panel is needed — the earlier version drew a
fixed-width panel that longer captions overflowed.

Set flight time to record, flight time per frame, frame rate, width and bitrate; the panel
shows the resulting frame count, video length and slow-motion factor. Export settings are
deliberately **not** in the shareable link — they describe the recording, not the physics.

Two implementation notes worth keeping:

- **WebCodecs, not MediaRecorder.** `MediaRecorder` timestamps frames by wall clock, so a
  simulation slower or faster than real time comes out stretched. `VideoEncoder` takes an
  explicit timestamp per frame, which decouples the video's time base from how long the
  computation took. WebCodecs emits encoded chunks and no container, so there is a small
  Matroska/WebM muxer here: one video track, SimpleBlocks in Clusters.
- **Rendering to an owned texture, not the canvas.** A WebGPU canvas does not present until a
  task boundary, so `drawImage()` immediately after `submit()` returns a blank frame — the
  first version of this exported 30 perfectly-timed black frames. Export renders into its own
  texture and copies it out, which is deterministic and works at any resolution without
  disturbing the visible canvas.

## Shareable links

Every setting is serialised into the URL fragment, so a link reproduces a configuration
exactly. **Copy shareable link** puts it on the clipboard; the address bar updates live as you
move sliders.

On length — it is not close to a problem:

| | characters |
|---|---|
| worst case, all 27 settings non-default | 153 params, 186 full URL |
| a typical tweak (`#er=18.5&tp=25&fd=2`) | 18 |
| everything at defaults | 0 — no fragment at all |

Against a commonly-cited safe limit of 2000 characters that is 10× headroom, and Chrome's own
limit is 32779. Three choices keep it that short:

- **Only non-defaults are written.** Most links carry a handful of keys.
- **Two-character keys** (`br`, `er`, `tp`, …).
- **The fragment, not the query string.** The fragment is never sent to a server, so no
  server request-line limit (Apache 8190, nginx 8192) applies at all — and assigning
  `location.hash` works on `file://` URLs, where Chrome throws on `history.replaceState`.

Back and forward work properly: discrete actions (a preset, a model change, resetting to
defaults) push a history entry, while dragging a slider replaces the current one so the
history does not fill with intermediate values. Navigating restores the configuration **and
restarts the shot** — otherwise you would be watching one geometry's flow inside another
geometry's barrel.

Links are validated on the way in: values are clamped to their slider range, unknown keys and
unparseable numbers are ignored. A hand-edited `#br=99999&er=-50&rs=abc&md=77` loads as a
valid configuration rather than breaking.

Round-tripping is tested: encoding a fully non-default configuration, resetting everything,
then decoding restores all 27 settings with zero mismatches.

## Two physical models

The sandbox carries **two solvers** and picks between them on the analytic exit Mach number.
Neither is right everywhere, and pretending otherwise was the previous version's mistake.

| | incompressible | compressible |
|---|---|---|
| equations | constant density, pressure projection | Navier–Stokes, conserved (ρ, ρu, E) |
| numerics | MAC grid, semi-Lagrangian, Jacobi Poisson | finite volume, HLLC + MUSCL, SSP-RK2 |
| acoustics | none — infinite sound speed | resolved |
| valid | below about Mach 0.3 | any speed |
| speed at Mach 0.02 | 16 ms of flight per second | 2.8 |

Below Mach 0.3 the incompressible model is not a shortcut, it is the correct limit: density
varies by under 5 %, and it is ~6× faster because it does not have to resolve sound waves the
flow does not care about. Above Mach 0.3 it is not approximate but inapplicable. The
compressible solver is right everywhere but pays a timestep set by the sound speed, so at
Mach 0.02 it takes about 50× more steps than the flow itself needs.

**Auto** picks incompressible below Mach 0.25 and compressible above it. Both can be forced.

## Compressible solver: validation

Not "it runs" — checked against exact solutions.

**Sod shock tube**, against the exact Riemann solution, 601 cells:

| | L1 error |
|---|---|
| density | 0.17 % |
| velocity | 0.18 % |
| pressure | 0.09 % |

with **zero overshoot** at every resolution tested, confirming the minmod limiter is TVD.
Grid convergence in L1 was 0.88 and 0.80 over successive doublings — first order, which is the
correct and expected rate for a solution containing discontinuities regardless of the scheme's
formal second-order accuracy.

**Speed of sound**: a 1 % Gaussian pressure pulse in still air propagated at **344 m/s**
against a theoretical 343.1 m/s — 0.26 % error. Total mass drifted by 4×10⁻⁴ %. This is the
quantity the incompressible model gets infinitely wrong.

**Choking**, discharging a reservoir through the 5 mm orifice at rising pressure ratios:

| p₀/p_amb | throat Mach | ṁ / (A·p₀) |
|---|---|---|
| 1.2 | 0.45 | 1.25e-3 |
| 1.5 | 0.70 | 1.73e-3 |
| 1.893 (critical) | 0.91 | 1.95e-3 |
| 3 | 1.14 | **2.057e-3** |
| 6 | 1.20 | **2.061e-3** |
| 12 | 1.21 | **2.059e-3** |

The mass flow **saturates** above the critical ratio — the flow chokes and stops responding to
pressure. The plateau sits at 87 % of the ideal 2.361e-3, which is the discharge coefficient
of a sharp-edged short-tube orifice: a real vena-contracta effect, not an error.

**Low Mach**: the vortex ring still forms and propagates at Mach 0.02, confirming the
Thornber low-Mach reconstruction fix is doing its job. Without it, upwind dissipation growing
as 1/M would have dissolved the ring — the very thing the tool exists to study.

## A correction to an earlier claim

An earlier version of this file said a converging nozzle "cannot exceed 313 m/s". That
conflated two different things and was too strong.

- Sonic velocity caps the **throat**. That part is right, and the choking table above
  demonstrates it.
- The **free jet downstream** of an underexpanded orifice can and does go supersonic, through
  a Prandtl–Meyer expansion at the lip. The real ceiling on the plume is `sqrt(2 cp T0)` at the
  **stagnation** temperature — and T0 rises when the gas is compressed.

Running the 250 mm/2 ms/12.5 mm-taper case properly shows why this matters. The compressible
solver finds a peak barrel pressure of **33.3 bar**, which matches the isentropic compression
ratio of that stroke (250/21)^1.4 = 32.6 almost exactly. That raises stagnation temperature to
~777 K, whose thermodynamic ceiling is 1250 m/s. The solver reports a peak of **1200 m/s** —
just under it, and therefore physical. The incompressible solver reported 2416 m/s, nearly
twice the ceiling.

So the 676 m/s reading was not impossible in principle; it needed a very large pressure ratio,
and those settings do produce one. What makes the configuration unbuildable is the 33 bar: the
piston would be pushing against about 6.5 kN, and the printed barrel would burst long first.

## Compressibility: a hard limit for the incompressible model

This solver is **incompressible by construction**. Density is constant, and the pressure
Poisson solve propagates information across the whole domain instantaneously — an infinite
speed of sound. It therefore has no acoustic waves, no shocks and no choking. Above roughly
**Mach 0.3 it is not approximate, it is inapplicable**, and it will still hand you a
confident-looking number.

Reference points for air at 20 °C, none of them simulated:

| | m/s |
|---|---|
| ambient speed of sound | 343 |
| **sonic throat — hard ceiling for a converging nozzle** | **313** |
| perfect de Laval nozzle expanding to vacuum | 767 |

The 313 m/s figure is the one that matters. A tapered barrel is a *converging* nozzle, and a
converging nozzle **cannot** produce supersonic flow at any driving pressure: it chokes at
Mach 1 at the throat and the mass flow stops responding. Exceeding it requires a
converging–diverging (de Laval) nozzle, and even a perfect one exhausting into vacuum caps at
767 m/s. Any reading above ~313 m/s from a converging geometry is a modelling artefact.

The constant-density assumption fails just as hard: real air is 0.63× stagnation density at
Mach 1 and 0.23× at Mach 2, so the solver's mass flux is out by 1.6× and 4.3× respectively.

### The guard

Whenever the analytic exit speed or the measured peak exceeds Mach 0.3, the metrics panel
puts a red banner **above** the numbers and strikes through the measured block. A separate
check flags drive settings that are unbuildable regardless of the flow — for example a
229 mm stroke in 2 ms needs the piston itself to reach 180 m/s (Mach 0.52) at about
28,800 g, which nothing hand- or spring-driven will do.

Making this regime *correct* would mean a different solver: compressible Euler or
Navier–Stokes with a density field, an energy equation and a shock-capturing scheme. That is
a rewrite, not a setting — and it is unnecessary for the actual design question, which lives
comfortably below Mach 0.05.

## Not modelled

The flange, handle, greebles and screw holes, since they are outside the flow.
The back of the barrel is treated as open so air can refill behind the piston.
Compressibility, and any elastic-diaphragm drive.
