# How Pest's arch plugin sees dependencies and can be extended

Research for issue #4, written 2026-10-17. It feeds two decisions: what counts as a dependency in the architecture spec, and where the line falls between this package and the separate Pest plugin.

**Method.** I read the source only and ran nothing. Most claims below come straight from code or from Pest's own test assertions. A few depend on following several code paths by hand; those are marked **(by reading)** and should be confirmed with a fixture before anything is built on them (see "Open questions surfaced").

## Sources (pinned)

The current Pest major is **5**. `pestphp/pest` v5.3.2 was released 2026-10-12, and Pest 5 first shipped on 2026-07-28 (D `support-policy.md:14`).

| Key | Source | Version and commit | Permalink base |
| --- | --- | --- | --- |
| **P** | `pestphp/pest` | v5.3.2 @ `73fe04f8` | `https://github.com/pestphp/pest/blob/73fe04f8c30a734fe2e269a71d4628cdf7131062/` |
| **A** | `pestphp/pest-plugin-arch` (required by P `composer.json` as `^5.0.0`) | v5.0.0 @ `244465d5`, the same commit as the head of the `5.x` branch | `https://github.com/pestphp/pest-plugin-arch/blob/244465d5dc9ea9fb5ac5c9badb7eb47d1594e4c1/` |
| **T** | `ta-tikoma/phpunit-architecture-test` (required by A as `^0.8.7`) | 0.8.7 @ `1248f3f5` | `https://github.com/ta-tikoma/phpunit-architecture-test/blob/1248f3f506ca9641d4f68cebcd538fa489754db8/` |
| **N** | `nikic/PHP-Parser` (required by T as `^4.18 \|\| ^5.0`) | v5.9.0 @ `9e33da95` | `https://github.com/nikic/PHP-Parser/blob/9e33da9553fe7786f0962b35f4e4ecf01be89def/` |
| **D** | `pestphp/docs` (the source of pestphp.com) | branch `5.x` @ `6e368f64` | `https://github.com/pestphp/docs/blob/6e368f64fed849966541bfdc1736b8180309343c/` |
| **L** | `pestphp/pest-plugin` (the plugin loader) | v5.0.0 | `https://github.com/pestphp/pest-plugin/blob/v5.0.0/` |
| **I** | Upstream threads | pestphp/pest#1227, pestphp/pest-plugin-arch#18 | |

A citation such as `A src/Blueprint.php:64-85` means that file and line range in source A at the pinned commit.

## Short answer

- **A "use" is a name in the syntax tree, not a resolved call graph.** For each PHP file, Pest collects every name node left after php-parser's `NameResolver` has run. It keeps the names that, at test runtime, are an existing class, interface, trait, enum or function, and adds six language constructs. That set covers `use` imports (even unused ones), type hints, `new`, static calls, `::class`, `instanceof`, `catch`, `extends`/`implements`, trait use, attributes and calls such as `app()`. It excludes docblocks, strings and dynamic names.
- **Edges are direct and per file.** Only the first class-like declaration in a file is described. Files without one, such as routes or helper files, are invisible. Vendor files carry no uses. `toUse` follows no trait or parent edges, but `toExtend` and `toImplement` go through reflection and do see inherited relationships.
- **Selection goes through Composer PSR-4 directories only, then a raw string-prefix match.** In most forms, a selector that matches nothing passes. That includes a typo'd *dependency* in `not->toUse`.
- **`ignoring()` is a raw `str_starts_with` prefix filter.** It is applied to subjects and to uses alike, and it does not respect namespace boundaries.
- **`@pest-arch-ignore-line` is partial and undocumented.** Only the "targeted" expectations honour it, and only on one specific line. No dependency expectation honours it.
- **Each test reports one violation.** `not->toUse` does not even name the offending class or line. Arch expectations take no custom failure message.
- **The extension surface for arch is internal.** `expect()->extend()` and `pest()->presets()->custom()` are documented. Building a real custom arch expectation, though, needs `@internal` `Pest\Arch` types, and the extractor itself lives in a 0.x third-party library.

