# DDR for agents: MCP, skills and strict syntax

> Machine-oriented DDR guide: MCP server setup and tools, a downloadable SKILL.md, a strict grammar for every diagram type, diagnostic codes with fixes, and do/don't rules.

Generated from the DDR engine 0.0.5-alpha.5 (docs/spec, core/ddr/diag.go, internal/server, docs/examples) by `go run ./cmd/ddr-docs`. Do not edit by hand.

Human-friendly reference: /docs/syntax.md · This file is also served as /llms.txt and /docs/agents.md.

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

## MCP server

`ddr mcp` runs a Model Context Protocol server over **stdio** (newline-delimited JSON-RPC 2.0, protocol 2025-06-18). Install the CLI from the GitHub releases (or `go install github.com/Fastering18/ddr/cmd/ddr@latest`), then add it to your client.

**Cursor** (`.cursor/mcp.json` or `~/.cursor/mcp.json`):

```json
{
  "mcpServers": {
    "ddr": { "command": "ddr", "args": ["mcp"] }
  }
}
```

**Claude Desktop** (`claude_desktop_config.json`):

```json
{
  "mcpServers": {
    "ddr": { "command": "/usr/local/bin/ddr", "args": ["mcp"] }
  }
}
```

**Any stdio client:** start `ddr mcp` and write one JSON-RPC message per line:

```jsonl
{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"me","version":"1"}}}
{"jsonrpc":"2.0","method":"notifications/initialized"}
{"jsonrpc":"2.0","id":2,"method":"tools/list"}
{"jsonrpc":"2.0","id":3,"method":"tools/call","params":{"name":"check","arguments":{"source":"Type: Flowchart\nTitle: T\n\nFlow {\n  Start\n  End\n}\n"}}}
```

**HTTP clients:** the server has no HTTP transport of its own. Run `ddr mcp` behind any stdio-to-HTTP MCP bridge, or call the same core with the CLI (`ddr check`, `ddr render`).

### Tools

#### `check`

Parse and analyze a .ddr source (a .tdr document gives only E108: check its .ddr files instead). Returns JSON {ok, errors, warnings, diagnostics:[{code,severity,message,line,col}], views:[{kind,name}]}. ok is true only when there are no errors and no warnings.

Input schema:

```json
{
  "additionalProperties": false,
  "properties": {
    "source": {
      "description": "Full text of a .ddr file.",
      "type": "string"
    }
  },
  "required": [
    "source"
  ],
  "type": "object"
}
```

#### `render`

Render a .ddr source to a standalone SVG document. Diagnostics, if any, follow the SVG after a line `<!-- diagnostics -->`.

Input schema:

```json
{
  "additionalProperties": false,
  "properties": {
    "source": {
      "description": "Full text of a .ddr file.",
      "type": "string"
    },
    "theme": {
      "default": "light",
      "enum": [
        "light",
        "dark"
      ],
      "type": "string"
    },
    "view": {
      "description": "View name from check (default: the first view).",
      "type": "string"
    }
  },
  "required": [
    "source"
  ],
  "type": "object"
}
```

#### `format`

Format a .ddr source like `ddr fmt` (idempotent, keeps comments, never reorders). Returns the formatted text, or an error listing the syntax errors.

Input schema:

```json
{
  "additionalProperties": false,
  "properties": {
    "source": {
      "description": "Full text of a .ddr file.",
      "type": "string"
    }
  },
  "required": [
    "source"
  ],
  "type": "object"
}
```

#### `list_types`

List every diagram and document type: id, the `Type:` header value, accepted aliases, file extension, views and a one-line summary.

Input schema:

```json
{
  "additionalProperties": false,
  "properties": {},
  "type": "object"
}
```

#### `get_syntax`

Return the strict DDR syntax guide for agents (Markdown). Pass `type` (an id from list_types) for just that type's section.

Input schema:

```json
{
  "additionalProperties": false,
  "properties": {
    "type": {
      "description": "Optional type id, e.g. flowchart, erd.",
      "type": "string"
    }
  },
  "type": "object"
}
```

## Skills

Download [SKILL.md](/docs/SKILL.md) and put it in your agent's skills folder (for example `.cursor/skills/ddr/SKILL.md` or `~/.claude/skills/ddr/SKILL.md`). It tells the agent when to use DDR, the workflow, the rules and one example per type, and points to the MCP tools when they are available.

Raw Markdown of this page: [/docs/agents.md](/docs/agents.md) and [/llms.txt](/llms.txt).

## Header keys

The header is `Key: value` lines at the top of the file, ending at the first blank line. Keys are case-insensitive; unknown keys give W102 and are kept.

| key | required | values | default | meaning |
|-----|----------|--------|---------|---------|
| `Type` | **yes** | see the type list | - | Selects the diagram type. Missing: W106; unknown: W107. |
| `Title` | no | text | - | Diagram title. Missing: W106 (warning only). |
| `Author` | no | text | - | Metadata, not drawn. |
| `Date` | no | text (ISO date recommended) | - | Metadata, not drawn. |
| `Version` | no | text | - | Metadata. |
| `Status` | no | text | - | Metadata. |
| `Tags` | no | comma list | - | Metadata. |
| `Theme` | no | `light` `dark` | light | Preferred theme for exports. |
| `Format` | no | text | - | Metadata. |
| `Direction` | no | `TB` `LR` | `TB` (`LR` for CDM/PDM) | Main layout direction. Also accepted inside a `Layout:` section. Ignored in Activity (W110). |
| `Lang` | no | `en` `id` | en | Language of generated use case specification text. |
| `Line Jumps` | no | `on` `off` | on | Draw a small hop where two straight lines cross. |

## Lines, comments and strings

- **Comments:** a full line starting with `#`. `Order #123` is text, and there are no trailing or `//` comments. `ddr fmt` keeps comments; an empty comment stays a bare `#`.
- **Statements:** inside `{ }` a statement ends at `;` or a newline; the trailing `;` is optional. In indented lists, one item per line after `- `.
- **Indentation:** spaces only (tabs give W101).
- **Colons** mean something only after a key at the start of a line (`Description:`), in an item head (`- Customer: Login`), and after an id (`pay: Enter payment`). Anywhere else a colon is text.
- **Strings:** plain text by default. Quote with `"…"` when a label contains `;`, `{`, `}`, `[`, `->`, starts with a keyword, or spans lines. Escapes: `\"`, `\\`, `\n`.
- **Multiline labels:** a quoted label may continue on the next lines; leading blanks up to the statement's column are dropped.

```ddr
# A comment line
Type: Flowchart
Title: Lexical example

Flow {
  Start
  Read order #123
  note: "Contains; a semicolon"
  End
}
```

## Sections

A section is `Name:` followed by an indented `- ` list, or `Name { … }` (a block). Section names are case-insensitive.

