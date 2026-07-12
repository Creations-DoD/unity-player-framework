# Player Framework — Interface Design Document Summary

> **Source:** PlayerFramework_IDD.html (2956 lines, v1.0)
> **Purpose:** Complete contract reference for all enums, structs, events, interfaces, and configs.
> **Rule:** Architecture doc naming wins over IDD naming wherever they conflict.
> **Namespace:** `PlayerFramework.Core.Contracts` / `PlayerFramework.Core.Events`

---

## 1. Document Overview

The IDD defines the complete contracts layer for the AAA Player Framework:

- **21 Enums** — Stable named identifiers across all subsystems
- **4 Data Structs** — Readonly value-type snapshots shared between subsystems
- **38 Event Structs** — Typed, allocation-free events on the IEventBus
- **16 Interfaces** — The only dependencies subsystems may take on each other
- **6 Config ScriptableObjects** — Data-driven tunables (zero hardcoded values)

**Relationship to Architecture Doc:**
The Architecture doc defines system-level design, folder structure, tick order, communication tiers, and development phases. The IDD defines the exact C# contracts (method signatures, field types, event payloads) that implement that design. During implementation, the IDD is the source of truth for *what* to build; the Architecture doc is the source of truth for *how* to wire it together.

**Naming Convention:**
Architecture doc naming wins where they conflict (see Architecture Summary Section 1 for the three overridden names).

---

## 2. All Enums (21)

### Player State

| Enum | Values | Subsystem | File |
|------|--------|-----------|------|
| `PlayerState` | `Alive`, `Dead`, `InVehicle`, `InCutscene`, `Spectating` | MetaStateMachine | `Core/Enums/PlayerState.cs` |

### Movement

| Enum | Values | Subsystem | File |
|------|--------|-----------|------|
| `MovementState` | `Idle`, `Walk`, `Sprint`, `Crouch`, `CrouchWalk`, `Prone`, `ProneCrawl`, `Slide`, `Jump`, `Fall`, `LedgeHang`, `SwimSurface`, `SwimUnderwater`, `Wade`, `Climb`, `Ladder`, `LedgeMantle`, `Vault`, `Mantle`, `Dodge`, `Lean`, `Knockback`, `Stunned` | MovementSubsystem | `Core/Enums/MovementState.cs` |
| `ForceMode` | `Force`, `Impulse`, `VelocityChange`, `Acceleration` | MovementSubsystem | `Core/Enums/ForceMode.cs` |
| `SurfaceType` | `Concrete`, `Grass`, `Metal`, `Wood`, `Dirt`, `Sand`, `Water`, `Snow`, `Gravel`, `Glass`, `Carpet`, `Tile` | MovementSubsystem / Audio | `Core/Enums/SurfaceType.cs` |

### Combat

| Enum | Values | Subsystem | File |
|------|--------|-----------|------|
| `DamageType` | `Bullet`, `Melee`, `Explosion`, `Fall`, `Environment`, `StatusEffect`, `Instant` | CombatSubsystem | `Core/Enums/DamageType.cs` |
| `CombatState` | `Idle`, `Attacking`, `Recovering`, `Staggered` | CombatSubsystem | `Core/Enums/CombatState.cs` |
| `MeleeAttackType` | `Light`, `Heavy`, `Finisher`, `Shove` | CombatSubsystem | `Core/Enums/MeleeAttackType.cs` |

### Weapon

| Enum | Values | Subsystem | File |
|------|--------|-----------|------|
| `WeaponState` | `Unequipped`, `Equipping`, `Idle`, `Firing`, `FiringADS`, `Reloading`, `Unequipping`, `NoAmmo` | WeaponSubsystem | `Core/Enums/WeaponState.cs` |
| `FireMode` | `Single`, `Burst`, `Auto` | WeaponSubsystem | `Core/Enums/FireMode.cs` |
| `ReloadType` | `Tactical`, `Empty` | WeaponSubsystem | `Core/Enums/ReloadType.cs` |

### Camera

| Enum | Values | Subsystem | File |
|------|--------|-----------|------|
| `CameraMode` | `FPS`, `TPS`, `Cinematic`, `Spectator` | CameraSubsystem | `Core/Enums/CameraMode.cs` |

### Input

| Enum | Values | Subsystem | File |
|------|--------|-----------|------|
| `InputContext` | `Gameplay`, `UI`, `Ladder`, `Cutscene`, `Vehicle`, `Dead`, `Dialogue` | InputSubsystem | `Core/Enums/InputContext.cs` |

### Interaction

| Enum | Values | Subsystem | File |
|------|--------|-----------|------|
| `InteractionType` | `Instant`, `Hold`, `Toggle`, `Repeatable` | InteractionSubsystem | `Core/Enums/InteractionType.cs` |

### Audio

| Enum | Values | Subsystem | File |
|------|--------|-----------|------|
| `VoiceLineType` | `Pain`, `Death`, `Effort`, `Breathing`, `Empty`, `Melee`, `Reload` | PlayerAudioSubsystem | `Core/Enums/VoiceLineType.cs` |
| `AudioCategory` | `Footstep`, `Voice`, `Weapon`, `Ambient` | PlayerAudioSubsystem | `Core/Enums/AudioCategory.cs` |

### Animation

| Enum | Values | Subsystem | File |
|------|--------|-----------|------|
| `IKTarget` | `LeftHand`, `RightHand`, `LeftFoot`, `RightFoot`, `LookAt`, `Body` | AnimationSubsystem | `Core/Enums/IKTarget.cs` |
| `IKHint` | `LeftElbow`, `RightElbow`, `LeftKnee`, `RightKnee` | AnimationSubsystem | `Core/Enums/IKHint.cs` |
| `ProceduralOffsetType` | `WeaponSway`, `BreathingLayer`, `RecoilLayer`, `LandingImpact` | AnimationSubsystem | `Core/Enums/ProceduralOffsetType.cs` |

### UI

| Enum | Values | Subsystem | File |
|------|--------|-----------|------|
| `CrosshairStyle` | `Default`, `ADS`, `Interacting`, `Hidden` | UICommsSubsystem | `Core/Enums/CrosshairStyle.cs` |
| `UIEventType` | `HealthChanged`, `AmmoChanged`, `StaminaChanged`, `PlayerDied`, `WeaponSwitched` | UICommsSubsystem | `Core/Enums/UIEventType.cs` |

### Parkour

