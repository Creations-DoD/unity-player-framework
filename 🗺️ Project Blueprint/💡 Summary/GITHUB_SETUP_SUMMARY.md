# Player Framework — GitHub Setup Summary

> **Source:** PlayerFramework_GitHub_Guide.docx (10 sections, June 2026)
> **Purpose:** Quick-reference for repository setup, branching, CI/CD, and team workflow.
> **Rule:** Follow this guide in order during initial repo bootstrap. Steps 1–4 are critical-path.

---

## 1. Repository Strategy

Two separate private repos. No monorepo.

| Repo | Name | Contents | Write Access |
|------|------|----------|-------------|
| **Framework** | `unity-player-framework` | Standalone Unity package (`_Framework/` folder). Own versioning, CHANGELOG, test suite. Zero game code. | Framework lead + senior engineers only |
| **Game (per title)** | `game-project-alpha` | Game-specific code (`_Game/` folder). References framework via UPM Git package. | Entire game team |

**Why two repos:** Enforces architecture at the repository level. Game engineers cannot edit framework code — they lack write access. Separation is structural, not just social.

### UPM Package Reference

Game repo references framework in `Packages/manifest.json`:

```json
{
  "dependencies": {
    "com.dod-studio.player-framework": "https://github.com/your-org/unity-player-framework.git#v1.2.0"
  }
}
```

Pinning to a tag (`#v1.2.0`) means game teams upgrade deliberately — never accidentally pulling breaking changes from main.

---

## 2. Day-One Setup Checklist

Complete in order. Steps 1–4 are non-negotiable — retrofitting them later is painful.

### Phase 0 — Before `git init`

| # | Task | Deliverable |
|---|------|-------------|
| 01 | **Force Text Serialization** | Unity → Edit → Project Settings → Editor → Asset Serialization → Mode: **Force Text**. Must be set before any asset exists. |
| 02 | **Create .gitignore** | Add Unity `.gitignore` BEFORE running `git init`. Committing `Library/` even once pollutes history permanently. |
| 03 | **Create two repos** | `unity-player-framework` (private) and `game-project-name` (private) on GitHub. |
| 04 | **Install Git LFS** | `git lfs install`. Verify: `git lfs version`. Required on every machine before first clone. |

### Phase 1 — Repository Bootstrap

| # | Task | Deliverable |
|---|------|-------------|
| 05 | **Add .gitattributes** | Run all `git lfs track` commands. Commit `.gitattributes` as first commit in both repos. |
| 06 | **Add package.json** | Framework repo root. Version `1.0.0`, package name `com.dod-studio.player-framework`. |
| 07 | **Create CHANGELOG.md** | With `## [Unreleased]` section. Single source of truth for version changes. |
| 08 | **Create develop branch** | `git checkout -b develop && git push -u origin develop`. |
| 09 | **Protect main and develop** | GitHub → Settings → Branches → Add rule. Require PR, CI pass, 1 reviewer for both. |

### Phase 2 — Team Configuration

| # | Task | Deliverable |
|---|------|-------------|
| 10 | **Add PR template** | `.github/pull_request_template.md` from Section 7 below. |
| 11 | **Add CODEOWNERS** | `.github/CODEOWNERS` from Section 9 below. Real GitHub usernames. |
| 12 | **Add CI workflows** | `.github/workflows/pr-validation.yml` and `release.yml` from Section 8. |
| 13 | **Add UNITY_LICENSE secret** | GitHub → Settings → Secrets → Actions → `UNITY_LICENSE`. Required for CI. |
| 14 | **Verify CI runs** | Open test PR to develop. Confirm editmode + playmode jobs appear and pass. |

### Phase 3 — Framework Scaffold Commit

| # | Task | Deliverable |
|---|------|-------------|
| 15 | **Create Core Contracts Layer** | All interfaces, event structs, enums, SO base classes. Zero logic. |
| 16 | **First meaningful commit** | `feat(infra): initial framework scaffold — core contracts layer` |
| 17 | **Tag v0.1.0** | `git tag v0.1.0 && git push origin v0.1.0` |
| 18 | **Wire game repo to framework** | Game repo `manifest.json` → add framework as UPM Git package pinned to `#v0.1.0` |
| 19 | **Verify the link** | Open game project in Unity. Confirm framework package in Window → Package Manager → In Project. |

