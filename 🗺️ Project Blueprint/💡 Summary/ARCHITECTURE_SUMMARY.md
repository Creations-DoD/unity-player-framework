# Player Framework — Architecture Summary

> **Source:** PlayerFramework_Architecture.html (15 sections, 2250 lines)
> **Purpose:** Quick-reference for implementation sessions. Read before every session.
> **Rule:** Architecture doc naming wins over IDD naming wherever they conflict.

---

## 1. Naming Convention (Locked)

Architecture doc names override IDD names. Use these exactly:

**Overridden Names:**

| IDD Name | Architecture Name | Notes |
|----------|-------------------|-------|
| `IMovementSystem` | `IMovementSubsystem` | All subsystems use "Subsystem" suffix |
| `UISubsystem` | `UICommsSubsystem` | Communicates events to UI data channels |
| `IUISubsystem` | `IUIBridgeSubsystem` | Bridge between framework and UI layer |

**Unchanged Names (both docs agree):**

| Name | Location |
|------|----------|
| `IInputProvider` | `Core/Interfaces/` |
| `IInteractable` | `Core/Interfaces/` (game code implements it) |
| `IPlayerSubsystem` | `Core/Interfaces/` (base contract) |
| `IServiceLocator` | `Core/Interfaces/` |
| `IEventBus` | `Core/Interfaces/` |
| `IInputSystem` | `Core/Interfaces/` |
| `IParkourSubsystem` | `Core/Interfaces/` |
| `IHealthSubsystem` | `Core/Interfaces/` |
| `ICombatSubsystem` | `Core/Interfaces/` |
| `IWeaponSubsystem` | `Core/Interfaces/` |
| `ICameraSubsystem` | `Core/Interfaces/` |
| `IAnimationSubsystem` | `Core/Interfaces/` |
| `IAudioSubsystem` | `Core/Interfaces/` |
| `IInteractionSubsystem` | `Core/Interfaces/` |

---

## 2. Architecture Rules (Non-Negotiable)

1. **Subsystems are plain C# classes** (not MonoBehaviours) — only `PlayerEntity` is a MonoBehaviour
2. **Dependencies flow inward to interfaces only** — no concrete cross-references between subsystems
3. **3-tier communication:** Direct Call (mediator via PlayerEntity), EventBus (typed structs), Data Channels (SO)
4. **All event types are value types (structs)** — zero heap allocation, enforced at design time
5. **Mandatory `.asmdef` per top-level folder** — compile-time enforcement of dependency rules
6. **Composition over inheritance for weapons** — WeaponInstance assembled from modules, no class hierarchy
7. **No gameplay logic in PlayerEntity** — it only orchestrates, owns subsystems, routes calls
8. **No direct input reading in subsystems** — always via `IInputProvider` -> `PlayerInputFrame` struct
9. **No string-based events or SendMessage** — typed struct events exclusively
10. **No `GetComponent` / `FindObjectOfType` across subsystems** — all deps injected at composition time

---

## 3. Folder Structure

```
Assets/
├── Framework/                          ← Framework.asmdef (core runtime)
│   ├── Player/
│   │   ├── Core/
│   │   │   ├── Interfaces/             ← All I*.cs interfaces
│   │   │   ├── Base/                   ← PlayerEntityBase, PlayerSubsystemBase, SubsystemRegistry
│   │   │   │   └── StateMachine/       ← IState, StateTransition, HSM, MetaStateMachine
│   │   │   ├── Events/
│   │   │   │   ├── PlayerEventBus.cs
│   │   │   │   └── Types/              ← MovementEvents, CombatEvents, WeaponEvents, HealthEvents, InputEvents
│   │   │   └── Data/                   ← PlayerInputFrame, PlayerFrameworkConfig
│   │   ├── Subsystems/
│   │   │   ├── Input/                  ← InputSubsystem, InputConfig, Providers/
│   │   │   ├── Movement/               ← MovementSubsystem, MovementConfig, StateMachine/States/, Modules/
│   │   │   ├── Parkour/                ← ParkourSubsystem, ParkourConfig, ParkourDetector
│   │   │   ├── Combat/                 ← CombatSubsystem, CombatConfig, Modules/ (Health, Damage, HitReaction, Death, Melee, Combo)
│   │   │   ├── Weapon/                 ← WeaponSubsystem, WeaponManager, WeaponInstance, Modules/, Projectile/, Effects/
│   │   │   ├── Camera/                 ← CameraSubsystem, CameraConfig, Modes/ (ICameraMode, FPS, TPS), Modules/
│   │   │   ├── Animation/              ← AnimationSubsystem, AnimationConfig, Modules/ (IK, Procedural, EventRelay)
│   │   │   ├── Audio/                  ← PlayerAudioSubsystem, PlayerAudioConfig, Modules/ (Footstep, Breathing, CombatVoice)
│   │   │   ├── Interaction/            ← InteractionSubsystem, InteractionConfig, IInteractable, InteractionDetector
│   │   │   └── UI/                     ← UICommsSubsystem, Channels/ (HealthDataChannel, AmmoDataChannel, StaminaDataChannel)
│   │   ├── Persistence/                ← ISaveable, PlayerSaveData
│   │   └── Multiplayer/                ← INetworkAdapter, NetworkInputProvider
│   ├── Debug/                          ← Framework.Debug.asmdef
│   │   ├── PlayerDebugger.cs
│   │   ├── SubsystemMonitor.cs
│   │   └── GizmoDrawer.cs
│   └── Shared/                         ← Framework.Shared.asmdef
│       ├── EventChannels/              ← TypedEventChannel.cs
│       ├── ObjectPool/                 ← ObjectPoolManager.cs
│       └── Audio/                      ← MainAudioManager.cs
│
├── Tests/                              ← Framework.Tests.asmdef
│   ├── EditMode/Subsystems/            ← MovementSubsystemTests, WeaponSystemTests, StateMachineTests
│   └── Mocks/                          ← MockInputProvider, MockAudioSubsystem, MockPlayerEntity
│
└── Games/
    └── [GameName]/                     ← Game.asmdef (references Framework.asmdef)
        ├── Player/                     ← GamePlayerEntity.cs, Config/
        └── Overrides/                  ← Game-specific subsystem overrides
```