| Enum | Values | Subsystem | File |
|------|--------|-----------|------|
| `ParkourActionType` | `None`, `Vault`, `Mantle`, `Climb`, `LedgeHang` | ParkourSubsystem | `Core/Enums/ParkourActionType.cs` |

---

## 3. All Data Structs (4)

All data structs are `readonly struct` — zero heap allocation, immutable, passed by value.

### PlayerInputFrame

**File:** `Core/Data/PlayerInputFrame.cs`
**Creator:** InputSubsystem (IInputProvider)
**Consumers:** All subsystems (via IInputSystem.CurrentFrame)

| Type | Field | Description |
|------|-------|-------------|
| `int` | FrameId | Monotonically increasing frame counter (multiplayer reconciliation/replay) |
| `float` | Timestamp | Time.fixedTime at capture |
| `Vector2` | MoveAxis | WASD / left stick (magnitude preserved, deadzone in HumanInputProvider) |
| `Vector2` | LookAxis | Mouse delta / right stick (raw, sensitivity applied in CameraSystem) |
| `bool` | JumpPressed | True for exactly one frame on first press |
| `bool` | JumpHeld | True while held (variable-height jump support) |
| `bool` | SprintHeld | True while sprint button held |
| `bool` | CrouchPressed | True for exactly one frame (toggle behaviour) |
| `bool` | CrouchHeld | True while held (hold-to-crouch behaviour) |
| `bool` | PronePressed | True for exactly one frame |
| `bool` | FirePressed | True for exactly one frame on first press |
| `bool` | FireHeld | True while held (automatic fire support) |
| `bool` | AimPressed | True for exactly one frame on ADS toggle |
| `bool` | AimHeld | True while held (hold-to-aim) |
| `bool` | ReloadPressed | True for exactly one frame |
| `bool` | InteractPressed | True for exactly one frame |
| `bool` | MeleePressed | True for exactly one frame |
| `int` | WeaponSwitchAxis | Scroll wheel delta / number key (-1=prev, +1=next, 0=none) |
| `bool` | DodgePressed | True for exactly one frame |

**Rules:** No heap allocation. Immutable after creation. One instance per tick, cached by InputSubsystem.

### PlayerKinematicState

**File:** `Core/Data/PlayerKinematicState.cs`
**Creator:** MovementSubsystem
**Consumers:** CameraSubsystem, AnimationSubsystem, UICommsSubsystem, PlayerAudioSubsystem

| Type | Field | Description |
|------|-------|-------------|
| `int` | FrameId | Simulation frame this snapshot corresponds to |
| `Vector3` | Position | World-space character position at base of capsule |
| `Quaternion` | Rotation | World-space yaw rotation (camera handles pitch) |
| `Vector3` | Velocity | Current velocity in m/s, world-space |
| `bool` | IsGrounded | True if in contact with walkable surface |
| `Vector3` | GroundNormal | Surface normal underfoot (Vector3.up when airborne) |
| `float` | SlopeAngle | Degrees between GroundNormal and Vector3.up |
| `MovementState` | CurrentMovementState | Active leaf state in locomotion HSM |
| `SurfaceType` | CurrentSurface | Material type underfoot |
| `float` | StaminaNormalized | 0-1 stamina value (0 = exhausted) |
| `float` | FallVelocity | Downward velocity magnitude (positive = falling) |

**Rules:** Published once per FixedTick. All subsystems read this, never query CharacterController directly.

### DamageData

**File:** `Core/Data/DamageData.cs`
**Creator:** WeaponSubsystem (HitDetectionModule), FallDamageModule, any damage source
**Consumers:** CombatSubsystem (DamageProcessor pipeline)

| Type | Field | Description |
|------|-------|-------------|
| `float` | Amount | Raw damage before armor/resistance (always positive) |
| `DamageType` | Type | Category for resistance calculation |
| `Vector3` | HitPoint | World-space impact point |
| `Vector3` | HitDirection | Normalised direction from source to target |
| `Vector3` | HitNormal | World-space surface normal at hit point |
| `string` | SourceId | Damage source identifier (kill attribution, analytics) |
| `bool` | BypassArmor | Skip armor reduction step |
| `bool` | BypassInvincibility | Apply even during invincibility frames |
| `float` | ArmorPenetration | 0-1 fraction of armor ignored (0=full armor, 1=no armor) |

**Rules:** Never mutated in-flight. Pipeline modifiers read but do not modify this struct.

### HitData

**File:** `Core/Data/HitData.cs`
**Creator:** WeaponSubsystem (HitDetectionModule)
**Consumers:** CombatSubsystem (via HitDetectedEvent)

| Type | Field | Description |
|------|-------|-------------|
| `Vector3` | HitPoint | World-space contact point |
| `Vector3` | HitNormal | Surface normal at contact point |
| `float` | Distance | Muzzle-to-hit distance in metres |
| `GameObject` | HitObject | Struck GameObject (null for terrain) |
| `string` | HitTag | Unity tag of hit object (material-based impact effects) |
| `bool` | IsHeadshot | True if hit collider tagged as headshot zone |

**Rules:** Part of HitDetectedEvent payload alongside DamageData.

---

## 4. All Event Structs (38)

All events are value types (structs) — zero heap allocation. Namespace: `PlayerFramework.Core.Events`.

### Input Events (8)

| Event | Fields | Publisher | Consumers |
|-------|--------|-----------|-----------|
| `FireInputEvent` | `bool IsPressed`, `bool IsHeld`, `int FrameId` | InputSystem | WeaponSystem |
| `AimInputEvent` | `bool IsADS`, `int FrameId` | InputSystem | WeaponSystem, CameraSystem |
| `MoveInputEvent` | `Vector2 Axis`, `int FrameId` | InputSystem | MovementSystem |
| `LookInputEvent` | `Vector2 Delta`, `int FrameId` | InputSystem | CameraSystem |
| `JumpInputEvent` | `bool IsHeld`, `int FrameId` | InputSystem | MovementSystem |
| `InteractInputEvent` | `bool IsHeld`, `int FrameId` | InputSystem | InteractionSystem |
| `WeaponSwitchInputEvent` | `int TargetSlot`, `int FrameId` | InputSystem | WeaponSystem |
| `InputContextChangedEvent` | `InputContext Previous`, `InputContext Current` | InputSystem | AnimationSystem, CameraSystem, UIBridgeSystem |

### Movement Events (5)

