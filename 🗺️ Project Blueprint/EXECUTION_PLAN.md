# Player Framework — Execution Plan

> **Last Updated:** 2025-07-12
> **ClickUp Space:** 901811697113
> **Target Unity Version:** Unity 6 (latest LTS)
> **Sessions:** 16 (Phase-based, clean context per session)

---

## Naming Convention (Locked)

Use **Architecture doc naming** over IDD naming throughout all implementations:
- `IMovementSubsystem` (Arch) over `IMovementSystem` (IDD)
- `UICommsSubsystem` (Arch) over `UISubsystem` (IDD)
- `IInputProvider` stays as-is (both docs agree)
- `IInteractable` goes in `Core/Interfaces/` (game code implements it)

---

## Key Architecture Rules (Enforced Every Session)

1. Subsystems are **plain C# classes** (not MonoBehaviours) — only `PlayerEntity` is a MonoBehaviour
2. Dependencies flow **inward to interfaces only** — no concrete cross-references
3. **3-tier communication**: Direct call (mediator), EventBus (typed structs), Data Channels (SO)
4. All event types are **value types** (structs) — zero heap allocation
5. Mandatory **`.asmdef`** per top-level folder
6. **Composition over inheritance** for weapons

---

## Folder Structure

```
Assets/Framework/
├── Player/
│   ├── Core/
│   │   ├── Interfaces/     ← All I*.cs interfaces
│   │   ├── Base/            ← PlayerEntityBase, PlayerSubsystemBase, SubsystemRegistry
│   │   ├── Events/          ← PlayerEventBus + Types/ subfolder
│   │   └── Data/            ← PlayerInputFrame, PlayerFrameworkConfig
│   ├── Subsystems/
│   │   ├── Movement/        ← States/ + Modules/
│   │   ├── Parkour/
│   │   ├── Combat/          ← Modules/ (Health, Damage, HitReaction, Death, Melee, Combo)
│   │   ├── Weapon/          ← Modules/ + Projectile/ + Effects/
│   │   ├── Camera/          ← Modes/ + Modules/
│   │   ├── Input/           ← Providers/
│   │   ├── Animation/       ← Modules/
│   │   ├── Audio/           ← Modules/
│   │   ├── Interaction/
│   │   └── UI/              ← Channels/
│   ├── Persistence/         ← ISaveable, PlayerSaveData
│   └── Multiplayer/         ← INetworkAdapter, NetworkInputProvider
├── Debug/                   ← PlayerDebugger, SubsystemMonitor, GizmoDrawer
└── Shared/                  ← TypedEventChannel, ObjectPoolManager, MainAudioManager

Assets/Tests/
├── EditMode/
└── Mocks/

Assets/Games/[GameName]/
├── Player/
└── Overrides/
```

---

## Session Breakdown

---

### SESSION 0 — Unity Project Setup + GitHub Bootstrap

**ClickUp Tasks:** 4 Unity Project Setup tasks + Documentation tasks
**Pattern:** N/A (project infrastructure)

**Files created:**
- Unity 6 project at this workspace path
- `.gitignore` (Unity template, BEFORE git init)
- `ProjectSettings/ProjectSettings.asset` → Force Text serialization
- Git repos: `unity-player-framework` (private) + game repo (private)
- `package.json` at framework root (UPM: `com.dod-studio.player-framework`, v1.0.0)
- `CHANGELOG.md` (Keep a Changelog format)
- `.gitattributes` (Git LFS for `.png`, `.jpg`, `.fbx`, `.wav`, `.mp3`, `.ogg`, `.anim`, `.controller`)
- `.github/pull_request_template.md`
- `.github/CODEOWNERS`
- `.github/workflows/pr-validation.yml` + `release.yml`
- `develop` branch + branch protection on `main` and `develop`
- Import Mixamo SWAT character to `Assets/Art/Characters/`
- Create full folder hierarchy under `Assets/Framework/`
- Create all `.asmdef` files:
  - `Framework.asmdef` (Player/)
  - `Framework.Debug.asmdef` (Debug/)
  - `Framework.Shared.asmdef` (Shared/)
  - `Framework.Tests.asmdef` (Tests/)