| name | used in | meaning |
|------|---------|---------|
| `Actors` | UseCase | Actor list. |
| `Use Cases` | UseCase | Use case list; alias `UseCases`. |
| `Flow` | Flowchart, Activity, use cases | The flow block. |
| `Activity` | Activity | Alias of `Flow` in activities. |
| `Graph` | Graph | Nodes and edges; alias `Nodes`. |
| `Nodes` | Graph | Alias of `Graph`. |
| `Edges` | Graph | Extra edges for the graph. |
| `Entities` | CDM, PDM, ERD | Entity list; alias `Entity`. |
| `Entity` | CDM, PDM, ERD | Alias of `Entities`. |
| `Tables` | PDM | Table list; alias `Table`. |
| `Table` | PDM | Alias of `Tables`. |
| `Relations` | ERD | Relation list; alias `Relation`. |
| `Relation` | ERD | Alias of `Relations`. |
| `References` | CDM, PDM | Reference list; also `Ref { }` or `Ref:` lines; `Refs` for per-segment `via`. |
| `Layout` | all | Section form of layout keys: `Layout:` then `Direction: LR`. |
| `Numbering` | UseCase | Header: use case codes. |
| `Show Codes` | UseCase | Header: print codes. |
| `Min Use Cases` | UseCase | Header: W120 threshold (default 6). |
| `Lanes` | Activity | Header: swimlanes. |
| `Line Jumps` | all | Header: `off` disables line hops. |

## Ids, labels and attributes

Every node is `id: Label [key: value, flag]`. Without an id the id is the slug of the label (`Enter payment details` → `enter-payment-details`). Ids match `[A-Za-z_][A-Za-z0-9_.-]*`. Several attribute groups are allowed: `Login [shape: round] [x: 10, y: 20]`. Unknown attributes give W105.

Node attributes:

| attribute | meaning |
|---|---|
| `x` `y` | Pinned centre position (numbers). Without them the node gets auto layout. Ignored in Activity (W110). |
| `w` `h` | Size override (numbers). |
| `shape` | `rect` `round` `pill` `ellipse` `circle` `diamond` `parallelogram` `actor` `cylinder` `box`. |
| `color` | `accent` `purple` `amber` `orange` `teal` `pink` `blue` `green` `red` `gray`, or `#hex`. |
| `style` | Node style hint. |
| `label` | Display label when the id is all you wrote. |
| `id` | Alias for the node, same as an `id:` prefix. One word (E107 otherwise); a duplicate is E204. |
| `system` `external` `primary` `secondary` `deprecated` | Flags: actor and use case kinds. |
| `side` | Actors: `left` or `right` of the system boundary. |
| `code` | Use case code set by hand, e.g. `[code: UC07]`. |
| `spec` | Use case specification table: `own` or `none`. |
| `identifier` `pi` | CDM identifier (`<pi>` and `<M>`); also a key in PDM. |
| `pk` `fk` | PDM primary / foreign key column. |
| `unique` | PDM unique column (kept, not drawn). |
| `null` `nullable` `not null` `nn` `required` | PDM nullability (kept, not drawn); `required` / `[mandatory]` / `[M]` mark CDM `<M>`. |

## Positions and bends

- `[x: 120, y: 80]` pins a node's centre (the editor writes it when you drag a node). Nodes without positions get auto layout.
- `[via: (220,120) (220,300)]` on an edge sets its bend points in order (the editor writes it when you drag a line).
- ERD attributes take `[x, y]` too. Activities ignore both (W110).

```ddr
Type: Graph
Title: Pinned

Graph {
  a: Client [x: 100, y: 80]
  b: Server [x: 400, y: 80]
  a -> b: request [via: (250,20)]
}
```

## Edges and chains

| where | syntax |
|---|---|
| Graph, Flowchart, Activity | `a -> b` / `a -> b: label [attrs]` |
| Flow step | `step -> [attrs]` styles the implicit edge to the next step; `step -> target` adds an edge |
| `if` | `if ok? -> [yes attrs] -> [no attrs] { … }` |
| `goto` | `goto id [attrs]` (no `->` after it: W127) |
| Use case | `- Customer: Register -> Customer [attrs]` |
| CDM / PDM | `Ref: a.id < b.a_id: name [attrs]` with `<` `>` `-` `<>` |

**Chains.** `a -> b -> c` adds one edge per consecutive pair. A label (`: text`) and attribute groups belong to the segment they follow; nothing is copied to other segments. Spaces around `->` are optional (`a->b->c`). Every unknown node is E203; nodes are never created by an edge. A targetless `-> [..]` inside a chain or a trailing `->` is ignored with W126. References chain with their own operators: `Ref: a.id < b.a_id: owns < c.b_id: keeps`.

Edge attributes:

| attribute | meaning |
|---|---|
| `via` | Bend points in order: `[via: (120,80) (240,80)]`. |
| `style` | `straight` (default, polyline) or `curve`. |
| `label` | Edge label (same as `: text`). |
| `dash` `dashed` | Flag: dashed line. |
| `arrow` | `end` (default) `none` `both`. |
| `color` | As for nodes. |
| `head` | Arrowhead: `triangle` (generalization), `open`, `filled`. |
| `from` `to` | Explicit endpoints on a data-model reference. |

```ddr
Type: Graph
Title: Chain

Graph {
  a: Browser
  b: API
  c: Database [shape: cylinder]
  a -> b: HTTPS -> c: SQL [dashed]
}
```

## Flow statements

Used in Flowchart, Activity and use case `Flow { }` blocks. Keywords count only at the start of a statement and are case-insensitive: `if`, `elif`, `else`, `loop`, `goto`, `Start`, `End`; in activities also `parallel` … `and`, `stop`.

```ebnf
stmt = step | "Start" [label] | "End" [label]
     | "if" cond block { "elif" cond block } [ "else" block ]
     | "loop" [cond] block | "goto" id | id ":" label | edge
     | "parallel" block { "and" block } | "stop" ;   (* activity only *)
```

- `if cond? { … }` leaves by `Yes` into the block and by `No` past it.
- `loop cond? { … }` asks before each pass; `loop { … }` asks `Repeat?`.
- `@Name` in a step links it to an actor.
- Line jumps: where two straight lines cross, one hops over the other. `Line Jumps: off` turns it off.

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

## HTTP API

The cloud server's REST API. Paths are under `/api/v1` except `/healthz`, `/ws/…` and the browser redirects of Google sign-in. JSON in and out; errors are `application/problem+json` (`{"type","title","status","detail"}`). Auth is the `ddr_session` cookie set by login/register; `optional` means signed-out callers see public projects only. State-changing requests must come from the same origin. Every route below is read from `internal/server/server.go`; a test fails if one is missing.

Diagrams are rendered in the browser (WebAssembly), by `ddr render` and by the MCP `render` tool; the server stores text and has no render endpoint yet.

### Health