## 1. How Pest builds a class's list of uses

1. **Parse and resolve names.** Each file is parsed with php-parser's newest-version parser and traversed with `NameResolver` (T `src/Services/ServiceContainer.php:51-56`). A file that fails to parse is silently skipped (T `src/Elements/ObjectDescriptionBase.php:50-61`).
2. **Pick the subject.** The *first* `Class_`, `Trait_`, `Interface_` or `Enum_` node in the file becomes the described object (T `ObjectDescriptionBase.php:66-73`). Anonymous classes have no `namespacedName` and are dropped (T `:79-85`). If `new ReflectionClass($name)` fails, for example because the class does not autoload, the object is dropped (T `:112-116`).
3. **Collect names.** Every `PhpParser\Node\Name` anywhere in the file's statements is collected, so names in any second class in the file count towards the first one. Only names for which `function_exists`, `enum_exists`, `class_exists`, `interface_exists` or `trait_exists` is true are kept. Autoload errors count as "not existing". A leading `\` is stripped (T `src/Asserts/Dependencies/ObjectDependenciesDescription.php:32-67`). Because these checks autoload, whether a name counts depends on the autoloaders that are live while the test runs.
4. **Add language constructs.** Pest adds the constructs `clone`, `empty`, `eval`, `exit`/`die`, `isset` and `print` as pseudo-uses (A `src/Objects/ObjectDescription.php:20-59`, A `src/Support/PhpCoreExpressions.php:26-33`).
5. **Treat vendor files as names only.** A file whose real path contains `/vendor/` becomes a `VendorObjectDescription` with an empty list of uses (A `src/Factories/ObjectDescriptionFactory.php:32-40`, A `src/Objects/VendorObjectDescription.php:19-37`).
6. **Post-filter non-vendor uses** (A `ObjectDescriptionFactory.php:50-57`):
   - `self`, `static`, `parent` and the object's own namespace string are removed (`:97-101`).
   - When `onlyUserDefinedUses` is true, PHP-internal functions, classes, interfaces and enums are removed, and so are the language constructs (`:77-92`). Framework and vendor code counts as user-defined, so `Illuminate\…` stays. The flag is true for most layers. It is false for the *subject* side of `toUse`/`not->toUse` (A `src/Blueprint.php:69`) and for the scanned namespaces in `toOnlyBeUsedIn` (A `Blueprint.php:192`).
7. **Match by exact name.** A dependency exists when a use string is identical to the name of an object in the dependency layer (T `src/Asserts/Dependencies/DependenciesAsserts.php:58-88`). The comparison is skipped when the subject layer and the dependency layer hold exactly the same set of objects (T `:72-75`).

## 2. What counts as a use

