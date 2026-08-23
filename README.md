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

## Not modelled

The flange, handle, greebles and screw holes, since they are outside the flow.
The back of the barrel is treated as open so air can refill behind the piston.
Compressibility, and any elastic-diaphragm drive.
