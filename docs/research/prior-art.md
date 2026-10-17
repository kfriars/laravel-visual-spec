# Prior art: tools that declare, enforce or picture architecture

Research for [#2](https://github.com/kfriars/laravel-visual-spec/issues/2), a ticket on the map in [#1](https://github.com/kfriars/laravel-visual-spec/issues/1). Researched on 2026-10-17. Versions and repository state were checked that day on GitHub and Packagist. Every repository link below is pinned to the commit that was read.

**Question.** What existing tools let a team declare an app's intended architecture, enforce it, picture it, or check a design against it before code exists, and what does each one leave undone?

Vocabulary follows [`GLOSSARY.md`](../../GLOSSARY.md): the **architecture spec** is the declared description of intended structure; a **plan** is a proposed set of new or changed classes and their intended dependencies, written before the code exists.

## Answer in brief

- **Enforcement is solved in PHP.** Deptrac, PHPat, PHPArkitect and Pest's arch plugin all check class-level dependencies against declared rules in CI. The real difference is where the rules live. Deptrac's rules are data (a YAML or PHP config). The rules of the other three are PHP code inside their own test or analysis engine.
- **Picturing existing code is common.** Deptrac (Graphviz, Mermaid), dependency-cruiser, Nx, Tach, Import Linter, Sloppy, Sonargraph and SonarQube all draw the dependencies the code has today.
- **Picturing intent with no code behind it is also common.** Structurizr and LikeC4 do this. None of the tools found pictures a plan: classes that don't exist yet, placed in the app's real layers, with their intended edges checked.
- **Plan validation barely exists anywhere.** The nearest tools are:
  - hello2morrow's **Zügel** MCP server, which checks one proposed dependency at a time. It is commercial and supports Java, C# and Python only.
  - **Sloppy**'s `architecture place`, which says where one new class belongs, from a prose description.
  - **Laravel Architecture Kit**'s `target` and `file-rules` reports, which give the rules for one future file path.
  - **Sonargraph** and **Structure101** simulations, which move, rename or delete existing code on a virtual model.
- **Two young Laravel-world packages overlap with the WIP's agent loop.** They are `heyosseus/sloppy` (v1.7.0, first created 2026-09-10) and `gracjankubicki/laravel-architecture-kit` (v0.9.3). Both offer an MCP server and a placement query. Neither validates a plan with several classes, and neither renders a plan for review.
- **Laravel module packages impose conventions, not boundaries.** `nwidart/laravel-modules` v13 and `internachi/modular` v3 set folder, namespace and Composer conventions, but declare and enforce no rules between modules.
- **Agent surfaces are richest outside PHP.** Nx, Structurizr and LikeC4 ship MCP servers. None of the four named PHP enforcers has one; agents get CLI output from them.
- **The gap.** The core bet needs an architecture spec that is:
  - engine-neutral data with documented meaning;
  - able to validate a whole plan with no code;
  - renderable as plans and as planned-versus-built differences for human review;
  - enforceable, with the same meaning, by a separate Pest plugin.

  None of the tools found does all of this. Most do none of the plan-side parts.

## Scope and method

- **Named in the ticket:**
  - PHP: Deptrac, PHPat, PHPArkitect, Pest arch.
  - Laravel: nWidart/laravel-modules, InterNACHI/modular.
  - Other ecosystems: ArchUnit, dependency-cruiser, Nx module boundaries, Structurizr/C4, Sonargraph, Structure101.
- **Turned up and added:**
  - PHP: Sloppy, Laravel Architecture Kit, VK Modulite.
  - Laravel: Laravel Shift Blueprint, a generator that writes code from a design file.
  - Other ecosystems: LikeC4; hello2morrow Zügel; SonarQube architecture, which succeeds Structure101; Tach and Import Linter (Python); Packwerk (Ruby); eslint-plugin-boundaries; go-arch-lint, in one line.
- **Excluded:** Laravel Brain, which has its own ticket. `grazulex/laravel-atlas` is a descriptive component mapper, like Brain in kind. It is noted, not analysed ([atlas-packagist]).
- **Method:**
  - Read docs and source from shallow clones, pinned in [Sources](#sources).
  - Read first-party sites for hosted and commercial products.
  - Took versions from GitHub and Packagist metadata.
  - Ran nothing. Claims marked "(grep)" mean a search of the cloned docs or source found nothing; they are absence-of-evidence, not proof.

## Per-tool findings

Each entry covers the five things the ticket asks for, then what the tool leaves undone:

- **Declaration:** what a declaration looks like.
- **Assignment:** how a class gets into a layer or module.
- **Picture:** how humans review or picture the architecture.
- **Before code:** whether it can check a design before the code exists.
- **Agent:** what an agent can consume.

### PHP

#### Deptrac (`deptrac/deptrac` 4.7.2, released 2026-09-15)

- **Declaration:**
  - Config file: YAML or PHP, with `deptrac.php` as the default ([deptrac-readme]). Deptrac's own config uses the PHP builder (`Layer::withName(...)->collectors(...)`) ([deptrac-self-config]).
  - Concepts: layers, a ruleset and violations ([deptrac-concepts]).
  - The ruleset is an allow-list: "By default, all dependencies between layers are forbidden!" ([deptrac-concepts]). Prefixing a layer with `+` also allows every layer it allows ([deptrac-config]).
  - You can use several config files, one per "architectural view" ([deptrac-concepts]).
  - None of the documented config keys holds a per-rule reason ([deptrac-config]).
- **Assignment:**
  - Each layer has collectors:
    - name patterns: `classLike`/`classNameRegex` regexes, `glob`, `functionName`, `tagValueRegex`;
    - location: `directory`;
    - type relations: `extends`, `implements`, `inherits`, `interface`, `trait`, `uses`, `attribute`, `method`;
    - other sources: `superglobal`, `composer` (a package), `php_internal`;
    - composition: `layer`, and `bool` combinators ([deptrac-collectors]).
  - A token "can be assigned to more than one layer" ([deptrac-concepts]).
  - `@deptrac-internal` (or a configured tag such as `@internal`) means a class cannot be referenced outside its layer ([deptrac-concepts], [deptrac-config]).
- **Picture:**
  - Graphviz formatters (display, dot, html, image) and a Mermaid formatter draw layers with edge counts. Both support groups and hidden layers ([deptrac-formatters], [deptrac-config]).
  - The picture shows actual dependencies between layers.
- **Before code:** No. Deptrac parses the files under `paths` ([deptrac-config]). Stub classes would be analysed like any other code.
- **Agent:**
  - Output formatters: JSON, JUnit, GitHub Actions, Code Climate and table ([deptrac-formatters]).
  - Debug commands ([deptrac-debugging]):
    - `debug:layer`, `debug:token`: which layers a token is in;
    - `debug:unassigned`: tokens in no layer;
    - `debug:dependencies`;
    - `debug:unused`: rules that allow no actual dependency;
    - `changed-files --with-dependencies`: experimental; the layers a set of files belongs to, plus the layers that depend on them.
  - No MCP server (grep).
- **Known violations:**
  - `skip_violations` plus a baseline formatter tolerate known violations.
  - A skip entry that no longer matches is reported as an error ([deptrac-concepts], [deptrac-formatters]).
  - Dependencies on code in no layer are "uncovered". They are reported only with `--report-uncovered` and fail the run only with `--fail-on-uncovered` ([deptrac-concepts]).
- **Leaves undone:** no plan input, no reasons on rules, and no picture of intent beyond the actual layer graph.

#### PHPat (`phpat/phpat` 0.12.5, released 2026-09-15)

- **Declaration:**
  - A PHPStan extension. Rules are public methods on PHP test classes, for example: `PHPat::rule()->classes(Selector::inNamespace('App\Domain'))->shouldNot()->dependOn()->classes(...)->because('...')`.
  - Test classes are registered in `phpstan.neon` with the tag `phpat.test` ([phpat-start], [phpat-rules]).
  - Assertions include `dependOn`/`canOnlyDependOn`, `extend`, `implement`, `include`, `construct`, `exist` and `beFinal` ([phpat-assertions]).
  - `nonIgnorable()` stops a rule's errors from being ignored or baselined ([phpat-rules]).
  - It is pre-1.0, and minor versions may break the API ([phpat-readme]).
- **Assignment:** selectors `classname` (exact or regex), `inNamespace`, `withFilepath`, `implements`, `extends`, `appliesAttribute`, `isInterface` and others, with `Not`, `AllOf` and `AnyOf` combinators ([phpat-selectors]).
- **Picture:** none.
- **Before code:** No.
- **Agent:** PHPStan's output formats, baseline and ignore features ([phpat-other]). The rules are PHP code, so an agent learns them by reading code.
- **Leaves undone:** no data form of the rules, no picture, no plan input.

#### PHPArkitect (`phparkitect/phparkitect` 1.3.1, released 2026-09-22)

- **Declaration:**
  - `phparkitect.php` returns a closure that adds rules to a `ClassSet`: `Rule::allClasses()->that(new ResideInOneOfTheseNamespaces('App\Domain'))->should(new NotHaveDependencyOutsideNamespace('App\Domain'))->because('...')` ([arkitect-readme]).
  - The `because()` text appears verbatim in violation output ([arkitect-readme]). The design philosophy says "The config communicates architectural intent" ([arkitect-philosophy]).
- **Assignment:** expressions such as `ResideInOneOfTheseNamespaces` (a prefix), the `...Exactly` variants, `HaveNameMatching`, `Extend`, `Implement`, `IsA`, `HaveTrait`, `HaveAttribute` and `ContainDocBlockLike`. `except()` takes wildcards ([arkitect-rules], [arkitect-readme]).
- **Picture:** none.
- **Before code:** No. `debug:expression` lists the classes that match an expression in today's code ([arkitect-readme]).
- **Agent:** `--format=json` or `gitlab`. All violations are collected unless `--stop-on-failure` is passed ([arkitect-readme]).
- **Known violations:**
  - `generate-baseline` creates a baseline; `prune-baseline` "only removes entries", so it "cannot" legitimise new violations.
  - Baseline matching survives line moves ([arkitect-readme]).
- **Leaves undone:** rules are PHP code; no picture; no plan.

#### Pest arch plugin (`pestphp/pest` v5.3.2, released 2026-10-12; `pestphp/pest-plugin-arch` has no GitHub releases)

- **Declaration:**
  - Expectations in test files, for example `arch()->expect('App\Models')->toOnlyBeUsedIn('App\Repositories')->ignoring('App\Models\User')`.
  - Presets: `php`, `security`, `laravel`, `strict`, `relaxed`, and custom presets ([pest-arch]).
- **Assignment:**
  - Namespace, class or function name strings. A namespace covers everything inside it. Wildcards such as `App\*\Traits` work since Pest 3.8 ([pest-arch]).
  - The plugin requires `ta-tikoma/phpunit-architecture-test` and PHP ^8.4 ([pest-arch-composer]).
  - `toOnlyBeUsedIn` scans every user namespace from Composer's PSR-4 mappings ([pest-arch-blueprint], lines 187–213).
- **Picture:** none.
- **Before code:** No.
- **Agent:**
  - Test results, plus `--log-junit` and `--teamcity` ([pest-cli]).
  - Pest 5 adds an Agent plugin. `--agent='<snippet>'` runs a one-off check inside the real suite, and its guidance installs through Laravel Boost ([pest-agent], [pest5]). It is unrelated to arch rules.
- **Reporting:**
  - `toOnlyUse` returns after the first disallowed use. `toOnlyBeUsedIn` reports only the first offending object ([pest-arch-blueprint], lines 150–213). This matches the WIP's note that Pest stops at the first violation.
  - `@pest-arch-ignore-line` and `@pest-arch-ignore-next-line` comments are honoured ([pest-arch-blueprint], lines 122–131).
- **Leaves undone:** rules are PHP code; one violation per expectation; no picture; no plan.

#### Sloppy (`heyosseus/sloppy` v1.7.0, released 2026-10-14, MIT), found in research

- **What it is:** a deterministic, local static analyser for "the debt AI coding agents leave behind". It has 27 rules, plus opt-in "your architecture" rules SL304 to SL308 ([sloppy-readme]).
- **Declaration:**
  - An architecture profile, either under `sloppy.architecture` in `config/sloppy.php` or in a `sloppy-architecture.php` file returning an array. Declaring it in both places is an error ([sloppy-config]).
  - Concepts ([sloppy-config]):
    - **Roles.**
    - **Presets:** `laravel`, `laravel-actions`, `service-repository`, `ddd`, `hexagonal`, `modular` and `none`.
    - **Policies:**
      - `may_depend_on` and `may_not_depend_on`;
      - `may_not` capabilities: `db`, `http`, `dispatch`, `request`, `env`, `view` and `container`;
      - shape: `public_methods` and `final`;
      - `advice`: "Write it for the agent that will read it".
    - **Module boundaries:** `'modules' => 'App\Modules\{module}\*'`, with `public` and `shared` patterns.
    - **`covers`:** paths where every class must play a role.
- **Assignment:**
  - A class takes the first role it matches. Matchers: `namespace`, `path`, `suffix`, `parent`, `extends`, `implements`, `uses` (a trait), `attribute`, `kind`, `any` and `not`.
  - A mistake in the profile "stops the run with a message naming the key, rather than silently matching nothing" ([sloppy-config]).
- **Picture:** `sloppy architecture graph` emits Mermaid, DOT or JSON. It shows classes per role and dependency counts between roles, with forbidden edges in red ([sloppy-config]).
- **Before code:** partly.
  - `sloppy architecture place "an action that refunds an order" --name=RefundOrder` returns the role, namespace, directory, class name and file, and what the role may depend on and do.
  - SL307 flags a new class in a covered path that plays no role, on the edit that created it ([sloppy-config]).
  - Neither takes a set of planned classes and dependencies.
- **Agent:**
  - MCP tools: `sloppy_scan`, `sloppy_diff`, `sloppy_rules`, `sloppy_health`, `sloppy_architecture`, `sloppy_place` and `sloppy_architecture_prompt` ([sloppy-agents]).
  - Hooks for Claude Code, Gemini CLI and Codex run Sloppy after each edit and before the agent may finish ([sloppy-agents]).
  - Profile authoring commands ([sloppy-config]):
    - `architecture init` infers a profile from the code, and detects `nwidart/laravel-modules`.
    - `import` translates a `deptrac.yaml`.
    - `prompt` gives an agent what it needs to write a profile from prose.
- **Leaves undone:**
  - No plan with several classes.
  - No picture of a plan, and no planned-versus-built view.
  - The profile is read only by Sloppy and its PHPStan extension.

#### Laravel Architecture Kit (`gracjankubicki/laravel-architecture-kit` v0.9.3, released 2026-10-15, MIT), found in research

- **What it is:** "generated architecture guidance, bounded project context for AI agents, AST-backed audit, and an optional guard for selected enforceable rules" ([kit-readme]).
- **Declaration:**
  - The architecture patterns a project uses are selected in `config/architectures.php`. That file references the package's enum classes, so the package is a runtime dependency.
  - A separate `.architecture-kit/target.json` declares a target architecture, in `info`, `warn` or `block` mode. It has `classification.roles`, `modules`, `placements`, and `dependencies` allow policies ([kit-readme]).
- **Assignment:** directory, namespace and whole-segment patterns. "Role classification follows architecture path segments in both top-level and domain-first layouts" ([kit-readme]).
- **Picture:** none found. The README mentions no diagram, Mermaid or other visual output (grep) ([kit-readme]).
- **Before code:** partly ([kit-readme]).
  - `file-rules` and `target` accept a future PHP file path and report the rules and expected placement for it.
  - `architecture-kit:make --agent` and the MCP `scaffold` tool return a plan and a skeleton without writing anything.
  - `impact` evaluates proposed signature changes, deletes and moves of existing code.
  - It has no input format for a plan with several classes and their dependencies.
- **Agent:**
  - MCP tools including `impact`, `scaffold`, `architecture-target`, `guard`, `reach` and `revision-diff`.
  - Generated Laravel Boost guidelines and skills.
  - JSON schemas via `--schema` ([kit-readme]).
  - `revision-diff` compares the architecture between Git revisions and includes a configuration channel ([kit-readme]). The README describes no command that runs the same code under a proposed change to the target.
- **Governance:** a target "reference" counts only once a person sets `accepted: true` and a non-empty `accepted_by`. "No package command or MCP tool writes or accepts a reference" ([kit-readme]).
- **Leaves undone:** no picture; no plan with several classes; the spec is its own config and is not consumed by Pest.

#### VK Modulite (`VKCOM/modulite` v1.1.12, released 2025-12-24), found in research

- **Declaration** ([modulite-docs], in Russian):
  - A `.modulite.yaml` in each module folder. It has:
    - `export`: everything not exported is internal by default;
    - `require`: other modules, Composer packages, outside classes, static methods, functions, constants and globals;
    - `allow-internal-access`: named exceptions, used to freeze existing violations;
    - `force-internal`.
  - Modules nest (for example `@messi/kernel`), and a child can be private to its parent.
- **Assignment:** by folder. Symbols are listed one by one, with no "masks" ([modulite-docs]).
- **Picture:** a PhpStorm plugin that the docs call "a convenient UI over the configs". It shows badges in the file tree, offers quick fixes that edit the YAML, and scopes find-usages to a module ([modulite-docs]).
- **Before code:** No. A new dependency gets added to `require` by quick fix, so the docs note that it "will be visible in review" ([modulite-docs]).
- **Agent:** the YAML files themselves.
- **Where it is checked:** in the IDE, in KPHP compilation, and on CI through a PHPStan plugin ([modulite-docs]). The `VKCOM/modulite-phpstan` repository was last pushed 2024-08-30 (GitHub metadata).
- **Leaves undone:** per-symbol lists are heavy for apps; the UI is IDE-only; no plan.

### Laravel

#### nWidart/laravel-modules (v13.0.0, released 2026-03-19)

- **Conventions it imposes** ([lm-readme], [lm-config], [lm-json-stub], [lm-composer-stub]):
  - Modules live in `Modules/<Name>`, set by `paths.modules`, under the namespace `Modules`.
  - Each module has a `module.json` (`name`, `alias`, `priority`, `providers`, `files`).
  - Each module has its own `composer.json` mapping `Modules\<Name>\` to `app/` by PSR-4. These are autoloaded through `wikimedia/composer-merge-plugin`.
  - Generator paths are set per class kind, for example `app/Actions` and `app/Console`.
  - Enable and disable status is stored in `modules_statuses.json`.
- **Assignment:** by folder and namespace convention.
- **Picture, before code, agent:** nothing architecture-specific. No rules for dependencies between modules (grep for `requires` and `depend` in `src`).
- **For the spec:** a common real layout to express, `Modules\{Name}\...` with Laravel-style folders inside. Sloppy's `init` already detects it ([sloppy-config]).

#### InterNACHI/modular (3.0.2, released 2026-03-23)

- **Conventions it imposes** ([modular-readme]):
  - Modules live in `app-modules/<name>/`. Each is a Composer package, registered as a path repository and required as `modules/<name>:*`.
  - Each module has `src/`, `tests/`, `routes/`, `resources/` and `database/`.
  - The default namespace is `Modules\`; the docs recommend changing it to an organisation namespace.
  - It adds `--module=` to most `make:` commands, and auto-discovers commands, migrations, policies, Blade components and listeners.
- **Positioning:** "as much a set of conventions as it is a package" ([modular-readme]). v3 needs PHP 8.3+ and Laravel 11+, and moves discovery into plugins ([modular-v3]).
- **Assignment:** by folder, with a PSR-4 mapping per module ([modular-stub]).
- **Rules:** none (grep for `depend` and `requires` in `src`). Each module's `composer.json` has a `require` block ([modular-stub]), but nothing uses it to police class references.

#### Laravel Shift Blueprint (`laravel-shift/blueprint` v2.13.0), found in research

- **What it does:** declares components before code exists, in a `draft.yaml`. The draft lists models with columns, and controllers with statements such as `validate`, `save`, `dispatch`, `fire` and `send`.
- **What it generates:** models, migrations, factories, controllers, routes, form requests, jobs, events, mail, views and tests ([blueprint-readme]).
- **Schema:** it ships a JSON Schema for drafts ([blueprint-schema]).
- **Why it matters:** it is the Laravel community's best-known "write the design as YAML first" tool. It has no concept of layers or allowed dependencies, so there is nothing to check a draft against.

### Other ecosystems

#### ArchUnit (Java; `TNG/ArchUnit` v1.5.1, released 2026-09-25)

- **Declaration:**
  - Java test code, for example: `layeredArchitecture().consideringAllDependencies().layer("Controller").definedBy("..controller..")...whereLayer("Service").mayOnlyBeAccessedByLayers("Controller")` ([au-what]).
  - Further rule families: onion architecture, slices, and modules ([au-lib]). A module can be defined by packages, or by annotations on `package-info`, such as `@AppModule(allowedDependencies = ..., exposedPackages = ...)`.
- **Assignment:** package identifiers such as `..controller..`.
- **Picture as the spec:** `adhereToPlantUmlDiagram(...)` derives rules from a PlantUML component diagram whose stereotypes are package identifiers. The diagram is the declaration ([au-lib]).
- **Before code:** No. It checks the classes it imports, so only code that exists.
- **Known violations:** `FreezingArchRule` stores violations in a plain-text store, reports only new ones, and "will automatically reduce the known stored violations" as they are fixed ([au-lib]).
- **Empty selectors:** a rule fails by default when its `should` part matches no classes (`archRule.failOnEmptyShould`) ([au-config]).
- **Agent:** test failures.
- **Leaves undone:** the diagram is checked only against existing code; no plan.

#### dependency-cruiser (JS/TS; v18.6.0, released 2026-10-11)

- **Declaration:**
  - `.dependency-cruiser.js` or JSON with `forbidden`, `allowed` (plus `allowedSeverity`), `required` and `options` sections.
  - Each rule has `name`, `comment`, `severity`, and `from`/`to` with `path`/`pathNot` regexes.
  - Group matching (`$1`) expresses "peer folders must not import each other" in one rule.
  - Other conditions: `couldNotResolve`, orphans, `reachable`, `circular` and dependency types ([dc-rules]).
- **Assignment:** file path regexes.
- **Picture:**
  - Reporters include `dot`, `ddot` (folder level), `archi`, `flat`, `dot-webpage`, `mermaid`, `d2`, `html` and `markdown`.
  - `--focus`, `--reaches`, `--collapse` and `--highlight` narrow the view ([dc-cli]).
  - `--affected <revision>` keeps only changed modules and their dependents. Fed to the Mermaid reporter, it can go "in a PR comment" ([dc-cli]).
- **Known violations:** `--baseline`, with `--baseline-mode` set to `full`, `shrink-only` or `format`, plus `--ignore-known` ([dc-cli]).
- **Before code:** No.
- **Agent:** a JSON reporter. No MCP server mentioned in the docs (grep).
- **Leaves undone:** no plan; its pictures show existing modules and their violations.

#### Nx module boundaries (Nx 23.3.0, released 2026-10-07)

- **Declaration** ([nx-boundaries]):
  - `tags` on each project, in `project.json` or under `nx` in `package.json`.
  - `depConstraints` in the `@nx/enforce-module-boundaries` ESLint rule, pairing a `sourceTag` with `onlyDependOnLibsWithTags`.
  - Alternatives: an experimental Oxlint plugin, and a Conformance rule on the Enterprise plan that works for other project types.
  - "Projects without any tags cannot depend on any other projects."
- **Assignment:** by tag, at the project (package) level, not the class level.
- **Picture:** `nx graph` is an interactive project graph. Clicking an edge shows which files created the dependency. `--file=output.json` exports the data, and `--focus` narrows the view ([nx-graph]).
- **Before code:** No.
- **Agent:** the Nx MCP server ([nx-mcp]):
  - workspace: `nx_workspace` (the project graph and `nx.json`), `nx_project_details`;
  - generators: `nx_generators`, `nx_generator_schema`;
  - `nx_visualize_graph`, which needs an IDE with Nx Console;
  - CI tools.
- **Leaves undone:** the graph page doesn't say whether boundary violations show on the graph ([nx-graph]); no plan.

#### Structurizr / C4 (`structurizr/structurizr` v2026.09.19)

- **Declaration:**
  - Structurizr DSL: `workspace { model { ... } views { ... } }`.
  - The model holds people, software systems, containers and components, linked by relationships written `->`, with views per C4 level ([structurizr-readme], [structurizr-cookbook]).
  - The tooling is now consolidated in `structurizr/structurizr`. The old `structurizr/java` repository is archived (GitHub metadata).
- **Assignment:** none by default; the model is written by hand. The `!components` keyword runs a component finder over a Java codebase, using BCEL and JavaParser ([structurizr-component], [structurizr-component-docs]).
- **Picture:** this is the product's purpose: many diagrams from one model ([structurizr-readme]).
- **Before code:** Yes, as a model. Inspections check the workspace itself, for example missing descriptions, disconnected elements, and elements in no view. They don't check source code ([structurizr-inspections]).
- **Agent:** an MCP server with `validate`, `parse` (DSL to a JSON workspace), `inspect`, Mermaid and PlantUML export, and workspace tools for a Structurizr server ([structurizr-mcp], [structurizr-mcp-src]).
- **Leaves undone:** no rules engine; no check of the model against code, apart from Java component discovery.

#### LikeC4 (v1.60.0, released 2026-10-15), found in research

- **Declaration:** `.c4` or `.likec4` files with `specification` (element kinds), `model`, `views` and `global` blocks. All the files merge into one model ([likec4-dsl]).
- **Picture:**
  - Live diagrams, a VS Code extension, and `likec4 start` for a local preview.
  - Code generation to Mermaid, DOT, D2 and PlantUML; export to Draw.io ([likec4-readme], [likec4-cli-readme]).
  - `likec4 validate` checks syntax errors and "Layout drift (outdated manual layout)" ([likec4-cli]).
- **Before code:** Yes, as a model, with no link to code.
- **Agent:**
  - An MCP server with about twenty tools, including `read-project-summary`, `query-graph`, `find-relationship-paths` and `element-diff` ([likec4-mcp]).
  - `render-view` renders an interactive diagram inline in chat, for hosts that support MCP Apps.
  - `preview-view` "renders a preview of a LikeC4 view defined by DSL text, against an existing project's real elements, without saving to disk".
- **Leaves undone:** no enforcement and no conformance check against code. Even so, `preview-view` is the closest thing found to showing a person a proposed change as a picture before anything is written.

#### Sonargraph-Architect and Zügel (hello2morrow, commercial)

- **Declaration:**
  - An architecture DSL. Its grammar includes `artifact`, `include`/`exclude`, `interface` (with `export`), `connector`, `connect ... to`, `require`, `apply`, `template` and `connection-scheme`, plus stereotypes such as `strict`, `relaxed`, `hidden`, `exposed` and `deprecated` ([sg-grammar]).
  - Aspects such as layering, components and client/server separation can live in separate files or in one complete file. You add each architecture file to the check ([sg-def]).
- **Assignment:** `include` and `exclude` patterns on artifacts ([sg-grammar]). Elements in no artifact are listed as "Unassigned internal/external components" ([sg-def]).
- **Picture:** exploration, graph and cycle views. Every arc can be traced to the code that causes it ([sg-product]).
- **Before code:** simulation only.
  - A virtual model holds resolutions and refactorings, while the read-only "Parser" model keeps the facts ([sg-vm]).
  - Virtual refactorings let you "Delete, move or rename elements virtually, see the effect on architecture and structure" before changing any code ([sg-product]).
- **Languages:** C#, C/C++, Java/Kotlin, TypeScript, Go and Python 3 ([sg-product]). No PHP.
- **Zügel MCP server** ([zugel]):
  - Tagline: "Teaches your architecture to AI coding agents".
  - "Every time the agent wants to add a dependency it asks Zügel whether that dependency respects your architecture and does not create or enlarge a cyclic dependency group."
  - "A negative reply comes with a correct alternative, before any code is generated."
  - Its configuration can be generated from Sonargraph architecture definitions.
  - It supports Java, C# and Python, and comes with one free licence per Sonargraph-Architect licence.
  - The page lists no visual output of its own.
- **Leaves undone:** commercial and closed; no PHP; Zügel checks dependencies one at a time rather than a reviewed plan; no picture of a plan.

#### Structure101, now SonarQube architecture

- **Status:** Sonar acquired Structure101 on 2024-10-15 ([sonar-pr]). Structure101 products are no longer sold ([s101-page]).
- **Structure101 Studio:** restructuring worked on a model ([s101-actions]).
  - Actions made through the levelized structure map (LSM) change the model, not the code, and are recorded in an action list.
  - The list is exported for developers to carry out in their IDE.
  - Actions that the code has since implemented are greyed out.
- **SonarQube architecture:**
  - A beta on SonarQube Cloud was announced 2025-12-16, for Java, JavaScript, TypeScript, Python and C# ([sonar-blog]).
  - In SonarQube Server, the current architecture is reverse-engineered on every analysis ([sq-arch]).
  - The intended architecture is defined in the UI as an allow-list between sibling containers: "any non-defined dependency between siblings will be considered as forbidden" ([sq-arch]).
  - Classes and files are "recursively grouped within their packages/folders/modules" ([sq-arch]).
  - Deviations (wrong dependencies and wrong locations) are raised as SonarQube issues ([sq-arch]).
  - Architecture is in the Developer, Enterprise and Data Center editions, not the Community Build ([sq-arch]).
- **Retired file format:** an earlier "architecture as code" config, `architecture.json` or `architecture.yaml`, held perspectives, groups with file-path `patterns`, and constraints with `from`, `to`, `message` and `relation`. It was "deprecated, pending removal in January 2026" ([sq-aac]).
- **Agent:** the SonarQube MCP server's "Vortex context" category covers "code architecture search, call flows, coding guidelines, and SCA dependency checks (requires an active subscription)" ([sq-mcp]).
- **Leaves undone:** no plan input; the intended architecture now lives in a server UI rather than the repo; PHP is not in the beta's language list.

#### Python, Ruby and JS boundary tools (for ideas)

- **Tach** (`tach-org/tach` v0.35.3):
  - `tach.toml` declares `[[modules]]` with `path`, `depends_on`, `cannot_depend_on`, `layer`, `visibility`, `utility` and `unchecked`; `[[interfaces]]` with `expose` patterns; and an ordered `layers` list ([tach-config], [tach-interfaces], [tach-toml]).
  - Commands: `tach show` (DOT, Mermaid, or a remote web viewer), `tach report`, `tach map` (JSON), and `tach sync` to "sync constraints with actual dependencies" ([tach-readme], [tach-commands]).
- **Import Linter:**
  - Contracts in INI or TOML, of type `layers`, `independence`, `forbidden`, `protected` and `acyclic_siblings`.
  - Layer contracts count indirect imports, and `exhaustive` fails when a layer is missing from the list ([il-layers], [il-protected]).
  - `import-linter explore` serves a local browser UI of the package graph, with drill-down, import totals and cycle breakers. `drawgraph` prints DOT ([il-ui]).
- **Packwerk** (`Shopify/packwerk` v3.3.1) ([packwerk-usage], [packwerk-readme]):
  - A `package.yml` in any folder makes a package, with `enforce_dependencies: true` or `strict` and a `dependencies` list.
  - Existing violations go in `package_todo.yml`, which "should not be added to, but worked off". Strict mode refuses new entries.
  - `metadata` is free-form, for example stewards.
  - Only dependency checks ship; other checks come from extension gems. Graphs come from the third-party graphwerk.
- **eslint-plugin-boundaries** (v7.2.0): `boundaries/elements` classify files by `type` and glob `pattern`. `boundaries/dependencies` policies take `default: "disallow"` and `from`/`allow`/`disallow` on element types, file categories and module origin ([boundaries-readme]).
- **go-arch-lint** (v1.19.0): a `.go-arch-lint.yml` "describing the ideal project architecture", with components and `mayDependOn`. It "compares the actual (code) and desired (config) dependency graph" and draws a graph ([gal-readme]).

## Comparison

### Declaration and assignment

| Tool | Where rules live | Data or code | Unit | How code is assigned | Undeclared dependency is |
| --- | --- | --- | --- | --- | --- |
| Deptrac | `deptrac.php` or `deptrac.yaml` | Data | Layer of tokens; several layers allowed | Collectors: regex, directory, extends, implements, attribute, Composer package, bool | Forbidden between layers |
| PHPat | PHP test classes registered in `phpstan.neon` | Code | Selector set | Selectors: namespace, name regex, file path, extends, implements, attribute | Allowed unless a rule forbids it (`canOnlyDependOn` gives an allow-list) |
| PHPArkitect | `phparkitect.php` | Code | Expression-selected set | Namespace prefix, name pattern, extends, implements, attribute | Allowed unless a rule forbids it (`DependsOnlyOnTheseNamespaces` gives an allow-list) |
| Pest arch | Test files | Code | Namespace, class or function string | Namespace covers everything inside; wildcards | Allowed unless an expectation forbids it (`toOnlyUse` gives an allow-list) |
| Sloppy | `config/sloppy.php` or `sloppy-architecture.php` | Data (PHP array) | Role, module | First matching role: namespace, path, suffix, parent, extends, implements, trait, attribute | Per-role policy; `may_depend_on` is an allow-list for classified targets |
| Laravel Architecture Kit | `config/architectures.php` and `.architecture-kit/target.json` | Data (PHP with enums; JSON) | Role, module | Path and namespace segments | Target: every matching policy must allow it |
| Modulite | `.modulite.yaml` per folder | Data | Module (folder) | Folder; symbols listed | Forbidden unless in `require`; internal unless exported |
| nWidart, modular | `module.json` and `composer.json` | Convention | Module | Folder plus PSR-4 | No rules |
| Blueprint | `draft.yaml` | Data | Component to generate | Not applicable | No rules |
| ArchUnit | Java tests, or a PlantUML diagram | Code, or diagram | Layer, slice, module | Package identifiers, annotations | Per the layer constraints |
| dependency-cruiser | `.dependency-cruiser.js` or JSON | Data | File | Path regex | Allowed unless forbidden; or a "not-in-allowed" warning |
| Nx | Tags in project config, plus ESLint config | Data | Project | Tags | Untagged projects may depend on nothing |
| Structurizr | DSL workspace | Data | C4 element | By hand (Java: component finder) | No enforcement |
| LikeC4 | `.c4` files | Data | Element | By hand | No enforcement |
| Sonargraph | DSL architecture files | Data | Artifact | `include`/`exclude` patterns | Governed by `connect` and interfaces |
| SonarQube | Server UI (formerly `architecture.json`/`.yaml`) | Data, in the server | Container | The code's own packages and folders | Forbidden between siblings |
| Tach | `tach.toml` | Data | Module | Python import path, globs | Forbidden outside `depends_on` (omitting `depends_on` allows all) |
| Import Linter | INI or TOML | Data | Package | Import names | Per contract type |
| Packwerk | `package.yml` per folder | Data | Package (folder) | Folder | A violation, when enforced |
| eslint-plugin-boundaries | ESLint config | Data | Element type | Glob pattern | As configured (`disallow` in the docs' example) |

### Picture, before code, agent surface

| Tool | Picture of existing code | Picture of intent | Check before code exists | Agent surface |
| --- | --- | --- | --- | --- |
| Deptrac | Graphviz or Mermaid layer graph with counts | No | No; stub files only | JSON, JUnit, Code Climate; `debug:*` commands; no MCP |
| PHPat | None | None | No | PHPStan formats |
| PHPArkitect | None | None | No | JSON, GitLab |
| Pest arch | None | None | No | JUnit, TeamCity |
| Sloppy | Role graph (Mermaid, DOT, JSON), forbidden edges red | No | Partly: `place` for one new class | MCP (7 tools); agent hooks |
| Laravel Architecture Kit | None | None | Partly: rules for one future path; scaffold; impact of signature, delete and move proposals | MCP; Boost guidelines and skills; JSON schemas |
| Modulite | PhpStorm badges and tree | Through the IDE | No | YAML files |
| nWidart, modular | None | None | Not applicable | Module manifests |
| Blueprint | Not applicable | Not applicable | Generates code from a draft; no rules | YAML and a JSON Schema |
| ArchUnit | None | A PlantUML diagram can be the rule | No | Test failures |
| dependency-cruiser | Many formats; `--affected` for PRs | No | No | JSON |
| Nx | Interactive graph; files behind each edge | No | No | Nx MCP |
| Structurizr | Java component finder only | Yes, C4 diagrams | Model only; no code check | MCP: validate, parse, inspect, export |
| LikeC4 | No | Yes, live diagrams | Model only; `preview-view` | MCP (about 20 tools) |
| Sonargraph and Zügel | Yes | Architecture view | Simulated moves, renames and deletes; Zügel checks one dependency | Zügel MCP |
| Structure101, SonarQube | LSM; SonarQube interactive map | Intended architecture in the UI | Structure101 action lists edit a model; SonarQube no | SonarQube MCP architecture search (subscription) |
| Tach | DOT, Mermaid, web viewer | No | No | `tach map` JSON; `report --raw` |
| Import Linter | `explore` UI; DOT | No | No | CLI |
| Packwerk | Third-party graphwerk | No | No | YAML files |
| eslint-plugin-boundaries | No | No | No | ESLint output |

## Ideas the architecture spec could borrow

These answer "what can the architecture spec express" with prior art. Each idea is a candidate, not a decision.

- **Ways to assign a class.**
  - Namespace alone, as the WIP proposes, is the narrowest option.
  - Deptrac collectors and Sloppy matchers add directory or path, name suffix or regex, `extends`/`implements`, trait, attribute, Composer package, and boolean combinations ([deptrac-collectors], [sloppy-config]).
  - Deptrac lets a token sit in several layers. Sloppy uses the first match ([deptrac-concepts], [sloppy-config]).
- **Default-deny allow-lists.**
  - Used by Deptrac (with `+Layer` to inherit what another layer allows), SonarQube between siblings, Nx for untagged projects, and eslint-plugin-boundaries (`default: "disallow"`) ([deptrac-config], [sq-arch], [nx-boundaries], [boundaries-readme]).
  - The opposite is the per-rule style of Pest arch, PHPat and PHPArkitect.
- **Modules matched by pattern rather than listed.** Sloppy's `{module}` capture, dependency-cruiser's `$1` group matching, and ArchUnit's `(*)` package identifiers ([sloppy-config], [dc-rules], [au-lib]). These cover "every feature folder" without naming each one.
- **Public surface of a module.**
  - Tach `interfaces.expose`, Modulite `export`, ArchUnit `exposedPackages`, Sloppy `boundaries.public`, and Deptrac `@deptrac-internal` ([tach-interfaces], [modulite-docs], [au-lib], [sloppy-config], [deptrac-concepts]).
  - These are prior art for the WIP's candidate feature-privacy rule (`Internal` versus `Contracts`).
- **Protected targets with named users.** Import Linter's `protected` contract with `allowed_importers`, and Pest's `toOnlyBeUsedIn` ([il-protected], [pest-arch]). These are prior art for the WIP's candidate contract-usage rules.
- **Capabilities, not just classes.** Sloppy's `may_not` list (`db`, `http`, `dispatch`, `request`, `env`, `view`, `container`), and Pest's `not->toUse()` on functions ([sloppy-config], [pest-arch]). These are prior art for the WIP's bans on `app()` and `resolve()`.
- **A reason on every rule.** PHPArkitect and PHPat `because()`, dependency-cruiser `comment`, Sloppy `advice`, and the `message` field in SonarQube's retired format ([arkitect-readme], [phpat-rules], [dc-rules], [sloppy-config], [sq-aac]). Deptrac documents no such field.
- **Baselines that can only shrink.** dependency-cruiser `--baseline-mode shrink-only`, PHPArkitect `prune-baseline`, ArchUnit `FreezingArchRule`, and Packwerk `strict` mode ([dc-cli], [arkitect-readme], [au-lib], [packwerk-usage]).
- **Coverage checks.**
  - Code in no layer: Deptrac `debug:unassigned` and `--fail-on-uncovered`, Import Linter `exhaustive`, Sloppy `covers` and SL307 ([deptrac-debugging], [deptrac-concepts], [il-layers], [sloppy-config]).
  - Rules or selectors that match nothing: ArchUnit `failOnEmptyShould`, Deptrac `debug:unused`, and Sloppy's refusal to run a profile that matches nothing ([au-config], [deptrac-debugging], [sloppy-config]).
- **Several views or aspects of one architecture.** Deptrac's several config files, Sonargraph's aspect files, Structurizr's views ([deptrac-concepts], [sg-def], [structurizr-readme]).
- **A diagram as the spec.** ArchUnit's PlantUML rules ([au-lib]).
- **Layout stability.** LikeC4 stores manual layouts, and `validate` flags layout drift. SonarQube levelizes containers left to right ([likec4-cli], [sq-arch]).
- **Human approval on record.** Modulite surfaces new dependencies as YAML diffs for review. Laravel Architecture Kit's target reference counts only with `accepted_by` ([modulite-docs], [kit-readme]).
- **Interop with existing specs.** Sloppy imports `deptrac.yaml`. Zügel generates its config from Sonargraph ([sloppy-config], [zugel]).

## The gap

The core bet is plan validation plus visual review, with enforcement as plumbing in a separate Pest plugin. Measured against that, the tools leave six gaps.

1. **A plan as an input.**
   - **Needed:** a declared set of new or changed classes and their intended dependencies, checked with no code, deterministically, with a complete list of violations.
   - **Found:** no tool takes that input. The partial answers are:
     - Zügel checks one dependency at a time, with no PHP support, and is commercial ([zugel]).
     - Sloppy's `place` handles one class, described in prose; it returns a role, and dependencies are checked only after the code is written ([sloppy-config]).
     - Laravel Architecture Kit's `file-rules`, `target` and `scaffold` handle one future path, or a skeleton ([kit-readme]).
     - Sonargraph virtual refactorings and Structure101 action lists edit a model of existing code ([sg-product], [s101-actions]).
     - Any code checker can check stub classes. That is the WIP's "skeleton gate", but it needs files on disk.
2. **A picture of a plan, and of planned versus built.**
   - **Found:** pictures of existing code, and pictures of hand-made models (Structurizr, LikeC4).
   - LikeC4's `preview-view` renders a draft view against real model elements, the closest analogue, but the model isn't code ([likec4-mcp]).
   - Change-scoped pictures exist only for code: dependency-cruiser `--affected` with Mermaid ([dc-cli]).
   - No tool found draws classes that don't exist yet inside the code's real layers, or a planned-versus-built difference.
3. **One architecture spec, two consumers.**
   - **Needed:** a spec that both the visual and plan tooling and a separate Pest plugin read, with one meaning.
   - **Found:** every rule format is bound to its engine.
     - Pest arch, PHPat and PHPArkitect hold their rules as PHP code. Another tool can't read those rules without running that code.
     - The data formats are each read only by their own engine: Deptrac, Sloppy, Laravel Architecture Kit, Tach, Packwerk, Modulite and dependency-cruiser.
     - Translation exists only one way, as an import: Sloppy reads `deptrac.yaml`, and Zügel reads Sonargraph's definitions ([sloppy-config], [zugel]).
     - No tool found generates or drives Pest arch tests from a data spec.
4. **Reviewing a change to the spec, with an impact preview.**
   - **Needed:** for a proposed edit to the architecture spec, what it would newly forbid, newly allow, or leave unassigned.
   - **Found:**
     - SonarQube edits the intended architecture directly in a server UI ([sq-arch]).
     - Laravel Architecture Kit diffs code and configuration across revisions ([kit-readme]).
     - Deptrac `debug:unused` lists rules that allow nothing ([deptrac-debugging]).
   - No tool found evaluates the same code under a proposed spec change.
5. **Laravel-aware but layout-neutral.**
   - **Needed:** handling real Laravel layouts, such as the default `app/`, nWidart `Modules/<Name>`, modular `app-modules/<name>/src`, and domain-first trees.
   - **Found:**
     - The module packages impose layouts without rules ([lm-config], [modular-readme]).
     - Sloppy's presets and `init`, and Laravel Architecture Kit's classification, understand some of these layouts ([sloppy-config], [kit-readme]).
     - The generic enforcers are layout-neutral but know nothing about Laravel.
6. **An agent surface over the spec, in PHP.**
   - **Needed:** deterministic CLI and MCP operations over the spec: rule lookup, layer-of, check-dependency, validate-plan.
   - **Found:**
     - Among PHP tools, only Sloppy and Laravel Architecture Kit have MCP servers. Neither validates plans.
     - None of the four established enforcers has one.
     - Outside PHP, Nx, Structurizr, LikeC4 and Zügel show what such a surface can look like ([nx-mcp], [structurizr-mcp], [likec4-mcp], [zugel]).

**What this means for the "build at all" decision.**

- The plan side is unoccupied in PHP and nearly unoccupied anywhere: gaps 1, 2 and 4.
- So is an engine-neutral spec with documented meaning: gap 3.
- But much of the agent loop around enforcement is already taken. Sloppy and Laravel Architecture Kit offer placement queries, role graphs, MCP servers, agent hooks and imports of existing specs.
- A new package's distinct value would therefore be plan validation, visual review and an engine-neutral spec. Another enforcer or another general-purpose MCP server would duplicate existing work.

## Open questions surfaced

1. **Build, or build on?** Sloppy (MIT, 157 stars, v1.7.0) and Laravel Architecture Kit (MIT, v0.9.3) already cover much of the WIP's agent operations. Sloppy's `place` is close to the WIP's `suggest-layer`, and `architecture graph` to its `diagram`; both have MCP servers. Should the package interoperate with them, import or export their profiles, or treat the overlap as evidence against parts of the scope?
2. **Interop with Deptrac.** Deptrac is the only widely used PHP rule format that is pure data, and Sloppy already imports it. Should the architecture spec import or emit `deptrac.yaml` (the WIP already asks about emitting it), or define its own format?
3. **Default-deny or rule-by-rule?** Deptrac, Nx and SonarQube deny undeclared dependencies between declared units. Pest arch, PHPat and PHPArkitect allow anything that no rule forbids. This changes what the Pest plugin must generate.
4. **What can assign a class?** Namespace only, as in the WIP's non-goal about codebases not laid out by namespace, or richer matchers (suffix, extends, implements, attribute, path) as in Deptrac and Sloppy? May a class belong to several layers?
5. **Must a rule that matches nothing fail?** ArchUnit and Sloppy say yes by default; Pest passes silently. If the architecture spec defines this, the Pest plugin's contract inherits it.
6. **Plan granularity.** Validate a whole plan at once (no tool found does), answer one dependency at a time (Zügel, Sloppy `place`), or both?
7. **How to draw classes that don't exist yet** inside diagrams of the real layers, keeping layout stable. LikeC4's `preview-view` (preview without saving) and its layout-drift check are concrete models to study.
8. **Where shrink-only baselines live.** In the architecture spec's semantics (Packwerk strict, dependency-cruiser shrink-only), or only in the Pest plugin?
9. **Is a module's public surface a first-class concept?** That is, exports or interfaces as in Tach, Modulite and ArchUnit, rather than a folder naming convention (the WIP's `Internal`/`Contracts`).
10. **One spec file or several aspect files?** Deptrac supports several config files and Sonargraph several aspect files.
11. **How human approval of spec changes is recorded.** PR review and CODEOWNERS only, or an explicit field such as Laravel Architecture Kit's `accepted_by`?
12. **Do plans run through the same engine as the code?** If the architecture spec is data with documented meaning, does plan validation evaluate rules with the same library the Pest plugin calls, or with an independent implementation checked against fixtures?

## Sources

Repository links are pinned to the commit read on 2026-10-17. Versions come from GitHub's latest release or Packagist on that date.

**PHP**

[deptrac-readme]: https://github.com/deptrac/deptrac/blob/66595d1ca307afff492d99501ffaeba147e848b0/README.md
[deptrac-self-config]: https://github.com/deptrac/deptrac/blob/66595d1ca307afff492d99501ffaeba147e848b0/deptrac.php
[deptrac-concepts]: https://github.com/deptrac/deptrac/blob/66595d1ca307afff492d99501ffaeba147e848b0/docs/concepts.md
[deptrac-config]: https://github.com/deptrac/deptrac/blob/66595d1ca307afff492d99501ffaeba147e848b0/docs/configuration.md
[deptrac-collectors]: https://github.com/deptrac/deptrac/blob/66595d1ca307afff492d99501ffaeba147e848b0/docs/collectors.md
[deptrac-formatters]: https://github.com/deptrac/deptrac/blob/66595d1ca307afff492d99501ffaeba147e848b0/docs/formatters.md
[deptrac-debugging]: https://github.com/deptrac/deptrac/blob/66595d1ca307afff492d99501ffaeba147e848b0/docs/debugging.md
[phpat-readme]: https://github.com/carlosas/phpat/blob/b7e0172bd931a99e5046f04175473b73968e56bb/README.md
[phpat-start]: https://github.com/carlosas/phpat/blob/b7e0172bd931a99e5046f04175473b73968e56bb/docs/getting-started.md
[phpat-rules]: https://github.com/carlosas/phpat/blob/b7e0172bd931a99e5046f04175473b73968e56bb/docs/documentation/rules.md
[phpat-assertions]: https://github.com/carlosas/phpat/blob/b7e0172bd931a99e5046f04175473b73968e56bb/docs/documentation/assertions.md
[phpat-selectors]: https://github.com/carlosas/phpat/blob/b7e0172bd931a99e5046f04175473b73968e56bb/docs/documentation/selectors.md
[phpat-other]: https://github.com/carlosas/phpat/blob/b7e0172bd931a99e5046f04175473b73968e56bb/docs/documentation/other.md
[arkitect-readme]: https://github.com/phparkitect/arkitect/blob/158b1f491ec83e857d1ba2185b6525e29cbd834c/README.md
[arkitect-philosophy]: https://github.com/phparkitect/arkitect/blob/158b1f491ec83e857d1ba2185b6525e29cbd834c/docs/philosophy.md
[arkitect-rules]: https://github.com/phparkitect/arkitect/blob/158b1f491ec83e857d1ba2185b6525e29cbd834c/docs/rules.md
[pest-arch]: https://github.com/pestphp/docs/blob/6e368f64fed849966541bfdc1736b8180309343c/arch-testing.md
[pest-cli]: https://github.com/pestphp/docs/blob/6e368f64fed849966541bfdc1736b8180309343c/cli-api-reference.md
[pest-agent]: https://github.com/pestphp/docs/blob/6e368f64fed849966541bfdc1736b8180309343c/agent.md
[pest5]: https://github.com/pestphp/docs/blob/6e368f64fed849966541bfdc1736b8180309343c/pest5-now-available.md
[pest-arch-composer]: https://github.com/pestphp/pest-plugin-arch/blob/244465d5dc9ea9fb5ac5c9badb7eb47d1594e4c1/composer.json
[pest-arch-blueprint]: https://github.com/pestphp/pest-plugin-arch/blob/244465d5dc9ea9fb5ac5c9badb7eb47d1594e4c1/src/Blueprint.php#L94-L213
[sloppy-readme]: https://github.com/Heyosseus/sloppy/blob/4379f3174b23ecfa24787dcc0421d3ad57d46b4c/README.md
[sloppy-config]: https://github.com/Heyosseus/sloppy/blob/4379f3174b23ecfa24787dcc0421d3ad57d46b4c/docs/configuration.md
[sloppy-agents]: https://github.com/Heyosseus/sloppy/blob/4379f3174b23ecfa24787dcc0421d3ad57d46b4c/docs/agents.md
[kit-readme]: https://github.com/gracjankubicki/laravel-architecture-kit/blob/bcea42374626ec3eb14fe7b5b0cfbb823f5711f3/README.md
[modulite-docs]: https://github.com/VKCOM/modulite/blob/b52b219404a9b8cab8312910b42686fd83f90070/docs/index.html
[atlas-packagist]: https://packagist.org/packages/grazulex/laravel-atlas

- Deptrac: [README][deptrac-readme], [own config][deptrac-self-config], [concepts][deptrac-concepts], [configuration][deptrac-config], [collectors][deptrac-collectors], [formatters][deptrac-formatters], [debugging][deptrac-debugging].
- PHPat: [README][phpat-readme], [getting started][phpat-start], [rules][phpat-rules], [assertions][phpat-assertions], [selectors][phpat-selectors], [other features][phpat-other].
- PHPArkitect: [README][arkitect-readme], [philosophy][arkitect-philosophy], [rules][arkitect-rules].
- Pest: [architecture testing][pest-arch], [CLI reference][pest-cli], [Agent plugin][pest-agent], [Pest 5 announcement][pest5], [arch plugin composer.json][pest-arch-composer], [arch plugin Blueprint.php][pest-arch-blueprint].
- Sloppy: [README][sloppy-readme], [configuration][sloppy-config], [coding agents][sloppy-agents].
- Laravel Architecture Kit: [README][kit-readme].
- VK Modulite: [docs][modulite-docs] (Russian).
- Laravel Atlas: [Packagist][atlas-packagist].

**Laravel**

[lm-readme]: https://github.com/nWidart/laravel-modules/blob/965931ef67f5ec1d1fa90998f8d1b10f912a1d71/README.md
[lm-config]: https://github.com/nWidart/laravel-modules/blob/965931ef67f5ec1d1fa90998f8d1b10f912a1d71/config/config.php
[lm-json-stub]: https://github.com/nWidart/laravel-modules/blob/965931ef67f5ec1d1fa90998f8d1b10f912a1d71/src/Commands/stubs/json.stub
[lm-composer-stub]: https://github.com/nWidart/laravel-modules/blob/965931ef67f5ec1d1fa90998f8d1b10f912a1d71/src/Commands/stubs/composer.stub
[modular-readme]: https://github.com/InterNACHI/modular/blob/428b63ec001acbf78225f132c25ebc5442621aa1/README.md
[modular-v3]: https://github.com/InterNACHI/modular/blob/428b63ec001acbf78225f132c25ebc5442621aa1/CHANGES-V3.md
[modular-stub]: https://github.com/InterNACHI/modular/blob/428b63ec001acbf78225f132c25ebc5442621aa1/stubs/composer-stub-latest.json
[blueprint-readme]: https://github.com/laravel-shift/blueprint/blob/373ccbf6f2e13f90724280f51875fb9ace0634c8/README.md
[blueprint-schema]: https://github.com/laravel-shift/blueprint/blob/373ccbf6f2e13f90724280f51875fb9ace0634c8/schema.json

- nWidart/laravel-modules: [README][lm-readme], [config][lm-config], [module.json stub][lm-json-stub], [composer stub][lm-composer-stub].
- InterNACHI/modular: [README][modular-readme], [v3 changes][modular-v3], [composer stub][modular-stub].
- Blueprint: [README][blueprint-readme], [draft schema][blueprint-schema].

**Other ecosystems**

[au-what]: https://github.com/TNG/ArchUnit/blob/a0640ffa3ea5806c4fbab0dfbd29e363646d0fa3/docs/userguide/004_What_to_Check.adoc
[au-lib]: https://github.com/TNG/ArchUnit/blob/a0640ffa3ea5806c4fbab0dfbd29e363646d0fa3/docs/userguide/008_The_Library_API.adoc
[au-config]: https://github.com/TNG/ArchUnit/blob/a0640ffa3ea5806c4fbab0dfbd29e363646d0fa3/docs/userguide/010_Advanced_Configuration.adoc
[dc-rules]: https://github.com/sverweij/dependency-cruiser/blob/c580daf6477aa5619bacad9c9091c9b7cc81882d/doc/rules-reference.md
[dc-cli]: https://github.com/sverweij/dependency-cruiser/blob/c580daf6477aa5619bacad9c9091c9b7cc81882d/doc/cli.md
[nx-boundaries]: https://nx.dev/docs/features/enforce-module-boundaries
[nx-graph]: https://nx.dev/docs/features/explore-graph
[nx-mcp]: https://nx.dev/docs/reference/nx-mcp
[structurizr-readme]: https://github.com/structurizr/structurizr/blob/da99caf2f2857e5373f7b58569152274f1725ba3/README.md
[structurizr-component]: https://github.com/structurizr/structurizr/blob/da99caf2f2857e5373f7b58569152274f1725ba3/structurizr-component/README.md
[structurizr-mcp-src]: https://github.com/structurizr/structurizr/blob/da99caf2f2857e5373f7b58569152274f1725ba3/structurizr-mcp/src/main/java/com/structurizr/mcp/DslTools.java
[structurizr-cookbook]: https://docs.structurizr.com/dsl/cookbook/component-view
[structurizr-component-docs]: https://docs.structurizr.com/java/component
[structurizr-inspections]: https://docs.structurizr.com/workspaces/inspections
[structurizr-mcp]: https://docs.structurizr.com/ai/mcp
[likec4-readme]: https://github.com/likec4/likec4/blob/d1760072255d81eb81244c2227fe7844196caae3/README.md
[likec4-mcp]: https://github.com/likec4/likec4/blob/d1760072255d81eb81244c2227fe7844196caae3/packages/mcp/README.md
[likec4-dsl]: https://github.com/likec4/likec4/blob/d1760072255d81eb81244c2227fe7844196caae3/apps/docs/src/content/docs/dsl/intro.mdx
[likec4-cli]: https://github.com/likec4/likec4/blob/d1760072255d81eb81244c2227fe7844196caae3/apps/docs/src/content/docs/tooling/cli.mdx
[likec4-cli-readme]: https://github.com/likec4/likec4/blob/d1760072255d81eb81244c2227fe7844196caae3/packages/likec4/README.md
[sg-product]: https://www.hello2morrow.com/products/sonargraph/architect
[sg-def]: https://eclipse.hello2morrow.com/doc/standalone/content/architecture_definition.html
[sg-grammar]: https://eclipse.hello2morrow.com/doc/standalone/content/architecture_grammar.html
[sg-vm]: https://eclipse.hello2morrow.com/doc/standalone/content/using_virtual_models.html
[zugel]: https://www.hello2morrow.com/products/zugel
[sonar-pr]: https://sonarsource.com/company/press-releases/sonar-acquires-structure101-to-strengthen-code-quality-offering
[s101-page]: https://sonarsource.com/structure101/
[s101-actions]: https://www.sonarsource.com/structure101/docs/java/studio/content/restructure101/actions
[sonar-blog]: https://www.sonarsource.com/blog/introducing-architecture-in-sonarqube/
[sq-arch]: https://docs.sonarsource.com/sonarqube-server/architecture
[sq-aac]: https://docs.sonarsource.com/sonarqube-server/2025.4/design-and-architecture/configuring-the-architecture-analysis
[sq-mcp]: https://docs.sonarsource.com/sonarqube-mcp-server/about-the-mcp-server
[tach-readme]: https://github.com/tach-org/tach/blob/bc92d71520a1acdb9aacd1f2362ca49b58259dda/README.md
[tach-config]: https://github.com/tach-org/tach/blob/bc92d71520a1acdb9aacd1f2362ca49b58259dda/docs/usage/configuration.md
[tach-commands]: https://github.com/tach-org/tach/blob/bc92d71520a1acdb9aacd1f2362ca49b58259dda/docs/usage/commands.md
[tach-interfaces]: https://github.com/tach-org/tach/blob/bc92d71520a1acdb9aacd1f2362ca49b58259dda/docs/usage/interfaces.md
[tach-toml]: https://github.com/tach-org/tach/blob/bc92d71520a1acdb9aacd1f2362ca49b58259dda/tach.toml
[il-layers]: https://github.com/seddonym/import-linter/blob/857ae28bc9cb383ce9c831e7e8345eb9b40667ae/docs/contract_types/layers.md
[il-protected]: https://github.com/seddonym/import-linter/blob/857ae28bc9cb383ce9c831e7e8345eb9b40667ae/docs/contract_types/protected.md
[il-ui]: https://github.com/seddonym/import-linter/blob/857ae28bc9cb383ce9c831e7e8345eb9b40667ae/docs/ui.md
[packwerk-readme]: https://github.com/Shopify/packwerk/blob/31e534283f93a225b520cddab3c2d735557c76de/README.md
[packwerk-usage]: https://github.com/Shopify/packwerk/blob/31e534283f93a225b520cddab3c2d735557c76de/USAGE.md
[boundaries-readme]: https://github.com/javierbrea/eslint-plugin-boundaries/blob/50f2d3193ec571569a478c811ec83171a16d11a3/README.md
[gal-readme]: https://github.com/fe3dback/go-arch-lint/blob/bf473afb033c7c3eb7aba68dc399cf919287131e/README.md

- ArchUnit: [what to check][au-what], [Library API][au-lib], [advanced configuration][au-config].
- dependency-cruiser: [rules reference][dc-rules], [CLI][dc-cli].
- Nx: [module boundaries][nx-boundaries], [explore the graph][nx-graph], [Nx MCP][nx-mcp].
- Structurizr: [README][structurizr-readme], [component module][structurizr-component], [MCP DslTools.java][structurizr-mcp-src], [component view cookbook][structurizr-cookbook], [component finder][structurizr-component-docs], [inspections][structurizr-inspections], [MCP docs][structurizr-mcp].
- LikeC4: [README][likec4-readme], [MCP][likec4-mcp], [DSL intro][likec4-dsl], [CLI docs][likec4-cli], [CLI package README][likec4-cli-readme].
- Sonargraph: [product page][sg-product], [defining an architecture][sg-def], [DSL grammar][sg-grammar], [virtual models][sg-vm].
- Zügel: [product page][zugel].
- Structure101 and SonarQube: [acquisition press release][sonar-pr], [Structure101 page][s101-page], [Studio actions][s101-actions], [SonarQube architecture blog post][sonar-blog], [SonarQube Server architecture docs][sq-arch], [retired architecture-as-code docs][sq-aac], [SonarQube MCP server][sq-mcp].
- Tach: [README][tach-readme], [configuration][tach-config], [commands][tach-commands], [interfaces][tach-interfaces], [own tach.toml][tach-toml].
- Import Linter: [layers contract][il-layers], [protected contract][il-protected], [UI][il-ui].
- Packwerk: [README][packwerk-readme], [usage][packwerk-usage].
- eslint-plugin-boundaries: [README][boundaries-readme].
- go-arch-lint: [README][gal-readme].

**Versions and metadata** (checked 2026-10-17):

- [GitHub GraphQL API](https://docs.github.com/en/graphql) `latestRelease` and `isArchived`, for the repositories above.
- Packagist package JSON, for `heyosseus/sloppy`, `gracjankubicki/laravel-architecture-kit` and `grazulex/laravel-atlas` (for example https://packagist.org/packages/heyosseus/sloppy.json).
- GitHub repository metadata, for licences and the last push to `VKCOM/modulite-phpstan`.