### The Three Non-Negotiables

1. **Force Text serialization** — set before any asset is created
2. **Git LFS** — configured before any binary file is added
3. **Branch protection** — enabled before any other engineer touches the repo

Everything else can be added incrementally. These three cannot be safely retrofitted.

---

## 3. Git LFS Configuration

### Setup Commands

```bash
git lfs install

git lfs track '*.png'
git lfs track '*.jpg'
git lfs track '*.psd'
git lfs track '*.tga'
git lfs track '*.fbx'
git lfs track '*.wav'
git lfs track '*.mp3'
git lfs track '*.ogg'
git lfs track '*.anim'
git lfs track '*.controller'
git lfs track '*.prefab'
git lfs track '*.unity'
git lfs track '*.asset'

git add .gitattributes
git commit -m 'chore: configure Git LFS tracking'
```

### What Goes Where

| Extension | Storage | Reason |
|-----------|---------|--------|
| `*.cs` | Regular Git | Text — diffs meaningful and cheap |
| `*.asmdef` | Regular Git | Text/JSON — assembly definitions |
| `*.meta` (scripts) | Regular Git | Text — Unity asset metadata |
| `*.asset` (SOs) | Regular Git | YAML with Force Text — diffable |
| `*.json`, `*.xml`, `*.yaml` | Regular Git | Text — configuration files |
| `*.md` | Regular Git | Text — docs and CHANGELOG |
| `*.png`, `*.jpg`, `*.psd`, `*.tga` | **LFS** | Binary — textures/sprites |
| `*.fbx`, `*.obj` | **LFS** | Binary — 3D models |
| `*.wav`, `*.mp3`, `*.ogg` | **LFS** | Binary — audio assets |
| `*.anim`, `*.controller` | **LFS** | Binary — Unity animation files |
| `*.prefab`, `*.unity`, `*.asset` | **LFS** | Binary when using Mixed serialization (use Force Text instead) |

> **Tip:** With Force Text, `.prefab`/`.unity` files become YAML and can stay in regular Git — giving meaningful diffs. Force Text is why this is non-negotiable.

### .gitattributes Content

Generated by the `git lfs track` commands above. Example output:

```
*.png filter=lfs diff=lfs merge=lfs -text
*.jpg filter=lfs diff=lfs merge=lfs -text
*.psd filter=lfs diff=lfs merge=lfs -text
*.tga filter=lfs diff=lfs merge=lfs -text
*.fbx filter=lfs diff=lfs merge=lfs -text
*.wav filter=lfs diff=lfs merge=lfs -text
*.mp3 filter=lfs diff=lfs merge=lfs -text
*.ogg filter=lfs diff=lfs merge=lfs -text
*.anim filter=lfs diff=lfs merge=lfs -text
*.controller filter=lfs diff=lfs merge=lfs -text
*.prefab filter=lfs diff=lfs merge=lfs -text
*.unity filter=lfs diff=lfs merge=lfs -text
*.asset filter=lfs diff=lfs merge=lfs -text
```

---

## 4. Branching Strategy

Simplified GitFlow — no separate release branches; tags on main serve that purpose.

### Branch Structure

```
main          ← always releasable / shippable build
develop       ← integration branch, all features merge here first
│
├── feature/movement-swim-states
├── feature/weapon-projectile-system
├── feature/parkour-vault-detector
├── bugfix/slide-state-transition-jitter
└── hotfix/critical-crash-on-death     ← branches from main, not develop
```

### Branch Rules

| Branch | Rule | Rationale |
|--------|------|-----------|
| `main` | Protected. No direct pushes. Merge via PR from develop only. | Always a known-good build. |
| `develop` | Protected. Features merge via PR only. CI must pass. | Integration testing before main. |
| `feature/*` | Branch from develop. Format: `feature/[subsystem]-[what]` | Subsystem ownership visible at a glance. |
| `bugfix/*` | Branch from develop. Format: `bugfix/[subsystem]-[description]` | Bugs fixed in context. |
| `hotfix/*` | Branch from **main**. Format: `hotfix/[critical-description]` | Critical fixes bypass develop. Merge to both main AND develop. |

### Branch Naming Prefixes

