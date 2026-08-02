---
name: godot-architecture
description: Define, review, and evolve the architecture of Godot 4 game projects. Use when planning or implementing Godot features, separating input, game/domain logic, physics, presentation, audio, and persistence; designing testable scenes and scripts; creating scenario tests; reviewing coupling to Node, SceneTree, Input, Autoloads, time, or randomness; or documenting architecture decisions. Applies primarily to Godot 4 with GDScript.
---

# Godot Architecture

Establish a pragmatic, testable architecture for a Godot 4 project without fighting Godot's scene-oriented design.

## Goals

- Keep game rules independently testable where doing so has clear value.
- Keep Godot-specific APIs at explicit boundaries.
- Make gameplay scenarios reproducible as state plus commands.
- Prefer simple structures for small projects and add layers only when they solve a current problem.
- Preserve editor usability, scene composition, signals, resources, and other Godot strengths.
- Avoid architectural purity that increases indirection without improving changeability or testing.

## Start by inspecting the project

Before proposing or changing architecture:

1. Read `project.godot`, the main scene, relevant `.tscn` files, scripts, tests, and repository instructions.
2. Identify:
   - Godot version and language.
   - Existing test framework and test commands.
   - Main gameplay loop and scene ownership.
   - Autoloads and globally mutable state.
   - Input actions.
   - Physics bodies, areas, navigation, animation, audio, save data, and networking boundaries.
   - Existing conventions that should be preserved.
3. Determine the smallest architectural change that satisfies the requested feature.
4. Do not reorganize unrelated files unless the task explicitly requests a broader refactor.

When information is incomplete, state assumptions in the architecture definition rather than silently inventing requirements.

## Default architecture

Use these conceptual responsibilities. They do not all need separate files or nodes.

### Input adapter

Translate device-specific input into gameplay intent.

May use:

- `Input`
- `_input`
- `_unhandled_input`
- `InputEvent`
- input maps
- controller, keyboard, mouse, or touch details

Produces:

- commands
- actions
- direction values
- requests expressed in game terminology

Must not own game rules or directly mutate unrelated scene objects.

Examples:

- `MoveIntent`
- `AttackCommand`
- `InteractCommand`
- `PauseRequested`

### Application or controller layer

Coordinate a use case and define update order.

Responsibilities may include:

- receiving gameplay commands
- invoking domain logic
- calling a physics adapter
- applying results to state
- emitting presentation events
- switching scenes through an explicit navigation boundary

Keep controllers thin. A controller may coordinate rules but should not become the primary home of those rules.

### Domain or game-logic layer

Own deterministic game rules and meaningful state transitions.

Prefer plain GDScript objects such as `RefCounted`, `Resource`, or small value-like objects when they do not need the scene tree.

Typical responsibilities:

- damage and healing
- cooldowns
- inventory rules
- scoring
- status effects
- abilities
- win and loss conditions
- AI decisions
- state machines
- command validation

The domain layer should not depend directly on:

- `Input`
- node paths
- animations
- labels or sprites
- audio playback
- scene changes
- wall-clock time
- global randomness
- unrelated Autoload state

Inject or pass time, randomness, configuration, and external services when reproducibility matters.

### Physics and world adapter

Own interaction with Godot's runtime world.

Typical APIs:

- `CharacterBody2D` or `CharacterBody3D`
- `move_and_slide`
- `Area2D` or `Area3D`
- ray casts and shape casts
- navigation servers and agents
- physics queries

Let domain logic calculate intent, allowed actions, desired velocity, or consequences. Let this adapter execute Godot physics and report observed results.

Do not duplicate Godot's physics engine in a pure model merely to achieve complete separation.

### Presentation layer

Render state and play feedback.

Typical responsibilities:

- sprites and meshes
- animation
- labels and HUD
- particles
- camera effects
- visual interpolation
- presentation-only timing

Presentation may observe state or consume events. It must not decide authoritative gameplay outcomes.

For example, the domain decides that an actor died; the presentation decides which death animation and particles to play.

### Audio layer

Translate domain or presentation events into sounds and music. Audio completion should not normally determine authoritative game state.

### Persistence and external adapters

Isolate:

- save files
- configuration storage
- platform APIs
- achievements
- analytics
- network transport
- backend services

Convert external data into explicit domain data and validate it at the boundary.

## Godot-specific dependency rule

Dependencies should generally point inward:

```text
Input / UI / Scene / Physics / Persistence
                  ↓
       Controller / Use Cases
                  ↓
          Domain Logic
```

The domain may define interfaces or callable expectations needed from outer layers, but must not import concrete scene or platform implementations.

Signals do not automatically create good separation. Evaluate who owns the event, who may emit it, and whether the signal hides an implicit dependency.

## Scene composition

Treat scenes as composition roots and reusable visual or world units.

Recommended pattern:

```text
PlayerRoot
├── InputAdapter
├── Controller
├── CharacterBody or PhysicsAdapter
├── Visuals
├── AnimationPlayer
└── Audio
```

This is illustrative, not mandatory. A small feature may combine controller, physics, and presentation in one node while extracting only valuable deterministic rules.

Prefer:

- exported references or explicit setup methods over long fragile node paths
- typed signals and typed GDScript where practical
- scene-owned dependencies over service-locator access
- explicit lifecycle ownership
- documented Autoload responsibilities

Avoid:

- pervasive `get_node("../../..")`
- nodes reaching across sibling scenes to mutate internals
- using Autoloads as unrestricted global variable bags
- gameplay rules embedded in animation callbacks without an explicit contract
- UI controls directly changing authoritative domain fields

## State and commands

For gameplay scenarios, model behavior as:

```text
Given initial state
When commands or events occur
Then observable game state and outputs satisfy expectations
```