| Construct | Counted? | Why (source) |
| --- | --- | --- |
| `use Foo\Bar;` import, even unused | **Yes** | `UseItem` has a `name` sub-node (N `lib/PhpParser/Node/UseItem.php:34-36`), and step 3 collects every `Name`. Pest's own fixture reports the violation at the import line (A `tests/ToOnlyUse.php:23-29`, which points at A `tests/Fixtures/Models/Product.php:7`). |
| Group use `use Foo\{Bar, Baz};` on its own | **No** (by reading) | The item names stay relative (`Bar`), and `NameResolver` only records an alias (N `lib/PhpParser/NodeVisitor/NameResolver.php:62-65,172-181`), so `class_exists('Bar')` fails. Actual uses of `Bar` in the code are resolved and do count. |
| Parameter, return, property, class-constant and property-hook types, including nullable, union and intersection types | Yes | Resolved to fully qualified names (N `NameResolver.php:98-128,184-212`). |
| `new Foo`, `Foo::bar()`, `Foo::$x`, `Foo::CONST`, `Foo::class`, `instanceof Foo` | Yes | N `NameResolver.php:131-139`. |
| `catch (Foo $e)` | Yes | N `NameResolver.php:140-143`. |
| `extends`, `implements`, interface `extends` | Yes | N `NameResolver.php:66-97`. |
| Trait use inside a class, including `insteadof` | Yes, as an edge to the trait only | N `NameResolver.php:150-166`. Names inside the trait's body belong to the trait's own file, not to the classes that use it. |
| Attributes `#[Foo(Bar::class)]` | Yes, both `Foo` and `Bar` | N `NameResolver.php:262-268` and the `ClassConstFetch` rule. |
| Function calls `app()`, `resolve()`, `request()` | Yes, as the global function name (e.g. `app`) | An unqualified function name inside a namespace is left unresolved (N `NameResolver.php:144-148,236-250`, N `lib/PhpParser/NameContext.php:123-131`). `function_exists('app')` is true for the global helper. The dependency side resolves `app` to a function object (A `src/Repositories/ObjectsRepository.php:79-83`). |
| `app(Foo::class)` | `app` and `Foo` | Two separate name nodes. |
| `app('foo')`, `config('x.model')`, `'App\Models\User'` in a string | Only `app`/`config` | Strings are not name nodes. |
| `$this->app->make(...)`, `new $class`, `$class::m()` | No | These are expressions, not `Name` nodes. |
| Facade call `Cache::get()` | The facade class only, not the service behind it | A plain name. |
| Docblocks (`@var`, `@param`, `@return Collection<User>`) | **No** | Comments are not name nodes. T has an unused docblock helper marked `@todo` (T `ObjectDependenciesDescription.php:75-113`). |
| Constants (`PHP_EOL`, `true`, `FOO`) | No | Filtered by the existence check. |
| A name that does not autoload (a typo, a removed class, a planned class not yet created) | **No, dropped silently** | T `ObjectDependenciesDescription.php:38-54`. |
| PHP-internal functions (`json_encode`) | Only on the subject side of `toUse`/`not->toUse`. Ignored by `toOnlyUse` | A `ObjectDescriptionFactory.php:77-92`, A `Blueprint.php:69`. |
| PHP-internal classes (`DateTime`, `PDO`) as a dependency target | **Never matched** (by reading) | `allByNamespace()` finds no PSR-4 directory and returns `[]` (A `ObjectsRepository.php:85-89`). So `not->toUse('DateTime')` passes vacuously. |
| Code in files with no class-like declaration (routes, config, helper files) | Invisible as a subject | T `ObjectDescriptionBase.php:75-77`. |
| Functions as subjects | Never have uses | A `src/Objects/FunctionDescription.php:20-36` sets `uses` to empty. |

**Reflection-based expectations are a different notion.** `toExtend`, `toImplement`, `toUseTrait(s)` and `toHaveAttribute` use reflection, not the list of uses. `isSubclassOf` and `implementsInterface` therefore see inherited relationships (P `src/Expectation.php:696-704,813-826`). A class that implements an interface through its parent passes `toImplement` but has no `toUse` edge to that interface. The positive and negative forms can also differ: `toUseTraits` walks parent classes and nested traits (P `Expectation.php:724-760`), while `not->toUseTraits` checks direct traits only (P `src/Expectations/OppositeExpectation.php:479-494`).

## 3. How subjects and dependencies are selected

- **Allowed inputs.** A subject (`expect(...)`) or a dependency argument is a string, or an array of strings, holding a class, a namespace, a function name or a language construct. Any string that contains `/` is rejected (A `src/ValueObjects/Targets.php:31-46`, A `src/ValueObjects/Dependency.php:26-35`).
- **Resolution.**
  - A function or construct name becomes a single function object (A `ObjectsRepository.php:73-83`).
  - Anything else is mapped onto directories through the root project's **Composer PSR-4 prefixes only**: `getPrefixesPsr4()`, which includes vendor packages' prefixes (A `ObjectsRepository.php:47-64,144-179`). For `App\Models` that means `app/Models/` plus `app/Models.php` if it exists. Classes loaded only through a classmap, PSR-0 or `files` entry can be neither subjects nor dependency targets.
  - The files found there are then filtered by **string prefix** on the class name (`leaveByNameStart`) (A `src/Factories/LayerFactory.php:63-65`, T `src/Elements/Layer/LayerLeave.php:31-38`). Because the directory lookup comes first, the subject side does not bleed into sibling namespaces such as `App\ModelsLegacy`.
  - User-defined functions whose lowercased name starts with the selector are added too (A `ObjectsRepository.php:114-137`).
