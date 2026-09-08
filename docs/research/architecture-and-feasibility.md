# Rhyzora Architecture and Feasibility Research

## Status

Research document. Rhyzora is still under design; terminology and boundaries in this document are hypotheses to validate through implementation and cross-backend experiments.

## 1. Research objective

Rhyzora is not primarily a UI toolkit and should not be designed as a TUI framework that later grows Web, desktop, CLI, and MCP adapters.

The research target is a higher-level system that:

1. describes application capabilities independently from presentation,
2. imports existing business logic through explicit engine contracts,
3. composes single operations into reusable task graphs and workflows,
4. describes user and agent interactions semantically rather than as widget trees,
5. optionally applies a motif that changes how application state and interaction feel,
6. projects the same semantics into multiple human, agent, development, and automation surfaces.

The principal output surfaces currently considered are:

- batch and interactive CLI,
- TUI,
- Web,
- desktop,
- MCP,
- CI / task-runner execution,
- editor tooling,
- playgrounds,
- generated test UIs, mocks, fixtures, and harnesses.

The key research question is therefore not "how can Nim render each UI?" but:

> What is the smallest transport- and presentation-independent semantic model that remains expressive enough to generate useful interfaces and workflows across these surfaces?

---

## 2. System hypothesis

The current architectural hypothesis is:

```text
                    Existing business logic
                             │
             ┌───────────────┼───────────────┐
             │               │               │
          OpenAPI      structured CLI     Native Nim
             │               │               │
             └───────────────┼───────────────┘
                             ▼
                       Engine adapters
                             │
                             ▼
                    Canonical Engine IR
                             │
                  Operation / Resource IR
                             │
                   ┌─────────┴─────────┐
                   │                   │
               Application          Flow
               semantics          Task graph
                   │                   │
                   └─────────┬─────────┘
                             ▼
                     Interaction / State
                             │
                           Motif
                             │
                             ▼
                     Backend lowering
                             │
       ┌────────┬────────┬───┼────┬─────────┬──────────┐
       ▼        ▼        ▼   ▼    ▼         ▼          ▼
      CLI      TUI      MCP Web Desktop   CI/Runner  Dev tools
```

No single external specification should become Rhyzora's canonical model. OpenAPI, machine-readable CLI descriptions, MCP schemas, and native Nim declarations are import/export formats or bindings around the canonical model.

---

## 3. Engine Contract

### 3.1 Boundary

An Engine Contract answers:

> What can this engine do, what typed data does each operation accept and return, and what observable effects and failures can occur?

It must not answer how the operation is displayed.

A minimal useful operation requires:

- stable identity,
- human-readable description,
- typed input,
- typed output,
- structured errors,
- invocation binding.

For high-quality interface generation, additional semantics are useful:

- read / write / destructive effects,
- idempotency,
- pagination,
- progress,
- cancellation,
- streaming,
- resource identity,
- relationships,
- long-running operation state.

### 3.2 Capability levels

A tentative capability scale is useful for both importers and backend diagnostics:

```text
E0  Invocation only
E1  Typed input
E2  Typed output
E3  Structured errors and effects
E4  Pagination / progress / cancellation / streaming
E5  Resource and relation semantics
```

E2 is the practical baseline for generic rich interface generation. An engine below E2 may still be usable, but generated interfaces should degrade explicitly rather than pretend that opaque text output is structured data.

### 3.3 Bindings

Operation semantics and transport are separate:

```text
Operation
  ├─ Native Nim binding
  ├─ HTTP / OpenAPI binding
  ├─ Process / CLI binding
  ├─ MCP binding
  └─ future bindings
```

This allows an Application DSL to refer to `searchUsers` without knowing whether it is implemented by a Nim proc, HTTP request, executable, or remote tool.

---

## 4. Canonical Schema IR

JSON Schema is an important interoperability source because OpenAPI and MCP both expose schema-shaped input/output descriptions. It should not, however, be the internal type system exposed directly to every backend.

Rhyzora should normalize external schemas into a smaller canonical type model, tentatively including:

```text
Null
Bool
Integer
Number
String
Array
Object
Enum
Union
Optional
Reference
```

Semantic types can refine primitive representation:

```text
Email
URL
Path
DateTime
Duration
Bytes
Percent
Identifier
Secret
```

The normalizer needs explicit outcomes:

```text
supported
supported-with-degradation
opaque
unsupported
```

This prevents every backend from becoming a complete JSON Schema, OpenAPI, or CLI-schema interpreter.

### Research risk

Schema normalization is one of the largest design risks. Polymorphism, recursive references, unions, read/write-only properties, content types, and external dialect features can easily leak protocol details into the canonical model.

The research goal is not universal schema equivalence. It is a stable canonical subset with explicit escape hatches and diagnostics.

---

## 5. External engine import