- Wire game repo to framework via UPM Git package reference
- Tag `v0.1.0`

---

### SESSION 1 — Core Contracts Layer

**ClickUp Tasks:** 01–05
**Pattern:** N/A (pure type definitions, zero logic)

**Files created:**

**Enums (21 total)** — `Core/` namespace:

| File | Enum | Values |
|------|------|--------|
| `Enums/PlayerState.cs` | `PlayerState` | Alive, Dead, InVehicle, InCutscene, Spectating |
| `Enums/MovementState.cs` | `MovementState` | Idle, Walk, Sprint, Crouch, CrouchWalk, Prone, ProneCrawl, Slide, Jump, Fall, LedgeHang, SwimSurface, SwimUnderwater, Wade, Climb, Ladder, LedgeMantle, Vault, Mantle, Dodge, Lean, Knockback, Stunned |
| `Enums/WeaponState.cs` | `WeaponState` | Unequipped, Equipping, Idle, Firing, FiringADS, Reloading, Unequipping, NoAmmo |
| `Enums/FireMode.cs` | `FireMode` | Single, Burst, Auto |
| `Enums/CameraMode.cs` | `CameraMode` | FPS, TPS, Cinematic, Spectator |
| `Enums/DamageType.cs` | `DamageType` | Bullet, Melee, Explosion, Fall, Environment, StatusEffect, Instant |
| `Enums/SurfaceType.cs` | `SurfaceType` | Concrete, Grass, Metal, Wood, Dirt, Sand, Water, Snow, Gravel, Glass, Carpet, Tile |
| `Enums/ForceMode.cs` | `ForceMode` | Force, Impulse, VelocityChange, Acceleration |
| `Enums/InputContext.cs` | `InputContext` | Gameplay, UI, Ladder, Cutscene, Vehicle, Dead, Dialogue |
| `Enums/ParkourActionType.cs` | `ParkourActionType` | None, Vault, Mantle, Climb, LedgeHang |
| `Enums/MeleeAttackType.cs` | `MeleeAttackType` | Light, Heavy, Finisher, Shove |
| `Enums/CombatState.cs` | `CombatState` | Idle, Attacking, Recovering, Staggered |
| `Enums/IKTarget.cs` | `IKTarget` | LeftHand, RightHand, LeftFoot, RightFoot, LookAt, Body |
| `Enums/IKHint.cs` | `IKHint` | LeftElbow, RightElbow, LeftKnee, RightKnee |
| `Enums/VoiceLineType.cs` | `VoiceLineType` | Pain, Death, Effort, Breathing, Empty, Melee, Reload |
| `Enums/ProceduralOffsetType.cs` | `ProceduralOffsetType` | WeaponSway, BreathingLayer, RecoilLayer, LandingImpact |
| `Enums/ReloadType.cs` | `ReloadType` | Tactical, Empty |
| `Enums/CrosshairStyle.cs` | `CrosshairStyle` | Default, ADS, Interacting, Hidden |
| `Enums/AudioCategory.cs` | `AudioCategory` | Footstep, Voice, Weapon, Ambient |
| `Enums/InteractionType.cs` | `InteractionType` | Instant, Hold, Toggle, Repeatable |
| `Enums/UIEventType.cs` | `UIEventType` | HealthChanged, AmmoChanged, StaminaChanged, PlayerDied, WeaponSwitched |

**Data Structs (4 total)** — `Core/Data/`:

| File | Struct | Key Fields |
|------|--------|------------|
| `PlayerInputFrame.cs` | `PlayerInputFrame` | FrameId, Timestamp, MoveAxis, LookAxis, JumpPressed/Held, SprintHeld, CrouchPressed/Held, PronePressed, FirePressed/Held, AimPressed/Held, ReloadPressed, InteractPressed, WeaponSwitchSlot, LeanLeft/Right, DodgePressed, PausePressed, ContextPush, ContextPop |
| `PlayerKinematicState.cs` | `PlayerKinematicState` | FrameId, Position, Rotation, Velocity, IsGrounded, GroundNormal, SlopeAngle, CurrentMovementState, CurrentSurface, StaminaNormalized, FallVelocity |
| `DamageData.cs` | `DamageData` | Amount, Type, HitPoint, HitDirection, HitNormal, SourceId, BypassArmor, BypassInvincibility, ArmorPenetration |
| `HitData.cs` | `HitData` | HitPoint, HitNormal, Distance, HitObject, HitTag, IsHeadshot |