- **Wildcards.** `*` matches exactly one namespace segment (`[^\\]+`) (A `LayerFactory.php:63-64`, D `arch-testing.md:740-752`). A bare `'*'` matches no PSR-4 prefix and so selects nothing (by reading). Pest's own suite has `arch('strict types')->expect('*')->toUseStrictTypes()` (A `tests/Arch.php:14`), and on this reading that test is vacuous.
- **Modifiers** (`classes()`, `interfaces()`, `traits()`, `enums()`, `abstractClasses()`, `extending()`, `implementing()`, `using()`) add exclusion callbacks (A `src/PendingArchExpectation.php:39-116`). `LayerFactory` applies those callbacks to **every** layer it builds for the expectation, dependency layers included (A `LayerFactory.php:71-73`, A `src/Options/LayerOptions.php:29-38`). **(By reading)** `expect('App')->classes()->not->toUse('App\Contracts')` therefore never matches interfaces, and `classes()->not->toUse('app')` never matches the `app` function. Both pass silently. Separately, the docs list an `abstracts()` modifier (D `arch-testing.md:888-899`), but the code only has `abstractClasses()` (A `PendingArchExpectation.php:49-54`).
- **Universe for `toOnlyBeUsedIn`, `toBeUsedInNothing` and `not->toBeUsed`.** These scan every non-vendor PSR-4 root except those under `<root>/tests` (A `src/Support/Composer.php:30-39,46-69`, A `Blueprint.php:191`). In a default Laravel app that includes `Database\Factories` and `Database\Seeders`.

### When a selector matches nothing

| Expectation | Empty subject | Empty or typo'd dependency |
| --- | --- | --- |
| Targeted (`toBeFinal`, `toImplement`, `toExtend`, `toHaveSuffix`, `toUseStrictTypes`, `toBeCasedCorrectly`, …) | **Passes**: the loop runs over no objects (A `Blueprint.php:94-143`) | n/a. An unknown class or interface name is passed to `ReflectionClass::isSubclassOf()`/`implementsInterface()`, which PHP rejects with an exception (by reading), so it errors rather than passes. |
| `toUse` (positive) | **Fails**: "Expecting 'X' to use 'Y'" (A `Blueprint.php:74-80`, A `tests/ToUse.php:69-76`) | **Fails** |
| `not->toUse` | **Passes** | **Passes** |
| `toOnlyUse` / `toUseNothing` | **Passes** | A typo'd *allowed* entry makes the rule stricter, so it fails loudly (safe) |
| `toOnlyBeUsedIn` / `toBeUsedInNothing` / `not->toBeUsed` | **Passes** | A typo'd *allowed user* makes the rule stricter, so it fails loudly (safe) |

Pest's own `php` preset shows the silent-pass case: it bans `'echo'`, `'goto'` and `'global'` through `not->toBeUsed()` (P `src/ArchPresets/Php.php:22-26,88`). None of those resolves to a function, a construct in `PhpCoreExpressions::$ENABLED` or a PSR-4 path, so those three entries can never fail (by reading). No warning is emitted anywhere for an empty selector. I found no such code in A or P.

## 4. Semantics of the expectations an architecture spec would generate