| Event | Fields | Publisher | Consumers |
|-------|--------|-----------|-----------|
| `MovementStateChangedEvent` | `MovementState PreviousState`, `MovementState NewState`, `float Timestamp` | MovementSystem | CameraSystem, AnimationSystem, PlayerAudioSystem, UIBridgeSystem |
| `PlayerLandedEvent` | `float ImpactVelocity`, `SurfaceType Surface`, `Vector3 LandPosition`, `float FallDamageDealt` | MovementSystem | CombatSystem, AnimationSystem, PlayerAudioSystem, CameraSystem |
| `PlayerJumpedEvent` | `Vector3 Position`, `float StaminaCost` | MovementSystem | AnimationSystem, PlayerAudioSystem |
| `SurfaceChangedEvent` | `SurfaceType PreviousSurface`, `SurfaceType NewSurface` | MovementSystem | PlayerAudioSystem |
| `StaminaChangedEvent` | `float Previous`, `float Current`, `bool IsExhausted`, `bool IsRecovered` | MovementSystem | UIBridgeSystem, PlayerAudioSystem, AnimationSystem |

### Weapon Events (7)

| Event | Fields | Publisher | Consumers |
|-------|--------|-----------|-----------|
| `WeaponFiredEvent` | `string WeaponId`, `Vector3 MuzzlePosition`, `Vector3 FireDirection`, `int ShotsRemaining`, `int FrameId` | WeaponSystem | AnimationSystem, PlayerAudioSystem, WeaponVFXHandler, RecoilSystem, AmmoSystem, UIBridgeSystem |
| `HitDetectedEvent` | `HitData Hit`, `DamageData Damage`, `string WeaponId`, `GameObject Target` | WeaponSystem (HitDetection) | CombatSystem |
| `WeaponReloadedEvent` | `string WeaponId`, `ReloadType Type`, `int NewMagazineCount`, `int ReserveAmmoRemaining` | WeaponSystem | AnimationSystem, PlayerAudioSystem, UIBridgeSystem |
| `WeaponSwitchedEvent` | `int FromSlot`, `int ToSlot`, `string NewWeaponId` | WeaponSystem | AnimationSystem, PlayerAudioSystem, CameraSystem, UIBridgeSystem |
| `WeaponEmptyEvent` | `string WeaponId` | WeaponSystem | PlayerAudioSystem, UIBridgeSystem |
| `AdsStateChangedEvent` | `bool IsADS`, `float ZoomLevel`, `float SensitivityMultiplier` | WeaponSystem | CameraSystem, MovementSystem, AnimationSystem |
| `RecoilEvent` | `Vector2 RecoilOffset`, `Vector2 RotationOffset`, `float RecoverySpeed` | WeaponSystem (RecoilSystem) | CameraSystem, AnimationSystem |

### Combat & Health Events (6)

| Event | Fields | Publisher | Consumers |
|-------|--------|-----------|-----------|
| `DamageReceivedEvent` | `DamageData Source`, `float ActualDamage`, `float RemainingHealth`, `bool WasBlocked` | HealthSystem | CameraSystem, AnimationSystem, PlayerAudioSystem, UIBridgeSystem |
| `HealthChangedEvent` | `float PreviousHealth`, `float CurrentHealth`, `float MaxHealth`, `float Delta` | HealthSystem | UIBridgeSystem, AnimationSystem |
| `PlayerDiedEvent` | `string Cause`, `string KillerSourceId`, `Vector3 DeathPosition` | HealthSystem | MovementSystem, WeaponSystem, CameraSystem, AnimationSystem, PlayerAudioSystem, UIBridgeSystem |
| `PlayerRespawnedEvent` | `Vector3 RespawnPosition`, `float StartingHealth` | HealthSystem | MovementSystem, WeaponSystem, AnimationSystem, UIBridgeSystem |
| `MeleeHitEvent` | `GameObject Target`, `MeleeAttackType AttackType`, `DamageData Damage` | CombatSystem | AnimationSystem, PlayerAudioSystem |
| `ComboAdvancedEvent` | `int ComboIndex`, `int TotalSteps`, `float WindowRemaining` | CombatSystem | AnimationSystem, UIBridgeSystem |

### Parkour Events (3)

| Event | Fields | Publisher | Consumers |
|-------|--------|-----------|-----------|
| `ParkourActionStartedEvent` | `ParkourActionType ActionType`, `Vector3 StartPosition` | ParkourSystem | MovementSystem, AnimationSystem, PlayerAudioSystem |
| `ParkourActionCompletedEvent` | `ParkourActionType ActionType`, `Vector3 EndPosition` | ParkourSystem | MovementSystem, AnimationSystem |
| `ParkourActionFailedEvent` | `ParkourActionType ActionType`, `string Reason` | ParkourSystem | UIBridgeSystem |

### Camera Events (2)

| Event | Fields | Publisher | Consumers |
|-------|--------|-----------|-----------|
| `CameraModeChangedEvent` | `CameraMode PreviousMode`, `CameraMode NewMode` | CameraSystem | AnimationSystem, UIBridgeSystem |
| `FOVChangedEvent` | `float PreviousFOV`, `float CurrentFOV` | CameraSystem | UIBridgeSystem |

### Interaction Events (3)

| Event | Fields | Publisher | Consumers |
|-------|--------|-----------|-----------|
| `InteractionCandidateChangedEvent` | `bool HasCandidate`, `string InteractionLabel`, `InteractionType Type` | InteractionSystem | UIBridgeSystem |
| `InteractionStartedEvent` | `string InteractionLabel`, `float Duration` | InteractionSystem | AnimationSystem, UIBridgeSystem |
| `InteractionCompletedEvent` | `string InteractionLabel` | InteractionSystem | AnimationSystem, UIBridgeSystem |

### Animation Events (4)

| Event | Fields | Publisher | Consumers |
|-------|--------|-----------|-----------|
| `FootstepAnimationEvent` | `bool IsLeftFoot`, `SurfaceType Surface`, `float Intensity` | AnimationSystem (EventRelay) | PlayerAudioSystem |
| `ReloadCompleteAnimationEvent` | `string WeaponId`, `ReloadType Type` | AnimationSystem (EventRelay) | WeaponSystem (ReloadHandler) |
| `MeleeHitWindowOpenedEvent` | `MeleeAttackType AttackType` | AnimationSystem (EventRelay) | CombatSystem (MeleeSystem) |
| `MeleeHitWindowClosedEvent` | `MeleeAttackType AttackType` | AnimationSystem (EventRelay) | CombatSystem (MeleeSystem) |