**`GET /healthz`** · auth: none  
Liveness probe (outside /api/v1).

Response:

```
200 {"status":"ok","db":true,"realtime":{…}} (503 "degraded" when the database is down)
```

**`GET /api/v1/health`** · auth: none  
Liveness probe.

Response:

```
200 {"status":"ok","db":true,"realtime":{…}} (503 "degraded" when the database is down)
```

### Auth

**`POST /api/v1/auth/register`** · auth: none  
Create an account and sign in (sets the session cookie). Rate limited.

Request:

```
{"email":"ana@example.com","password":"correct horse battery","name":"Ana"}
```
Response:

```
201 {"id":"7b6c…","email":"ana@example.com","name":"Ana","avatarUrl":null,"emailVerified":true,"createdAt":"2026-10-22T10:00:00Z"}
```

**`POST /api/v1/auth/login`** · auth: none  
Sign in with email and password (sets the session cookie). Rate limited.

Request:

```
{"email":"ana@example.com","password":"correct horse battery"}
```
Response:

```
200 {"id":"7b6c…","email":"ana@example.com","name":"Ana","avatarUrl":null,"emailVerified":true,"createdAt":"2026-10-22T10:00:00Z"}
```

**`POST /api/v1/auth/logout`** · auth: optional  
End the current session and clear the cookie.

Response:

```
204 No Content
```

**`POST /api/v1/auth/verify`** · auth: none  
Confirm an email address with the token from the mail.

Request:

```
{"token":"v_abc…"}
```
Response:

```
200 {"id":"7b6c…","email":"ana@example.com","name":"Ana","avatarUrl":null,"emailVerified":true,"createdAt":"2026-10-22T10:00:00Z"}
```

**`POST /api/v1/auth/verify/resend`** · auth: session  
Send a new verification link (valid 24h, single use). At most one per minute and 10 per day per user; otherwise 429 with `Retry-After`. Mail goes through Resend (`RESEND_API_KEY`) or SMTP (`SMTP_HOST`…), from `MAIL_FROM`; with neither, the link is written to the server log.

Response:

```
200 {"retryAfter":60,"remainingToday":9,"cooldown":60,"dailyLimit":10}
```

**`GET /api/v1/auth/verify/status`** · auth: session  
Whether your email is verified, whether verification is required (`EMAIL_VERIFICATION_REQUIRED`; when false new emails are verified at once and resend answers 409 `verification_disabled`), and when Resend is allowed again.

Response:

```
200 {"verified":false,"required":true,"retryAfter":42,"remainingToday":8,"cooldown":60,"dailyLimit":10}
```

**`POST /api/v1/auth/forgot`** · auth: none  
Request a password reset mail. Always 204, so it cannot probe for accounts.

Request:

```
{"email":"ana@example.com"}
```
Response:

```
204 No Content
```

**`POST /api/v1/auth/reset`** · auth: none  
Set a new password with the reset token.

Request:

```
{"token":"r_abc…","password":"new long password"}
```
Response:

```
204 No Content
```

**`GET /api/v1/auth/oauth/{provider}`** · auth: none  
Generic OAuth route for other providers: always 501. Google sign-in has its own routes (`GET /auth/google`) once they are on this server.

Response:

```
501 {"type":"…/not_implemented","status":501,"detail":"…"}
```

**`GET /api/v1/auth/oauth/{provider}/callback`** · auth: none  
Callback of the generic OAuth route: always 501.

Response:

```
501 {"type":"…/not_implemented","status":501,"detail":"…"}
```

**`GET /api/v1/auth/providers`** · auth: none  
Which sign-in methods this server offers.

Response:

```
200 {"password":true,"google":true}
```

**`GET /auth/google`** · auth: none  
Start Google sign-in (outside /api/v1). `?next=/path` is where to land afterwards (same-site paths only).

Response:

```
302 Location: https://accounts.google.com/o/oauth2/v2/auth?…
```

**`GET /auth/google/callback`** · auth: none  
Google redirects here (outside /api/v1); signs in or creates the account, sets the session cookie.

Request:

```
`?code=…&state=…`
```
Response:

```
302 Location: /<next> (or /login?error=<code>)
```

### Account

**`GET /api/v1/me`** · auth: session  
The signed-in user.

Response:

```
200 {"id":"7b6c…","email":"ana@example.com","name":"Ana","avatarUrl":null,"emailVerified":true,"createdAt":"2026-10-22T10:00:00Z"}
```

**`PATCH /api/v1/me`** · auth: session  
Change name, email, language or password. A new password, or a new email on an account with a password, needs currentPassword. A new email starts unverified and gets a verification link; email changes share the resend limits (1 per minute, 10 per day) and are capped at 5 per day (429 with `Retry-After`).

Request:

```
{"name":"Ana B.","email":"ana@new.example","currentPassword":"…"}
```
Response:

```
200 {"id":"7b6c…","email":"ana@example.com","name":"Ana","avatarUrl":null,"emailVerified":true,"createdAt":"2026-10-22T10:00:00Z"}
```

**`PUT /api/v1/me/avatar`** · auth: session  
Upload an avatar: the raw PNG, JPEG or WebP as the body (max 2 MB). Stored as a 256x256 WebP in S3-compatible storage (`S3_ENDPOINT`, `S3_REGION`, `S3_BUCKET`, `S3_ACCESS_KEY_ID`, `S3_SECRET_ACCESS_KEY`, `S3_FORCE_PATH_STYLE`); the previous one is deleted. 422 for other types, 413 when too large.

Request:

```
<image bytes>
```
Response:

```
200 {"id":"7b6c…","email":"ana@example.com","name":"Ana","avatarUrl":null,"emailVerified":true,"createdAt":"2026-10-22T10:00:00Z"}
```

**`DELETE /api/v1/me/avatar`** · auth: session  
Remove the uploaded avatar (a Google profile picture shows again if there is one).

Response:

```
200 {"id":"7b6c…","email":"ana@example.com","name":"Ana","avatarUrl":null,"emailVerified":true,"createdAt":"2026-10-22T10:00:00Z"}
```

**`GET /api/v1/users/{id}/avatar`** · auth: none  
An uploaded avatar (image/webp). With `?v=<etag>` it is cached for a year; sends an ETag and answers If-None-Match with 304.

Response:

```
200 image/webp
```

**`GET /api/v1/me/sessions`** · auth: session  
List your sessions.

Response:

```
200 {"items":[{"id":"…","createdAt":"…","lastSeenAt":"…","expiresAt":"…","userAgent":"Firefox","ip":"203.0.113.5","current":true}]}
```

**`DELETE /api/v1/me/sessions/{id}`** · auth: session  
Sign out one session.

Response:

```
204 No Content
```

**`GET /api/v1/me/invites`** · auth: session  
Pending invites for your email.

Response:

```
200 {"items":[{"id":"c1e2…","projectId":"3f2a…","email":"bo@example.com","role":"editor","status":"pending","createdAt":"…","expiresAt":"…"}],"emailVerified":true}
```

**`POST /api/v1/me/invites/{id}/accept`** · auth: session  
Accept an invite addressed to you.

Response:

```
200 {"projectId":"3f2a…","role":"editor"}
```

**`POST /api/v1/me/invites/{id}/decline`** · auth: session  
Decline an invite addressed to you.

Response:

```
204 No Content
```

**`DELETE /api/v1/me`** · auth: session  
Delete your account.

Response:

```
204 No Content
```

**`GET /api/v1/me/identities`** · auth: session  
Linked sign-in identities (e.g. Google).

Response:

```
200 {"items":[{"provider":"google","email":"ana@example.com"}]}
```

**`DELETE /api/v1/me/identities/{provider}`** · auth: session  
Unlink an identity.

Response:

```
204 No Content
```

### Projects

**`GET /api/v1/projects`** · auth: optional  
List projects. `?filter=` (`all` default; `public` lists public projects signed out), `?limit=`, `?offset=`.

Response:

```
200 {"items":[{"id":"3f2a…","name":"Online Store","slug":"online-store","description":"","visibility":"private","template":false,"ownerId":"7b6c…","ownerName":"Ana","role":"owner","createdAt":"…","updatedAt":"…"}],"limit":50,"offset":0}
```

**`POST /api/v1/projects`** · auth: session  
Create a project.

Request:

```
{"name":"Online Store","description":"","visibility":"private"}
```
Response:

```
201 {"id":"3f2a…","name":"Online Store","slug":"online-store","description":"","visibility":"private","template":false,"ownerId":"7b6c…","ownerName":"Ana","role":"owner","createdAt":"…","updatedAt":"…"}
```

**`POST /api/v1/projects/import`** · auth: session  
Import a .ddrproj.zip (raw application/zip body or multipart field `file`). `?name=` overrides the name.

Request:

```
<zip bytes>
```
Response:

```
201 {"id":"3f2a…","name":"Online Store","slug":"online-store","description":"","visibility":"private","template":false,"ownerId":"7b6c…","ownerName":"Ana","role":"owner","createdAt":"…","updatedAt":"…"}
```

**`GET /api/v1/projects/{pid}`** · auth: optional  
One project (members, link holders, or public).

Response:

```
200 {"id":"3f2a…","name":"Online Store","slug":"online-store","description":"","visibility":"private","template":false,"ownerId":"7b6c…","ownerName":"Ana","role":"owner","createdAt":"…","updatedAt":"…"}
```

**`PATCH /api/v1/projects/{pid}`** · auth: session  
Rename, describe, change visibility or template flag.

Request:

```
{"visibility":"public"}
```
Response:

```
200 {"id":"3f2a…","name":"Online Store","slug":"online-store","description":"","visibility":"private","template":false,"ownerId":"7b6c…","ownerName":"Ana","role":"owner","createdAt":"…","updatedAt":"…"}
```

**`DELETE /api/v1/projects/{pid}`** · auth: session  
Delete a project (owner).

Response:

```
204 No Content
```

**`POST /api/v1/projects/{pid}/duplicate`** · auth: session  
Copy a project you can view into your account.

Request:

```
{"name":"Online Store copy"}
```
Response:

```
201 {"id":"3f2a…","name":"Online Store","slug":"online-store","description":"","visibility":"private","template":false,"ownerId":"7b6c…","ownerName":"Ana","role":"owner","createdAt":"…","updatedAt":"…"}
```

**`GET /api/v1/projects/{pid}/export.zip`** · auth: optional  
Download the project as .ddrproj.zip.

Response:

```
200 application/zip
```

**`GET /api/v1/p/{slug}`** · auth: none  
A public project and its files by slug.

Response:

```
200 {"project":{"id":"3f2a…","name":"Online Store","slug":"online-store","description":"","visibility":"private","template":false,"ownerId":"7b6c…","ownerName":"Ana","role":"owner","createdAt":"…","updatedAt":"…"},"files":[{"id":"91d0…","seq":4,"path":"diagrams/flow.ddr","kind":"ddr","version":1,"etag":"\"v1\"","size":64,"createdAt":"…","updatedAt":"…"}]}
```

### Files

**`GET /api/v1/projects/{pid}/files`** · auth: optional  
List files.

Response:

```
200 {"items":[{"id":"91d0…","seq":4,"path":"diagrams/flow.ddr","kind":"ddr","version":1,"etag":"\"v1\"","size":64,"createdAt":"…","updatedAt":"…"}]}
```

**`POST /api/v1/projects/{pid}/files`** · auth: session  
Create a file (`kind` ddr or tdr, inferred from the path).

Request:

```
{"path":"diagrams/flow.ddr","content":"Type: Flowchart\nTitle: Flow\n"}
```
Response:

```
201 {"id":"91d0…","seq":4,"path":"diagrams/flow.ddr","kind":"ddr","version":1,"etag":"\"v1\"","size":64,"createdAt":"…","updatedAt":"…"}
```

**`GET /api/v1/projects/{pid}/files/{fid}`** · auth: optional  
File metadata. Sends an ETag.

Response:

```
200 {"id":"91d0…","seq":4,"path":"diagrams/flow.ddr","kind":"ddr","version":1,"etag":"\"v1\"","size":64,"createdAt":"…","updatedAt":"…"}
```

**`PATCH /api/v1/projects/{pid}/files/{fid}`** · auth: session  
Rename or replace content. Send `If-Match` or `version`; a stale version is 412.

Request:

```
{"path":"diagrams/checkout.ddr","version":1}
```
Response:

```
200 {"id":"91d0…","seq":4,"path":"diagrams/flow.ddr","kind":"ddr","version":1,"etag":"\"v1\"","size":64,"createdAt":"…","updatedAt":"…"}
```

**`DELETE /api/v1/projects/{pid}/files/{fid}`** · auth: session  
Delete a file.

Response:

```
204 No Content
```

**`GET /api/v1/projects/{pid}/files/{fid}/text`** · auth: optional  
The file text (text/plain, ETag; If-None-Match gives 304).

Response:

```
200 text/plain
Type: Flowchart
Title: Flow
```

**`PUT /api/v1/projects/{pid}/files/{fid}/text`** · auth: session  
Replace the text (text/plain body). `If-Match: "v1"`; `?reset=true` drops the live Yjs state.

Request:

```
Type: Flowchart
Title: Flow
```
Response:

```
200 {"id":"91d0…","seq":4,"path":"diagrams/flow.ddr","kind":"ddr","version":1,"etag":"\"v1\"","size":64,"createdAt":"…","updatedAt":"…"}
```

### Sharing

**`GET /api/v1/projects/{pid}/members`** · auth: session  
List members.

Response:

