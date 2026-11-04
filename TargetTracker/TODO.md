# TODO

- Check possible schedule discrepancies.
In general, it may be tricky to be in the right schedule order, depending on what is tracked.
**Information may be outdated by one unit of track_every.**

    - Check the bins (`POSTSTEP` vs `ANALYSIS`)
    - Check the `BEFORE/AFTER` conditions

- CMake sure `sf_valid` and `sf_active` are set correctly and consistently.

    - It may be a bit tricky, depending on when the thorn is executed,
    especially if the surface is also tracked by another thorn or another target.
    - Check the case of the surface following mode.

**/!\ Both these points are related to the following problem:**

## Issue with sf_active and sf_valid

### Encountered problem

_Context ("HBH a2"): Hairy BH run, where a lump of matter separates from the original horizon,
collapses and forms a second horizon. This is one of the motivations to be able
to track a local CoM, transitioning into a horizon._

For HBH a2, the run was checkpointed at t=330, at a time where the second mesh center tracks the local CoM of matter, but the second horizon hasn't formed yet.

```python
##### Relevant excepts from par file #####

# Target tracking
#------------------------------------------------------------------------------

# Track matter CoM to follow the collapse into second BH
CarpetTracker::surface[1]                       = 1

# Auxi tracker for initial times: opposite of BH
TargetTracker::track                      [0] = yes
TargetTracker::which_surface_to_store_info[0] = 1
TargetTracker::target_x                   [0] = "PunctureTracker::pt_loc_x[0]"
TargetTracker::target_y                   [0] = "PunctureTracker::pt_loc_y[0]"
# TargetTracker::target_z                   [0] = "PunctureTracker::pt_loc_z[0]"
TargetTracker::track_every                [0] = 512/4 # Same as AH
TargetTracker::track_opposite             [0] = yes

# Switch activation of trackers 0 and 1 with Triggers

TargetTracker::track                      [1] = no
TargetTracker::which_surface_to_store_info[1] = 1
TargetTracker::target_x                   [1] = "TargetTracker::target_com_x[1]"
TargetTracker::target_y                   [1] = "TargetTracker::target_com_y[1]"
# TargetTracker::target_z                   [1] = "TargetTracker::target_com_z[1]"
TargetTracker::track_every                [1] = 512/4 # Same as AH
TargetTracker::compute_com                [1] = yes
TargetTracker::com_every                  [1] = 512/4 # Same as AH, should be OK with restricted radius and refinement levels
TargetTracker::com_compute_inside_radius  [1] = 2.5

# Horizons
#------------------------------------------------------------------------------

AHFinderDirect::find_every = 512/4

AHFinderDirect::run_at_CCTK_POST_RECOVER_VARIABLES = no # <-- Not steerable...

AHFinderDirect::move_origins            = yes
AHFinderDirect::reshape_while_moving    = yes
AHFinderDirect::predict_origin_movement = yes

AHFinderDirect::initial_guess__coord_sphere__radius  [2] = 0.2
AHFinderDirect::shiftout_factor                      [2] = 1.5   # expand surface guess
AHFinderDirect::which_surface_to_store_info          [2] = 1
AHFinderDirect::reset_horizon_after_not_finding      [2] = no
AHFinderDirect::track_origin_from_grid_scalar        [2] = yes
AHFinderDirect::track_origin_source_x                [2] = "TargetTracker::target_loc_x[1]"
AHFinderDirect::track_origin_source_y                [2] = "TargetTracker::target_loc_y[1]"
AHFinderDirect::track_origin_source_z                [2] = "TargetTracker::target_loc_z[1]"
AHFinderDirect::find_after_individual_time           [2] = 300.0

# QuasiLocalMeasures
#-------------------------------------------------------------------------------

QuasiLocalMeasures::surface_index      [1] = 1
```

**Problem:** _(With the current state of the code and the ET)_ 
\
At recovery, `sf_valid[1] == 1` in the checkpoint because it was last changed by `TargetTracker`.
Since `AHFinderDirect` is not re-run at recovery, `sf_valid[1]` is not changed and remains 1.
This conflicts with `QuasiLocalMeasures` -- which "coincidentally" runs between `AHFinderDirect` and `TargetTracker` --, because it checks `sf_valid <= 0` to skip a surface.
Entering `QLM` without radius/origin information for the surface leads to a crash (in `llama/Coordinates`).

### Why do we set sf_valid and sf_active?

- `CarpetMask` checks `if(sf_active)`
- `CarpetTracker` checks `if(sf_active)` and `if(sf_valid > 0)`

### Possibilities within current state of the code

- **_Not possible:_** use `PunctureTracker`. Parameters are not steerable, doesn't quite work to have it initially when there is no puncture, and the problem of the initial location remains.

- **Helper: Terminate when second horizon is found**, and steer everything by hand in the par file, while having surfaces separated.
Namely, steer `AHFinderDirect::track_origin_source_x/y/z`.

- **Separate tracking surfaces and horizons**.
Have surface 1 for `CarpetTracker` and surface 2 for `AHFinderDirect`.
Have tracker 1 track surface 1 (only centroid), and tracker 2 track surface 2 (with all surface info, fed to `QLM`).
\
**_Conceptual problem:_** The entanglement shows back again eventually, because physically these surfaces are the same.
1. When the second horizon is formed, trigger-steer `TargetTracker::target_x/y/z[1]` to surface 2's centroid (2-3 triggers).
It continues to feed surface 1 for `CarpetTracker`, and surface 2 for `AHF`'s tracked origin.
It's like an unwinded "surface to tracker mode".
2. With fairly minimal changes to `TargetTracker`, we could have `TargetTracker::which_surface_to_store_info` be always steerable, which slightly simplifies the previous point.
3. Deactivate `AHF`'s tracking and just let it follow and set surface 2 by itself, and change `CarpetTracker` to surface 2 --> breaks the separation.

### Remedies and modifications?

- About the schedule:
1. Have a `SphericalSurface_HasBeenSet` group in `ANALYSIS`?
2. Have `TargetTracker` at `POSTSTEP`? The problem is that `Trigger` is run at `ANALYSIS`.

- Use `SphericalSurface::set_spherical`, or embed it in `TargetTracker`, so that the surface has valid information?
\
_This surface is not physical, would "pollute" QLM and possibly `Trigger` checks, for instance (where `NaN` fails comparisons)._

- Change some of the conditions?
1. `QLM`'s `sf_valid <= 0` + more failsafes?
2. Change `CarpetTracker`'s `sf_valid > 0` to `>= 0`, so that 0 could mean "has some info but not all", and `TargetTracker` could set `sf_valid = 0`?

## Other remarks

### Untracked surfaces movement

When a target is deactivated, it stays at the same position (from `TargetTracker::target_loc`).
However, surfaces seem to live their lives and their centroids do not stay at the exact same position,
even if the surface is not tracked by `TargetTracker`.

In my small tests, the movement is small/negligible, but this may impact unit tests.


## Other ideas

- Target by group: cf `NSTracker`, use a whole group instead of x/y/z variables independently.

- Change surface radius from a source

- Thorn `TargetModifier`:
A helper thorn that would modify targets more cleanly.
In particular, it could modify several attributes at the same time and be used in conjunction with `Trigger`, allowing to use one `Trigger` to modify several parameters.

