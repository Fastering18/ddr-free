---
name: ddr-diagrams
description: Write, check and render DDR (.ddr) diagrams as text: flowcharts, graphs and architecture, UML use case and activity diagrams, CDM, PDM and Chen ERD, plus .tdr documents. Use when the user asks for a diagram, a data model or a use case specification, or edits a .ddr/.tdr file.
---

# DDR diagrams

DDR 0.0.5-alpha.5 draws diagrams from plain text. Generated from the engine; full guide: /docs/agents.md (or the MCP tool `get_syntax`).

## Workflow

1. Pick the type from the table below.
2. Write the source following the grammar of that type (call `get_syntax` with `type` when the MCP server is available).
3. Run `check` (MCP) or `ddr check file.ddr` and fix every error and warning.
4. Run `format` / `ddr fmt -w file.ddr`.
5. Render with `render` (MCP) or `ddr render -o out.svg file.ddr`.

## Rules

Workflow: `get_syntax` → write the source → `check` until `ok` is true (no errors, no warnings) → `format` → `render`.

**Do**

- Start every .ddr file with `Type:` and `Title:` lines, then a blank line.
- Use exactly the `Type:` values in the type list; use `Type: Graph` for architecture diagrams.
- Declare every node before an edge or `goto` names it, and give nodes you reference an explicit id (`pay: Enter payment`).
- Indent with spaces (two per level). One statement per line.
- Quote labels that contain `;`, `{`, `}`, `[`, `->` or start with a keyword.
- Run `check` after every edit and fix every error and warning before you answer.
- Run `format` (or `ddr fmt -w`) to normalize the file; it never reorders statements.

**Don't**

- Don't invent types (`Sequence`, `Class`, `Architecture` and `UserStory` are not types).
- Don't use tabs (W101) or Markdown fences inside a .ddr file.
- Don't add `[x, y]` or `[via: …]` unless the user asked for fixed positions; auto layout is better.
- Don't write `[x, y]` in activities: they are ignored (W110).
- Don't repeat foreign keys in a CDM; the `Ref:` line is the link.
- Don't put chains (`a -> b`) inside ERD relations, or `->` after a `goto`.
- Don't use `//` or trailing comments: only full-line `#` comments exist.
- Don't run `check` on a .tdr document (E108); check the .ddr files it embeds.

## Diagram types

