# Sloppy and Laravel Architecture Kit, from source

Research for [#15](https://github.com/kfriars/laravel-visual-spec/issues/15), a ticket on the map in [#1](https://github.com/kfriars/laravel-visual-spec/issues/1). It goes deeper than the survey for [#2](https://github.com/kfriars/laravel-visual-spec/issues/2) ([prior-art]), which named these two packages as the closest neighbours to the core bet. Researched on 2026-10-17.

**Question.** What do Sloppy (`heyosseus/sloppy`) and Laravel Architecture Kit (`gracjankubicki/laravel-architecture-kit`) actually do? Could laravel-visual-spec be built on either, interoperate with them, or be made redundant by them?

Vocabulary follows [`GLOSSARY.md`](../../GLOSSARY.md). The **architecture spec** is our declared description of intended structure. A **plan** is a proposed set of new or changed classes and their intended dependencies, written before the code exists. Each package's own artifacts keep their own names: Sloppy's "architecture profile", and Kit's "configuration", "target" and "reference".

## Answer in brief

- **Both work on code that already exists.**
  - Neither takes a plan with several classes and their intended dependencies.
  - Neither judges a dependency whose ends don't exist yet.
  - Neither draws a plan, or planned against built.
  - The plan side of the core bet is still unoccupied.
- **Sloppy is a code-quality analyser for PHP written by AI agents. Its architecture part is an opt-in add-on, 13 days old.**
  - It shipped in v1.4.0 and v1.5.0 on 2026-10-04.
  - The declaration is a PHP array, the "architecture profile". It holds ordered roles with matchers; per-role policies (allow and deny lists, capability bans, class shape, advice); module boundaries matched by a `{module}` pattern; and `covers`.
  - Sloppy's own analyser enforces the profile as rules SL304 to SL308.
  - It draws a role-level graph of the edges the code actually has, in Mermaid, DOT or JSON. An edge is red when a policy forbids it.
- **Laravel Architecture Kit is a much larger Laravel tool for guidance, audit and code intelligence.** It has 511 PHP source files and a Rust parsing engine. Its architecture model has three parts:
  - A closed vocabulary of 8 roles and 19 kinds, assigned by folder conventions or by declarations.
  - A hard-coded layer policy for the current audit.
  - A separate `.architecture-kit/target.json` file. Only this file lets a project declare dependency allow-lists. Kit checks it against the current code and reports a migration.
  - Kit has no visual output of any kind.
- **Before code exists, each answers for one class only:**
  - Sloppy's `place` ranks roles by keyword overlap with a prose description.
  - Kit's `target` and `file-rules` report the expected role, kind, module, placement and rule names for one future file path.
  - Neither evaluates the dependencies a planned class will have.
- **Both have rich agent surfaces:**
  - Sloppy: 7 MCP tools; post-edit and before-finish hooks for Claude Code, Gemini CLI and Codex; generated rulesets; a versioned JSON report.
  - Kit: 19 read-only MCP tools on `laravel/mcp`; a before-finish (`Stop`) hook; compact versioned JSON, with a JSON Schema for each command's output.
- **Neither declaration is designed to be read by another tool.**
  - Sloppy's profile is PHP, executed with `require`. It rejects unknown keys. Its JSON export flattens matchers to prose.
  - Kit's `target.json` is plain JSON, but strictly validated, tied to the closed vocabulary, and without a published input schema.
  - Both accept custom rules, which run only inside their own engines.
- **Maturity differs sharply.**
  - Sloppy is popular for its age: 157 stars, about 39,000 Packagist downloads, and issues from outside users. It has one maintainer and is five weeks old.
  - Kit has 3 stars, 1,569 downloads and no dependents. Every issue was filed by its own two accounts. It is pre-1.0, with breaking changes documented across minor releases. It must be installed as a runtime dependency, and since 2026-10-15 it parses with a native binary by default.
- **Neither makes laravel-visual-spec redundant on plan validation plus visual review.** Both already cover much of the agent loop around enforcement that the WIP draft sketches.

## Scope and method

- **Versions read:**
  - Sloppy: default branch at commit [`4379f31`][s-tree] (2026-10-15). It is four commits after the v1.7.0 tag. The `compare` API shows those commits add only the SL113 rule and its docs; no file under `src/Architecture/` changed.
  - Kit: commit [`bcea423`][k-tree], which is the v0.9.3 tag (2026-10-15).
- **How I read them:**
  - Downloaded the source archives at those commits, plus the files the archives leave out (Sloppy's `docs/`, Kit's `docs/` and `engine/`) from the same commits.
  - Read them; ran nothing.
  - Every behaviour claim below links to a file pinned to the commit read, with line numbers where they help.
- **Maturity data:** the GitHub API (repository, releases, tags, contributors, issues, compare) and Packagist package JSON, both on 2026-10-17.
- **"(grep)"** marks an absence found by searching the source. It is absence of evidence, not proof.
- **Upstream issues and pull requests are counted, not linked.**

## Sloppy (`heyosseus/sloppy` v1.7.0)

### What it is

- **Purpose:** "Static analysis for the debt AI coding agents leave behind" ([s-composer]).
  - It runs 27 rules on any PHP 8.3+ project, with Laravel-aware rules for Laravel 12 and 13.
  - It is "deterministic and local: no model, no API key, no network" ([s-readme], lines 9–13 and 40–42).
- **Where the architecture fits:** a row in the rule table, "Your architecture (opt-in)": SL304 Layer Violation, SL305 Forbidden Capability, SL306 Boundary Violation, SL307 Misplaced Class and SL308 Role Shape ([s-readme], line 95).
- **Who reads the profile:** the same roles also feed the older rules about where code belongs (SL201, SL202, SL206, SL207, SL208) ([s-config], lines 81–85). So the profile is primarily input to Sloppy's own analyser.

### Declaration

- **Where it lives** ([s-loader], lines 23–48):
  - under `sloppy.architecture` in `config/sloppy.php`, or
  - in `sloppy-architecture.php` at the project root.

  The file is loaded with `require` and must return an array. Declaring the profile in both places throws.
- **Top-level keys:** only `preset`, `roles`, `policies`, `boundaries` and `covers` are allowed. Any other key stops the run ([s-profile], lines 48 and 76–85).
- **What each concept means:**
  - **Role:** a named part a class can play, with matchers, an optional `description` and an `intended_abstraction` flag ([s-role]; [s-roleparser], lines 28–53).
    - Names are lower-case and hyphenated ([s-profile], line 37).
    - Project roles are tried first, in written order, then preset roles. A project role with a preset role's name replaces it, and `false` removes it ([s-profile], lines 206–226).
  - **Preset:** a shipped bundle of roles, policies and boundaries. The presets are `laravel` (the default), `laravel-actions`, `service-repository`, `ddd`, `hexagonal`, `modular` and `none`.
    - They are PHP constants, selected by a closed `match`; an unknown name throws ([s-presets], lines 7–18 and 266–282).
    - The `laravel` preset enforces nothing new ([s-config], lines 132–140).
  - **Policy:** what one role may depend on and do ([s-policy], lines 7–46; [s-policyparser], lines 14 and 111–135).
    - `may_not_depend_on` forbids each role or class-name glob it lists.
    - `may_depend_on` is an allow-list, but it judges only dependencies on classes that play a role. Dependencies on the same role are always allowed. Framework, vendor and unclassified classes are "never judged by it" ([s-policy], lines 95–118).
    - `may_not` bans capabilities ([s-capability]): `db.read`, `db.write`, `http`, `dispatch`, `request`, `env`, `view`, `container`.
    - `public_methods` and `final` set a shape for classes in the role.
    - `advice` is a sentence attached to every finding the policy produces ("Write it for the agent that will read it") ([s-config], lines 165–180).
    - There is one `advice` per policy, not a reason per allowed edge.
  - **Boundaries** ([s-boundaries], lines 36–83):
    - `modules` is a pattern with exactly one `{module}`, such as `App\Modules\{module}\*`, so a module is covered without being listed.
    - `public` holds globs relative to a module's root; `shared` holds fully qualified globs.
    - A class may name another module's class only when that class matches `public` or `shared`.
  - **`covers`:** path globs where every class should play a role. It feeds SL307 ([s-profile], lines 27–28 and 113–122).
- **Validation:**
  - An unknown key, unknown role name, unknown capability or malformed `{module}` pattern throws a `ProfileException` that names the key path ([s-policyparser], lines 21–31, 73–83 and 122–129).
  - A role with no matchers throws, because "it would match nothing" ([s-roleparser], lines 71–77).
  - A role that matches no class in the analysed paths only produces a console warning from `sloppy architecture` ([s-runner], lines 75–77).
  - The survey's "refusal to run a profile that matches nothing" is therefore narrower than it reads: Sloppy refuses empty matchers, not empty matches.

### How it assigns classes

- **First match wins:** a class takes the first role whose matcher matches. The others are kept as "also matched" for explanations ([s-map], lines 39–47).
- **Matcher keys** ([s-matchkey], lines 13–62):
  - `namespace`: a glob on the fully qualified name.
  - `path`: a glob on the project-relative path.
  - `suffix`: compared as written, not as a glob.
  - `parent`: the class it extends directly.
  - `extends`: any ancestor, followed through parents the project declares ([s-classfacts], lines 80–90).
  - `implements`, `uses` (a trait), `attribute`: as named.
  - `kind`: class, interface, trait or enum.
  - Combinators: `any` and `not` ([s-roleparser], lines 58–80).
- **How keys combine:** keys within a role must all match, and a list value matches if any item does.
- **Globs:** `*` is any text, backslashes included, and nothing else is special ([s-glob]).
- **Deptrac alignment:** the vocabulary deliberately "follows deptrac's collectors where they overlap" ([s-matchkey], lines 10–11).
- **One answer everywhere:** both rule analysis and project-wide commands build `ClassFacts`, "so a class cannot get one role in a finding and another in `sloppy architecture`" ([s-classfacts], lines 12–19).
- **Only analysed code gets a role.** A dependency target is classified only if Sloppy's project index knows it. Otherwise its role is `null` ([s-map], lines 56–64). The index is built from the configured paths only ([s-snapshot], lines 44–74).

### What counts as a dependency

- **Counted:** every class name inside a class declaration ([s-scanner], lines 13–53):
  - `extends`, `implements`, traits, attributes;
  - parameter, property and return types;
  - `new`, static calls and constants;
  - `X::class`;
  - `instanceof` and `catch`.
- **Not counted:** docblocks, function names, constant names, and `self`/`static`/`parent`. `use` imports sit outside the class node, so they are not counted either (grep).
- **One edge per pair:** each target is recorded once per class, at its first mention.
- **Judging an edge** ([s-sl304], lines 77–105):
  - SL304 judges each target by the role it plays, or by a glob on its name.
  - SL306 judges module crossings ([s-sl306], lines 83–107).
  - SL305 counts capabilities only from unambiguous evidence: known Eloquent models, facades, helpers and injected framework types. `$order->save()` on a variable is not counted ([s-capscan], lines 20–29; [s-config], lines 191–193).

### What it can check before code exists, and how complete that is

- **`sloppy architecture place "<description>" --name=<Class>`**, and the MCP tool `sloppy_place` ([s-config], lines 230–238; [s-place-tool]), works like this ([s-placement], lines 7–54 and 121–138):
  - It splits the description into words and scores each role. A word scores 3 when it matches the role's name or suffix, and 1 when it matches the role's description or its most common namespace.
  - It returns the best role, up to three alternatives, and the role's policy written as instructions.
  - It is deterministic and calls no model.
  - The suggested namespace, directory and suffix come from existing classes in the role: the most common namespace and directory, and a suffix shared by at least half of them ([s-placement], lines 98–119; [s-snapshot], lines 117–172).
  - So for a role no class plays yet, the namespace, directory and file come back null.
  - It takes no dependencies as input and checks none.
- **SL307 Misplaced Class** is reported only by `sloppy diff` and the agent hooks ([s-sl307], lines 22–35 and 72–75).
  - It flags a class that the change adds in a covered path and that plays no role.
  - It needs the file to exist, and a git base to compare against.
- **Compared with validating a whole plan,** the WIP's candidate validate-plan, these cover one of its checks: "every class lands in a layer", for one class at a time, from prose. They do not check:
  - whether planned dependencies are allowed;
  - module boundaries;
  - capabilities;
  - whether the plan agrees with itself.

  A class that doesn't exist yet is unknown to the index, so it has no role. Code that names it is judged only by `may_not_depend_on` name globs, never by the `may_depend_on` allow-list ([s-map], lines 59–64; [s-policy], lines 97–105).
- **The skeleton route:** writing stub classes and running Sloppy (the WIP's "skeleton gate") does work. SL304, SL305 and SL306 judge whatever parses.

### What it pictures, and how

- **`sloppy architecture graph`**, in Mermaid (the default), DOT or JSON ([s-config], lines 240–250; [s-graph]):
  - **Nodes:** roles, labelled with their class counts.
  - **Edges:** role-to-role dependency counts, sorted by source and then target, drawn left to right ([s-graph], lines 33–73 and 93–133).
  - **Red edges:** an edge with any forbidden dependency is red, labelled "N (M forbidden)", with one example pair kept in JSON ([s-edge]).
  - **The invariant:** the graph is "drawn from the same dependencies SL304 judges, so a red edge in the picture is always a finding in the report" ([s-graph], lines 10–17).
- **Left out:**
  - classes with no role, and framework and vendor code ([s-graph], lines 14–17 and 43–54);
  - SL306 boundary crossings and SL305 capabilities; only role policies colour edges (grep in [s-graph]);
  - edges within one role.
- **Not offered:** no class-level drill-down, no picture of an allowed-but-unused rule, and no HTML or interactive view (the README roadmap lists "HTML reports" as still ahead; [s-readme], lines 221–227).
- **Formats:** the JSON form carries `schema: 1`.

### Agent surface and output contracts

- **MCP server** (`vendor/bin/sloppy-mcp` or `php artisan sloppy:mcp`), with 7 tools ([s-agents], lines 264–311):
  - `sloppy_scan`, `sloppy_diff`, `sloppy_rules`, `sloppy_health`;
  - `sloppy_architecture`: the overview, one class explained, or the graph;
  - `sloppy_place`;
  - `sloppy_architecture_prompt`.

  It speaks MCP revisions 2025-06-18, 2025-03-26 and 2024-11-05 ([s-mcp], line 48). Its instructions tell agents to call `sloppy_diff` before finishing and `sloppy_place` before creating a class ([s-mcp], lines 244–245).
- **Hooks**, installed by `sloppy agents install` ([s-agents], lines 7–49 and 83–110):
  - After each PHP edit, `sloppy hook post-edit` reports findings *that edit* introduced, compared with `HEAD`.
  - When the agent tries to finish, `sloppy hook stop` blocks once on new findings at or above `fail_on`.
  - Hooks exist for Claude Code, Gemini CLI and Codex. Output to the model is capped at ten findings.
- **Rulesets:** `sloppy:rules` writes rules into `CLAUDE.md`, `AGENTS.md`, `.cursorrules`, Copilot, Windsurf or Gemini files, or a Laravel Boost guideline.
  - A declared profile adds an "Architecture of this project" section: each role, where it lives, its suffix and its policy as instructions ([s-agents], lines 146–240; [s-brief], lines 7–13).
  - The JSON ruleset embeds the same brief under `architecture` ([s-ruleset], lines 197–212).
- **Profile authoring commands** ([s-config], lines 252–284; [s-runner], lines 30–40):
  - `architecture init` infers a profile from the code.
  - `architecture import` translates a Deptrac file. It looks only for `deptrac.yaml`, `deptrac.yml`, `depfile.yaml` and `depfile.yml`, not `deptrac.php` ([s-deptrac], lines 10–26).
  - `architecture prompt` gives an agent what it needs to write a profile from prose.
  - Each prints its proposal and writes only on confirmation or `--write`.
- **Output contracts:**
  - The scan and diff report is a "published contract": `schema` is bumped when a field changes meaning, keys are only added within a version, and output is byte-identical across runs ([s-json], lines 13–17).
  - `sloppy architecture --format=json` returns `schema: 1` with the preset, source, roles (name, origin, description, `matches`, classes, policy), `unclassified`, `boundaries` and `covers` ([s-report], lines 150–190).
  - In that output `matches` is a prose string such as `namespace App\Http\* or ...`, built by `describe()` ([s-matcher]; [s-pattern], lines 42–45). The JSON can't be turned back into a profile.
  - No JSON Schema file ships for any of these (grep for `$schema`).
- **Pest:** `expectCleanSloppyDiff()`, `expectCleanSloppyScan()` and `expectSloppyScoreAtLeast()` run Sloppy inside a Pest test ([s-pest]). Sloppy neither generates nor drives Pest arch tests (grep).

### Extension points: could another package consume or add to its declaration?

- **Consuming the profile:**
  - Reading it means executing PHP: `require` of `sloppy-architecture.php`, or loading the merged `config/sloppy.php` ([s-loader], line 42; [s-config], lines 9–28).
  - The classes that parse and hold it (`Profile`, `Policy`, `RoleParser`) carry no `@api` marker. The only `@api` markers are `Sloppy::forProject()` and one other method on `Sloppy` ("Used by heyosseus/phpstan-sloppy"), three `AnalysisContext` methods, and `ArchitectureMap`, which is reached through `AnalysisContext::roleOf()` ([s-sloppy], line 54; [s-map], line 19; grep for `@api`).
  - The JSON overview is readable without PHP, but lossy, because matchers are prose.
- **Adding to it:**
  - The profile rejects keys it doesn't know ([s-profile], lines 76–85; [s-policyparser], lines 194–206). Another package cannot annotate it in place with reasons, plan data or layout hints.
  - Presets, matcher keys and capabilities are closed: a `match`, an enum and an enum ([s-presets], lines 266–282; [s-matchkey]; [s-capability]).
- **Custom rules:** extend `BaseRule`, register under `custom_rules`, and call `$context->roleOf($class)` or `hasRole()`. They score, diff, baseline and appear in rulesets like built-in rules ([s-custom], lines 9–117).
- **Generating a profile:** another tool could write a `sloppy-architecture.php` file, since it is a plain returned array. Or it could emit `deptrac.yaml` and let `sloppy architecture import` translate it. Constructs that don't translate are noted in comments rather than guessed ([s-config], lines 276–279).

### Maturity

- **Licence:** MIT. `LICENSE.md` is present ([s-license]), and `composer.json` declares MIT ([s-composer]).
- **History:**
  - The repository was created 2026-09-10.
  - 21 releases, v0.1.0 (2026-09-10) to v1.7.0 (2026-10-14). v1.0.0 came on 2026-09-25 (GitHub releases).
  - 125 commits on the default branch (GitHub API).
  - The architecture features shipped in v1.4.0 and v1.5.0, both on 2026-10-04 ([s-changelog], lines 101–207).
- **Maintainers:**
  - One Packagist maintainer, `Heyosseus`.
  - GitHub contributors: `Heyosseus` (53), `yognevoy` (2), plus Copilot and Dependabot bots.
- **Adoption** (Packagist, 2026-10-17):
  - 39,274 installs in total, 37,646 of them in the last month.
  - 158 favourites, and 157 GitHub stars.
  - 1 dependent: `heyosseus/phpstan-sloppy`, by the same author, with 3,105 installs.
- **Issues:** 43 issues and pull requests in total. The 8 issues all came from users outside the project and are all closed; none is open.

### Where it falls short of plan validation plus visual review

- **No plan input:** one class at a time, from prose, with no dependencies.
- **Unclassified targets go unjudged:** the allow-list ignores targets the index doesn't know, which includes every class a plan has not built yet.
- **Placement leans on existing code:** a new role with no classes gets no namespace or directory.
- **The picture shows only actual role-level edges.**
  - Nothing is planned, and there is no planned-versus-built difference.
  - It shows no boundary or capability violations, and no class drill-down.
  - Nothing keeps the layout stable across diagrams.
- **No impact preview** for a change to the profile (grep).
- **The profile is bound to Sloppy.** It is PHP, closed to extra keys, and read only by Sloppy and its PHPStan extension.
- **Enforcement is Sloppy's own analyser, not Pest.** Findings carry a confidence score (90 for SL304 and SL306; [s-sl304], line 98; [s-sl306], line 100).

## Laravel Architecture Kit (`gracjankubicki/laravel-architecture-kit` v0.9.3)

### What it is

- **Self-description:** "Laravel tooling for four explicit capabilities: generated architecture guidance, bounded project context for AI agents, AST-backed audit, and an optional guard for selected enforceable rules" ([k-readme], line 26).
- **The catalog:** most of the package is a catalog of 18 Laravel patterns ([k-readme], lines 584–606). Each has prose guidance, generated Boost guidelines and skills, and, for some, audit rules:
  - Thin Controllers, Form Requests, Actions, Services, Query Objects, Data Objects, Value Objects;
  - Ports And Adapters, Saloon, Inertia, Fortify, Laravel AI, and others.
- **Code intelligence:** a large part is source-only code intelligence for agents. It covers `impact` (including signature, delete and move proposals), `path`, `reach`, `search`, `revision-diff`, `public-api` and `upgrade-plan` ([k-readme], lines 100–191 and 1044–1148).
- **Engine:** since v0.9.0 (2026-10-15) the default is a Rust engine built on Mago 1.57 ([k-engine], lines 1–3; [k-cargo], lines 9–16; [k-changelog], lines 15–25).
  - A missing Mago binary is an error, "never" a fallback to PHP.
  - Binaries cover macOS and Linux only; "Windows is outside the binary distribution" ([k-standalone], lines 23–46).
- **Naming:** Kit's `architecture-kit:plan` is an install plan. It recommends patterns from evidence and predicts generated-file changes ([k-readme], line 298; [k-planner]). It is not a plan in our glossary's sense.

### Declaration

Kit has three declarations.

- **`config/architectures.php`** ([k-config]; [k-readme], lines 28 and 208):
  - `enabled` lists the selected patterns as `Architecture` enum cases, or as slugs of custom architectures. That is why the package "is a runtime dependency", and installing it with `--dev` "is unsupported and is blocked".
  - It also holds `runtime`, `audit` settings and custom `rules`.
- **`audit.classification`**, inside that file, declares what existing directories contain ([k-readme], lines 1011–1042; [k-classmap], lines 17–70):
  - `roles` rows pair a `path`, `namespace` or `pattern` selector with a `role` and/or a `kind`.
  - `modules` rows give a `name` and a `path` or `namespace`, with optional `parent`.
  - `unknown_role` is `off`, `warn` or `error`.
  - Roles come from a closed list of 8: `domain`, `application`, `adapter`, `infrastructure`, `composition`, `port`, `unknown`, `test`.
  - Kinds come from a closed list of 19: `action`, `query`, `controller`, `model`, `service`, `job`, `listener`, `event`, `policy`, `request`, `resource`, `data`, `value-object`, `enum`, `exception`, `builder`, `port`, `provider`, `test` ([k-classmap], lines 19–21).
  - With declarations present, the file must be a static array. Kit reads it from the parsed syntax tree without executing it ([k-readme], line 1040; [k-classmap], lines 105–120).
  - Module relations are reported as intra-module, inter-module or unassigned. These are "not module dependency policy" ([k-readme], line 1038).
- **`.architecture-kit/target.json`** declares "the architecture you want for a selected area", separate from the current audit ([k-readme], lines 1065–1113; [k-target-def]):
  - **Fields:** `id`, `version`, `paths`, `classification` (same shape as above), `placements`, `dependencies`, `mode`, `only` and `reference`. Any other field throws ([k-target-def], line 24).
  - **`placements`:** rows of `kind`, optional `module`, optional `role`, and `paths`. A placement specific to a module takes precedence ([k-target-eval], lines 116–136).
  - **`dependencies`:** rows of `from` (a selector on `role`, `kind` and/or `module`) and `allow` (a list of selectors).
    - "Every matching policy must allow a dependency."
    - "An empty `allow` list forbids all recognized project dependencies from that selector" ([k-readme], line 1090; [k-target-eval], lines 71–91).
    - A row carries no reason or message field ([k-target-def], line 70).
  - **`mode`:** `info` (the default), `warn` or `block`. **`only`:** `all`, or `new` once a reference is accepted ([k-target-def], line 34).
  - **`reference`:** a path to a human-accepted snapshot. The report offers a `reference_candidate` with `accepted: false` and `accepted_by: null`. A person saves it, then sets `accepted: true` and a non-empty `accepted_by`. "No package command or MCP tool writes or accepts a reference" ([k-readme], line 1108; [k-target-ref], lines 13–31).
  - **The approval check is a string check:** `accepted_by` must be a non-empty string. Nothing verifies who wrote it ([k-target-ref], line 30).
  - **Limits:** paths must be literal directories with no `*`. Lists hold at most 1,000 rows. The file must be under 100 KB ([k-target-def], lines 87–146).

### How it assigns classes

- **Precedence** ([k-role-classifier], lines 21–41):
  - Test paths are always `test`.
  - Next come declared mappings, then interface-naming heuristics for `port`, then folder conventions.
- **Folder conventions** ([k-role-classifier], lines 112–143):
  - `Http/Controllers`, `Http/Requests` and `Http/Resources` are `adapter`; `Http/Integrations` is `infrastructure`.
  - `Providers` is `composition`; `Infrastructure` and `Adapters` are `infrastructure`.
  - `Actions`, `Services`, `Queries`, `Jobs` and `Listeners` are `application`.
  - `Models`, `Domain`, `Data`, `ValueObjects` and `Enums` are `domain`.
  - The first recognised segment of the path wins.
- **Kinds** also default from the last recognised folder segment ([k-role-classifier], lines 56–66).
- **Declared selectors** ([k-readme], line 1034; [k-classmap], lines 178–203):
  - Directory prefixes, namespace prefixes, or directory globs, where `*` is one segment and `**` is zero or more.
  - A literal selector beats a glob, and the most specific literal wins within its family.
  - Conflicting globs, or a path and namespace declaration that disagree, throw instead of guessing.
  - A class sits in exactly one role. Modules can nest, through `parent` or directory nesting.

### What counts as a dependency

- **Strong edges** ([k-graph-builder], lines 369–421 and 453–466; [k-edge]):
  - `extends`, `implements`, `trait`;
  - `parameter`, `return` and `property` types, including nullable, union and intersection types;
  - `new`, `static`, `instanceof`, `catch`, `attribute`, `class-constant`.
- **Weak edges:** `X::class` and the class arguments of Eloquent relation methods (`hasMany` and the like). They are `class-reference` and `eloquent-relation`, "visible to context, but do not independently trigger layer errors" ([k-readme], line 346).
- **Not counted:** "Imports alone are not dependencies" ([k-readme], line 344).
- **Duplicates are kept:** every occurrence becomes its own edge, with its path and line.
- **The current layer rule** fires only on strong edges between known project symbols ([k-layer-rule], lines 20–39). Its policy is hard-coded ([k-layer-policy], lines 11–24):
  - `domain` may not depend on `adapter`, `infrastructure` or `application`.
  - `application` and `port` may not depend on `adapter` or `infrastructure`.
  - `unknown`, `composition`, `infrastructure`, `adapter` and `test` may depend on anything.
- **The project-graph rules are a fixed list:** port bypass, layer dependency, namespace cycle and missing test ([k-ruleset], lines 18–27).

### What it can check before code exists, and how complete that is

- **`architecture-kit:file-rules <path>`**, and MCP `file-rules`, accept a planned path ([k-file-guidance], lines 43–83). For that path it returns:
  - the governing and shared architectures, each `enforced` or `advisory`;
  - rule names;
  - the classification;
  - when a target exists, the target report for the path ([k-file-guidance], line 75).
- **`architecture-kit:target <future path>`**, and MCP `architecture-target`, return `status: future_file` with the path's current classification and its `expected` role, kind, module and placement paths. It adds: "Namespace selectors require the new class namespace; no namespace is guessed" ([k-target], lines 159–172).
- **`architecture-kit:make <architecture> <Name> --agent`**, and MCP `scaffold`, return a file plan and skeleton for one element without writing it ([k-readme], lines 63–72).
  - Only 8 of the 18 patterns have templates: Actions, Services, Query Objects, Data Objects, Value Objects, API Resources, Form Requests and Thin Controllers ([k-scaffold], lines 26–37).
  - The skeletons are "built to pass this package's own audit".
- **`impact`** evaluates proposed changes to existing code: a new method signature, a delete, or a move or rename ([k-readme], lines 173–191 and 322–340). It reports `breaking`, `check` and `compatible` rows. `safe_to_change` is always false.
- **The `target` report itself** checks current code against a declared future architecture ([k-target-eval], lines 14–111):
  - Each existing class is evaluated under its expected classification.
  - Every existing strong edge is checked against `dependencies`, using the expected roles of both ends.
  - It produces `conformant`, `migration`, `requires_check` or `outside_scope` per element, plus a `migration_order` that puts dependencies before callers ([k-target], lines 175–207).
  - This is the nearest either package gets to "evaluate this code under a different architecture". But it evaluates only edges that already exist. An edge to a class outside the target becomes `requires_check` ([k-target-eval], lines 63–69).
- **Compared with validating a whole plan,** Kit can say where one planned file should go and which rules will govern it. It cannot:
  - take a list of planned classes;
  - judge their planned dependencies, since `context` throws `E_CONTEXT_SUBJECT_NOT_FOUND` for a symbol that doesn't exist ([k-context], line 124);
  - check a plan against the current audit's layer policy.

### What it pictures, and how

- **Nothing.** A search of `src/`, the README, `STANDALONE.md`, `docs/` and `resources/` finds no Mermaid, Graphviz, DOT, PlantUML, Cytoscape, SVG or HTML output (grep). The only diagrams in the repository are badges and a banner.
- **Graph tools return JSON:** `architecture-graph`, `path` and `reach` ([k-static-graph], lines 1–5; [k-mcp-graph]).

### Agent surface and output contracts

- **MCP:** a `laravel/mcp` server, `architecture-kit`, with 19 tools, each marked `#[IsReadOnly]` ([k-mcp-tools]; [k-readme], lines 853–879):
  - `enabled-architectures`, `architecture-rules`, `file-rules`, `scaffold`;
  - `architecture-context`, `architecture-search`, `architecture-graph`, `search`, `path`, `reach`, `impact`;
  - `audit-changed`, `guard`, `explain-finding`, `doctor`;
  - `architecture-target`, `revision-diff`, `public-api`, `plan-upgrade`.

  The server "does not regenerate files, install hooks, run migrations, write code, or mutate application data".
- **Hooks:** `architecture-kit:install-agents --hooks` adds a `Stop` hook for Claude Code and Codex. It runs `artisan architecture-kit:guard --changed --strict --json` ([k-readme], lines 515–525; [k-hooks], lines 125–146 and 276).
- **Output contracts** ([k-readme], lines 472–500):
  - `--agent` gives compact single-line JSON with a version field `v`, short keys, finding codes in place of messages, `trunc`/`total`/`shown`, and `next` hints.
  - `--schema` prints the JSON Schema of each command's output. For example, the target report's schema is draft 2020-12 with `additionalProperties: false` ([k-target-schema]).
  - Reports carry `analysis.status` (`none`, `complete`, `incomplete` or `limit`), freshness, explicit budgets, and totals marked as lower bounds ([k-readme], lines 117, 149 and 169).
- **Guidance:** generated `.ai/guidelines/architecture-kit.md` and `.ai/skills/architecture-kit-{architecture}/SKILL.md` for Laravel Boost ([k-readme], lines 210–260).

### Extension points: could another package consume or add to its declaration?

- **Consuming it:**
  - `target.json` is plain JSON and the easiest of the four declarations here for another tool to read or write.
  - But its meaning is tied to the closed role and kind lists. There is no input schema: `--schema` describes the report, not the file ([k-target-schema]).
  - Unknown fields throw ([k-target-def], lines 24 and 167).
  - `config/architectures.php` references the package's enums. When it holds classification declarations, Kit itself reads it statically. Another tool would need the same static parsing, or would have to execute it with Kit installed.
- **Adding to it:**
  - **Custom audit rules** implement `AuditRule::check(FileContext)`. They are per file, and their findings take part in suppression, baseline, `audit`, `guard`, hooks and MCP ([k-audit-rule]; [k-readme], lines 793–831).
  - Under the default Mago engine a rule must implement `PortableAuditRule` and read the neutral syntax model. Rules that use PhpParser need `analysis.engine = php`.
  - Project-graph rules are a fixed list, so a custom rule cannot add a dependency policy over the graph ([k-ruleset]).
  - **Custom architectures** are prose only. They generate guidance, and Kit "does not infer deterministic AST checks from prose" ([k-readme], lines 770–791).
  - Roles, kinds and the layer policy are constants ([k-classmap], lines 19–21; [k-layer-policy]).
- **No public PHP API is declared** for consumers (grep for `@api`). `UPGRADE.md` documents behaviour and contract changes across minor releases ([k-upgrade]).

### Maturity

- **Licence:** MIT. `LICENSE.md` is present ("Copyright (c) 2026 Gracjan Kubicki") ([k-license]), and `composer.json` declares MIT ([k-composer]).
- **History:**
  - The repository was created 2026-07-01.
  - 32 package releases, v0.1.0 (2026-07-06) to v0.9.3 (2026-10-15), four of them on 2026-10-15. There is also an `engine-v0.1.0` release.
  - 129 commits (GitHub API).
  - Classification declarations and `target.json` arrived in v0.7.0 (2026-10-07) ([k-changelog], lines 65–91). The Mago engine became the default in v0.9.0 (2026-10-15).
- **Maintainers:**
  - Packagist lists one maintainer, `taqie`.
  - GitHub contributors are `gk-hf` (98 commits) and `gracjankubicki` (12), plus `github-actions` and Dependabot bots.
- **Adoption** (Packagist and GitHub, 2026-10-17): 1,569 installs in total (372 in the last month), 2 favourites, 3 GitHub stars, 0 forks, 0 dependents.
- **Issues:**
  - 26 issues and pull requests, all closed. Every one was opened by `gk-hf`, `gracjankubicki` or Dependabot, and several are in Polish.
  - No issue shows outside use.
- **Dependencies:**
  - It requires `laravel/framework`, `laravel/mcp`, `laravel/prompts`, `nikic/php-parser` and others at runtime ([k-composer]).
  - It must be a runtime `require`, not `require-dev` ([k-readme], line 208).
  - The default engine is a native binary installed separately ([k-standalone], lines 23–46).
  - A standalone global install avoids touching the application ([k-standalone], lines 1–20).

### Where it falls short of plan validation plus visual review

- **No pictures at all.**
- **No plan input:** one future path at a time, with no dependency checks for planned classes.
- **Fixed architecture vocabulary.**
  - A project can map folders onto 8 roles and 19 kinds, but cannot name its own layers.
  - The current audit's layer policy cannot be changed.
  - Only `target.json` takes allow-lists, and only for checking a migration of existing code.
- **No impact preview** of a change to `target.json` or the classification. `revision-diff` compares code and configuration between revisions, but doesn't evaluate the same code under a proposed declaration (grep; [k-readme], lines 1115–1148).
- **No reason per rule** in `target.json` dependency rows.
- **The declarations are Kit's own**, read by no other tool, and not consumed by Pest.

## Side by side

Against the operations the WIP draft lists as candidates ([wip], "Operations"):

| Candidate operation | Sloppy | Laravel Architecture Kit |
| --- | --- | --- |
| `scan`: actual edges with file and line | No edge export. The graph JSON has role-level counts and one example; findings carry file and line | `architecture-graph`, `architecture-context` and `path`: class-level edges with kind, strength, path and line |
| `layer-of`: one class's layer | `sloppy architecture <Class>` or `sloppy_architecture`, for existing classes; shows the matcher and what else matched | `architecture-context` for existing symbols; `file-rules` and `target` also for a planned path |
| `check-dependency`: from class to class | None (grep) | None. `context` marks each existing edge `allowed` ([k-context], lines 136–160) |
| `suggest-layer`: from planned dependencies | `place`, from prose keywords; ignores dependencies | `target` and `file-rules` for a given path; `scaffold` for a given pattern and name |
| `validate-plan` | None | None |
| `impact` of a rule change | None | None for a declaration change. `impact` covers code changes; `revision-diff` covers revisions |
| `conformance`: planned against built | `sloppy diff`: findings the change introduced | `target`: current against target, with migration order. `revision-diff`: between revisions |
| `diagram` | `architecture graph`: Mermaid, DOT or JSON, roles only | None |

| Property | Sloppy | Laravel Architecture Kit |
| --- | --- | --- |
| Declaration format | PHP array, executed | PHP config with enums, read statically when it declares classification; JSON `target.json` |
| Units | Open, project-named roles; `{module}` modules | 8 fixed roles, 19 fixed kinds, named modules |
| Assignment | First matching role; 9 matcher keys plus `any`/`not` | Most specific selector; conflicts fail; folder conventions by default |
| Undeclared dependency | Allowed unless a policy forbids it. `may_depend_on` judges only targets that play a role | Current audit: fixed policy. Target: every matching rule must allow it |
| `X::class` | A dependency | A weak edge, not a layer error |
| Reason on a rule | `advice` per policy | None in `target.json` |
| Enforcer | Sloppy's analyser, PHPStan extension, Pest expectation | Kit's audit and guard |
| Human approval | None in the profile | `accepted_by` on a target reference |

## Corrections and additions to the survey

- **Sloppy's empty-match check is narrower** than the survey's "refusal to run a profile that matches nothing". Sloppy refuses a role with no matchers; a role that matches no class only gets a console warning ([s-roleparser], lines 71–77; [s-runner], lines 75–77).
- **Sloppy imports Deptrac YAML only**, not `deptrac.php`. The survey reports `deptrac.php` as Deptrac's default ([s-deptrac], line 26).
- **Kit's current layer policy is hard-coded.** `target.json` is the only place a project declares allowed dependencies ([k-layer-policy]; [k-target-def]).
- **Kit's `plan` command is an install plan,** not a plan of classes ([k-readme], line 298).
- **Kit is much broader than an architecture tool:** guidance, audits of Laravel patterns, impact, path, reach, revision and public-API analysis. It now defaults to a native Mago binary ([k-readme], lines 26 and 195; [k-standalone], lines 23–46).

## Options for laravel-visual-spec

These are options with their evidence, not a recommendation. Choosing between them belongs to a ticket with a human in the loop.

### Relative to Sloppy

- **Build on Sloppy:** ship laravel-visual-spec as a layer over Sloppy's profile, adding plan validation and pictures.
  - *For:*
    - MIT licence.
    - The most adopted of the two.
    - It already has open project-named roles with matchers aligned with Deptrac; allow and deny policies; `{module}` boundaries with public and shared surfaces; capabilities; and `covers`.
    - It has a role graph, an MCP server, hooks, and `init`, `import` and `prompt` for authoring.
    - Custom rules can read roles through an `@api` method ([s-matchkey]; [s-policy]; [s-boundaries]; [s-custom], lines 101–117).
  - *Against:*
    - The profile is PHP, executed, and closed to extra keys. A plan, a reason per edge or a layout can't be added without upstream changes ([s-loader], line 42; [s-profile], lines 76–85).
    - The classes that hold the profile carry no `@api` marker.
    - The allow-list ignores targets that play no role or that the index doesn't know, which is exactly what a plan contains ([s-policy], lines 103–105).
    - Enforcement is Sloppy's analyser, not a Pest plugin.
    - One maintainer; the architecture features are 13 days old; presets, matcher keys and capabilities are closed.
- **Interoperate with Sloppy:** keep a separate architecture spec and translate to or from Sloppy.
  - *For:*
    - `sloppy-architecture.php` is a plain returned array that a generator could write.
    - Sloppy already imports `deptrac.yaml`, so emitting Deptrac YAML reaches Sloppy without bespoke code ([s-deptrac]).
    - The `sloppy_architecture` MCP tool and `--format=json` expose the resolved roles and policies ([s-report], lines 150–190).
  - *Against:*
    - The JSON export can't be turned back into a profile, because matchers come out as prose ([s-pattern], lines 42–45).
    - The import is lossy for regular expressions and collectors with no equivalent ([s-config], lines 276–279).
    - Semantics differ: first-match roles; an allow-list that skips unclassified targets; `X::class` counted as a dependency. So the same declaration could pass in one tool and fail in the other.
- **Borrow ideas from Sloppy.**
  - *For:*
    - Deptrac-aligned matcher keys; first-match order with an "also matched" explanation ([s-map], lines 39–47).
    - `{module}` capture with `public` and `shared` ([s-boundaries]).
    - Precise capability detection, with precision chosen over recall ([s-capscan], lines 20–29).
    - `covers` with new-class-only reporting ([s-sl307]).
    - `advice` written for the agent.
    - Errors that name the key path.
    - `init` and `prompt` to draft a declaration from code or prose.
    - The invariant "a red edge in the picture is always a finding" ([s-graph], lines 10–17).
    - Hooks that report only what an edit introduced and block finishing once ([s-agents], lines 29–49).
  - *Against:* none found beyond the cost of designing these features ourselves.
- **Ignore Sloppy.**
  - *For:*
    - Its core is code-quality analysis, and the architecture part is an opt-in add-on.
    - It does no plan validation and draws no plan.
    - Its profile and enforcement both live inside its own engine.
  - *Against:*
    - Its agent loop already covers much of what the WIP sketches: placement, graph, MCP, hooks, rulesets and Deptrac import.
    - Its adoption (39,274 installs in five weeks) means users may already hold a profile.
    - Ignoring it would mean a Laravel user runs two different, disagreeing architecture declarations.

### Relative to Laravel Architecture Kit

- **Build on Kit:** read or extend `target.json` and the classification.
  - *For:*
    - MIT licence.
    - `target.json` is plain JSON with strict validation, `role`/`kind`/`module` selectors, "every matching rule must allow" semantics, and migration reporting against existing code.
    - Human-accepted references with fingerprints that bind them to the declaration.
    - Bounded, versioned agent JSON, with a schema for each command ([k-target-def]; [k-target-eval]; [k-target-ref]; [k-readme], lines 472–500).
  - *Against:*
    - No pictures.
    - A closed role and kind vocabulary, and a hard-coded current layer policy ([k-classmap], lines 19–21; [k-layer-policy]).
    - No input schema for `target.json`, and no reason field.
    - Only per-file custom rules; project-graph rules are fixed.
    - It must be a runtime dependency, and its default engine is a native binary with no Windows build.
    - 3 stars, 0 dependents, issues filed only by its own two accounts.
    - Pre-1.0, with frequent releases and documented breaking changes.
    - The architecture model is a small part of a 511-file package.
- **Interoperate with Kit:** emit or read `target.json` and classification rows from the architecture spec.
  - *For:*
    - `target.json` is JSON and needs no PHP to write.
    - Roles, kinds and modules map onto layer-like units.
    - `architecture-target` and `revision-diff` return structured, schema-described reports that a visual layer could draw ([k-target-schema]).
  - *Against:*
    - A spec with project-named layers cannot be expressed in 8 fixed roles.
    - Kit's strong and weak edge rules, and its conflict-fails assignment, differ from Sloppy's and possibly from Pest's.
    - Interop has to target a moving pre-1.0 contract with no outside users to protect it.
- **Borrow ideas from Kit.**
  - *For:*
    - Strong and weak edges (`::class` and Eloquent relation targets are weak) ([k-readme], line 346).
    - Path-and-line evidence on every edge.
    - `analysis.status`, lower-bound totals and freshness ([k-readme], lines 117 and 149).
    - A `--schema` for every agent command.
    - A declared target kept apart from current enforcement ("Target expectations do not enable, disable or suppress current audit rules"; [k-target], line 104).
    - `migration_order`.
    - `requires_check` for unknown dependencies.
    - The most specific selector winning, with conflicts failing ([k-classmap], lines 178–203).
    - Classification provenance.
    - "No namespace is guessed" for future files ([k-target], line 172).
    - An `accepted_by` reference that no tool can write ([k-target-ref]).
  - *Against:* none found beyond the cost of designing these features ourselves.
- **Ignore Kit.**
  - *For:*
    - Low adoption.
    - No pictures and no plan validation.
    - The architecture vocabulary is closed and Laravel-pattern-specific.
    - It must be a runtime dependency.
  - *Against:*
    - Its `target.json` plus reference workflow is the closest existing model of "declare intended structure, compare built code, approve with a named person", which laravel-visual-spec will need in some form.
    - Its agent-output discipline (bounded, schema-described, uncertainty explicit) is already proven in a Laravel package.

### Does either make laravel-visual-spec redundant?

- **No, on the evidence read.**
  - Neither validates a plan with several classes.
  - Neither judges dependencies between classes that don't exist yet.
  - Neither pictures a plan or planned-versus-built.
  - Neither offers a declaration that a separate Pest plugin could read with one meaning.
- **But both shrink the space the package would have to itself.** Placement queries, a role graph of actual edges (Sloppy), current-versus-target reports (Kit), MCP servers and before-finish hooks already exist in the Laravel world. Each came out within the last two weeks.

## Open questions surfaced

1. **Which interop formats, if any?** Should the architecture spec emit, or import, Sloppy's `sloppy-architecture.php`, Kit's `target.json`, or only `deptrac.yaml`, which Sloppy already imports? Each carries different semantics.
2. **How is a class assigned?** Ordered first-match roles (Sloppy), or most-specific-selector-wins with conflicts failing (Kit)? The choice changes what an assignment error looks like, and what "unassigned" means.
3. **Is `X::class` a dependency?** Sloppy counts it; Kit treats it and Eloquent relation targets as weak. Pest's arch plugin must agree with the architecture spec, so its behaviour here needs checking before the spec defines it.
4. **Open or closed layer vocabulary?** Project-named layers (Sloppy and the WIP), or fixed roles and kinds that tools can reason about (Kit)?
5. **Does an allow-list judge targets nobody classified?** Sloppy's `may_depend_on` skips classes that play no role. A plan's new classes always fall in that gap unless the plan assigns them.
6. **Is a target architecture in scope?** That is, a declared state the code hasn't reached yet, checked as a migration, with an accepted reference (Kit's `target.json`). Or does the architecture spec describe only the enforced present?
7. **Are capabilities in the architecture spec?** Sloppy shows that precise detection of `db`, `http`, `dispatch`, `env`, `container` and the rest is possible, but it is not class-level. The WIP principle "Architecture is class-level" and its bans on `app()` and `resolve()` meet here.
8. **Do we ask the upstream authors for extension points?** For example, Sloppy accepting extra keys or a data form of the profile, or Kit publishing an input schema or open roles. That is a human call, and it shapes whether "build on" is viable.
9. **Runtime or dev dependency?** Kit requires `require` because its config references enums. The WIP assumes `require-dev`. Should the architecture spec avoid any format that forces a runtime dependency?

## Sources

All links are pinned to the commits read: Sloppy `4379f3174b23ecfa24787dcc0421d3ad57d46b4c`, Laravel Architecture Kit `bcea42374626ec3eb14fe7b5b0cfbb823f5711f3` (tag v0.9.3). Maturity figures come from the GitHub API and from Packagist package JSON (`https://packagist.org/packages/heyosseus/sloppy.json`, `https://packagist.org/packages/gracjankubicki/laravel-architecture-kit.json`, and their `dependents.json`), read on 2026-10-17.

[prior-art]: https://github.com/kfriars/laravel-visual-spec/blob/research/prior-art/docs/research/prior-art.md
[wip]: ../wip-spec-2026-10-16.md

**Sloppy**

[s-tree]: https://github.com/Heyosseus/sloppy/tree/4379f3174b23ecfa24787dcc0421d3ad57d46b4c
[s-readme]: https://github.com/Heyosseus/sloppy/blob/4379f3174b23ecfa24787dcc0421d3ad57d46b4c/README.md
[s-composer]: https://github.com/Heyosseus/sloppy/blob/4379f3174b23ecfa24787dcc0421d3ad57d46b4c/composer.json
[s-license]: https://github.com/Heyosseus/sloppy/blob/4379f3174b23ecfa24787dcc0421d3ad57d46b4c/LICENSE.md
[s-changelog]: https://github.com/Heyosseus/sloppy/blob/4379f3174b23ecfa24787dcc0421d3ad57d46b4c/CHANGELOG.md
[s-config]: https://github.com/Heyosseus/sloppy/blob/4379f3174b23ecfa24787dcc0421d3ad57d46b4c/docs/configuration.md
[s-agents]: https://github.com/Heyosseus/sloppy/blob/4379f3174b23ecfa24787dcc0421d3ad57d46b4c/docs/agents.md
[s-json]: https://github.com/Heyosseus/sloppy/blob/4379f3174b23ecfa24787dcc0421d3ad57d46b4c/docs/json-output.md
[s-custom]: https://github.com/Heyosseus/sloppy/blob/4379f3174b23ecfa24787dcc0421d3ad57d46b4c/docs/custom-rules.md
[s-profile]: https://github.com/Heyosseus/sloppy/blob/4379f3174b23ecfa24787dcc0421d3ad57d46b4c/src/Architecture/Profile.php
[s-loader]: https://github.com/Heyosseus/sloppy/blob/4379f3174b23ecfa24787dcc0421d3ad57d46b4c/src/Architecture/ProfileLoader.php
[s-role]: https://github.com/Heyosseus/sloppy/blob/4379f3174b23ecfa24787dcc0421d3ad57d46b4c/src/Architecture/Role.php
[s-roleparser]: https://github.com/Heyosseus/sloppy/blob/4379f3174b23ecfa24787dcc0421d3ad57d46b4c/src/Architecture/RoleParser.php
[s-matchkey]: https://github.com/Heyosseus/sloppy/blob/4379f3174b23ecfa24787dcc0421d3ad57d46b4c/src/Architecture/MatchKey.php
[s-matcher]: https://github.com/Heyosseus/sloppy/blob/4379f3174b23ecfa24787dcc0421d3ad57d46b4c/src/Architecture/Matcher.php
[s-pattern]: https://github.com/Heyosseus/sloppy/blob/4379f3174b23ecfa24787dcc0421d3ad57d46b4c/src/Architecture/PatternMatcher.php
[s-glob]: https://github.com/Heyosseus/sloppy/blob/4379f3174b23ecfa24787dcc0421d3ad57d46b4c/src/Architecture/Glob.php
[s-classfacts]: https://github.com/Heyosseus/sloppy/blob/4379f3174b23ecfa24787dcc0421d3ad57d46b4c/src/Architecture/ClassFacts.php
[s-map]: https://github.com/Heyosseus/sloppy/blob/4379f3174b23ecfa24787dcc0421d3ad57d46b4c/src/Architecture/ArchitectureMap.php
[s-policy]: https://github.com/Heyosseus/sloppy/blob/4379f3174b23ecfa24787dcc0421d3ad57d46b4c/src/Architecture/Policy.php
[s-policyparser]: https://github.com/Heyosseus/sloppy/blob/4379f3174b23ecfa24787dcc0421d3ad57d46b4c/src/Architecture/PolicyParser.php
[s-capability]: https://github.com/Heyosseus/sloppy/blob/4379f3174b23ecfa24787dcc0421d3ad57d46b4c/src/Architecture/Capability.php
[s-capscan]: https://github.com/Heyosseus/sloppy/blob/4379f3174b23ecfa24787dcc0421d3ad57d46b4c/src/Architecture/CapabilityScanner.php
[s-boundaries]: https://github.com/Heyosseus/sloppy/blob/4379f3174b23ecfa24787dcc0421d3ad57d46b4c/src/Architecture/Boundaries.php
[s-presets]: https://github.com/Heyosseus/sloppy/blob/4379f3174b23ecfa24787dcc0421d3ad57d46b4c/src/Architecture/Presets.php
[s-scanner]: https://github.com/Heyosseus/sloppy/blob/4379f3174b23ecfa24787dcc0421d3ad57d46b4c/src/Architecture/DependencyScanner.php
[s-graph]: https://github.com/Heyosseus/sloppy/blob/4379f3174b23ecfa24787dcc0421d3ad57d46b4c/src/Architecture/DependencyGraph.php
[s-edge]: https://github.com/Heyosseus/sloppy/blob/4379f3174b23ecfa24787dcc0421d3ad57d46b4c/src/Architecture/RoleEdge.php
[s-placement]: https://github.com/Heyosseus/sloppy/blob/4379f3174b23ecfa24787dcc0421d3ad57d46b4c/src/Architecture/Placement.php
[s-snapshot]: https://github.com/Heyosseus/sloppy/blob/4379f3174b23ecfa24787dcc0421d3ad57d46b4c/src/Architecture/ArchitectureSnapshot.php
[s-brief]: https://github.com/Heyosseus/sloppy/blob/4379f3174b23ecfa24787dcc0421d3ad57d46b4c/src/Architecture/AgentBrief.php
[s-deptrac]: https://github.com/Heyosseus/sloppy/blob/4379f3174b23ecfa24787dcc0421d3ad57d46b4c/src/Architecture/DeptracImporter.php
[s-sl304]: https://github.com/Heyosseus/sloppy/blob/4379f3174b23ecfa24787dcc0421d3ad57d46b4c/src/Rules/Architecture/LayerViolationRule.php
[s-sl306]: https://github.com/Heyosseus/sloppy/blob/4379f3174b23ecfa24787dcc0421d3ad57d46b4c/src/Rules/Architecture/BoundaryViolationRule.php
[s-sl307]: https://github.com/Heyosseus/sloppy/blob/4379f3174b23ecfa24787dcc0421d3ad57d46b4c/src/Evidence/MisplacedClassSource.php
[s-runner]: https://github.com/Heyosseus/sloppy/blob/4379f3174b23ecfa24787dcc0421d3ad57d46b4c/src/Runner/ArchitectureRunner.php
[s-report]: https://github.com/Heyosseus/sloppy/blob/4379f3174b23ecfa24787dcc0421d3ad57d46b4c/src/Runner/ArchitectureReport.php
[s-ruleset]: https://github.com/Heyosseus/sloppy/blob/4379f3174b23ecfa24787dcc0421d3ad57d46b4c/src/Agent/RulesetGenerator.php
[s-mcp]: https://github.com/Heyosseus/sloppy/blob/4379f3174b23ecfa24787dcc0421d3ad57d46b4c/src/Mcp/McpServer.php
[s-place-tool]: https://github.com/Heyosseus/sloppy/blob/4379f3174b23ecfa24787dcc0421d3ad57d46b4c/src/Mcp/Tools/PlaceTool.php
[s-sloppy]: https://github.com/Heyosseus/sloppy/blob/4379f3174b23ecfa24787dcc0421d3ad57d46b4c/src/Sloppy.php
[s-pest]: https://github.com/Heyosseus/sloppy/blob/4379f3174b23ecfa24787dcc0421d3ad57d46b4c/pest/Expectations.php

**Laravel Architecture Kit**

[k-tree]: https://github.com/gracjankubicki/laravel-architecture-kit/tree/bcea42374626ec3eb14fe7b5b0cfbb823f5711f3
[k-readme]: https://github.com/gracjankubicki/laravel-architecture-kit/blob/bcea42374626ec3eb14fe7b5b0cfbb823f5711f3/README.md
[k-standalone]: https://github.com/gracjankubicki/laravel-architecture-kit/blob/bcea42374626ec3eb14fe7b5b0cfbb823f5711f3/STANDALONE.md
[k-upgrade]: https://github.com/gracjankubicki/laravel-architecture-kit/blob/bcea42374626ec3eb14fe7b5b0cfbb823f5711f3/UPGRADE.md
[k-changelog]: https://github.com/gracjankubicki/laravel-architecture-kit/blob/bcea42374626ec3eb14fe7b5b0cfbb823f5711f3/CHANGELOG.md
[k-composer]: https://github.com/gracjankubicki/laravel-architecture-kit/blob/bcea42374626ec3eb14fe7b5b0cfbb823f5711f3/composer.json
[k-license]: https://github.com/gracjankubicki/laravel-architecture-kit/blob/bcea42374626ec3eb14fe7b5b0cfbb823f5711f3/LICENSE.md
[k-config]: https://github.com/gracjankubicki/laravel-architecture-kit/blob/bcea42374626ec3eb14fe7b5b0cfbb823f5711f3/config/architectures.php
[k-static-graph]: https://github.com/gracjankubicki/laravel-architecture-kit/blob/bcea42374626ec3eb14fe7b5b0cfbb823f5711f3/docs/static-graph.md
[k-engine]: https://github.com/gracjankubicki/laravel-architecture-kit/blob/bcea42374626ec3eb14fe7b5b0cfbb823f5711f3/engine/README.md
[k-cargo]: https://github.com/gracjankubicki/laravel-architecture-kit/blob/bcea42374626ec3eb14fe7b5b0cfbb823f5711f3/engine/Cargo.toml
[k-target-def]: https://github.com/gracjankubicki/laravel-architecture-kit/blob/bcea42374626ec3eb14fe7b5b0cfbb823f5711f3/src/Target/TargetDefinition.php
[k-target]: https://github.com/gracjankubicki/laravel-architecture-kit/blob/bcea42374626ec3eb14fe7b5b0cfbb823f5711f3/src/Target/ArchitectureTarget.php
[k-target-eval]: https://github.com/gracjankubicki/laravel-architecture-kit/blob/bcea42374626ec3eb14fe7b5b0cfbb823f5711f3/src/Target/TargetEvaluation.php
[k-target-ref]: https://github.com/gracjankubicki/laravel-architecture-kit/blob/bcea42374626ec3eb14fe7b5b0cfbb823f5711f3/src/Target/TargetReference.php
[k-target-schema]: https://github.com/gracjankubicki/laravel-architecture-kit/blob/bcea42374626ec3eb14fe7b5b0cfbb823f5711f3/src/Target/TargetSchema.php
[k-classmap]: https://github.com/gracjankubicki/laravel-architecture-kit/blob/bcea42374626ec3eb14fe7b5b0cfbb823f5711f3/src/Classification/ClassificationMappings.php
[k-role-classifier]: https://github.com/gracjankubicki/laravel-architecture-kit/blob/bcea42374626ec3eb14fe7b5b0cfbb823f5711f3/src/Architecture/RoleClassifier.php
[k-layer-policy]: https://github.com/gracjankubicki/laravel-architecture-kit/blob/bcea42374626ec3eb14fe7b5b0cfbb823f5711f3/src/Audit/ProjectGraph/LayerPolicy.php
[k-layer-rule]: https://github.com/gracjankubicki/laravel-architecture-kit/blob/bcea42374626ec3eb14fe7b5b0cfbb823f5711f3/src/Audit/Rules/ProjectGraph/LayerDependencyRule.php
[k-ruleset]: https://github.com/gracjankubicki/laravel-architecture-kit/blob/bcea42374626ec3eb14fe7b5b0cfbb823f5711f3/src/Audit/ProjectGraph/ProjectRuleSet.php
[k-graph-builder]: https://github.com/gracjankubicki/laravel-architecture-kit/blob/bcea42374626ec3eb14fe7b5b0cfbb823f5711f3/src/Audit/ProjectGraph/ProjectGraphBuilder.php
[k-edge]: https://github.com/gracjankubicki/laravel-architecture-kit/blob/bcea42374626ec3eb14fe7b5b0cfbb823f5711f3/src/Audit/ProjectGraph/DependencyEdge.php
[k-context]: https://github.com/gracjankubicki/laravel-architecture-kit/blob/bcea42374626ec3eb14fe7b5b0cfbb823f5711f3/src/Context/ArchitectureContext.php
[k-file-guidance]: https://github.com/gracjankubicki/laravel-architecture-kit/blob/bcea42374626ec3eb14fe7b5b0cfbb823f5711f3/src/Guidance/FileGuidance.php
[k-audit-rule]: https://github.com/gracjankubicki/laravel-architecture-kit/blob/bcea42374626ec3eb14fe7b5b0cfbb823f5711f3/src/Audit/AuditRule.php
[k-scaffold]: https://github.com/gracjankubicki/laravel-architecture-kit/blob/bcea42374626ec3eb14fe7b5b0cfbb823f5711f3/src/Scaffolding/ScaffoldTemplates.php
[k-planner]: https://github.com/gracjankubicki/laravel-architecture-kit/blob/bcea42374626ec3eb14fe7b5b0cfbb823f5711f3/src/Planning/ArchitecturePlanner.php
[k-hooks]: https://github.com/gracjankubicki/laravel-architecture-kit/blob/bcea42374626ec3eb14fe7b5b0cfbb823f5711f3/src/Install/Hooks/HookWriter.php
[k-mcp-tools]: https://github.com/gracjankubicki/laravel-architecture-kit/tree/bcea42374626ec3eb14fe7b5b0cfbb823f5711f3/src/Mcp/Tools
[k-mcp-graph]: https://github.com/gracjankubicki/laravel-architecture-kit/blob/bcea42374626ec3eb14fe7b5b0cfbb823f5711f3/src/Mcp/Tools/ArchitectureGraph.php