- **`not->toUse(deps)`** runs the *positive* `toUse` once per dependency and inverts the result through an "opposite" callback (P `OppositeExpectation.php:70-78`, A `src/SingleArchExpectation.php:152-178`). For a single subject string this is universal: the rule fails if any class in the subject layer uses any object in the dependency layer.
  - **Array subjects are existential (by reading).** `expect(['App\Http','App\Console'])->not->toUse('App\Models')` passes whenever *at least one* subject does not use `App\Models`. Positive `toUse` throws at the first subject that lacks the dependency (A `Blueprint.php:68-81`), and that throw is swallowed as a pass for the negated form. Pest's tests cover only single-element arrays (A `tests/ToUse.php:10-21`).
  - The same applies to **`expect([A, B])->not->toBeUsedIn(X)`**, which is built as `ToUse(X, [A, B])` with the opposite callback (P `OppositeExpectation.php:577-585`, A `src/Expectations/ToBeUsedIn.php:22-36`). It fails only if X uses *all* of A and B.
  - Arrays on the dependency side of `not->toUse` are safe, because each dependency gets its own expectation.
- **`toOnlyUse(allowed)`.** Allowed names are the subject's own objects plus every object in the allowed layers, compared by exact name (A `Blueprint.php:150-180`). PHP-internal symbols are ignored. Framework and vendor classes and helpers such as `collect()` are not ignored, so they must be listed or ignored explicitly.
- **`toOnlyBeUsedIn(allowed)`** builds the scanned universe and removes allowed users by **name prefix** (`excludeByNameStart`) (A `Blueprint.php:187-213`). Two consequences follow, both **(by reading)**:
  - The subject's own namespace is *not* excluded automatically. In `expect('App\Models')->toOnlyBeUsedIn('App\Repositories')`, a model that references another model is a violation unless `App\Models` is also listed. Pest's own Laravel preset and docs list the subject among the allowed users, which is consistent with this (P `src/ArchPresets/Laravel.php:152-153`, D `arch-testing.md:25-27`).
  - Allowed prefixes bleed into siblings: allowing `App\Features\Billing` also allows `App\Features\BillingReports`.
- **Boundary workaround (by reading).** Every prefix check is a plain `str_starts_with`, so appending a trailing `\` (as in `'App\Features\Billing\'`) gives namespace-exact matching in `ignoring()` and in the allowed list of `toOnlyBeUsedIn`. That trailing `\` also still resolves the directory for selectors.
- **Laravel preset collision.** The preset's `laravel` rules require every class in `App\Features` to be a class with a `resolve` method, for Laravel Pennant features (P `Laravel.php:30-36`). This collides with any architecture spec that uses an `App\Features` layer when the app also enables `arch()->preset()->laravel()`.

## 5. `ignoring()` and `@pest-arch-ignore-line`

**`ignoring()`**

- `->ignoring($x)` appends to the expectation it is chained to. A `GroupArchExpectation` forwards the call to all of its parts (A `SingleArchExpectation.php:61-68`, A `src/GroupArchExpectation.php:34-41`). A preset applies it to all of its expectations (P `src/ArchPresets/AbstractPreset.php:37-43`).
- It is merged with the per-test global list from `$this->arch()->ignore([...])` and `ignoreGlobalFunctions()` (A `LayerOptions.php:29-38`, A `src/Options/TestCaseOptions.php:26-41`, D `arch-testing.md:791-801`).
- The merged list is applied to **every layer**, subject and dependency alike, in two ways (A `LayerFactory.php:39-58,67-69`, A `Blueprint.php:102-106`):
  1. It drops each *use* whose string starts with an entry.
  2. It drops each *object* whose name starts with an entry.
- The match is a raw string prefix. `ignoring('App\Http\Controllers\LegacyReportController')` also ignores `…LegacyReportControllerV2`, and `ignoring('app')` also ignores `app_path()`.
- **Laziness.** Expectations are verified lazily, on the next chained call or on `__destruct` (A `SingleArchExpectation.php:104-109,144-147`). That is why an `ignoring()` placed *after* the expectation still takes effect, and why it affects only that one expectation (A `tests/ToUse.php:48-57`).

**`@pest-arch-ignore-line` and `@pest-arch-ignore-next-line`**