```
200 {"items":[{"userId":"…","email":"bo@example.com","name":"Bo","avatarUrl":null,"role":"editor","addedAt":"…"}]}
```

**`POST /api/v1/projects/{pid}/members`** · auth: session  
Invite by email (same as POST invites).

Request:

```
{"email":"bo@example.com","role":"editor"}
```
Response:

```
201 {"id":"c1e2…","projectId":"3f2a…","email":"bo@example.com","role":"editor","status":"pending","createdAt":"…","expiresAt":"…"}
```

**`PATCH /api/v1/projects/{pid}/members/{uid}`** · auth: session  
Change a role; `owner` transfers ownership.

Request:

```
{"role":"viewer"}
```
Response:

```
200 {"userId":"…","role":"viewer"}
```

**`DELETE /api/v1/projects/{pid}/members/{uid}`** · auth: session  
Remove a member (or leave).

Response:

```
204 No Content
```

**`GET /api/v1/projects/{pid}/invites`** · auth: session  
List invites.

Response:

```
200 {"items":[{"id":"c1e2…","projectId":"3f2a…","email":"bo@example.com","role":"editor","status":"pending","createdAt":"…","expiresAt":"…"}]}
```

**`POST /api/v1/projects/{pid}/invites`** · auth: session  
Invite by email.

Request:

```
{"email":"bo@example.com","role":"editor"}
```
Response:

```
201 {"id":"c1e2…","projectId":"3f2a…","email":"bo@example.com","role":"editor","status":"pending","createdAt":"…","expiresAt":"…"}
```

**`DELETE /api/v1/invites/{id}`** · auth: session  
Revoke an invite.

Response:

```
204 No Content
```

**`POST /api/v1/invites/{id}/resend`** · auth: session  
Resend an invite mail.

Response:

```
200 {"id":"c1e2…","projectId":"3f2a…","email":"bo@example.com","role":"editor","status":"pending","createdAt":"…","expiresAt":"…"}
```

**`GET /api/v1/invites/lookup`** · auth: none  
Describe an invite by `?token=`.

Response:

```
200 {"projectName":"Online Store","inviterName":"Ana","email":"bo@example.com","role":"editor","status":"pending"}
```

**`GET /api/v1/invites/{token}`** · auth: none  
Same as lookup, token in the path.

Response:

```
200 {"projectName":"Online Store","inviterName":"Ana","email":"bo@example.com","role":"editor","status":"pending"}
```

**`POST /api/v1/invites/accept`** · auth: session  
Accept an invite by token.

Request:

```
{"token":"i_abc…"}
```
Response:

```
200 {"projectId":"3f2a…","role":"editor"}
```

**`POST /api/v1/invites/decline`** · auth: none  
Decline an invite by token.

Request:

```
{"token":"i_abc…"}
```
Response:

```
204 No Content
```

**`GET /api/v1/projects/{pid}/links`** · auth: session  
List share links.

Response:

```
200 {"items":[{"id":"5aa1…","projectId":"3f2a…","role":"viewer","token":"s_9x…","url":"https://example.com/s/s_9x…","createdAt":"…","expiresAt":null}]}
```

**`POST /api/v1/projects/{pid}/links`** · auth: session  
Create a share link. Editor links need `confirm: true`.

Request:

```
{"role":"viewer","expiresInHours":72}
```
Response:

```
201 {"id":"5aa1…","projectId":"3f2a…","role":"viewer","token":"s_9x…","url":"https://example.com/s/s_9x…","createdAt":"…","expiresAt":null}
```

**`DELETE /api/v1/projects/{pid}/links/{lid}`** · auth: session  
Revoke a link.

Response:

```
204 No Content
```

**`DELETE /api/v1/projects/{pid}/links`** · auth: session  
Revoke a link given as `?id=`.

Response:

```
204 No Content
```

**`GET /api/v1/links/lookup`** · auth: none  
Describe a share link by `?token=`.

Response:

```
200 {"projectId":"3f2a…","projectName":"Online Store","role":"viewer","expiresAt":null}
```

**`POST /api/v1/links/accept`** · auth: session  
Join a project through a share link.

Request:

```
{"token":"s_9x…"}
```
Response:

```
200 {"projectId":"3f2a…","role":"viewer"}
```

**`GET /api/v1/projects/{pid}/access-requests`** · auth: session  
List access requests (managers).

Response:

```
200 {"items":[{"id":"e4f5…","projectId":"3f2a…","userId":"8c9d…","message":"May I view this?","status":"pending","createdAt":"…"}]}
```

**`POST /api/v1/projects/{pid}/access-requests`** · auth: session  
Ask the owner for access.

Request:

```
{"message":"May I view this?"}
```
Response:

```
201 {"id":"e4f5…","projectId":"3f2a…","userId":"8c9d…","message":"May I view this?","status":"pending","createdAt":"…"}
```

**`PATCH /api/v1/access-requests/{id}`** · auth: session  
Approve or deny a request.

Request:

```
{"status":"approved","role":"viewer"}
```
Response:

```
200 {"id":"e4f5…","projectId":"3f2a…","userId":"8c9d…","message":"May I view this?","status":"pending","createdAt":"…"}
```

### Planned

**`POST /api/v1/projects/{pid}/assets`** · auth: session  
Upload an asset (planned).

Response:

```
501 {"type":"…/not_implemented","title":"Not Implemented","status":501,"detail":"not available in v0.0.1"}
```

**`GET /api/v1/assets/{aid}`** · auth: optional  
Read an asset (planned).

Response:

```
501 {"type":"…/not_implemented","title":"Not Implemented","status":501,"detail":"not available in v0.0.1"}
```

**`GET /api/v1/projects/{pid}/versions`** · auth: session  
Version history (planned).

Response:

```
501 {"type":"…/not_implemented","title":"Not Implemented","status":501,"detail":"not available in v0.0.1"}
```

**`POST /api/v1/projects/{pid}/versions`** · auth: session  
Save a version (planned).

Response:

```
501 {"type":"…/not_implemented","title":"Not Implemented","status":501,"detail":"not available in v0.0.1"}
```

**`POST /api/v1/projects/{pid}/versions/{vid}/restore`** · auth: session  
Restore a version (planned).

Response:

```
501 {"type":"…/not_implemented","title":"Not Implemented","status":501,"detail":"not available in v0.0.1"}
```

**`GET /api/v1/files/{fid}/threads`** · auth: session  
Comment threads (planned).

Response:

```
501 {"type":"…/not_implemented","title":"Not Implemented","status":501,"detail":"not available in v0.0.1"}
```

**`POST /api/v1/files/{fid}/threads`** · auth: session  
Start a thread (planned).

Response:

```
501 {"type":"…/not_implemented","title":"Not Implemented","status":501,"detail":"not available in v0.0.1"}
```

**`POST /api/v1/threads/{tid}/comments`** · auth: session  
Reply (planned).

Response:

```
501 {"type":"…/not_implemented","title":"Not Implemented","status":501,"detail":"not available in v0.0.1"}
```

**`PATCH /api/v1/threads/{tid}`** · auth: session  
Resolve a thread (planned).

Response:

```
501 {"type":"…/not_implemented","title":"Not Implemented","status":501,"detail":"not available in v0.0.1"}
```

**`POST /api/v1/projects/{pid}/export`** · auth: session  
Server-side render/export job (planned). Today rendering runs in the browser (WebAssembly), the `ddr render` CLI and the `render` MCP tool.

Response:

```
501 {"type":"…/not_implemented","title":"Not Implemented","status":501,"detail":"not available in v0.0.1"}
```

**`GET /api/v1/exports/{id}`** · auth: session  
Export job status (planned).

Response:

```
501 {"type":"…/not_implemented","title":"Not Implemented","status":501,"detail":"not available in v0.0.1"}
```

**`POST /api/v1/assist`** · auth: session  
AI assist (planned).

Response:

```
501 {"type":"…/not_implemented","title":"Not Implemented","status":501,"detail":"not available in v0.0.1"}
```

### Realtime

**`GET /ws/projects/{pid}`** · auth: session  
WebSocket for live editing (outside /api/v1). Binary frames `kind u8 | docRef uvarint | payload`: 0x01 subscribe to file (docRef = file seq), 0x02 unsubscribe, 0x10 y-protocols sync, 0x11 awareness, 0x12 checkpoint, 0x13 seq ack, 0x20 file tree (JSON), 0x30 presence (JSON), 0x40 events (JSON), 0x7E error, 0x7F ping.

Request:

```
GET with `Upgrade: websocket` and the session cookie
```
Response:

```
101 Switching Protocols
```

## Flowchart

<!-- type:flowchart -->
Classic flowchart (ISO 5807): terminators, process boxes, decision diamonds, input/output parallelograms.

- **Header:** `Type: Flowchart`
- **Accepted `Type:` values** (case-insensitive): `flowchart`, `flow`
- **File:** `.ddr`  
- **Views:** `flow`

Grammar:

```ebnf
file      = header blank "Flow" "{" { stmt } "}" ;
stmt      = step | start | end | if | loop | goto | edge ;
start     = "Start" [ label ] ;
end       = "End" [ label ] ;
step      = [ id ":" ] label { attrs } [ "->" ( attrs | target [ ":" label ] { attrs } ) ] ;
if        = "if" cond [ "->" attrs [ "->" attrs ] ] block { "elif" cond block } [ "else" block ] ;
loop      = "loop" [ cond ] block ;
goto      = "goto" id [ attrs ] ;
edge      = id "->" id [ ":" label ] { attrs } { "->" id [ ":" label ] { attrs } } ;
block     = "{" { stmt } "}" ;
```

| key | required | values | default | meaning |
|-----|----------|--------|---------|---------|
| `Flow` | **yes** | block | - | The one flow of the chart. |

- Write one `Flow { }` block. Statements end at a newline or `;`.
- `Start` and `End` are rounded terminators. Write `End` once, as the last statement: every branch that finishes flows into it (a later `End` with a different label gives W128 once that check lands).
- `if cond? { … } else { … }` draws a diamond with `Yes` / `No` exits. Rename them with clauses: `if ok? -> [label: Accepted] -> [label: Rejected] { … }`.
- `loop cond? { … }` repeats the block while the condition holds; `loop { … }` asks `Repeat?`.
- `goto id` jumps to the step with that id (`id: Label` declares one).
- A step whose first word is `Input`, `Output`, `Read`, `Print`, `Display`, `Show`, `Enter` or `Prompt` is drawn as a parallelogram; `[shape: rect]` turns that off for one step.
- `@Name` in a step links it to an actor (inside use cases).

Example (`docs/examples/flowchart.ddr`):

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

### Flowchart: Start/End labels and the one End

```ebnf
start = "Start" [ ":" ] [ label ] ;
end   = "End" [ ":" ] [ label ] ;
```

- `Start: Order received` and `Start Order received` are the same: the label of the start terminator. `End: label` likewise.
- A flowchart or graph draws **one** End. Every `End` (an early exit inside an `if`, the end of an `else`, the last statement) routes its own arrow into the first End written.
- A later `End` with a different label still joins that End, but its label is not drawn: W128. Give every `End` the same label, or none.

Example (`docs/examples/pending/flowchart-start-end-labels.ddr`):

```ddr
Type: Flowchart
Title: Order approval

Flow {
  Start: Order received;
  if In stock? {
    Ship order;
    End: Order closed;
  } else {
    Notify customer;
    End: Order closed;
  }
}
```

## Graph

<!-- type:graph -->
Free boxes-and-arrows graph: node statements and explicit `a -> b` edges.

- **Header:** `Type: Graph`
- **File:** `.ddr`  
- **Views:** `graph`

Grammar:

```ebnf
file  = header blank ( "Graph" | "Nodes" ) "{" { node | edge } "}" [ "Edges" "{" { edge } "}" ] ;
node  = [ id ":" ] label { attrs } ;
edge  = id "->" id [ ":" label ] { attrs } { "->" id [ ":" label ] { attrs } } ;
```

| key | required | values | default | meaning |
|-----|----------|--------|---------|---------|
| `Graph` | **yes** | block (alias `Nodes`) | - | Nodes and edges. |
| `Edges` | no | block | - | Extra edges added to the same graph. |

- Declare nodes as `id: Label [attrs]`; edges name ids: `api -> db: SQL`.
- Edges never create nodes: an unknown id is E203.
- Use `shape`, `color` and `[style: curve]` / `[dashed]` to style.

Example (`docs/examples/graph.ddr`):

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

## Architecture

<!-- type:architecture -->
Architecture diagrams are written as `Type: Graph` with shapes (`cylinder` for stores, `actor` for people, `round` for services) and colors per tier.

- **Header:** `Type: Graph`
- **File:** `.ddr`  
- **Views:** `graph`

Grammar:

```ebnf
Same as Graph.
```

- There is no separate `Type: Architecture`; it gives W107. Write `Type: Graph`.
- Suggested shapes: `actor` people, `round` / `pill` services, `cylinder` databases and queues, `box` external systems.
- Use `Direction: LR` for left-to-right tiers and `[dashed]` for asynchronous calls.

Example (`docs/examples/architecture.ddr`):

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

## Use case

<!-- type:usecase -->
UML use case diagram: actors outside a system boundary, use cases inside, include / extend / generalization.

- **Header:** `Type: UseCase`
- **Accepted `Type:` values** (case-insensitive): `usecase`, `use case`
- **File:** `.ddr`  
- **Views:** `overview`, plus `flow: <use case>` per use case with a `Flow`

Grammar:

