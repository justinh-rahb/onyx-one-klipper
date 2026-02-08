# Markforged Onyx One — Kinematic Reference

## Overview

The Markforged Onyx One uses a **hybrid kinematic** system that doesn't map cleanly to any standard Klipper kinematic type. It combines a CoreXY-style diagonal belt with a Cartesian-style pure Y belt.

This document explains how the kinematics work, why standard `hybrid_corexy` doesn't configure correctly out of the box, and how to use `generic_cartesian` to define the correct transform.

## Belt Routing

The Onyx One has two belts for XY motion:

| Belt | Color | Physical Location | Toolhead Motion (FORCE_MOVE) |
|------|-------|-------------------|------------------------------|
| Belt A | RED | Left side of gantry | X+, Y- (diagonal) |
| Belt B | YELLOW | Right side + crossbar | Y+ (pure Y) |

**Z axis** is an independent leadscrew, standard Cartesian.

## The Transform Problem

### Forward Kinematics (what physically happens)

When you drive each stepper independently:
- **Stepper A** (red belt) moves 1 step → toolhead moves **+1 X, -1 Y**
- **Stepper B** (yellow belt) moves 1 step → toolhead moves **+1 Y**

As a matrix:

```
┌ X ┐   ┌  1  0 ┐   ┌ stepper_a ┐
│   │ = │       │ × │           │
└ Y ┘   └ -1  1 ┘   └ stepper_b ┘
```

So: `X = stepper_a`, `Y = -stepper_a + stepper_b`

### Inverse Kinematics (what Klipper needs)

Klipper's `carriages:` parameter defines the **inverse** transform — given a desired toolhead position, where does each stepper need to be?

Solving the forward equations for stepper positions:

```
stepper_a = X
stepper_b = X + Y
```

As a matrix:

```
┌ stepper_a ┐   ┌ 1  0 ┐   ┌ X ┐
│           │ = │      │ × │   │
└ stepper_b ┘   └ 1  1 ┘   └ Y ┘
```

### The Config

```ini
[stepper a]
carriages: x        # stepper_a = 1*X + 0*Y

[stepper b]
carriages: x+y      # stepper_b = 1*X + 1*Y
```

## Why NOT `x-y` and `y`

This was our first attempt. It defines the **forward** kinematics:
- `x-y` = "when this stepper moves, toolhead goes X+, Y-"
- `y` = "when this stepper moves, toolhead goes Y+"

But `carriages:` wants the **inverse**. Using forward kinematics as the config values causes Klipper to compute motor commands incorrectly. The symptom: pure X commands only drive one motor instead of both, causing the toolhead to drift diagonally.

## Why NOT `hybrid_corexy`

Klipper's built-in `hybrid_corexy` defines:
- `stepper_x` → `corexy_stepper_alloc('-')` → tracks X - Y
- `stepper_y` → `cartesian_stepper_alloc('y')` → tracks Y

This is a valid kinematic for machines where one belt is X-Y coupled and one is pure Y. However:

1. The stepper naming (`stepper_x`, `stepper_y`) is confusing because neither stepper maps directly to one axis
2. The direction of coupling may not match (the Onyx One needed X and X+Y, not X-Y and Y)
3. Debugging requires understanding C-level stepper allocators

`generic_cartesian` lets you define the exact transform in plain config, making it much easier to diagnose and fix.

## Diagnostic Method

If the kinematics seem wrong, use FORCE_MOVE to determine the physical truth:

```
; Enable force_move in printer.cfg:
; [force_move]
; enable_force_move: True

FORCE_MOVE STEPPER="stepper a" DISTANCE=10 VELOCITY=5
; Watch toolhead. Note EXACTLY which direction it moves.

FORCE_MOVE STEPPER="stepper b" DISTANCE=10 VELOCITY=5
; Watch toolhead. Note EXACTLY which direction it moves.
```

The results tell you the forward kinematics. Invert the matrix to get the `carriages:` values.

### Quick reference for common results:

| Stepper A does | Stepper B does | carriages A | carriages B |
|----------------|----------------|-------------|-------------|
| X+, Y- | Y+ | x | x+y |
| X+, Y+ | Y+ | x | -x+y |
| X+ | X+, Y+ | x-y | y |
| X+ | X-, Y+ | x+y | y |

## Common Failure Modes

### "Both motors spin on Y moves"
**This is correct.** In this kinematic, a pure Y move requires both motors. Stepper A must compensate for its Y component.

### "Only one motor spins on X moves, toolhead drifts"
The inverse transform is wrong. You're probably using forward kinematics in `carriages:`. Swap to the inverse.

### "Homing works but coordinated moves are wrong"
Homing works because each axis homes independently. The coupling only matters during coordinated moves. Check the `carriages:` values.

### "Jackhammering / motors fighting"
Two things are wrong simultaneously — usually the `carriages:` sign AND a `dir_pin` inversion. Fix one at a time.

### "Axes seem swapped"
The stepper plugged into driver socket X might actually be belt B. Use STEPPER_BUZZ to verify which physical motor each config section controls.
