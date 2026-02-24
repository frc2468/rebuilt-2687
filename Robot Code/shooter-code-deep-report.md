# Shooter System Deep-Dive Report

## Scope
This report explains the shooter system starting at `Teleop.vi`, then tracing all shooter-related paths into:
- teleop orchestration
- ballistic setpoint generation
- shooter command scheduling
- shooter controller execution
- hopper/transfer feed coordination

## Method and confidence
- Method used: static string/call-reference extraction from VI/CTL binaries and project structure.
- High confidence: module boundaries, cross-VI references, command-framework pattern, Spark MAX dependency, ballistic LUT pipeline.
- Medium confidence: exact button mappings, exact enum member names, exact branch conditions/timing thresholds.

## 1. Top-level execution path
Primary call chain from teleop:
1. `Teleop.vi`
2. `Support Code/Teleop Handlers/Shoot Teleop Handler.vi`
3. One of:
   - shooter command path (`Shooter Immediate.vi`, `Hood Manual.vi`, `Reset Hood.vi`, etc.)
   - ballistic path (`Shoot Fuel.vi` -> `Distance to Hub.vi` + `Compute LUT.vi`)
4. Shooter command helper/infrastructure
5. `Shooter Controller.vi` hardware loop
6. Hopper/transfer immediate or reserve commands for feeding

Observed in `Teleop.vi`:
- `Shoot Teleop Handler.vi`
- joystick I/O (`WPI_JoystickGetValues.vi`, `WPI_JoystickSetOutputs.vi`)
- edge processing (`Simple Edge Detection.vi`)
- network reads (`NT Read Value.vi`, `NT Read Number.vi`, `NT Read Numeric Array.vi`)
- external button/axis typedef references (`XBox Buttons Indexer.ctl`, `XBox Axes Indexer.ctl`)

Interpretation:
- Shooter decisions are made inside a per-cycle teleop loop.
- Operator inputs and NT-driven values are both available at teleop layer.

## 2. Teleop shooter orchestrator
File: `Support Code/Teleop Handlers/Shoot Teleop Handler.vi`

Observed dependencies:
- Shot generation:
  - `Shoot Fuel.vi`
  - `Choose Manual Shot Type.vi`
  - `Shooter Immediate.vi`
  - `Hood Manual.vi`
  - `Reset Hood.vi`
- Feed/transport:
  - `Hopper Immediate.vi`
  - `Transfer Immediate.vi`
  - `Reserve Hopper.vi`
  - `Reserve Transfer.vi`
- Input gating:
  - `WPI_JoystickGetValues.vi`
  - `Simple Edge Detection.vi`
  - Xbox axis/button typedefs

Interpretation:
- This is the central policy VI for shooter behavior in teleop.
- It appears to choose between manual shot profiles and computed/LUT shots.
- It coordinates shooter spinup and feeder path commands in the same decision point.

## 3. Ballistics and setpoint synthesis
### 3.1 `Shoot Fuel.vi`
Observed:
- references `Distance to Hub.vi`
- references `Compute LUT.vi`

Interpretation:
- `Shoot Fuel.vi` is the bridge from robot geometry/sensor state into shooter/hood setpoint outputs.

### 3.2 `Distance to Hub.vi`
Observed:
- references `Get Hub Location.vi`
- references NT readers (`NT Read Value.vi`, `NT Read Number.vi`, `NT Read Float Array.vi`)

Interpretation:
- Distance is likely computed using robot pose plus target/hub pose from NT or shared globals.

### 3.3 `Compute LUT.vi`
Observed:
- references `Get LUT Polymorphic.vi`
- references `Get Shooter LUT.vi`
- references `Get Hood LUT.vi`
- references `Get Distance LUT.vi`
- references `Polynomial Interpolation.vi`

Interpretation:
- Shooter and hood outputs are likely interpolated from empirical tables keyed by distance.

### 3.4 LUT providers
Files:
- `Get LUT Polymorphic.vi`
- `Get Shooter LUT.vi`
- `Get Hood LUT.vi`
- `Get Distance LUT.vi`

Observed:
- polymorphic selector contains string labels for `Distance`, `Shooter`, `Hood`.

Interpretation:
- LUT source selection is encapsulated and reusable.
- `Compute LUT.vi` can request a matched LUT set and interpolate shot values.

