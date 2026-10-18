# Do the architectural mistakes that matter show up at class level?

Research input for the plan-validation half of laravel-visual-spec. Written 2026-10-17. Everything here is a finding, not a decision. "Architecture spec" and "plan" are used as defined in `GLOSSARY.md`. A plan lists new or changed classes and their intended dependencies, with no code.

The question is whether the architectural mistakes that matter on real Laravel apps show up as **class-level references** or **only inside method bodies**. Class-level means class A names class B, a vendor class or a function. Method-body means which method is called on a dependency that is otherwise allowed, the shape of a query, or control flow.

## Answer in brief

- **Rules teams already enforce are almost all class-level.**
  - Four of the eight apps have an architecture test suite: Pinkary, Vito, Advising App and Coolify.
  - Their 165 hand-written rules split C1 53, C2 38, C3 3 and C4 71.
  - 94 of those rules are about architecture. Of these, 91 (97%) are class-level references (C1+C2) and 3 (3%) are method-body (C3).
  - All three C3 rules are Vito's. Vito writes them as regex or string searches over source text, not as Pest arch expectations.
  - The 44 expectations in the Pest presets that Pinkary and Vito switch on contain no C3.
- **Mistakes teams actually fixed are mostly class-level, but less so than the rules.**
  - In the six reference apps, 252 commit diffs were read and 153 commits qualified as architectural fixes.
  - They split C1 59, C2 36, C3 58 and C4 0.
  - So **C1+C2 is 95 of 153 (62%) and C3 is 58 of 153 (38%).**
- **Most C3 mistakes have no realistic class-level stand-in.**
  - For 46 of the 58 C3 commits, no class-level rule that is realistic for that app would have flagged the code before the fix. That is 30% of all qualifying commits. "Realistic" here means few peers violate the rule at the app's pinned HEAD.
  - For 5 more, a realistic rule exists and the fix removed the reference it would forbid.
  - For 3, a realistic rule exists, but the fixed class still held the reference afterwards.
  - For 4, the rule is only partly realistic.
- **The C3 share varies by app:**

  | App | C3 share | Fact behind it |
  | --- | --- | --- |
  | Bagisto | 21% (6/29) | 12/131 controllers import a model at HEAD |
  | Koel | 29% (7/24) | |
  | Advising App | 31% (9/29) | |
  | Firefly III | 48% (10/21) | |
  | BookStack | 50% (15/30) | |
  | Coolify | 55% (11/20) | 223/250 Livewire components import `App\Models` |

