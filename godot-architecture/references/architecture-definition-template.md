# Godot Architecture Definition

## 1. Context

- Project:
- Godot version:
- Language:
- Genre / gameplay model:
- Existing structure:
- Existing test framework:
- Constraints:

## 2. Architectural drivers

Rank the factors that materially affect the design.

1.
2.
3.

## 3. Responsibility map

| Responsibility | Owner | Godot dependency allowed? | Notes |
|---|---|---:|---|
| Physical input mapping | | Yes | |
| Use-case coordination | | Limited | |
| Authoritative gameplay state | | Prefer no | |
| Game rules | | Prefer no | |
| Physics execution | | Yes | |
| Presentation | | Yes | |
| Audio | | Yes | |
| Persistence | | At adapter boundary | |
| Scene navigation | | At adapter boundary | |

## 4. Dependency rule

```text
[outer adapters]
       ↓
[controllers / use cases]
       ↓
[domain state and rules]
```

Allowed exceptions:

- 
- 

Forbidden dependencies:

- 
- 

## 5. State ownership and lifecycle

| State | Authoritative owner | Created by | Destroyed by | Observed by |
|---|---|---|---|---|
| | | | | |

Describe how state survives or resets across scene changes.

## 6. Input, command, and event flow

Example flow:

```text
InputEvent
→ InputAdapter
→ GameplayCommand
→ Controller
→ Domain transition
→ Physics request or domain event
→ View / Audio
```

Commands:

- 

Domain events:

- 

Presentation-only events:

- 

## 7. Scene composition

```text
Main
└── ...
```

Explain which scene acts as the composition root and how dependencies are supplied.

## 8. Boundaries

### Time

### Randomness

### Physics

### Persistence

### Networking

### Scene transitions

### Autoloads

## 9. Testing strategy

### Logic tests

- 

### Scenario tests

- 

### Scene integration tests

- 

### Input mapping tests

- 

### End-to-end smoke tests

- 

### Headless / CI command

```bash
# project-specific command
```

## 10. Representative scenario

### Scenario name

**Given**

- 

**When**

- 

**Then**

- 

### Execution path

```text
...
```

### Test level and rationale

- Logic:
- Scene integration:
- End to end:

## 11. Suggested project structure

```text
res://
├── ...
```

The structure must express ownership. Do not create folders solely to imitate a generic architecture.

## 12. Migration plan

1.
2.
3.

Each step should leave the project runnable and should identify its verification method.

## 13. Trade-offs and rejected alternatives

| Decision | Benefit | Cost | Rejected alternative |
|---|---|---|---|
| | | | |

## 14. Open questions

- 