| Prefix | Use For | Example |
|--------|---------|---------|
| `feature/` | New subsystem or feature | `feature/movement-swim-states` |
| `bugfix/` | Non-critical bug fix | `bugfix/weapon-reload-ammo-count` |
| `hotfix/` | Critical production fix | `hotfix/crash-on-player-death` |
| `refactor/` | Code cleanup, no behavior change | `refactor/camera-modifier-stack` |
| `perf/` | Performance improvement | `perf/input-event-bus-allocation` |
| `test/` | Test coverage addition | `test/movement-hsm-transitions` |
| `docs/` | Documentation only | `docs/iinputprovider-xml-comments` |
| `chore/` | Build, CI, dependency updates | `chore/update-unity-6-0-1` |

### Protection Rules (GitHub UI)

For both `main` and `develop`:

- Require a pull request before merging
- Require at least 1 approving review
- Require status checks to pass before merging (link CI workflow)
- Require branches to be up to date before merging
- Do not allow bypassing the above settings (includes admins)

### Merge Strategy

- **feature → develop:** Squash merge (clean history)
- **develop → main:** Regular merge (preserve integration context)
- **hotfix → main + develop:** Regular merge to both

---

## 5. Commit Conventions

### Conventional Commits Format

```
<type>(<scope>): <short description>
```

| Field | Values |
|-------|--------|
| **type** | `feat` \| `fix` \| `refactor` \| `perf` \| `test` \| `docs` \| `chore` \| `style` |
| **scope** | Subsystem touched (see table below) |

### Valid Scopes

| Scope | Subsystem / Area |
|-------|-----------------|
| `movement` | MovementSystem, MovementHSM, all movement states, StaminaModule, SurfaceDetection |
| `weapon` | WeaponSystem, FireHandler, ReloadHandler, AmmoSystem, RecoilSystem, AdsSystem |
| `camera` | CameraSystem, FPSCameraMode, TPSCameraMode, all camera modifiers |
| `combat` | CombatSystem, HealthSystem, DamageProcessor, MeleeSystem, ComboSystem |
| `animation` | AnimationSystem, AnimatorProxy, IKSystem, ProceduralAnimationSystem |
| `audio` | PlayerAudioSystem, FootstepModule, BreathingModule, VoiceModule |
| `input` | InputSystem, IInputProvider, InputContextManager, InputRouter |
| `parkour` | ParkourSystem, VaultHandler, MantleHandler, ClimbHandler |
| `interaction` | InteractionSystem, IInteractable, InteractionQueue |
| `ui` | UIBridgeSystem, data channels |
| `infra` | EventBus, ServiceLocator, LifecycleManager, SubsystemRegistry |
| `statemachine` | HierarchicalStateMachine, IState, StateTransitionTable |
| `test` | Any test file in Tests/ directory |
| `ci` | GitHub Actions workflows, build scripts |
| `deps` | Unity version, package updates, third-party dependencies |
| `docs` | Documentation, XML comments, CHANGELOG, README |

### Good Examples

```
feat(movement): add SwimSurface and SwimUnderwater leaf states to WATER super-state
fix(weapon): reload handler not resetting ammo count on empty tactical reload
perf(input): pre-allocate InputEvent delegate list to eliminate per-frame GC
refactor(camera): extract FOVModifier from CameraSystem into separate module
test(movement): add unit tests for HSM transition from Sprint to Slide state
docs(infra): add XML docs to all IInputProvider interface methods
chore(deps): bump Unity version to 6.0.1
feat(combat): add DamageProcessor pipeline with armor and resistance modifiers
fix(parkour): vault geometry detection failing on slopes over 15 degrees
```

### Bad Examples

```
fixed stuff          ✗ Too vague
updates              ✗ No scope, no type
WIP                  ✗ Not a commit message
feat: add swimming    ✗ Missing scope
feat: add swimming, parkour, and IK system    ✗ Too broad — should be multiple commits
movement: swim states added    ✗ Missing type
```

**Rule:** If a commit description requires the word "and", it should be two commits. Each commit = one atomic change to one subsystem.

---

## 6. Pull Request Workflow

### PR Title Format

```
[SUBSYSTEM] Short description of what changed
```