- They are honoured only inside `Blueprint::targeted()` (A `Blueprint.php:116-134`), and only on the single line that the expectation's line finder returns. For most targeted expectations that is the first line containing the substring `class` (e.g. P `Expectation.php:702,821`; A `src/Support/FileLineFinder.php:18-35`).
- `expectToUse`, `expectToOnlyUse` and `expectToOnlyBeUsedIn` contain no such check (A `Blueprint.php:64-85,150-213`). So `toUse`, `not->toUse`, `toOnlyUse`, `toUseNothing`, `toOnlyBeUsedIn`, `toBeUsedIn`, `toBeUsedInNothing` and `not->toBeUsed` never honour it.
- **History.** The feature was added on 2024-08-07 (A commit `4e334d6`). In I pestphp/pest#1227 the maintainer confirmed it was added "when expectations are done in a certain way, but not in all of them", then "removed the feature" from the docs and pointed users to `->ignoring()`. The fix PR, I pestphp/pest-plugin-arch#18, was closed unmerged on 2025-03-30. The 3.x, 4.x and 5.x docs do not mention it (D, by grep).

## 6. Stopping and reporting

- **One violation per test.**
  - Each expectation throws at its first failure: targeted on the first failing object (A `Blueprint.php:108-138`), `toOnlyUse` with `return` after the first bad use (A `:169-174`), and `toOnlyBeUsedIn` from `$objects[0]` (A `:203-207`).
  - The first throw aborts the rest of the chain in that test, and a preset runs as a single test.
  - Separate `arch()` calls are separate tests (A `src/Autoload.php:18-25`).
- **Message formats.** Arch expectations accept no custom message. Only the `arch('description')` test name can carry a reason.

| Expectation | Message | Location |
| --- | --- | --- |
| `not->toUse` | `Expecting '{subject}' not to use '{dependency}'.` A subject longer than about 38 characters is shortened, e.g. `'Tests\Fixtures\Controllers\Pr…roller'` (P `OppositeExpectation.php:664-676`, A `tests/ToUse.php:30-46`) | **None.** No offending class, file or line. |
| `toOnlyUse` / `toUseNothing` | `Expecting '{subject}' to only use '{allowed}'. However, it also uses '{use}'.` (A `src/Expectations/ToOnlyUse.php:38-51`) | File and line of the first `Name` node that matches the use, usually the `use` import line (A `Blueprint.php:245-293`, A `tests/ToOnlyUse.php:23-29`) |
| `toOnlyBeUsedIn` / `not->toBeUsed` | `Expecting '{used class}' not to be used on '{user class}'.` (A `src/Expectations/ToOnlyBeUsedIn.php:40-58`) | Same, in the user's file (A `tests/ToOnlyBeUsedIn.php:26-32`) |
| Targeted | `Expecting '{relative path}' {what}.` (A `src/Expectations/Targeted.php:46-53`) | The line returned by the line finder |
| `toUse` (positive) | `Expecting '{subject}' to use '{dependency}'.` (A `src/Expectations/ToUse.php:39-44`) | None |

- Locations reach the editor through Collision's `RenderableOnCollisionEditor` (A `src/Exceptions/ArchExpectationFailedException.php:15-34`).
- `--arch` and `--exclude-arch` map to `--group=arch` and `--exclude-group=arch` (A `src/Plugin.php:17-28`).
- **Cost.** The existence checks autoload every referenced name. A dependency layer on a vendor namespace parses and reflects every file in that directory (T `ObjectDescriptionBase.php:112-116`). Results are cached per process (A `ObjectsRepository.php:93-111`).

## 7. Extension points and their stability

**Documented, public-facing**

- `expect()->extend()`, `intercept()` and `pipe()` (D `custom-expectations.md`).
- `pest()->presets()->custom('name', fn (array $userNamespaces) => [...])`, then `arch()->preset()->name()` (D `arch-testing.md:707-738`, P `src/Configuration/Presets.php:10-15`). Note that D `creating-plugins.md` documents `pest()->preset('ddd', …)` instead. `Configuration::__call` forwards that to `UsesCall`, which has no `preset` method (P `src/Configuration.php:96-99`), so the documented form appears to be stale (by reading).
- `Pest\Plugin::uses(Trait::class)` in an autoloaded file, to add `$this->…` methods (P `src/Plugin.php:19-24`, D `creating-plugins.md`). It is the mechanism the arch plugin itself uses (A `src/Autoload.php:10`).
- Plugin classes listed in composer `extra.pest.plugins` and loaded by interface (L `src/Loader.php`). Example: A's `composer.json` registers `Pest\Arch\Plugin`, which implements `HandlesArguments`.

