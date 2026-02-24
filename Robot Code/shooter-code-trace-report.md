# Shooter Code Trace Report (Starting at `Teleop.vi`)

## Scope and method
- Requested scope: start from `Robot Code/Teleop.vi` and trace shooter behavior.
- `pylabview` was not available in this environment and could not be installed (network-restricted session), so this report is based on static VI-binary string extraction and project metadata.
- Confidence is high for call relationships (referenced subVIs/type defs), and medium for exact branch logic/button conditions.

## Top-level call path from teleop
From `Robot Code/Teleop.vi`, the shooter branch is anchored by:
- `Shoot Teleop Handler.vi`
- `Shooter Operations.ctl`
- `WPI_JoystickGetValues.vi`
- `Simple Edge Detection.vi`
- `NT Read Value.vi` / `NT Read Number.vi` / `NT Read Numeric Array.vi`

`Teleop.vi` also references other handlers (`Drive`, `Intake`, `Climb`), indicating shooter control is one subsystem branch within a larger teleop cycle.

## Shooter teleop handler responsibilities
`Robot Code/Support Code/Teleop Handlers/Shoot Teleop Handler.vi` references:
- Shot generation:
  - `Shoot Fuel.vi`
  - `Choose Manual Shot Type.vi`
  - `Shooter Immediate.vi`
  - `Hood Manual.vi`
  - `Reset Hood.vi`
- Ball transport coordination:
  - `Hopper Immediate.vi`
  - `Transfer Immediate.vi`
  - `Reserve Hopper.vi`
  - `Reserve Transfer.vi`
- Input/state helpers:
  - `WPI_JoystickGetValues.vi`
  - `Simple Edge Detection.vi`
  - `XBox Buttons Indexer.ctl` (external dependency)
  - `XBox Axes Indexer.ctl` (external dependency)

Interpretation:
- This VI appears to be the teleop shooting orchestrator.
- It handles both manual and computed shots, and coordinates shooter + feeder systems (hopper/transfer) in one place.

## Computed shot path (ballistics)
`Robot Code/Support Code/Ballistic Utils/Shoot Fuel.vi` references:
- `Distance to Hub.vi`
- `Compute LUT.vi`

`Robot Code/Support Code/Ballistic Utils/Compute LUT.vi` references:
- `Get Shooter LUT.vi`
- `Get Hood LUT.vi`
- `Get Distance LUT.vi`
- `Polynomial Interpolation.vi`

`Robot Code/Support Code/Distance to Hub.vi` references:
- `Get Hub Location.vi`
- NetworkTables read VIs (`NT Read Value.vi`, `NT Read Number.vi`, `NT Read Float Array.vi`)

Interpretation:
- Auto-shot setpoints are likely generated from robot-to-hub distance and one or more lookup tables.
- `Shoot Fuel.vi` is likely the handoff point from vision/pose distance to shooter/hood commands.

## Manual shot path
Manual shot support is indicated by:
- `Choose Manual Shot Type.vi`
- `Hood Manual.vi`
- `Reset Hood.vi`

Interpretation:
- Operator likely selects one of several manual shot profiles and can directly command hood behavior (adjust/reset) independent of auto LUT output.

## Command interface into shooter subsystem
Shooter command VIs all reference common command infrastructure:
- `Robot Code/Shooter/Commands/Shooter Immediate.vi`
- `Robot Code/Shooter/Commands/Reserve Shooter.vi`
- `Robot Code/Shooter/Commands/Read Shooter Operation.vi`

Common dependencies across those VIs:
- `Shooter Command Helper.vi`
- `Shooter Operations.ctl`
- `Shooter Setpoints.ctl`
- `Command Status Info.ctl`
- `Completion Notifier.ctl`
- `Completion Status.ctl`

Interpretation:
- This subsystem uses a standard command queue/status pattern.
- `Immediate` and `Reserve` likely map to immediate execution vs. queued reservation semantics.

## Shooter controller runtime (hardware-facing)
`Robot Code/Shooter/Implementation/Shooter Controller.vi` references:
- Command/state plumbing:
  - `Read Shooter Operation.vi`
  - `Shooter Check for New Command.vi`
  - `Shooter Command Helper.vi`
  - `Shooter Controller Initialization.vi`
  - `Shooter Controller Interactive Check.vi`
  - `Shooter Published Globals.vi`