### 3.5 Manual shot selection utilities
Files:
- `Choose Manual Shot Type.vi`
- `Shmove Shot Adjust.vi`

Observed:
- `Shmove Shot Adjust.vi` references LUT access VIs.

Interpretation:
- Manual mode may still apply distance/LUT-based trim or offset logic.

## 4. Shooter command API (what teleop calls)
Files:
- `Shooter Immediate.vi`
- `Reserve Shooter.vi`
- `Read Shooter Operation.vi`
- `Interpolated Shot.vi`
- `Hood Manual.vi`
- `Reset Hood.vi`

Common observed dependencies:
- `Shooter Command Helper.vi`
- `Shooter Operations.ctl`
- `Shooter Setpoints.ctl`
- `Command Status Info.ctl`
- `Completion Notifier.ctl`
- `Completion Status.ctl`

Important observed embedded text:
- `Read Shooter Operation.vi` includes: "This returns the most recently scheduled command. This doesn't indicate whether the command finished and is running the default command."
- multiple command VIs include template guidance text for immediate command behavior.

Interpretation:
- Commands are constructed as setpoint+operation packets and submitted through shared helper infrastructure.
- Completion notifier/status supports command synchronization without direct controller coupling.
- `Reserve` vs `Immediate` likely differentiates ownership/queuing semantics.

## 5. Shooter subsystem data model
### 5.1 `Shooter Operations.ctl`
Observed text:
- "This enumeration lists all operating modes that the subsystem supports. You should maintain the first three operations."

Interpretation:
- Operations enum is central dispatch key for controller case handling.
- First three enum values are framework-required defaults (common in this code style).

### 5.2 `Shooter Setpoints.ctl`
Observed text:
- "This typedef contains all command fields for a given subsystem..."
- references `Robot Velocities.ctl` and `Shooter Operations.ctl`

Interpretation:
- Setpoint struct includes operation enum + subsystem-specific numeric fields + possibly duration/flags.
- Shared `Robot Velocities.ctl` indicates trajectory/velocity context can be embedded in commands.

### 5.3 `Shooter Published Globals.vi`
Observed:
- references command status/notifier typedefs + shooter ops/setpoints

Interpretation:
- Global published state likely contains current command, command timing metadata, notifier, and completion status.

## 6. Shooter infrastructure runtime
### 6.1 `Shooter Controller Initialization.vi`
Observed embedded text:
- "called once for the subsystem"
- allocates notifier and combines timing/default command into unified data wire

Interpretation:
- Initializes subsystem state bundle and command framework plumbing.

### 6.2 `Shooter Check for New Command.vi`
Observed embedded text:
- "called for each controller loop iteration"
- checks new commands, updates timing data, notifies on completion

Interpretation:
- Handles command lifecycle transitions and completion signaling.

### 6.3 `Shooter Command Helper.vi`
Observed:
- references `Should Abort Operation.vi`

Interpretation:
- Encapsulates command submit/read/abort policy and likely protects against stale or conflicting command execution.

### 6.4 `Shooter Controller Interactive Check.vi`
Observed:
- interactive execute/abort testing text

Interpretation:
- In-development diagnostics harness for running command cases manually.

## 7. Shooter hardware controller
File: `Shooter/Implementation/Shooter Controller.vi`

Observed framework dependencies:
- `Read Shooter Operation.vi`
- `Shooter Check for New Command.vi`
- `Shooter Command Helper.vi`
- `Shooter Controller Initialization.vi`
- `Shooter Controller Interactive Check.vi`
- NT read/write VIs

Observed Spark MAX stack:
- open/config:
  - `SPARK Open.vi`, `SPARK Open Brushless.vi`
  - `SPARK Configure Basic.vi`
  - `SPARK Configure Power.vi`
  - `SPARK Configure Soft Limit.vi`
  - `SPARK Configure Closed Loop.vi`
  - `SPARK Configure Closed Loop Gains.vi`
  - `SPARK Configure Closed Loop Output Range.vi`
- control/feedback:
  - `SPARK Set Output Advanced.vi`
  - `SPARK Get Encoder Status.vi`
  - `SPARK Get Encoder Status Position.vi`
  - `SPARK Get Encoder Status Velocity.vi`
  - `SPARK Set Sensor Position.vi`