**What a custom arch expectation needs**

- An `expect()->extend()` closure is returned directly, and so stays lazy and supports `->ignoring()`, *only if* its declared return type is exactly `Pest\Arch\Contracts\ArchExpectation` (P `Expectation.php:337-339`). Otherwise the closure's return value is discarded and the `Expectation` is returned (P `:343-348`), so `->ignoring()` cannot be chained.
- `ArchExpectation` is `@internal` (A `src/Contracts/ArchExpectation.php:10-15`). So are its building blocks: `Targeted`, `Blueprint`, `SingleArchExpectation`, `GroupArchExpectation`, `LayerOptions`, `ObjectsRepository`, `ObjectDescriptionFactory`, `Composer` and every `Objects\*` class. Every file in A except `Autoload.php`, `Layer.php` and `ValueObjects/Violation.php` carries `@internal` (by grep).

**Also `@internal`, even though documented or reached through documented calls**

- P's `Configuration` (P `src/Configuration.php:11-15`), `Preset` and `Preset::custom` (P `src/Preset.php:18-21,63-66`), and every preset class.
- The `Extendable` trait behind `expect()->extend()` (P `src/Concerns/Extendable.php:9-14`).
- All plugin hook interfaces: `HandlesArguments`, `Bootable`, `Terminable`, `AddsOutput`, `HandlesOriginalArguments` (P `src/Contracts/Plugins/*.php`).
- L's `Loader`.

**The extractor**

The extractor is T: a third-party 0.x library with no `@internal` or `@api` markers. A pins it as `^0.8.7`, which means 0.8.x only.

**Stability promise**

- Pest promises semver (D `support-policy.md:20-26`). By the usual reading of `@internal`, that excludes everything listed above, which may change in any release.
- In practice the code has been stable. From A v4.0.2 to v5.0.0 the only `src/` changes are docblock removals in three files. From v3.1.1 to v4.0.2 the changes are small (git diff).

**What a separate Pest plugin can build on** (options for the boundary decision, not a recommendation)

| Option | Uses | Pros | Cons |
| --- | --- | --- | --- |
| a. Emit or declare tests with documented arch expectations (`not->toUse`, `toOnlyBeUsedIn`, `not->toImplement`, `ignoring`) | Public API only | Stable surface; "Pest is the judge" holds literally | Inherits everything above: one failure per test, no offending class for `not->toUse`, prefix matching, existential array subjects, the empty-selector pass, and the modifiers that filter dependency layers. The picture must copy T's extraction exactly. |
| b. A custom preset (`pest()->presets()->custom`) | Documented, but `@internal` underneath | One line for users | A whole preset is a single test, so it stops at the first violation across *all* rules |
| c. Custom arch expectations via `extend()` returning `ArchExpectation`, built on `Targeted`/`Blueprint` | `@internal` A types | Native look; `ignoring()` works | Tied to internals and to T 0.8.x; needs a minor-version pin, not just a major-version pin |
| d. Plain Pest tests (`test`, `expect`, datasets) that call this package's own rule library and extractor, and fail with a full violation list | Public API only | One extractor shared by the picture and the judge; complete lists; custom messages with the reason; exact namespace matching | The dependency semantics become the package's own, not Pest's arch semantics; the package must maintain the extractor itself |

## 8. Verdicts on the WIP's "Pest behaviours we design around" table