### 5.1 OpenAPI

OpenAPI is expected to be a high-feasibility importer for ordinary typed HTTP operations.

Typical lowering:

```text
operationId                -> Operation identity
path/query/header/body     -> Operation input
response schema            -> Operation output
HTTP method + path         -> HTTP binding
response/error definitions -> structured outcomes
```

Likely difficulty increases around:

- multiple content types,
- complex authentication,
- callbacks and webhooks,
- SSE and streaming,
- highly polymorphic schemas,
- long-running asynchronous job protocols.

The importer should therefore report which Engine capability level was recovered rather than claiming complete semantic recovery.

### 5.2 CLI

CLI import is less reliable than OpenAPI because invocation syntax does not imply output structure.

A command specification may describe:

```text
command
subcommand
arguments
options
environment
exit codes
```

but a rich TUI/Web/MCP projection also needs typed output.

Preferred CLI engines expose machine-readable output, for example JSON, together with a schema or equivalent contract. Human-oriented stdout scraping should be an adapter escape hatch, not the canonical integration mechanism.

Process execution must use structured executable + argv + environment + stdin/stdout/stderr handling. Rhyzora should not construct shell command strings as its normal execution model.

### 5.3 Native Nim

Native Nim is the strongest integration path because proc signatures, generated metadata, templates, and macros can provide operation information without a separate parser.

The key feasibility question is how much metadata can be extracted statically while keeping runtime invocation straightforward and avoiding excessive macro coupling.

---

## 6. Application Semantics

The Application DSL should describe user or agent intent above widget level.

Preferred semantic vocabulary includes concepts such as:

```text
browse
search
inspect
select
edit
confirm
execute
navigate
observe
```

instead of requiring applications to start from:

```text
Label
Input
Button
Table
Dialog
```

A backend chooses the representation appropriate to its environment.

For example, the same `confirm destructiveAction` may lower to:

```text
CLI       y/N prompt
TUI       modal
Web       confirmation dialog
Desktop   native or custom dialog
MCP       destructive operation metadata / policy boundary
```

The invariant is semantic equivalence, not visual equivalence.

### Progressive control

A single abstraction level is unlikely to be sufficient. Rhyzora should support controlled descent:

```text
Semantic application DSL
        ↓
Presentation hints / component DSL
        ↓
Backend primitives
```

High-level semantics should remain the default. Special interfaces can descend without forcing all applications to become backend-specific.

---

## 7. Flow and Task Runner model

Rhyzora should also compose operations into executable workflows. This is not an ancillary build feature; it is the temporal counterpart to the interface projection model.

### 7.1 Task

A Task is an execution unit. It may invoke:

- native Nim,
- an Engine operation,
- a structured CLI/process binding,
- an HTTP/OpenAPI operation,
- MCP,
- another Flow.

A Task should be able to declare or expose:

```text
input
output
dependencies
condition
effect
retry
timeout
cancellation
concurrency policy
cache policy
artifacts
progress
logs
```

### 7.2 Flow

A Flow composes tasks as a graph rather than merely a sequential script.

```text
            build
           /     \
        lint     test
           \     /
           package
              │
           publish
```

The graph model enables:

- parallel execution,
- dependency resolution,
- failure propagation,
- retry,
- cancellation,
- incremental execution,
- resume,
- cache reuse,
- artifact tracking,
- critical-path observation.

### 7.3 Artifact edges

Ordering alone is insufficient for build- and automation-oriented flows. Tasks should be able to produce and consume typed or at least identified artifacts:

```text
Task -> Artifact -> Task
```

This creates a basis for cache identity, invalidation, incremental execution, downloads, CI integration, and generated result views.

### 7.4 Flow as Operation

A completed Flow definition should be exposable as a higher-level Operation.

```text
primitive operations
        ↓
       tasks
        ↓
       flow
        ↓
higher-level operation
        ↓
   another flow
```

This permits business logic composition rather than restricting Rhyzora to presentation composition.

### 7.5 Flow projection

A single Flow can project differently:

```text
CLI       command + textual progress
TUI       task graph / progress / logs
Web       run form + job progress
Desktop   job control UI
MCP       typed high-level tool
CI        non-interactive execution
```

The UI should be derived from Flow state rather than requiring applications to hand-author progress bars and task lists.

---

## 8. Motif

Rhyzora should distinguish a Motif from a conventional visual theme.

A Theme usually controls visual tokens such as color, typography, spacing, or borders. A Motif can additionally map semantic application state and interaction into a coherent experiential language.

Tentative structure:

```text
Motif
├─ Theme
├─ Assets
├─ Vocabulary
├─ Feedback
├─ Interaction flavor
├─ Character
└─ Semantic mappings
```

For example, an application state such as:

```text
waiting
running
success
error
```