- **Where C3 happens:** controllers 24 of 58, then models 8, repositories 7, Livewire components 5, jobs 3. The remaining 11 are spread across other roles.
- **Strongest C3 examples.** In each one, the class held only references its peers commonly hold, so its dependency list looked normal before the fix. What was wrong was what its methods did.
  - Coolify [`0540b2eae5`](https://github.com/coollabsio/coolify/commit/0540b2eae567b344661f26f4c154be77b1efc20f), a Claude co-authored commit:
    - **Before:** the `General` Livewire component deleted `SERVICE_FQDN_*` environment variables through `$this->application->environment_variables()->where(...)->delete()` when the build pack changed.
    - **Fix:** that logic moved to the `Application` model's `saving` hook.
    - **Why no class-level rule catches it:** the component still uses the model, and 223 of 250 Livewire components import `App\Models`.
  - BookStack [`546cfb0dcc`](https://github.com/BookStackApp/BookStack/commit/546cfb0dcc801c4fa6560ca0e6d18b4f1edd830f):
    - **Before:** 15 files (controllers, repos, tools) ran static `Page::visible()->find(...)` query scopes.
    - **Fix:** the scopes were replaced by an injected `PageQueries`.
    - **Why no class-level rule catches it:** `use BookStack\Entities\Models\Page` stayed in the files.
  - Bagisto [`8f67109880`](https://github.com/bagisto/bagisto/commit/8f671098805810501195711eac5583f5009e55d5) and [`abd3bba617`](https://github.com/bagisto/bagisto/commit/abd3bba6177656819a0f4cf21e8fb4853b5fe741):
    - **Before:** controllers fired the before/after domain events around repository calls.
    - **Fix:** the event firing moved into the repositories' `create`/`update` overrides.
    - **Why no class-level rule catches it:** 58 of 131 controllers use the `Event` facade at HEAD.
  - Firefly III [`b627d42160`](https://github.com/firefly-iii/firefly-iii/commit/b627d42160a5e5678318a06cf12d4811a5724951):
    - **Before:** repositories did their own persistence with `Bill::create()` and `$bill->delete()`.
    - **Fix:** persistence moved into `BillFactory` and the internal update and destroy services.
    - **Why no class-level rule catches it:** 30 of 32 repositories import `FireflyIII\Models`.
  - Advising App [`4624f6b02b`](https://github.com/canyongbs/advisingapp/commit/4624f6b02bd467c12a608bc05223c3ed7c946bc2):
    - **Before:** the `Form`, `Event` and `Survey` models each held about 75 lines of `replicate()`/`save()` logic.
    - **Fix:** that logic moved into `Duplicate*` actions.
    - **Why no class-level rule catches it:** no class-level rule separates the before and after states.
  - The C3 rules teams do write look like this one. Vito's [ConventionsTest.php:47](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/ConventionsTest.php#L47) says that code in two action folders that writes `GithubApp` or `Plugin` must also call `forgetVersion()`.
- **How the tags split:**

  | | Role | Location | Role across locations |
  | --- | --- | --- | --- |
  | Commits (153) | 103 (67%) | 41 (27%) | 9 (6%) |
  | Rules (94) | 81 (86%) | 12 (13%) | 1 (1%) |

  - C3 commits are nearly all about a role: 55 of 58.
  - C1 commits are mostly about a location: 35 of 59.
  - 34 of the 59 C1 commits are placement: a class sitting in the wrong folder, module or package, not a reference to another class.
- **AI agents:**
  - AI co-author trailers appear only since 2025. Coolify has 662 such commits, Koel 16, Advising App 14 (Copilot), Bagisto 3, Firefly III 2 and BookStack 0.
  - One case of AI-written code being restructured later was found. Koel's Claude co-authored [`8a4b9347e6`](https://github.com/koel/koel/commit/8a4b9347e6c7b8a759ef751e22dd774b732b2c76) added a static `App\Helpers\Network` helper. It was later made injectable (C3, reference form only) and later moved into `App\Services\Network`.
  - In Coolify the AI-co-authored commits are themselves doing restructuring: 5 of its 20 qualifying fixes carry Claude trailers.

## Classes, tags and coding conventions

**Classes** (exactly one per rule or commit):

- **C1 app-to-app reference.** A class in part X names a class in part Y, through an import, type hint, `new`, static call, extends, implements or trait use. A plan's dependency list would show it.
- **C2 external reference.** The class names a vendor or framework class, a facade or a global function, such as `DB::`, `Http::`, `env()`, `app()` or `request()`.
- **C3 method-body only.** The referenced class was legitimately in use already. The mistake was which methods were called on it, or what the body did.
- **C4 not architecture.** Naming, suffixes, `final`, strict types, debug calls, class shape or style.

**Tags** (C1 to C3 only):

- **Location:** a layer, domain, module or package as a place.
- **Role:** a kind of class wherever it sits. In default-skeleton apps a role folder such as `App\Http\Controllers` counts as a role.
- **Role across locations:** a role inside each of several modules, packages or domains.
- **App-wide:** function bans that apply to all of `App` are tagged `location (app-wide)` and counted as location.

**Kinds** (commits only):

- **placement:** the fix moved a class to another folder, module or package.
- **dependency:** the class the offending code was in before the fix (the "from-class") lost a reference.
- **body:** only method calls or body logic changed, and every reference was one its peers normally hold.
- **form:** the same class is named before and after. Only the reference form changed, for example a static call or `new` became injection or `app(X::class)`.

**Conventions used to keep the counts mechanical:**

- **One rule is one assertion as written.**
  - A chained `->toExtend(...)->toHaveSuffix(...)` counts as two rules.
  - One assertion over a list of targets (`->and(...)` or an array) counts as one rule.
  - A per-module loop that generates rules counts as one rule.
  - Each `it()`/`test()` block counts as one rule.
  - Vito's four tests about its own exception registry are not counted.
- **Mixed targets take the highest architecture class: C1 over C2 over C4.** For example, `not->toUse([...])` with both an app namespace and a vendor class is C1. These rows are marked "mixed": 15 C1 rules and 1 C2 rule. Mixing never moves a rule between C1+C2 and C3.
- **"Folder X holds only subclasses of Y" counts as a reference to Y.** This covers `toExtend` and `toImplement`. It is C2 for a vendor Y and C1 for an app Y, and is marked "placement": 21 C2 rules and 12 C1 rules. Checks on kind alone (`toBeEnums`, `toBeTraits`, `toBeInterfaces`) are class shape, so C4.
- **Function bans:**
  - Debug and security bans with no location are C4. This covers Pest's `php` and `security` presets, Coolify's `rand` bans and Vito's credential-logging check.
  - Bans with an architectural reason are C2: `env()`, `sleep()` banned for testability, `request()` allowed only in Http, and the Process component banned for remote work.
- **The C3 test for commits.** A commit is C3 when two things hold:
  - Before the fix, the from-class held only references its peers legitimately use.
  - After the fix, it still names that kind of thing.

  It is C1 or C2 in either of these cases:
  - The fix removed a reference the role does not normally hold, as judged from counts at HEAD.
  - Moving or removing that reference was the commit's point.
- **Side-effect buckets for C3** (column "Bucket" in the C3 table):
  - **A:** a class-level rule realistic at HEAD would have flagged the code before the fix, and the fix removed that reference.
  - **B:** such a rule exists, but the from-class still held the reference after the fix, often as a stale import.
  - **P:** the rule is only partly realistic. Roughly 5 to 20% of peers violate it, or only a call-form rule fits.
  - **N:** no realistic class-level rule exists.

## Method

- **Primary sources only:** the apps' own repositories at pinned commits, and Pest's preset source.
  - Each app was cloned with `--filter=blob:none` into its own empty directory and checked out at its pin.
  - Nothing was run. No composer, npm, artisan, tests or app code. Cloned content was treated as data.
  - Advising App is under the Elastic License 2.0, so only short tokens are quoted from it.
- **Rules.**
  - Every `arch()`, `it()` and `test()` in each app's test directories that asserts on structure was read.
  - Files were found by grepping for `arch(`, `->preset()`, `toUseNothing`, `toOnlyBeUsedIn`, `toBeUsedInNothing`, `not->toUse` and `toOnlyUse`.
  - Pest presets were read from `pestphp/pest` tag `v5.3.2`. Pinkary locks v5.3.2 and Vito locks v5.3.0; `src/ArchPresets` has no diff between the two tags.
  - Koel, BookStack, Bagisto and Firefly III have no architecture test suite.
  - Not counted:
    - Firefly III's static-analysis switches (ergebnis `noParameterWithContainerTypeDeclaration`, mago `no-request-all`).
    - Larastan's default `env()` rule.
    - Prose agent instructions: Koel `CLAUDE.md`/`AGENTS.md`, Coolify `AGENTS.md`, Bagisto `CLAUDE.md`/`AGENTS.md`, Firefly III `agents.md`, Advising App `.ai/rules/*.md` and Vito `.github/instructions/*.md`.
- **Commits.** The funnel had five steps:
  1. Full commit messages were searched with `git log --no-merges -i -E --grep` for: move, extract, decoupl, refactor, layer, circular, "out of (the) controller", "into (an) action/service/job", repositor, dependenc, inject, "don't use", boundar, module.
  2. Subjects were kept only if they matched a word-boundary form of the same words, so "remove" no longer counts as "move". Dependency bumps, translations, CI and changelog commits were dropped.
  3. Commits were kept only if they touch non-test backend PHP. Tests, migrations, views, config, routes and lang were excluded.
  4. Six miner agents, one per app, did the reading:
     - They screened every subject left after step 3 and ran extra searches (see the funnel table).
     - They read diffs and classified each one with the same written rubric: the definitions above, a qualify/not-qualify list, and three calibration commits from Koel.
     - A commit qualified only if its point was where code lives or what it depends on. Bug fixes, features, in-place renames, style, framework upgrades and de-duplication inside one layer did not qualify.
  5. Lead audit:
     - Every row was read.
     - The diffs behind the featured examples and the calibration commits were checked by hand.
     - 3 rows were reclassified to C3 (form): [`65765e8357`](https://github.com/koel/koel/commit/65765e83572dd18d949db3fe55f6cc186231db90), [`835a421909`](https://github.com/firefly-iii/firefly-iii/commit/835a421909fc3069542dcc41f47e77884768ae0d) and [`1284f9cecc`](https://github.com/firefly-iii/firefly-iii/commit/1284f9cecc3b5205c198f1d5a1fb3a583be3e5ff). Each names the same class before and after.
     - 1 parallel-branch duplicate was dropped: Firefly III `745e0aa525`.
     - All 252 SHAs were verified to be commits reachable from each app's pinned HEAD.
- **Realism.** For each C3 commit, the miner counted how many peers (same role or location) hold the reference at the pinned HEAD, for example 223/250 Livewire components import `App\Models`. Those counts are in the C3 table.
- **AI trailers.**
  - Full messages were scanned for `Co-Authored-By` trailers or "Generated with" lines naming an agent, and for agent author accounts.
  - For the backend PHP files those commits added, later commits that deleted or moved the same path were looked for.
  - This tracks whole files only, not logic later moved out of AI-written methods.

## Sources

Links are `https://github.com/<repo>/tree/<sha>`; tables link to files at these SHAs and to commits.

| Key | Repo | Pinned commit | Role in this study |
| --- | --- | --- | --- |
| koel | koel/koel | `7c34114c6e8d099278c732b85853f720c2546d82` | reference app, history mined |
| coolify | coollabsio/coolify | `6964714b5da26d600621eb137c9845e6240f7d52` | reference app, history mined, arch suite |
| bookstack | BookStackApp/BookStack | `d3db80004b51a395a9dabdd406f5fbda3d4d76e4` | reference app, history mined |
| advising | canyongbs/advisingapp | `d945da2b74325e5ecdbc3cc7210a46768ea4fe1e` | reference app, history mined, arch suite |
| bagisto | bagisto/bagisto | `748d3ea117dcf121bfbf57a4d8d996c81144510f` | reference app, history mined |
| firefly | firefly-iii/firefly-iii | `b9c31006e8d3008df3d12e23de86b66088c9480e` | reference app, history mined |
| pinkary | pinkary-project/pinkary.com | `f4a528d4e5f46045aa3175d117615c0e1c14270b` | arch suite |
| vito | vitodeploy/vito | `8598e4dc0bce12d2b2f915766a100b445b7d9e22` (default branch `4.x` HEAD on 2026-10-17; commit dated 2026-10-07) | arch suite |
| pest | pestphp/pest | tag `v5.3.2` | arch presets |

## Totals

Rules (hand-written, one row per assertion):

| App | C1 | C2 | C3 | C4 | Total | C1+C2 share of architecture rules |
| --- | --- | --- | --- | --- | --- | --- |
| Pinkary | 18 | 17 | 0 | 36 | 71 | 100% |
| Vito | 35 | 17 | 3 | 32 | 87 | 95% |
| Advising App | 0 | 4 | 0 | 1 | 5 | 100% |
| Coolify | 0 | 0 | 0 | 2 | 2 | - |
| **All** | **53** | **38** | **3** | **71** | **165** | **97%** |

Pest presets that Pinkary and Vito use (counted separately, because they are Pest's rules, not the teams'):

| Preset | C1 | C2 | C3 | C4 | Total | Used by |
| --- | --- | --- | --- | --- | --- | --- |
| laravel | 2 | 18 | 0 | 22 | 42 | Pinkary |
| php | 0 | 0 | 0 | 1 | 1 | Pinkary, Vito |
| security | 0 | 0 | 0 | 1 | 1 | Pinkary, Vito |

Mistakes fixed (qualifying commits):

| App | Diffs read | Qualifying | C1 | C2 | C3 | C4 | C1+C2 share |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Koel | 49 | 24 | 9 | 8 | 7 | 0 | 71% |
| Coolify | 41 | 20 | 3 | 6 | 11 | 0 | 45% |
| BookStack | 40 | 30 | 10 | 5 | 15 | 0 | 50% |
| Advising App | 44 | 29 | 14 | 6 | 9 | 0 | 69% |
| Bagisto | 40 | 29 | 18 | 5 | 6 | 0 | 79% |
| Firefly III | 38 | 21 | 5 | 6 | 10 | 0 | 52% |
| **All** | **252** | **153** | **59** | **36** | **58** | **0** | **62%** |

Tags by class (commits):

| Class | Role | Location | Role across locations | Kinds |
| --- | --- | --- | --- | --- |
| C1 (59) | 18 | 35 | 6 | placement 34, dependency 25 |
| C2 (36) | 30 | 3 | 3 | dependency 29, placement 7 |
| C3 (58) | 55 | 3 | 0 | body 55, form 3 |

Tags (rules): C1 role 48 / location 5; C2 role 31 / location 6 / role across locations 1; C3 role 2 / location 1.

Role across locations shows up only in the two modular apps' commits (Advising App 4, Bagisto 5) and in one Advising App rule.

C3 side-effect buckets (58 commits): A 5, B 3, P 4, N 46.

Sensitivity of the headline commit share:

| If | C1+C2 share of commits |
| --- | --- |
| As counted | 62% (95/153) |
| Bucket-A commits are counted as class-level | 65% (100/153) |
| Buckets A, B and P are counted as class-level | 70% (107/153) |
| Placement commits (41) are left out, keeping only reference-type C1/C2 | 48% (54/112) |


## Evidence 1: rules teams already enforce

### What the suites contain

- **Pinkary** has 71 hand-written rules plus the `php`, `security` and `laravel` presets.
  - Almost every architecture rule is a role folder's allow-list, either `->toOnlyBeUsedIn([...])` or `->not->toBeUsed()`, or a role folder's base class.
  - The allow-lists move with features. Since 2024, at least 9 commits widened an allow-list or ignore list under `tests/Arch`, and 7 of them are feature commits. Examples:
    - [`814fff2900`](https://github.com/pinkary-project/pinkary.com/commit/814fff2900845c5f1994c6bbc7dcc34b6acf26ee), "permit use of models in App\EventActions namespace".
    - [`240ce16914`](https://github.com/pinkary-project/pinkary.com/commit/240ce16914579f2fc80507bdfbfefbd6fbc6bbb0), which adds `App\Policies` to the enum allow-list.
    - [`eeb8a060fb`](https://github.com/pinkary-project/pinkary.com/commit/eeb8a060fb35216692493b4cb978b3438013e71a) and [`b9f9c08e0f`](https://github.com/pinkary-project/pinkary.com/commit/b9f9c08e0f51225b96ed11d974d116eab27fe1e7).
  - Entries were removed from the lists only when a class was deleted or renamed, for example [`59048707`](https://github.com/pinkary-project/pinkary.com/commit/590487070aa355d3a2131e455b9b54f8ce7ffdd5), which renamed `App\EventActions` to `App\Actions`.
- **Vito** has 87 hand-written rules plus the `php` and `security` presets. The suite was added on 2026-08-01 in [#1223](https://github.com/vitodeploy/vito/commit/7651bb41e5b0f8a69eac54c818b02e0038b49fc4), "Architecture tests: enforce documented standards".
  - Its class-level rules state responsibilities. For example, it has these exceptions to "models do not depend on actions": "Invokes business logic from the model layer instead of from an Action, Job or Controller" (5 models).
  - It also has these exceptions to "controllers do not dispatch jobs directly": "the orchestration is invisible to the Action layer" (3 controllers).
  - Two files note that "Pest's negative dependency expectations silently pass when given an array of source namespaces", so Vito writes one expectation per source namespace. See [ExtensionPointsTest.php:17](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/ExtensionPointsTest.php#L17) and [SecurityTest.php:10](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/SecurityTest.php#L10).
  - Its 3 C3 rules are `it()` blocks that search file contents with regexes or plain string matches:
    - a write to `GithubApp`/`Plugin` must be paired with `forgetVersion()`;
    - jobs that use `UniqueQueue` must contain `$this->run(`;
    - resources must not contain `parent::toArray`.
- **Vito keeps a single registry of exceptions, with reasons** ([tests/ArchTestCase.php](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/ArchTestCase.php#L28)). Each entry is a known, tolerated violation in today's code:

  | Exception key | Rule class | Entries |
  | --- | --- | --- |
  | `models.not-on-abstract-model`, `models.foreign-base-class` | C1 placement | 7 + 2 |
  | `policies.user-owned` (must use `HasRolePolicies`) | C1 | 9 |
  | `models.calling-actions` | C1 | 5 |
  | `http.controllers-dispatching-jobs` | C1 | 3 |
  | `jobs.without-unique-queue` | C1 | 3 |
  | `extension-points.non-tooling-classes`, `extension-points.non-channel-classes` | C1 placement | 2 + 1 |
  | `http.resources-calling-actions` | C1 | 1 |
  | `actions.accepting-request`, `http.controllers-validating`, `http.controllers-querying`, `foundation.blocking-sleep`, `security.local-process` | C2 | 1 each |
  | `foundation.non-throwable-in-exceptions` | C2 placement | 1 |
  | `conventions.writes-without-bootstrap-bust` | C3 | 4 |
  | `foundation.var-export`, `foundation.non-cryptographic-hashing`, `foundation.local-shell`, `foundation.assert`, `foundation.missing-return-type`, `jobs.without-failure-handler`, `routing.public-endpoints` | C4 | 1, 3, 2, 1, 3, 3, 22 |

- **Advising App** has 5 rules.
  - Three concern which UUID trait models use. One of them is generated once per module, across 37 modules.
  - One forbids using the vendor `OwenIt\Auditing\Auditable` trait directly in `App\Models` and six modules. The app's own wrapper concern must be used instead.
  - One requires the permission string `product_admin.` to appear only in migrations (C4).
- **Coolify** has 2 rules, both bans on insecure PRNG functions. Its other source-reading tests (for example `tests/Unit/DatabaseHealthcheckCommandInjectionTest.php`) check specific behaviour in specific files and are not counted as rules.

### Rule tables

#### Pinkary (71 hand-written rules)

| # | Where | Rule | Class | Tag | Note |
| --- | --- | --- | --- | --- | --- |
| 1 | [ArchTest.php:14](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/ArchTest.php#L14) | App uses strict types | C4 |  |  |
| 2 | [ArchTest.php:18](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/ArchTest.php#L18) | App classes are final (Type ignored) | C4 |  |  |
| 3 | [ArchTest.php:26](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/ArchTest.php#L26) | App classes are not abstract | C4 |  |  |
| 4 | [ArchTest.php:34](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/ArchTest.php#L34) | App classes are readonly, except 12 role namespaces | C4 |  |  |
| 5 | [ArchTest.php:55](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/ArchTest.php#L55) | App classes extend nothing, except 13 role namespaces | C4 |  |  |
| 6 | [ArchTest.php:78](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/ArchTest.php#L78) | App properties are documented | C4 |  |  |
| 7 | [ArchTest.php:79](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/ArchTest.php#L79) | App methods are documented | C4 |  |  |
| 8 | [ConsoleTest.php:8](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/ConsoleTest.php#L8) | App\Console\Commands extend Illuminate Command | C2 | role | placement |
| 9 | [ConsoleTest.php:9](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/ConsoleTest.php#L9) | Commands have suffix Command | C4 |  |  |
| 10 | [ConsoleTest.php:10](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/ConsoleTest.php#L10) | Commands have handle() | C4 |  |  |
| 11 | [ConsoleTest.php:11](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/ConsoleTest.php#L11) | Commands implement only SignalableCommandInterface | C2 | role |  |
| 12 | [ConsoleTest.php:12](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/ConsoleTest.php#L12) | Commands are not used by any app class | C1 | role |  |
| 13 | [ContractsTest.php:7](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/ContractsTest.php#L7) | App\Contracts are interfaces | C4 |  |  |
| 14 | [EnumsTest.php:7](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/EnumsTest.php#L7) | App\Enums are enums | C4 |  |  |
| 15 | [EnumsTest.php:8](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/EnumsTest.php#L8) | Enums extend nothing | C4 |  |  |
| 16 | [EnumsTest.php:9](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/EnumsTest.php#L9) | Enums use nothing (no app or vendor dependency) | C1 | role | mixed: also bars vendor |
| 17 | [EnumsTest.php:10](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/EnumsTest.php#L10) | Enums have toArray() | C4 |  |  |
| 18 | [EnumsTest.php:11](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/EnumsTest.php#L11) | Enums only used in Commands, Requests, Responses, Livewire, Models, Policies | C1 | role |  |
| 19 | [ExceptionsTest.php:7](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/ExceptionsTest.php#L7) | App\Exceptions implement Throwable | C2 | role | placement |
| 20 | [ExceptionsTest.php:8](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/ExceptionsTest.php#L8) | Exceptions only used in Commands, Controllers, Livewire, Services | C1 | role |  |
| 21 | [FactoriesTest.php:7](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/FactoriesTest.php#L7) | Database\Factories extend Eloquent Factory | C2 | role | placement |
| 22 | [FactoriesTest.php:9](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/FactoriesTest.php#L9) | Factories use app trait RefreshOnCreate | C1 | role | must-use |
| 23 | [FactoriesTest.php:10](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/FactoriesTest.php#L10) | Factories have definition() | C4 |  |  |
| 24 | [FactoriesTest.php:12](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/FactoriesTest.php#L12) | Factories only used in App\Models | C1 | role |  |
| 25 | [GlobalTest.php:5](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/GlobalTest.php#L5) | dd, dump, ray, die, var_dump, sleep, dispatch, dispatch_sync never used | C2 | location (app-wide) | mixed: 5 of 8 are debug functions |
| 26 | [GlobalTest.php:9](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/GlobalTest.php#L9) | session(), auth(), request() only in App\Http, App\Rules, App\Livewire, one job, Autocomplete types | C2 | location |  |
| 27 | [GlobalTest.php:21](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/GlobalTest.php#L21) | App\Actions have handle() | C4 |  |  |
| 28 | [GlobalTest.php:22](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/GlobalTest.php#L22) | Actions are final | C4 |  |  |
| 29 | [GlobalTest.php:23](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/GlobalTest.php#L23) | Actions are readonly | C4 |  |  |
| 30 | [GlobalTest.php:24](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/GlobalTest.php#L24) | Actions extend nothing | C4 |  |  |
| 31 | [HttpTest.php:7](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/HttpTest.php#L7) | Controllers extend nothing | C4 |  |  |
| 32 | [HttpTest.php:8](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/HttpTest.php#L8) | Controllers are not used by any app class | C1 | role |  |
| 33 | [HttpTest.php:12](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/HttpTest.php#L12) | Middleware have handle() | C4 |  |  |
| 34 | [HttpTest.php:13](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/HttpTest.php#L13) | Middleware use Illuminate Request | C2 | role | must-use |
| 35 | [HttpTest.php:14](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/HttpTest.php#L14) | Middleware are not used by any app class | C1 | role |  |
| 36 | [HttpTest.php:18](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/HttpTest.php#L18) | App\Http\Requests extend FormRequest | C2 | role | placement |
| 37 | [HttpTest.php:19](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/HttpTest.php#L19) | Requests have rules() | C4 |  |  |
| 38 | [HttpTest.php:20](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/HttpTest.php#L20) | Requests are used in App\Http\Controllers | C1 | role |  |
| 39 | [HttpTest.php:24](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/HttpTest.php#L24) | App\Http\Responses extend nothing | C4 |  |  |
| 40 | [HttpTest.php:25](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/HttpTest.php#L25) | Responses have toResponse() | C4 |  |  |
| 41 | [HttpTest.php:26](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/HttpTest.php#L26) | Responses are used in App\Http\Controllers | C1 | role |  |
| 42 | [JobsTest.php:7](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/JobsTest.php#L7) | Jobs have handle() | C4 |  |  |
| 43 | [JobsTest.php:8](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/JobsTest.php#L8) | Jobs have a constructor | C4 |  |  |
| 44 | [JobsTest.php:9](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/JobsTest.php#L9) | Jobs extend nothing | C4 |  |  |
| 45 | [JobsTest.php:10](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/JobsTest.php#L10) | Jobs implement ShouldQueue | C2 | role | placement |
| 46 | [LivewireTest.php:7](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/LivewireTest.php#L7) | App\Livewire are classes | C4 |  |  |
| 47 | [LivewireTest.php:9](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/LivewireTest.php#L9) | Livewire classes extend Livewire\Component | C2 | role | placement |
| 48 | [LivewireTest.php:11](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/LivewireTest.php#L11) | Livewire classes have render() | C4 |  |  |
| 49 | [LivewireTest.php:13](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/LivewireTest.php#L13) | Livewire classes only used in Controllers, App\Http\Livewire, AppServiceProvider | C1 | role |  |
| 50 | [LivewireTest.php:19](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/LivewireTest.php#L19) | Livewire classes do not use redirect(), to_route(), back() | C2 | role |  |
| 51 | [LivewireTest.php:23](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/LivewireTest.php#L23) | App\Livewire\Concerns are traits | C4 |  |  |
| 52 | [MailablesTest.php:7](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/MailablesTest.php#L7) | Mailables have a constructor | C4 |  |  |
| 53 | [MailablesTest.php:8](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/MailablesTest.php#L8) | App\Mail extend Mailable | C2 | role | placement |
| 54 | [ModelsTest.php:7](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/ModelsTest.php#L7) | Models have casts() | C4 |  |  |
| 55 | [ModelsTest.php:10](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/ModelsTest.php#L10) | App\Models extend Eloquent Model | C2 | role | placement |
| 56 | [ModelsTest.php:13](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/ModelsTest.php#L13) | Models only used in 17 listed namespaces | C1 | role |  |
| 57 | [ModelsTest.php:33](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/ModelsTest.php#L33) | Every model has a factory (13 models) | C4 |  |  |
| 58 | [ModelsTest.php:43](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/ModelsTest.php#L43) | Every *_at attribute is cast to datetime | C4 |  |  |
| 59 | [NotificationsTest.php:7](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/NotificationsTest.php#L7) | Notifications have a constructor | C4 |  |  |
| 60 | [NotificationsTest.php:8](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/NotificationsTest.php#L8) | App\Notifications extend Notification | C2 | role | placement |
| 61 | [NotificationsTest.php:9](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/NotificationsTest.php#L9) | Notifications only used in 2 action folders, Commands, Controllers, Observers, Livewire\Concerns, one component | C1 | role |  |
| 62 | [ProvidersTest.php:7](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/ProvidersTest.php#L7) | App\Providers extend ServiceProvider | C2 | role | placement |
| 63 | [ProvidersTest.php:8](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/ProvidersTest.php#L8) | Providers are not used by any app class | C1 | role |  |
| 64 | [RulesTest.php:7](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/RulesTest.php#L7) | App\Rules extend nothing | C4 |  |  |
| 65 | [RulesTest.php:8](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/RulesTest.php#L8) | Rules implement ValidationRule | C2 | role | placement |
| 66 | [RulesTest.php:9](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/RulesTest.php#L9) | Rules only used in Controllers, Requests, Livewire | C1 | role |  |
| 67 | [ServicesTest.php:7](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/ServicesTest.php#L7) | App\Services\ParsableContentProviders implement app ParsableContentProvider contract | C1 | role | placement |
| 68 | [ServicesTest.php:8](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/ServicesTest.php#L8) | ParsableContentProviders only used in App\Services | C1 | location |  |
| 69 | [ViewTest.php:7](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/ViewTest.php#L7) | App\View\Components extend View Component | C2 | role | placement |
| 70 | [ViewTest.php:8](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/ViewTest.php#L8) | View components have render() | C4 |  |  |
| 71 | [ViewTest.php:9](https://github.com/pinkary-project/pinkary.com/blob/f4a528d4e5f46045aa3175d117615c0e1c14270b/tests/Arch/ViewTest.php#L9) | View components are not used by any app class | C1 | role |  |

#### Vito (87 hand-written rules)

| # | Where | Rule | Class | Tag | Note |
| --- | --- | --- | --- | --- | --- |
| 1 | [ActionsTest.php:5](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/ActionsTest.php#L5) | Actions do not use Inertia, RedirectResponse, Redirector | C2 | role |  |
| 2 | [ActionsTest.php:13](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/ActionsTest.php#L13) | Actions do not use Illuminate Request or request() (1 exception) | C2 | role |  |
| 3 | [ActionsTest.php:18](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/ActionsTest.php#L18) | Actions do not use App\Http\Controllers | C1 | role |  |
| 4 | [ConventionsTest.php:18](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/ConventionsTest.php#L18) | SocketEvent not used in DTOs, Enums, Http\Resources, Policies, ValidationRules | C1 | location |  |
| 5 | [ConventionsTest.php:28](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/ConventionsTest.php#L28) | DTOs do not use Controllers, Jobs, DB, Http facade, Inertia | C1 | role | mixed: also bars facades/vendor |
| 6 | [ConventionsTest.php:38](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/ConventionsTest.php#L38) | Listeners have handle() | C4 |  |  |
| 7 | [ConventionsTest.php:43](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/ConventionsTest.php#L43) | Console commands do not use Controllers or Inertia | C1 | role | mixed: also bars vendor |
| 8 | [ConventionsTest.php:47](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/ConventionsTest.php#L47) | Code in Actions/GithubApp and Actions/Plugins that writes GithubApp or Plugin (create/update/delete/save) must also call forgetVersion() (4 exceptions) | C3 | location | method-body regex over source text |
| 9 | [EnumsTest.php:6](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/EnumsTest.php#L6) | App\Enums are enums | C4 |  |  |
| 10 | [EnumsTest.php:10](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/EnumsTest.php#L10) | Enums implement app VitoEnum contract | C1 | role |  |
| 11 | [EnumsTest.php:14](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/EnumsTest.php#L14) | Enums do not use Actions, Http, Jobs, Models, DB facade | C1 | role | mixed: also bars DB facade |
| 12 | [EnumsTest.php:24](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/EnumsTest.php#L24) | Every enum is string-backed | C4 |  |  |
| 13 | [EnumsTest.php:38](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/EnumsTest.php#L38) | Every enum case has colour and label | C4 |  |  |
| 14 | [ExtensionPointsTest.php:23](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/ExtensionPointsTest.php#L23) | App\ServerProviders extend app AbstractProvider | C1 | role | placement |
| 15 | [ExtensionPointsTest.php:28](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/ExtensionPointsTest.php#L28) | App\DNSProviders extend app AbstractDNSProvider | C1 | role | placement |
| 16 | [ExtensionPointsTest.php:33](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/ExtensionPointsTest.php#L33) | App\SourceControlProviders extend app abstract | C1 | role | placement |
| 17 | [ExtensionPointsTest.php:38](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/ExtensionPointsTest.php#L38) | App\StorageProviders extend app abstract | C1 | role | placement |
| 18 | [ExtensionPointsTest.php:43](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/ExtensionPointsTest.php#L43) | App\Services extend app AbstractService | C1 | role | placement |
| 19 | [ExtensionPointsTest.php:48](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/ExtensionPointsTest.php#L48) | App\SiteTypes extend app abstract | C1 | role | placement |
| 20 | [ExtensionPointsTest.php:53](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/ExtensionPointsTest.php#L53) | App\WorkflowActions extend app abstract | C1 | role | placement |
| 21 | [ExtensionPointsTest.php:58](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/ExtensionPointsTest.php#L58) | App\Tooling extend app AbstractTooling (2 exceptions) | C1 | role | placement |
| 22 | [ExtensionPointsTest.php:64](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/ExtensionPointsTest.php#L64) | App\NotificationChannels extend app abstract (1 exception) | C1 | role | placement |
| 23 | [ExtensionPointsTest.php:70](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/ExtensionPointsTest.php#L70) | App\Notifications extend framework Notification | C2 | role | placement |
| 24 | [ExtensionPointsTest.php:75](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/ExtensionPointsTest.php#L75) | App\ValidationRules implement ValidationRule | C2 | role | placement |
| 25 | [ExtensionPointsTest.php:80](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/ExtensionPointsTest.php#L80) | App\Tables extend Forjed InertiaTable Table | C2 | role | placement |
| 26 | [ExtensionPointsTest.php:85](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/ExtensionPointsTest.php#L85) | Tables have suffix Table | C4 |  |  |
| 27 | [ExtensionPointsTest.php:90](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/ExtensionPointsTest.php#L90) | Console commands extend framework Command | C2 | role | placement |
| 28 | [ExtensionPointsTest.php:95](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/ExtensionPointsTest.php#L95) | Console commands have handle() | C4 |  |  |
| 29 | [ExtensionPointsTest.php:100](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/ExtensionPointsTest.php#L100) | Providers have suffix ServiceProvider | C4 |  |  |
| 30 | [ExtensionPointsTest.php:104](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/ExtensionPointsTest.php#L104) | DNSProviders do not use Controllers or Inertia | C1 | role | mixed: also bars vendor |
| 31 | [ExtensionPointsTest.php:108](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/ExtensionPointsTest.php#L108) | ServerProviders do not use Controllers or Inertia | C1 | role | mixed: also bars vendor |
| 32 | [ExtensionPointsTest.php:112](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/ExtensionPointsTest.php#L112) | SourceControlProviders do not use Controllers or Inertia | C1 | role | mixed: also bars vendor |
| 33 | [ExtensionPointsTest.php:116](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/ExtensionPointsTest.php#L116) | StorageProviders do not use Controllers or Inertia | C1 | role | mixed: also bars vendor |
| 34 | [ExtensionPointsTest.php:120](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/ExtensionPointsTest.php#L120) | Tooling does not use Controllers or Inertia | C1 | role | mixed: also bars vendor |
| 35 | [FoundationTest.php:20](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/FoundationTest.php#L20) | dd, ddd, exit never used | C4 |  | debug |
| 36 | [FoundationTest.php:24](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/FoundationTest.php#L24) | env() never used in app code | C2 | location (app-wide) |  |
| 37 | [FoundationTest.php:28](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/FoundationTest.php#L28) | sleep(), usleep() never used; use Sleep facade (1 exception) | C2 | location (app-wide) |  |
| 38 | [FoundationTest.php:33](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/FoundationTest.php#L33) | App\Traits are traits | C4 |  |  |
| 39 | [FoundationTest.php:37](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/FoundationTest.php#L37) | App\Contracts are interfaces | C4 |  |  |
| 40 | [FoundationTest.php:41](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/FoundationTest.php#L41) | Enums only in App\Enums | C4 |  | placement by language kind |
| 41 | [FoundationTest.php:46](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/FoundationTest.php#L46) | App\Exceptions implement Throwable (1 exception) | C2 | role | placement |
| 42 | [FoundationTest.php:52](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/FoundationTest.php#L52) | Nothing outside App\Exceptions implements Throwable | C2 | role | placement |
| 43 | [FoundationTest.php:57](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/FoundationTest.php#L57) | Every method has a return type (3 exceptions) | C4 |  |  |
| 44 | [HttpTest.php:7](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/HttpTest.php#L7) | Controllers have suffix Controller | C4 |  |  |
| 45 | [HttpTest.php:12](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/HttpTest.php#L12) | Nothing outside Controllers has suffix Controller | C4 |  |  |
| 46 | [HttpTest.php:17](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/HttpTest.php#L17) | Controllers extend app base Controller | C1 | role | placement |
| 47 | [HttpTest.php:23](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/HttpTest.php#L23) | Controllers do not use Validator facade (1 exception) | C2 | role |  |
| 48 | [HttpTest.php:28](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/HttpTest.php#L28) | Controllers do not use DB facade (1 exception) | C2 | role |  |
| 49 | [HttpTest.php:33](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/HttpTest.php#L33) | Controllers do not use App\Jobs (3 exceptions) | C1 | role |  |
| 50 | [HttpTest.php:38](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/HttpTest.php#L38) | No app class extends FormRequest (validation lives in actions) | C2 | role |  |
| 51 | [HttpTest.php:45](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/HttpTest.php#L45) | Http\Resources have suffix Resource | C4 |  |  |
| 52 | [HttpTest.php:46](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/HttpTest.php#L46) | Http\Resources extend JsonResource | C2 | role | placement |
| 53 | [HttpTest.php:48](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/HttpTest.php#L48) | Resources do not use Actions, Jobs, DB, Validator (1 exception) | C1 | role | mixed: also bars facades |
| 54 | [HttpTest.php:58](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/HttpTest.php#L58) | Controllers only used in App\Http and App\Providers | C1 | role |  |
| 55 | [HttpTest.php:62](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/HttpTest.php#L62) | Resources never call parent::toArray() | C3 | role | method-body; reason is field whitelisting (borderline security) |
| 56 | [JobsTest.php:8](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/JobsTest.php#L8) | Jobs implement ShouldQueue | C2 | role | placement |
| 57 | [JobsTest.php:13](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/JobsTest.php#L13) | Jobs have suffix Job | C4 |  |  |
| 58 | [JobsTest.php:18](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/JobsTest.php#L18) | Jobs have handle() | C4 |  |  |
| 59 | [JobsTest.php:23](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/JobsTest.php#L23) | Jobs do not use Controllers, Inertia, Request | C1 | role | mixed: also bars vendor |
| 60 | [JobsTest.php:31](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/JobsTest.php#L31) | Every job uses Queueable and app trait UniqueQueue (3 exceptions) | C1 | role | must-use; mixed: also vendor trait |
| 61 | [JobsTest.php:50](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/JobsTest.php#L50) | Every job has failed() (3 exceptions) | C4 |  |  |
| 62 | [JobsTest.php:64](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/JobsTest.php#L64) | Jobs that use UniqueQueue call $this->run( in their body | C3 | role | method-body string check |
| 63 | [MigrationsTest.php:25](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/MigrationsTest.php#L25) | Migrations contain none of dispatch(, App\Jobs, SSH::, App\Facades\SSH, Http::, Bus::, Queue::, SocketEvent::, broadcast(, Notification::, Mail:: | C1 | location | mixed: mostly facades/functions; token search over source text |
| 64 | [ModelsTest.php:9](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/ModelsTest.php#L9) | Models extend app AbstractModel (9 exceptions) | C1 | role | placement |
| 65 | [ModelsTest.php:19](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/ModelsTest.php#L19) | No class outside App\Models extends Eloquent Model | C2 | role | placement |
| 66 | [ModelsTest.php:24](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/ModelsTest.php#L24) | Models do not have suffix Model | C4 |  |  |
| 67 | [ModelsTest.php:30](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/ModelsTest.php#L30) | Models do not use Controllers, Inertia, Request, Validator | C1 | role | mixed: also bars vendor |
| 68 | [ModelsTest.php:39](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/ModelsTest.php#L39) | Models do not use App\Actions (5 exceptions) | C1 | role |  |
| 69 | [ModelsTest.php:44](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/ModelsTest.php#L44) | Every enum cast targets an enum implementing VitoEnum | C4 |  |  |
| 70 | [ModelsTest.php:66](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/ModelsTest.php#L66) | Credential attributes are cast encrypted or hashed | C4 |  | security |
| 71 | [PoliciesTest.php:7](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/PoliciesTest.php#L7) | Policies have suffix Policy | C4 |  |  |
| 72 | [PoliciesTest.php:12](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/PoliciesTest.php#L12) | Nothing outside Policies has suffix Policy | C4 |  |  |
| 73 | [PoliciesTest.php:17](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/PoliciesTest.php#L17) | Policies do not use Controllers, Jobs, DB, Validator, Inertia | C1 | role | mixed: also bars facades/vendor |
| 74 | [PoliciesTest.php:27](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/PoliciesTest.php#L27) | Policies only used in App\Policies and App\Providers | C1 | role |  |
| 75 | [PoliciesTest.php:31](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/PoliciesTest.php#L31) | Every policy uses app trait HasRolePolicies (9 exceptions) | C1 | role | must-use |
| 76 | [PoliciesTest.php:47](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/PoliciesTest.php#L47) | Every public policy method takes User first and returns bool | C4 |  |  |
| 77 | [PoliciesTest.php:74](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/PoliciesTest.php#L74) | Every policy maps to an existing model by name | C4 |  |  |
| 78 | [RoutingTest.php:8](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/RoutingTest.php#L8) | Every route is named | C4 |  |  |
| 79 | [RoutingTest.php:20](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/RoutingTest.php#L20) | Route names are unique | C4 |  |  |
| 80 | [RoutingTest.php:32](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/RoutingTest.php#L32) | Every non-public route has auth middleware | C4 |  | security |
| 81 | [RoutingTest.php:54](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/RoutingTest.php#L54) | Every api route has a token ability middleware | C4 |  | security |
| 82 | [RoutingTest.php:72](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/RoutingTest.php#L72) | Every project api route has can-see-project middleware | C4 |  | security |
| 83 | [SecurityTest.php:20](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/SecurityTest.php#L20) | Process component, Process facade, proc_open, popen never used (1 exception) | C2 | location (app-wide) |  |
| 84 | [SecurityTest.php:30](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/SecurityTest.php#L30) | App\Facades\SSH not used in DTOs, Enums, Http\Resources, Policies | C1 | location |  |
| 85 | [SecurityTest.php:34](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/SecurityTest.php#L34) | App\Helpers\SSH not used in DTOs, Enums, Http\Resources, Policies | C1 | location |  |
| 86 | [SecurityTest.php:38](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/SecurityTest.php#L38) | No log call mentions a credential-shaped variable | C4 |  | security; method-body |
| 87 | [SecurityTest.php:72](https://github.com/vitodeploy/vito/blob/8598e4dc0bce12d2b2f915766a100b445b7d9e22/tests/Arch/SecurityTest.php#L72) | Resources expose no *_path keys | C4 |  | security; method-body |

#### Advising App (5 hand-written rules)

| # | Where | Rule | Class | Tag | Note |
| --- | --- | --- | --- | --- | --- |
| 1 | [ArchTest.php:49](https://github.com/canyongbs/advisingapp/blob/d945da2b74325e5ecdbc3cc7210a46768ea4fe1e/tests/Tenant/Unit/ArchTest.php#L49) | Core App\Models models do not use HasVersion4Uuids (legacy list ignored) | C2 | role |  |
| 2 | [ArchTest.php:66](https://github.com/canyongbs/advisingapp/blob/d945da2b74325e5ecdbc3cc7210a46768ea4fe1e/tests/Tenant/Unit/ArchTest.php#L66) | In every module, Models do not use HasVersion4Uuids (generated per module) | C2 | role-across-locations |  |
| 3 | [ArchTest.php:74](https://github.com/canyongbs/advisingapp/blob/d945da2b74325e5ecdbc3cc7210a46768ea4fe1e/tests/Tenant/Unit/ArchTest.php#L74) | Legacy models that use HasUuids must use HasVersion4Uuids | C2 | role |  |
| 4 | [ArchTest.php:92](https://github.com/canyongbs/advisingapp/blob/d945da2b74325e5ecdbc3cc7210a46768ea4fe1e/tests/Tenant/Unit/ArchTest.php#L92) | String "product_admin." appears only in database/migrations | C4 |  | string literal search |
| 5 | [AuditTraitUsageTest.php:37](https://github.com/canyongbs/advisingapp/blob/d945da2b74325e5ecdbc3cc7210a46768ea4fe1e/app-modules/audit/tests/Tenant/AuditTraitUsageTest.php#L37) | App\Models and 6 modules do not use vendor OwenIt Auditable directly (app wrapper only) | C2 | location |  |

#### Coolify (2 hand-written rules)

| # | Where | Rule | Class | Tag | Note |
| --- | --- | --- | --- | --- | --- |
| 1 | [InsecurePrngArchTest.php:9](https://github.com/coollabsio/coolify/blob/6964714b5da26d600621eb137c9845e6240f7d52/tests/Unit/InsecurePrngArchTest.php#L9) | App does not use mt_rand, mt_srand | C4 |  | security |
| 2 | [InsecurePrngArchTest.php:13](https://github.com/coollabsio/coolify/blob/6964714b5da26d600621eb137c9845e6240f7d52/tests/Unit/InsecurePrngArchTest.php#L13) | App does not use rand, srand | C4 |  | security |

#### Pest presets used by Pinkary and Vito (44 expectations)

| # | Where | Rule | Class | Tag | Note |
| --- | --- | --- | --- | --- | --- |
| 1 | [Laravel.php:16](https://github.com/pestphp/pest/blob/v5.3.2/src/ArchPresets/Laravel.php#L16) | App\Traits are traits | C4 |  |  |
| 2 | [Laravel.php:19](https://github.com/pestphp/pest/blob/v5.3.2/src/ArchPresets/Laravel.php#L19) | App\Concerns are traits | C4 |  |  |
| 3 | [Laravel.php:22](https://github.com/pestphp/pest/blob/v5.3.2/src/ArchPresets/Laravel.php#L22) | No enums outside App\Enums | C4 |  | placement by language kind |
| 4 | [Laravel.php:26](https://github.com/pestphp/pest/blob/v5.3.2/src/ArchPresets/Laravel.php#L26) | App\Enums are enums | C4 |  |  |
| 5 | [Laravel.php:30](https://github.com/pestphp/pest/blob/v5.3.2/src/ArchPresets/Laravel.php#L30) | App\Features are classes | C4 |  |  |
| 6 | [Laravel.php:34](https://github.com/pestphp/pest/blob/v5.3.2/src/ArchPresets/Laravel.php#L34) | App\Features have resolve() | C4 |  |  |
| 7 | [Laravel.php:38](https://github.com/pestphp/pest/blob/v5.3.2/src/ArchPresets/Laravel.php#L38) | App\Exceptions implement Throwable | C2 | role | placement |
| 8 | [Laravel.php:43](https://github.com/pestphp/pest/blob/v5.3.2/src/ArchPresets/Laravel.php#L43) | Nothing outside App\Exceptions implements Throwable | C2 | role | placement |
| 9 | [Laravel.php:47](https://github.com/pestphp/pest/blob/v5.3.2/src/ArchPresets/Laravel.php#L47) | Middleware have handle() | C4 |  |  |
| 10 | [Laravel.php:51](https://github.com/pestphp/pest/blob/v5.3.2/src/ArchPresets/Laravel.php#L51) | App\Models extend Eloquent Model | C2 | role | placement |
| 11 | [Laravel.php:56](https://github.com/pestphp/pest/blob/v5.3.2/src/ArchPresets/Laravel.php#L56) | Models do not have suffix Model | C4 |  |  |
| 12 | [Laravel.php:60](https://github.com/pestphp/pest/blob/v5.3.2/src/ArchPresets/Laravel.php#L60) | Nothing outside App\Models extends Model | C2 | role | placement |
| 13 | [Laravel.php:64](https://github.com/pestphp/pest/blob/v5.3.2/src/ArchPresets/Laravel.php#L64) | Requests have suffix Request | C4 |  |  |
| 14 | [Laravel.php:68](https://github.com/pestphp/pest/blob/v5.3.2/src/ArchPresets/Laravel.php#L68) | Requests extend FormRequest | C2 | role | placement |
| 15 | [Laravel.php:72](https://github.com/pestphp/pest/blob/v5.3.2/src/ArchPresets/Laravel.php#L72) | Requests have rules() | C4 |  |  |
| 16 | [Laravel.php:75](https://github.com/pestphp/pest/blob/v5.3.2/src/ArchPresets/Laravel.php#L75) | Nothing outside Requests extends FormRequest | C2 | role | placement |
| 17 | [Laravel.php:79](https://github.com/pestphp/pest/blob/v5.3.2/src/ArchPresets/Laravel.php#L79) | Commands have suffix Command | C4 |  |  |
| 18 | [Laravel.php:83](https://github.com/pestphp/pest/blob/v5.3.2/src/ArchPresets/Laravel.php#L83) | Commands extend Command | C2 | role | placement |
| 19 | [Laravel.php:87](https://github.com/pestphp/pest/blob/v5.3.2/src/ArchPresets/Laravel.php#L87) | Commands have handle() | C4 |  |  |
| 20 | [Laravel.php:91](https://github.com/pestphp/pest/blob/v5.3.2/src/ArchPresets/Laravel.php#L91) | Nothing outside Commands extends Command | C2 | role | placement |
| 21 | [Laravel.php:95](https://github.com/pestphp/pest/blob/v5.3.2/src/ArchPresets/Laravel.php#L95) | App\Mail extend Mailable | C2 | role | placement |
| 22 | [Laravel.php:99](https://github.com/pestphp/pest/blob/v5.3.2/src/ArchPresets/Laravel.php#L99) | App\Mail implement ShouldQueue | C2 | role |  |
| 23 | [Laravel.php:103](https://github.com/pestphp/pest/blob/v5.3.2/src/ArchPresets/Laravel.php#L103) | Nothing outside App\Mail extends Mailable | C2 | role | placement |
| 24 | [Laravel.php:107](https://github.com/pestphp/pest/blob/v5.3.2/src/ArchPresets/Laravel.php#L107) | Jobs implement ShouldQueue | C2 | role | placement |
| 25 | [Laravel.php:111](https://github.com/pestphp/pest/blob/v5.3.2/src/ArchPresets/Laravel.php#L111) | Jobs have handle() | C4 |  |  |
| 26 | [Laravel.php:115](https://github.com/pestphp/pest/blob/v5.3.2/src/ArchPresets/Laravel.php#L115) | Listeners have handle() | C4 |  |  |
| 27 | [Laravel.php:118](https://github.com/pestphp/pest/blob/v5.3.2/src/ArchPresets/Laravel.php#L118) | App\Notifications extend Notification | C2 | role | placement |
| 28 | [Laravel.php:122](https://github.com/pestphp/pest/blob/v5.3.2/src/ArchPresets/Laravel.php#L122) | Nothing outside Notifications extends Notification | C2 | role | placement |
| 29 | [Laravel.php:126](https://github.com/pestphp/pest/blob/v5.3.2/src/ArchPresets/Laravel.php#L126) | Providers have suffix ServiceProvider | C4 |  |  |
| 30 | [Laravel.php:129](https://github.com/pestphp/pest/blob/v5.3.2/src/ArchPresets/Laravel.php#L129) | Providers extend ServiceProvider | C2 | role | placement |
| 31 | [Laravel.php:133](https://github.com/pestphp/pest/blob/v5.3.2/src/ArchPresets/Laravel.php#L133) | Providers are not used by any app class | C1 | role |  |
| 32 | [Laravel.php:136](https://github.com/pestphp/pest/blob/v5.3.2/src/ArchPresets/Laravel.php#L136) | Nothing outside Providers extends ServiceProvider | C2 | role | placement |
| 33 | [Laravel.php:140](https://github.com/pestphp/pest/blob/v5.3.2/src/ArchPresets/Laravel.php#L140) | Nothing outside Providers has suffix ServiceProvider | C4 |  |  |
| 34 | [Laravel.php:144](https://github.com/pestphp/pest/blob/v5.3.2/src/ArchPresets/Laravel.php#L144) | Nothing outside Controllers has suffix Controller | C4 |  |  |
| 35 | [Laravel.php:148](https://github.com/pestphp/pest/blob/v5.3.2/src/ArchPresets/Laravel.php#L148) | Controllers have suffix Controller | C4 |  |  |
| 36 | [Laravel.php:152](https://github.com/pestphp/pest/blob/v5.3.2/src/ArchPresets/Laravel.php#L152) | App\Http only used in App\Http and App\Providers | C1 | location |  |
| 37 | [Laravel.php:155](https://github.com/pestphp/pest/blob/v5.3.2/src/ArchPresets/Laravel.php#L155) | Controllers have only resource-style public methods | C4 |  |  |
| 38 | [Laravel.php:158](https://github.com/pestphp/pest/blob/v5.3.2/src/ArchPresets/Laravel.php#L158) | dd, ddd, dump, env, exit, ray never used | C2 | location (app-wide) | mixed: 5 of 6 are debug functions; env() is C2 |
| 39 | [Laravel.php:167](https://github.com/pestphp/pest/blob/v5.3.2/src/ArchPresets/Laravel.php#L167) | Policies have suffix Policy | C4 |  |  |
| 40 | [Laravel.php:173](https://github.com/pestphp/pest/blob/v5.3.2/src/ArchPresets/Laravel.php#L173) | App\Attributes implement ContextualAttribute | C2 | role |  |
| 41 | [Laravel.php:174](https://github.com/pestphp/pest/blob/v5.3.2/src/ArchPresets/Laravel.php#L174) | App\Attributes have #[Attribute] | C4 |  |  |
| 42 | [Laravel.php:175](https://github.com/pestphp/pest/blob/v5.3.2/src/ArchPresets/Laravel.php#L175) | App\Attributes have resolve() | C4 |  |  |
| 43 | [Php.php:14](https://github.com/pestphp/pest/blob/v5.3.2/src/ArchPresets/Php.php#L14) | 79 debug, legacy-mysql, echo/print and xdebug functions never used | C4 |  | debug |
| 44 | [Security.php:14](https://github.com/pestphp/pest/blob/v5.3.2/src/ArchPresets/Security.php#L14) | md5, sha1, uniqid, rand, mt_rand, eval, exec, shell_exec, unserialize, extract and 10 more never used | C4 |  | security |


## Evidence 2: mistakes teams actually fixed

### Search funnel

| App | Message matches (step 1) | Subject matches (step 2) | Touch backend PHP (step 3) | Extra searches by the miner | Diffs read | Qualifying |
| --- | --- | --- | --- | --- | --- | --- |
| Koel | 587 | 257 | 114 | about 95 more subjects from `--grep 'facade\|saloon\|helper\|inject\|service\|into a\|out of'`; one `-S 'auth()'` over models/services/repositories | 49 | 24 |
| Coolify | 2,428 | 1,322 | 817 | `git log -- bootstrap/helpers`; commits adding files under `app/Actions`, `app/Helpers`, `app/Services`, `app/Repositories`, `app/Contracts`, `app/Traits` | 41 | 20 |
| BookStack | 605 | 211 | 120 | `--grep 'organi[sz]\|namespace\|domain\|restructur\|folder\|re-?arrang\|directory\|structure' -- app` (found 5 reorganisations) | 40 | 30 |
| Advising App | 1,461 | 380 | 254 | renames across modules or role folders (`-M --diff-filter=R`, found 4) | 44 (9 judged from `--stat` only, none qualifying) | 29 |
| Bagisto | 1,696 | 748 | 433 | none | 40 | 29 |
| Firefly III | 2,034 | 454 | 340 | `--grep` for "from/out of the model/controller" (found 1, not qualifying) | 38 | 21 |
| **All** | **8,811** | **3,372** | **2,078** | | **252** | **153** |

The qualifying commits run from 2015 to 2026:

| Year | 2015 | 2016 | 2017 | 2018 | 2019 | 2020 | 2021 | 2022 | 2023 | 2024 | 2025 | 2026 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Commits | 3 | 7 | 4 | 22 | 7 | 10 | 5 | 6 | 25 | 24 | 20 | 20 |

### Patterns worth knowing before reading the tables

- **Placement is the largest single kind of C1 fix.** 34 of the 59 C1 commits move a class to the right folder, module or package. Examples:
  - Bagisto [`7206988b0e`](https://github.com/bagisto/bagisto/commit/7206988b0e079507cb5665cb985158a42eb1a7f7) moved 29 admin controllers out of 13 domain packages into Admin. 15 of them referenced `Webkul\Admin\...` back, a domain-to-Admin cycle.
  - Advising App [`62e6a4a05a`](https://github.com/canyongbs/advisingapp/commit/62e6a4a05a9720fe189eeb8d7c7ac7f43ac0e908) moved 33 classes from the `assistant` module to `ai`, breaking an ai/assistant cycle.
  - One placement fix had no reference to show for it: Bagisto [`32986ba8b0`](https://github.com/bagisto/bagisto/commit/32986ba8b0a822fa598c844560fe09e87fcbd55d) moved an install command from Core to the Installer package. The class referenced only framework classes, so only its namespace showed the misplacement.
- **Stale imports.** Several fixes removed the calls but left the `use` lines behind. An import-only check would still flag the class after these fixes:
  - Coolify [`ff3d0b29e3`](https://github.com/coollabsio/coolify/commit/ff3d0b29e3d16bdce1864021d008cad8ad4ef57f)
  - Firefly III [`c51df8cd83`](https://github.com/firefly-iii/firefly-iii/commit/c51df8cd83575242a9f1b2d996f0f0829362600c) and [`0b33e1ff09`](https://github.com/firefly-iii/firefly-iii/commit/0b33e1ff093bd4b2045558ec17b9b62195682c42)
  - BookStack [`546cfb0dcc`](https://github.com/BookStackApp/BookStack/commit/546cfb0dcc801c4fa6560ca0e6d18b4f1edd830f)
- **Some fixes run in the reverse direction:**
  - BookStack [`a641b4da2c`](https://github.com/BookStackApp/BookStack/commit/a641b4da2c08e9533a20174d7a3543694f6b7c4c) swapped an injected DB connection for the `DB` facade, so the connection opens later.
  - Coolify [`3e0d48faea`](https://github.com/coollabsio/coolify/commit/3e0d48faeaab950bfd063dfca908f1d140316ede) inlined an action and a data class back into a global helper.
  - Koel [`096eb7e610`](https://github.com/koel/koel/commit/096eb7e610cf5935f836bcfbb660fbd327da1b53) removed an "over-engineered service" in favour of model methods.
  - Not qualifying, but in the same direction:
    - Koel `f50d460a61` replaced events with a job.
    - Koel `52285a1c48` put `new Streamer(...)` back in a controller.
    - Firefly III `53a6c10ada` replaced an injected guard with `auth()`.
- **Reference form only.** Three commits name the same class before and after:
  - Koel [`65765e8357`](https://github.com/koel/koel/commit/65765e83572dd18d949db3fe55f6cc186231db90) changed a static `Network::isSafeUrl()` call to an injected `Network`.
  - Firefly III [`835a421909`](https://github.com/firefly-iii/firefly-iii/commit/835a421909fc3069542dcc41f47e77884768ae0d) changed `Processor::make()` to `app(Processor::class)`.
  - Firefly III [`1284f9cecc`](https://github.com/firefly-iii/firefly-iii/commit/1284f9cecc3b5205c198f1d5a1fb3a583be3e5ff) changed `new XTransformer` to `app(XTransformer::class)`.

  A dependency list that records only "A depends on B" cannot tell the before state from the after state. They are counted as C3 (form).
- **The same mistake lands in different classes in different apps.** Moving validation into FormRequests is a good example:
  - In Koel [`bca8668ace`](https://github.com/koel/koel/commit/bca8668ace6ee5c4e091da3420d02d9af1cd5900) the fix swapped the controllers' `Illuminate\Http\Request` type hints for app FormRequests. Koel controllers rarely type-hint `Request` at HEAD (10/175), so it is C2.
  - In Advising App [`513631e556`](https://github.com/canyongbs/advisingapp/commit/513631e556c185c22843b72cfb89e26df9f4f758) the same move is C3, because 53/92 Advising App controllers import `Request` at HEAD.

### C3 commits: would a class-level rule have caught them anyway?

"Realistic at HEAD" gives the miners' counts at each app's pinned commit. Buckets are defined under "Classes, tags and coding conventions".

| # | App | Commit | From (role) | Kind | Side-effect class-level rule | Realistic at HEAD? | Bucket |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | Koel | [`1d5c8e84b6`](https://github.com/koel/koel/commit/1d5c8e84b610ab28ad3aefe5d32c2b4c74a9f2a0) | model | body | App\Models may not use App\Events or call event() | realistic: 0 files under app/Models use App\Events or event() at HEAD; would catch 3 of the 4 moved methods (increasePlayCount fired no event) | A |
| 2 | Koel | [`20282ed633`](https://github.com/koel/koel/commit/20282ed6334fd6acd3f498a663c77a92a964506a) | controller | body | controllers may not use App\Models (import); narrower: controllers may not make static calls on App\Models (Model::query/where/fi… | import rule unrealistic: 132/175 controllers import App\Models at HEAD; static-call rule close to realistic: 9/175 controllers still call Song::query()/findMany etc. at… | P |
| 3 | Koel | [`9d9dc0b397`](https://github.com/koel/koel/commit/9d9dc0b3973e0caafa4b3c7ea13938e4a942d4ea) | model | body | App\Models may not use Illuminate\Support\Facades\DB or Illuminate\Database\Query\JoinClause | realistic: 0/29 top-level models use DB or JoinClause at HEAD (only 2 model concerns, ManagesPlayables and SupportsDeleteWhereValueNotIn, use DB); would have flagged Alb… | A |
| 4 | Koel | [`cee987f3f0`](https://github.com/koel/koel/commit/cee987f3f0f5e9725bc029672789be323534bcf8) | controller | body | controllers may not use App\Http\Resources | unrealistic: 49/175 controllers import App\Http\Resources at HEAD | N |
| 5 | Koel | [`c703c2d649`](https://github.com/koel/koel/commit/c703c2d649f12574889f4e6ac70809842f91b5fe) | AI tool | body | App\Ai\Tools may not use App\Repositories | unrealistic: 29/38 AI tools import App\Repositories at HEAD | N |
| 6 | Koel | [`65765e8357`](https://github.com/koel/koel/commit/65765e83572dd18d949db3fe55f6cc186231db90) | service | form | none | - | N |
| 7 | Koel | [`db30acc1b0`](https://github.com/koel/koel/commit/db30acc1b08f11c277dced277dfc06b7e300ee6a) | controller | body | controllers may not use App\Http\Resources (or App\Exceptions) | unrealistic: 49/175 controllers import App\Http\Resources and 20/175 import App\Exceptions at HEAD | N |
| 8 | Coolify | [`ff3d0b29e3`](https://github.com/coollabsio/coolify/commit/ff3d0b29e3d16bdce1864021d008cad8ad4ef57f) | Livewire | body | App\Livewire may not use Lcobucci\JWT or the Http facade | realistic at HEAD: 0/250 Livewire import Lcobucci, 9/250 import Http facade; but the dead imports survived this commit, so an import rule would still flag the after-state | B |
| 9 | Coolify | [`469e404725`](https://github.com/coollabsio/coolify/commit/469e40472588cb79af33f07fbdccc46d9776c482) | Livewire | body | App\Livewire may not call instant_remote_process() | partly realistic: 17/250 Livewire components still call it at HEAD; this one kept an SSH-performing helper call | P |
| 10 | Coolify | [`4e680deb93`](https://github.com/coollabsio/coolify/commit/4e680deb93d41bd298f974d128c9877747b8c9c3) | model | body | App\Models may not call instant_remote_process() | unrealistic: 16/69 models call it at HEAD (and Service kept one call after this commit) | N |
| 11 | Coolify | [`f9375f91ec`](https://github.com/coollabsio/coolify/commit/f9375f91ec1b4196211b8281da881cba209ffaf1) | helper file | body | none (moved out of a helper file, not a class; callers still call the same global functions) | - | N |
| 12 | Coolify | [`d632eb2be9`](https://github.com/coollabsio/coolify/commit/d632eb2be989d2447f9e73a64a831e3f411cf66f) | Livewire | body | none (Livewire calling actions and model methods is the intended pattern; 42/250 Livewire import App\Actions at HEAD) | - | N |
| 13 | Coolify | [`8c5c249c6a`](https://github.com/coollabsio/coolify/commit/8c5c249c6abf6666dabe3e3a7d4efe47a7274834) | controller | body | App\Http\Controllers may not call instant_remote_process() | realistic: 1/58 controllers call it at HEAD; but this controller still called it after this commit | B |
| 14 | Coolify | [`e74da06465`](https://github.com/coollabsio/coolify/commit/e74da06465460d5f3446fe24e33156b8826655cf) | Livewire | body | App\Livewire may not use App\Models | unrealistic: 223/250 Livewire components import App\Models at HEAD | N |
| 15 | Coolify | [`0540b2eae5`](https://github.com/coollabsio/coolify/commit/0540b2eae567b344661f26f4c154be77b1efc20f) | Livewire | body | App\Livewire may not use App\Models | unrealistic: 223/250 Livewire components import App\Models at HEAD | N |
| 16 | Coolify | [`70fb4c6869`](https://github.com/coollabsio/coolify/commit/70fb4c6869ed9e72ed62b9e409f29819d42632b9) | model | body | none (the before-state named nothing a rule could catch; the fix adds a reference) | - | N |
| 17 | Coolify | [`f8f27fff13`](https://github.com/coollabsio/coolify/commit/f8f27fff138fea0b03121b14fa002459328a620f) | job | body | none plausible ('App\Jobs may not use Cron\CronExpression' holds at HEAD, 1/64 jobs, but is not an architectural rule) | - | N |
| 18 | Coolify | [`bf72fd5d88`](https://github.com/coollabsio/coolify/commit/bf72fd5d8823c788d5f1f8eb18898f9839ed6546) | controller | body | App\Http\Controllers may not use App\Models | unrealistic: 48/58 controllers import App\Models at HEAD | N |
| 19 | BookStack | [`f7a5a0705b`](https://github.com/BookStackApp/BookStack/commit/f7a5a0705bba304a705df74b1e7e724827900a0c) | repository | body | none | - | N |
| 20 | BookStack | [`2a2cc858f0`](https://github.com/BookStackApp/BookStack/commit/2a2cc858f0f95c606f2ad2542f915d3bd761775f) | controller | body | none | - | N |
| 21 | BookStack | [`8b550991a4`](https://github.com/BookStackApp/BookStack/commit/8b550991a4dbcd7a05e08e8f600b5444d5f4004d) | repository | body | none | - | N |
| 22 | BookStack | [`31f5786e01`](https://github.com/BookStackApp/BookStack/commit/31f5786e01fc5ed439f347c6979679612baca4fb) | controller | body | none | - | N |
| 23 | BookStack | [`92690d1ae9`](https://github.com/BookStackApp/BookStack/commit/92690d1ae92363467c6a49664a558320b8b5771b) | controller | body | none | - | N |
| 24 | BookStack | [`9d7ce59b18`](https://github.com/BookStackApp/BookStack/commit/9d7ce59b18c5cbeef017349f38f063a63b762188) | controller | body | controllers may not call error_log() | realistic: 0/74 controller files call error_log() at HEAD | A |
| 25 | BookStack | [`c157dc3490`](https://github.com/BookStackApp/BookStack/commit/c157dc3490c39aa236e22714019a649ebeb11f13) | controller | body | controllers may not use the Activity facade (BookStack\Facades\Activity) | partly: 5/74 controller files import Facades\Activity at HEAD (7 call Activity::); BookController kept it right after this commit | P |
| 26 | BookStack | [`99b14621f9`](https://github.com/BookStackApp/BookStack/commit/99b14621f9bb5f8f02a16a15beb7576a73b2f550) | repository | body | none | - | N |
| 27 | BookStack | [`9e7bcacf8c`](https://github.com/BookStackApp/BookStack/commit/9e7bcacf8c5615526add66e950407d02ec1ac185) | exception handler | body | none | - | N |
| 28 | BookStack | [`3f9527f166`](https://github.com/BookStackApp/BookStack/commit/3f9527f166740b8580adbf513600fd46104f47d0) | repository | body | none | - | N |
| 29 | BookStack | [`abc283fc64`](https://github.com/BookStackApp/BookStack/commit/abc283fc64b7ed3e79f5e499f934130b6146a02b) | controller | body | controllers may not use BookStack\Util\WebSafeMimeSniffer or Symfony StreamedResponse | realistic but narrow: 0/74 controllers use WebSafeMimeSniffer, 1/74 mention StreamedResponse at HEAD | A |
| 30 | BookStack | [`222c665018`](https://github.com/BookStackApp/BookStack/commit/222c665018cd7fc231d2970307e3a7423e4a377f) | repository | body | none | - | N |
| 31 | BookStack | [`546cfb0dcc`](https://github.com/BookStackApp/BookStack/commit/546cfb0dcc801c4fa6560ca0e6d18b4f1edd830f) | controller | body | controllers may not use BookStack\Entities\Models | partly: 12/74 controller files still import Entities\Models at HEAD (0 static ::visible() calls left); cross-domain variant unrealistic (Activity 24/54, View 17/42, Refe… | P |
| 32 | BookStack | [`b77ab6f3af`](https://github.com/BookStackApp/BookStack/commit/b77ab6f3af996f924bd2ddbac6e88f61a8bf6cf9) | model | body | BookStack\Search may not use BookStack\Entities\Models | unrealistic: 4/14 Search files import Entities\Models at HEAD | N |
| 33 | BookStack | [`dead0bfb5f`](https://github.com/BookStackApp/BookStack/commit/dead0bfb5fd5b725230a5020973ddd4fa9560a95) | controller | body | Entities controllers may not use BookStack\References (or any other domain's non-model classes) | References-only rule realistic (1/74 controllers outside References import it at HEAD); general cross-domain rule unrealistic (60/74 controller files import another doma… | N |
| 34 | Advising App | [`ae829cc5a2`](https://github.com/canyongbs/advisingapp/commit/ae829cc5a28ca9d433993bd9d1e0bcc242989915) | controller | body | controllers may not call config() | unrealistic: 8/92 controllers call config() at HEAD | N |
| 35 | Advising App | [`5792abb26d`](https://github.com/canyongbs/advisingapp/commit/5792abb26d01b5f75b8b9a6f0834f992274d5391) | controller | body | none | - | N |
| 36 | Advising App | [`9536b57956`](https://github.com/canyongbs/advisingapp/commit/9536b579566010bc6ea9207060d116abd8f28917) | notification | body | notification classes may not use *\Models\* | unrealistic: 28/32 notification classes import a Models class at HEAD | N |
| 37 | Advising App | [`119a7aa51c`](https://github.com/canyongbs/advisingapp/commit/119a7aa51cb76844a07d36ce13787b620e13fb71) | model | body | none | - | N |
| 38 | Advising App | [`4624f6b02b`](https://github.com/canyongbs/advisingapp/commit/4624f6b02bd467c12a608bc05223c3ed7c946bc2) | model | body | none | - | N |
| 39 | Advising App | [`513631e556`](https://github.com/canyongbs/advisingapp/commit/513631e556c185c22843b72cfb89e26df9f4f758) | controller | body | controllers may not use Illuminate\Http\Request (or call auth()) | unrealistic: 53/92 controllers import Illuminate\Http\Request at HEAD; 7/92 use auth()/Auth | N |
| 40 | Advising App | [`1fcb03c1c6`](https://github.com/canyongbs/advisingapp/commit/1fcb03c1c6078cf9df79ba6a7c30a082ab22acac) | controller | body | none | - | N |
| 41 | Advising App | [`b721d0f1ce`](https://github.com/canyongbs/advisingapp/commit/b721d0f1ce3da02e82072b8bd3f35d68cd1586f1) | job | body | none | - | N |
| 42 | Advising App | [`76f99430d8`](https://github.com/canyongbs/advisingapp/commit/76f99430d8895387a79b6a09b7d1b28c7dfb7cf2) | job | body | AdvisingApp\Engagement may not use AdvisingApp\Notification\Enums\NotificationChannel (or jobs may not) | unrealistic: 26/97 engagement src files import NotificationChannel at HEAD (4/100 module jobs) | N |
| 43 | Bagisto | [`abd3bba617`](https://github.com/bagisto/bagisto/commit/abd3bba6177656819a0f4cf21e8fb4853b5fe741) | controller | body | controllers may not use Illuminate\Support\Facades\Event | unrealistic: 58/131 controllers import the Event facade at HEAD | N |
| 44 | Bagisto | [`8f67109880`](https://github.com/bagisto/bagisto/commit/8f671098805810501195711eac5583f5009e55d5) | controller | body | controllers may not use Illuminate\Support\Facades\Event | unrealistic: 58/131 controllers import the Event facade at HEAD (42 Admin, 15 Shop) | N |
| 45 | Bagisto | [`2ca4f7087e`](https://github.com/bagisto/bagisto/commit/2ca4f7087e6662952ceb3f3cd0f38dccaad474f2) | repository | body | repositories may not import another package's concrete (non-Proxy) Models (catches use Webkul\Attribute\Models\Attribute) | realistic: 1/103 repositories import another package's non-proxy model at HEAD (RMA/RMARepository) | A |
| 46 | Bagisto | [`f6180869fb`](https://github.com/bagisto/bagisto/commit/f6180869fb252d1e93ecab221df618655707a8f6) | payment manager | body | none | - | N |
| 47 | Bagisto | [`1a29eb2a24`](https://github.com/bagisto/bagisto/commit/1a29eb2a24e1c19c55cea72371f569adc94c4eb5) | Blade view | body | none | - | N |
| 48 | Bagisto | [`cf506db98b`](https://github.com/bagisto/bagisto/commit/cf506db98bd4b541e86359cc03bd54c4030b41d1) | controller | body | none | - | N |
| 49 | Firefly III | [`cc88d5962e`](https://github.com/firefly-iii/firefly-iii/commit/cc88d5962ecddf893c8454af248bb3688d73f02a) | controller | body | controllers may not use the Mail facade (Illuminate\Support\Facades\Mail) | realistic: 0/269 controllers reference Mail at HEAD (but this commit left 2 Mail::send calls in AuthController, so the rule would still flag after it) | B |
| 50 | Firefly III | [`9a30fbd05a`](https://github.com/firefly-iii/firefly-iii/commit/9a30fbd05a34ec11ae3d3f721eacf720a470569b) | controller | body | none (controllers legitimately use FormRequests; 'controllers may not use FireflyIII\User/Models' would only catch User::get) | unrealistic: 219/269 controllers import FireflyIII\Models or FireflyIII\User at HEAD | N |
| 51 | Firefly III | [`7bbfb692de`](https://github.com/firefly-iii/firefly-iii/commit/7bbfb692de78cf5bef3d3899a90bf73a6121f5a3) | controller | body | controllers may not use FireflyIII\Models | unrealistic: 219/269 controllers import FireflyIII\Models or User at HEAD | N |
| 52 | Firefly III | [`b627d42160`](https://github.com/firefly-iii/firefly-iii/commit/b627d42160a5e5678318a06cf12d4811a5724951) | repository | body | Repositories may not use FireflyIII\Models | unrealistic: 30/32 *Repository.php import FireflyIII\Models at HEAD | N |
| 53 | Firefly III | [`1b304bf85e`](https://github.com/firefly-iii/firefly-iii/commit/1b304bf85edc3f6a42f40367c83a4a693935e84b) | model | body | none (model trait still uses CacheProperties and relation queries; controllers keep models) | n/a; a 'controllers may not use Models' rule would be unrealistic: 219/269 | N |
| 54 | Firefly III | [`1645490f5c`](https://github.com/firefly-iii/firefly-iii/commit/1645490f5c3de08335ebe4d4afab75eb8e9f215f) | transformer | body | Transformers may not use FireflyIII\Models | unrealistic: 24/26 transformers import FireflyIII\Models at HEAD | N |
| 55 | Firefly III | [`3c9b7c07af`](https://github.com/firefly-iii/firefly-iii/commit/3c9b7c07aff5236fc04589f276a36e251d3226bd) | import storage | body | none (import classes keep many model imports) | n/a: app/Import removed by HEAD | N |
| 56 | Firefly III | [`835a421909`](https://github.com/firefly-iii/firefly-iii/commit/835a421909fc3069542dcc41f47e77884768ae0d) | event handler | form | none | - | N |
| 57 | Firefly III | [`1284f9cecc`](https://github.com/firefly-iii/firefly-iii/commit/1284f9cecc3b5205c198f1d5a1fb3a583be3e5ff) | controller | form | none | - | N |
| 58 | Firefly III | [`e4a9abc315`](https://github.com/firefly-iii/firefly-iii/commit/e4a9abc315ce862ceb84b65a34a1159a7265d639) | command | body | none (commands/jobs legitimately use TransactionRules classes; Processor import stayed) | n/a | N |

### Commit tables

One table per app, in date order. "What was fixed" and "Evidence" come from the miners' rows; the lead shortened them. "(lead audit)" marks a class the lead changed. The diffs read that did not qualify are listed under each table, with the reason.

#### Koel: 24 qualifying of 49 diffs read

| # | Commit | Date | What was fixed | Class | Tag | Kind | Evidence (abridged) |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | [`3e747101a9`](https://github.com/koel/koel/commit/3e747101a969a63eae7f991234bbc1bb411d7026) | 2015-12-21 | Model stopped reading the authenticated user via auth(); the caller passes the User in. | C2 | role | dependency | 4 static model methods used `$userId ?: auth()->user()->id`; now take `User $user` and InteractionController passes `$request->user()`; `DB::table` stays in batchUnlike |
| 2 | [`27933d49cf`](https://github.com/koel/koel/commit/27933d49cf0a97b022556b3405b381f1d6044fbf) | 2016-08-21 | App code stopped calling env() directly and reads values through config. | C2 | location | dependency | `env('LASTFM_API_KEY')`, `env('STREAMING_METHOD')`, `env('ALLOW_DOWNLOAD', true)` etc. replaced by `config('koel.*')` in 9 app classes (external contributor); at HEAD only app/Providers/RouteServiceProvider.php still ca… |
| 3 | [`3049804811`](https://github.com/koel/koel/commit/304980481147a4d96aa3f75b2fb192361ac404b3) | 2017-04-20 | Non-HTTP service classes moved out of App\Http into App\Services. | C1 | location | placement | 7 streamer classes (PHPStreamer, S3Streamer, TranscodingStreamer, XSendFile/XAccelRedirect streamers, Streamer, StreamerInterface) moved App\Http\Streamers to App\Services\Streamers, namespace-only; SongController impor… |
| 4 | [`47c36f289e`](https://github.com/koel/koel/commit/47c36f289eca79dfafd80e124ac0b0d5961f91fb) | 2017-04-29 | Model stopped reading the authenticated user via the auth() helper; the controller passes the User in. | C2 | role | dependency | Song::scrobble() read `auth()->user()->lastfm_session_key` (parent); fix adds `User $user` param and ScrobbleController passes `$request->user()`; auth() gone from the model |
| 5 | [`bca8668ace`](https://github.com/koel/koel/commit/bca8668ace6ee5c4e091da3420d02d9af1cd5900) | 2017-12-09 | Controllers stopped type-hinting the framework Request; validation moved into app FormRequest classes. | C2 | role | dependency | controllers swapped `use Illuminate\Http\Request` type hints for per-action app FormRequests; e.g. LastfmController::callback's `abort_unless($token = $request->token, ...)` became LastfmCallbackRequest rules (at HEAD 1… |
| 6 | [`1d5c8e84b6`](https://github.com/koel/koel/commit/1d5c8e84b610ab28ad3aefe5d32c2b4c74a9f2a0) | 2018-08-18 | Business and persistence logic moved from model static methods into a service. | C3 | role | body | 4 static business methods (increasePlayCount, toggleLike, batchLike, batchUnlike: firstOrCreate/save + `event(new SongLikeToggled)`) moved from the Interaction model to a new InteractionService; model dropped `use App\E… |
| 7 | [`4fc5117509`](https://github.com/koel/koel/commit/4fc51175096cda8a7e007302f5efa3bf40ddcc90) | 2018-08-18 | Models stopped calling the Last.fm integration; a service fetches the info and stores the cover/image. | C1 | role | dependency | Album::getInfo()/Artist::getInfo() called `Lastfm::getAlbumInfo()/getArtistInfo()` (App\Facades\Lastfm, an app facade), Log facade and file_get_contents; removed from the models (Album dropped `use App\Facades\Lastfm; u… |
| 8 | [`8f7654a220`](https://github.com/koel/koel/commit/8f7654a220bacf466019efe28d4f1e2aed10dfde) | 2018-08-19 | Model stopped calling the Last.fm integration through an app facade; the controller calls the injected service. | C1 | role | dependency | Song::scrobble() called `Lastfm::scrobble(...)` (alias `use Lastfm;` = App\Facades\Lastfm, an app facade over the Last.fm service); method removed and logic placed in ScrobbleController using injected App\Services\Lastf… |
| 9 | [`a837432a4e`](https://github.com/koel/koel/commit/a837432a4e8e230848f45f3fdfe41553a96bd7ba) | 2018-08-19 | App facade calls replaced by constructor injection; the service's Cache facade replaced by an injected repository. | C1 | location | dependency | `MediaCache::get()/clear()` (alias of App\Facades\MediaCache) replaced by constructor-injected App\Services\MediaCacheService in a command, a controller and a listener; alias removed from config/app.php; the service swa… |
| 10 | [`7c7693179d`](https://github.com/koel/koel/commit/7c7693179da1b77b1f5895e94244aad7acfcc818) | 2018-08-22 | Controller stopped instantiating concrete streamers; selection moved to an injected factory (52285a1c later put `new Streamer(...)` back in PlayController). | C1 | role | dependency | SongController::play dropped 5 `use App\Services\Streamers\*` and `new S3Streamer/TranscodingStreamer/XSendFileStreamer/XAccelRedirectStreamer/PHPStreamer` plus a switch on config(); now injects StreamerFactory, which r… |
| 11 | [`20282ed633`](https://github.com/koel/koel/commit/20282ed6334fd6acd3f498a663c77a92a964506a) | 2018-08-29 | Query building moved from controllers, a model and a service into new repository classes. | C3 | role | body | repository layer added; DataController dropped `use App\Models\{Interaction,Playlist,Setting,User}` and static queries (`Setting::pluck`, `Playlist::byCurrentUser()`, `User::all()`); other controllers keep model imports… |
| 12 | [`bf1d158c90`](https://github.com/koel/koel/commit/bf1d158c9070ca63bc65cfdbeb67d060a1ae6724) | 2018-08-29 | AWS SDK and cache facade usage moved out of the Song model into an injected service. | C2 | role | dependency | Song::getObjectStoragePublicUrl() used `AWS::createClient('s3')` facade, Aws\AwsClient and `Cache::remember`; moved to S3Service with injected S3ClientInterface + Cache repository; DownloadService and S3Streamer inject… |
| 13 | [`b227ece517`](https://github.com/koel/koel/commit/b227ece517fd3a31003cba464582bdfa4b079601) | 2018-08-29 | Cache facade/helper replaced by an injected cache contract in two services. | C2 | role | dependency | removed `cache()` helper calls (LastfmService) and `use Cache;` + `Cache::remember` (iTunesService); both now constructor-inject Illuminate\Contracts\Cache\Repository; `use Log;` facade kept |
| 14 | [`351efe4092`](https://github.com/koel/koel/commit/351efe4092393dd708a9c625aa9674663ba30ec5) | 2018-08-30 | HTTP, cache and logging logic moved out of the framework kernel subclass into an injected service. | C2 | role | dependency | removed `use Cache; use GuzzleHttp\Client; use Log;` and getLatestVersion() (Cache::remember, `new Client()`, Log::error) from App\Application (extends Illuminate Foundation Application); new service injects Client, Cac… |
| 15 | [`096eb7e610`](https://github.com/koel/koel/commit/096eb7e610cf5935f836bcfbb660fbd327da1b53) | 2018-09-04 | The Last.fm API client stopped owning user-preference logic; it moved to the User model and an extra service layer was removed. | C1 | role | dependency | LastfmService (extends App\Services\ApiClient) dropped `use App\Models\User`, the injected UserPreferenceService and get/set/deleteUserSessionKey(); UserPreferenceService deleted; preference handling now User::savePrefe… |
| 16 | [`84b05c449f`](https://github.com/koel/koel/commit/84b05c449f9f2ff271ff3906f5a178ed57aef358) | 2020-09-07 | Web-route controllers moved out of the API controller namespace. | C1 | location | placement | controllers serving web routes (download, play, Last.fm connect/callback, iTunes) moved out of App\Http\Controllers\API together with routes/api.php to routes/web.php; bundled with a new query-token guard (feature part) |
| 17 | [`9d9dc0b397`](https://github.com/koel/koel/commit/9d9dc0b3973e0caafa4b3c7ea13938e4a942d4ea) | 2022-08-09 | Query-building scopes moved from models into dedicated builder classes. | C3 | role | body | scopes and static withMeta() query code (leftJoin with JoinClause, DB::raw) moved out of models into App\Builders; Album and Artist dropped `use Illuminate\Support\Facades\DB` + `use Illuminate\Database\Query\JoinClause… |
| 18 | [`a4366fe3b1`](https://github.com/koel/koel/commit/a4366fe3b170f6a89a64224ef993abe6db8d3527) | 2024-03-22 | Third-party API client moved from the Services folder to the Integrations folder. | C1 | location | placement | SpotifyClient (wraps SpotifyWebAPI, no base class) moved App\Services\ApiClients to App\Http\Integrations\Spotify next to the Saloon connectors; SpotifyService import updated |
| 19 | [`cee987f3f0`](https://github.com/koel/koel/commit/cee987f3f0f5e9725bc029672789be323534bcf8) | 2026-03-13 | Serialization of AI tool results moved from the controller body into per-action serializer classes. | C3 | role | body | controller dropped 6 `use App\Http\Resources\*` + FavoriteableType and a ~60-line `match ($result->action)` serializer; the 11 new App\Ai\Serializers classes import the same Resources; controller still builds response()… |
| 20 | [`c703c2d649`](https://github.com/koel/koel/commit/c703c2d649f12574889f4e6ac70809842f91b5fe) | 2026-03-13 | Entity lookup logic moved from two AI tools into an App\Ai\Services class (also removes duplication). | C3 | role | body | both tools dropped 5 `use App\Repositories\*Repository` and a duplicated resolve() `match` over repo->search()/findOne(); they now inject FavoriteableEntityResolver; other tools keep using repositories; borderline DRY |
| 21 | [`a42eae8b77`](https://github.com/koel/koel/commit/a42eae8b77160d9f51aa761f393dcdfc92bea77d) | 2026-03-20 | Non-Eloquent class moved out of App\Models (convention: App\Models classes extend Eloquent Model). | C2 | role | placement | SongZipArchive (ZipArchive wrapper, no `extends Model`) lived in App\Models; moved to App\Values (namespace-only); DownloadService import updated; message: "not an Eloquent model" |
| 22 | [`9ab2bae1da`](https://github.com/koel/koel/commit/9ab2bae1da943ec35842ce3df4ab73bddef01dab) | 2026-05-14 | Stateless helper moved out of Services/Facades into App\Helpers, deleting the facade indirection. | C1 | location | placement | 7-line stateless BOM detector lived as App\Services\Util behind App\Facades\Util + provider + config alias; collapsed into static App\Helpers\Encoding\Bom; Artist `use App\Facades\Util` replaced by `use App\Helpers\Enco… |
| 23 | [`65765e8357`](https://github.com/koel/koel/commit/65765e83572dd18d949db3fe55f6cc186231db90) | 2026-05-26 | Static helper calls replaced by an injectable instance; the class is still named, only the reference form changed (static call to injection/app()). | C3 (lead audit) | location | form | static calls `Network::isSafeUrl()/isPublicHost()` removed from PodcastService, SafeUrl, EpisodePlayable; Network methods made instance; PodcastService constructor-injects Network, SafeUrl defaults to `app(Network::clas… |
| 24 | [`db30acc1b0`](https://github.com/koel/koel/commit/db30acc1b08f11c277dced277dfc06b7e300ee6a) | 2026-10-07 | HTTP error-response mapping moved from controller bodies into exception render() methods. | C3 | role | body | controllers dropped try/catch mapping (`catch (DuplicateSongUploadException)` to `response()->json(new DuplicateUploadResource(...), 409)`, `abort(400, ...)`) and the imports of both exceptions + DuplicateUploadResource… |

<details><summary>Koel: 25 diffs read that did not qualify</summary>

| Commit | Date | Subject | Why not |
| --- | --- | --- | --- |
| [`aa7267419d`](https://github.com/koel/koel/commit/aa7267419d6d7ec9f0c93fb852b561b1d0daafc0) | 2017-06-04 | Massive refactorings | renames (isUnknown() to is_unknown accessor), result constants, private-method extraction |
| [`796088944d`](https://github.com/koel/koel/commit/796088944d6b48b45267c031e0230d0d374e11b6) | 2017-06-04 | Add more refactorings | docblocks and import additions only |
| [`caf13fbac0`](https://github.com/koel/koel/commit/caf13fbac0abc6a09a0e7f8c7eaee6e77b770a9e) | 2017-06-04 | Refactor Download service | class extracted from the Download service for reuse (SongZipArchive, placed in App\Models; the placement was later fixed by a42eae8b) |
| [`02d36c692e`](https://github.com/koel/koel/commit/02d36c692e4816e0e1cb589e9a4efaf26c4936b3) | 2018-08-23 | Refactor InitCommand | private-method extraction inside one command |
| [`f4ca7cf09f`](https://github.com/koel/koel/commit/f4ca7cf09f5dd62d37e9ef45ba32629304666326) | 2018-08-29 | Refactor S3 | duplicate of bf1d158c (same patch applied on a different parent; both reachable from master) |
| [`2349a66a31`](https://github.com/koel/koel/commit/2349a66a310279dcd2d9923ea7ceaee70bd6a125) | 2018-08-30 | Refactor php streamer | body cleanup inside PHPStreamer (range parsing extracted to a private method) |
| [`453fb76c5d`](https://github.com/koel/koel/commit/453fb76c5d1dcdc8e34ac6b369ac6ab221d9b6df) | 2018-08-30 | Refactor and fix requests | DRY: shared AbstractRequest base for FormRequests (authorize/rules boilerplate); class shape within one role |
| [`5c870ee254`](https://github.com/koel/koel/commit/5c870ee2544d98ae8399ccf70713b3dbf0230e16) | 2018-09-04 | Refactor Lastfm's connect | style: URL building rewritten with sprintf |
| [`0f9bb32956`](https://github.com/koel/koel/commit/0f9bb3295692ee6332f964e438062d0daf557e6c) | 2018-09-04 | Refactor Lastfm's disconnect | return value change in one controller method |
| [`ee6ac687c4`](https://github.com/koel/koel/commit/ee6ac687c49d2035204d8bfee0abc9d841015d27) | 2018-10-19 | Refactor the observers (#836) | private-method extraction inside observers |
| [`08e4953217`](https://github.com/koel/koel/commit/08e49532179c1e8ef687712d8125e3077cbb2a40) | 2022-07-08 | feat: decouple artist/album and the media information | feature: info arrays replaced by AlbumInformation/ArtistInformation value objects, plus frontend changes |
| [`444e91ab57`](https://github.com/koel/koel/commit/444e91ab57d546c1fa6d5168e33331eb0cbdd8b3) | 2022-08-01 | chore: add User dependency to SmartPlaylistService | adds an optional User param to SmartPlaylistService::getSongs; injected Guard kept as fallback, nothing removed |
| [`d8282557d7`](https://github.com/koel/koel/commit/d8282557d74f61171c868e3bcd47c83c51e4c6dd) | 2024-01-01 | refactor: better repository method naming and usage | repository method renames (getByIds to getMany, getOneById to getOne/findOne) |
| [`52285a1c48`](https://github.com/koel/koel/commit/52285a1c48d24ce89a0bd37c057a119e1144503a) | 2024-02-24 | refactor: use adapters for streamers | redesign inside the Services layer (factory replaced by Streamer + adapter strategy); PlayController now does `new Streamer(...)` |
| [`1afd8ba7d5`](https://github.com/koel/koel/commit/1afd8ba7d50d53589946ec48337f2d23a57b3e6f) | 2024-06-05 | refactor: create Song query with contextual user and type | builder API reshaping (user stored on SongBuilder, Song::query(type, user)) |
| [`f50d460a61`](https://github.com/koel/koel/commit/f50d460a610fd43b54c627ae0088dd6bb3269e24) | 2025-07-04 | refactor: use job instead of event for song folder extraction (#2003) | mechanism swap (event+listener replaced by a queued job at one call site); SongService keeps using events (LibraryChanged); also moves controllers/services fro… |
| [`ecb206d4f3`](https://github.com/koel/koel/commit/ecb206d4f3ab2f1f4a1fdd6e4f94c6d2c4d28dc9) | 2026-03-13 | refactor: extract ResolvesSongFromRequest trait for AI tools (#2277) | DRY: shared song-lookup helper (App\Ai\Services\SongRequestResolver) across sibling AI tools; point is reuse; tools keep repositories elsewhere |
| [`a7b5775a2a`](https://github.com/koel/koel/commit/a7b5775a2a36a5cffae275806c738d680d728960) | 2026-04-02 | refactor: fix bugs and improve repositories and query builders (#2369) | bug fixes, method renames and indexes in repositories/builders |
| [`48998f9136`](https://github.com/koel/koel/commit/48998f913659808bf4420d97278c733a677dd5b0) | 2026-04-28 | refactor: consolidate DownloadService storage dispatch via factory (#2425) | polymorphism inside the Services layer (SongStorageFactory replaces if-chains); same layer |
| [`fed79599e5`](https://github.com/koel/koel/commit/fed79599e53ae0bcd0f29502804082314797f4c2) | 2026-04-29 | refactor: extract model-image cleanup into a field-bound ModelImageObserver (#2432) | DRY plus bug fix: identical image-file cleanup in 5 observers moved into one helper; observers keep File facade and model refs |
| [`899ad5d4ab`](https://github.com/koel/koel/commit/899ad5d4ab72b8c85538ed69efa5166a9cc3ca95) | 2026-05-12 | refactor(services): organize Services/ into feature subfolders (#2458) | namespace regrouping inside the same App\Services role folder (feature subfolders); no layer or dependency change |
| [`3b024063a5`](https://github.com/koel/koel/commit/3b024063a5b8e00910562451e5d86268ac2dcd77) | 2026-06-11 | refactor: auto-hash user passwords via Eloquent cast (#2570) | framework-feature adoption ('hashed' cast) that removes 7 duplicated Hash::make calls; Hash facade still used for check/needsRehash |
| [`7f321705d7`](https://github.com/koel/koel/commit/7f321705d7afc31a60181e8c6eab85c0220869bf) | 2026-09-27 | refactor: store branding the same way as the AI settings (#2742) | feature (multi-organization): branding stored per Organization; signature changes |
| [`a9953b7682`](https://github.com/koel/koel/commit/a9953b76821cc48a5bb566a45b0c4802a6c9813e) | 2026-10-07 | refactor: check whether a song can be scrobbled in one place (#2793) | DRY: shared Song::isScrobbleable() predicate used by a controller and 4 listeners; references unchanged |
| [`72ab1e8d9b`](https://github.com/koel/koel/commit/72ab1e8d9bc4b744b2043a5259e1a06d733a9a29) | 2026-10-08 | refactor: use the existing URL and user-agent helpers (#2812) | DRY: reuse existing app_url()/http_user_agent() helpers and Uri |

</details>

#### Coolify: 20 qualifying of 41 diffs read

| # | Commit | Date | What was fixed | Class | Tag | Kind | Evidence (abridged) |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | [`e74c464857`](https://github.com/coollabsio/coolify/commit/e74c464857a278f463cab6bafa806b4be60735af) | 2023-03-21 | Queue job stops shelling out via the Process facade and writing the DB itself; a dedicated process-runner class does it. | C2 | role | dependency | job dropped `use Illuminate\Support\Facades\DB;` and `use ...\Facades\Process;`: `Process::start($sshCommand,...)` SSH run plus `DB::transaction` throttled activity writes (~100 lines) moved to new RemoteProcess class;… |
| 2 | [`83a7ef21f3`](https://github.com/coollabsio/coolify/commit/83a7ef21f388906ffbdb8c6b22ce0e13624d4e3c) | 2023-03-21 | An enum and an action-like invokable relocated from Services to the folders for their kind. | C1 | role | placement | pure moves: enum ProcessStatus App\Services -> App\Enums; invokable RemoteProcess App\Services -> App\Actions; only namespace/use lines change in the job and CoolifyProcess |
| 3 | [`cf121062a1`](https://github.com/coollabsio/coolify/commit/cf121062a1886baa5f5f0892767538b8f8288e6a) | 2023-03-21 | Second step of the same series: the dispatcher service moves into the Actions folder beside its runner. | C1 | role | placement | CoolifyProcess (creates activity + dispatches ExecuteCoolifyProcess job) moved App\Services -> App\Actions\RemoteProcess and renamed DispatchRemoteProcess; bootstrap/helpers.php import updated; new ActivityTypes enum; s… |
| 4 | [`ff3d0b29e3`](https://github.com/coollabsio/coolify/commit/ff3d0b29e3d16bdce1864021d008cad8ad4ef57f) | 2023-03-31 | Livewire deploy button stops running the whole deployment inline and dispatches a job. | C3 | role | body | ~200 lines of deploy() orchestration (docker cmd building, `Yaml::dump` compose, GitHub JWT via Lcobucci + `Http::withHeaders` token fetch, `remoteProcess()`) moved into a queued job; component still uses App\Models\App… |
| 5 | [`469e404725`](https://github.com/coollabsio/coolify/commit/469e40472588cb79af33f07fbdccc46d9776c482) | 2023-07-28 | Proxy-config save/check logic shared by Livewire and StartProxy moves into two invokable actions. | C3 | role | body | Proxy::submit() lost `config('coolify.proxy_config_path')`, base64/md5 of the compose file, `$server->proxy->last_saved_settings =` and `instant_remote_process(["echo ... base64 -d > .../docker-compose.yml"])`, now `res… |
| 6 | [`0ef386b4a8`](https://github.com/coollabsio/coolify/commit/0ef386b4a870ea244048ab47b76d620c2373007c) | 2023-10-14 | Livewire components stop running docker commands over SSH directly; stop and proxy logic moves into actions and a job. | C2 | role | dependency | 9 calls to global SSH helpers (instant_remote_process, getCurrentApplicationContainerStatus, startDatabaseProxy/stopDatabaseProxy) removed from 5 Livewire components (0 left), replaced by StopApplication::run / StopData… |
| 7 | [`4e680deb93`](https://github.com/coollabsio/coolify/commit/4e680deb93d41bd298f974d128c9877747b8c9c3) | 2023-12-08 | Service teardown orchestration moves from a model lifecycle hook into an action run by the deletion job. | C3 | role | body | ~30-line deleting hook (collect storages, cascade-delete env vars/apps/dbs, `instant_remote_process(["docker volume rm -f ..."])`) moved to DeleteService, dispatched by DeleteResourceJob; model gains SoftDeletes and `us… |
| 8 | [`ba636a95dc`](https://github.com/coollabsio/coolify/commit/ba636a95dc4ca5d27711e79756840e78cbd2d1a6) | 2024-09-16 | Key parsing/validation with phpseclib moves out of the Livewire form into the PrivateKey model. | C2 | role | dependency | component dropped `use phpseclib3\Crypt\PublicKeyLoader;` and `use DanHarrin\LivewireRateLimiting\WithRateLimiting;`: PublicKeyLoader::load validation/public-key extraction, key generation, trim+create moved to PrivateK… |
| 9 | [`f9375f91ec`](https://github.com/coollabsio/coolify/commit/f9375f91ec1b4196211b8281da881cba209ffaf1) | 2024-09-16 | Global SSH helper functions become a static helper class behind unchanged wrapper functions. | C3 | location | body | ~200 lines of global functions (server_ssh_configuration, ensureMultiplexedConnection, shouldResetMultiplexedConnection, ssh/scp command building) moved into static class methods; generateSshCommand()/generateScpCommand… |
| 10 | [`d632eb2be9`](https://github.com/coollabsio/coolify/commit/d632eb2be989d2447f9e73a64a831e3f411cf66f) | 2025-02-04 | Start/restart sequencing moves from the Livewire navbar into the StartService action. | C3 | role | body | Navbar's start/restart/forceDeploy/pullAndRestart sequences (`$service->parse()`, StopService::run, PullImage::run, StartService::run) collapsed into StartService flags (pullLatestImages, stopBeforeStart); PullImage del… |
| 11 | [`e891318396`](https://github.com/coollabsio/coolify/commit/e891318396575b51cc44d33e279eb8fbef5d2614) | 2025-03-24 | Runtime config mutation is pulled out of a notification channel and helpers into an injected service. | C2 | role | dependency | EmailChannel: 6 `config()->set('mail...')` calls -> 0, ConfigurationRepository constructor-injected; helpers/notifications.php 8 config() calls -> 0; Server model and Boarding swap `config()->set('constants.ssh.mux_enab… |
| 12 | [`8c5c249c6a`](https://github.com/coollabsio/coolify/commit/8c5c249c6abf6666dabe3e3a7d4efe47a7274834) | 2025-09-05 | Webhook controller hands preview teardown to the existing deletion job. | C3 | role | body | `$found->forceDelete()` + `instant_remote_process(["docker rm -f $container_name"])` replaced by `DeleteResourceJob::dispatch($found)` in 2 handlers; controller keeps ApplicationPreview model and one other instant_remot… |
| 13 | [`e74da06465`](https://github.com/coollabsio/coolify/commit/e74da06465460d5f3446fe24e33156b8826655cf) | 2025-09-11 | Two Livewire components' copy-pasted cloning persistence moves into one global helper. | C3 | role | body | ~140 duplicated lines in each component of ->replicate()/->save() for app, settings, tags, tasks, previews, volumes, env vars plus StopApplication/VolumeCloneJob dispatch moved into global clone_application(); component… |
| 14 | [`0540b2eae5`](https://github.com/coollabsio/coolify/commit/0540b2eae567b344661f26f4c154be77b1efc20f) | 2025-11-18 | Buildpack-switch cleanup moves from the Livewire handler into the model's saving hook. (AI co-authored) | C3 | role | body | updatedBuildPack() lost the switch-away-from-dockercompose cleanup (null docker_compose_* fields, 4 `environment_variables()->where('key','LIKE','SERVICE_FQDN_%')->delete()` calls); moved to Application::booted saving h… |
| 15 | [`70fb4c6869`](https://github.com/coollabsio/coolify/commit/70fb4c6869ed9e72ed62b9e409f29819d42632b9) | 2025-11-20 | Model's duplicated status-aggregation rules replaced by the shared service. (AI co-authored) | C3 | role | body | ~200 lines of hand-written status state machine in Service::getStatusAttribute replaced by calls to existing App\Services\ContainerStatusAggregator; the model GAINS `use App\Services\ContainerStatusAggregator`, no refer… |
| 16 | [`d27070b215`](https://github.com/coollabsio/coolify/commit/d27070b215fa96edf0fcae2650e6a2b6dd9e11a2) | 2025-12-08 | PR-closed cleanup (DB updates plus SSH container kills) moves out of four webhook controllers into one action. (AI co-authored) | C2 | role | dependency | lines naming instant_remote_process or ApplicationDeploymentQueue in Webhook controllers: 10 (Github 7, others 1 each) -> 0; inline cancel-deployment `->update()`, `docker ps`/`docker rm -f` SSH and container sweep move… |
| 17 | [`f8f27fff13`](https://github.com/coollabsio/coolify/commit/f8f27fff138fea0b03121b14fa002459328a620f) | 2026-03-23 | Duplicated cron-dedup logic in two scheduler jobs moves into a global helper. | C3 | role | body | identical private shouldRunNow() (~45 lines, CronExpression + Cache-based dedup) removed from 2 jobs into a global helper; both drop `use Cron\CronExpression`, ServerManagerJob also drops the Cache facade, ScheduledJobM… |
| 18 | [`3e0d48faea`](https://github.com/coollabsio/coolify/commit/3e0d48faeaab950bfd063dfca908f1d140316ede) | 2026-03-26 | Reverse direction: two single-consumer layers (action + DTO) collapsed back into the global helper. (AI co-authored) | C1 | location | dependency | helper drops `use App\Actions\CoolifyTask\PrepareCoolifyTask;` and `use App\Data\CoolifyTaskArgs;` (both deleted, single consumer); activity()->log + `dispatch(new CoolifyTask(...))` now inline in remote_process(); othe… |
| 19 | [`bf72fd5d88`](https://github.com/coollabsio/coolify/commit/bf72fd5d8823c788d5f1f8eb18898f9839ed6546) | 2026-07-19 | Storage-deletion guard moves from three API controllers into the two volume models. | C3 | role | body | identical `if ($storage->scheduledBackups()->exists()) return response()->json([...], 422)` rule removed from 3 API controllers; models gain abortIfScheduledBackupsExist() that calls global `abort(422, ...)` (an HTTP co… |
| 20 | [`cbee84353e`](https://github.com/coollabsio/coolify/commit/cbee84353e2357f320dc529d158ba7823517d85e) | 2026-09-29 | Broadcast events stop reading the session user and receive the team explicitly. (AI co-authored) | C2 | role | dependency | event constructors fell back to `auth()->user()->currentTeam()->id`; auth() removed (event files using auth(): 14 before, 1 after, DatabaseStatusChanged); team id now a required ctor arg via BroadcastsToTeam; StopDataba… |

<details><summary>Coolify: 21 diffs read that did not qualify</summary>

| Commit | Date | Subject | Why not |
| --- | --- | --- | --- |
| [`e7763f3b73`](https://github.com/coollabsio/coolify/commit/e7763f3b7361c14acd454a04d46f465d01e55319) | 2023-05-03 | Refactor for CoolifyTask. | renames (RemoteProcess actions/DTO/job -> CoolifyTask names; runRemoteCommandSync -> instantRemoteProcess helper); no layer or dependency change |
| [`392c1650db`](https://github.com/coollabsio/coolify/commit/392c1650db38351c81e7718f70087eadf4d4a513) | 2023-08-17 | separate backups on fs & s3 layer | changes the on-disk backup directory path; 'layer' is a filesystem path, not code structure |
| [`a4f1fcba58`](https://github.com/coollabsio/coolify/commit/a4f1fcba5820f891cac23cf8d3ae5915ffc70477) | 2023-10-11 | move subscription to livewire + show manage subscription button for people already subscr… | feature/UI: controller page action replaced by a full-page Livewire component plus a new button |
| [`5c27f43b3d`](https://github.com/coollabsio/coolify/commit/5c27f43b3dcd98ad24aefdcc82486af7d0517bad) | 2023-10-12 | move autoupdate job to actions | mechanical adoption of lorisleiva AsAction (__invoke->handle, resolve(...)() -> ::run()); logic already lived in App\Actions\Server\UpdateCoolify |
| [`871d09bd96`](https://github.com/coollabsio/coolify/commit/871d09bd96602f91f12187d5b8fc40efb743030d) | 2024-09-17 | Feat: Move more functions to the PrivateKey Model | feature: adds a duplicate-fingerprint check and query helpers to PrivateKey; nothing removed from any other class |
| [`83549965ca`](https://github.com/coollabsio/coolify/commit/83549965ca733227f520ddb3c316af4333bc071a) | 2024-10-01 | Refactor instanceSettings() function for improved code readability | mechanical swap of `InstanceSettings::get()` for the equivalent instanceSettings() global helper in 31 files (drops 13 `use App\Models\InstanceSettings`); read… |
| [`8fbd0b44d5`](https://github.com/coollabsio/coolify/commit/8fbd0b44d5fd7e87a69e4c305037b87015747d49) | 2024-10-30 | Refactor Server model to remove SSH multiplexing file and add ServerCleanupMux job | feature: adds an hourly mux-cleanup job to the scheduler; Server model untouched in this commit |
| [`7fb1a1fc75`](https://github.com/coollabsio/coolify/commit/7fb1a1fc759ab0ba3b171c80839df3443cb7bc4e) | 2024-11-04 | fix: use Auth(), add new db proxy stop event refactor clickhouse view | style: auth()->user()->id -> Auth::id() facade swap in 14 files plus a new event; no dependency direction change |
| [`376a2341af`](https://github.com/coollabsio/coolify/commit/376a2341af833fcfc11f2d03d1cccd59edebfef7) | 2024-11-07 | refactor scheduled task job (and related stuffs) | mixed Livewire modernisation (typed props, #[Locked], mount params instead of request()->route()), team scoping and a new executeNow feature; no layer move |
| [`5877cead89`](https://github.com/coollabsio/coolify/commit/5877cead8912e4fd10fdd49557efc298e9973047) | 2025-01-08 | feat: move volume data cloning to a Job | only adds VolumeCloneJob inside a same-day feature series (3723c846, e01401a4); no code removed elsewhere |
| [`7582d7dd8b`](https://github.com/coollabsio/coolify/commit/7582d7dd8b6a3a2b72b9cbd0bbeb034cce25b366) | 2025-01-10 | refactor: streamline job status retrieval and clean up repository interface | dead-code removal from CustomJobRepository + interface, and HorizonManage::getJobStatus de-duplicated onto the existing getJobStatus() helper |
| [`57f61d4589`](https://github.com/coollabsio/coolify/commit/57f61d45895e57da75457cf438233f45ad01b3b8) | 2025-01-20 | fix: docker image parser | bug fix: a 6-line image:tag split in a Livewire component replaced by a new DockerImageParser service with corrected registry/port parsing |
| [`93ec2f804b`](https://github.com/coollabsio/coolify/commit/93ec2f804b02278980e2bb85b4a8e49dd5924a38) | 2025-04-13 | refactor(commands): reorganize OpenAPI and Services generation commands into a new namesp… | rename into a Commands/Generate subfolder; grouping inside the same role folder, no dependency change |
| [`5bdf2e8481`](https://github.com/coollabsio/coolify/commit/5bdf2e84816f5bfa195b1e1c06db6a27e3d60df9) | 2025-07-14 | refactor(previews): move preview domain generation logic to ApplicationPreview model for… | method moved between two models of the same layer (Application -> ApplicationPreview, which reference each other via relations); encapsulation, no layer or dep… |
| [`f4e5c195fe`](https://github.com/coollabsio/coolify/commit/f4e5c195fe8eb5436395e4ee030729f3c31f73c8) | 2025-10-09 | refactor: replace direct SslCertificate queries with server relationship methods for cons… | query-style consistency (SslCertificate::where('server_id') -> $server->sslCertificates()); 7 Livewire components drop `use App\Models\SslCertificate` but keep… |
| [`0865ecd3db`](https://github.com/coollabsio/coolify/commit/0865ecd3dbe96350a315c48c0280d9b4f1193b72) | 2025-11-05 | refactor: move RestoreDatabase command to Cloud namespace | PSR-4 namespace correction for a file already in Commands/Cloud/; grouping inside the same role folder |
| [`4002044877`](https://github.com/coollabsio/coolify/commit/40020448773a5200de8f0c8dd840c6cb21a2cf77) | 2025-12-04 | Refactor: Move sentinel update checks to ServerManagerJob and add tests for hourly dispat… | scheduling behaviour change (hourly, independent of update_check_frequency): a dispatch loop moves from Console Kernel into an existing scheduler job; Kernel k… |
| [`b8226be774`](https://github.com/coollabsio/coolify/commit/b8226be774eb2fc5cc735d11bd85e250347a7ac4) | 2026-04-28 | refactor(server): dispatch event for reachability notifications, drop retry loop | behaviour change: drops a blocking sleep/retry loop in Server::isReachableChanged and fires an event from ServerConnectionCheckJob |
| [`79174b749d`](https://github.com/coollabsio/coolify/commit/79174b749d8e59bf8cc5d3828ecc7ae051837705) | 2026-04-30 | refactor(helpers): extract STANDALONE_DATABASE_MODELS registry, add tests | DRY inside global helpers (8 per-model if-blocks -> loop over a constant); same layer |
| [`85ec70643f`](https://github.com/coollabsio/coolify/commit/85ec70643f4a5960b5ccce694b8de7b5425a563c) | 2026-09-04 | refactor(notifications): build scheduled task links from the instance url | link-building behaviour change (base_url() instead of route() plus instance-fqdn host swap); taskLink() deleted from Application/Service, which keep route() in… |
| [`3452faec87`](https://github.com/coollabsio/coolify/commit/3452faec87b4e8015814892f34a58ede255e4b6f) | 2026-09-29 | fix(livewire): bind team broadcast listeners to the mounted team | bug fix (team switch in another tab); listener building moved into a Livewire trait (same role) that still calls currentTeam() |

</details>

#### BookStack: 30 qualifying of 40 diffs read

| # | Commit | Date | What was fixed | Class | Tag | Kind | Evidence (abridged) |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | [`3d18a04c39`](https://github.com/BookStackApp/BookStack/commit/3d18a04c3932f1724a67152e89619dda42a5e7cf) | 2015-09-04 | Social login (Socialite driver use + env-based driver config) moved out of the auth controller into a service. | C2 | role | dependency | AuthController removed `use Laravel\Socialite\Contracts\Factory as Socialite` + ctor injection, 3x env('<DRIVER>_APP_ID'...) config checks and driver validation; all moved to new SocialAuthService which the controller n… |
| 2 | [`8e274a5a84`](https://github.com/BookStackApp/BookStack/commit/8e274a5a84cc11c20c0d62f9ecc0b08cd9410305) | 2016-03-02 | Role CRUD and permission syncing moved from a controller that injected models into a new PermissionsRepo. | C1 | role | dependency | PermissionController ctor injected models `BookStack\Role` + `BookStack\Permission` and did where()/newInstance()/save()/permissions()->sync() + admin/registration-role rules; fix removes both model imports, injects new… |
| 3 | [`be2ca9d4bb`](https://github.com/BookStackApp/BookStack/commit/be2ca9d4bbc8850ab70fce9549cf1087d4e4d721) | 2018-07-15 | Hand-built LdapRepo (service-locator + new in controller) folded into LdapService and injected. | C1 | role | dependency | LoginController had `use BookStack\Repos\LdapRepo` + `new LdapRepo($this->userRepo, app(LdapService::class))`; fix deletes LdapRepo (a non-repository living in Repos/ that wrapped LdapService + config()), folds group sy… |
| 4 | [`919660678b`](https://github.com/BookStackApp/BookStack/commit/919660678bec2b94eaa84ac60d0313f5ef07dfb7) | 2018-09-25 | Reorganised from code-type folders (Services, Repos, root models) into feature folders. | C1 | location | placement | 109 files: root models (Book, Page, User, Role...), app/Services/* and app/Repos/* renamed into feature folders Actions/, Auth/(Access, Permissions), Entities/, Settings/, Uploads/; Facades pulled out of Services; contr… |
| 5 | [`f7a5a0705b`](https://github.com/BookStackApp/BookStack/commit/f7a5a0705bba304a705df74b1e7e724827900a0c) | 2019-09-19 | Shelf-append relation logic moved from the repo onto the Bookshelf model. | C3 | role | body | BookRepo::appendBookToShelf (books()->max('order') + books()->attach) moved to model method Bookshelf::appendBook; BookRepo drops `use BookStack\Entities\Bookshelf` but keeps Book model; repos legitimately name models |
| 6 | [`2a2cc858f0`](https://github.com/BookStackApp/BookStack/commit/2a2cc858f0f95c606f2ad2542f915d3bd761775f) | 2019-09-19 | Global view-data sharing moved from base controller to middleware; notification flashing centralised. | C3 | role | body | base Controller ctor middleware closure doing view()->share('signedIn'/'currentUser') moved into new GlobalViewData middleware; ~40 session()->flash() calls wrapped by base-controller showSuccess/Warning/ErrorNotificati… |
| 7 | [`8b550991a4`](https://github.com/BookStackApp/BookStack/commit/8b550991a4dbcd7a05e08e8f600b5444d5f4004d) | 2019-09-20 | Slug generation and permission rebuild moved out of EntityRepo into the entity models and a SlugGenerator class. | C3 | role | body | EntityRepo slugExists/nameToSlug/findSuitableSlug moved to new SlugGenerator; repo calls `$this->permissionService->buildJointPermissionsForEntity()` replaced by Entity::rebuildPermissions() (model now uses new Permissi… |
| 8 | [`31f5786e01`](https://github.com/BookStackApp/BookStack/commit/31f5786e01fc5ed439f347c6979679612baca4fb) | 2019-10-05 | Large pass moving persistence and tree/sort logic out of entity controllers and the god EntityRepo into per-entity repos and manager classes. | C3 | role | body | squashed 72-file PR: BookController drops UserRepo + Uploads\ImageRepo; cover-image save (imageRepo->saveNew + $book->save()) moved to BookRepo::updateCoverImage, ~80-line sort algorithm moved to BookSortController/Book… |
| 9 | [`349b4629be`](https://github.com/BookStackApp/BookStack/commit/349b4629bef25e4631c1024749726415507b2cd5) | 2019-12-30 | API token authentication moved from middleware (which queried the ApiToken model) into a dedicated auth guard. | C1 | role | dependency | ApiAuthenticate removed `use BookStack\Api\ApiToken`, `use Hash`, `ApiToken::query()->where('token_id'..)`, Hash::check and access-api check; now only auth()->shouldUse('api') + auth()->authenticate(); token logic in ne… |
| 10 | [`92690d1ae9`](https://github.com/BookStackApp/BookStack/commit/92690d1ae92363467c6a49664a558320b8b5771b) | 2020-01-26 | Registration business rules moved from the register controller into a RegistrationService; social routes split to their own controller. | C3 | role | body | registerUser() (setting('registration-restrict') domain check, userRepo->registerNew, socialAccounts()->save, confirmation send, auth()->login) moved to new RegistrationService; RegisterController drops UserRepo/EmailCo… |
| 11 | [`e287d965f5`](https://github.com/BookStackApp/BookStack/commit/e287d965f5ed6d72bb5e83fa655207f96e3257df) | 2020-05-13 | Zip-building (ZipArchive) moved out of the export controller into the export service. | C2 | role | dependency | BookExportController::zip removed `use ZipArchive` + `new ZipArchive()` and the addEmptyDir/addFromString loop; moved to ExportService::bookToZip. HEAD: 0/74 controllers reference ZipArchive (3 non-controller classes do) |
| 12 | [`9d7ce59b18`](https://github.com/BookStackApp/BookStack/commit/9d7ce59b18c5cbeef017349f38f063a63b762188) | 2020-05-23 | Failed-login logging moved from the login controller into the activity service. | C3 | role | body | LoginController::logFailedAccess (config('logging.failed_access_message') + error_log()) moved to ActivityService, called via Activity facade; controller still calls config() 4x and session() after |
| 13 | [`c157dc3490`](https://github.com/BookStackApp/BookStack/commit/c157dc3490c39aa236e22714019a649ebeb11f13) | 2020-11-07 | Activity logging orchestration moved from controllers into repos so web and API share it. | C3 | role | body | Activity::add(..., 'book_create'/'chapter_update'...) calls removed from web + API controllers and added inside repo create/update/destroy (dup between web and API); ChapterController drops `use Activity`, BookControlle… |
| 14 | [`c7a2d568bf`](https://github.com/BookStackApp/BookStack/commit/c7a2d568bf693add30c8402d68d1f46f09a44c5b) | 2020-11-21 | Eloquent models inside the Entities domain gathered into a Models subfolder; helper classes into Tools. | C2 | role | placement | 10 classes moved app/Entities -> app/Entities/Models; Managers/* + SearchOptions + SlugGenerator -> Tools/. All but Deletion extend app base BookStack\Model; Deletion extends Illuminate Eloquent Model directly (so only… |
| 15 | [`99b14621f9`](https://github.com/BookStackApp/BookStack/commit/99b14621f9bb5f8f02a16a15beb7576a73b2f550) | 2021-01-01 | Entity permission updating moved from the repos (and controllers' request parsing) into a dedicated tool. | C3 | role | body | BaseRepo::updatePermissions (permissions()->delete/createMany, save, rebuildPermissions, Activity::addForEntity) + 4 per-repo wrappers moved to new PermissionsUpdater; request parsing moved out of 4 controllers into upd… |
| 16 | [`9e7bcacf8c`](https://github.com/BookStackApp/BookStack/commit/9e7bcacf8c5615526add66e950407d02ec1ac185) | 2021-05-08 | Rendering of NotifyException moved from the global exception handler into the exception itself. | C3 | role | body | Handler::render's NotifyException branch (session()->flash + redirect) and isExceptionType/getOriginalMessage helpers removed; NotifyException now implements Illuminate\Contracts\Support\Responsable::toResponse; Handler… |
| 17 | [`a641b4da2c`](https://github.com/BookStackApp/BookStack/commit/a641b4da2c08e9533a20174d7a3543694f6b7c4c) | 2021-08-31 | Service swapped an injected DB connection for the DB facade to delay connection opening. | C2 | role | dependency | removed ctor-injected `Illuminate\Database\Connection as Database` ($this->db->table(...) x4) and replaced with `Illuminate\Support\Facades\DB::table`; REVERSE of the usual direction: injection opened the DB connection… |
| 18 | [`3f9527f166`](https://github.com/BookStackApp/BookStack/commit/3f9527f166740b8580adbf513600fd46104f47d0) | 2021-12-19 | Page copy logic moved from the repo into a Cloner tool, with the permission check lifted to the controller. | C3 | role | body | PageRepo::copy (new draft, tag copying, publishDraft) moved to new Cloner tool; its userCan('page-create') check moved up into PageController::checkOwnablePermission; PageRepo still calls userCan() twice afterwards |
| 19 | [`abc283fc64`](https://github.com/BookStackApp/BookStack/commit/abc283fc64b7ed3e79f5e499f934130b6146a02b) | 2022-06-08 | Download/streaming response construction moved out of the base controller into a response factory. | C3 | role | body | 4 download/stream response builders removed from base Controller with imports `BookStack\Util\WebSafeMimeSniffer`, `Symfony\...\StreamedResponse`, `Illuminate\Http\Response`; Controller now returns `new DownloadResponse… |
| 20 | [`837fd74bf6`](https://github.com/BookStackApp/BookStack/commit/837fd74bf691129467361ce8066ba31f64408647) | 2022-08-16 | Search code pulled out of the Entities domain into its own Search domain. | C1 | location | placement | 5 search classes moved out of the Entities domain (Tools, Models) into new top-level BookStack\Search; consumers (controllers, RegenerateSearch command, Entity model, PermissionApplicator) only update imports |
| 21 | [`f4388d5e4a`](https://github.com/BookStackApp/BookStack/commit/f4388d5e4a6353f6f79e03c638ed2971fbf48c54) | 2022-09-22 | App-wide removal of the laravel/ui package dependency from the auth controllers. | C2 | location | dependency | auth controllers stop using laravel/ui traits such as `Illuminate\Foundation\Auth\AuthenticatesUsers` and `SendsPasswordResetEmails`; package dropped from composer.json; behaviour reimplemented as app methods + app-owne… |
| 22 | [`295cd01605`](https://github.com/BookStackApp/BookStack/commit/295cd0160525125bbd7756d7ad07392ae7201cb8) | 2023-05-17 | Big restructure from role folders (Http/Controllers, Interfaces, Traits) into domain folders. | C1 | location | placement | 207 files: Http/Controllers (+Api, Auth, Images) moved into per-domain Controllers folders; Auth/* split into Access, Users, Permissions; Actions -> Activity (+Models, Tools); Interfaces/ and Traits/ dissolved into owni… |
| 23 | [`8e3f8de627`](https://github.com/BookStackApp/BookStack/commit/8e3f8de6278c455d529f14e8dfb245dd81e3d1aa) | 2023-09-11 | Notifications moved from a role folder into the domains that send them. | C1 | location | placement | role folder app/Notifications dissolved: notifications moved into owning domains (Access, Settings), base MailNotification -> App; classes gain Notification suffix; consumers only update imports |
| 24 | [`222c665018`](https://github.com/BookStackApp/BookStack/commit/222c665018cd7fc231d2970307e3a7423e4a377f) | 2024-02-05 | Read queries split out of PageRepo into a PageQueries class; repos keep writes. | C3 | role | body | PageRepo read methods getById/getBySlug/getByOldSlug/getTemplates (Page::visible() queries) removed (-87 lines); 12 controllers inject PageQueries instead (template listing query now built in PageTemplateController); Pa… |
| 25 | [`546cfb0dcc`](https://github.com/BookStackApp/BookStack/commit/546cfb0dcc801c4fa6560ca0e6d18b4f1edd830f) | 2024-02-07 | Static model query-scope calls across controllers, repos and tools replaced by injected query classes. | C3 | role | body | `Page::visible()->find..`, `Bookshelf::visible()` static scope calls in 15 files replaced by injected PageQueries/BookshelfQueries; the `use ...Entities\Models\Page` imports stayed (removed as unused in b77ab6f3); Image… |
| 26 | [`b77ab6f3af`](https://github.com/BookStackApp/BookStack/commit/b77ab6f3af996f924bd2ddbac6e88f61a8bf6cf9) | 2024-02-07 | Query helpers moved off the entity models into the Queries classes. | C3 | role | body | model scopeWhereSlugs, static getBySlugs and static $listAttributes/$contentAttributes moved into Queries classes; SearchRunner drops `use BookStack\Entities\Models\BookChild` and builds via EntityQueries; plus app-wide… |
| 27 | [`42b9700673`](https://github.com/BookStackApp/BookStack/commit/42b9700673e7b2e5a04c9f888a05d98261ed36e3) | 2024-10-15 | Export code moved out of the Entities domain into its own Exports domain. | C1 | location | placement | 6 export controllers + ExportFormatter + PdfGenerator moved from the Entities domain into new BookStack\Exports domain (routes updated); same commit starts ZipExportBuilder (feature) |
| 28 | [`5b0cb3dd50`](https://github.com/BookStackApp/BookStack/commit/5b0cb3dd506c108b5d5d13c5c07c4f02e6107608) | 2025-01-29 | Single-use global helper turned into a class in the Sorting domain. | C1 | location | placement | global helper function sortUrl() removed from app/App/helpers.php; replaced by class BookStack\Sorting\SortUrl built in AuditLogController and passed to the view (its only call site) |
| 29 | [`ac0cd9995d`](https://github.com/BookStackApp/BookStack/commit/ac0cd9995d8b420e33e392ba82d40bde8df94205) | 2025-01-29 | Book sorting code moved out of the Entities domain into a Sorting domain. | C1 | location | placement | BookSortController, BookSortMap, BookSortMapItem moved from Entities into new BookStack\Sorting; ~200 lines of sort-application logic extracted from Entities\Tools\BookContents into new Sorting\BookSorter |
| 30 | [`dead0bfb5f`](https://github.com/BookStackApp/BookStack/commit/dead0bfb5fd5b725230a5020973ddd4fa9560a95) | 2026-08-07 | Sidebar view-data assembly moved out of the book controller into pluggable view sidebar-section classes. | C3 | role | body | BookController index/show stop building popular/new lists, parent shelves, watch options, activity, reference count; drops imports Activity\ActivityQueries, Activity\Tools\UserEntityWatchOptions, References\ReferenceFet… |

<details><summary>BookStack: 10 diffs read that did not qualify</summary>

| Commit | Date | Subject | Why not |
| --- | --- | --- | --- |
| [`6bcd89acf7`](https://github.com/BookStackApp/BookStack/commit/6bcd89acf76163ca09c95903f27ceed925718061) | 2017-07-22 | Moved utf8mb4 migration to command instead of migration | operational/behaviour change: DB-encoding conversion removed from a migration (it broke live upgrades) and replaced by a command that prints SQL; not about pla… |
| [`5c2e3f4e56`](https://github.com/BookStackApp/BookStack/commit/5c2e3f4e565b6e7626e1ca5aa83d6ebe4e56e940) | 2018-09-22 | Extracted download response logic into controller method | DRY within the controller role: repeated response()->make download headers moved into a base Controller method; also a header bug fix (#581) |
| [`85f330c79a`](https://github.com/BookStackApp/BookStack/commit/85f330c79a14d0393af46d5ccb1105b822950d91) | 2018-10-13 | Extracted many page-specific repo methods into page-specific repo | split of a god EntityRepo into a PageRepo within the same repo layer (plus EntityRepo moved into Entities/Repos); controllers only switch which repo they call |
| [`e5155a5dcb`](https://github.com/BookStackApp/BookStack/commit/e5155a5dcb3553da589ac2955989c32315ddf79a) | 2019-08-18 | Refactored confirm actions to their own controller | controller split within the same role (RegisterController -> ConfirmEmailController); only a 2-line setting() check moved into EmailConfirmationService |
| [`60d0f96cd7`](https://github.com/BookStackApp/BookStack/commit/60d0f96cd754a85c637a5f34dcc0341f4ef72a46) | 2019-09-15 | Extracted some methods into a BookRepo | split of EntityRepo into an entity-specific BookRepo subclass within the repo layer; controllers just rename calls |
| [`9bba84684f`](https://github.com/BookStackApp/BookStack/commit/9bba84684fa53a6c4f38c4fd7cea08552208b228) | 2019-11-16 | Appeased codeclimate by extracting out external_auth_id group matching | method extraction inside ExternalAuthService for complexity score + Role docblock; no placement/dependency change |
| [`2d4f708c79`](https://github.com/BookStackApp/BookStack/commit/2d4f708c7961f7ef7da3fd1d89d39a99581afa9e) | 2022-07-12 | Extracted permission building out of permission service | split of PermissionService into JointPermissionBuilder within the same service layer; repo/command just inject the builder (PermissionsRepo also swaps injected… |
| [`a6a78d2ab5`](https://github.com/BookStackApp/BookStack/commit/a6a78d2ab5bd36caadcaf63948a26ea34dacf713) | 2022-09-27 | Refactored app service providers | provider housekeeping within the provider role: closure bindings -> $singletons/$bindings, unused providers removed, view tweaks moved to ViewTweaksServiceProv… |
| [`ec4cbbd004`](https://github.com/BookStackApp/BookStack/commit/ec4cbbd0041122f30d13afc016f8a5fac72e0754) | 2022-10-30 | Refactored common list handling operations to new class | DRY: per-controller sort/search option arrays replaced by a BookStack\Util\SimpleListOptions value object; controllers still call setting(); plus UI feature |
| [`f732ef05d5`](https://github.com/BookStackApp/BookStack/commit/f732ef05d5b60a48ce3b42e05155e9e91d92d927) | 2024-10-23 | ZIP Exports: Reorganised files, added page md parsing | reshuffle of still-unreleased ZIP export feature files inside the Exports domain (ZipExportModels -> ZipExports/Models) during development; not fixing shipped… |

</details>

#### Advising App: 29 qualifying of 44 diffs read

| # | Commit | Date | What was fixed | Class | Tag | Kind | Evidence (abridged) |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | [`8a861c5ad4`](https://github.com/canyongbs/advisingapp/commit/8a861c5ad417e3da6914e4f904e0bfbd82a31b7c) | 2023-07-25 | First modularisation step: case-domain models moved out of app/ into a new case module. | C1 | location | placement | 5 case models renamed from `App\Models` to `Assist\CaseModule\Models`; app/ Livewire components and controllers switch imports to the module namespace. Domain models sat in the core skeleton instead of their module. |
| 2 | [`0b697a5ebe`](https://github.com/canyongbs/advisingapp/commit/0b697a5ebeec8db5fee520b416de7014312c4840) | 2023-08-15 | Audit settings page moved from core app/ into the audit module that owns its settings. | C1 | location | placement | Core page in `App\Filament\Pages` imported module class `Assist\Audit\Settings\AuditSettings`; page moved into the audit module (registered via new AuditPlugin), so app/ no longer names the audit module. |
| 3 | [`5e00f9f7fb`](https://github.com/canyongbs/advisingapp/commit/5e00f9f7fb71ba7c22238c9155713cfcf405028e) | 2023-09-13 | Service locator call in a Filament/Livewire page replaced by method injection. | C2 | role | dependency | Filament page method `ask()` did `$ai = app(AIInterface::class)`; now `ask(AIInterface $ai)` method injection. Removed global helper `app()` service-location call; contract still named. |
| 4 | [`b409c0f602`](https://github.com/canyongbs/advisingapp/commit/b409c0f6020f20ab6253504fe62ecb23c91da14d) | 2023-09-22 | Generic timeline infrastructure extracted from the engagement module into its own module. | C1 | location | placement | Generic Timeline page + Timelineable contracts lived in engagement; assist-data-model StudentEngagementTimeline imported `Assist\Engagement\...\Pages\Timeline`. Split into new timeline module; importers now use `Assist\… |
| 5 | [`ae829cc5a2`](https://github.com/canyongbs/advisingapp/commit/ae829cc5a28ca9d433993bd9d1e0bcc242989915) | 2023-10-19 | Embed-authorization checks moved out of the widget controller into a route middleware. | C3 | role | body | Controller still binds `Form $form`; 18 lines of embed/referer authorization (`$form->embed_enabled`, allowed_domains, `config('app.url')`) moved into new middleware. Removed `config()` call is incidental. |
| 6 | [`5792abb26d`](https://github.com/canyongbs/advisingapp/commit/5792abb26d01b5f75b8b9a6f0834f992274d5391) | 2023-10-20 | FormKit schema generation moved from controller into an action class. | C3 | role | body | Controller keeps `Form` binding; inline `match ($field['type'])` building FormKit schema arrays (20+ lines) moved to an action; controller now adds imports (FormField, the action). Nothing removed at class level. |
| 7 | [`21850503fc`](https://github.com/canyongbs/advisingapp/commit/21850503fc36e19126f1b362593ddedaf6e6e86c) | 2023-11-22 | Email template resource/model relocated from core app/ into the engagement module (with a model redesign). | C1 | location | placement | Resource + pages moved app/ -> engagement module; division module's Division model dropped `use App\Models\Concerns\HasEmailTemplates`. Mixed with redesign: model changed from branding (logo, colour) to content template. |
| 8 | [`9536b57956`](https://github.com/canyongbs/advisingapp/commit/9536b579566010bc6ea9207060d116abd8f28917) | 2023-12-20 | Delivery-result persistence moved from the base notification class into the channel classes. | C3 | role | body | BaseNotification keeps `OutboundDeliverable`/`NotificationDeliveryStatus` imports; `afterSendSms` with `$deliverable->update([...delivery_status...])` (TODO: extract) moved to static `SmsChannel::afterSending`. Also rem… |
| 9 | [`725a52feca`](https://github.com/canyongbs/advisingapp/commit/725a52feca8dde6d28612dfe431e33d8c05e79fc) | 2024-01-09 | Filament cluster classes moved from the Pages folder to the Clusters folder. | C2 | role | placement | 12 classes that `extends Filament\Clusters\Cluster` sat in `App\Filament\Pages`; moved to `App\Filament\Clusters`. Kind defined by vendor base class. |
| 10 | [`5a664e6d3e`](https://github.com/canyongbs/advisingapp/commit/5a664e6d3e59b8cb16a38c64ff0b4efb77ab407d) | 2024-01-23 | Tenancy controller/middleware grouped under the Multitenancy domain folder. | C1 | location | placement | Landlord-only controller (already importing `App\Multitenancy\Actions\CreateTenant`, DTOs, Requests) and middleware moved from generic App\Http into the App\Multitenancy domain folder. |
| 11 | [`119a7aa51c`](https://github.com/canyongbs/advisingapp/commit/119a7aa51cb76844a07d36ce13787b620e13fb71) | 2024-01-23 | Approval business operation moved out of the Eloquent model into an action. | C3 | role | body | Model keeps `User` + relations; `approvedBy(User)` doing `$this->responses()->create([...approved...])` (with TODO "extract to an independent action") moved to ApproveChangeRequest action; page calls the action. |
| 12 | [`26795aa641`](https://github.com/canyongbs/advisingapp/commit/26795aa64170d9926337f186337b02e5791b6951) | 2024-04-17 | Vendor-override job moved into the app's Overrides folder convention. | C2 | role | placement | Class `extends Filament\Actions\Imports\Jobs\ImportCsv` (vendor override) sat in App\Jobs; moved to App\Overrides mirroring the vendor namespace and bound in AppServiceProvider like the other overrides. |
| 13 | [`4624f6b02b`](https://github.com/canyongbs/advisingapp/commit/4624f6b02bd467c12a608bc05223c3ed7c946bc2) | 2024-05-09 | Replication logic moved out of three models into dedicated actions, called from List pages. | C3 | role | body | ~75 lines each of `replicateRelatedData`/replicateSteps/replicateFields (`->replicate()`, `->save()`) moved from 3 models to Duplicate* actions; models keep related-model refs. Removed `App\Models\Contracts\CanBeReplica… |
| 14 | [`513631e556`](https://github.com/canyongbs/advisingapp/commit/513631e556c185c22843b72cfb89e26df9f4f758) | 2024-05-17 | Validation and authorization moved from AI controllers into form requests. | C3 | role | body | Controllers keep `AiThread` binding; inline `$request->validate([...])` and ownership check `$thread->user()->is(auth()->user())` moved into FormRequests. Removed `Illuminate\Http\Request` import and `auth()` calls (inc… |
| 15 | [`62e6a4a05a`](https://github.com/canyongbs/advisingapp/commit/62e6a4a05a9720fe189eeb8d7c7ac7f43ac0e908) | 2024-05-20 | Module reorganisation moving prompt/chat-sharing classes into the ai module, breaking an ai<->assistant cycle. | C1 | location | placement | ai module traits CanManagePromptLibrary/CanManageThreads imported `AdvisingApp\Assistant\Models\Prompt`, `...\Assistant\Jobs\ShareAssistantChatsJob` while assistant depends on ai (cycle); 33 classes moved assistant -> a… |
| 16 | [`1fcb03c1c6`](https://github.com/canyongbs/advisingapp/commit/1fcb03c1c6078cf9df79ba6a7c30a082ab22acac) | 2024-06-05 | Thread-locked business rule moved from controllers into the actions. | C3 | role | body | Controllers keep `AiThread` binding and action calls; business rule `if ($thread->locked_at) throw new AiThreadLockedException()` moved from both controllers into the actions. |
| 17 | [`b721d0f1ce`](https://github.com/canyongbs/advisingapp/commit/b721d0f1ce3da02e82072b8bd3f35d68cd1586f1) | 2024-06-06 | Job-hosted static orchestration helper replaced by an action class. | C3 | role | body | Job had static `dispatchForAssistant()` building a `Bus::chain` + batch; Filament pages called it statically. Orchestration moved to new action injected into page closures; pages still dispatch the (renamed) job for mod… |
| 18 | [`5f5ff17e70`](https://github.com/canyongbs/advisingapp/commit/5f5ff17e701f331928afd2262682a8e048f6df15) | 2024-06-24 | Provider-specific DTOs moved out of the generic AI module into the OpenAI integration module. | C1 | location | placement | OpenAI-specific DTOs (vector stores, files, threads) moved from generic ai module to integration-open-ai; generic `AiService` contract drops `use ...ThreadsDataTransferObject` and retrieveThread/modifyThread methods. |
| 19 | [`835b8b7b61`](https://github.com/canyongbs/advisingapp/commit/835b8b7b61b0dee635bb7ef277e41679c375a8fe) | 2024-07-11 | Report widgets moved from the prospect module into the report module that uses them. | C1 | location | placement | report module page ProspectEnagagementReport imported 3 `AdvisingApp\Prospect\Filament\Widgets\*`; widgets moved into report module. Note: 386d6025f moves ProspectStats/ProspectTasks back 12 min later. |
| 20 | [`8e88bcaf0f`](https://github.com/canyongbs/advisingapp/commit/8e88bcaf0f593c054fa84a20a3c25358ff310a3c) | 2025-01-29 | Every module's notifications drop the in-house base class/channel traits for the framework Notification. | C1 | role-across-locations | dependency | e.g. AlertCreatedNotification removed `use ...Notification\Notifications\BaseNotification`, `DatabaseChannelTrait`, `implements DatabaseNotification`; now `extends Illuminate\Notifications\Notification` with `via()`. Cu… |
| 21 | [`f2a011da2b`](https://github.com/canyongbs/advisingapp/commit/f2a011da2b4c3f5e5219ded66c5402206285ec07) | 2025-02-12 | Queued jobs moved out of an Actions folder into Jobs. | C2 | role-across-locations | placement | Both classes `implements ShouldQueue` + use Dispatchable/Queueable but sat in `...\IntegrationTwilio\Actions`; moved to `...\Jobs`. Kind defined by vendor contract. |
| 22 | [`a633b045ff`](https://github.com/canyongbs/advisingapp/commit/a633b045ffa7b6c5eb5be01b25b5ad4eba4371d7) | 2025-03-12 | Local password settings relocated from the MFA module to the authorization module. | C1 | location | placement | Password-policy settings + page lived in the MFA module; moved to authorization; App\Observers\UserObserver switches `use ...MultifactorAuthentication\Settings\LocalPasswordSettings` to the Authorization one. Mixed with… |
| 23 | [`57d8f299cc`](https://github.com/canyongbs/advisingapp/commit/57d8f299ccc9a924d64b6d5d84ff609492b06619) | 2025-05-06 | Engagement-specific enum moved from the notification module into the engagement module. | C1 | location | placement | Enum in notification module imported `AdvisingApp\Engagement\Models\Engagement` (lower module naming higher); moved to engagement, where it now imports Notification enums instead (correct direction). |
| 24 | [`71d3612f00`](https://github.com/canyongbs/advisingapp/commit/71d3612f00e90703f92764dca601ad4229775622) | 2025-05-06 | Campaign-specific scope moved from core app/ into the campaign module. | C1 | location | placement | Core scope in `App\Models\Scopes` imported module model `AdvisingApp\Campaign\Models\CampaignAction`; moved into the campaign module. Rest of the commit is reindentation. |
| 25 | [`d0ef09d1d1`](https://github.com/canyongbs/advisingapp/commit/d0ef09d1d1517d10154fe4dedaadec905513a8f2) | 2025-05-14 | Queued job moved from Actions to Jobs. | C2 | role-across-locations | placement | Class `implements ShouldQueue, ShouldBeUnique` with Dispatchable sat in `AdvisingApp\Campaign\Actions`; namespace changed to `...\Campaign\Jobs`. |
| 26 | [`d9699f8975`](https://github.com/canyongbs/advisingapp/commit/d9699f897544b4f90cde688761c0e56541b29a94) | 2025-05-15 | Domain module stops depending on the campaign module; campaign execution moved into campaign jobs. | C1 | location | dependency | Interaction model removed `use AdvisingApp\Campaign\Models\CampaignAction`, `implements ...Campaign\Models\Contracts\ExecutableFromACampaignAction` and static `executeFromCampaignAction()`; logic now in a campaign-modul… |
| 27 | [`86eddf147e`](https://github.com/canyongbs/advisingapp/commit/86eddf147e30a0c3616d2c611148f62b2d74f7ff) | 2025-05-20 | Queued job moved from Actions to Jobs. | C2 | role-across-locations | placement | ShouldQueue class `ExecuteCampaignActions` moved `...\Campaign\Actions` -> `...\Campaign\Jobs`; Kernel import updated; unused artisan command deleted. At HEAD 1 ShouldQueue class remains in an Actions dir (form DeliverF… |
| 28 | [`76f99430d8`](https://github.com/canyongbs/advisingapp/commit/76f99430d8895387a79b6a09b7d1b28c7dfb7cf2) | 2025-05-23 | Recipient-deliverability checks moved out of the delivery job into the notification channels. | C3 | role | body | Job keeps Engagement/EngagementNotification refs; `canReceiveEmail()`/`canReceiveSms()` gating moved into channels (now records FailedDispatch event). Removed `use AdvisingApp\Notification\Enums\NotificationChannel` fro… |
| 29 | [`61fedc19dc`](https://github.com/canyongbs/advisingapp/commit/61fedc19dc250376f85443ce078d11aafd6d7fcb) | 2026-10-01 | Scheduler no longer loops tenants; per-tenant work fans out through orchestrator jobs. | C1 | role | dependency | Kernel removed `use App\Models\Tenant` + `Tenant::query()->each(...)` scheduling `tenants:artisan ... --tenant=` per tenant (also ~12 model imports, Log facade); now schedules landlord orchestrator jobs. 4 commands lose… |

<details><summary>Advising App: 15 diffs read that did not qualify</summary>

| Commit | Date | Subject | Why not |
| --- | --- | --- | --- |
| [`b10237bb1e`](https://github.com/canyongbs/advisingapp/commit/b10237bb1e509e54d28a076073e03bad51d70ec8) | 2023-10-13 | Refactor to use injected context | class shape: QueryBuilder operator methods take state from object instead of parameters (internal API redesign) |
| [`c4f73c784d`](https://github.com/canyongbs/advisingapp/commit/c4f73c784db0910ca4a535edcd9bd1244ab19d7e) | 2023-12-20 | Initial pass on unifying notifications. | stat-only: feature (new OutboundDeliverable delivery infrastructure); notifications->notification module rename incidental |
| [`a6757c31a0`](https://github.com/canyongbs/advisingapp/commit/a6757c31a0e9cce1baab7cdc8faef37e03b44d6a) | 2023-12-21 | Move Twilio configs into settings. | feature: Twilio credentials made admin-editable (DB settings + Filament page); `config('services.twilio.*')` -> `app(TwilioSettings::class)` adds a cross-modul… |
| [`ecbaa693cc`](https://github.com/canyongbs/advisingapp/commit/ecbaa693cc4e61aa8030eb955c04bca68d2bd35e) | 2025-02-06 | Refactor notification classes to remove OutboundDeliverable dependency in beforeSend meth… | data-model migration: legacy OutboundDeliverable model being retired in favour of EmailMessage/SmsMessage; not a misplaced dependency |
| [`538b322244`](https://github.com/canyongbs/advisingapp/commit/538b3222441ba1161954488e59f32f8cf776c7b5) | 2025-04-30 | Move to the correct spot | moves a feature-flagged model entry inside an array in AiApplication enum; not structural |
| [`b2720cb8d7`](https://github.com/canyongbs/advisingapp/commit/b2720cb8d76a32f59fa7ff226713996419e8e101) | 2025-05-06 | refactor: replace EmailMessageDisplayStatus with EngagementDisplayStatus in EngagementsRe… | rename in place (enum renamed inside notification module; the move followed in 57d8f299cc) |
| [`977a4cffa9`](https://github.com/canyongbs/advisingapp/commit/977a4cffa9f34a73a560384f549c20e17f1ef5f3) | 2025-06-04 | refactor: streamline recipient filtering and authorization for bulk email and text actions | simplification + added authorization on bulk actions; no layer change |
| [`70858f0fc0`](https://github.com/canyongbs/advisingapp/commit/70858f0fc0b705a2ad8a439bfda49fcbee34c275) | 2025-06-18 | [ADVAPP-1567]: Refactor APIs used for artificial intelligence with infrastructure provide… | stat-only: feature (OpenAI Responses API services, Prism handlers) |
| [`e9ac68dcf7`](https://github.com/canyongbs/advisingapp/commit/e9ac68dcf7e601e8cb027c0530d9fcc6132820e5) | 2025-08-13 | [ADVAPP-1805]: Refactor pipelines as a feature of an individual project, rather than a st… | stat-only: feature (pipelines nested under projects; ListPipelines page becomes ManageProjectPipelines in project module as a consequence) |
| [`3f59c98a15`](https://github.com/canyongbs/advisingapp/commit/3f59c98a15f322816dbfe2bac945aff7ceb58776) | 2025-10-10 | [ADVAPP-2014]: Update Filament resource structure: case-management module (#1980) | mechanical framework upgrade: Filament v4 resource directory convention (Resources/CaseForms/CaseFormResource.php); one of ~12 per-module commits |
| [`121ce2b732`](https://github.com/canyongbs/advisingapp/commit/121ce2b7329d1df3cb2d4cd1f7690e39088b82b8) | 2026-07-07 | [ADVAPP-2757]: Refactor and improve access to reporting (#2662) | stat-only: feature (report access models, Reporting page, feature flag) |
| [`9b5afcf913`](https://github.com/canyongbs/advisingapp/commit/9b5afcf91398c87b0093147c6dd2583af98f1704) | 2026-08-19 | [ADVAPP-2805]: Decouple notification branding from Division (#2768) | feature/data-model: per-division notification branding replaced by one global NotificationSettings (prep for deleting Division module) |
| [`41f3af99ba`](https://github.com/canyongbs/advisingapp/commit/41f3af99ba7905035cf3603fa68e16f1d7712920) | 2026-08-19 | [ADVAPP-2794]: Refactor the set password and one-time-login flow to match Apidapter (#276… | stat-only: feature redesign of OTP login flow (controllers replaced by Filament auth pages + actions) |
| [`b13ffd501b`](https://github.com/canyongbs/advisingapp/commit/b13ffd501ba4552e6257168a973061f149886fb0) | 2026-08-20 | [ADVAPP-2889]: Refactor the way automated chat naming works (#2767) | stat-only: feature (auto-naming job, named_by_user_at column, feature flag) |
| [`aa76a25892`](https://github.com/canyongbs/advisingapp/commit/aa76a2589261a095ff9bbe5a0da700cea8f2fd6a) | 2026-10-07 | [ADVAPP-2886]: Improve infrastructure design to support high scale operations (#2898) | stat-only: infrastructure feature (SQS fair queue, queue monitoring, SES via SQS) |

</details>

#### Bagisto: 29 qualifying of 40 diffs read

| # | Commit | Date | What was fixed | Class | Tag | Kind | Evidence (abridged) |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | [`f236da61ab`](https://github.com/bagisto/bagisto/commit/f236da61ab7d483a3670d89ca91b0f7bb79862a2) | 2018-07-31 | controllers switched from models to repositories | C1 | role | dependency | UserController `use Webkul\User\Models\Admin; use Webkul\User\Models\Role;` with static `Admin::create/findOrFail`, `Role::all()`; replaced by injected AdminRepository/RoleRepository (aliased as Admin/Role). Country/Loc… |
| 2 | [`21a16b8cf0`](https://github.com/bagisto/bagisto/commit/21a16b8cf0a5f4ab79859d50e42636d937ddd992) | 2018-08-16 | datagrids moved out of the view-composer role into their own DataGrids classes | C1 | role | placement | datagrid definitions were view composers bound via View::composer('admin::settings.locales.index', 'Webkul\Admin\Http\ViewComposers\DataGrids\LocalesComposer') in ComposerServiceProvider; renamed/moved to Webkul\Admin\D… |
| 3 | [`abd3bba617`](https://github.com/bagisto/bagisto/commit/abd3bba6177656819a0f4cf21e8fb4853b5fe741) | 2018-12-27 | catalog before/after event firing moved from controllers into repositories | C3 | role | body | controllers fired `Event::fire('catalog.product.create.before')` etc. around repo calls and looped `$product->delete()` in massDestroy; moved into repository create/update/delete. CategoryController drops the Event impo… |
| 4 | [`2a5e8b1ddd`](https://github.com/bagisto/bagisto/commit/2a5e8b1dddb31c47d26ce32a5c80f846444b4e25) | 2019-06-27 | vendor intl helper library removed app-wide in favour of an app helper | C2 | location | dependency | removed vendor global helpers `country()->name(...)` (3 API resources, views) and `currency(...)` (Core.php x4) from propaganistas/laravel-intl; replaced by core()->country_name() and \NumberFormatter; package dropped f… |
| 5 | [`8030b097e7`](https://github.com/bagisto/bagisto/commit/8030b097e714a716eb132d24eeb09f62067c6a18) | 2020-04-23 | commands moved from the host app into the owning packages | C1 | location | placement | package-specific artisan commands sat in the root App\Console\Commands; moved into Webkul\Core and Webkul\Product and registered by their service providers; app Console Kernel deleted |
| 6 | [`42e3a94b98`](https://github.com/bagisto/bagisto/commit/42e3a94b988243ee916e0432994d525bbde355ce) | 2020-06-11 | raw SQL moved from a listener into a repository method | C2 | role | dependency | removed the only `\DB::update("update product_flat f join product_attribute_values ...")` from the listener (~L141 parent); listener now calls productFlatRepository->updateAttributeColumn($attribute). Listener keeps Sch… |
| 7 | [`5db4797158`](https://github.com/bagisto/bagisto/commit/5db479715849799a6700129cfb587961cec859ae) | 2020-08-03 | product copy persistence moved from controller to repository; controller stops naming models | C1 | role | dependency | removed `use Webkul\Attribute\Models\Attribute;` + `use Webkul\Product\Models\ProductFlat;`, `Attribute::query()->where([...])` x4, `new ProductFlat()`, ->replicate()/->save() loops (~70 lines); controller now `$this->p… |
| 8 | [`51a7b9ddc2`](https://github.com/bagisto/bagisto/commit/51a7b9ddc2e2f2e8d1e13e78e15680e89222ee9b) | 2021-03-09 | controller writes through the repository instead of the DB facade | C2 | role | dependency | removed the only `\DB::table('velocity_meta_data')->insert([...])` (L243 parent) from the controller; now `$this->velocityMetaDataRepository->create([...])`. Msg: raw table access conflicted with concord/SaaS model over… |
| 9 | [`8f67109880`](https://github.com/bagisto/bagisto/commit/8f671098805810501195711eac5583f5009e55d5) | 2022-01-10 | before/after persistence event dispatch moved from controllers into repositories | C3 | role | body | controllers wrapped repository create/update with `Event::dispatch('customer.registration.before/after')`, `customer.update.*`, `core.configuration.save.*`; moved into repository overrides. Controllers drop `use Illumin… |
| 10 | [`2ca4f7087e`](https://github.com/bagisto/bagisto/commit/2ca4f7087e6662952ceb3f3cd0f38dccaad474f2) | 2022-10-07 | per-type product copy logic moved from the repository into the product type classes | C3 | role | body | the copy implementation (replicate attribute values, flat rows, images via Storage::copy) moved from ProductRepository into AbstractType::copy() with per-type hooks; repo drops Storage, Str, ProductFlat and cross-packag… |
| 11 | [`f5d9365dc6`](https://github.com/bagisto/bagisto/commit/f5d9365dc67550d5da4cea1a83a1e1a8e286c622) | 2023-05-18 | storefront HTTP layer moved out of the Customer domain package into Shop | C1 | location | placement | domain Customer package held storefront controllers; RegistrationController imported `Webkul\Shop\Mail\SubscriptionEmail` and used shop:: translations (Customer -> Shop). Controllers, requests, mails, middleware moved t… |
| 12 | [`7206988b0e`](https://github.com/bagisto/bagisto/commit/7206988b0e079507cb5665cb985158a42eb1a7f7) | 2023-05-23 | admin controllers moved out of every domain package into the Admin package | C1 | role-across-locations | placement | 29 admin controllers lived in 13 domain packages, each with its own base Controller; 15 of them (Attribute, Category, Core, Product, Tax, User) referenced Webkul\Admin\... classes (domain -> Admin cycle). Moved into Adm… |
| 13 | [`ea67b95f6d`](https://github.com/bagisto/bagisto/commit/ea67b95f6ddc1f684a37170ab5583255ea760337) | 2023-06-13 | dashboard statistics queries moved from controller into a service and repositories | C2 | role | dependency | removed `use Illuminate\Support\Facades\DB;` + `use Carbon\Carbon;` and ~280 lines of query building (`$this->orderItemRepository->getModel()->leftJoin(...)->addSelect(DB::raw(...))`); 5 repository injections replaced b… |
| 14 | [`af7ac1a1cf`](https://github.com/bagisto/bagisto/commit/af7ac1a1cfb52f8056ce09ce4349e0d015e7d355) | 2023-09-19 | admin form requests moved out of domain packages into the Admin package | C1 | role-across-locations | placement | 9 FormRequests lived in domain packages (e.g. Webkul\Category\Http\Requests\CategoryRequest, Webkul\Core\Http\Requests\MassDestroyRequest) and were used only by Admin controllers (11 controllers imported them); moved in… |
| 15 | [`0a99fe4311`](https://github.com/bagisto/bagisto/commit/0a99fe4311afc9dbd729cb553f387f2e244e167c) | 2023-09-19 | storefront form request moved from Checkout into Shop | C1 | role-across-locations | placement | Shop API OnepageController imported `Webkul\Checkout\Http\Requests\CustomerAddressForm`; request moved from domain Checkout package into Shop |
| 16 | [`98d5b761c5`](https://github.com/bagisto/bagisto/commit/98d5b761c54ce1d1a7f9c3488e507491d281892a) | 2023-09-21 | indexing work moved out of listeners into queued jobs | C1 | role | dependency | Order/Refund listeners dropped ctor-injected `Webkul\Product\Helpers\Indexers\Inventory` and now dispatch UpdateCreateInventoryIndex job; Product listener dropped Inventory/Price/ElasticSearch indexer refs for Bus::chai… |
| 17 | [`32986ba8b0`](https://github.com/bagisto/bagisto/commit/32986ba8b0a822fa598c844560fe09e87fcbd55d) | 2023-10-20 | install command relocated from Core to the Installer package (domain placement, not visible via references) | C1 | location | placement | namespace-only move (99% similar); class imported only Illuminate Command/DB/File, so nothing but its namespace showed the misplacement. CoreServiceProvider now registers \Webkul\Installer\Console\Commands\Install |
| 18 | [`62b8d7267f`](https://github.com/bagisto/bagisto/commit/62b8d7267f26c2e058d3ee484ab5ead87502e4bd) | 2023-11-08 | theme customization model+repository moved from storefront package to Theme package | C1 | location | dependency | Admin ThemeController and FPC listener imported `Webkul\Shop\Repositories\ThemeCustomizationRepository` (Admin->Shop, FPC->Shop); model/contract/proxy/repository moved to Theme package, imports become Webkul\Theme\... |
| 19 | [`04fe8a4cae`](https://github.com/bagisto/bagisto/commit/04fe8a4caed8fe10756dbed9f34b6614371342e8) | 2024-01-10 | Admin controller no longer calls a Shop listener directly; fires the event instead | C1 | location | dependency | removed `use Webkul\Shop\Listeners\Invoice as ShopInvoiceListener;` + ctor injection + `$this->shopInvoiceListener->afterCreated($invoice)`; replaced by `Event::dispatch('sales.invoice.save.after', $invoice)` (adds Even… |
| 20 | [`f6180869fb`](https://github.com/bagisto/bagisto/commit/f6180869fb252d1e93ecab221df618655707a8f6) | 2024-01-23 | payment-method-specific availability rule moved from the payment manager into the COD method class | C3 | role | body | generic Payment::getPaymentMethods() special-cased `getCode() == 'cashondelivery'` with Cart::getCart()->haveStockableItems(); moved into CashOnDelivery::isAvailable(). `use Webkul\Checkout\Facades\Cart` stays (dead) in… |
| 21 | [`789da165e2`](https://github.com/bagisto/bagisto/commit/789da165e25080f7b272fa6e1a4834f416a0a92c) | 2024-05-20 | saved-filter controller moved out of the DataGrid package into Admin | C1 | location | placement | FilterController lived in the DataGrid package but `extends Webkul\Admin\Http\Controllers\Controller` and used auth()->guard('admin'); deleted and its store/get/destroy moved into Admin DataGridController; DataGrid prov… |
| 22 | [`6619109cb8`](https://github.com/bagisto/bagisto/commit/6619109cb8fdc89457fbfb92959d74458afbe364) | 2024-06-06 | view stops service-locating a repository; controller supplies the data | C1 | role | dependency | views resolved `app('Webkul\Core\Repositories\LocaleRepository')->all()` in @php; removed; controller now injects LocaleRepository and passes $allLocales. At HEAD 16/563 Blade views still app()-resolve a Webkul reposito… |
| 23 | [`2316f8d9ff`](https://github.com/bagisto/bagisto/commit/2316f8d9ff106acae1044907f4c86fb65c3add83) | 2024-07-08 | export class moved from Admin to DataGrid so the DataGrid package no longer depends on Admin | C1 | location | dependency | base DataGrid package imported `use Webkul\Admin\Exports\DataGridExport;` (DataGrid -> Admin); class moved into DataGrid package, import becomes Webkul\DataGrid\Exports\DataGridExport |
| 24 | [`1a29eb2a24`](https://github.com/bagisto/bagisto/commit/1a29eb2a24e1c19c55cea72371f569adc94c4eb5) | 2025-05-08 | logic moved from a Blade view into the Core helper | C3 | role | body | Blade layout had a 45-line @php block building speculation-rules arrays from core()->getConfigData(); moved to Core::getSpeculationRules(). View still calls core() after (only the computation moved) |
| 25 | [`8b6b036d1b`](https://github.com/bagisto/bagisto/commit/8b6b036d1b103eaf772a1567fbb0bbba83caace5) | 2026-02-12 | install command stops writing the admins table via DB facade and delegates to the installer DatabaseManager | C2 | role | dependency | removed `use Illuminate\Support\Facades\DB;` + `DB::table('admins')->updateOrInsert(...)` (~L424 parent) from the console command; now `$this->databaseManager->createAdminUser([...])`. Command still uses Event/File faca… |
| 26 | [`cf506db98b`](https://github.com/bagisto/bagisto/commit/cf506db98bd4b541e86359cc03bd54c4030b41d1) | 2026-03-02 | prompt construction and AI config selection moved from controllers into the MagicAI service | C3 | role | body | controllers still use the MagicAI facade and core() after; moved prompt building (OnepageController::getCheckoutPrompt ~30 lines, review-translation prompt) and provider/model config selection (setProvider/setModel/setP… |
| 27 | [`81d1b5a58c`](https://github.com/bagisto/bagisto/commit/81d1b5a58c38a35365cb0b8d7b5830abeba8c097) | 2026-03-24 | search decoupled from Elasticsearch: repository depends on a search-engine contract/manager instead of the ES class | C1 | role | dependency | ProductRepository ctor-injected concrete `ElasticSearchRepository` and branched on `$this->searchEngine == 'elastic'`; now injects SearchEngineManager resolving Contracts\SearchEngine. ElasticSearchRepository (did not e… |
| 28 | [`cce091fc93`](https://github.com/bagisto/bagisto/commit/cce091fc93c50c825c6c9e7123b5481c06d43fd2) | 2026-10-14 | Customer repository reads Sales orders via the proxy instead of the concrete model | C1 | role-across-locations | dependency | removed `use Webkul\Sales\Models\Order;` and 2 `Order::query()` calls in syncNewRegisteredCustomerInformation() from the Customer repository; now `Webkul\Sales\Models\OrderProxy::query()`. Msg: "the customer model alrea… |
| 29 | [`54dbbd514b`](https://github.com/bagisto/bagisto/commit/54dbbd514b0eda718e8f6e0ecfc69fe23a5aa759) | 2026-10-15 | Core stops naming concrete models of Sales/Customer; uses contracts and concord proxies instead | C1 | role-across-locations | dependency | Core package classes imported other packages' concrete models: `use Webkul\Sales\Models\Invoice` (command), `use Webkul\Customer\Models\CustomerGroup` (Core.php), `use Webkul\Customer\Models\Customer` (Address model bel… |

<details><summary>Bagisto: 11 diffs read that did not qualify</summary>

| Commit | Date | Subject | Why not |
| --- | --- | --- | --- |
| [`bc087010ee`](https://github.com/bagisto/bagisto/commit/bc087010ee6198a45ae39a8f83998efcb75f3c54) | 2018-08-22 | Repository pattern changes | build-vs-buy: in-house repository base replaced by prettus/l5-repository; adds a vendor dependency rather than fixing a misplaced one |
| [`adc7052afa`](https://github.com/bagisto/bagisto/commit/adc7052afa1336b81e4c34488e8f6b21b5e1a113) | 2018-10-04 | Cart Dependency Resolved From Composer to Global helpers and removed the form field check… | early WIP feature/bug work; replaces a CartComposer view composer with the cart() global helper (toward globals, not fixing a mistake) |
| [`97dcf89740`](https://github.com/bagisto/bagisto/commit/97dcf89740a9044aefb5e34be2ad1bf1ed7e6c92) | 2019-12-30 | refactored model reference | style: string class name -> ::class import (same dependency) |
| [`9386475dec`](https://github.com/bagisto/bagisto/commit/9386475dec8bd866086a2c99b6873f3b6966889b) | 2022-07-26 | Removed auth guard middleware dependency from constructor. | bug fix: auth()->guard()->user() read in constructor before middleware; moved into each action (auth() still used) |
| [`0d073a2ef0`](https://github.com/bagisto/bagisto/commit/0d073a2ef014d4ec9079cd650a6a21dff7e4a7e3) | 2022-08-10 | Refactored repositories model method | style/types: model() gets `: string` return type and string class names |
| [`129b1c5b59`](https://github.com/bagisto/bagisto/commit/129b1c5b5909cf85b0143ed9bbde3ef0bcdadb91) | 2023-10-10 | Refactor Dashboard code | mixed reporting feature (new reporting controllers/routes) + rename Services\DashboardService -> Helpers\Reporting; Core::getTimeInterval removal incidental |
| [`2eaf748494`](https://github.com/bagisto/bagisto/commit/2eaf748494561142f3703d8b44eb03caf0c49caa) | 2024-05-29 | remove shop dependency | feature removal: storefront saved-filter endpoints dropped |
| [`8d8454ded4`](https://github.com/bagisto/bagisto/commit/8d8454ded4070856259e8f29131aa1cc5af5c29b) | 2026-04-20 | fix: system file moved | config array + translations moved from Omnibus to Admin system.php; no class change |
| [`6cb89050a7`](https://github.com/bagisto/bagisto/commit/6cb89050a71d741f4b90b4b0a9513b6982f3e64e) | 2026-09-29 | feat: added reverb, moved breadcrumbs to shop for consistency and refined configuration | feature (Reverb broadcasting); the move is a routes/breadcrumbs.php definition file, not a class |
| [`611419023e`](https://github.com/bagisto/bagisto/commit/611419023e4c7086fd8ec1b2be5d125c02a89152) | 2026-10-14 | refactor: drop the dead cart facade import from the rule validator | unused-import cleanup; the Checkout Cart facade was never referenced |
| [`61968f0ad4`](https://github.com/bagisto/bagisto/commit/61968f0ad4ee44e020a81de275198fc9635b16ea) | 2026-10-14 | refactor: move order item ownership onto the order model | dedupe: both controllers already delegated to $order->ownsItems(); only thin wrapper methods removed |

</details>

#### Firefly III: 21 qualifying of 38 diffs read

| # | Commit | Date | What was fixed | Class | Tag | Kind | Evidence (abridged) |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | [`9028ad36ad`](https://github.com/firefly-iii/firefly-iii/commit/9028ad36ad19400b560b30f99fb0d3dd266db31a) | 2015-06-27 | Vendor chart-library usage moved out of chart controllers behind app chart-generator interfaces. | C2 | role | dependency | chart controllers removed `use Grumpydictator\Gchart\GChart` and method-injected `GChart $chart` plus addColumn/addRow/generate calls; now `App::make('FireflyIII\Generator\Chart\Bill\BillChartGenerator')` bound in Firef… |
| 2 | [`f9e2a677d9`](https://github.com/firefly-iii/firefly-iii/commit/f9e2a677d9476a37d56edffdbe8bda9fcaf2e4f3) | 2016-02-17 | Eager-loading rule-group query and primary-trigger query moved from the controller into repositories. | C2 | role | dependency | controller removed `use Illuminate\Database\Eloquent\Relations\HasMany` with the `Auth::user()->ruleGroups()->orderBy()->with([HasMany closures])->get()` query and `$rule->ruleTriggers()->where(...)->first()`; now getRu… |
| 3 | [`50e5c21735`](https://github.com/firefly-iii/firefly-iii/commit/50e5c2173543fcbe97da963e5e1dd1eb731d0c0f) | 2016-02-18 | Shared journal-collector query class relocated from the Export feature module to the repository layer for reuse. | C1 | location | placement | general-purpose journal query class (user + accounts + date range) moved out of the Export feature folder into Repositories\Journal; only caller Export\Processor updated (container string id). Contributor commit |
| 4 | [`8008311d9c`](https://github.com/firefly-iii/firefly-iii/commit/8008311d9c93a21f910ab2136a54566e933e338b) | 2016-03-03 | Repository stops reading the logged-in user from the Auth facade; the user is injected via the constructor. | C2 | role | dependency | removed `use Auth;` and ~12 `Auth::user()->accounts()...` calls from AccountRepository; User now constructor-injected (`__construct(User $user)`), Auth::check()/Auth::user() moved into the AccountServiceProvider binding… |
| 5 | [`cc88d5962e`](https://github.com/firefly-iii/firefly-iii/commit/cc88d5962ecddf893c8454af248bb3688d73f02a) | 2016-03-28 | Registration side effects (welcome mail, owner role) moved from the auth controller into event handlers. | C3 | role | body | register() lost `Mail::send(...)`+Swift_TransportException catch and `$repository->attachRole(...,'owner')`; now `event(new UserRegistration($user, ip))`. Controller still imports Mail, Message, Swift_TransportException… |
| 6 | [`9a30fbd05a`](https://github.com/firefly-iii/firefly-iii/commit/9a30fbd05a34ec11ae3d3f721eacf720a470569b) | 2016-10-23 | Input-to-data mapping moved out of controllers into form request classes. | C3 | role | body | controllers' `$request->get(...)` array-mapping blocks moved to FormRequest getPiggyBankData()/getRuleData() etc.; DomainController `User::get()` -> UserRepositoryInterface::all() but still imports FireflyIII\User; cont… |
| 7 | [`194fe178c0`](https://github.com/firefly-iii/firefly-iii/commit/194fe178c0299771a52b8f6804af3d7cd6df29d9) | 2017-08-04 | External password-breach API client relocated from Support to Services. | C1 | location | placement | HIBP client (uses vendor `Requests::get`) and its interface moved from Support to Services, next to Services/Currency external-API clients; FireflyValidator also dropped stale `use Requests; use Requests_Exception; use… |
| 8 | [`7bbfb692de`](https://github.com/firefly-iii/firefly-iii/commit/7bbfb692de78cf5bef3d3899a90bf73a6121f5a3) | 2018-02-17 | Controllers stop computing a piggy bank's saved amount through model relations and call a repository method instead. | C3 | role | body | `$piggyBank->currentRelevantRep()->currentamount` replaced with `$repository->getCurrentAmount($piggyBank)`; controllers still route-bind PiggyBank and already used PiggyBankRepositoryInterface |
| 9 | [`b627d42160`](https://github.com/firefly-iii/firefly-iii/commit/b627d42160a5e5678318a06cf12d4811a5724951) | 2018-02-22 | Persistence logic moved out of repositories into a factory and internal update/destroy services. | C3 | role | body | BillRepository store/update/destroy no longer call `Bill::create()`/`$bill->delete()`/note writes; they delegate to `app(BillFactory/BillDestroyService/BillUpdateService)`; repo dropped `use FireflyIII\Models\Note` but… |
| 10 | [`1b304bf85e`](https://github.com/firefly-iii/firefly-iii/commit/1b304bf85edc3f6a42f40367c83a4a693935e84b) | 2018-02-25 | Query/persistence helpers moved off the TransactionJournal model into the journal repository. | C3 | role | body | removed model methods amount()/amountPositive()/budgetId()/categoryAsString()/dateAsString() (relation queries, CacheProperties, `$this->save()`) from TransactionJournalTrait and Tag::tagSum(); callers now use JournalRe… |
| 11 | [`1645490f5c`](https://github.com/firefly-iii/firefly-iii/commit/1645490f5c3de08335ebe4d4afab75eb8e9f215f) | 2018-03-19 | Callers stop using model helper methods and go through the account repository. | C3 | role | body | `$account->getMeta('x')` and `$account->notes()->first()` replaced by `$this->repository->getMetaValue()/getNote()`; transformer dropped `use FireflyIII\Models\Note` but still type-hints Account/AccountType; controller… |
| 12 | [`3c9b7c07af`](https://github.com/firefly-iii/firefly-iii/commit/3c9b7c07aff5236fc04589f276a36e251d3226bd) | 2018-03-24 | Import storage stops building journals by hand and uses the journal factory. | C3 | role | body | ImportSupport lost storeJournal()/createTransaction()/storeBudget()/storeCategory()/storeTags() (`new TransactionJournal`, `new Transaction`, `->save()`) and imports of Budget/Category/Transaction/TagRepositoryInterface… |
| 13 | [`835a421909`](https://github.com/firefly-iii/firefly-iii/commit/835a421909fc3069542dcc41f47e77884768ae0d) | 2018-08-24 | Static factory calls on the rule processor replaced with container-resolved instances for mockability. | C3 (lead audit) | location | form | callers' static `Processor::make($rule)` replaced by `app(Processor::class)` + `->make($rule)`; Processor made non-final with public ctor. Callers still name Processor (via app(X::class)); only the static-call reference… |
| 14 | [`1284f9cecc`](https://github.com/firefly-iii/firefly-iii/commit/1284f9cecc3b5205c198f1d5a1fb3a583be3e5ff) | 2018-12-15 | API controller stops instantiating transformers with new and resolves them from the container so they can be mocked. | C3 (lead audit) | role | form | controller `new AccountTransformer($params)`/`new PiggyBankTransformer`/`new TransactionTransformer` -> `app(XTransformer::class)` + setParameters(); transformer dropped static `TransactionCurrency::find()` for reposito… |
| 15 | [`e4a9abc315`](https://github.com/firefly-iii/firefly-iii/commit/e4a9abc315ce862ceb84b65a34a1159a7265d639) | 2019-06-07 | Rule-application orchestration moved from a console command and a job into a new RuleEngine class. | C3 | role | body | rule-group/rule stop-processing loop around `app(Processor::class)->make()->handleJournalArray()` moved into new RuleEngine::processJournalArray(); command and job still import Processor (stale), models and repositories… |
| 16 | [`16a511cf79`](https://github.com/firefly-iii/firefly-iii/commit/16a511cf79920dcf5a6680d5cf9342a9c980285d) | 2020-06-16 | Localised UI text moved out of the API transformer into the web controller. | C2 | role | dependency | removed `'next_expected_match_diff' => ... trans('firefly.today') : ...->diffForHumans(...)` (the transformer's only trans() call) and moved it into web BillController::index; transformer keeps Carbon. HEAD: 0/26 transf… |
| 17 | [`bee219ebf7`](https://github.com/firefly-iii/firefly-iii/commit/bee219ebf7dcf9fc7d2bf085f7069e7efa299182) | 2024-03-07 | Static singleton factory replaced by a container-bound singleton. | C1 | location | dependency | removed `use ...\TransactionRules\Factory\ExpressionLanguageFactory` + static `ExpressionLanguageFactory::get()` (static-singleton class deleted) from 3 callers; ExpressionLanguage now a container singleton fetched with… |
| 18 | [`d4e14dd262`](https://github.com/firefly-iii/firefly-iii/commit/d4e14dd262763d8c48e19a767d1be7faa03259b8) | 2025-08-03 | Balance computation and currency conversion moved out of the API transformer into the enrichment step. | C1 | role | dependency | transformer removed `use ...\Support\Http\Api\ExchangeRateConverter` (`new ExchangeRateConverter()`), `use ...\Support\Facades\Steam` (`Steam::finalAccountBalance/bcround`) and Log facade; balance + primary-currency con… |
| 19 | [`0b33e1ff09`](https://github.com/firefly-iii/firefly-iii/commit/0b33e1ff093bd4b2045558ec17b9b62195682c42) | 2026-02-03 | Primary-currency recalculation (raw DB queries) moved from a console command and event handler into a dedicated service. | C2 | role | dependency | command lost ~175 lines incl. all `DB::table('transactions')`/`DB::raw(...)` calls, `new ExchangeRateConverter()`, `app(PiggyBankRepositoryInterface)`; now `new PrimaryAmountRecalculationService()->recalculate()`. Comma… |
| 20 | [`c51df8cd83`](https://github.com/firefly-iii/firefly-iii/commit/c51df8cd83575242a9f1b2d996f0f0829362600c) | 2026-02-04 | Domain-event dispatch moved from controllers and a job into the repository and clone service. | C1 | role | dependency | removed `new TransactionGroupEventFlags()` + `event(new CreatedSingleTransactionGroup(...))` (+ Preferences::mark in API) from 2 controllers and a job; now fired in repository store() and GroupCloneService; job `$recurr… |
| 21 | [`1893a33d84`](https://github.com/firefly-iii/firefly-iii/commit/1893a33d84d606bbf7a4b73845348faa2e5220ca) | 2026-03-20 | Listener stops resetting amounts with raw DB writes and only delegates to the recalculation service. | C2 | role | dependency | listener removed `use Illuminate\Support\Facades\DB`, `Illuminate\Database\Query\Builder`, Budget/PiggyBank/UserGroup models and 2 repository interfaces; `DB::table(...)->update([col => null])` resets and `->saveQuietly… |

<details><summary>Firefly III: 17 diffs read that did not qualify</summary>

| Commit | Date | Subject | Why not |
| --- | --- | --- | --- |
| [`9a3aed8038`](https://github.com/firefly-iii/firefly-iii/commit/9a3aed80389fc5435156d97485893c319cf50a87) | 2015-01-02 | Moved code to relate transfers to another class. Still needs some work. | actions split out of TransactionController into a new RelatedController (same role) plus a search helper |
| [`c00bcd78cc`](https://github.com/firefly-iii/firefly-iii/commit/c00bcd78ccddc2df84fc051250b456dc25ab58d2) | 2015-07-07 | Some dependency clean up. | App::make()/App::abort() facade calls replaced with app()/abort() helpers: same dependency, syntax only |
| [`745e0aa525`](https://github.com/firefly-iii/firefly-iii/commit/745e0aa52560c1b6786f93e2d41c48f62328028a) | 2016-03-20 | Move to object held user object. | Lead audit: same change as 8008311d9c landing on a parallel branch; counted once, as Koel's duplicate f4ca7cf09f was. |
| [`8ef7c5ac33`](https://github.com/firefly-iii/firefly-iii/commit/8ef7c5ac33d2f36cb15e82ad94d37b0e06e91c18) | 2016-10-10 | Moved find() method to new class. | method moved between sibling data-access classes (AccountCrud -> AccountRepository) while undoing the Crud split; same role |
| [`8e542531b3`](https://github.com/firefly-iii/firefly-iii/commit/8e542531b35459f2560499039e179aeb8d998f05) | 2016-11-05 | Move collecting journals to the collector. | BillRepository::getJournals replaced by the shared JournalCollector (DRY within the data-access layer); controllers now `new JournalCollector(auth()->user())` |
| [`53a6c10ada`](https://github.com/firefly-iii/firefly-iii/commit/53a6c10ada71646008d9df3056093217ee2bfc41) | 2018-02-09 | Remove reference to guard from models. | reverse direction: injected $guard parameter in model routeBinder() replaced by the auth() helper |
| [`bc4e06568d`](https://github.com/firefly-iii/firefly-iii/commit/bc4e06568d0b2344d514b15ca2c2af3eb84a4e0e) | 2018-04-29 | Move references to repository | method-injected repositories turned into controller properties (same dependencies); currency select list feature removed |
| [`89834baf01`](https://github.com/firefly-iii/firefly-iii/commit/89834baf01ff367fd07419eb2c4ad6f7ed59fb43) | 2018-07-14 | Refactor references to static facades. Improve budget controller code. | swaps Preferences/View facade aliases for app('preferences')/app('view') locator calls: same dependency, different syntax; day calc moved into a controller tra… |
| [`16fba15b5c`](https://github.com/firefly-iii/firefly-iii/commit/16fba15b5ce9dae8802552039f485d8a635c54e7) | 2018-08-11 | Refactor various methods away from controllers | controller methods moved into Support\Http\Controllers traits (PeriodOverview, AugumentData) that the same controllers use: same role |
| [`717b9d21fd`](https://github.com/firefly-iii/firefly-iii/commit/717b9d21fd36307283553619048498e47329e0bd) | 2019-08-30 | Move method to correct repository. | methods moved between budget repositories (splitting a god repository; same role). One of 13 same-subject commits |
| [`2f14753405`](https://github.com/firefly-iii/firefly-iii/commit/2f1475340548d4bf424f15d1c5e0bf2e30a9f61f) | 2021-02-16 | Refactor API code. | API controllers split into per-action controllers (same role) |
| [`80237d8bc3`](https://github.com/firefly-iii/firefly-iii/commit/80237d8bc3571d6a26814c3846ccfb8a6344798c) | 2023-10-22 | Refactor methods that request the old currency preference. | data-model change: default currency no longer stored as a preference |
| [`1d138eed8d`](https://github.com/firefly-iii/firefly-iii/commit/1d138eed8d91bf4024d40e28de89268a23386d04) | 2023-10-28 | Refactor currency repository. | feature refactor: user-group currency repository introduced, dead CurrencyController removed |
| [`f41397eb43`](https://github.com/firefly-iii/firefly-iii/commit/f41397eb4365a2fa320bf5105fcd5f5fb6ce0949) | 2024-03-07 | refactor: add method on RuleAction to compute action value | convenience refactor moving expression evaluation INTO the RuleAction model (adds a model -> TransactionRules dependency); not fixing a misplacement |
| [`8eb0313841`](https://github.com/firefly-iii/firefly-iii/commit/8eb0313841c373435c5ed7336ed74d58f5c41bfa) | 2025-03-08 | Remove references to user group specific repositories and instead use the normal ones. | merges the parallel Repositories\UserGroups\* set into the normal repositories (DRY within the repository layer); callers just swap imports |
| [`0431c95b49`](https://github.com/firefly-iii/firefly-iii/commit/0431c95b49e0b2c6ce94e13c2d34f51394c14cbb) | 2026-01-18 | Move rule actions for rule events. #11544 | handler class re-housed as a queued listener (Handlers/Events -> Listeners convention migration; same role) |
| [`0296110533`](https://github.com/firefly-iii/firefly-iii/commit/02961105338c6a7308aab7bef1da7b2ef480b685) | 2026-09-19 | Move a selection of code to group logic. | feature: user-group scoped account lookups (new *ForGroup methods) |

</details>


## Evidence 3: commits co-authored by AI agents

- **Commits carrying an agent trailer** (a `Co-Authored-By` line or a "Generated with" line naming an agent) or written by an agent account:

  | App | Commits | Agents and years |
  | --- | --- | --- |
  | Coolify | 662 | Claude, 2025 to 2026 |
  | Koel | 16 | mostly Claude, 2025 to 2026 |
  | Advising App | 14 | Copilot, 2026 |
  | Bagisto | 3 | 2026 |
  | Firefly III | 2 | 2026 |
  | BookStack | 0 | |

- **Was AI-written code later restructured?** The check was whether non-test backend PHP files added by those commits were later deleted or moved.
  - **Koel:**
    - [`8a4b9347e6`](https://github.com/koel/koel/commit/8a4b9347e6c7b8a759ef751e22dd774b732b2c76) (2026-03-08) is a squash of eight sub-commits, each with a `Co-Authored-By: Claude` trailer. It added `app/Helpers/Network.php`, which has static methods, and three `app/Services/RadioStream*.php` classes.
    - [`65765e8357`](https://github.com/koel/koel/commit/65765e83572dd18d949db3fe55f6cc186231db90) (2026-05-26, no trailer), "make Network helper injectable so tests don't need internet", changed every caller from static calls to an injected instance. This is C3 (form).
    - [`5f6ce2cefd`](https://github.com/koel/koel/commit/5f6ce2cefd08f437a269236b677ad971517ccbb6) (2026-06-04, an SSRF fix) moved the file to `app/Services/Network/Network.php`.
    - The three radio classes were moved in a bulk regrouping of `App\Services` into feature subfolders, [`899ad5d4ab`](https://github.com/koel/koel/commit/899ad5d4ab72b8c85538ed69efa5166a9cc3ca95). That regrouping does not qualify, because it stays within the same role folder.
  - **Coolify:** AI-trailer commits added 80 backend PHP files. 8 were later deleted and none were moved; the deletions were feature removals or rewrites.
  - **Advising App:** AI-trailer commits added 12 files. 2 were deleted, both feature-flag clean-ups.
  - **BookStack, Bagisto, Firefly III:** AI-trailer commits added no backend PHP files.
- **AI-co-authored commits among the qualifying fixes.** 5 of Coolify's 20 qualifying fixes are themselves Claude co-authored:

  | Commit | Class |
  | --- | --- |
  | [`0540b2eae5`](https://github.com/coollabsio/coolify/commit/0540b2eae567b344661f26f4c154be77b1efc20f) | C3 |
  | [`70fb4c6869`](https://github.com/coollabsio/coolify/commit/70fb4c6869ed9e72ed62b9e409f29819d42632b9) | C3 |
  | [`cbee84353e`](https://github.com/coollabsio/coolify/commit/cbee84353e2357f320dc529d158ba7823517d85e) | C2 |
  | [`d27070b215`](https://github.com/coollabsio/coolify/commit/d27070b215fa96edf0fcae2650e6a2b6dd9e11a2) | C2 |
  | [`3e0d48faea`](https://github.com/coollabsio/coolify/commit/3e0d48faeaab950bfd063dfca908f1d140316ede) | C1, the reverse-direction inlining |

  No other app's qualifying fixes carry a trailer.
- **Limit:** this check follows whole files only. Logic moved out of AI-written methods into other files, without the file itself moving, is not detected.

## Caveats

- **Sample size and selection.**
  - The commits are a keyword-selected sample: 252 diffs read out of 2,078 keyword matches that touch backend PHP. They are not a census of every architectural fix.
  - Commits whose subject says nothing structural, such as most "fix" and "feat" commits, were mostly not reached. Coolify [`d27070b215`](https://github.com/coollabsio/coolify/commit/d27070b215fa96edf0fcae2650e6a2b6dd9e11a2), labelled `fix`, is one of the few included.
- **Who did the reading.**
  - Six agents read the diffs, one per app, using one written rubric. The lead read every row, checked the featured diffs by hand, and changed 4 rows.
  - Judgement calls remain on the C2/C3 boundary. The miners flagged these:
    - Bagisto [`ea67b95f6d`](https://github.com/bagisto/bagisto/commit/ea67b95f6ddc1f684a37170ab5583255ea760337) is C2 because the `DB` facade left a controller, and only 4/131 controllers use `DB` at HEAD.
    - Firefly III [`f9e2a677d9`](https://github.com/firefly-iii/firefly-iii/commit/f9e2a677d9476a37d56edffdbe8bda9fcaf2e4f3) is C2 only through a `HasMany` import that 0/284 controllers have at HEAD.
- **Realism is judged at HEAD.**
  - It is judged at each app's pinned HEAD, not at the time of the fix. A sweep that removed a reference everywhere makes the reference look rare at HEAD; BookStack [`c157dc3490`](https://github.com/BookStackApp/BookStack/commit/c157dc3490c39aa236e22714019a649ebeb11f13) is one example.
  - Those cases are in bucket P, not A.
- **Rule counts depend on how a suite is written.**
  - Vito writes one `arch()` per source namespace, which yields more rows than one assertion over a list would.
  - Pinkary chains several assertions per `arch()`, which yields more C4 rows.
  - The C1+C2 share of architecture rules is 97% in total, and still 95% or more within each app's own suite.
- **Blade views and global helper files.**
  - A few from-sides are not classes: Bagisto [`1a29eb2a24`](https://github.com/bagisto/bagisto/commit/1a29eb2a24e1c19c55cea72371f569adc94c4eb5) (C3) and [`6619109cb8`](https://github.com/bagisto/bagisto/commit/6619109cb8fdc89457fbfb92959d74458afbe364) (C1) start in Blade views. Coolify [`f9375f91ec`](https://github.com/coollabsio/coolify/commit/f9375f91ec1b4196211b8281da881cba209ffaf1), [`e74da06465`](https://github.com/coollabsio/coolify/commit/e74da06465460d5f3446fe24e33156b8826655cf) and [`f8f27fff13`](https://github.com/coollabsio/coolify/commit/f8f27fff138fea0b03121b14fa002459328a620f) move logic into or out of `bootstrap/helpers/*.php`.
  - A plan that lists only classes would not contain these from-sides at all.