---

## 5. All Interfaces (16)

**Note:** The IDD defines 16 interfaces. The Architecture doc lists 16 as well (with naming overrides applied). `IPlayerEntity` is referenced in method signatures (IInteractable) but not defined as a standalone interface section in the IDD.

### IPlayerSubsystem (Base Contract)

**File:** `Core/Interfaces/IPlayerSubsystem.cs`
**Implemented by:** All subsystems

| Member | Signature | Description |
|--------|-----------|-------------|
| Property | `bool IsEnabled { get; }` | True when active and receiving Tick calls |
| Method | `void Initialize(IServiceLocator services)` | Resolve deps from service locator, subscribe to events |
| Method | `void PostInitialize()` | Cross-subsystem wiring after all Init complete |
| Method | `void Enable()` | Activate subsystem, resume tick processing |
| Method | `void Disable()` | Deactivate subsystem, stop tick processing |
| Method | `void Tick(float deltaTime)` | Per-frame update (fixed priority order) |
| Method | `void FixedTick(float fixedDeltaTime)` | Fixed-rate physics update |
| Method | `void LateTick(float deltaTime)` | Post-frame update (camera, IK) |
| Method | `void Shutdown()` | Cleanup, unsubscribe events, release resources |

**Preconditions:** Initialize() called once; Enable/Disable follow lifecycle; Tick only when IsEnabled.

### IServiceLocator

**File:** `Core/Interfaces/IServiceLocator.cs`
**Implemented by:** (framework-internal implementation)

| Member | Signature | Description |
|--------|-----------|-------------|
| Method | `T GetService<T>() where T : class` | Retrieve registered service by type |
| Method | `void RegisterService<T>(T service) where T : class` | Register a service under type key T |
| Method | `bool HasService<T>() where T : class` | Check if service is registered |
| Method | `void UnregisterService<T>() where T : class` | Remove service registration |

**Preconditions:** Register during Construction phase; Resolve during Initialize(). Not a global singleton — scoped per player entity.

### IEventBus

**File:** `Core/Interfaces/IEventBus.cs`
**Implemented by:** PlayerEventBus

| Member | Signature | Description |
|--------|-----------|-------------|
| Method | `void Subscribe<T>(Action<T> handler) where T : struct` | Register handler for event type T |
| Method | `void Unsubscribe<T>(Action<T> handler) where T : struct` | Remove handler (reference equality) |
| Method | `void Publish<T>(T eventData) where T : struct` | Synchronously invoke all handlers for T |
| Method | `void PublishDeferred<T>(T eventData) where T : struct` | Queue event for next FlushDeferred() |
| Method | `void FlushDeferred()` | Deliver all queued deferred events |
| Method | `void SetChannelEnabled<T>(bool enabled) where T : struct` | Enable/disable delivery for event type T |

**Rules:** No re-entrant publishing for same type. Subscribe in Initialize()/Enable(), not in Tick(). Unsubscribe in Disable()/Shutdown().

### IInputProvider

**File:** `Core/Interfaces/IInputProvider.cs`
**Implemented by:** HumanInputProvider, AIInputProvider, NetworkInputProvider, NullInputProvider

| Member | Signature | Description |
|--------|-----------|-------------|
| Method | `PlayerInputFrame GetCurrentFrame()` | Returns input snapshot for current tick (allocation-free) |

**Purpose:** The multiplayer seam. Framework is agnostic to input source.

### IInputSystem (extends IPlayerSubsystem)

**File:** `Core/Interfaces/IInputSystem.cs`
**Implemented by:** InputSubsystem

| Member | Signature | Description |
|--------|-----------|-------------|
| Property | `PlayerInputFrame CurrentFrame { get; }` | Most recent input snapshot |
| Property | `InputContext ActiveContext { get; }` | Top of context stack |
| Method | `void PushContext(InputContext context)` | Push new input context (immediately active) |
| Method | `void PopContext()` | Restore previous context (no-op if stack has 1) |
| Method | `void SetInputProvider(IInputProvider provider)` | Replace input source at runtime |
| Method | `IInputProvider GetInputProvider()` | Return current provider (never null) |

**Publishes:** `InputContextChangedEvent`

### IMovementSystem (extends IPlayerSubsystem)

**File:** `Core/Interfaces/IMovementSystem.cs`
**Implemented by:** MovementSubsystem

| Member | Signature | Description |
|--------|-----------|-------------|
| Property | `PlayerKinematicState KinematicState { get; }` | Authoritative kinematic snapshot |
| Property | `MovementState CurrentState { get; }` | Active leaf state in locomotion HSM |
| Property | `bool IsGrounded { get; }` | On walkable surface |
| Property | `Vector3 Velocity { get; }` | Current world-space velocity (m/s) |
| Property | `float StaminaNormalized { get; }` | 0-1 stamina fraction |
| Property | `bool IsStaminaExhausted { get; }` | True when stamina reached 0 |
| Property | `SurfaceType CurrentSurface { get; }` | Material underfoot |
| Method | `void ApplyExternalForce(Vector3 force, ForceMode mode)` | Apply knockback, wind, launch pad force |
| Method | `void SetMovementEnabled(bool enabled)` | Lock/unlock locomotion input |
| Method | `void AddSpeedModifier(string sourceId, float multiplier)` | Multiplicative speed modifier (ADS, stamina, water) |
| Method | `void RemoveSpeedModifier(string sourceId)` | Remove named speed modifier |
| Method | `void TeleportTo(Vector3 position, Quaternion rotation)` | Instant move bypassing physics |
| Method | `bool CanTransitionTo(MovementState targetState)` | Query HSM transition validity |
| Method | `void ForceTransitionTo(MovementState targetState)` | Force state (bypasses guards, use sparingly) |

**Publishes:** `MovementStateChangedEvent`, `PlayerLandedEvent`, `PlayerJumpedEvent`, `SurfaceChangedEvent`, `StaminaChangedEvent`

### IParkourSystem (extends IPlayerSubsystem)

**File:** `Core/Interfaces/IParkourSystem.cs`
**Implemented by:** ParkourSubsystem