```
[Movement]  Add SwimSurface and SwimUnderwater states to WATER super-state
[Weapon]    Fix ReloadHandler not resetting ammo count on empty reload
[Infra]     Add [AuthoritativeState] attribute and snapshot interface to EventBus
[Camera]    Extract FOVModifier into standalone modifier following modifier-stack pattern
```

### PR Template

File: `.github/pull_request_template.md`

```markdown
## What changed
<!-- Brief description of what this PR adds, fixes, or changes -->

## Subsystem(s) affected
- [ ] Movement    - [ ] Weapon      - [ ] Camera    - [ ] Combat
- [ ] Input       - [ ] Audio       - [ ] Parkour   - [ ] Interaction
- [ ] Animation   - [ ] UI Bridge   - [ ] Infra     - [ ] Tests

## Interface changes
- [ ] No interface changes (no MAJOR bump needed)
- [ ] New interface method added (MINOR bump)
- [ ] Existing interface modified — list affected consumers below:

## Performance
- [ ] No allocations introduced in Tick() / FixedTick() / LateTick() paths
- [ ] Profiler checked (attach screenshot if perf-sensitive change)

## Checklist
- [ ] Unit tests added or updated
- [ ] No direct cross-subsystem references introduced
- [ ] ScriptableObject config updated if new tunable values added
- [ ] XML docs added to any new public interface methods
- [ ] CHANGELOG.md updated under [Unreleased]
```

### Review Requirements

| PR Type | Required Reviewers |
|---------|-------------------|
| Any framework PR | At least 1 engineer (not the author) |
| Changes to any `IXxx` interface | **Framework lead (required)** |
| Changes to EventBus or LifecycleManager | **Framework lead (required)** |
| Changes to Tests/ | Subsystem owner + framework lead |
| Performance-sensitive changes | Subsystem owner (requires profiler evidence) |

### The One Rule

> No PR that removes an existing interface method, renames an event struct, or changes the signature of an existing public API merges without a **written migration guide** in the PR description and a **MAJOR version bump** in `package.json`. No exceptions.

---

## 7. CI/CD Pipeline

### Workflow 1 — PR Validation

Runs on every PR targeting `develop` or `main`. Must pass before merge.

File: `.github/workflows/pr-validation.yml`

```yaml
name: PR Validation
on:
  pull_request:
    branches: [develop, main]

jobs:
  editmode-tests:
    name: Edit Mode Tests (fast)
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          lfs: true
      - uses: game-ci/unity-test-runner@v4
        with:
          unityVersion: 6.0.0f1
          testMode: editmode
          artifactsPath: test-results/editmode
        env:
          UNITY_LICENSE: ${{ secrets.UNITY_LICENSE }}

  playmode-tests:
    name: Play Mode Tests (integration)
    runs-on: ubuntu-latest
    needs: editmode-tests
    steps:
      - uses: actions/checkout@v4
        with:
          lfs: true
      - uses: game-ci/unity-test-runner@v4
        with:
          unityVersion: 6.0.0f1
          testMode: playmode
          artifactsPath: test-results/playmode
        env:
          UNITY_LICENSE: ${{ secrets.UNITY_LICENSE }}

  upload-results:
    name: Upload Test Results
    runs-on: ubuntu-latest
    needs: [editmode-tests, playmode-tests]
    if: always()
    steps:
      - uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: test-results/**
```

**Flow:** PR opens → editmode tests run → playmode tests run (only if editmode passes) → results uploaded as artifacts.

### Workflow 2 — Release Tagging

Runs automatically when a PR merges to `main`. Reads version from `package.json` and creates a Git tag + GitHub Release.

File: `.github/workflows/release.yml`

```yaml
name: Tag Release
on:
  push:
    branches: [main]

jobs:
  release:
    name: Create Version Tag
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Read version from package.json
        id: version
        run: |
          VERSION=$(node -p "require('./package.json').version")
          echo "VERSION=$VERSION" >> $GITHUB_OUTPUT

      - name: Check tag does not already exist
        run: |
          if git rev-parse "v$VERSION" >/dev/null 2>&1; then
            echo 'Tag already exists — did you forget to bump package.json?'
            exit 1
          fi

      - name: Create and push tag
        run: |
          git config user.name  'github-actions[bot]'
          git config user.email 'github-actions[bot]@users.noreply.github.com'
          git tag v$VERSION
          git push origin v$VERSION

      - name: Create GitHub Release
        uses: actions/create-release@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          tag_name: v$VERSION
          release_name: v$VERSION
          body_path: CHANGELOG.md
          draft: false
          prerelease: false
```