**Event Structs (38 total)** — `Core/Events/`:

Input Events (8):
- `FireInputEvent`, `AimInputEvent`, `MoveInputEvent`, `LookInputEvent`, `JumpInputEvent`, `InteractInputEvent`, `WeaponSwitchInputEvent`, `InputContextChangedEvent`

Movement Events (5):
- `MovementStateChangedEvent`, `PlayerLandedEvent`, `PlayerJumpedEvent`, `SurfaceChangedEvent`, `StaminaChangedEvent`

Weapon Events (7):
- `WeaponFiredEvent`, `HitDetectedEvent`, `WeaponReloadedEvent`, `WeaponSwitchedEvent`, `WeaponEmptyEvent`, `AdsStateChangedEvent`, `RecoilEvent`

Combat/Health Events (6):
- `DamageReceivedEvent`, `HealthChangedEvent`, `PlayerDiedEvent`, `PlayerRespawnedEvent`, `MeleeHitEvent`, `ComboAdvancedEvent`

Parkour Events (3):
- `ParkourActionStartedEvent`, `ParkourActionCompletedEvent`, `ParkourActionFailedEvent`

Camera Events (2):
- `CameraModeChangedEvent`, `FOVChangedEvent`

Interaction Events (3):
- `InteractionCandidateChangedEvent`, `InteractionStartedEvent`, `InteractionCompletedEvent`

Animation Events (4):
- `FootstepAnimationEvent`, `ReloadCompleteAnimationEvent`, `MeleeHitWindowOpenedEvent`, `MeleeHitWindowClosedEvent`

**Interfaces (16 total)** — `Core/Interfaces/`:
- `IPlayerSubsystem`, `IServiceLocator`, `IEventBus`
- `IInputProvider`, `IInputSystem`
- `IMovementSubsystem`, `IParkourSubsystem`
- `IHealthSubsystem`, `ICombatSubsystem`, `IWeaponSubsystem`
- `ICameraSubsystem`, `IAnimationSubsystem`
- `IAudioSubsystem`, `IInteractionSubsystem`, `IUIBridgeSubsystem`
- `IInteractable`

**ScriptableObject Configs (6)** — `Core/Data/`:
- `PlayerFrameworkConfig.cs` (root config)
- `MovementConfigSO.cs` (30+ fields from IDD)
- `CombatConfigSO.cs`
- `WeaponDefinitionSO.cs` (15+ fields)
- `CameraConfigSO.cs`
- `AnimationConfigSO.cs`

---

### SESSION 2 — Infrastructure Systems Part 1

**ClickUp Tasks:** 06–08
**Pattern:** EventBus = typed pub/sub, ServiceLocator = scoped registry, SubsystemRegistry = lifecycle

**Files created:**
- `Core/Events/PlayerEventBus.cs` — implements IEventBus, struct-constrained generics, PublishDeferred/FlushDeferred, SetChannelEnabled
- `Core/Base/ServiceLocator.cs` — implements IServiceLocator, Dictionary<Type, object>, full preconditions/throws
- `Core/Base/SubsystemRegistry.cs` — registration, ordered Init/Enable/Disable/Shutdown, tick routing
- `Core/Base/LifecycleManager.cs` — explicit tick ordering: Input → Movement → Combat → Weapon → Camera → Animation → Audio
- Tick ordering documented and enforced

---

### SESSION 3 — Infrastructure Systems Part 2

**ClickUp Tasks:** 09–11
**Pattern:** Mediator (PlayerEntity), HSM (state machine infra), Debug overlay