**Assembly Definitions:**

| .asmdef File | Location | Purpose |
|-------------|----------|---------|
| `Framework.asmdef` | `Player/` | Core runtime — all subsystems and interfaces |
| `Framework.Debug.asmdef` | `Debug/` | Editor + runtime debug tooling |
| `Framework.Shared.asmdef` | `Shared/` | Shared utilities (ObjectPool, MainAudioManager, TypedEventChannel) |
| `Framework.Tests.asmdef` | `Tests/` | EditMode + PlayMode tests |
| `Game.asmdef` | `Games/[GameName]/` | Per-game implementation layer |

---

## 4. Dependency Map

**Golden Rule:** Dependencies only flow inward toward interfaces — never outward toward concrete implementations.

### What PlayerEntity Owns
- All subsystems via their **interfaces only**
- `PlayerEventBus` (owns it)
- `SubsystemRegistry`
- `CharacterController` (scene component)
- `Animator` (scene component)
- `IInputProvider` (injected — human / AI / network / replay)

### What SubsystemRegistry Manages
- Registration of all subsystems
- Ordered Init/Enable/Disable/Shutdown
- Tick routing (Tick / FixedTick / LateTick)
- Enforced tick ordering: Input -> Movement -> Combat -> Weapon -> Camera -> Animation -> Audio

### Per-Subsystem Dependencies

| Subsystem | Depends On |
|-----------|------------|
| MovementSubsystem | `IInputProvider` (interface), `PlayerEventBus`, `MovementConfig` (SO), `CharacterController` |
| CameraSubsystem | `PlayerEventBus`, `CameraConfig` (SO), LeanModule output (via shared data struct, NOT module ref) |
| WeaponSubsystem | `IInputProvider`, `PlayerEventBus`, `ProjectileSystem` (owns), `WeaponDefinition` assets (SO) |
| CombatSubsystem | `PlayerEventBus`, `CombatConfig` (SO), **ZERO knowledge of WeaponSubsystem** |
| AnimationSubsystem | `PlayerEventBus` (subscribes to ALL state events), `Animator`, `AnimationConfig` |
| PlayerAudioSubsystem | `PlayerEventBus`, `IMainAudioManager`, `SurfaceType` data (via event payload) |
| InteractionSubsystem | `IInputProvider` (interact button), `PlayerEventBus` |
| UICommsSubsystem | `PlayerEventBus`, Data channel ScriptableObjects (writes to them), **ZERO knowledge of HUD** |

### Forbidden Dependencies

| Forbidden Path | Reason |
|---------------|--------|
| MovementSubsystem -> WeaponSubsystem | Use events for ADS slow modifier |
| CombatSubsystem -> WeaponSubsystem | Damage comes as event payload |
| AnimationSubsystem -> MovementSubsystem | Listen to events, not direct state |
| AudioSubsystem -> WeaponSubsystem | Weapon audio events carry AudioClip refs |
| UICommsSubsystem -> any UI class | One-way: framework -> UI, never UI -> framework |
| Any Subsystem -> GameLayer classes | Framework must remain game-agnostic |

### DI Strategy
- **Injected at construction:** `IInputProvider`, `PlayerEventBus`, SO configs, `IMainAudioManager`
- **Resolved at runtime:** `CharacterController`, `Animator`, `Camera` (scene components)
- **Never via:** `GetComponent<T>`, `FindObjectOfType<T>`, static singletons, string-keyed lookups

---

## 5. Tick Order / Lifecycle

### Tick Execution Order (Every Frame)
```
Input -> Movement -> Combat -> Weapon -> Camera -> Animation -> Audio
```

### Full Lifecycle Phases
```
Init -> PostInit -> Enable -> Tick -> FixedTick -> LateTick -> Disable -> Shutdown
```

| Phase | When | Purpose |
|-------|------|---------|
| **Init** | Awake/Start | Subsystem registration, dependency injection, Component discovery |
| **PostInit** | After all Init complete | Cross-subsystem wiring that requires all subsystems to exist |
| **Enable** | On enable | Reactivate subsystem (resumes tick processing) |
| **Tick** | Every frame (fixed order) | Per-frame logic: input read, movement, combat, weapon, camera, animation, audio |
| **FixedTick** | Physics tick | Physics-dependent logic (CharacterController moves, projectile physics) |
| **LateTick** | After all Ticks | Camera follow (after movement), final IK adjustments |
| **Disable** | On disable | Pause subsystem, stop tick processing |
| **Shutdown** | OnDestroy | Cleanup, unsubscribe events, release resources |

