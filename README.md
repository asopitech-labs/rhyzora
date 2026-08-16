# Rhyzora

**Build CLI, TUI, Web, desktop, and MCP interfaces from declarative application and engine contracts.**

Rhyzora is a Nim framework for defining application capabilities and interactions independently from how they are presented.

**[Explore the concept site](https://asopitech-labs.github.io/rhyzora/)**

Instead of implementing a CLI, TUI, Web application, desktop application, and MCP server separately, Rhyzora describes the underlying operations and application semantics once and projects them into different interfaces.

```text
                         Application
                             │
                     Application DSL
                             │
                     Application IR
                             │
          ┌──────────┬───────┼───────┬──────────┐
          │          │       │       │          │
         CLI        TUI     MCP     Web      Desktop
```

Business logic is kept separate from presentation. Existing APIs, command-line tools, and native Nim libraries can be connected through engine contracts.

```text
 OpenAPI ─────┐
 CLI ─────────┤
 Native Nim ──┼── Engine Contract ── Application DSL
 MCP ─────────┤                         │
 Other ───────┘                         │
                                        ▼
                          CLI / TUI / MCP / Web / Desktop
```

## Why Rhyzora?

Applications increasingly expose the same capabilities through multiple interfaces.

A single operation may need to appear as:

* a CLI command for scripts and developers,
* an interactive TUI for terminal users,
* a Web or desktop interface for human users,
* an MCP tool for AI agents.

These interfaces usually duplicate the same information: operations, arguments, types, validation, errors, actions, resources, and workflows.

Rhyzora makes those semantics explicit and reusable.

## Follow and participate

Rhyzora is being shaped in the open. You do not need to write code to take part.

- **[Star the repository](https://github.com/asopitech-labs/rhyzora)** to save the project and help others discover it.
- **[Watch the repository](https://github.com/asopitech-labs/rhyzora/subscription)** to follow the activity that matters to you.
- **[Share the concept site](https://asopitech-labs.github.io/rhyzora/)** with people building applications across multiple interfaces.
- **[Join the discussion](https://github.com/asopitech-labs/rhyzora/discussions)** to propose a use case or challenge the model.
- **[Open an issue](https://github.com/asopitech-labs/rhyzora/issues/new/choose)** for a concrete feature, problem, or implementation task.

## Core concepts

### Engine Contract

An engine describes **what an application can do**, independently from its user interface.

Conceptually:

```nim
operation searchUsers:
  input:
    query: string
    limit: int = 20

  output:
    seq[User]

  effect:
    read
```

An engine may be backed by a native Nim implementation or imported/adapted from an external interface such as OpenAPI or a structured CLI specification.

Transport-specific details remain in bindings rather than leaking into the application model.

```text
Operation
    ├── Native binding
    ├── HTTP / OpenAPI binding
    ├── CLI binding
    └── MCP binding
```

### Application DSL

The application layer describes **how capabilities are used**, rather than how individual widgets are rendered.

Conceptually:

```nim
app Users:
  use userEngine

  screen Main:
    browse users:
      search name
      show name, email, status

      select -> Detail

  screen Detail:
    inspect selected

    actions:
      delete selected:
        confirm
```

The application does not need to know whether `users` is provided by an HTTP API, an executable, or a native Nim procedure.

### Interface backends

Backends project application semantics into interfaces appropriate for their environment.

The same `delete` action might become:

```text
CLI       user delete <id>
TUI       Delete action + confirmation
Web       Delete button + confirmation dialog
Desktop   Native application action
MCP       delete_user tool
```

Rhyzora preserves the operation's meaning rather than attempting to reproduce an identical UI across every backend.

## Engine interoperability

Rhyzora's engine model is designed around typed operations.

```text
              Engine Contract
                    │
          ┌─────────┼─────────┐
          │         │         │
        Input     Output     Effects
        Schema    Schema     / Errors
```

External interface descriptions can be normalized into this model.

Planned sources include:

* native Nim engines,
* OpenAPI,
* machine-readable CLI specifications,
* MCP tools,
* other structured operation descriptions.

The canonical engine contract, rather than any external specification, defines the boundary between business logic and Rhyzora.

## Semantic UI generation

Rhyzora operates above individual widgets whenever possible.

Instead of describing:

```text
Label
Input
Button
Table
Dialog
```

an application can describe:

```text
search
browse
inspect
edit
select
confirm
action
```

Each backend decides how those interactions should be represented.

Lower-level presentation hints and components can be used when an application requires more control.

```text
Semantic application
        │
        ▼
Presentation hints
        │
        ▼
Backend components
        │
        ▼
Platform primitives
```

## Architecture

```text
                       Rhyzora
                          │
                    Schema Model
                          │
                  Operation Model
                          │
                   Engine Contract
                          │
                    Application IR
                          │
              Interaction / State Model
                          │
       ┌─────────┬────────┼────────┬─────────┐
       │         │        │        │         │
      CLI       TUI      MCP      Web     Desktop
```

The core is independent of terminal renderers, Web frameworks, desktop toolkits, and transport implementations.

## Design principles

**Semantics before presentation.**
Describe what an application does before deciding how it looks.

**Business logic stays outside the UI.**
Engines expose typed capabilities. Interfaces consume them.

**One application, multiple interfaces.**
CLI, TUI, Web, desktop, and agent interfaces are different projections of the same application semantics.

**Use existing systems.**
Existing Web APIs and command-line applications should be usable as engines rather than rewritten.

**Structured contracts over output scraping.**
Typed input and output contracts are preferred over interpreting human-oriented text.

**Progressive control.**
Use high-level semantic definitions by default and descend into presentation or platform-specific primitives only when necessary.

**Backend-appropriate UX.**
Rhyzora does not attempt to make a terminal, browser, desktop application, and AI agent behave identically.

## Status

Rhyzora is currently under design and early development.

The architecture and DSL are subject to change while the canonical schema model, operation model, engine boundary, and backend projection model are validated.

## License

MIT