**Files created:**
- `Core/Base/PlayerEntityBase.cs` — MonoBehaviour orchestrator, owns all subsystems, drives lifecycle, under 200 lines
- `Core/Base/PlayerSubsystemBase.cs` — abstract base implementing IPlayerSubsystem lifecycle
- `Core/Base/StateMachine/IState.cs` — `Enter()`, `Tick(float)`, `Exit()`
- `Core/Base/StateMachine/StateTransition.cs` — condition predicate + target state (data, not embedded)
- `Core/Base/StateMachine/StateTransitionTable.cs` — ordered transition evaluation
- `Core/Base/StateMachine/HierarchicalStateMachine.cs` — HSM with parent states, history states
- `Core/Base/StateMachine/MetaStateMachine.cs` — ALIVE/DEAD/IN_VEHICLE/IN_CUTSCENE
- `Core/Events/Types/MovementEvents.cs` (event file grouping)
- `Core/Events/Types/CombatEvents.cs`, `WeaponEvents.cs`, `HealthEvents.cs`, `InputEvents.cs`
- `Debug/PlayerDebugger.cs` — per-subsystem log channels
- `Debug/SubsystemMonitor.cs` — runtime debug window
- `Debug/GizmoDrawer.cs` — gizmo scaffolding

---

### SESSION 4 — Input System + Movement Core

**ClickUp Tasks:** 12–14
**Pattern:** Input = Adapter, Movement = HSM

**Files created:**
- `Subsystems/Input/InputSubsystem.cs` — implements IInputSystem, reads IInputProvider, publishes input events
- `Subsystems/Input/InputConfigSO.cs` — Input Action Asset reference
- `Subsystems/Input/Providers/HumanInputProvider.cs` — wraps Unity Input System
- `Subsystems/Input/Providers/NullInputProvider.cs` — for testing
- `Subsystems/Movement/MovementSubsystem.cs` — implements IMovementSubsystem, owns CharacterController
- `Subsystems/Movement/MovementConfigSO.cs` — full field set from IDD
- `Subsystems/Movement/StateMachine/MovementStateMachine.cs` — locomotion HSM root
- `Subsystems/Movement/StateMachine/States/GroundedState.cs` — parent state
- `Subsystems/Movement/StateMachine/States/IdleState.cs`
- `Subsystems/Movement/StateMachine/States/WalkState.cs`
- `Subsystems/Movement/StateMachine/States/SprintState.cs`
- `Subsystems/Movement/StateMachine/States/JumpState.cs`
- `Subsystems/Movement/StateMachine/States/AirborneState.cs`
- Ground detection, gravity, slope handling, step detection logic in MovementSubsystem

---

### SESSION 5 — Camera FPS + Animation Base

**ClickUp Tasks:** 15–16
**Pattern:** Camera = Strategy + Decorator, Animation = Pure Observer

**Files created:**
- `Subsystems/Camera/CameraSubsystem.cs` — implements ICameraSubsystem, owns Camera transform
- `Subsystems/Camera/CameraConfigSO.cs` — FOV, sensitivity, pitch limits, etc.
- `Subsystems/Camera/Modes/ICameraMode.cs` — strategy interface
- `Subsystems/Camera/Modes/FPSCameraMode.cs` — mouse look, pitch clamp
- `Subsystems/Camera/Modules/DynamicFOVModule.cs`
- `Subsystems/Animation/AnimationSubsystem.cs` — implements IAnimationSubsystem, writes Animator params
- `Subsystems/Animation/AnimationConfigSO.cs`
- `Subsystems/Animation/Modules/AnimationEventRelay.cs` — bridges Animator events → EventBus

---

### SESSION 6 — Extended Movement States

**ClickUp Tasks:** 17–19
**Pattern:** HSM leaf states, Strategy (camera modifiers)

**Files created:**
- `Subsystems/Movement/StateMachine/States/CrouchState.cs` (sub-HSM: CrouchIdle, CrouchWalk)
- `Subsystems/Movement/StateMachine/States/ProneState.cs`
- `Subsystems/Movement/StateMachine/States/SlideState.cs`
- `Subsystems/Movement/StateMachine/States/LeanState.cs`
- `Subsystems/Movement/Modules/LeanModule.cs`
- `Subsystems/Camera/Modules/LeanCameraModule.cs`

