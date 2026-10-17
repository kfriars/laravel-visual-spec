# What Laravel Brain actually does

Research for [#5](https://github.com/kfriars/laravel-visual-spec/issues/5). Researched 2026-10-17. This feeds the decision on whether a new package is warranted.

## Answer in brief

Laravel Brain (`laramint/laravel-brain`) is a `require-dev` Laravel package. It parses the app's PHP with `nikic/php-parser`, starts from each entry point (routes, commands, schedules, channels, Filament pages, `laravel/ai` agents), and traces method calls forward. The result is a graph of method-level nodes, which it stores as JSON (one file per tab), shows in a local React viewer at `/_laravel-brain`, and exposes to agents through context exports, generated agent files and an optional MCP server.

It describes **what the code reaches from its entry points**. It has no concept of intended structure: no layers, rules, allowed or forbidden edges, contracts, plans or violations. It also does not observe runtime behaviour. Apart from a few optional live reads (the router, the database catalogue, git), it is static analysis.

So nothing in Brain makes laravel-visual-spec redundant, and Brain's graph cannot stand in for our scanner. It only contains classes that some entry point reaches, which by Brain's own measurement leaves out 45–86% of `app/`. It also assigns kinds such as "controller" or "repository" by matching class names. Brain is a complement: agents can ask it flow questions while they write a plan. Several of its ideas are worth borrowing. Reusing its code is held back by a missing LICENSE file, the lack of a public API, and a single maintainer.

Of the eleven WIP claims checked, six are true and five are only partly true. None is false outright. The WIP is wrong in one place that matters: it calls what Brain describes "runtime flow". It is static flow from entry points.

## Sources and method

- **Repository:** [github.com/laramint/laravel-brain](https://github.com/laramint/laravel-brain). It exists under that name and has not moved. Its default branch is `v2.x`. All source citations below are at commit [`07df09a`](https://github.com/laramint/laravel-brain/tree/07df09a40269db5bdac7ec242141bb825eae4e51), the `v2.x` head from 2026-09-18. Its source is identical to tag `v2.7.1` (`7c0e508`); the two differ by one line in `.github/workflows/pr-template.yml`. Line numbers refer to that commit.
- **Packagist:** [packagist.org/packages/laramint/laravel-brain](https://packagist.org/packages/laramint/laravel-brain), read through its JSON API on 2026-10-17.
- **GitHub metadata** (stars, issues, PRs, releases, contributors, licence API) was read with `gh` on 2026-10-17.
- The repository was cloned and read. None of its code was run.

Below, `B/` is short for `https://github.com/laramint/laravel-brain/blob/07df09a40269db5bdac7ec242141bb825eae4e51/`.

## How it works

### Pipeline

`php artisan brain:scan` runs a chain of analyzers over the project, all of them reading source files ([README.md L263–284](https://github.com/laramint/laravel-brain/blob/07df09a40269db5bdac7ec242141bb825eae4e51/README.md#L263-L284)):

- `RouteAnalyzer`, `MiddlewareAnalyzer` and `ControllerAnalyzer` read route files, middleware configuration and controllers.
- `MethodTracer` traces call chains from each entry method. It follows constructor-injected properties, typed variables, `new`, static calls, and job and event dispatches.
- `ModelAnalyzer`, `ConsoleAnalyzer`, `ChannelAnalyzer`, `FilamentAnalyzer`, `AiAnalyzer`, `ContainerBindingAnalyzer`, `FacadeAnalyzer` and about 30 other analyzers add the rest.
- `GraphBuilder` assembles the nodes and edges. `GraphSplitter` cuts them into per-entry-point "tabs".

Parsing uses `nikic/php-parser` ^5 ([composer.json L8](https://github.com/laramint/laravel-brain/blob/07df09a40269db5bdac7ec242141bb825eae4e51/composer.json#L8)) with `NameResolver` in additive mode ([src/Parser/PhpFileParser.php L11, L49, L151](https://github.com/laramint/laravel-brain/blob/07df09a40269db5bdac7ec242141bb825eae4e51/src/Parser/PhpFileParser.php#L151)). The package has about 34,600 lines of PHP under `src/` and 105 test files.

### Data model

- **Node:** `{id, type, label, data}`, where `data` is a free-form array ([src/Graph/Node.php L9–14](https://github.com/laramint/laravel-brain/blob/07df09a40269db5bdac7ec242141bb825eae4e51/src/Graph/Node.php#L9-L14)). It carries things like the FQCN, method, file, flowchart steps, cyclomatic complexity, DB queries, cache operations, outgoing HTTP calls and security exposure.
- **Edge:** `{id, source, target, label, type}` ([src/Graph/Edge.php L9–15](https://github.com/laramint/laravel-brain/blob/07df09a40269db5bdac7ec242141bb825eae4e51/src/Graph/Edge.php#L9-L15)). The id is an `xxh128` hash of source, target, type and label, so it stays stable as unrelated edges come and go ([src/Graph/GraphBuilder.php L2896–2919](https://github.com/laramint/laravel-brain/blob/07df09a40269db5bdac7ec242141bb825eae4e51/src/Graph/GraphBuilder.php#L2896-L2919)).
- **Granularity is mostly per method.** A call-chain hop gets the id `fqcn::method`, and controller actions get `action::…` ids ([GraphBuilder.php L1221–1239](https://github.com/laramint/laravel-brain/blob/07df09a40269db5bdac7ec242141bb825eae4e51/src/Graph/GraphBuilder.php#L1221-L1239)). A service with three traced methods therefore appears as three nodes. Models, events, enums, views, controllers and similar kinds are class-level.
- **Kinds come from name heuristics, not from anything the app declares.**
  - Anything whose FQCN contains `Controller`, `\Http\` or `\Livewire\` is treated as a controller.
  - `Repository` means repository, `Job` means job and `Event` means event.
  - `\Models\` means model.
  - Everything else is a "service".

  See [GraphBuilder.php L1241–1294](https://github.com/laramint/laravel-brain/blob/07df09a40269db5bdac7ec242141bb825eae4e51/src/Graph/GraphBuilder.php#L1241-L1294) and the same logic in [src/Analysis/MethodTracer.php L1150–1182](https://github.com/laramint/laravel-brain/blob/07df09a40269db5bdac7ec242141bb825eae4e51/src/Analysis/MethodTracer.php#L1150-L1182). Interface, enum, trait and abstract-class kinds are read from the declaration. The only kind that can be configured is the `actions` path.
- **Edge types** number about 50. Some examples: `route-to-controller`, `controller-to-action`, `route-to-middleware`, `action-to-{service,model,job,event,…}`, `event-to-listener`, `model-relationship`, `binding-registered-in`, `binding-resolution`, `facade-resolves-to`, `inherits-method`, `controller-to-interface` (`type-hint`), `view-to-view`, `schedule-to-command`, `event-to-channel`, and the `filament-*` and `ai-*` families. They are all emitted through `addEdge` call sites in [GraphBuilder.php](https://github.com/laramint/laravel-brain/blob/07df09a40269db5bdac7ec242141bb825eae4e51/src/Graph/GraphBuilder.php#L622).
- **Persistence:**
  - The scan writes a manifest (`graphFormatVersion: 2`, [src/Graph/GraphSplitter.php L798](https://github.com/laramint/laravel-brain/blob/07df09a40269db5bdac7ec242141bb825eae4e51/src/Graph/GraphSplitter.php#L798)) and one subgraph per tab.
  - Each tab is "a forward-only slice of the lifecycle".
  - No single combined graph is saved; consumers rebuild one by taking the union of the tabs ([src/Ai/MergedGraph.php L10–17](https://github.com/laramint/laravel-brain/blob/07df09a40269db5bdac7ec242141bb825eae4e51/src/Ai/MergedGraph.php#L10-L17)). A node that falls outside every tab is therefore not persisted.

### How edges are extracted: statically, with a few live reads

**Static, and only from entry points.** The graph grows forward from entry points, so classes that nothing traced reaches are simply absent. Brain's source says so directly. Most of `app/` owns nothing in the graph, "45% to 86% of it across the applications measured — because the graph only holds what an entry point reaches" ([src/Analysis/ProjectAnalyzer.php L355–357](https://github.com/laramint/laravel-brain/blob/07df09a40269db5bdac7ec242141bb825eae4e51/src/Analysis/ProjectAnalyzer.php#L355-L357)). On one app the graph held 45 of 211 event classes and 27 of 113 job classes ([src/Analysis/Reachability/ReachabilityAnalyzer.php L13–17](https://github.com/laramint/laravel-brain/blob/07df09a40269db5bdac7ec242141bb825eae4e51/src/Analysis/Reachability/ReachabilityAnalyzer.php#L13-L17)). On a large modular app, 79–90% of declared classes came back unreached ([README.md L538](https://github.com/laramint/laravel-brain/blob/07df09a40269db5bdac7ec242141bb825eae4e51/README.md#L538)).

**Unresolvable receivers are dropped without a trace.** When the tracer can't resolve the type of the variable a method is called on, it returns and no edge is recorded ([MethodTracer.php L819–822](https://github.com/laramint/laravel-brain/blob/07df09a40269db5bdac7ec242141bb825eae4e51/src/Analysis/MethodTracer.php#L819-L822), [`resolveVar` L1072–1095](https://github.com/laramint/laravel-brain/blob/07df09a40269db5bdac7ec242141bb825eae4e51/src/Analysis/MethodTracer.php#L1072-L1095)). That case includes `app(Foo::class)->bar()`: there is no handling of `app()` or `resolve()` anywhere in the tracer. Framework classes (`Illuminate\`, `Laravel\`) are skipped on purpose ([L1184–1189](https://github.com/laramint/laravel-brain/blob/07df09a40269db5bdac7ec242141bb825eae4e51/src/Analysis/MethodTracer.php#L1184-L1189)).

**Live and runtime inputs.** These are all optional, and none of them traces a request:

- `auto_discover_routes` reads the booted router through `app('router')->getRoutes()`. It is off by default ([src/Analysis/RouteAnalyzer.php L740–750](https://github.com/laramint/laravel-brain/blob/07df09a40269db5bdac7ec242141bb825eae4e51/src/Analysis/RouteAnalyzer.php#L740-L750)).
- Table statistics and schema are read from the live database ([src/Analysis/SchemaInspector.php L123, L156](https://github.com/laramint/laravel-brain/blob/07df09a40269db5bdac7ec242141bb825eae4e51/src/Analysis/SchemaInspector.php#L123)).
- Git churn comes from running `git` ([src/Analysis/GitChurnAnalyzer.php L96](https://github.com/laramint/laravel-brain/blob/07df09a40269db5bdac7ec242141bb825eae4e51/src/Analysis/GitChurnAnalyzer.php#L96)).
- The route stress test sends real HTTP requests through `laramint/laravel-stress`.

**Verdict:** edges are extracted statically. Brain never executes application code paths to discover them.

## What it can say about architecture, as opposed to runtime flow

**What it can say:**

- what a given route, command or job reaches, and in what order (sequence diagrams per route, [README.md L68](https://github.com/laramint/laravel-brain/blob/07df09a40269db5bdac7ec242141bb825eae4e51/README.md#L68));
- who directly calls a node, one hop back, inside the scanned graph ([src/Ai/UsageFinder.php L10–14](https://github.com/laramint/laravel-brain/blob/07df09a40269db5bdac7ec242141bb825eae4e51/src/Ai/UsageFinder.php#L10-L14));
- which concrete class sits behind an interface or a facade;
- model relationships, observers and policies;
- complexity and churn hotspots;
- how exposed each route is from a security point of view;
- which classes nothing reaches (with the optional reachability pass).

**What it cannot say:**

- **Anything about intent.** A search of `src/`, `frontend/src/`, the README and the docs finds no concept of layers, rules, allowed or forbidden dependencies, violations or plans. Brain records what exists. It has nothing to compare it against.
- **Whether a plan is valid.** It takes no plan as input and works only on code that already exists.
- **A complete class-level dependency graph.** Unreached classes are missing, nodes are per method, kinds are guessed from names, and unresolvable receivers leave no edge. An architecture check needs every class under `app/` with class-level edges.
- **Anything in CI.** The commands, routes and MCP server register only when `app()->isLocal()` is true, and that check is hard-coded ([src/LaravelBrainServiceProvider.php L47–50](https://github.com/laramint/laravel-brain/blob/07df09a40269db5bdac7ec242141bb825eae4e51/src/LaravelBrainServiceProvider.php#L47-L50)). `brain:scan` prints a text summary and has no machine-readable verdict or failing exit code for structural problems ([src/Commands/ScanCommand.php L466–509](https://github.com/laramint/laravel-brain/blob/07df09a40269db5bdac7ec242141bb825eae4e51/src/Commands/ScanCommand.php#L466-L509)).

Brain's own generated "Architecture" section is a list of node-type counts plus a fixed line, `Flow: Route → Middleware → Controller → Service → Model`. That line is written for every app, whatever its actual structure ([src/Ai/RulesExporter.php L161–166](https://github.com/laramint/laravel-brain/blob/07df09a40269db5bdac7ec242141bb825eae4e51/src/Ai/RulesExporter.php#L161-L166)).

## The WIP's claims, checked

These are the claims from `docs/wip-spec-2026-10-16.md`, "Lessons from Laravel Brain", plus the licence statement.

| # | WIP claim | Verdict | Evidence |
| --- | --- | --- | --- |
| 1 | Flags dispatches it can't resolve statically instead of showing no impact | **Partly true** | It flags **job** dispatches only: `dispatch()`, `dispatch_sync()`, configured helpers, `$this->dispatch()`, `Bus::dispatch()`, and `Bus::chain` or `batch` with members that aren't literals ([MethodTracer.php L74–80, L121–138, L474–477, L757–771, L867–888](https://github.com/laramint/laravel-brain/blob/07df09a40269db5bdac7ec242141bb825eae4e51/src/Analysis/MethodTracer.php#L867-L888)). `event($var)` is not flagged, despite README L38 saying "job/event". The flag is a per-method set that shows up only as an "Unresolved disp." count in the CLI summary ([ScanCommand.php L484–486](https://github.com/laramint/laravel-brain/blob/07df09a40269db5bdac7ec242141bb825eae4e51/src/Commands/ScanCommand.php#L484-L486)) and on the analysis result. It is never drawn in the graph or the viewer: neither `src/Graph` nor `frontend/src` references it. Other unresolvable calls, including `app()` lookups, are dropped silently (see above). Brain has no "unknown edge". |
| 2 | Colours unreached classes grey as a question, not a verdict | **True** | [frontend/src/utils/graphConstants.ts L96–98](https://github.com/laramint/laravel-brain/blob/07df09a40269db5bdac7ec242141bb825eae4e51/frontend/src/utils/graphConstants.ts#L96-L98) ("Deliberately grey … a question, not a verdict"), and README L570. Each unreached class carries the references Brain could see but not follow: container binding, facade, config, inherited, class-string ([ReachabilityAnalyzer.php L11–26, L45–76](https://github.com/laramint/laravel-brain/blob/07df09a40269db5bdac7ec242141bb825eae4e51/src/Analysis/Reachability/ReachabilityAnalyzer.php#L11-L26)). The pass is **off by default** ([config/laravel-brain.php L717–729](https://github.com/laramint/laravel-brain/blob/07df09a40269db5bdac7ec242141bb825eae4e51/config/laravel-brain.php#L717-L729)). |
| 3 | Reports isolated nodes and nodes outside any tab as separate counts | **True** | [ScanCommand.php L488–499](https://github.com/laramint/laravel-brain/blob/07df09a40269db5bdac7ec242141bb825eae4e51/src/Commands/ScanCommand.php#L488-L499), computed at [ProjectAnalyzer.php L932–933](https://github.com/laramint/laravel-brain/blob/07df09a40269db5bdac7ec242141bb825eae4e51/src/Analysis/ProjectAnalyzer.php#L932-L933). They appear in the CLI text summary only. |
| 4 | Exports deterministic, token-budgeted context for agents | **Partly true** | The budget is estimated at 4 characters per token ([src/Ai/ContextExporter.php L11, L201](https://github.com/laramint/laravel-brain/blob/07df09a40269db5bdac7ec242141bb825eae4e51/src/Ai/ContextExporter.php#L201)). The default is 6,000 tokens and the minimum is 500 ([src/Commands/ExportContextCommand.php L14](https://github.com/laramint/laravel-brain/blob/07df09a40269db5bdac7ec242141bb825eae4e51/src/Commands/ExportContextCommand.php#L14)). The traversal is breadth-first to depth 3 (L123–153). Structural sections are "always included" and only source snippets and package tables get trimmed (L314–316), so the budget is soft. Output is deterministic for a given scan, but it embeds that scan's timestamp (L205), so two scans of identical code give different bytes. |
| 5 | Writes the scan as JSON to `storage/` and serves the viewer through app routes, local only | **True** | The default store is `storage/app/laravel-brain/` ([src/Storage/GraphStoreFactory.php L22](https://github.com/laramint/laravel-brain/blob/07df09a40269db5bdac7ec242141bb825eae4e51/src/Storage/GraphStoreFactory.php#L22)), written as `.graph-manifest.json` plus one `.graph-{tab}.json` per tab. A `database` driver is the alternative. Routes live under `/_laravel-brain`, with a catch-all route that serves the React SPA and the JSON ([routes/brain.php L7–18](https://github.com/laramint/laravel-brain/blob/07df09a40269db5bdac7ec242141bb825eae4e51/routes/brain.php#L7-L18)). "Local only" is a hard-coded `isLocal()` check that also gates the commands, so there is no CI mode ([ServiceProvider L47–68](https://github.com/laramint/laravel-brain/blob/07df09a40269db5bdac7ec242141bb825eae4e51/src/LaravelBrainServiceProvider.php#L47-L68)). The routes have no authentication; the POST routes have only a same-origin check. |
| 6 | Watch mode re-traces only changed files and merges the result | **Partly true** | Watch mode polls file modification times ([ScanCommand.php L80–117](https://github.com/laramint/laravel-brain/blob/07df09a40269db5bdac7ec242141bb825eae4e51/src/Commands/ScanCommand.php#L80-L117)). It takes the scoped fast path only when three things hold: no file was added or deleted, every change is inside `source_paths`, and the changed files' own edges are unchanged. Any edit that adds or removes a call triggers a full rebuild ([src/Analysis/Incremental/IncrementalAnalyzer.php L9–24, L98–101](https://github.com/laramint/laravel-brain/blob/07df09a40269db5bdac7ec242141bb825eae4e51/src/Analysis/Incremental/IncrementalAnalyzer.php#L9-L24); [ProjectAnalyzer.php L770–784](https://github.com/laramint/laravel-brain/blob/07df09a40269db5bdac7ec242141bb825eae4e51/src/Analysis/ProjectAnalyzer.php#L770-L784)). So the incremental path covers exactly the edits that leave edges alone, which is the case an edge-oriented scanner cares about least. |
| 7 | Wires edges on interfaces through to the concrete class bound in the container | **True, within limits** | Brain parses service providers in `container_bindings.provider_paths` (default `app/Providers`) for `bind`, `singleton`, `scoped`, their `*If` variants and the `$bindings` property ([src/Analysis/ContainerBindingAnalyzer.php L20–35, L43, L51](https://github.com/laramint/laravel-brain/blob/07df09a40269db5bdac7ec242141bb825eae4e51/src/Analysis/ContainerBindingAnalyzer.php#L20-L51)). When a traced hop lands on an interface or abstract class, it adds `binding-registered-in` and `binding-resolution` edges ([GraphBuilder.php L1411–1471](https://github.com/laramint/laravel-brain/blob/07df09a40269db5bdac7ec242141bb825eae4e51/src/Graph/GraphBuilder.php#L1411-L1471), called at L745). Limits: a closure concrete gives no concrete class (ContainerBindingAnalyzer L202, L277); each abstract has one record, and the last one written wins ([ContainerBindingRegistry.php L9, L16–22](https://github.com/laramint/laravel-brain/blob/07df09a40269db5bdac7ec242141bb825eae4e51/src/Analysis/ContainerBindingRegistry.php#L9-L22)); contextual `when()->needs()->give()` bindings are not handled; and only interfaces some traced chain reaches get the edges. It also confirms the WIP's aside that Brain "parse[s] service providers statically". |
| 8 | Ships an MCP server through `laravel/mcp` | **True** | `laravel/mcp` is optional: it appears only under `suggest` and `require-dev` ([composer.json L19, L23](https://github.com/laramint/laravel-brain/blob/07df09a40269db5bdac7ec242141bb825eae4e51/composer.json#L19-L23)). Brain checks `class_exists` and a config flag, then calls `Mcp::local('brain', …)` ([ServiceProvider L79–90](https://github.com/laramint/laravel-brain/blob/07df09a40269db5bdac7ec242141bb825eae4e51/src/LaravelBrainServiceProvider.php#L79-L90)). The server has 10 tools (9 marked `IsReadOnly`, plus `rescan`) and 2 resources ([src/Mcp/BrainMcpServer.php L25–52](https://github.com/laramint/laravel-brain/blob/07df09a40269db5bdac7ec242141bb825eae4e51/src/Mcp/BrainMcpServer.php#L25-L52)). It needs Laravel 11+. The tools read the last saved scan, not the live code. |
| 9 | `brain:generate-rules` writes descriptive files such as `CLAUDE.md` and `AGENTS.md` | **True** | It has 8 targets: `CLAUDE.md`, `AGENTS.md`, `CODEX.md`, `CONVENTIONS.md`, `.github/copilot-instructions.md`, `.junie/guidelines.md`, `.cursor/rules/laravel-brain.mdc` and `.windsurf/rules/laravel-brain.md` ([RulesExporter.php L12–21](https://github.com/laramint/laravel-brain/blob/07df09a40269db5bdac7ec242141bb825eae4e51/src/Ai/RulesExporter.php#L12-L21)). Each file is **overwritten whole** with `file_put_contents`, after a confirmation prompt (or a 409 response) unless forced ([src/Commands/GenerateRulesCommand.php L65–82](https://github.com/laramint/laravel-brain/blob/07df09a40269db5bdac7ec242141bb825eae4e51/src/Commands/GenerateRulesCommand.php#L65-L82)). The browser can trigger the same write through `POST /_laravel-brain/api/generate-rules` ([src/Http/Controllers/BrainController.php L208–272](https://github.com/laramint/laravel-brain/blob/07df09a40269db5bdac7ec242141bb825eae4e51/src/Http/Controllers/BrainController.php#L208-L272)). The content is descriptive (stack, counts, top routes, hotspots, smells, packages), plus the fixed flow line noted above. Open upstream issue [#138](https://github.com/laramint/laravel-brain/issues/138) reports it overwriting Laravel Boost's guidelines in `CLAUDE.md`. |
| 10 | Brain describes runtime flow, read-only (system diagram, non-goals, "Where we differ") | **Partly true** | The flow part is true; the runtime part is not. Brain traces call flow statically (see "How edges are extracted"). It is also not read-only: a scan writes to `storage/`, `generate-rules` writes agent files, and the stress test sends HTTP requests. The questions the WIP says Brain answers are real: sequence diagrams per route ([README.md L68](https://github.com/laramint/laravel-brain/blob/07df09a40269db5bdac7ec242141bb825eae4e51/README.md#L68)) and finding callers ([UsageFinder.php L10–14](https://github.com/laramint/laravel-brain/blob/07df09a40269db5bdac7ec242141bb825eae4e51/src/Ai/UsageFinder.php#L10-L14)), the latter one hop back and only within the reached graph. |
| 11 | MIT-licensed; code reusable if we keep the licence notice | **Partly true** | MIT is *declared* in [composer.json L5](https://github.com/laramint/laravel-brain/blob/07df09a40269db5bdac7ec242141bb825eae4e51/composer.json#L5), on Packagist (v2.7.1 `license: ["MIT"]`), in the README badge and at L742, and in [docs/index.md L121–123](https://github.com/laramint/laravel-brain/blob/07df09a40269db5bdac7ec242141bb825eae4e51/docs/index.md#L121-L123). But the repo has **no LICENSE file**, in any branch or at any point in its history. README L742 links to a file that doesn't exist, GitHub's licence API returns 404, and `licenseInfo` is null. MIT's one condition is to keep "the above copyright notice and this permission notice", and the repo contains neither. Borrowing ideas is fine regardless, since ideas aren't covered by copyright. Copying code cleanly needs a licence text with a named copyright holder. |

## Maturity

These figures were taken on 2026-10-17.

- **Age and releases.** The repo and the `laramint` GitHub org were both created on 2026-04-29. Since then there have been 18 tags, from `v1.0.0` to `v2.7.1`, and 14 GitHub releases. The latest, `v2.7.1`, came out on 2026-09-18. The package went from v2.0 to v2.7 in about four months.
- **Activity.** Commits per month across all branches, by author date: April 21, May 98, June 18, July 27, August 116, September 162, October 0 so far. Nothing has been merged since 2026-09-18. Six PRs opened by SanderMuller on 2026-10-16 are waiting for review.
- **Maintainers.** Effectively one: `mrmarchone` (Abdelrahman Muhammed).
  - He is the only Packagist maintainer and the org's only public member.
  - He merged all 94 merged PRs.
  - GitHub contributor counts: mrmarchone 213, webard 67, SanderMuller 62, everyone else 6 or fewer.
  - Two outside contributors wrote most of the July–September analysis features: SanderMuller with 44 PRs, and webard with 35, including PRs #115–#135 on 2026-09-01.
- **Adoption.**
  - GitHub: 937 stars, 53 forks, 11 watchers.
  - Packagist: 393,692 installs in total, 91,578 in the last 30 days, 3,721 per day. These counts include CI installs, so read them as an upper bound.
  - Two Packagist dependents: `sandermuller/richter`, which describes itself as "built on Laravel Brain" (15k installs), and `pepperfm/laravel-starter-kit`.
- **Issues.** 3 open out of about 40. Most were closed within days of being opened.
- **Quality signals.**
  - CI runs PHP 8.2–8.5 against Laravel 9–13 ([.github/workflows/ci.yml](https://github.com/laramint/laravel-brain/blob/07df09a40269db5bdac7ec242141bb825eae4e51/.github/workflows/ci.yml)), even though `composer.json` claims PHP `>=8.0`.
  - The code is heavily commented with measured rationale, and there is a benchmark harness under `benchmark/`.
  - There are signs of fast-moving drift. The README promises a LICENSE file that doesn't exist, and names a cytoscape `cose-bilkent` layout although the viewer is React + D3 + dagre with no Cytoscape ([frontend/package.json](https://github.com/laramint/laravel-brain/blob/07df09a40269db5bdac7ec242141bb825eae4e51/frontend/package.json)). `generate-rules` says "7 files" and lists 7 targets in its signature while shipping 8 ([GenerateRulesCommand.php L14](https://github.com/laramint/laravel-brain/blob/07df09a40269db5bdac7ec242141bb825eae4e51/src/Commands/GenerateRulesCommand.php#L14)).

## Overlap and complement with the core bet

The core bet is two things: checking a **plan** against the **architecture spec** before the code exists, and letting humans review both visually.

| Part of the bet | Overlap with Brain |
| --- | --- |
| Plan validation against an architecture spec | **None.** Brain has no intent model and no plan input, and it only analyses code that exists. |
| Visual review | **Only in the medium.** Both are dev packages serving a local SPA graph. Brain's picture is a set of per-entry-point flow tabs with method-level nodes and name-guessed kinds, and it answers "what does this route do?". Ours is layers with allowed and forbidden edges, plan overlays and conformance, and it answers "is this allowed?" and "does this plan fit?". |
| Scanner of actual edges | **Not a substitute.** Brain's graph covers only reached code (45–86% of `app/` is absent), is per method, guesses kinds from names, silently drops unresolvable calls, and runs only in `local`. |
| Agent access (MCP, context) | **Parallel, not shared.** Both can be registered side by side. An agent drafting a plan can ask Brain "what does route X touch?" and ask us "is this dependency allowed?". |
| Agent guidance files | **Conflict.** `brain:generate-rules`, from the CLI or the browser, overwrites `CLAUDE.md` and `AGENTS.md` in full, and adds a fixed flow statement that can contradict the architecture spec. |

**Conclusion:** Brain complements the core bet and doesn't compete with it. The WIP is right to keep "describing flow" out of scope. It should stop calling that flow "runtime", because neither tool observes runtime behaviour.

## Redundancy and reuse

**Does Brain make a new package redundant?** No. Nothing in it declares intended structure, accepts a plan, checks one, or enforces anything.

**Ideas worth borrowing.** These can be taken freely:

- Edge ids derived from content, which keep output stable ([GraphBuilder.php L2896–2919](https://github.com/laramint/laravel-brain/blob/07df09a40269db5bdac7ec242141bb825eae4e51/src/Graph/GraphBuilder.php#L2896-L2919)).
- "Computed, never guessed" labelling for values the scan can't read, such as cache keys, AI models and tool lists.
- Separate counts for separate failure modes.
- Unreached items reported together with the references the tool could see but not follow.
- Registering the optional `laravel/mcp` server behind a `class_exists` check.
- Static parsing of providers, including the `*If` variants and `$bindings`.
- A PSR-4 map that also covers path repositories and `nwidart/laravel-modules` (`src/Analysis/ComposerPsr4Map.php`).
- A confirm or `--dry-run` step before writing agent files.

**Code that could be reused.** The best candidates are `ContainerBindingAnalyzer` with its record and registry, `FacadeAnalyzer`, `ComposerPsr4Map` and `PhpFileParser`. There are two ways to reuse them:

1. **A Composer dependency.** This is proven to work: `sandermuller/richter` imports `ProjectAnalyzer`, `GraphBuilder`, `MethodTracer`, `PhpFileParser` and `Graph`/`Node`/`Edge`. The costs are real, though:
   - it pulls in a 34k-line package plus `laramint/laravel-stress`;
   - Brain makes no public-API promise (no `@api` markers, only 5 `@internal` markers), and its internals change often between minor versions;
   - the project depends on a single maintainer.
2. **Copying specific files with attribution.** In practice this is blocked until upstream adds a LICENSE file that names a copyright holder.

## Open questions surfaced

1. Should the WIP's system diagram, non-goals and "Where we differ" stop calling Brain "runtime flow"? Brain is static entry-point flow, and so is anything we might build in that space.
2. Do we ask upstream to add a LICENSE file with a named copyright holder before copying any code, or commit to borrowing ideas only?
3. Is a "Brain integration" (for example, its flow context in the plan view) worth tying to a local-only, single-maintainer package with no public API? Or do we stay a sibling MCP server and nothing more?
4. How do we protect our prescriptive agent guidance from `brain:generate-rules`, given it overwrites whole files from both the CLI and the browser? Options are hooks and CODEOWNERS only, a documented coexistence pattern, or an upstream change (see [#138](https://github.com/laramint/laravel-brain/issues/138)).
5. Brain offers no precedent for "unknown" edges, because it drops unresolvable calls silently. What exactly does our scanner label unknown: `app()`, `resolve()`, `make()`, string container keys, variables of unknown type?
6. Is static provider parsing enough for "the implementation behind each contract", given the limits Brain hits (closure concretes, contextual bindings, last binding wins)? Or do we read bindings from a booted app?
7. Brain gives no model for a CI mode, since it registers nothing outside `local`. Our CI mode has to be designed from scratch.