| Member | Signature | Description |
|--------|-----------|-------------|
| Property | `bool IsExecutingAction { get; }` | True during vault/mantle/climb |
| Property | `ParkourActionType CurrentAction { get; }` | Active action type |
| Method | `bool CanVault(out VaultData vaultData)` | Geometry query for vaultable obstacle |
| Method | `bool CanMantle(out MantleData mantleData)` | Geometry query for mantleable ledge |
| Method | `bool CanClimb(out ClimbData climbData)` | Detect climbable surface |
| Method | `void RequestVault()` | Initiate vault if valid |
| Method | `void RequestMantle()` | Initiate mantle if valid |
| Method | `void RequestClimb()` | Initiate climb if valid |
| Method | `void AbortCurrentAction()` | Immediate termination, return to normal movement |

**Publishes:** `ParkourActionStartedEvent`, `ParkourActionCompletedEvent`, `ParkourActionFailedEvent`

### IHealthSystem (extends IPlayerSubsystem)

**File:** `Core/Interfaces/IHealthSystem.cs`
**Implemented by:** CombatSubsystem (HealthModule)

| Member | Signature | Description |
|--------|-----------|-------------|
| Property | `float CurrentHealth { get; }` | Current HP (0 to MaxHealth) |
| Property | `float MaxHealth { get; }` | Maximum HP |
| Property | `float HealthNormalized { get; }` | 0-1 fraction |
| Property | `bool IsAlive { get; }` | True when HP > 0 |
| Property | `bool IsInvincible { get; }` | True during invincibility frames |
| Method | `void ApplyDamage(DamageData damage)` | Submit damage to pipeline (armor > resistance > invincibility > apply) |
| Method | `void ApplyHealing(float amount, string sourceId)` | Restore HP (direct, no pipeline) |
| Method | `void SetInvincible(bool invincible, float duration)` | Toggle invincibility (0 = permanent until cancelled) |
| Method | `void SetMaxHealth(float maxHealth, bool scaleCurrentHealth)` | Update max HP (proportional or clamp) |
| Method | `void Kill(string cause)` | Instant death bypassing pipeline |
| Method | `void Revive(float healthPercent)` | Restore from dead (0-1 fraction) |

**Publishes:** `DamageReceivedEvent`, `HealthChangedEvent`, `PlayerDiedEvent`, `PlayerRespawnedEvent`

### ICombatSystem (extends IPlayerSubsystem)

**File:** `Core/Interfaces/ICombatSystem.cs`
**Implemented by:** CombatSubsystem

| Member | Signature | Description |
|--------|-----------|-------------|
| Property | `CombatState CurrentState { get; }` | Active combat state |
| Property | `bool IsAttacking { get; }` | True during melee attack + hitbox active |
| Property | `int CurrentComboIndex { get; }` | 0-based combo step (-1 when not in combo) |
| Property | `float ComboWindowRemaining { get; }` | Seconds remaining in combo window |
| Method | `bool RequestMeleeAttack(MeleeAttackType attackType)` | Request melee (advances combo if window open) |
| Method | `void CancelCurrentAttack()` | Cancel active attack, reset combo |
| Method | `bool IsInComboWindow()` | True if combo input would advance chain |

**Publishes:** `MeleeHitEvent`, `ComboAdvancedEvent`

### IWeaponSystem (extends IPlayerSubsystem)

**File:** `Core/Interfaces/IWeaponSystem.cs`
**Implemented by:** WeaponSubsystem

| Member | Signature | Description |
|--------|-----------|-------------|
| Property | `IWeaponInstance ActiveWeapon { get; }` | Runtime state of equipped weapon |
| Property | `int ActiveSlotIndex { get; }` | 0-based slot (-1 if none) |
| Property | `WeaponState ActiveWeaponState { get; }` | Active weapon SM state |
| Property | `bool CanFire { get; }` | True if fire is valid |
| Property | `bool CanReload { get; }` | True if reload is valid |
| Property | `bool IsADS { get; }` | True if in ADS mode |
| Method | `void EquipWeapon(WeaponDefinitionSO definition, int slotIndex)` | Place weapon in slot |
| Method | `void UnequipWeapon(int slotIndex)` | Remove weapon from slot |
| Method | `bool HasWeaponInSlot(int slotIndex)` | Check if slot occupied |
| Method | `void SwitchToSlot(int slotIndex)` | Begin weapon switch sequence |
| Method | `void RequestFire()` | Full fire pipeline (state > ammo > spread > raycast > events) |
| Method | `void RequestReload()` | Auto-detect tactical vs empty reload |
| Method | `void SetADS(bool active)` | Enter/exit ADS mode |
| Method | `void AddAmmoToReserve(string calibre, int amount)` | Add ammo to global reserve |
| Method | `int GetReserveAmmo(string calibre)` | Query reserve ammo count |

**Publishes:** `WeaponFiredEvent`, `HitDetectedEvent`, `WeaponReloadedEvent`, `WeaponSwitchedEvent`, `WeaponEmptyEvent`, `AdsStateChangedEvent`, `RecoilEvent`

### ICameraSystem (extends IPlayerSubsystem)

**File:** `Core/Interfaces/ICameraSystem.cs`
**Implemented by:** CameraSubsystem

| Member | Signature | Description |
|--------|-----------|-------------|
| Property | `CameraMode ActiveMode { get; }` | Current camera rig |
| Property | `float CurrentFOV { get; }` | Effective FOV with all overrides |
| Property | `Vector3 CameraPosition { get; }` | World-space camera position |
| Property | `Quaternion CameraRotation { get; }` | World-space camera rotation |
| Property | `Ray AimRay { get; }` | Camera forward ray (weapon aim direction) |
| Method | `void SetMode(CameraMode mode)` | Switch camera rig (cross-fade transition) |
| Method | `void AddModifier(ICameraModifier modifier)` | Add modifier to stack (priority-ordered) |
| Method | `void RemoveModifier(ICameraModifier modifier)` | Remove modifier (no-op if not found) |
| Method | `void RequestShake(float trauma)` | Add shake trauma (0-1, intensity = trauma^2) |
| Method | `void SetFOVOverride(float targetFOV, float blendDuration, string sourceId)` | Register FOV override (lowest FOV wins) |
| Method | `void ClearFOVOverride(string sourceId)` | Remove FOV override |
| Method | `void SetPitchLimits(float minDegrees, float maxDegrees)` | Update pitch clamping range |
| Method | `void SetSensitivity(float horizontal, float vertical)` | Update look sensitivity |

**Publishes:** `CameraModeChangedEvent`, `FOVChangedEvent`

### IAnimationSystem (extends IPlayerSubsystem)