- Hardware control stack (REV Spark MAX):
  - `SPARK Open Brushless.vi`
  - `SPARK Configure Basic.vi`
  - `SPARK Configure Closed Loop.vi`
  - `SPARK Configure Closed Loop Gains.vi`
  - `SPARK Configure Closed Loop Output Range.vi`
  - `SPARK Configure Power Current Limit.vi`
  - `SPARK Configure Soft Limit.vi`
  - `SPARK Get Encoder Status.vi`
  - `SPARK Set Output Advanced.vi`
  - `SPARK Set Sensor Position.vi`
- NetworkTables I/O:
  - `NT Read Value.vi`
  - `NT Read Float Array.vi`
  - `NT Write Value.vi`
  - `NT Write Boolean.vi`

Interpretation:
- Shooter controller runs a command-consumption loop, configures Spark MAX closed-loop behavior, reads encoder feedback, and writes telemetry/flags to NT.
- Closed-loop velocity/position behavior is highly likely, given Spark closed-loop and encoder usage.

## Cross-subsystem feed coordination during shooting
Because shooter teleop references both immediate and reserve commands for hopper and transfer:
- `Hopper Immediate.vi` / `Reserve Hopper.vi`
- `Transfer Immediate.vi` / `Reserve Transfer.vi`

Interpretation:
- Firing is not just flywheel command; it includes explicit ball-path gating through transfer and hopper.
- The reserve pattern suggests protection against command races and subsystem contention.

## Architecture summary (from teleop to actuator)
1. `Teleop.vi` invokes `Shoot Teleop Handler.vi` each teleop cycle.
2. Shoot handler reads joystick inputs and edge events.
3. Shot intent flows through either:
   - auto/LUT path: `Shoot Fuel.vi` -> `Distance to Hub.vi` + `Compute LUT.vi`, or
   - manual path: `Choose Manual Shot Type.vi` + hood manual/reset VIs.
4. Shooter commands are emitted through command helper pattern (`Shooter Immediate/Reserve`).
5. `Shooter Controller.vi` consumes operations/setpoints, applies Spark MAX closed-loop outputs, and publishes status/telemetry.
6. Hopper/Transfer commands run alongside shooter commands to feed balls during qualified shot states.

## Risks and unknowns
- Exact button mappings were not recoverable here because `XBox Buttons Indexer.ctl` and `XBox Axes Indexer.ctl` are external dependencies not present in the repo.
- Exact enum members inside `Shooter Operations.ctl` are not fully readable from raw binary strings.
- Exact conditional logic (thresholds, timing, sequencing guards) requires VI block-diagram decode via a full LabVIEW parser/runtime.

## Files traced in this report
- `Robot Code/Teleop.vi`
- `Robot Code/Support Code/Teleop Handlers/Shoot Teleop Handler.vi`
- `Robot Code/Support Code/Ballistic Utils/Shoot Fuel.vi`
- `Robot Code/Support Code/Ballistic Utils/Compute LUT.vi`
- `Robot Code/Support Code/Ballistic Utils/Get Shooter LUT.vi`
- `Robot Code/Support Code/Distance to Hub.vi`
- `Robot Code/Shooter/Commands/Shooter Immediate.vi`
- `Robot Code/Shooter/Commands/Reserve Shooter.vi`
- `Robot Code/Shooter/Commands/Read Shooter Operation.vi`
- `Robot Code/Shooter/Commands/Hood Manual.vi`
- `Robot Code/Shooter/Commands/Reset Hood.vi`
- `Robot Code/Shooter/Implementation/Shooter Controller.vi`
- `Robot Code/Shooter/Implementation/Infrastructure/Shooter Check for New Command.vi`
- `Robot Code/Shooter/Implementation/Infrastructure/Shooter Command Helper.vi`
- `Robot Code/Shooter/Implementation/Infrastructure/Shooter Controller Initialization.vi`
- `Robot Code/Shooter/Implementation/Shooter Published Globals.vi`
- `Robot Code/Shooter/Implementation/Shooter Operations.ctl`
- `Robot Code/Shooter/Implementation/Shooter Setpoints.ctl`
- `Robot Code/Hopper/Commands/Hopper Immediate.vi`
- `Robot Code/Transfer/Commands/Transfer Immediate.vi`
- `Robot Code/RebuiltApprentice.lvproj`