may be mapped by a character-oriented motif into different expressions or animations, while the underlying application knows only the semantic state.

A motif must not corrupt machine-readable surfaces. Batch CLI and MCP should normally suppress or neutralize decorative output unless the semantics themselves are useful to those consumers.

### Motif composition

Motifs should be composable rather than one monolithic skin. A conceptual model could combine dimensions such as:

```text
base mood
character
visual language
accent
feedback style
```

This is a research area: composition needs predictable precedence and conflict handling, otherwise motif packs will become non-composable bundles of overrides.

---

## 9. Backend capability model

Different outputs do not have identical interaction capabilities. Rhyzora therefore needs an explicit backend capability model.

Candidate capabilities include:

```text
Interactive
Selection
Forms
Navigation
Modal
Pointer
Streaming
Progress
Animation
Sound
StructuredOutput
AgentTool
```

Each lowering should resolve to one of:

```text
native
fallback
unsupported
```

This model is required to prevent false portability. A pointer gesture should not be silently treated as an MCP concept; a destructive confirmation should not be omitted simply because a backend has no dialog widget.

---

## 10. Backend feasibility

### CLI

Feasibility: high.

Typed operations map naturally to commands, subcommands, arguments, options, structured output, and prompts. CLI is also a useful early backend because it tests the semantic model without introducing a graphical rendering system.

### MCP

Feasibility: high.

Typed operation input/output and descriptions map naturally to tool definitions. MCP is a useful architectural test because it has almost no visual presentation. If the Application/Engine IR can project cleanly to MCP, the core is less likely to be accidentally UI-specific.

### TUI

Feasibility: high, implementation effort medium.

The desired output capability should be comparable to mature TUI frameworks: layout, tables, lists, trees, inputs, forms, focus, scroll, modal interactions, Unicode, resize, progress, and asynchronous state.

Rhyzora should not copy a specific TUI implementation model. Terminal I/O, width calculation, cell buffering, diff rendering, focus management, and platform quirks should be isolated beneath the semantic/backend boundary.

### Web

Feasibility: high for server-driven interfaces; medium for rich client applications.

Forms, browse/detail views, actions, validation, navigation, and workflow progress can be generated directly from semantic IR. Rich reactive applications introduce client-runtime and synchronization concerns but do not invalidate the architecture.

### Desktop

Feasibility: medium.

The semantic model is applicable, but toolkit lifecycle, native menus, clipboard, drag-and-drop, OS integration, packaging, and accessibility add significant backend-specific work. Desktop should not shape the core IR.

### Development and testing surfaces

Feasibility: high once the canonical IR is stable.

A playground, generated test UI, mock engine, fixture generator, and harness adapter are all projections of the same contracts. They are valuable because they test whether Rhyzora's semantics are sufficiently explicit to drive a surface other than a product UI.

---

## 11. Nim feasibility

Nim is a credible implementation language for this design for reasons beyond runtime performance.

### Strengths

- macros and templates can provide an embedded declarative DSL,
- compile-time AST access can validate references and generate metadata,
- ordinary Nim objects can represent IR independently of macro expansion,
- native C/C++ interoperability supports backend integration,
- native binaries fit CLI/TUI/server/task-runner use cases,
- asynchronous execution is available for operations and flows.

### Important constraint

The IR must not live only inside macros.

The intended pipeline is:

```text
Nim DSL
   ↓
frontend / macro parsing
   ↓
ordinary canonical IR
   ↓
validation
   ↓
backend lowering / runtime metadata
```

Directly generating TUI or Web code from DSL macros would couple syntax to backend implementation and undermine multi-surface reuse.

### Compile time vs runtime

A hybrid approach is appropriate.

Compile-time candidates:

- operation existence,
- schema compatibility,
- application references,
- static capability checks,
- DSL structural validation.

Runtime responsibilities:

- engine invocation,
- API results,
- process execution,
- authentication/session state,
- navigation/focus,
- flow scheduling,
- progress and streaming.

---

## 12. Principal feasibility risks

Current risk ranking:

| Area | Risk | Why it matters |
| --- | --- | --- |
| Canonical semantic IR | High | Every importer, workflow, and backend depends on this boundary |
| Schema normalization | High | External specifications contain richer semantics than every backend can preserve |
| CLI output contracts | High | CLI syntax alone does not provide typed result data |
| Capability/fallback rules | Medium-High | Prevents false claims of cross-backend equivalence |
| Flow execution semantics | Medium-High | Retry, cancellation, artifacts, caching, and resumption interact |
| Motif composition | Medium | Easy to become an unstructured override system |
| Async/stream abstraction | Medium | Needed across HTTP, processes, UI state, and workflows |
| TUI rendering | Medium | Significant engineering but isolated beneath the core model |
| OpenAPI import | Medium | Ordinary operations are straightforward; complex protocol features are not |
| Web backend | Medium | Simple interfaces are direct; rich clients add lifecycle/state concerns |
| Desktop backend | Medium-High | Large platform-specific surface, but low impact on core feasibility |
| CLI backend | Low | Natural projection of typed operations |
| MCP backend | Low | Natural projection of typed operations |