### Which Phases Use Which Tick Methods

| Subsystem | Tick | FixedTick | LateTick |
|-----------|------|-----------|----------|
| Input | Yes | No | No |
| Movement | Yes | Yes (CharacterController) | No |
| Combat | Yes | No | No |
| Weapon | Yes | No | No |
| Camera | No | No | Yes (after movement settles) |
| Animation | Yes | No | Yes (IK final pass) |
| Audio | Yes (event-driven) | No | No |

---

## 6. Design Patterns Used (14+)

| # | Pattern | Applied To | How It Works |
|---|---------|------------|--------------|
| 1 | **Mediator** | PlayerEntity | Sole communication hub. Subsystems never call each other directly. All orchestration flows through PlayerEntity. Not a god class — carries zero gameplay logic. |
| 2 | **Component** | All subsystems | Self-contained components. Swap or extend any without touching others. Unity-native mental model. |
| 3 | **Event-Driven (Observer)** | Cross-subsystem reactions, UI, Audio | Subsystems emit typed events; any number of listeners react with zero awareness of each other. |
| 4 | **Interface Segregation** | All subsystem contracts | Systems depend on thinnest possible contract. IMovementSubsystem can have 4 methods while concrete class has 40. |
| 5 | **Hierarchical State Machine** | Movement, Combat, Weapon | Eliminates boolean flag hell. Prevents invalid state combinations. Clean transition hooks. HSM groups states logically — new sub-state only needs transitions within its group. |
| 6 | **Data-Driven (ScriptableObjects)** | All configuration data | Designers tune behaviour without code changes. Multiple games use same code with different data assets. |
| 7 | **Service Locator (scoped)** | Cross-cutting concerns (Audio, Debug) | Avoids deep dependency chains for globally-needed services. Bounded to framework scope — not global singleton. |
| 8 | **Strategy** | Camera modes, Surface detection, Weapon types | Runtime-swappable algorithms. FPS->TPS camera switch is a strategy swap, not structural change. |
| 9 | **Decorator** | Camera effects (shake, recoil, lean) | Stackable effects applied as ordered transforms to final camera position, not mixed into mode logic. |
| 10 | **Adapter** | Input system | Wraps Unity Input System API into `PlayerInputFrame` value type. When Unity changes their Input API, only the adapter changes. |
| 11 | **Composition over Inheritance** | Weapon system | WeaponInstance assembled at runtime from WeaponDefinition SO + modules (Shooting, Reload, Recoil, ADS, Ammo). No class hierarchy. |
| 12 | **Factory** | WeaponInstance assembly | WeaponManager assembles the correct module combination from WeaponDefinition data. |
| 13 | **Chain of Responsibility** | Damage pipeline (Combat) | Damage flows through pipeline of modifiers (armor, shields, invincibility, resistances) before application. Each link can modify or absorb damage. |
| 14 | **Visitor** | Interaction system | Player (visitor) visits interactables. Each IInteractable accepts the visit and knows what to do. Framework has zero knowledge of what any specific interactable does. |
| 15 | **Object Pool** | Audio sources, Projectiles | Pool of pre-warmed AudioSources. Never `PlayClipAtPoint`. Pooled projectile lifecycle for high-frequency short-lived objects. |
| 16 | **Reactive Data Binding** | UI (ScriptableObject channels) | SO data channels for framework->UI communication. HUD holds reference to HealthDataChannel asset, not to any gameplay class. |

---

## 7. Development Order (43 Steps, 7 Phases)

### Phase 0 — Framework Infrastructure (Weeks 1-2)

| Step | Name | Key Deliverable | Depends On |
|------|------|-----------------|------------|
| 01 | Core Contracts Layer | All interfaces, event structs, data structs, enums, SO base classes | Nothing (defines shared language) |
| 02 | Event Bus | Typed, allocation-free pub/sub bus. Struct events only. Zero reflection. | Step 01 |
| 03 | Service Locator | Scoped service locator for cross-cutting concerns | Step 01 |
| 04 | SubsystemRegistry + LifecycleManager | Registration, ordered Init/Enable/Disable/Shutdown, tick routing | Steps 01-03 |
| 05 | PlayerController Skeleton | Coordinator MonoBehaviour, subsystem wiring, lifecycle routing. Under 200 lines. | Steps 01-04 |
| 06 | State Machine Infrastructure | IState, StateTransition, HSM base, MetaStateMachine. Scaffolding only, no concrete states. | Step 01 |
| 07 | Debug Overlay + FrameworkLogger | SubsystemMonitor window, per-subsystem log channels, GizmoDrawer | Steps 01-04 |

### Phase 1 — Locomotion Foundation (Weeks 3-4)

| Step | Name | Key Deliverable | Depends On |
|------|------|-----------------|------------|
| 08 | Input System | Unity Input System adapter -> PlayerInputFrame pipeline. HumanInputProvider + NullInputProvider. | Steps 01-07 |
| 09 | Movement System — Core States | MovementHSM with Idle, Walk, Sprint, Jump, Fall. CharacterController integration. | Steps 01-08 |
| 10 | Ground Detection + Gravity | Ground snap, step detection, slope angle limits, gravity | Step 09 |
| 11 | Camera System — FPS Mode | FPS camera rig. Mouse look, pitch clamp, head bob scaffold. Baseline only. | Step 09 |
| 12 | Animation System — Base Layer | Animator parameter bridge. Locomotion blend tree for Idle/Walk/Sprint/Jump. | Step 09 |