---

### SESSION 7 — Movement Modules

**ClickUp Tasks:** 20–23
**Pattern:** Strategy (surface detection), Observer (events)

**Files created:**
- `Subsystems/Movement/Modules/StaminaModule.cs` — drain/regen, configurable per state
- `Subsystems/Movement/Modules/SurfaceDetectionModule.cs` — raycast, SurfaceDefinitionLibrarySO
- `Subsystems/Movement/Modules/FallDamageModule.cs` — peak velocity tracking → DamageData event
- `Subsystems/Movement/Modules/HeadBobModule.cs` — procedural camera bob

---

### SESSION 8 — Weapon System Core Part 1

**ClickUp Tasks:** 24–27
**Pattern:** Composition + Factory, State Machine (per weapon)

**Files created:**
- `Subsystems/Weapon/WeaponSubsystem.cs` — implements IWeaponSubsystem
- `Subsystems/Weapon/WeaponManager.cs` — slot management, equip/unequip
- `Subsystems/Weapon/WeaponInstance.cs` — runtime composition of modules
- `Subsystems/Weapon/WeaponDefinitionSO.cs` — full field set
- `Subsystems/Weapon/Modules/ShootingModule.cs` — hitscan fire pipeline
- `Subsystems/Weapon/Modules/AmmoModule.cs` — magazine, reserve, consumption
- `Subsystems/Weapon/Modules/ReloadModule.cs` — reload state machine, timing
- `Subsystems/Weapon/Modules/SpreadModule.cs` — accuracy cones
- `Subsystems/Weapon/Projectile/HitDetectionModule.cs` — raycast, hit registration, DamageData creation

---

### SESSION 9 — Weapon System Core Part 2

**ClickUp Tasks:** 28–30
**Pattern:** Observer (audio/VFX as pure consumers), Decorator (camera modifiers)

**Files created:**
- `Subsystems/Weapon/Modules/RecoilModule.cs` — RecoilProfileSO, per-shot pattern
- `Subsystems/Weapon/Modules/ADSModule.cs` — ADS state, zoom, sensitivity
- `Subsystems/Camera/Modules/RecoilCameraModule.cs` — visual recoil application
- `Subsystems/Weapon/Effects/WeaponVFXHandler.cs` — muzzle flash, casing eject, impact
- `Subsystems/Weapon/Effects/WeaponAudioHandler.cs` — fire, reload, empty sounds

---

### SESSION 10 — IK + Procedural Animation

**ClickUp Tasks:** 31–32
**Pattern:** Observer (reactive), Procedural layers

**Files created:**
- `Subsystems/Animation/Modules/IKModule.cs` — foot IK, hand IK, aim IK, spine tracking
- `Subsystems/Animation/Modules/ProceduralAnimModule.cs` — weapon sway, breathing, procedural layers

---

### SESSION 11 — Health + Combat

**ClickUp Tasks:** 33–36
**Pattern:** Observer + Chain of Responsibility (damage pipeline)

**Files created:**
- `Subsystems/Combat/CombatSubsystem.cs` — implements ICombatSubsystem
- `Subsystems/Combat/CombatConfigSO.cs`
- `Subsystems/Combat/Modules/HealthModule.cs` — HP pool, shields, regen
- `Subsystems/Combat/Modules/DamageModule.cs` — Chain of Responsibility: armor → resistance → invincibility → apply
- `Subsystems/Combat/Modules/HitReactionModule.cs` — hit direction, stagger
- `Subsystems/Combat/Modules/DeathModule.cs` — death state, respawn hooks
- `Subsystems/Combat/Modules/MeleeModule.cs` — hitbox activation, melee trace
- `Subsystems/Combat/Modules/ComboModule.cs` — combo timing, sequence tracking

---

### SESSION 12 — Audio + Interaction + UI Bridge

**ClickUp Tasks:** 37–39
**Pattern:** Audio = Observer + Object Pool, Interaction = Visitor, UI = Reactive Data Binding