| id | `Type:` | file | views |
|----|---------|------|-------|
| [flowchart](#flowchart) | `Flowchart` | `.ddr` | `flow` |
| [graph](#graph) | `Graph` | `.ddr` | `graph` |
| [architecture](#architecture) | `Graph` | `.ddr` | `graph` |
| [usecase](#usecase) | `UseCase` | `.ddr` | `overview`, plus `flow: <use case>` per use case with a `Flow` |
| [activity](#activity) | `Activity` | `.ddr` | `activity` |
| [cdm](#cdm) | `CDM` | `.ddr` | `cdm` |
| [pdm](#pdm) | `PDM` | `.ddr` | `pdm` |
| [erd](#erd) | `ERD` | `.ddr` | `erd` |
| [tdr](#tdr) | - | `.tdr` | page preview |

## Diagnostic codes

Every code the engine emits (read from `core/ddr/diag.go`). `E` codes are errors: the diagram is not drawn correctly. `W` codes are warnings: drawn, but fix them.

| code | severity | meaning | fix |
|------|----------|---------|-----|
| `E101` | error | unclosed `{` | Add the missing `}` (or remove the stray `{`). |
| `E102` | error | bad indentation | Indent with spaces to match the item above; fields go two spaces under their `- ` item. |
| `E103` | error | unknown or unexpected statement (also an empty actor group, `- Clerk {}`) | Check the statement spelling, or quote the label (`"if this is text"`). An actor group needs at least one case. |
| `E104` | error | unexpected `}` | Remove the extra `}`. |
| `E105` | error | unclosed `"` | Close the string with `"`, or escape a literal quote as `\"`. |
| `E106` | error | unclosed `[` | Close the attribute group with `]`. |
| `E107` | error | bad attribute syntax | Write attributes as `[key: value, flag]`; an `id` is one word. |
| `E108` | error | a `.tdr` document given to `check` (prose with embedded `.ddr` diagrams): check the `.ddr` files it embeds instead | Run `check` on the `.ddr` files the document embeds (the paths in `::diagram[...]` / `::usecases[...]`), not on the `.tdr` itself. |
| `E201` | error | unknown actor | Declare the actor under `Actors:` or fix its spelling. |
| `E202` | error | unknown use case in `Includes` / `Extends` | Name an existing use case in `Includes` / `Extends`. |
| `E203` | error | unknown edge or goto target | Declare the target node (with `id: Label`) before using it in an edge or `goto`; edges never create nodes. |
| `E204` | error | duplicate id | Give each node a unique id. |
| `E205` | error | activity: unknown lane (not in `Lanes:`) | Add the lane to `Lanes:` or fix the prefix. |
| `E206` | error | activity: more than one start (`start` written twice or after a step) | Write `start` at most once, as the first statement (it is implicit anyway). |
| `E207` | error | data model: a relationship names an entity or table that is not declared | Declare the entity or table, or fix the name in the `Ref:` / relation. |
| `W101` | warning | tab indentation | Replace tabs with spaces. |
| `W102` | warning | unknown key (kept) | Remove or rename the key; unknown keys are kept but unused. |
| `W104` | warning | actor with no use cases | Link the actor to a use case or remove it. |
| `W105` | warning | unknown attribute (ignored) | Remove the attribute or use one from the attribute tables. |
| `W106` | warning | missing `Type` / `Title` header | Add `Type:` and `Title:` header lines at the top. |
| `W107` | warning | unknown `Type` | Use one of the documented types (Architecture is `Type: Graph`). |
| `W108` | warning | a use case includes or extends itself (the line is not drawn) | Remove the self include / extend. |
| `W109` | warning | include cycle (drawn; the cases keep their actor lines) | Break the include cycle. |
| `W110` | warning | layout adjustment (activities: `[x, y]` ignored, Direction ignored): a `[via: ...]` bend pulled inside the system boundary so an actor stays outside, a line that cannot fit without reaching an actor, or an actor pinned inside the boundary | Remove the `[x, y]` / `[via: …]` the layout cannot honor (activities ignore them). |
| `W111` | warning | activity: the branches of a decision meet again without a merge | Let the branches meet again (the structured `if` always does). |
| `W112` | warning | activity: a parallel branch reaches the flow after the join without passing it | Let every parallel branch reach the join. |
| `W113` | warning | activity: a step no flow from the start reaches | Connect the step to the flow or delete it. |
| `W114` | warning | activity: a lane in `Lanes:` with no steps | Remove the empty lane from `Lanes:` or put a step in it. |
| `W115` | warning | activity: a `parallel` block with an empty branch (left out) | Add a step to the empty parallel branch or remove the branch. |
| `W116` | warning | activity: an action with more than one incoming or outgoing flow | Use a decision or merge instead of several `a -> b` flows on one action. |
| `W117` | warning | activity: the layout did not converge; the result may overlap | Simplify the activity (fewer crossing `goto`s); the result may overlap. |
| `W118` | warning | data model: a reference that is not `a.attr < b.attr`, or an attribute the reference names but the entity does not have | Write references as `a.attr < b.attr` with attributes the entities have; per-segment `via` goes in `Refs`. |
| `W119` | warning | data model: a relationship from an entity to itself (not drawn) | Self relationships are not drawn in CDM/PDM; model them with a link entity. |
| `W120` | warning | `Numbering:` on and fewer main use cases than `Min Use Cases:` (default 6) | Add main use cases or lower `Min Use Cases:`. |
| `W121` | warning | two use cases have the same code | Give each use case a unique `[code: …]`. |
| `W122` | warning | `Alternative N` / `Alur Alternatif N.M`, or a `goto N` inside it, names a main flow step that does not exist | Number the alternative after an existing main flow step. |
| `W123` | warning | a `[code: ...]` that does not follow the `Numbering:` format (`UC-01` under `Numbering: UC`) | Follow the `Numbering:` format, e.g. `UC-01` under `Numbering: UC-`. |
| `W124` | warning | ERD: a relation that joins fewer than two entities | Make the relation name at least two entities (the same one twice for a self relation); no chains inside relations. |
| `W125` | warning | ERD: an entity without a key attribute (`*`) | Mark a key attribute with `*`. |
| `W126` | warning | a targetless `-> [..]` clause or a trailing `->` inside an edge chain (ignored; `fmt` drops a bare trailing `->`) | Remove the targetless `-> [..]` or trailing `->` from the chain (`ddr fmt` drops a bare trailing `->`). |
| `W127` | warning | `goto x -> y`: a goto jumps to one target, the `->` part is ignored | Write `goto x` alone; add a separate edge if you need one. |
| `W128` | warning | flowchart/graph: a later `End` with a different label joins the one End; its label is not drawn | Write one `End` (or give every `End` the same label); all paths join the diagram's single End. |
| `W129` | warning | unknown `Lang:` value (not `en` or `id`, any case); falls back to English (`en`) | Use `Lang: en` or `Lang: id` (or drop the line for English). |

## Examples

### Flowchart

```ddr
Type: Flowchart
Title: Password reset
Direction: TB

# One Flow block. Steps run top to bottom; `if`, `loop` and `goto` branch.
Flow {
  Start;
  Enter email address;
  if Account exists? {
    Send reset link;
    open: Open link from email;
    if Link still valid? {
      Choose new password;
    } else {
      Show expired message;
      goto open;
    }
  } else {
    Show generic confirmation;
  }
  End;
}
```

### Graph

```ddr
Type: Graph
Title: Order service
Direction: LR

Graph {
  user: Customer [shape: actor];
  web: Web app [shape: round];
  api: Order API;
  queue: Event queue [shape: pill, color: amber];
  db: Orders database [shape: cylinder];
  mail: Email service [shape: round, color: teal];
  user -> web: browses;
  web -> api: JSON over HTTPS;
  api -> db: SQL;
  api -> queue -> mail: order.created;
}
```

### Architecture

```ddr
Type: Graph
Title: Web platform architecture
Direction: LR

# Architecture diagrams are Graphs with shapes and colors for each tier.
Graph {
  browser: Browser [shape: round];
  cdn: CDN [shape: pill, color: teal];
  app: App server;
  worker: Background worker [color: purple];
  cache: Cache [shape: cylinder, color: red];
  db: Database [shape: cylinder];
  store: File storage [shape: cylinder, color: amber];
  browser -> cdn: static files;
  browser -> app: API calls;
  app -> cache;
  app -> db: reads and writes;
  app -> worker: jobs [dashed];
  worker -> store: uploads;
}
```

### Use case

```ddr
Type: UseCase
Title: Online Store
Author: Product Team

Actors:
- Customer
- Store Manager
- Payment Provider [system]

Use Cases:
- Customer: Browse catalog
  Description: The customer looks through products by category.
- Customer: Place order
  Description: The customer buys the items in the cart.
  Includes: Pay for order
- Customer, Payment Provider: Pay for order
  Description: The order total is charged to the customer.
- Customer: Track order
  Extends: Place order
- Store Manager: Manage products
  Description: The manager adds, edits and removes products.
```

### Activity

```ddr
Type: Activity
Title: Support ticket
Lanes: Customer, Support, System

Flow {
  Customer: Submit ticket
  System: Assign ticket number
  Support: Review ticket
  if Needs more information? {
    Support: Ask customer for details
    Customer: Reply with details
  } else {
    Support: Accept ticket
  }
  parallel {
    Support: Resolve issue
  } and {
    System: Notify customer of progress
  }
  Customer: Confirm resolution
}
```

### Conceptual data model (CDM)

```ddr
Type: CDM
Title: Online Store

Entities:
- Customer {
    Customer ID: varchar(12) [identifier]
    Full Name: varchar(80) [mandatory]
    Email: varchar(120)
  }
- Order {
    Order ID: varchar(12) [identifier]
    Order Date: date [mandatory]
    Total: double
  }
- Product {
    Product ID: varchar(12) [identifier]
    Name: varchar(100) [mandatory]
    Price: double
  }

Ref: Customer < Order: places
Ref: Order <> Product: contains
```

### Physical data model (PDM)

```ddr
Type: PDM
Title: Online Store

Tables:
- customers {
    customer_id: bigint [pk]
    full_name: varchar(120) [not null]
    email: varchar(180) [not null]
  }
- orders {
    order_id: bigint [pk]
    customer_id: bigint [not null]
    ordered_at: timestamp [not null]
  }
- order_items {
    order_id: bigint [pk]
    product_id: bigint [pk]
    quantity: int [not null]
  }
- products {
    product_id: bigint [pk]
    name: varchar(100) [not null]
    price: numeric(10,2) [not null]
  }

Ref: customers.customer_id < orders.customer_id
Ref: orders.order_id < order_items.order_id
Ref: products.product_id < order_items.product_id
```

### Entity relationship (ERD, Chen)

```ddr
Type: ERD
Title: Online Store

Entities:
- Customer { Customer ID *; Name; Email }
- Order { Order ID *; Order Date; Customer ID ** }
- Product { Product ID *; Name; Price }

Relations:
- Places { Customer: 1; Order: N }
- Contains { Order: M; Product: N; Quantity }
```

### Documents (.tdr) and use case specifications

```tdr
Title: Online Store requirements
Author: Product Team
---

# Introduction

This document describes the Online Store. Each figure is a live view of a
`.ddr` file, so it stays in sync with the diagram source.

::diagram[diagrams/usecase.ddr]{view="overview" caption="Use cases"}

## Use case specifications

::usecases[diagrams/usecase.ddr]{title=""}
```