**File:** `Core/Interfaces/IAnimationSystem.cs`
**Implemented by:** AnimationSubsystem

| Member | Signature | Description |
|--------|-----------|-------------|
| Property | `bool IsTransitioning { get; }` | True during cross-fade |
| Method | `void SetLayerWeight(int layerIndex, float weight, float blendTime)` | Set Animator layer weight |
| Method | `float GetLayerWeight(int layerIndex)` | Query layer weight |
| Method | `void SetIKTarget(IKTarget target, Transform targetTransform, float positionWeight, float rotationWeight)` | Set IK goal target |
| Method | `void ClearIKTarget(IKTarget target)` | Clear IK goal |
| Method | `void SetIKHint(IKHint hint, Transform hintTransform, float weight)` | Set limb IK pole vector |
| Method | `void TriggerAnimation(string triggerName)` | Set Animator trigger (cached hash) |
| Method | `void SetBool(string parameterName, bool value)` | Set Animator bool (cached hash) |
| Method | `void SetFloat(string parameterName, float value, float dampTime)` | Set Animator float with damping |
| Method | `void SetProceduralOffset(ProceduralOffsetType type, Vector3 positionOffset, Quaternion rotationOffset)` | Set procedural anim offset (additive) |
| Method | `void ClearProceduralOffset(ProceduralOffsetType type)` | Reset procedural offset to identity |

**Publishes:** `FootstepAnimationEvent`, `ReloadCompleteAnimationEvent`, `MeleeHitWindowOpenedEvent`, `MeleeHitWindowClosedEvent`

### IPlayerAudioSystem (extends IPlayerSubsystem)

**File:** `Core/Interfaces/IPlayerAudioSystem.cs`
**Implemented by:** PlayerAudioSubsystem

| Member | Signature | Description |
|--------|-----------|-------------|
| Property | `bool IsBreathingActive { get; }` | True while breathing module playing |
| Property | `float FootstepVolume { get; set; }` | Footstep volume multiplier [0,1] |
| Property | `float VoiceVolume { get; set; }` | Voice line volume multiplier [0,1] |
| Method | `void SetSurfaceContext(SurfaceType surface)` | Set current surface for footstep clips |
| Method | `void SetMovementIntensity(float normalizedIntensity)` | Set movement intensity [0,1] |
| Method | `void SetBreathingState(float exertionLevel)` | Set breathing intensity [0,1] |
| Method | `void RequestVoiceLine(VoiceLineType type, bool interruptCurrent)` | Play voice line (queue if busy) |
| Method | `void StopCurrentVoiceLine()` | Stop current voice line and clear queue |

**Publishes:** None (pure event consumer)

### IInteractionSystem (extends IPlayerSubsystem)

**File:** `Core/Interfaces/IInteractionSystem.cs`
**Implemented by:** InteractionSubsystem

| Member | Signature | Description |
|--------|-----------|-------------|
| Property | `IInteractable CurrentCandidate { get; }` | Highest-priority interactable in range |
| Property | `bool HasCandidate { get; }` | True when interactable in range |
| Property | `bool IsInteracting { get; }` | True during hold interaction |
| Property | `float InteractionProgress { get; }` | 0-1 hold interaction progress |
| Method | `bool TryInteract()` | Execute interaction (instant or begin hold) |
| Method | `void CancelInteraction()` | Cancel hold interaction in progress |
| Method | `void SetDetectionRange(float range)` | Update detection radius |
| Method | `void SetDetectionAngle(float degrees)` | Update detection cone half-angle |

**Publishes:** `InteractionCandidateChangedEvent`, `InteractionStartedEvent`, `InteractionCompletedEvent`

### IUIBridgeSystem (extends IPlayerSubsystem)

**File:** `Core/Interfaces/IUIBridgeSystem.cs`
**Implemented by:** UICommsSubsystem

| Member | Signature | Description |
|--------|-----------|-------------|
| Property | `bool IsHUDVisible { get; }` | True when HUD shown |
| Method | `void SetHUDVisible(bool visible)` | Show/hide entire HUD |
| Method | `void SetCrosshairStyle(CrosshairStyle style)` | Change crosshair visual |
| Method | `void ShowInteractionPrompt(string actionName, string description, string bindingDisplayString)` | Display interaction prompt |
| Method | `void HideInteractionPrompt()` | Hide interaction prompt |
| Method | `void ShowDamageIndicator(Vector3 worldDamageSourcePosition)` | Trigger directional damage indicator |

### IInteractable

**File:** `Core/Interfaces/IInteractable.cs`
**Implemented by:** Game-layer world objects (Visitor pattern)

| Member | Signature | Description |
|--------|-----------|-------------|
| Property | `string InteractionLabel { get; }` | Short label for prompt (e.g. "Pick up") |
| Property | `string InteractionDescription { get; }` | Object description (e.g. "Health Pack") |
| Property | `bool IsInteractable { get; }` | Whether interaction is currently possible |
| Property | `float InteractionDuration { get; }` | Seconds required (0 = instant) |
| Property | `InteractionType Type { get; }` | Instant, Hold, Toggle, or Repeatable |
| Method | `bool CanInteract(IPlayerEntity player)` | Final validation before execution |
| Method | `void OnInteractionStart(IPlayerEntity player)` | Called when interaction begins |
| Method | `void OnInteractionProgress(float progress)` | Called each frame during hold (0-1) |
| Method | `void OnInteractionComplete(IPlayerEntity player)` | Called on successful completion |
| Method | `void OnInteractionCancel(IPlayerEntity player)` | Called if cancelled before completion |

---

## 6. All ScriptableObject Configs (6)

The IDD defines 6 Config SOs. The Architecture doc references 9 (InputConfigSO, MovementConfigSO, CombatConfigSO, AnimationConfigSO, InteractionConfigSO, and the 6 below) — the additional 3 may be defined in subsystem-specific documentation.

### MovementConfigSO

**File:** `Subsystems/Movement/MovementConfigSO.cs`
**Used by:** MovementSubsystem, StaminaModule, FallDamageModule