No current item makes the project infeasible. The architectural risk is concentrated in choosing semantics that are either too weak to generate useful surfaces or too rich and protocol-specific to remain stable.

---

## 13. Failure modes to avoid

### 13.1 TUI-first architecture

Do not define a widget tree as the canonical application representation and later translate it to CLI/MCP/Web. That makes non-visual backends second-class and forces presentation semantics into the core.

### 13.2 OpenAPI as the core IR

OpenAPI should be an importer/binding, not the canonical application model. HTTP semantics are not universal application semantics.

### 13.3 Universal CLI scraping

Do not claim arbitrary human-oriented CLI output can become structured rich interfaces automatically. Structured output and typed contracts are the preferred boundary.

### 13.4 Identical interfaces everywhere

The invariant is shared semantics, not identical presentation. Backend-specific UX is expected.

### 13.5 Macro-only implementation

Do not make compile-time Nim AST transformations the only representation of the application. The IR must be inspectable, testable, serializable where useful, and consumable by multiple backend compilers/runtimes.

### 13.6 Theme-only motif model

Do not reduce Motif to colors and fonts. Conversely, do not let Motif alter operation semantics or machine contracts.

### 13.7 Shell-centric task runner

Flow execution must compose typed operations and artifacts, not merely concatenate shell recipes.

---

## 14. Feasibility experiments

The most useful experiments are cross-product tests rather than isolated backend demos.

### Experiment A: canonical operation round-trip

Represent a small domain with:

```text
User
- id
- name
- email
- status

listUsers
getUser
createUser
deleteUser
```

Provide three engine implementations:

- native Nim,
- OpenAPI-backed HTTP,
- structured CLI-backed.

All three must normalize to equivalent Operation IR.

### Experiment B: non-visual projection

Generate both:

- CLI,
- MCP.

from the same canonical operations.

Success indicates that the core has not accidentally become a UI description language.

### Experiment C: semantic interactive projection

Define one Application DSL that provides browse, inspect, edit, and destructive confirmation semantics. Generate:

- interactive CLI,
- TUI,
- basic Web.

Compare required backend-specific overrides.

### Experiment D: Flow composition

Define:

```text
build
  ├─ lint
  └─ test
       ↓
    package
       ↓
    publish
```

Use a mixture of native, process, and HTTP operations. Verify parallelism, failure propagation, retry, cancellation, progress, and artifact edges.

Expose the Flow as:

- CLI command,
- TUI progress view,
- MCP tool.

### Experiment E: Motif independence

Apply at least two materially different motifs to the same application and flow. Verify that:

- TUI/Web presentation changes substantially,
- semantic operation identity does not change,
- batch CLI remains deterministic,
- MCP schemas and machine-readable results remain stable.

### Experiment F: capability degradation

Create semantics intentionally unsupported by one backend and verify explicit `native / fallback / unsupported` diagnostics rather than silent loss.

---

## 15. Go / No-Go criteria

The project should be considered architecturally viable when the following hold simultaneously:

1. Native Nim, OpenAPI, and structured CLI implementations can represent the same domain through one canonical Operation IR.
2. The same IR can generate useful CLI and MCP outputs without UI-specific metadata becoming mandatory.
3. One semantic Application definition can generate useful TUI and Web interfaces with backend-specific hints rather than backend-specific rewrites.
4. A Flow can compose operations from more than one engine type and expose itself again as a higher-level operation.
5. Typed Flow state can drive progress interfaces without hand-authored progress UI logic.
6. Motifs can change presentation and feedback without changing machine contracts.
7. Unsupported external schema or backend capabilities produce explicit diagnostics and controlled degradation.

Failure of an individual external-spec feature is not by itself a No-Go. Failure to maintain a stable semantic boundary across these experiments is.

---

## 16. Current assessment

Desk-based feasibility assessment: **Go, with the canonical semantic model as the primary research problem.**

The project does not depend on an unknown rendering technique or an unavailable Nim capability. CLI, MCP, TUI, Web, task execution, and native engine bindings are all individually implementable.

The uncertain part is the shared abstraction boundary:

```text
External contracts
       ↓
Canonical Schema + Operation + Resource semantics
       ↓
Application + Flow semantics
       ↓
Backend-specific projections
```

Research and implementation should therefore evaluate this boundary before optimizing any individual UI backend.

Rhyzora should be judged successful not by whether it can generate five superficial frontends, but by whether an engine or library that satisfies one explicit contract can gain useful human, agent, development, and automation interfaces without duplicating its business semantics.