**Flow:** PR merges to main → reads `package.json` version → checks tag doesn't exist → creates + pushes tag → creates GitHub Release from CHANGELOG.

### Unity License Setup

1. Unity account → Manage Licences → Activate (Personal or Pro)
2. GitHub → Settings → Secrets → Actions → New secret → `UNITY_LICENSE`
3. Paste contents of `.ulf` license file
4. `game-ci/unity-test-runner` handles activation/deactivation automatically

---

## 8. UPM Package Setup

### package.json (Framework Repo Root)

```json
{
  "name": "com.dod-studio.player-framework",
  "version": "1.0.0",
  "displayName": "AAA Player Framework",
  "description": "Modular player entity composition platform",
  "unity": "6.0",
  "author": {
    "name": "Your Studio"
  }
}
```

### Package Structure

```
unity-player-framework/          ← repo root = package root
├── package.json
├── CHANGELOG.md
├── README.md
├── LICENSE.md
├── .gitignore
├── .gitattributes
├── .github/
│   ├── CODEOWNERS
│   ├── pull_request_template.md
│   └── workflows/
│       ├── pr-validation.yml
│       └── release.yml
├── Assets/
│   ├── Framework/
│   │   ├── Core/               ← Interfaces, Events, Base classes
│   │   ├── Subsystems/         ← All subsystem implementations
│   │   ├── Debug/              ← Debug tools
│   │   └── Shared/             ← Shared utilities
│   └── Tests/                  ← EditMode + PlayMode tests
```

### How Game Repo References It

```json
// game-repo/Packages/manifest.json
{
  "dependencies": {
    "com.dod-studio.player-framework": "https://github.com/your-org/unity-player-framework.git#v1.2.0"
  }
}
```

### Versioning (SemVer)

| Change Type | Bump | Example |
|-------------|------|---------|
| New subsystem added | MINOR | 1.1.0 — ParkourSystem added |
| New method on existing interface (with default impl) | MINOR | 1.1.0 |
| Existing interface method changed | **MAJOR** | 2.0.0 — `IMovementSubsystem` contract changed |
| Event struct field added | PATCH | 1.1.1 — new field with sensible default |
| Event struct field removed | **MAJOR** | 2.0.0 |
| Bug fix, no interface change | PATCH | 1.1.1 |
| Performance improvement, no API change | PATCH | 1.1.1 |

### Tagging a Release

```bash
# After merging to main and updating package.json version
git tag v1.2.0
git push origin v1.2.0
# GitHub shows the tag under Releases
# Game repos update manifest.json pin to #v1.2.0
```

### Updating Game Repo Framework Version

```
Edit manifest.json → change #v1.1.0 to #v1.2.0 → git commit → Unity re-imports
```

---

## 9. Unity Project Settings

### Force Text Serialization

**Must be set before first commit.** Edit → Project Settings → Editor → Asset Serialization → Mode: **Force Text**

Why it matters:

| With Force Text | Without Force Text |
|----------------|-------------------|
| `.prefab`, `.unity`, `.asset` stored as human-readable YAML | Binary blobs |
| Git can diff them — see exactly what changed | Diffs meaningless |
| Merge conflicts resolvable | Binary merge conflicts always destructive |
| Code review meaningful — see component values | Review blind |

### .gitignore (Both Repos)

```gitignore
# Unity generated — never commit these
[Ll]ibrary/
[Tt]emp/
[Oo]bj/
[Bb]uild/
[Bb]uilds/
[Ll]ogs/
[Mm]emoryCaptures/
[Uu]serSettings/

# Asset import artifacts
*.pidb.meta
*.pdb.meta
*.mdb.meta

# Visual Studio / Rider
.vs/
.idea/
*.csproj
*.unityproj
*.sln
*.suo
*.user
*.pidb

# OS generated
.DS_Store
.DS_Store?
Thumbs.db
ehthumbs.db

# Build outputs
*.apk
*.aab
*.unitypackage
*.app

# Crashlytics
crashlytics-build.properties
```

> The `Library/` folder alone can exceed **10 GB** on large projects. Add `.gitignore` before `git init`, not after.