- typedefs:
  - `SPARK PID Gains.ctl`
  - `SPARK Feed Forward Gains.ctl`
  - `SPARK Closed Loop Slot.ctl`
  - `Spark MAX Control Type.ctl`
  - `Spark MAX Idle Mode.ctl`

Interpretation:
- Controller executes operation-driven control with Spark MAX closed-loop support and encoder feedback.
- Both read and write NT hooks imply runtime tuning/telemetry or dashboard integration.

## 8. Feed path integration (hopper/transfer)
Shooter teleop directly calls:
- `Hopper Immediate.vi` / `Reserve Hopper.vi`
- `Transfer Immediate.vi` / `Reserve Transfer.vi`

Observed shared pattern:
- both subsystems use same command status/notifier framework style and immediate/reserve split.

Interpretation:
- Shot firing is coupled to feed subsystem command scheduling.
- Reserve commands likely reduce contention while shooter ramps or while command ownership is held elsewhere.

## 9. Practical behavior model (end-to-end)
Based on observed references, shooter teleop likely behaves as follows:
1. Read joystick/buttons/axes and edge events.
2. Determine shot mode:
   - manual profile path (`Choose Manual Shot Type`, hood manual/reset), or
   - auto interpolated path (`Shoot Fuel` / LUT interpolation).
3. Publish shooter operation + setpoints through command helper (`Immediate` or `Reserve`).
4. In parallel, schedule hopper/transfer feed commands when shot state allows.
5. Shooter controller loop checks for new command, runs Spark MAX control, updates feedback/NT, and signals completion.

## 10. Known unknowns and validation tasks
Unknowns (from binary-only static method):
- exact operation enum members and numeric constants
- exact joystick button/axis mapping
- exact readiness criteria (velocity tolerance, dwell time, abort conditions)
- exact feed gating logic and interlocks

Recommended validation:
1. Open block diagrams for `Shoot Teleop Handler.vi`, `Shoot Fuel.vi`, and `Shooter Controller.vi` in LabVIEW and export screenshots of case structures.
2. Dump typedef enum values for `Shooter Operations.ctl`.
3. Capture NT key names read/written by shooter controller and distance utility.
4. Run in simulation/log mode and correlate command operation transitions with hopper/transfer commands.

## 11. Files covered
- `Teleop.vi`
- `Support Code/Teleop Handlers/Shoot Teleop Handler.vi`
- `Support Code/Ballistic Utils/Shoot Fuel.vi`
- `Support Code/Ballistic Utils/Compute LUT.vi`
- `Support Code/Ballistic Utils/Get LUT Polymorphic.vi`
- `Support Code/Ballistic Utils/Get Shooter LUT.vi`
- `Support Code/Ballistic Utils/Get Hood LUT.vi`
- `Support Code/Ballistic Utils/Get Distance LUT.vi`
- `Support Code/Ballistic Utils/Choose Manual Shot Type.vi`
- `Support Code/Ballistic Utils/Shmove Shot Adjust.vi`
- `Support Code/Distance to Hub.vi`
- `Shooter/Commands/Hood Manual.vi`
- `Shooter/Commands/Interpolated Shot.vi`
- `Shooter/Commands/Read Shooter Operation.vi`
- `Shooter/Commands/Reserve Shooter.vi`
- `Shooter/Commands/Reset Hood.vi`
- `Shooter/Commands/Shooter Immediate.vi`
- `Shooter/Implementation/Infrastructure/Shooter Check for New Command.vi`
- `Shooter/Implementation/Infrastructure/Shooter Command Helper.vi`
- `Shooter/Implementation/Infrastructure/Shooter Controller Initialization.vi`
- `Shooter/Implementation/Infrastructure/Shooter Controller Interactive Check.vi`
- `Shooter/Implementation/Shooter Controller.vi`
- `Shooter/Implementation/Shooter Operations.ctl`
- `Shooter/Implementation/Shooter Published Globals.vi`
- `Shooter/Implementation/Shooter Setpoints.ctl`
- `Hopper/Commands/Hopper Immediate.vi`
- `Hopper/Commands/Reserve Hopper.vi`
- `Transfer/Commands/Transfer Immediate.vi`
- `Transfer/Commands/Reserve Transfer.vi`