### Phase 2 — Movement Completeness (Weeks 5-6)

| Step | Name | Key Deliverable | Depends On |
|------|------|-----------------|------------|
| 13 | Crouch + Prone States | Extend MovementHSM. Capsule height modification, speed reduction, sub-states. | Steps 09-12 |
| 14 | Slide State | Friction-based momentum from sprint. Transitions to Crouch on slide-end. | Steps 13 |
| 15 | StaminaModule | Stamina pool, consumption rates per state, exhaustion threshold, recovery delay | Steps 09-12 |
| 16 | SurfaceDetectionModule | Raycast-based ground material detection. SurfaceChangedEvent on contact change. | Steps 09-12 |
| 17 | FallDamageModule | Peak downward velocity tracking. DamageData event on landing if threshold exceeded. | Steps 09-12, 15 |
| 18 | Lean State + Camera Modifier | Left/right lean. LeanCameraModifier applies lateral offset via LeanEvent. | Steps 11, 13 |

### Phase 3 — Weapon Foundation + IK (Weeks 7-9)

| Step | Name | Key Deliverable | Depends On |
|------|------|-----------------|------------|
| 19 | Weapon System — Core | WeaponManager, fixed WeaponSlot array, WeaponDefinitionSO. Composition model. | Steps 01-08 |
| 20 | FireHandler — Hitscan | Fire pipeline: state check -> ammo check -> spread -> raycast -> events. No projectiles yet. | Step 19 |
| 21 | AmmoSystem | Clip state per WeaponInstance. Global reserve pools per calibre. Tactical vs empty reload. | Step 19 |
| 22 | ReloadHandler | Tactical and empty reload paths. Per-weapon interruptibility from WeaponDefinitionSO. | Steps 20-21 |
| 23 | RecoilSystem + Camera Modifier | Per-weapon RecoilProfileSO. Publishes RecoilEvent — Camera and Animation consume independently. | Steps 11, 19 |
| 24 | AdsSystem + FOV Modifier | ADS state, zoom, sensitivity. FOVModifier driven by AdsStateChangedEvent. | Steps 11, 19 |
| 25 | WeaponAudioHandler + WeaponVFXHandler | Pure event consumers. Subscribe to WeaponFiredEvent, ReloadedEvent, EmptyEvent. Never called directly. | Steps 19-22 |

### Phase 4 — Combat + Polish (Weeks 10-12)

| Step | Name | Key Deliverable | Depends On |
|------|------|-----------------|------------|
| 26 | IK System + Procedural Animation | Foot IK, hand IK, aim IK, spine tracking. Weapon sway, breathing offset. | Steps 09, 12 |
| 27 | HealthSystem + DamageProcessor | Damage pipeline: armor -> resistance -> invincibility -> HP deduction. Chain of Responsibility. | Steps 01-19 |
| 28 | HitDetectionSystem | Hitscan and projectile-collision variants. HitDetectedEvent into damage pipeline. | Steps 19-20, 27 |
| 29 | HitReactionSystem + DeathSystem | Hit reactions as states. DeathModule: death state, respawn hooks. Death fans out to camera, animation, audio, UI. | Steps 27-28 |
| 30 | MeleeSystem + ComboSystem | Melee hitbox activation, combo timing windows, finisher detection. Flat state machine with timing thresholds. | Step 27 |
| 31 | PlayerAudioSystem | FootstepModule (surface-aware), BreathingModule (stamina-driven), CombatVoiceModule (health events). | Steps 15-16, 27 |
| 32 | Interaction System + UIBridge | IInteractable priority queue, overlap detection. UIBridge: gameplay events -> SO data channels for HUD. | Steps 01-08, 27 |

### Phase 5 — Advanced Systems (Weeks 13-15)

| Step | Name | Key Deliverable | Depends On |
|------|------|-----------------|------------|
| 33 | Parkour System | ParkourDetector geometry queries. Vault, Mantle, Climb. Command-pattern sequences. | Steps 09-10 |
| 34 | ProjectileSystem | Pooled projectiles. Burst-compiled IJobParallelFor + RaycastCommand. Flyweight definition. | Steps 19, 20 |
| 35 | Swimming + Ladder States | WATER super-state (Surface, Underwater, Wade). VERTICAL Ladder state. | Step 13 |
| 36 | Dodge State | Directional dodge/roll. Invulnerability frames. Stamina burst cost. EVASIVE super-state. | Steps 15, 35 |
| 37 | TPS Camera Mode | TPSCameraMode strategy swap. Orbit camera with over-shoulder collision avoidance. | Step 11 |

### Phase 6 — Production Readiness (Weeks 16-18)