| Type | Field | Default | Description |
|------|-------|---------|-------------|
| `float` | WalkSpeed | 3.0 m/s | Character walk speed |
| `float` | SprintSpeed | 6.0 m/s | Character sprint speed |
| `float` | CrouchSpeed | 1.5 m/s | Crouch walk speed |
| `float` | ProneSpeed | 0.8 m/s | Prone crawl speed |
| `float` | SwimSpeed | 2.0 m/s | Horizontal surface swim speed |
| `float` | UnderwaterSpeed | 1.5 m/s | Submerged movement speed |
| `float` | ClimbSpeed | 2.0 m/s | Vertical climb speed |
| `float` | JumpForce | 5.5 m/s | Initial vertical velocity on jump |
| `float` | Gravity | 20.0 m/s^2 | Downward gravity while airborne |
| `float` | FallGravityMultiplier | 1.8 | Gravity multiplier after apex |
| `float` | AirControlMultiplier | 0.3 | Horizontal air control (0=none, 1=full) |
| `float` | MaxSlopeAngle | 45 degrees | Maximum walkable slope |
| `float` | StepHeight | 0.3 m | Maximum step height |

**Note:** Line truncated in IDD — additional fields expected for stamina, fall damage, and slide parameters.

### WeaponDefinitionSO

**File:** `Subsystems/Weapon/WeaponDefinitionSO.cs`
**Used by:** WeaponSubsystem (composition model — one asset per weapon type)

| Type | Field | Default | Description |
|------|-------|---------|-------------|
| `string` | WeaponId | -- | Unique identifier (save/load, network) |
| `string` | DisplayName | -- | Player-facing name |
| `FireMode` | FireMode | Auto | Single, Burst, or Auto |
| `float` | FireRate | 600 RPM | Rounds per minute |
| `int` | BurstCount | 3 | Shots per burst (Burst mode only) |
| `float` | Damage | 25.0 | Base damage per shot |
| `DamageType` | DamageType | Bullet | Damage type for resistance |
| `int` | MagazineCapacity | 30 | Max rounds in magazine |
| `int` | ReserveCapacity | 90 | Max reserve ammo |
| `string` | Calibre | 5.56 | Calibre identifier (AmmoSystem pool) |
| `float` | HitScanRange | 200 m | Max hitscan range (ignored if projectile) |
| `ProjectileDefinitionSO` | ProjectileDefinition | null | If set, uses physical projectile |

**Note:** Line truncated in IDD — additional fields expected for spread, recoil profile, reload time, ADS multiplier, VFX/Audio references.

### RecoilProfileSO

**File:** `Subsystems/Weapon/RecoilProfileSO.cs`
**Used by:** WeaponSubsystem (RecoilSystem)

| Type | Field | Default | Description |
|------|-------|---------|-------------|
| `Vector2[]` | RecoilPattern | -- | Per-shot (horizontal, vertical) offsets in degrees (cycles) |
| `float` | RandomnessAmount | 0.1 | Random offset per pattern point (0=predictable) |
| `float` | CameraRecoilMultiplier | 1.0 | Scales camera recoil offset |
| `float` | AnimationRecoilMultiplier | 0.6 | Scales weapon animation recoil |
| `float` | RecoverySpeed | 10.0/s | Recoil recovery rate |
| `float` | RecoveryDelay | 0.1 s | Time after last shot before recovery |
| `float` | MaxAccumulatedRecoil | 15.0 degrees | Max total offset before pattern reset |

### CameraConfigSO

**File:** `Subsystems/Camera/CameraConfigSO.cs`
**Used by:** CameraSubsystem

| Type | Field | Default | Description |
|------|-------|---------|-------------|
| `float` | DefaultFOV | 90 degrees | Default field of view |
| `float` | SprintFOVBoost | 5 degrees | FOV increase during sprinting |
| `float` | FOVTransitionSpeed | 8.0 | FOV interpolation speed |
| `float` | SensitivityX | 1.0 | Horizontal look sensitivity |
| `float` | SensitivityY | 1.0 | Vertical look sensitivity |
| `float` | PitchMin | -80 degrees | Maximum upward look angle |
| `float` | PitchMax | 80 degrees | Maximum downward look angle |
| `float` | HeadBobFrequency | 1.8 Hz | Head bob oscillation frequency |
| `float` | HeadBobAmplitude | 0.05 m | Head bob vertical displacement |
| `float` | ShakeDecayRate | 1.2 | Trauma decay rate per second |
| `float` | ShakeMaxAngle | 5 degrees | Max shake rotation at trauma=1 |
| `float` | TpsDistance | 3.0 m | TPS camera distance |
| `float` | TpsHeight | 1.2 m | TPS camera height |

**Note:** Line truncated in IDD — additional fields expected for TPS shoulder offset, lean parameters.

### ParkourConfigSO

**File:** `Subsystems/Parkour/ParkourConfigSO.cs`
**Used by:** ParkourSubsystem, ParkourDetector

| Type | Field | Default | Description |
|------|-------|---------|-------------|
| `float` | MaxVaultHeight | 1.2 m | Maximum obstacle height for vaulting |
| `float` | MinVaultHeight | 0.4 m | Minimum vault height (below = step over) |
| `float` | VaultReach | 0.9 m | Forward reach for vault geometry |
| `float` | VaultClearanceRequired | 1.8 m | Min clearance above vault target |
| `float` | MaxMantleHeight | 2.2 m | Maximum ledge height for mantling |
| `float` | MantleReach | 0.7 m | Horizontal reach for mantle detection |
| `float` | VaultDuration | 0.6 s | Vault animation sequence time |
| `float` | MantleDuration | 1.0 s | Mantle animation sequence time |
| `float` | ClimbSpeed | 2.0 m/s | Vertical climb movement speed |
| `float` | ClimbStaminaDrainRate | 10.0/s | Stamina drain per second while climbing |
| `float` | MaxLedgeHangDuration | 3.0 s | Max hang time before dropping |

### PlayerAudioConfigSO

**File:** `Subsystems/Audio/PlayerAudioConfigSO.cs`
**Used by:** PlayerAudioSubsystem

| Type | Field | Default | Description |
|------|-------|---------|-------------|
| `SurfaceAudioLibrarySO` | SurfaceAudioLibrary | -- | Maps SurfaceType to footstep/impact clips |
| `AudioClip[]` | BreathingIdleClips | -- | Calm breathing clips |
| `AudioClip[]` | BreathingExertedClips | -- | Moderate exertion clips |
| `AudioClip[]` | BreathingExhaustedClips | -- | Exhaustion clips |
| `AudioClip[]` | PainClips | -- | Damage received clips |
| `AudioClip[]` | DeathClips | -- | Death clips |
| `AudioClip[]` | EffortClips | -- | Jump, dodge, heavy melee clips |
| `float` | FootstepVolumeWalk | 0.6 | Walking footstep volume |
| `float` | FootstepVolumeSprint | 1.0 | Sprinting footstep volume |
| `float` | FootstepVolumeCrouch | 0.2 | Crouching footstep volume |