**Files created:**
- `Subsystems/Audio/PlayerAudioSubsystem.cs` — implements IAudioSubsystem
- `Subsystems/Audio/PlayerAudioConfigSO.cs`
- `Subsystems/Audio/Modules/FootstepModule.cs` — surface-aware
- `Subsystems/Audio/Modules/BreathingModule.cs` — stamina-driven
- `Subsystems/Audio/Modules/CombatVoiceModule.cs` — pain, death, effort
- `Subsystems/Interaction/InteractionSubsystem.cs` — implements IInteractionSubsystem
- `Subsystems/Interaction/InteractionConfigSO.cs`
- `Subsystems/Interaction/InteractionDetector.cs` — overlap/raycast
- `Subsystems/UI/UICommsSubsystem.cs` — implements IUIBridgeSubsystem
- `Subsystems/UI/Channels/HealthDataChannel.cs` — SO reactive variable
- `Subsystems/UI/Channels/AmmoDataChannel.cs`
- `Subsystems/UI/Channels/StaminaDataChannel.cs`

---

### SESSION 13 — Parkour + Advanced Movement

**ClickUp Tasks:** 40–43
**Pattern:** Parkour = State Machine + Strategy + Template Method, Projectile = Object Pool

**Files created:**
- `Subsystems/Parkour/ParkourSubsystem.cs` — implements IParkourSubsystem
- `Subsystems/Parkour/ParkourConfigSO.cs`
- `Subsystems/Parkour/ParkourDetector.cs` — geometry queries
- `Subsystems/Weapon/Projectile/ProjectileSystem.cs` — pooled projectiles, Burst raycasting
- `Subsystems/Movement/StateMachine/States/SwimState.cs` (sub-HSM: Surface, Underwater, Wade)
- `Subsystems/Movement/StateMachine/States/LadderState.cs`
- `Subsystems/Camera/Modes/TPSCameraMode.cs`

---

### SESSION 14 — Production Readiness (Part 1)

**ClickUp Tasks:** 44–45
**Pattern:** Persistence hooks, Editor tooling

**Files created:**
- `Player/Persistence/ISaveable.cs`
- `Player/Persistence/PlayerSaveData.cs`
- `[AuthoritativeState]` attribute + snapshot interfaces on all owning subsystems
- `Debug/Editor/StateMachineVisualizer.cs`
- `Debug/Editor/WeaponConfigEditor.cs`
- `Debug/Editor/SubsystemMonitorWindow.cs`

---

### SESSION 15 — Production Readiness (Part 2)

**ClickUp Tasks:** 46–49
**Pattern:** Testing, optimization, documentation

**Files created:**
- `Tests/Mocks/MockInputProvider.cs`
- `Tests/Mocks/MockPlayerEntity.cs`
- `Tests/Mocks/MockAudioSubsystem.cs`
- `Tests/EditMode/StateMachineTests.cs`
- `Tests/EditMode/EventBusTests.cs`
- `Tests/EditMode/DamagePipelineTests.cs`
- `Tests/PlayMode/MovementFlowTests.cs`
- `Tests/PlayMode/WeaponFireTests.cs`
- `Tests/PlayMode/CombatLifecycleTests.cs`
- `Player/Multiplayer/INetworkAdapter.cs`
- `Player/Multiplayer/NetworkInputProvider.cs`
- GC allocation audit
- Pool coverage verification
- Physics raycast batching check
- Animator hash cache validation
- Multiplayer boundary audit document
- XML docs on all public interface methods
- `CHANGELOG.md` updates
- Version bumps to `v1.0.0`

---

## Missing Task: Dodge State

The Architecture doc defines Step 36 "Dodge State" (directional dodge/roll, invulnerability frames, stamina burst cost, EVASIVE super-state). This was added to ClickUp and included in Session 6.

---

## Session Context Strategy

Each session will:
1. Read the **Architecture doc** sections relevant to that phase
2. Read the **IDD** for exact interface/event signatures
3. Read output files from the previous session
4. Implement, test, commit with **Conventional Commits** (e.g., `feat(movement): add SwimSurface state`)
5. Update ClickUp task status