| WIP claim | Verdict | Evidence | Consequence |
| --- | --- | --- | --- |
| Arch tests stop at the first violation | **True, and worse than stated** | A `Blueprint.php:108-138,169-174,203-207`; the chain aborts on the first throw. `not->toUse` reports *no* offending class or line (P `OppositeExpectation.php:664-676`, A `tests/ToUse.php:30-46`). | "One test per rule pair" still tells the agent only the layer pair. To find the class it needs a scan, per-class datasets, or option d. |
| `@pest-arch-ignore-line` is always honoured | **False** | Honoured only by targeted expectations, on one line (A `Blueprint.php:116-134`). Never honoured by dependency expectations (A `Blueprint.php:64-85,150-213`). Undocumented; upstream fix closed unmerged (I #1227, #18). | The `grep` CI guard is harmless but matters only for targeted rules such as `not->toImplement` and `toBeCasedCorrectly`, where the comment can suppress a violation. It cannot weaken `not->toUse` or `toOnlyBeUsedIn` rules. |
| A selector that matches nothing passes | **Mostly true** | True for targeted, `not->toUse`, `toOnlyUse` and `toOnlyBeUsedIn`. False for positive `toUse`, which fails. Also true for a typo'd *dependency* in `not->toUse` (section 3 table). No warning anywhere. | The unassigned-class test catches a typo'd *subject* namespace only indirectly. A check that every layer and contract selector in the architecture spec resolves to at least one class would catch both sides directly. |
| Custom arch expectations rely on internal Pest classes | **True** | A `src/Contracts/ArchExpectation.php:10-15`, P `Expectation.php:337-339`. Even the plugin hooks are `@internal` (P `src/Contracts/Plugins/*.php`). | "Pin Pest's major version" is not enough: `@internal` code can change in a minor release, and T is 0.x. Pin the minor version or keep a fixture suite that runs on every upgrade. |

### Other WIP statements that touch Pest

- **"The scanner uses `nikic/php-parser` with its `NameResolver`, which Pest's arch plugin already pulls in."** True (transitively, through T `composer.json`; T `ServiceContainer.php:53-54`). Matching Pest also requires T's runtime existence filter, the first-class-per-file rule, the import-counts rule and A's post-filters (section 1).
- **Failure-message template `{Layer} must not depend on {Target}… Why: {reason}`.** This is not achievable through arch expectations, which take no message. Only the `arch()` description can carry it.
- **Layer allowlist "dependencies on unmapped code are ignored, as in Deptrac".** This holds if the rules are generated as `not->toUse` per forbidden pair. It does not hold if they are generated as `toOnlyUse`, which counts vendor and framework names.
- **Feature privacy through `toOnlyBeUsedIn` per feature.** This works only if the feature's own namespace is in the allowed list (section 4). Watch for prefix bleed between feature names such as `Billing` and `BillingReports`.

## Open questions surfaced

1. **Boundary.** Should the Pest plugin enforce through Pest's arch expectations (option a or c, inheriting T's extractor and the gaps above)? Or should it run this package's own extractor inside plain Pest tests (option d), so that the picture and the judge share one implementation?
2. **Definition of a dependency.** Does the architecture spec adopt Pest's notion as-is, where unused imports count, docblocks do not, only the first class in a file is described, and names that do not autoload are dropped? Or does it define its own notion, which the Pest plugin then implements?
3. **Symbols that do not autoload.** Pest drops them silently. Should the package report them as "unknown"? This matters for planned-but-not-yet-created classes at the plan skeleton gate.
4. **Complete violation lists.** Given one failure per test and no class named in `not->toUse` failures, how does the Pest plugin give agents a full list: per-class datasets, a collect-all test, or `scan` output?
5. **Fixture confirmation.** Every **(by reading)** item needs a fixture run before it is relied on, especially the existential array subjects, modifiers filtering dependency layers, intra-namespace use under `toOnlyBeUsedIn`, internal PHP classes never matching, and the trailing-`\` workaround. If confirmed, should these be reported upstream?
6. **Laravel preset coexistence.** How should an `App\Features` layer coexist with the `laravel` preset's Pennant rules for `App\Features`?
