# WPILib 2027 Migration Plan

Incremental refactors we can finish **during the 2026 season** so the jump to
WPILib 2027 / Systemcore is mostly import + vendordep updates, not architecture
work under kickoff pressure.

**Status:** living checklist. Update checkboxes as work lands.  
**Sources of truth (re-check before acting):**

- [New for 2027](https://docs.wpilib.org/en/2027/docs/yearly-overview/yearly-changelog.html)
- [Systemcore testing notes](https://github.com/wpilibsuite/SystemcoreTesting)
- [Removed features](https://docs.wpilib.org/en/2027/docs/yearly-overview/removed-features.html)
- AdvantageKit / Phoenix / PathPlanner / PhotonVision 2027 release notes when published

---

## Why this matters

2027 is not a normal year bump. The control system moves from **roboRIO →
Systemcore**, Java packages move from `edu.wpi.first` → `org.wpilib`, and several
dashboard / NT3 tools disappear. Vendor libraries (AdvantageKit, Phoenix 6,
PathPlanner, PhotonVision, REV, Studica) must be re-imported for Systemcore.

The VS Code project importer will rewrite many package names automatically.
What it **cannot** fix for us:

1. Custom wrappers tightly coupled to Phoenix / WPILib APIs (`LoggedTalon`, etc.)
2. Hardware assumptions that do not exist on Systemcore (Servo, SPI IMUs, …)
3. Scattered CAN / DIO / field constants that multiply touch points
4. Dead vendordeps and template code that still need compatible 2027 builds
5. Game-year assets (AprilTags, PathPlanner, `ShotCalculator`)

Small refactors now shrink that surface.

---

## Current baseline (2026 REBUILT)

| Area | Today | Migration implication |
|------|--------|------------------------|
| Framework | AdvantageKit `LoggedRobot` + commands v2 | Already constructs in `Robot()` — good. Commands v3 is optional later. |
| Drive / vision | AdvantageKit IO interfaces | Best-protected surfaces; keep them. |
| Mechanisms | `LoggedTalon` / `LoggedDIO` / etc. | Highest vendor API churn risk. |
| Controllers | `CommandXboxController` | Becomes `Gamepad` / `CommandGamepad` in 2027 — isolate usage. |
| Dashboards | Some `SmartDashboard.putData` | **SmartDashboard removed in 2027** — migrate off early. |
| Field layouts | `FieldConstants` vs `VisionConstants` disagree | Fix before game-year swap. |
| Leftovers | REV / Studica / Servo / Limelight / NavX unused | Extra vendordeps to re-validate or delete. |

---

## Working rules

1. **One concern per PR** — mergeable in a practice night; easy to revert.
2. **Behavior-preserving first** — move wiring / APIs without changing robot feel.
3. **Prefer seams over rewrites** — interfaces, constants maps, thin adapters.
4. **Do not adopt Commands v3 mid-season** unless there is clear payoff; plan awareness only.
5. **Do not port to Systemcore hardware APIs early** — wait for stable vendordeps; prep by removing dependencies that Systemcore drops.
6. **Verify after each PR:** `./gradlew build` (and sim smoke when hardware-related).

---

## Phase 0 — Inventory (1 short session)

Goal: know every migration touchpoint before touching code.

- [ ] List every `edu.wpi.first.*` usage category (geometry, units, commands, HID, NT, hardware)
- [ ] List every vendor API surface we call (Phoenix 6, PathPlanner, Photon, AdvantageKit Logger)
- [ ] Confirm which DIO / analog / PWM / I2C devices we actually use on-robot
- [ ] Note Systemcore removals that hit us: **Servo** (`LoggedServo` / `RioServo`), SPI IMUs (NavX path if ever re-enabled), Shuffleboard / SmartDashboard
- [ ] Snapshot vendordep versions in `vendordeps/` into this doc’s “Baseline” section when starting work
- [ ] Assign owners for Phases 1–4

**Done when:** team agrees on priority order below; no code required.

---

## Phase 1 — Shrink the blast radius (low risk, high leverage)

Small cleanups that remove work from import day.

### 1.1 Delete or quarantine unused hardware paths

- [ ] Remove or clearly quarantine unused templates:
  - `util/SparkUtil.java` (+ drop **REVLib** vendordep if nothing else needs it)
  - `GyroIONavX.java` (+ drop **Studica** if unused)
  - `VisionIOLimelight.java` if Limelight is not returning this season
  - `util/LoggedServo/**` (Servo support removed on Systemcore)
  - Dead `batteryTracking` wiring if it stays commented out
- [ ] Keep CANdle / LED path either wired and tested or removed from the compile surface

**Why:** every unused vendordep is another 2027 alpha/beta compatibility wait.

### 1.2 Centralize hardware IDs

- [ ] Create a single `Constants` (or `HardwareMap`) home for CAN IDs, CAN buses (`rio` / `canivore`), DIO, and analog channels
- [ ] Move IDs out of `RobotContainer`, `Indexer`, `Shooter`, `Led`, etc. into that map
- [ ] Document bus + device purpose in comments next to each ID

**Why:** Systemcore multi-bus support and rewiring will touch one file instead of many constructors.

### 1.3 Unify field / AprilTag sources

- [ ] Make `VisionConstants` and `FieldConstants` load the **same** `AprilTagFieldLayout`
- [ ] Single constant for “this year’s field enum” so 2027 game swap is one edit
- [ ] Fix known brittle transforms in `VisionConstants` (e.g. integer division in camera offsets)

**Why:** silent vision/pose mismatch is worse during a year change than a compile error.

### 1.4 Leave SmartDashboard

- [ ] Replace `SmartDashboard.putData` in `Robot.java`, `Hood`, `Turret`, `Flywheel` with AdvantageKit / Elastic / NT4 patterns already used elsewhere
- [ ] Prefer `LoggedNetworkBoolean` / `Logger` / Elastic bindings over Shuffleboard/SDB widgets
- [ ] Grep for `smartdashboard` / `shuffleboard` and drive to zero usages

**Why:** both SmartDashboard and Shuffleboard are **removed in 2027**.

### 1.5 Controller boundary

- [ ] Keep HID usage inside `RobotContainer` (already mostly true)
- [ ] Avoid leaking `XboxController` / `CommandXboxController` types into subsystems or commands — pass `DoubleSupplier` / `BooleanSupplier` / `Trigger` only
- [ ] Optional thin `DriverControls` facade so 2027 `Gamepad` rename is one class

**Why:** 2027 collapses gamepad classes into `Gamepad` / `CommandGamepad`.

**Done when:** unused vendordeps gone or justified; IDs centralized; one field layout; no SmartDashboard; controllers not leaking.

---

## Phase 2 — Protect custom abstractions (medium effort)

These are the files most likely to break when Phoenix / WPILib APIs shift.

### 2.1 Harden `LoggedTalon`

- [ ] Audit `util/LoggedTalon/**` and `PhoenixUtil` for direct use of APIs that churn yearly (config apply helpers, sim hooks, control requests)
- [ ] Ensure every mechanism can run on **NoOpp** and **Sim** without Phoenix (already the bringup story — keep it true)
- [ ] Add a short “Phoenix touch list” comment or README section listing the Phoenix types we depend on
- [ ] Prefer calling our wrapper methods from subsystems over raw `TalonFX` APIs

**Why:** import day should update wrappers once, not every subsystem.

### 2.2 Align mechanism construction with drive/vision seams

Drive/vision use IO interfaces; intake/indexer/shooter mix injection and internal mode switches.

- [ ] Prefer constructing hardware in `RobotContainer` and injecting into subsystems (intake already does this)
- [ ] Push indexer/shooter construction toward the same pattern where cheap
- [ ] Do **not** rewrite everything to full AdvantageKit IO unless it reduces duplication — consistency of *where motors are created* matters more than identical patterns

**Why:** fewer places that construct Phoenix devices → fewer 2027 compile fixes.

### 2.3 Units hygiene

- [ ] Prefer `edu.wpi.first.units` (and later `org.wpilib` units) over `edu.wpi.first.math.util.Units` for new code
- [ ] Incrementally replace `math.util.Units` call sites in high-traffic files (`DriveCommands`, `Module`, `ShotCalculator`, gyro/module IO)

**Why:** units library keeps evolving; less legacy conversion surface.

### 2.4 `Rotation2d` awareness

2027 silently wraps `Rotation2d.getRadians()` / `getDegrees()` / `getRotations()`.

- [ ] Audit continuous / unwrapped angle uses (turret, hood, gyro integration, PID errors)
- [ ] Where unwrapped angles are required, store `double` / `Angle` instead of relying on `Rotation2d` getters
- [ ] Add unit tests or logged invariants for turret/hood wrap behavior if not already covered

**Why:** this is a silent behavioral break — catch it before Systemcore.

**Done when:** Phoenix lives behind wrappers; construction is centralized; continuous angles are explicit; units debt is shrinking.

---

## Phase 3 — Software habits that survive the import

### 3.1 Robot lifecycle

- [ ] Keep all init in `Robot()` / `RobotContainer()` — never reintroduce `robotInit()` (removed in 2027)
- [ ] Treat Test mode as ephemeral: 2027 renames it to **Utility**; avoid hardcoding mode name strings in dashboards/scripts

### 3.2 Commands

- [ ] Stay on Commands v2 for the rest of 2026 unless migrating a leaf feature
- [ ] Avoid APIs removed in 2027: `RamseteCommand`, `SwerveControllerCommand`, `Command.schedule()` (deprecated) — we appear clean; keep it that way
- [ ] Prefer PathPlanner / our drive commands over WPILib swerve controller commands

### 3.3 NetworkTables

- [ ] Stay on NT4 only (AdvantageKit `NT4Publisher` already) — NT3 is gone in 2027
- [ ] Prefer Elastic + AdvantageScope over any remaining classic dashboards

### 3.4 Simulation & replay

- [ ] Keep REAL / SIM / REPLAY mode switch working (`Constants.currentMode`)
- [ ] Ensure mechanism sims compile without robot hardware
- [ ] Periodically run desktop sim after Phase 1–2 PRs

### 3.5 Devcontainer / tooling

- [ ] Note that 2027 wants newer OS / Java 25; track `.devcontainer/devcontainer.json` as a kickoff task, not mid-season
- [ ] Keep Gradle + Spotless green so import diffs stay readable

**Done when:** no NT3 / SDB / removed command APIs; sim still boots; lifecycle stays constructor-based.

---

## Phase 4 — Kickoff-week playbook (do not do early)

Execute only when 2027 tools + vendordeps are stable enough for a branch.

1. Install 2027 WPILib; import project (package rename `edu.wpi.first` → `org.wpilib`)
2. Re-add vendordeps: AdvantageKit, Phoenix 6, PathPlanner, PhotonVision, any still-needed others
3. Update `LoggedTalon` / drive IO / vision IO for vendor API breaks
4. Replace `CommandXboxController` → `CommandGamepad` (or v3 equivalent) at the controls facade
5. Regenerate `TunerConstants` for Systemcore / current swerve
6. Swap field layout + PathPlanner assets + `ShotCalculator` maps for the new game
7. Retest NoOpp → Sim → Real bringup per [BRINGUP.md](./BRINGUP.md)
8. Validate I2C / DIO / Expansion Hub wiring against Systemcore pinouts (I2C SCL/SDA swap, etc.)
9. Update `.devcontainer` WPILib VSIX / Java version
10. Smoke: teleop drive, auto warmup, vision pose, intake/index/shoot cycles, Elastic layout

Optional later: Commands v3 migration as its own project, not coupled to hardware bringup.

---

## Suggested PR sequence (smallest useful slices)

| Order | PR | Approx. size | Depends on |
|------:|----|--------------|------------|
| 1 | Remove unused Spark / NavX / Servo / Limelight (and vendordeps) | S | — |
| 2 | Central `HardwareMap` / CAN+DIO constants | S–M | — |
| 3 | Unify field layout between vision + field constants | S | — |
| 4 | Eliminate SmartDashboard usages | S | — |
| 5 | `DriverControls` facade (suppliers only outward) | S | — |
| 6 | Continuous-angle audit for turret/hood/gyro | S–M | — |
| 7 | Inject indexer/shooter motors from container | M | 2 |
| 8 | Units migration in drive/module hot paths | M | — |
| 9 | `LoggedTalon` API touch-list + sim/NoOpp CI smoke | M | 1, 7 |

Ship 1–5 before end of build season if possible; 6–9 as capacity allows.

---

## Risk register (repo-specific)

| Risk | Impact | Mitigation |
|------|--------|------------|
| Phoenix 6 API / Systemcore timing changes | Mechanisms fail to compile or config | Keep NoOpp/Sim; concentrate Phoenix in `LoggedTalon` |
| AdvantageKit lag behind WPILib alphas | Blocked import | Follow Mechanical Advantage + SystemcoreTesting; keep IO seams |
| PathPlanner / Photon not ready for an alpha | Autos / vision blocked | Keep path assets data-driven; vision behind `VisionIO` |
| `Rotation2d` wrapping | Turret/hood/heading bugs | Phase 2.4 before hardware change |
| SmartDashboard habits | Broken tuning at kickoff | Phase 1.4 now |
| Servo / SPI / removed HAL features | Dead code fails compile | Delete `LoggedServo` / unused IMU paths now |
| Scattered CAN IDs | Painful Systemcore rewire | Phase 1.2 |
| Split AprilTag layouts | Bad poses after field update | Phase 1.3 |

---

## Out of scope (intentionally)

- Full Commands v3 rewrite during competition season
- Early Systemcore-only HAL experiments on competition branch
- Game-specific shot maps / autos for 2027 before game reveal
- Rewriting AdvantageKit swerve from scratch — extend the existing template

---

## Progress log

| Date | Change | PR / notes |
|------|--------|------------|
| 2026-08-29 | Initial plan created from 2027 alpha docs + repo audit | — |

When a Phase item finishes, check it off above and add a one-line entry here.