Use commands when they clarify intent or enable replay, networking, undo, deterministic simulation, or testing. Do not create a class for every button press when an enum, small data object, or method call is sufficient.

A command should express player or system intent, not device details.

Good:

- move in a direction
- use selected item
- confirm dialogue choice
- attack target

Weak:

- W key was pressed
- gamepad button 2 changed

## Time and randomness

Avoid hidden nondeterminism in game rules.

Preferred forms:

```gdscript
func tick(delta: float) -> void:
    cooldown = maxf(cooldown - delta, 0.0)
```

```gdscript
func choose_drop(rng: RandomNumberGenerator) -> DropResult:
    ...
```

For systems requiring clocks, define a small clock boundary or pass the relevant timestamp. Seed random generators in scenario tests.

Use real frame timing only in tests whose purpose is to verify integration with Godot's loop, animation, timers, or physics.

## Testing strategy

Use a layered test portfolio.

### Logic tests

Test deterministic rules without loading scenes.

Use for:

- calculations
- state transitions
- command validation
- inventory and combat rules
- cooldowns
- AI decisions
- save-data transformations

These should form the majority of tests.

### Scenario tests

Express important gameplay behavior as initial state, actions, and expected outcomes.

Example:

```text
Given a locked door and a player holding its key
When the player uses the key on the door
Then the key is consumed and the door becomes open
```

Prefer executing domain commands directly unless the scenario specifically concerns input mapping.

A scenario may test several collaborating domain objects. Do not force every scenario to be a narrow unit test.

### Scene integration tests

Instantiate the real `.tscn` when verifying:

- node wiring
- signal connections
- physics integration
- scene lifecycle
- UI synchronization
- animation triggers
- resource setup
- scene transitions

Expose semantic test/query methods where useful, such as:

- `get_score()`
- `is_battle_finished()`
- `get_living_enemy_count()`

Do not assert deep node paths or presentation text when a stable gameplay-facing observation is available.

### Input tests

Separate these concerns:

1. A physical `InputEvent` maps to the correct gameplay command.
2. The gameplay command produces the correct outcome.

Only a small number of end-to-end tests should exercise both through the full stack.

### End-to-end smoke tests

Cover a few critical journeys, such as:

- booting the main scene
- starting a game
- completing a representative encounter
- saving and loading
- reaching a result screen

Keep them few because they are slower and more fragile.

## Async and frame-dependent behavior

Prefer waiting for meaningful completion signals or conditions over arbitrary sleeps.

Use frame advancement only when the behavior requires Godot's process or physics loop.

Avoid assertions tied to exact animation frames unless frame accuracy is the actual requirement.

Always clean up instantiated nodes and restore global state after tests.

## Framework handling

Use the test framework already present in the repository.

When none exists:

- propose GUT or GdUnit4 based on project needs
- do not install an add-on without user approval when installation changes project dependencies
- provide the exact proposed command and directory convention
- ensure headless execution is possible for CI

Do not assume framework APIs from memory when editing a real project. Inspect the installed add-on version, documentation included in the repository, or existing tests.

## Architecture definition workflow

When asked to define architecture:

1. Summarize relevant project constraints.
2. Identify architectural drivers:
   - testability
   - game genre and simulation model
   - physics dependence
   - networking or replay requirements
   - team size
   - content workflow
   - expected project lifetime
3. Produce a responsibility map.
4. Define allowed dependency directions.
5. Define state ownership and lifecycle.
6. Define command and event flow.
7. Define boundaries for time, randomness, persistence, networking, and scene changes.
8. Define the test portfolio and representative scenarios.
9. Propose a directory and scene structure that fits the existing project.
10. Record trade-offs and rejected alternatives.
11. Provide a staged migration plan when changing existing code.
12. Validate the design against at least one concrete gameplay flow.

Use `references/architecture-definition-template.md` for the output shape.

## Implementation workflow

When implementing or refactoring:

1. Preserve current behavior with focused characterization tests when feasible.
2. Extract deterministic rules before moving scene ownership.
3. Introduce explicit inputs and outputs at the boundary.
4. Change one dependency direction at a time.
5. Add tests at the lowest sufficient level.
6. Instantiate real scenes only for integration concerns.
7. Run relevant tests and the project's standard validation commands.
8. Report what was verified and what remains unverified.

Avoid large folder-only refactors that move files without reducing coupling.

## Review checklist

Flag architecture concerns when:

- game rules call `Input` directly
- domain code uses node paths
- UI owns authoritative game state
- animations determine rules accidentally
- global Autoload state is modified from many unrelated places
- random or time-based behavior cannot be reproduced
- scene tests rely on deep hierarchy details
- controllers contain most game rules
- signals form undocumented many-to-many control flow
- a pure model attempts to reimplement Godot physics
- every layer has interfaces and factories despite only one implementation
- a proposed abstraction has no current testability or changeability benefit

Do not classify all Node-based logic as wrong. Explain the concrete cost and recommend the smallest useful correction.

## Required response qualities

When giving architecture guidance or making changes:

- distinguish requirements, assumptions, and recommendations
- show ownership and data flow, not only folder names
- include at least one concrete scenario walkthrough
- describe test boundaries
- identify Godot-specific compromises
- avoid claiming tests passed unless they were executed
- keep names aligned with the project's language and conventions

## Completion criteria

An architecture task is complete when the result states:

- who owns authoritative state
- how input becomes gameplay intent
- where rules execute
- how physics observations feed back into state
- how presentation observes outcomes
- how scenes and dependencies are composed
- how time, randomness, saves, and globals are controlled
- which tests run without scenes and which require real scenes
- how at least one important gameplay scenario is tested
- what trade-offs remain