### Committed Project Settings

These Unity project settings should be committed (they live under `ProjectSettings/`):

| File | Purpose |
|------|---------|
| `ProjectSettings/EditorBuildSettings.asset` | Scene list for builds |
| `ProjectSettings/InputManager.asset` | Input axes (legacy) |
| `ProjectSettings/TagManager.asset` | Tags and layers |
| `ProjectSettings/PhysicsManager.asset` | Physics settings |
| `ProjectSettings/QualitySettings.asset` | Quality levels |
| `ProjectSettings/TimeManager.asset` | Fixed timestep |

---

## 10. Common Operations

### Add a New Subsystem

```bash
# 1. Branch from develop
git checkout develop && git pull
git checkout -b feature/combat-combosystem

# 2. Create folder structure
mkdir -p Assets/Framework/Subsystems/Combat/Modules/Combo

# 3. Add interface to Core/Contracts/
#    Add event structs to Core/Events/Types/
#    Add config SO to Subsystems/Combat/

# 4. Commit with conventional format
git add Assets/Framework/
git commit -m 'feat(combat): add ComboSystem interface, events, and config SO'

# 5. Push and open PR
git push -u origin feature/combat-combosystem
# GitHub → New PR → base: develop
```

### Create a Framework Release

```bash
# 1. Ensure all changes are on main via develop
# 2. Update package.json version
#    Edit package.json → "version": "1.3.0"

# 3. Update CHANGELOG.md
#    Move items from [Unreleased] to [1.3.0] with date

# 4. Commit, merge to main, tag
git commit -m 'chore(deps): bump version to 1.3.0'
# Merge to main (via PR)
git checkout main && git pull
git tag v1.3.0
git push origin v1.3.0

# 5. CI auto-creates GitHub Release from CHANGELOG
```

### Resolve .unity / .prefab Merge Conflicts

With Force Text serialization, these are YAML — conflicts are text-based and resolvable.

```bash
# Option 1: Use Unity's merge tool
Unity → Edit → Preferences → External Tools → Merge Tool → configure

# Option 2: Manual YAML merge
# Open both versions in text editor
# .unity files have clear structure:
#   m_ObjectHideFlags: 0
#   m_PrefabParentObject: ...
#   Serialized properties follow
# Resolve conflicting property values, keeping the intended one

# Option 3: Accept theirs/hours (destructive — only for non-critical assets)
git checkout --theirs path/to/file.prefab
git add path/to/file.prefab
```

> **Tip:** The more granular your commits (one subsystem at a time), the fewer .unity conflicts you'll have.

### Update Framework from Game Repo

```bash
# 1. Check available framework versions
# GitHub → unity-player-framework → Releases

# 2. Update game repo's manifest.json
# Change: "com.dod-studio.player-framework": "...#v1.2.0"
# To:     "com.dod-studio.player-framework": "...#v1.3.0"

# 3. Commit and open Unity
git commit -m 'chore(deps): update player framework to v1.3.0'
# Unity auto-reimport on focus

# 4. Check CHANGELOG for breaking changes
# If MAJOR bump: follow migration guide in release notes
```

### Quick Reference Card

| Task | Command / Location |
|------|-------------------|
| Create feature branch | `git checkout develop && git pull && git checkout -b feature/movement-swim-states` |
| Stage framework files only | `git add Assets/Framework/` (never `git add -A` without reviewing) |
| Write a commit | `feat(movement): add SwimSurface and SwimUnderwater states` |
| Push and open PR | `git push -u origin feature/movement-swim-states` → GitHub → New PR → base: develop |
| Bump framework version | Edit `package.json` version → update `CHANGELOG.md` → commit → merge to main → auto-tagged |
| Update game repo framework | Edit `manifest.json` → change `#v1.1.0` to `#v1.2.0` → commit → Unity re-imports |
| Check LFS files | `git lfs ls-files` |
| Verify branch protection | GitHub → Settings → Branches |
| Force Text setting | Unity → Edit → Project Settings → Editor → Asset Serialization → Force Text |

---

> *"The repository structure should reflect the architecture. Two repos, clear ownership, protected branches, typed commits. The architecture document says the framework must be ignorant of the game. The version control strategy enforces that at the repository level."*
> — AAA Player Framework Engineering Guide, June 2026