```ebnf
file      = header blank "Actors:" { actor } "Use Cases:" { case | group } ;
actor     = "-" name { attrs } [ "->" name "[head: triangle]" ] ;
case      = "-" actors ":" name { attrs } [ "->" actor { attrs } ] { field } ;
group     = "-" actors [ ":" ] "{" { name { attrs } [ "{" fields "}" ] ( ";" | NL ) } "}" ;
actors    = name { "," name } ;
field     = key ":" text | ( "Flow" | "Main Flow" | "Alternative" [ N ] | "Exceptions" ) "{" { stmt } "}" ;
key       = "Description" | "Includes" | "Extends" | "Precondition" | "Postcondition" | "Actors" | ... ;
```

| key | required | values | default | meaning |
|-----|----------|--------|---------|---------|
| `Actors` | **yes** | list | - | One actor per `- ` line. Flags: `[system]`, `[external]`, `[primary]`, `[secondary]`; `[side: left\|right]`. |
| `Use Cases` | **yes** | list | - | `- Actor, Actor: Use case` with indented fields. |
| `Numbering` | no | prefix, e.g. `UC` or `UC-` | off | Gives each main case a code UC01, UC02, … (sub-cases UC03.1). |
| `Show Codes` | no | `yes` `no` | no | Print the code above the name. |
| `Min Use Cases` | no | number | 6 | With `Numbering:`, fewer main cases warns W120. |

- Fields of a case (indented under it): `Description`, `Includes`, `Extends`, `Precondition`, `Postcondition`, `Flow { }`, `Main Flow { }`, `Alternative N { }`, `Exceptions { }`.
- `Includes: A, B` draws dashed `«include»` arrows; `Extends: Base` draws `«extend»` toward the base.
- Generalization: `- Child -> Parent [head: triangle]` under Actors or Use Cases.
- Group cases that share actors: `- Customer { Browse; Search }`.
- `[code: UC07]` sets a code by hand; `[spec: none]` leaves a case out of spec tables.
- Specification keys also have Indonesian aliases (`Deskripsi`, `Alur Dasar`, `Alur Alternatif N`, `Kebutuhan Spesifik`, …) for teams that write specs in Indonesian.

Example (`docs/examples/usecase.ddr`):

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

## Activity

<!-- type:activity -->
UML activity diagram with swimlanes, decisions, merges, fork/join bars and one implicit start and end.

- **Header:** `Type: Activity`
- **Accepted `Type:` values** (case-insensitive): `activity`, `activity diagram`
- **File:** `.ddr`  
- **Views:** `activity`

Grammar:

```ebnf
file      = header blank ( "Flow" | "Activity" ) "{" { stmt } "}" ;
stmt      = [ lane ":" ] ( step | if | loop | parallel | goto | "start" | "end" | "stop" | edge ) ;
step      = [ id ":" ] label { attrs } ;
if        = "if" cond [ "[yes:" text "," "no:" text "]" ] block { "elif" cond block } [ "else" block ] ;
loop      = "loop" [ cond ] block ;
parallel  = "parallel" block "and" block { "and" block } ;
```

| key | required | values | default | meaning |
|-----|----------|--------|---------|---------|
| `Lanes` | no | comma list | lanes in order of first use | Swimlane columns, left to right. A lane named in a step must be listed (E205) when `Lanes:` is given. |
| `Flow` | **yes** | block (alias `Activity`) | - | The activity. |

- `Lane: step` puts a statement in a lane; a statement without a prefix stays in the previous lane.
- Start and end are implicit: one initial node before the first statement and one activity final after the last. `end` / `stop` in a branch routes into that same final.
- `parallel { … } and { … }` draws a fork bar and a join bar.
- Layout is automatic: `[x, y]` and `[via: …]` are ignored (W110).

Example (`docs/examples/activity.ddr`):

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

### Language of generated labels (Lang)

```ebnf
lang = ( "Lang" | "Language" ) ":" ( "en" | "id" ) ;
```

- One header line sets the language of every generated label: activity guards (`[Yes]` / `[No]` or `[Ya]` / `[Tidak]`, `Repeat?` / `Ulangi?`) and use case specification labels and step text.
- Without a `Lang:` line the language is **English** (`en`).
- Any other value gives W129 and the default is used.
- Guards you write yourself (`[yes: valid, no: invalid]`) are never translated.

Example (`docs/examples/pending/activity-lang.ddr`):

```ddr
Type: Activity
Title: Leave request
Lang: en
Lanes: Employee, Manager

Flow {
  Employee: Submit leave request
  Manager: Review request
  if Approved? {
    Manager: Record approval
  } else {
    Employee: Revise request
  }
}
```

## Conceptual data model (CDM)

<!-- type:cdm -->
PowerDesigner-style conceptual model: entities, attributes with identifiers `<pi>` and mandatory `<M>`, named relationships with crow's feet.

- **Header:** `Type: CDM`
- **Accepted `Type:` values** (case-insensitive): `cdm`, `conceptual`, `conceptual data model`
- **File:** `.ddr`  
- **Views:** `cdm`

Grammar:

```ebnf
file      = header blank "Entities:" { entity } { ref } ;
entity    = "-" name ( "{" attr { ( ";" | NL ) attr } "}" | NL { INDENT attr NL } ) ;
attr      = name [ ":" type ] { attrs } ;           (* [identifier] [pi] [mandatory] [M] *)
ref       = "Ref:" end op end [ ":" name ] { attrs } { op end [ ":" name ] } ;
end       = entity_name [ "." attr_name ] ;
op        = "<" | ">" | "-" | "<>" ;
```

| key | required | values | default | meaning |
|-----|----------|--------|---------|---------|
| `Entities` | **yes** | list | - | One entity per `- ` item. |
| `Ref` | no | line (repeatable) | - | A relationship. Also as a `References:` list or a `Ref { }` block. |

- `<` one-to-many, `>` many-to-one, `-` one-to-one, `<>` many-to-many.
- `[identifier]` (or `[pi]`) marks `<pi>` and `<M>`; `[mandatory]` / `[M]` marks only `<M>`.
- SQL-ish types are shown conceptually: `varchar(100)` → `Variable characters (100)`, `int` → `Integer`, `double` → `Long float`.
- Do not repeat foreign keys in a CDM: `Ref: Customer < Order` is the link.
- Unnamed relationships are labeled `Relationship_1`, `Relationship_2`, …

Example (`docs/examples/cdm.ddr`):

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

## Physical data model (PDM)

<!-- type:pdm -->
Physical tables with SQL types and PK/FK columns; references are written between columns.

- **Header:** `Type: PDM`
- **Accepted `Type:` values** (case-insensitive): `pdm`, `physical`, `physical data model`
- **File:** `.ddr`  
- **Views:** `pdm`

Grammar:

```ebnf
file      = header blank ( "Tables:" | "Entities:" ) { table } { ref } ;
table     = "-" name ( "{" column { ( ";" | NL ) column } "}" | NL { INDENT column NL } ) ;
column    = name ":" sqltype { attrs } ;             (* [pk] [fk] [unique] [not null] [null] *)
ref       = "Ref:" table "." column op table "." column [ ":" name ] { attrs } ;
```

| key | required | values | default | meaning |
|-----|----------|--------|---------|---------|
| `Tables` | **yes** | list (alias `Entities`) | - | One table per `- ` item. |
| `Ref` | no | line (repeatable) | - | `Ref: a.col < b.col`. The many side (or the right side of `-`) is the FK. |

- Mark primary keys with `[pk]`; a column on the many side of a `Ref:` is shown as `FK` automatically.
- `[not null]`, `[null]`, `[unique]` are accepted and kept.
- Attributes on a ref line go last: `Ref: a.id < b.a_id: owns [via: (420,80)]`.

Example (`docs/examples/pdm.ddr`):

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

## Entity relationship (ERD, Chen)

<!-- type:erd -->
Chen notation: entities are rectangles, relations diamonds, attributes ovals; key `*` (bold, underlined), foreign key `**`, derived `#` (dashed), composite `name { ... }`.

- **Header:** `Type: ERD`
- **Accepted `Type:` values** (case-insensitive): `erd`, `er`, `entity relationship`, `chen`
- **File:** `.ddr`  
- **Views:** `erd`

Grammar:

```ebnf
file      = header blank "Entities:" { entity } "Relations:" { relation } ;
entity    = "-" name "{" attr { ";" attr } "}" ;
attr      = mark name [ "[x:" n "," "y:" n "]" ] [ "{" attr { ";" attr } "}" ]
          | name [ "*" | "**" ] [ "#" ] [ "[x:" n "," "y:" n "]" ] [ "{" attr { ";" attr } "}" ] ;
mark      = "*" | "**" ;
relation  = "-" name "{" member { ";" member } "}" ;
member    = entity_name [ ":" card ] | attr ;          (* card: 1, M, N, 0..1, ... *)
```

| key | required | values | default | meaning |
|-----|----------|--------|---------|---------|
| `Entities` | **yes** | list | - | `- Name { Attr *; Attr; Ref ** }`. |
| `Relations` | **yes** | list | - | `- Verb { EntityA: 1; EntityB: N; attribute }`. |

- `Name *` is a key attribute, drawn bold and underlined with no star; `*Name` also works.
- `Name **` is a foreign key: accepted and kept, drawn as a plain attribute for now.
- `Name #` is a derived attribute, drawn with a dashed outline. Only a trailing ` #` counts; a line that starts with `#` is still a comment.
- `Name { A; B }` is a composite attribute: A and B hang off Name, and they may nest again.
- In a relation, a line naming an entity joins it; `Entity: card` also prints the cardinality.
- Any other line in a relation is an attribute of the relation.
- A relation may join three or more entities; a self relation names the same entity twice.
- Do not write chains (`A -> B`) inside a relation (W124).

Example (`docs/examples/erd.ddr`):

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

### ERD: ternary relationships

A relation joins every entity it names, so a ternary (or wider) relationship is one diamond with three member lines:

```ebnf
relation = "-" name "{" member ";" member ";" member { ";" ( member | attr ) } "}" ;
```

- Each entity gets its own line from the diamond, with its cardinality beside the entity.
- Lines that are not entity names (`Term`) are attributes of the relation.
- A relation naming fewer than two entities gives W124.

Example (`docs/examples/erd-ternary.ddr`):

```ddr
Type: ERD
Title: Course enrollment

Entities:
- Student { Student ID *; Name }
- Course { Course ID *; Title }
- Teacher { Teacher ID *; Name }

Relations:
- Enrolls { Student: N; Course: M; Teacher: 1; Term }
```

### ERD: key, derived and composite attributes

Attribute marks and nesting:

```ebnf
attr = name [ "*" | "**" ] [ "#" ] [ "{" attr { ";" attr } "}" ] ;
```

- `User ID *`: key, bold and underlined (W125 when an entity has none).
- `User ID **`: foreign key, no mark drawn yet.
- `Age #`: derived, dashed oval.
- `Address { Country; City }`: composite; its children are drawn farther out on the same side, any depth.

Example (`docs/examples/erd-attributes.ddr`):

```ddr
Type: ERD
Title: Users and Orders

# key `*`, foreign key `**`, derived `#`, composite `{ }`
Entities:
- User {
    User ID *
    Name
    Age #
    Address {
      Country
      City
      District
    }
  }
- Order { *Order ID; Order Date; User ID **; Total # }

Relations:
- Places { User: 1; Order: N }
```

### ERD: optional cardinality

The cardinality after `Entity:` is optional:

```ebnf
member = entity_name [ ":" card ] ;   (* card: 1 | M | N | 0..1 | 1..N | ... ; free text *)
```

- `Employee: 0..1` prints `0..1` beside Employee.
- `Employee` alone joins the entity and prints nothing (no default cardinality is assumed).

Example (`docs/examples/erd-optional-card.ddr`):

```ddr
Type: ERD
Title: Employees and parking spots

Entities:
- Employee { Employee ID *; Name }
- Parking Spot { Spot ID *; Level }
- Department { Department ID *; Name }

Relations:
- Uses { Employee: 0..1; Parking Spot: 0..1 }
- Works In { Employee; Department }
```

## Documents (.tdr) and use case specifications

<!-- type:tdr -->
Markdown documents with a header and directives that embed live diagrams and use case specification tables.

- **Header:** none (a `.tdr` file)
- **File:** `.tdr`  
- **Views:** page preview

Grammar:

```ebnf
file      = [ { key ":" value NL } "---" NL ] markdown ;
directive = "::diagram[" path "]" [ "{" "view=" q [ "caption=" q ] "}" ]
          | "::usecases[" path "]" [ "{" "title=" q "}" ] ;
```

| key | required | values | default | meaning |
|-----|----------|--------|---------|---------|
| `Title` | no | text | - | Title block. |
| `Subtitle` | no | text | - | Title block. |
| `Author` | no | text | - | Title block. |
| `Date` | no | text | - | Title block. |

- The body is CommonMark + GFM tables; callouts use `> [!NOTE]`.
- `::diagram[path]{view="overview" caption="…"}` embeds a live figure; `view` defaults to the first view.
- `::usecases[path]{title="…"}` prints one specification table per use case. The CLI prints the same tables: `ddr spec -format html|md|json file.ddr`.
- An unknown file or view shows an inline error box instead of failing the page.
- `ddr check` and the MCP `check` tool are for `.ddr` files only: given a `.tdr` document they report only E108 (a file named `.tdr`, or a file without a `.ddr` extension whose header has no `Type:` and ends with `---`). Check each embedded `.ddr` file instead.

Example (`docs/examples/document.tdr`):

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
<!-- end-types -->