---

## 7. Field Count Verification

| Category | Count |
|----------|-------|
| Total Enums | 21 |
| Total Enum Values (sum across all) | 108 |
| Total Data Structs | 4 |
| Total Data Struct Fields | 40 (19 + 11 + 9 + 6) |
| Total Event Structs | 38 |
| Total Event Struct Fields | 105 |
| Total Interfaces | 16 |
| Total Interface Methods (incl. properties) | 92 |
| Total Config SOs | 6 |
| Total SO Fields (visible) | 62+ |

**Note:** The IDD stats bar reports 16 interfaces, 92 methods, 38 events, 4 data structs, 21 enums, and 6 config SOs. The SO field count is approximate due to line truncation in the HTML source.

---

## 8. Cross-Reference Table

### Interface -> Event Publishing

| Interface | Publishes |
|-----------|-----------|
| IInputSystem | FireInputEvent, AimInputEvent, MoveInputEvent, LookInputEvent, JumpInputEvent, InteractInputEvent, WeaponSwitchInputEvent, InputContextChangedEvent |
| IMovementSystem | MovementStateChangedEvent, PlayerLandedEvent, PlayerJumpedEvent, SurfaceChangedEvent, StaminaChangedEvent |
| IHealthSystem | DamageReceivedEvent, HealthChangedEvent, PlayerDiedEvent, PlayerRespawnedEvent |
| ICombatSystem | MeleeHitEvent, ComboAdvancedEvent |
| IWeaponSystem | WeaponFiredEvent, HitDetectedEvent, WeaponReloadedEvent, WeaponSwitchedEvent, WeaponEmptyEvent, AdsStateChangedEvent, RecoilEvent |
| IParkourSystem | ParkourActionStartedEvent, ParkourActionCompletedEvent, ParkourActionFailedEvent |
| ICameraSystem | CameraModeChangedEvent, FOVChangedEvent |
| IAnimationSystem | FootstepAnimationEvent, ReloadCompleteAnimationEvent, MeleeHitWindowOpenedEvent, MeleeHitWindowClosedEvent |
| IPlayerAudioSystem | None (pure consumer) |
| IInteractionSystem | InteractionCandidateChangedEvent, InteractionStartedEvent, InteractionCompletedEvent |
| IUIBridgeSystem | None (writes to SO data channels) |

### Interface -> Event Consumption

| Interface | Consumes |
|-----------|----------|
| IInputSystem | (reads IInputProvider only) |
| IMovementSystem | PlayerInputFrame (via IInputProvider), PlayerDiedEvent, ParkourActionStartedEvent, ParkourActionCompletedEvent, AdsStateChangedEvent |
| IHealthSystem | (receives DamageData via ApplyDamage method call) |
| ICombatSystem | MeleeHitWindowOpenedEvent, MeleeHitWindowClosedEvent |
| IWeaponSystem | FireInputEvent, AimInputEvent, WeaponSwitchInputEvent, InteractInputEvent, PlayerInputFrame, ReloadCompleteAnimationEvent |
| IParkourSystem | (reads movement state, PlayerInputFrame) |
| ICameraSystem | LookInputEvent, AimInputEvent, MovementStateChangedEvent, AdsStateChangedEvent, RecoilEvent, WeaponSwitchedEvent |
| IAnimationSystem | MovementStateChangedEvent, WeaponFiredEvent, WeaponReloadedEvent, AdsStateChangedEvent, HealthChangedEvent, PlayerDiedEvent, RecoilEvent, InputContextChangedEvent, CameraModeChangedEvent, InteractionStartedEvent, InteractionCompletedEvent, StaminaChangedEvent |
| IPlayerAudioSystem | MovementStateChangedEvent, SurfaceChangedEvent, StaminaChangedEvent, HealthChangedEvent, PlayerDiedEvent, DamageReceivedEvent, PlayerJumpedEvent, PlayerLandedEvent, ParkourActionStartedEvent, WeaponFiredEvent, WeaponEmptyEvent, FootstepAnimationEvent |
| IInteractionSystem | InteractInputEvent |
| IUIBridgeSystem | MovementStateChangedEvent, StaminaChangedEvent, HealthChangedEvent, PlayerDiedEvent, PlayerRespawnedEvent, WeaponFiredEvent, WeaponReloadedEvent, WeaponSwitchedEvent, WeaponEmptyEvent, DamageReceivedEvent, InteractionCandidateChangedEvent, InteractionStartedEvent, InteractionCompletedEvent, CameraModeChangedEvent, FOVChangedEvent, ComboAdvancedEvent, InputContextChangedEvent, ParkourActionFailedEvent |

### Event -> Data Struct Usage

| Event | Carries Data Structs |
|-------|---------------------|
| HitDetectedEvent | `HitData`, `DamageData` |
| DamageReceivedEvent | `DamageData` |
| MeleeHitEvent | `DamageData` |

### Config SO -> Subsystem Usage

| Config SO | Used By Subsystems |
|-----------|-------------------|
| MovementConfigSO | MovementSubsystem, StaminaModule, FallDamageModule |
| WeaponDefinitionSO | WeaponSubsystem, WeaponManager, WeaponInstance |
| RecoilProfileSO | WeaponSubsystem (RecoilSystem), CameraSubsystem (RecoilCameraModule) |
| CameraConfigSO | CameraSubsystem |
| ParkourConfigSO | ParkourSubsystem, ParkourDetector |
| PlayerAudioConfigSO | PlayerAudioSubsystem (FootstepModule, BreathingModule, CombatVoiceModule) |

---

## Appendix: Key Implementation Notes

1. **All event struct handlers are synchronous** — handlers run in subscription order during Publish().
2. **Subscribe in Initialize(), Unsubscribe in Shutdown()** — never in Tick().
3. **No LINQ in any Tick/FixedTick/LateTick method** — zero GC allocation requirement.
4. **Pre-hash all Animator parameter names** at init via Animator.StringToHash().
5. **DamageData is never mutated in-flight** — pipeline modifiers read it, produce final amount separately.
6. **PlayerInputFrame is immutable** — one instance per tick, cached by InputSubsystem.
7. **Architecture doc naming overrides IDD naming** for 3 interfaces (see Architecture Summary Section 1).