| Step | Name | Key Deliverable | Depends On |
|------|------|-----------------|------------|
| 38 | Save/Load State Hooks | [AuthoritativeState] attribute. Serializable snapshot interface on all owning subsystems. | All prior steps |
| 39 | Editor Tools | StateMachineVisualizer, WeaponConfigEditor, SubsystemMonitorWindow | All prior steps |
| 40 | Full Test Suite | EditMode: state machine, event bus, damage pipeline. PlayMode: movement flow, weapon fire, combat lifecycle. | All prior steps |
| 41 | Optimisation Pass | GC allocation audit, pool coverage, physics raycast batching, Animator hash cache validation. | All prior steps |
| 42 | Multiplayer Boundary Audit | Review [AuthoritativeState] tags. Document client-predicted paths. Define NetworkInputProvider contract. | All prior steps |
| 43 | API Documentation | XML docs on all public interface methods. Architecture guide. Subsystem ownership register. | All prior steps |

### Parallel Work Streams (after Phase 0)

| Stream 1 — Gameplay | Stream 2 — Presentation | Stream 3 — Systems |
|---------------------|------------------------|-------------------|
| Movement, Combat, Weapon | Camera, Animation, VFX | Audio, Input, UI, Save |
| Lead: Gameplay Engineer | Lead: Anim/Camera Engineer | Lead: Systems Engineer |

Phases 1-3 can overlap across streams once Phase 0 interfaces are locked.

---

## 8. Subsystem Specifications

### InputSubsystem

| Property | Value |
|----------|-------|
| **Class** | `InputSubsystem` |
| **Interface** | `IInputSystem` (implements) / `IInputProvider` (consumed) |
| **File** | `Subsystems/Input/InputSubsystem.cs` |
| **Responsibilities** | Reading from Unity Input System; assembling `PlayerInputFrame` struct each frame; routing to `IInputProvider` abstraction; rebinding, action map management |
| **Does NOT own** | Gameplay reactions to input; movement or combat logic; UI navigation input (separate action map) |
| **Dependencies** | `IInputProvider` (injected), `InputConfigSO`, `PlayerEventBus` |
| **Publishes** | `FireInputEvent`, `AimInputEvent`, `MoveInputEvent`, `LookInputEvent`, `JumpInputEvent`, `InteractInputEvent`, `WeaponSwitchInputEvent`, `InputContextChangedEvent` |
| **Consumes** | `IInputProvider` frame data |

### MovementSubsystem

| Property | Value |
|----------|-------|
| **Class** | `MovementSubsystem` |
| **Interface** | `IMovementSubsystem` |
| **File** | `Subsystems/Movement/MovementSubsystem.cs` |
| **Responsibilities** | CharacterController velocity; grounded/airborne detection; physics resolution; stamina resource; surface type detection; fall velocity tracking; all locomotion states |
| **Does NOT own** | Camera position; animation state; footstep audio; damage on landing; weapon ADS movement penalties |
| **Dependencies** | `IInputProvider`, `PlayerEventBus`, `MovementConfigSO`, `CharacterController` |
| **Modules** | `MovementStateMachine`, `StaminaModule`, `SurfaceDetectionModule`, `FallDamageModule`, `HeadBobModule`, `LeanModule` |
| **Publishes** | `MovementStateChangedEvent`, `PlayerLandedEvent`, `PlayerJumpedEvent`, `SurfaceChangedEvent`, `StaminaChangedEvent` |
| **Consumes** | `PlayerInputFrame` (via IInputProvider) |

### ParkourSubsystem

| Property | Value |
|----------|-------|
| **Class** | `ParkourSubsystem` |
| **Interface** | `IParkourSubsystem` |
| **File** | `Subsystems/Parkour/ParkourSubsystem.cs` |
| **Responsibilities** | Environment geometry probing for vaultable/mantleable surfaces; transition into/out of vault and mantle movement; climb ledge detection and execution |
| **Does NOT own** | Movement physics during parkour (delegates to MovementSubsystem states); animation (event-driven); input reading |
| **Dependencies** | `PlayerEventBus`, `ParkourConfigSO`, `ParkourDetector` |
| **Publishes** | `ParkourActionStartedEvent`, `ParkourActionCompletedEvent`, `ParkourActionFailedEvent` |
| **Consumes** | Movement state, `PlayerInputFrame` (mantle/vault input) |

### CombatSubsystem

| Property | Value |
|----------|-------|
| **Class** | `CombatSubsystem` |
| **Interface** | `ICombatSubsystem` / `IHealthSubsystem` |
| **File** | `Subsystems/Combat/CombatSubsystem.cs` |
| **Responsibilities** | Health pool; damage validation and application; invincibility frames; death/revive lifecycle; melee hitbox timing; combo sequencing; hit reactions |
| **Does NOT own** | Which weapon caused damage (receives `DamageData` event); respawn logic (game-layer); score/rewards; enemy AI reactions |
| **Dependencies** | `PlayerEventBus`, `CombatConfigSO` |
| **Modules** | `HealthModule`, `DamageModule` (Chain of Responsibility), `HitReactionModule`, `DeathModule`, `MeleeModule`, `ComboModule` |
| **Publishes** | `DamageReceivedEvent`, `HealthChangedEvent`, `PlayerDiedEvent`, `PlayerRespawnedEvent`, `MeleeHitEvent`, `ComboAdvancedEvent` |
| **Consumes** | `DamageReceivedEvent` (from any source — weapons, fall damage, environment) |

### WeaponSubsystem

| Property | Value |
|----------|-------|
| **Class** | `WeaponSubsystem` |
| **Interface** | `IWeaponSubsystem` |
| **File** | `Subsystems/Weapon/WeaponSubsystem.cs` |
| **Responsibilities** | Active weapon slot; weapon equip/unequip transitions; per-weapon module state; firing logic; reload state machines; recoil generation; ADS management; ammo tracking; hit detection raycasts; projectile spawning; weapon VFX; weapon audio |
| **Does NOT own** | Health damage application (emits DamageEvents for other entities' CombatSubsystem); animation (drives AnimationSubsystem via events); UI ammo display (via UICommsSubsystem) |
| **Dependencies** | `IInputProvider`, `PlayerEventBus`, `WeaponDefinition` assets (SO) |
| **Sub-components** | `WeaponManager`, `WeaponInstance` (per weapon), `ProjectileSystem` |
| **Modules** | `ShootingModule`, `ReloadModule`, `RecoilModule`, `ADSModule`, `AmmoModule`, `SpreadModule`, `WeaponVFXModule`, `HitDetectionModule` |
| **Publishes** | `WeaponFiredEvent`, `HitDetectedEvent`, `WeaponReloadedEvent`, `WeaponSwitchedEvent`, `WeaponEmptyEvent`, `AdsStateChangedEvent`, `RecoilEvent` |
| **Consumes** | `PlayerInputFrame` (fire, aim, reload, switch) |

### CameraSubsystem

| Property | Value |
|----------|-------|
| **Class** | `CameraSubsystem` |
| **Interface** | `ICameraSubsystem` |
| **File** | `Subsystems/Camera/CameraSubsystem.cs` |
| **Responsibilities** | Camera Transform ownership; active camera mode (FPS/TPS); FOV management; lean offset integration; recoil offset integration; camera shake queue; pitch/yaw limits |
| **Does NOT own** | Player position (reads from MovementSubsystem output transform); input reading; game-specific cutscene cameras |
| **Dependencies** | `PlayerEventBus`, `CameraConfigSO`, LeanModule output (via shared data struct) |
| **Modes** | `ICameraMode` (Strategy pattern): `FPSCameraMode`, `TPSCameraMode` |
| **Modules** | `DynamicFOVModule`, `LeanCameraModule`, `RecoilCameraModule`, `CameraShakeModule` |
| **Publishes** | `CameraModeChangedEvent`, `FOVChangedEvent` |
| **Consumes** | `MovementStateChangedEvent`, `AdsStateChangedEvent`, `RecoilEvent`, lean offset data |

### AnimationSubsystem

| Property | Value |
|----------|-------|
| **Class** | `AnimationSubsystem` |
| **Interface** | `IAnimationSubsystem` |
| **File** | `Subsystems/Animation/AnimationSubsystem.cs` |
| **Responsibilities** | All Animator parameter writes; blend tree configuration; animation layer weights; IK targets and weights; procedural animation layers (weapon sway, breathing offset); relay of Animator events to EventBus. **Full-body FPS rig** — complete skeleton with spine/head IK, foot IK, hand IK on weapon grip points. |
| **Does NOT own** | Gameplay decisions (ZERO if/else for gameplay); triggers are always reactions to events, never direct calls |
| **Dependencies** | `PlayerEventBus` (subscribes to ALL state events), `Animator`, `AnimationConfigSO` |
| **Modules** | `IKModule` (foot, hand, aim, spine), `ProceduralAnimModule` (weapon sway, breathing), `AnimationEventRelay` (Animator events -> EventBus) |
| **Publishes** | `FootstepAnimationEvent`, `ReloadCompleteAnimationEvent`, `MeleeHitWindowOpenedEvent`, `MeleeHitWindowClosedEvent` |
| **Consumes** | `MovementStateChangedEvent`, `WeaponFiredEvent`, `WeaponReloadedEvent`, `AdsStateChangedEvent`, `HealthChangedEvent`, `PlayerDiedEvent`, `RecoilEvent` |

### PlayerAudioSubsystem

| Property | Value |
|----------|-------|
| **Class** | `PlayerAudioSubsystem` |
| **Interface** | `IAudioSubsystem` |
| **File** | `Subsystems/Audio/PlayerAudioSubsystem.cs` |
| **Responsibilities** | All audio sourced from the player entity; surface-aware footstep scheduling; exertion-based breathing; pain/death/effort voice lines; requests to MainAudioManager for pooled sources |
| **Does NOT own** | Weapon audio (owned by WeaponVFXModule); environment audio; music; does not create or manage AudioSources directly |
| **Dependencies** | `PlayerEventBus`, `IMainAudioManager`, surface type data (via event payload) |
| **Modules** | `FootstepModule` (surface-aware), `BreathingModule` (stamina-driven), `CombatVoiceModule` (pain, death, effort) |
| **Publishes** | None (pure consumer) |
| **Consumes** | `MovementStateChangedEvent`, `SurfaceChangedEvent`, `StaminaChangedEvent`, `HealthChangedEvent`, `PlayerDiedEvent`, `DamageReceivedEvent` |

### InteractionSubsystem

| Property | Value |
|----------|-------|
| **Class** | `InteractionSubsystem` |
| **Interface** | `IInteractionSubsystem` |
| **File** | `Subsystems/Interaction/InteractionSubsystem.cs` |
| **Responsibilities** | Overlap detection for IInteractable objects; maintaining current interactable candidate; calling IInteractable.Interact() on player request; emitting interaction events |
| **Does NOT own** | What happens on interaction (each IInteractable's concern); UI prompt display (event-driven to UICommsSubsystem) |
| **Dependencies** | `IInputProvider` (interact button), `PlayerEventBus`, `InteractionConfigSO`, `InteractionDetector` |
| **Publishes** | `InteractionCandidateChangedEvent`, `InteractionStartedEvent`, `InteractionCompletedEvent` |
| **Consumes** | `InteractInputEvent` |

### UICommsSubsystem

| Property | Value |
|----------|-------|
| **Class** | `UICommsSubsystem` |
| **Interface** | `IUIBridgeSubsystem` |
| **File** | `Subsystems/UI/UICommsSubsystem.cs` |
| **Responsibilities** | Subscribing to player events; translating events into SO data channel updates that UI elements directly observe; decoupling HUD from all gameplay subsystems |
| **Does NOT own** | UI rendering logic; layout; any gameplay decisions; it is a pure translator |
| **Dependencies** | `PlayerEventBus`, Data channel SOs |
| **Channels** | `HealthDataChannel`, `AmmoDataChannel`, `StaminaDataChannel` |
| **Publishes** | Writes to data channels (SO reactive variables) |
| **Consumes** | `HealthChangedEvent`, `WeaponFiredEvent`, `WeaponReloadedEvent`, `WeaponSwitchedEvent`, `StaminaChangedEvent`, `PlayerDiedEvent` |

---

## 9. State Machine Architecture

### MetaStateMachine (Outermost Layer)
Governs player entity's top-level behavioural mode. All other state machines only run within the **ALIVE** meta-state.

```
MetaStateMachine
├── ALIVE          → LocomotionHSM, CombatHSM, WeaponHSM all active
├── DEAD           → All input disabled, camera enters death mode
├── IN_VEHICLE     → MovementSubsystem replaced by VehicleMovementSubsystem
└── IN_CUTSCENE    → Scripted animation, input blocked, camera under cinematic control
```

### LocomotionHierarchicalStateMachine

```
LocomotionHSM
├── GROUNDED (parent)
│   ├── IDLE
│   ├── WALK
│   ├── SPRINT
│   ├── CROUCH (sub-HSM: CROUCH_IDLE, CROUCH_WALK)
│   ├── PRONE (transition only from Crouch or Slide)
│   └── SLIDE (sprint + crouch on grounded)
├── AIRBORNE (parent)
│   ├── JUMPING (initial upward velocity)
│   ├── APEX (near-zero vertical velocity, coyote time)
│   └── FALLING (negative velocity)
├── MANTLE (animation-driven, input locked)
├── VAULT (animation-driven, input locked)
├── CLIMBING (wall/rope: CLIMBING_IDLE, CLIMBING_MOVING)
├── LADDER (axis-restricted: LADDER_IDLE, LADDER_MOVING)
└── SWIMMING (buoyancy-driven: SURFACE_SWIM, UNDERWATER)
```

**Key Transition Rules:**
- GROUNDED -> AIRBORNE: `!isGrounded && !isOnLadder`
- AIRBORNE -> GROUNDED: `isGrounded && landingVelocity within threshold`
- SPRINT -> SLIDE: `crouchInput && currentSpeed > slideMinSpeed`
- SLIDE -> CROUCH: `slideVelocity < slideEndThreshold`
- AIRBORNE -> MANTLE: `ParkourDetector.DetectMantle() && mantleInput`
- ANY -> DEAD: MetaStateMachine transition (overrides all)

### Weapon State Machine (Per WeaponInstance)

```
WeaponStateMachine
├── HOLSTERED (not active weapon)
├── EQUIPPING (animation plays, cannot fire)
├── IDLE (ready, no user input)
├── FIRING
│   ├── SEMI (single shot, returns to IDLE)
│   └── AUTO (continuous, held)
├── ADS (aimed-down-sights, sub-states mirror IDLE/FIRING)
├── RELOADING (locked until complete or interrupted)
│   ├── MAG_OUT
│   ├── MAG_IN
│   └── CHAMBER (bolt-action/pump variants)
├── SWITCHING (weapon swap in progress)
└── NO_AMMO (visual/audio feedback, prevents FIRING entry)
```

### Combat State Machine

```
CombatStateMachine
├── IDLE
├── ATTACKING
├── RECOVERING
└── STAGGERED
```

### HSM Implementation Notes
- Each state is a class implementing `IState` with `Enter()`, `Tick(float dt)`, `Exit()`. No switch statements.
- Transitions are data — `StateTransition` = condition predicate + target state. Evaluated in priority order each tick.
- State enter/exit emit events automatically via base class — AnimationSubsystem and AudioSubsystem react.
- **History states** for sub-HSMs — returning from Vault/Mantle to Grounded remembers last active sub-state (idle/walk/sprint).
- **No `GameObject.SetActive`** in state transitions — state machines operate on data, not scene graph.

---

## 10. Communication Tiers

### Tier 1: Direct Call (Mediator Pattern)

| Property | Value |
|----------|-------|
| **Mechanism** | Interface method call via PlayerEntity or direct owned-reference |
| **When to use** | Same-frame, guaranteed delivery. Orchestration calls from PlayerEntity to subsystems. Tight owner->owned dependency. |
| **Examples** | `movementSubsystem.ApplyExternalForce(v)`, `cameraSubsystem.SetMode(CameraMode.TPS)` |
| **Risk if overused** | Creates coupling. Only acceptable from the mediator (PlayerEntity) downward, or within an owner to its own modules. |

### Tier 2: EventBus (Typed Structs, Deferred Publishing)

| Property | Value |
|----------|-------|
| **Mechanism** | Typed struct events on PlayerEventBus. Zero-allocation when using value type events. |
| **When to use** | Cross-subsystem reactions. Producer doesn't need to know consumers. Any number of listeners. Async reaction acceptable. |
| **Examples** | `eventBus.Emit(new PlayerLandedEvent{...})` -> Audio plays landing sound, Animation triggers land anim, Combat applies fall damage |
| **Risk if overused** | Hot paths get overhead. Reserve for meaningful state transitions, not per-frame data. |

### Tier 3: Data Channels (ScriptableObject Variables)

| Property | Value |
|----------|-------|
| **Mechanism** | ScriptableObject reactive variables. Any asset can hold a reference. Survives scene changes. |
| **When to use** | UI data binding. Persistent state across scenes. Consumer shouldn't be in same scene dependency graph as producer. |
| **Examples** | `HealthDataChannel` (SO) <- CombatSubsystem writes -> HUD HealthBar reads (no scene dep) -> GameOver screen reads |
| **Risk if overused** | Not suitable for high-frequency per-frame data. SO writes incur minor overhead vs raw float. |

### End-to-End Communication Flows

**Player fires weapon:**
```
InputSubsystem -> PlayerInputFrame { firePressed = true }
WeaponSubsystem -> ShootingModule.TryFire()
HitDetectionModule.PerformRaycast() -> [hit registered]
Emit(WeaponFiredEvent)         [Tier 2] -> Animation, Audio, VFX, Recoil, Ammo, UIComms
Emit(DamageDealtEvent)         [Tier 2] -> Target's CombatSubsystem, Animation, Audio
UICommsSubsystem -> AmmoDataChannel    [Tier 3] -> HUD updates
```

**Player dies:**
```
CombatSubsystem -> HealthModule.ApplyDamage() -> HP = 0
DeathModule -> Emit(PlayerDiedEvent)  [Tier 2]
  -> MovementSubsystem: DisableInput(), enter death state
  -> WeaponSubsystem: DisableWeapons()
  -> AnimationSubsystem: death animation
  -> PlayerAudioSubsystem: death voice line
  -> CameraSubsystem: death camera mode
  -> UICommsSubsystem: death flag to channel [Tier 3]
  -> GameLayer: respawn, score (NOT framework concern)
```

**Weapon reload:**
```
InputSubsystem -> inputFrame { reloadPressed = true }
WeaponSubsystem -> ReloadModule.TryReload()
ReloadModule -> WeaponStateMachine.Transition(RELOADING)
Emit(ReloadStartedEvent) [Tier 2] -> Animation plays reload, Audio plays sound, UI shows indicator
AnimationEventRelay -> Emit(ReloadCompleteAnimEvent) [Tier 2]
ReloadModule -> AmmoModule.RefillMagazine()
UICommsSubsystem -> updates AmmoDataChannel [Tier 3]
```

---

## 11. Missing Tasks / Gaps

### Dodge State (Step 36)
- Directional dodge/roll with invulnerability frames
- Stamina burst cost
- Housed in EVASIVE super-state within LocomotionHSM
- Was missing from ClickUp; has been added to Session 6 in execution plan

### Framework.Shared Layer
- `TypedEventChannel.cs` — generic ScriptableObject event channel
- `ObjectPoolManager.cs` — pooled object management
- `MainAudioManager.cs` — game-wide audio manager (separate from player audio)
- Assembly: `Framework.Shared.asmdef`

### Persistence Scaffolding (Session 14)
- `ISaveable.cs` interface
- `PlayerSaveData.cs` data structure
- `[AuthoritativeState]` attribute for marking serializable runtime state
- Snapshot interfaces on all owning subsystems

### Multiplayer Scaffolding (Session 15)
- `INetworkAdapter.cs` — adapter interface for networking layer
- `NetworkInputProvider.cs` — implements `IInputProvider` for network authority
- Multiplayer boundary audit (review all `[AuthoritativeState]` tags)
- Client-predicted path documentation
- NetworkInputProvider implementation contract for networking team

---

## Appendix: Key Performance Rules

- **Zero GC allocations in Tick/FixedTick/LateTick** — profiling requirement, not recommendation
- **Pre-hash all Animator parameter names** — `Animator.StringToHash()` at init, cache as static int
- **Dirty-flag pattern** on Animator parameter writes — only write when values change
- **Surface detection rate-limited** — runs every N frames, not every frame
- **InteractionDetector uses OverlapSphereNonAlloc** — pre-allocated buffer, not per-call allocation
- **No LINQ in any Tick() method**
- **Event bus subscriber lists are pre-allocated arrays**, not `List<T>`
- **Transition conditions must be O(1)** — pure data comparisons, no physics queries
- **AudioSource pool** — never `PlayClipAtPoint`
- **Object pool for projectiles** — pool-first from start, not retrofitted
