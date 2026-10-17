# Extracting class dependencies statically from a Laravel app

Research for [#6](https://github.com/kfriars/laravel-visual-spec/issues/6) on the map [#1](https://github.com/kfriars/laravel-visual-spec/issues/1). Researched 2026-10-17 against the releases current on that date. This is input to "What counts as a dependency?" ([#10](https://github.com/kfriars/laravel-visual-spec/issues/10)) and to the "Laravel container knowledge" fog on the map. It decides nothing.

The WIP draft ([docs/wip-spec-2026-10-16.md](../wip-spec-2026-10-16.md)) proposes nikic/php-parser with `NameResolver`. Here that is one candidate among five, checked against the source. Pest arch's *semantics* belong to [#4](https://github.com/kfriars/laravel-visual-spec/issues/4); its extractor is compared here only as an extractor.

## Answer in brief

- **Every candidate is built on nikic/PHP-Parser 5.** PHPStan, Deptrac, BetterReflection and Pest's extractor all require it ([phpstan-src composer.json](https://github.com/phpstan/phpstan-src/blob/2.3.1/composer.json), [deptrac composer.json](https://github.com/deptrac/deptrac/blob/4.7.2/composer.json), [BetterReflection composer.json](https://github.com/Roave/BetterReflection/blob/6.74.0/composer.json), [phpunit-architecture-test composer.json](https://github.com/ta-tikoma/phpunit-architecture-test/blob/0.8.7/composer.json)). So the real choice is what runs on top of the parser: collecting names (Pest), collecting names plus docblocks (Deptrac), reflecting declarations (BetterReflection), or full type inference (PHPStan, plus Larastan for Laravel).
- **The syntactic extractors agree on the core:** imports, native type hints, `new`, static calls, static properties and constants, extends/implements/traits, attributes, `::class`, `instanceof` and `catch`. They **disagree at the edges**, and those edges matter if the picture and the Pest plugin are to agree:
  - Pest counts unused imports. Deptrac does not by default (its code defaults to `class` and `function`, though its docs say `use` is a default).
  - Deptrac reads docblocks and generics. Pest does not.
  - Pest silently drops any name that doesn't autoload.
  - Deptrac copies a parent's dependencies into each subclass.
- **No syntactic extractor sees string class names, inferred receiver types (`$post->comments`), or code outside PSR-4 class files:** routes, config, Blade views and Livewire single-file components.
- **PHPStan sees the most, but it isn't built to hand out edges.** It infers types, and with Larastan it resolves container lookups and facades against the booted app. Its dependency data, though, is internal, per file rather than per class, and over-inclusive: every referenced class comes with its ancestors. The only stable way in is a custom Collector, which costs a full analysis. Larastan boots the app to work.
- **Incremental rescans are easy for a pure name-based extractor.** Its output for a file depends on that file alone, so caching per file by content hash is sound; Deptrac does exactly that. PHPStan has the strongest incremental engine, but it serves PHPStan's own analysis. Pest arch has no persistent cache.
- **Laravel blind spots fall into four classes:**
  - **A.** Visible, but the edge points at the wrong thing: the facade instead of its root, the abstract in `app(Foo::class)`, or `#[Bind]` and `#[ObservedBy]` pointing backwards.
  - **B.** Invisible, but recoverable by replaying Laravel's conventions statically: real-time facades, aliases, event discovery, policy guessing, Blade tags, route files, constant-argument bindings, factories.
  - **C.** Recoverable only by booting the app: closure bindings, custom guess callbacks, environment-dependent wiring, deferred providers.
  - **D.** Unrecoverable even by booting: `morphTo` targets, `app($runtimeString)`, dynamic components.
- **Laravel already exposes much of B and C once booted**, through `route:list --json`, `event:list --json`, `model:show --json` and `config:show`. But no first-party command lists container bindings, and the container stores class bindings as closures.

## Sources examined

Shallow clones at the tag current on 2026-10-17. All code citations link to that tag.

| Project | Tag | Commit | Role here |
| --- | --- | --- | --- |
| nikic/PHP-Parser | v5.9.0 | `9e33da9553fe` | Parser and `NameResolver` |
| pestphp/pest-plugin-arch | v5.0.0 | `244465d5dc9e` | Pest arch plugin (requires the next row) |
| ta-tikoma/phpunit-architecture-test | 0.8.7 | `1248f3f506ca` | The extractor Pest arch uses |
| pestphp/pest | v5.3.2 | `73fe04f8c30a` | Pest core (context only) |
| phpstan/phpstan-src | 2.3.1 | `cb3b7e99489b` | Dependency data, result cache, Collectors |
| larastan/larastan | v3.13.0 | `24ee69a7705f` | Laravel knowledge for PHPStan |
| deptrac/deptrac | 4.7.2 | `060c7edca503` | Extractors, emitters, collectors, cache |
| Roave/BetterReflection | 6.74.0 | `76b224f12057` | Static reflection |
| laravel/framework | v13.37.0 | `685f8d6c2ab4` | Where Laravel's indirection lives |
| livewire/livewire | v4.4.7 | `5976d18b1c80` | Livewire component resolution |

No code from these clones was run. Cost statements are therefore either published figures (PHPStan) or derived from reading the code, and labelled as such.

## 1. The extractors

### 1.1 nikic/PHP-Parser with `NameResolver`

`NameResolver` is a resolution pass, not a dependency extractor: it rewrites names in place to fully qualified `Name\FullyQualified` nodes, and you write the visitor that collects them. It resolves names in:

- `extends` and `implements`;
- attributes;
- signatures (parameter and return types);
- property, class-constant and property-hook types;
- `StaticCall`, `StaticPropertyFetch`, `ClassConstFetch` (so `Foo::class`), `New_` and `Instanceof_`, but only when the class is a literal name;
- `catch` types;
- trait uses and their adaptations.

See [NameResolver.php L55–169](https://github.com/nikic/PHP-Parser/blob/v5.9.0/lib/PhpParser/NodeVisitor/NameResolver.php#L55-L169).

Unqualified function and constant names inside a namespace are left unresolved, with a `namespacedName` attribute, because PHP falls back to the global function at runtime ([L246–250](https://github.com/nikic/PHP-Parser/blob/v5.9.0/lib/PhpParser/NodeVisitor/NameResolver.php#L246-L250); [Name_resolution.markdown L32](https://github.com/nikic/PHP-Parser/blob/v5.9.0/doc/component/Name_resolution.markdown)).

It does not touch doc comments or string literals. The docs say the `NameContext` "is intended to be used for name resolution operations outside the AST itself, such as class names inside doc comments" ([Name_resolution.markdown L84](https://github.com/nikic/PHP-Parser/blob/v5.9.0/doc/component/Name_resolution.markdown)). So docblocks need a second parser, such as phpstan/phpdoc-parser, which is what Deptrac uses.

Resolution depends only on the file's own namespace and `use` statements, so a visitor's output for a file is a function of that file alone. That is what makes per-file caching sound. The library has no caching or incremental mode of its own; its performance doc only advises reusing parser objects and running without Xdebug ("approximately *five times slower*") ([Performance.markdown](https://github.com/nikic/PHP-Parser/blob/v5.9.0/doc/component/Performance.markdown)).

### 1.2 Pest arch's extractor (ta-tikoma/phpunit-architecture-test, used by pest-plugin-arch)

pest-plugin-arch v5.0.0 requires `ta-tikoma/phpunit-architecture-test ^0.8.7` ([composer.json](https://github.com/pestphp/pest-plugin-arch/blob/v5.0.0/composer.json)), which requires `nikic/php-parser ^4.18 || ^5.0`. The WIP's claim that Pest "already pulls in" PHP-Parser holds, transitively.

How the extractor works:

- **Parsing.** It creates the parser and one `NameResolver` ([ServiceContainer.php L51–56](https://github.com/ta-tikoma/phpunit-architecture-test/blob/0.8.7/src/Services/ServiceContainer.php#L51-L56)). It keeps only the **first** class, interface, trait or enum in each file, then calls `new ReflectionClass($name)`. That loads, and therefore executes, the file; if it fails, the file is skipped ([ObjectDescriptionBase.php L63–116](https://github.com/ta-tikoma/phpunit-architecture-test/blob/0.8.7/src/Elements/ObjectDescriptionBase.php#L63-L116)).
- **Uses.** An object's dependencies are **every `Node\Name` anywhere in the file's statements**, kept only if `function_exists`, `enum_exists`, `class_exists`, `interface_exists` or `trait_exists` returns true for it. Each check autoloads the name, and any name that fails is dropped silently ([ObjectDependenciesDescription.php L32–67](https://github.com/ta-tikoma/phpunit-architecture-test/blob/0.8.7/src/Asserts/Dependencies/ObjectDependenciesDescription.php#L32-L67)).
- **Pest's extra filtering.** Pest then:
  - drops PHP-internal symbols (`isUserDefined()`);
  - drops `self`, `static`, `parent` and the class's own namespace name;
  - adds the language constructs `clone`, `empty`, `eval`, `exit`/`die`, `isset` and `print` as pseudo-uses.

  See [ObjectDescriptionFactory.php L50–101](https://github.com/pestphp/pest-plugin-arch/blob/v5.0.0/src/Factories/ObjectDescriptionFactory.php#L50-L101), [Objects/ObjectDescription.php L29–59](https://github.com/pestphp/pest-plugin-arch/blob/v5.0.0/src/Objects/ObjectDescription.php#L29-L59) and [PhpCoreExpressions.php L26–33](https://github.com/pestphp/pest-plugin-arch/blob/v5.0.0/src/Support/PhpCoreExpressions.php#L26-L33).
- **Docblocks.** These are parsed only to describe method arguments, return types and properties, never into `uses` ([MethodDescription.php L49–56](https://github.com/ta-tikoma/phpunit-architecture-test/blob/0.8.7/src/Asserts/Methods/Elements/MethodDescription.php#L49-L56); `getDocBlockTypeWithNamespace` in [ObjectDependenciesDescription.php L75–113](https://github.com/ta-tikoma/phpunit-architecture-test/blob/0.8.7/src/Asserts/Dependencies/ObjectDependenciesDescription.php#L75-L113)).
- **Which files it scans.** Only directories registered as Composer **PSR-4** prefixes are scanned ([ObjectsRepository.php L53–63, L144–178](https://github.com/pestphp/pest-plugin-arch/blob/v5.0.0/src/Repositories/ObjectsRepository.php#L53-L178); [Composer.php L46–69](https://github.com/pestphp/pest-plugin-arch/blob/v5.0.0/src/Support/Composer.php#L46-L69)). In a standard Laravel app that leaves out `routes/`, `config/`, `bootstrap/` and `resources/views/`.
- **Caching.** Parsed objects are cached in memory per process, keyed by the namespace a test asked for. Requests for overlapping namespaces, such as `App` and `App\Http`, parse the same files again. Nothing is written to disk ([ObjectsRepository.php L94–110](https://github.com/pestphp/pest-plugin-arch/blob/v5.0.0/src/Repositories/ObjectsRepository.php#L94-L110)).

These choices have consequences, all derived from the code above rather than run:

- An **unused** `use Foo;` import counts as a dependency, because the import's name is a `Node\Name`.
- An unused **group** import (`use App\Models\{User, Post};`) does not count. Its items are relative names such as `User` under a prefix ([GroupUse.php](https://github.com/nikic/PHP-Parser/blob/v5.9.0/lib/PhpParser/Node/Stmt/GroupUse.php)), so the existence checks fail.
- A reference to a deleted or misspelled class vanishes instead of showing up as a dangling edge.
- Root aliases such as `\Cache::get()`, and real-time facades such as `Facades\App\...`, only survive the filter if Laravel's `AliasLoader` is registered in the test process.

### 1.3 Deptrac

Deptrac has two parts:

- **Extractors** record every reference into an AST map.
- **Emitters** turn the map into dependencies, according to `analyser.types`.

**Collectors** are different: they decide which *layer* a token belongs to. Their kinds are `classLike`, `directory`, `implements`, `extends`, `inherits`, `attribute`, `composer`, `glob`, `tagValueRegex`, `method`, `uses`, `bool`, `layer` and others ([docs/collectors.md](https://github.com/deptrac/deptrac/blob/4.7.2/docs/collectors.md)). They don't find dependencies.

What the extractors and emitters do:

- **Dependency kinds** are an enum ([DependencyType.php](https://github.com/deptrac/deptrac/blob/4.7.2/src/Contract/Ast/AstMap/DependencyType.php)): `use`, `inherit`, `returntype`, `parameter`, `new`, `static_property`, `static_method`, `instanceof`, `catch`, `variable` (property or `@var`), `throw`, `const`, three anonymous-class kinds, `attribute`, `superglobal_variable` and `unresolved_function_call`.
- **`::class`** is recorded as `const`, through the `ClassConstFetch` extractor ([ClassConstantExtractor.php](https://github.com/deptrac/deptrac/blob/4.7.2/src/DefaultBehavior/Ast/Extractors/ClassConstantExtractor.php)).
- **Docblocks:**
  - `@param`, `@return` and `@throws` on methods;
  - `@var` on variables and assignments;
  - class-level `@method` and `@property` tags.

  Generics, array shapes and callables are resolved with phpstan/phpdoc-parser ([ClassMethodExtractor.php](https://github.com/deptrac/deptrac/blob/4.7.2/src/DefaultBehavior/Ast/Extractors/ClassMethodExtractor.php), [ClassLikeExtractor.php L55–81](https://github.com/deptrac/deptrac/blob/4.7.2/src/DefaultBehavior/Ast/Extractors/ClassLikeExtractor.php#L55-L81), [TypeResolver.php L90–100](https://github.com/deptrac/deptrac/blob/4.7.2/src/Core/Ast/Parser/TypeResolver.php#L90-L100)).
- **Default types, where docs and code disagree.** The docs say `class` and `use` are defaults ([configuration.md L40–50](https://github.com/deptrac/deptrac/blob/4.7.2/docs/configuration.md)). The code defaults to `class` and `function` ([Configuration.php L254–258](https://github.com/deptrac/deptrac/blob/4.7.2/src/Supportive/DependencyInjection/Configuration.php#L254-L258), [DeptracExtension.php L60–64](https://github.com/deptrac/deptrac/blob/4.7.2/src/Supportive/DependencyInjection/DeptracExtension.php#L60-L64)). `use` statements are recorded on the *file*, not the class ([UseExtractor.php](https://github.com/deptrac/deptrac/blob/4.7.2/src/DefaultBehavior/Ast/Extractors/UseExtractor.php)). So with the code defaults, an unused import is **not** a dependency.
- **Inherited dependencies.** After emitting, Deptrac copies every dependency of a class's ancestors onto the class as an "inherit dependency" ([Core/Dependency/DependencyResolver.php `flattenDependencies`](https://github.com/deptrac/deptrac/blob/4.7.2/src/Core/Dependency/DependencyResolver.php)).
- **Code outside classes.** The optional `file` type attributes references made outside classes to the file itself ([FileDependencyEmitter.php](https://github.com/deptrac/deptrac/blob/4.7.2/src/DefaultBehavior/Dependency/FileDependencyEmitter.php)). That is the one built-in way among these tools to see the `::class` references in `routes/web.php` or `config/*.php`.
- **PHPStan parser mode.** An optional, documented-as-experimental `phpstan_parser` flag parses with PHPStan's scope ([configuration.md L76–90](https://github.com/deptrac/deptrac/blob/4.7.2/docs/configuration.md); [PhpStanParser.php](https://github.com/deptrac/deptrac/blob/4.7.2/src/DefaultBehavior/Ast/Parser/PhpStanParser.php)). It improves how docblock and parameter types resolve. But there is still no method-call or property-fetch extractor, and `new $x` is still skipped ([NewExtractor.php](https://github.com/deptrac/deptrac/blob/4.7.2/src/DefaultBehavior/Ast/Extractors/NewExtractor.php)).
- **Cache.** Deptrac keeps a per-file reference cache, `.deptrac.cache`, keyed by `sha1_file`, and it is on by default ([AstFileReferenceFileCache.php](https://github.com/deptrac/deptrac/blob/4.7.2/src/Core/Ast/Parser/Cache/AstFileReferenceFileCache.php); [docs/index.md L155–156](https://github.com/deptrac/deptrac/blob/4.7.2/docs/index.md)). Emitting, flattening and layer resolution still run in full on every invocation ([CONTRIBUTING.md, "Deptrac Engine"](https://github.com/deptrac/deptrac/blob/4.7.2/docs/CONTRIBUTING.md)).

### 1.4 Roave/BetterReflection

BetterReflection is reflection without loading code: "if a class has not already been declared, you can safely assume that Better Reflection won't load the class" ([features.md L6](https://github.com/Roave/BetterReflection/blob/6.74.0/docs/features.md)).

It sees declarations only:

- parent, interfaces and traits;
- native parameter, return and property types;
- attributes, whose arguments are compiled to values, so `#[UsePolicy(PostPolicy::class)]` yields the class-name string ([ReflectionAttribute.php L96–107](https://github.com/Roave/BetterReflection/blob/6.74.0/src/Reflection/ReflectionAttribute.php#L96-L107));
- property default values, such as a provider's `$bindings` array ([ReflectionProperty.php L469](https://github.com/Roave/BetterReflection/blob/6.74.0/src/Reflection/ReflectionProperty.php#L469)).

What it leaves out:

- **Docblocks.** Docblock types are "Not used and not planned to be used in the future" ([features.md L16–18](https://github.com/Roave/BetterReflection/blob/6.74.0/docs/features.md)); docblock parsing was removed in 5.0 ([UPGRADE.md L10](https://github.com/Roave/BetterReflection/blob/6.74.0/UPGRADE.md)).
- **Method bodies.** No public method-body AST was found in `src/Reflection/` at 6.74.0. Body references (`new`, static calls, `::class` in expressions) need a separate parse of `getLocatedSource()`.
- **Persistent caching.** Caching is in memory only ([MemoizingSourceLocator.php](https://github.com/Roave/BetterReflection/blob/6.74.0/src/SourceLocator/Type/MemoizingSourceLocator.php)).

PHPStan uses a fork, `ondrejmirtes/better-reflection` ([phpstan-src composer.json](https://github.com/phpstan/phpstan-src/blob/2.3.1/composer.json)).

### 1.5 PHPStan and Larastan

**What PHPStan records.** In 2.3, each expression and statement handler returns the dependencies of the code it analysed: the classes in its inferred types, the class names it references, function and constant reflections, included files and used traits. These are kept so the result cache knows what to re-analyse ([Dependencies.php L14–25](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Dependency/Dependencies.php#L14-L25)). This replaced the old `DependencyResolver` walk: "Return dependencies in the results of the handlers (get rid of old DependencyResolver)" (#6662, [2.3.0 release notes](https://github.com/phpstan/phpstan/releases/tag/2.3.0)). It is type-based. Examples:

- A method call records the receiver type, the return type and the class that declares the method ([MethodCallHandler.php L395–404](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Analyser/ExprHandler/MethodCallHandler.php#L395-L404)).
- A static call records the called class and the declaring class ([StaticCallHandler.php L463–480](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Analyser/ExprHandler/StaticCallHandler.php#L463-L480)).
- `new $class` records the class when `$class` is a known constant class string ([NewHandler.php L331–333](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Analyser/ExprHandler/NewHandler.php#L331-L333), [DependencyTypes.php L93–109](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Dependency/DependencyTypes.php#L93-L109)).
- Attribute *arguments* are processed as expressions. The attribute class itself is not added in that handler ([AttributesHandler.php L46–76](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Analyser/AttributesHandler.php#L46-L76)).

**Why this data is not an edge list:**

- **Per file.** It is resolved to *files and Composer packages*, not classes ([DependencyResolver.php L110–197, L336–361](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Dependency/DependencyResolver.php#L110-L197)).
- **Over-inclusive.** Every referenced class is expanded to the class plus its parents, interfaces, traits, `@mixin`, `@property` and `@method` types, `@template` bounds, `@extends` and `@implements`, and imported type aliases ([DependencyResolver.php L236–334](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Dependency/DependencyResolver.php#L236-L334)). That is right for cache invalidation and wrong for architecture.
- **Internal.** Nothing in `src/Dependency/` carries `@api`.
- **No export command.** The old `dump-deps` command was removed in 1.0.0 ([1.0.0 release notes](https://github.com/phpstan/phpstan/releases/tag/1.0.0)).

**Supported ways in:**

- **Collectors** are `@api` ([Collector.php](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Collectors/Collector.php)). A Collector sees each node together with its `Scope`, so it can ask for inferred types. Collected data "should be arrays and scalar values", and "Collectors are seamlessly integrated with the result cache" ([phpstan.org/developing-extensions/collectors](https://phpstan.org/developing-extensions/collectors)).
- **`DependencyTracker`** is new in 2.3 and `@api` ([DependencyTracker.php](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Analyser/DependencyTracker.php); "Result cache: value and file dependencies declared on the scope", 2.3.0 release notes). It lets an extension declare `trackFileDependency()` (for example a config file), `trackDirectoryDependency()` (for example `app/Listeners`, `*.php`) and `trackClassDependency()`, the last for "a class named in a string, in a PHPDoc tag PHPStan does not resolve, or in a configuration file".

**Result cache.** It stores, per file, the inverted dependencies, a file hash and "exported nodes", meaning the signatures a file declares. When a file's body changes but its exported nodes don't, the files that depend on it are not re-analysed ([ResultCacheManager.php L756–866](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Analyser/ResultCache/ResultCacheManager.php#L756-L866)). The [result-cache docs](https://phpstan.org/user-guide/result-cache) list what invalidates everything (versions, configuration, the analysed path list) and say "If the list of paths differs from run to run, the cache is rebuilt from the ground up each time."

**Published cost** ([PHPStan 2.3 blog](https://phpstan.org/blog/phpstan-2-3-leap-in-performance)):

- **Full run:** WordPress takes 19.47 s without Turbo and 7.93 s with Turbo on 2.3.0; Drupal takes 22 s.
- **Warm cache on Drupal:**
  - no change: 1.58 s;
  - 10 files changed: 2.84 s;
  - 10% of files re-analysed: 6.21 s;
  - cache cleared (11,194 files): 22.08 s.
- **Memory:** no figures are published.

**Larastan.** Its README says it "boots your application's container, so it can resolve types that are only possible to compute at runtime" ([README.md L24](https://github.com/larastan/larastan/blob/v3.13.0/README.md)). Its `bootstrap.php` requires `bootstrap/app.php` and bootstraps the console kernel ([bootstrap.php](https://github.com/larastan/larastan/blob/v3.13.0/bootstrap.php)). What it adds:

- **Container lookups.** `app()`, `resolve()` and `make()` with a constant-string argument are typed by actually calling `$container->make($abstract)`, which instantiates the service ([AppMakeHelper.php L25–64](https://github.com/larastan/larastan/blob/v3.13.0/src/ReturnTypes/AppMakeHelper.php#L25-L64), [HasContainer.php L32–41](https://github.com/larastan/larastan/blob/v3.13.0/src/Concerns/HasContainer.php#L32-L41)).
- **Facades.** Facade methods resolve through `getFacadeRoot()` to the concrete class's native method ([FacadesMethodsExtension.php L38–75, L106–118](https://github.com/larastan/larastan/blob/v3.13.0/src/Methods/FacadesMethodsExtension.php#L38-L118)).
- **Relations.** Eloquent relation properties take their type from the relation method's `TResult` generic ([ModelRelationsExtension.php L85–108](https://github.com/larastan/larastan/blob/v3.13.0/src/Properties/ModelRelationsExtension.php#L85-L108)).

Larastan does not model routes, listeners, policies or observers as dependencies, and it doesn't use `DependencyTracker` yet (no match in `src/` at v3.13.0).

## 2. Reference kinds each extractor sees

"PHP-Parser + NR" means PHP-Parser with `NameResolver` plus a visitor you write, so its column shows what is *available*, not what a tool records. "No" means not seen at all. For Pest, every row is further subject to its autoload filter: names that don't resolve are dropped.

| Reference kind | PHP-Parser + NR | Pest arch extractor | Deptrac (nikic mode) | BetterReflection | PHPStan (+ Larastan) |
| --- | --- | --- | --- | --- | --- |
| `use` imports | Available; import names are already fully qualified | Yes, **including unused**; unused group-use items are dropped | Only with the `use` type, which the code doesn't enable by default; file-level | No | No; imports only feed name resolution ([L44–49](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Dependency/DependencyResolver.php#L44-L49)) |
| Native type hints (param, return, property, constant, hook; union, nullable, intersection) | Yes | Yes | Yes (`parameter`, `returntype`, `variable`) | Yes | Yes |
| `new Foo` | Yes; `new $x` no | Yes; `new $x` no | Yes; `new $x` no, even in PHPStan mode | No (body not exposed) | Yes, and `new $x` when `$x` is a constant class string |
| Static calls, static properties, class constants | Yes | Yes | Yes | No | Yes, plus the declaring class |
| extends / implements / traits | Yes | Yes | Yes, and **ancestors' dependencies copied into subclasses** | Yes | Yes, every referenced class expanded to its ancestors |
| Attributes | Yes: attribute name, and arguments as expressions | Yes: name and `::class` in arguments | Yes: `attribute`, and `::class` arguments as `const` | Yes: name, arguments compiled to values | Arguments yes; attribute class not added in that handler |
| Docblocks and generics | No (comments); `NameContext` available | **No** | Yes: `@param`, `@return`, `@throws`, `@var`, `@method`, `@property`, with generics | **No** (removed in 5.0) | Yes, fully resolved, including class-level `@mixin`, `@extends`, `@template` |
| `Foo::class` | Yes | Yes | Yes (`const`) | Only in constant expressions (attributes, defaults) | Yes |
| String class names (`'App\Models\Post'`) | No | No | No | No (just a string value) | Only where a type expects a class string, or for dynamic `new`/`instanceof`; extensions can add more with `trackClassDependency` |
| Typed receivers (`$this->repo->find()`, `$post->comments`) | No | No | No | No | Yes; with Larastan, magic Eloquent properties too |
| `instanceof`, `catch`, `throw` | Yes | Yes | Yes (`throw` via `@throws`) | No | Yes |
| Function calls (`app()`, `event()`) | Name left unqualified in a namespace | Yes, user-defined functions by name (`app`, `resolve`) | `function_call` type for user functions; others marked unresolved and skipped | No | Yes, with the return type (typed through the container by Larastan) |
| Code outside PSR-4 class files (routes, config, Blade, Livewire single-file components) | If you feed it those files | **No**: PSR-4 directories only, first class-like per file | Only with the `file` type, and only for PHP files in `paths` | If located | Only the paths it analyses; Blade not at all |
| Line numbers | Yes (node attributes) | Yes (found when reporting) | Yes (`FileOccurrence`) | Declaration lines only | No: file-level only |

## 3. Cost and incremental support

| Extractor | Work per run | Needs autoload or boot | Incremental rescans | Output shape | API stability |
| --- | --- | --- | --- | --- | --- |
| PHP-Parser + NR | One parse and traverse per file | Neither | None built in; per-file content-hash caching is sound because output depends only on the file | Whatever you emit | Stable public library |
| Pest arch extractor | Parses every file under each requested namespace, autoloads every referenced name, and loads every scanned class through `ReflectionClass` | Composer autoload, and it **executes** each scanned file | **None.** In-memory per process; overlapping namespace requests re-parse. That each `--parallel` worker builds its own cache is an inference from the per-process singleton | `uses` list per class | `@internal` in pest-plugin-arch |
| Deptrac | Parse plus phpdoc parse per file, then emit, flatten and match layers | Neither in nikic mode; PHPStan container in PHPStan mode | **Yes for parsing** (`.deptrac.cache`, sha1 per file, on by default); analysis recomputed every run | Typed dependencies with file and line | Public extension points (extractors, emitters, collectors, events) |
| BetterReflection | Parses a file on demand per reflected symbol | Neither ("won't load the class") | None; memoised in memory | Declarations only | Stable public library |
| PHPStan + Larastan | Full type inference over all analysed files. Published: WordPress 7.93–19.47 s, Drupal 22 s; Larastan adds booting the app | PHPStan: static reflection. Larastan: **boots the app** and instantiates services it resolves | **Best in class:** a dependency-aware result cache that skips dependents when signatures don't change (Drupal warm runs 1.58–6.21 s for small changes); Collector data rides on it | File→file internally. Class edges only through a custom Collector | Collectors and `DependencyTracker` are `@api`; the dependency data is internal |

Cost ordering, as an inference not measured here: PHPStan parses with the same nikic parser and then infers types. So a parse-only extractor does a strict subset of PHPStan's per-file work, and PHPStan's full-run times are a loose upper bound for it. Turbo, a C++ extension that "shadows majority of the analyser engine" (2.3.0 release notes), makes that bound looser. Pest adds autoloading and executing every scanned file on top of parsing.

## 4. Where the extractors disagree

These are the differences that would make a picture drawn by one extractor disagree with a Pest plugin built on another:

1. **Unused imports.** Pest counts them. Deptrac's code defaults don't. A hand-written PHP-Parser visitor does whichever you choose.
2. **Names that don't resolve.** Pest keeps only names that autoload, so a deleted or misspelled class, an unregistered alias (`\Cache`) or a real-time facade without the alias loader all drop out of the edge list. A pure syntactic extractor keeps them.
3. **Docblock-only references** (`@return Collection<int, Post>`, `@var Order $o`). Deptrac and PHPStan count them. Pest doesn't.
4. **Inheritance.** Deptrac copies a parent's dependencies onto every subclass. Pest checks each class's own names. PHPStan expands to ancestors in the other direction: a reference to a class also depends on that class's ancestors.
5. **Which files are in scope.** Pest reads only PSR-4 directories and the first class-like per file. Deptrac reads its configured `paths`. PHPStan reads the analysed paths.
6. **Inferred dependencies.** Only PHPStan sees `$post->comments` → `Comment`, or `app('payments')` → `StripeGateway`, the latter through Larastan's booted container.

## 5. Laravel blind spots

A "name-based extractor" below means PHP-Parser + NR, Pest or Deptrac. "Static recovery" means parsing PHP or config without booting. "Boot" means loading the app (Larastan's approach, or Artisan's introspection commands).

| # | Blind spot | What name-based extractors see | Static recovery | Boot recovery | Class |
| --- | --- | --- | --- | --- | --- |
| 1 | Facades | An edge to the facade class, not its root | `getFacadeAccessor()` return (a `::class` or a container key), then resolve the key through bindings | `Facade::getFacadeRoot()` (Larastan) | A |
| 1b | Real-time facades and root aliases | Pest: dropped unless the alias loader is registered. Others: a non-existent `Facades\...` or `Cache` name | Yes: strip `Facades\`; aliases from `Facade::defaultAliases()` plus `config/app.php` | Yes | B |
| 2 | `app()`, `resolve()`, `make()` | `app(Foo::class)`: an edge to `Foo`, the abstract. `app('payments')`: nothing | Constant keys via parsed bindings and core aliases | `make()` (instantiates; Larastan) | A / B / D |
| 3 | Class names in config | Nothing; config files aren't classes. Consumers read string keys | Parse `config/*.php` arrays; `env()` defaults are visible | `config()->all()`, `config:show` (environment-dependent) | B / C |
| 4 | Auto-discovered listeners | Listener → event (type hint). Dispatcher → event. Not event → listener | Replay `DiscoverEvents` with the AST | `event:list --json`, `getRawListeners()` | B / C |
| 5 | Policies | `Gate::policy()` calls and `#[UsePolicy]`; the naming guess and call sites are invisible | Replay `guessPolicyName` | `Gate::getPolicyFor()`, `model:show --json` | B / C |
| 6 | Observers | `#[ObservedBy]` and `observe()` calls visible, with the model→observer direction | Parse the attribute and calls | `getRawListeners()` with `eloquent.*` keys | A / B |
| 7 | Blade components | Nothing; Blade is not PHP | Replay `guessClassName`, or compile views and parse the output | `view:cache`, then parse compiled PHP | B / D |
| 8 | Livewire components | Class components are classes, but references to them sit in Blade. Single-file components are **not seen at all** | Parse the PHP block of each component file | The Livewire compiler's cached classes | B / D |
| 9 | Routes → controllers | Nothing; route files aren't classes (Deptrac's `file` type can see them) | Parse `Route::` calls with `::class` | `route:list --json` | B / C |
| 10 | Eloquent relationships | `hasMany(Comment::class)` visible. String targets and callers' `$post->comments` invisible | String targets via a constant class string | `ModelInspector` invokes the methods; Larastan types | B / C |
| 11 | Morph maps | `morphMap()` entries visible in providers; `morphTo()` targets unknown | The set of possible targets | `Relation::morphMap()`; the actual target is data | D |
| 12 | Container bindings | Provider → abstract and provider → concrete; consumer → concrete missing | Constant-argument `bind()`/`singleton()`, `$bindings`, `#[Bind]` | `getBindings()` returns closures; `make()` instantiates | A / B / C |

### 5.1 Facades

```php
use Illuminate\Support\Facades\Cache;           // edge: → Illuminate\Support\Facades\Cache
class ReportService {
    public function totals() { return Cache::remember('totals', 60, fn () => /* ... */); }
}
```

- **What extractors see.** Name-based extractors see the facade class only. The facade names its root by a container key: `Cache::getFacadeAccessor()` returns `'cache'` ([Facades/Cache.php L84–87](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/Support/Facades/Cache.php#L84-L87)), which is resolved at runtime through `static::$app[$name]` ([Facade.php L209–245](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/Support/Facades/Facade.php#L209-L245)).
- **Why it matters.** For framework facades the edge to the facade is probably what an architecture spec wants anyway. App-defined facades that front domain services hide the real target.
- **Static recovery.**
  - Read `getFacadeAccessor()`'s return value.
  - Map core keys with `registerCoreContainerAliases()`, for example `'cache' => [CacheManager::class, Factory::class]` ([Application.php L1644–1662](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/Foundation/Application.php#L1644-L1662)).
  - Map app keys through the parsed bindings (§5.12).
- **Boot recovery.** `getFacadeRoot()`, as Larastan does ([FacadesMethodsExtension.php L106–118](https://github.com/larastan/larastan/blob/v3.13.0/src/Methods/FacadesMethodsExtension.php#L106-L118)). This instantiates the root.

### 5.1b Real-time facades and root aliases

```php
use Facades\App\Billing\Invoicer;  // real-time facade for App\Billing\Invoicer
\Cache::flush();                    // root alias, no import
```

- **Real-time facades.** These are generated by `AliasLoader`. Any class under the `Facades\` prefix is created on demand in `storage/framework/cache/facade-<sha1>.php` by an autoloader prepended to the stack ([AliasLoader.php L26, L70–103, L174–176](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/Foundation/AliasLoader.php#L70-L103)).
- **Root aliases.** These come from `Facade::defaultAliases()` ([Facade.php L273](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/Support/Facades/Facade.php#L273)) merged with `config/app.php` `aliases` ([config/app.php L186](https://github.com/laravel/framework/blob/v13.37.0/config/app.php#L186)).
- **What extractors see.** Pest drops both kinds of name unless the alias loader is registered in the test process (§1.2).
- **Static recovery.** Fully deterministic: strip the `Facades\` prefix, and look aliases up in the default list plus the app's config.

### 5.2 `app()`, `resolve()` and container lookups by string

```php
$gateway = app(PaymentGateway::class);  // seen as → PaymentGateway (the abstract)
$mailer  = resolve('mailer');           // string key: no class edge
$report  = app("reports.{$type}");      // computed: unrecoverable
```

`app()` is `Container::getInstance()->make($abstract)` and `resolve()` calls `app()` ([foundation helpers.php L132–139, L822–825](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/Foundation/helpers.php#L132-L139)). Each form is handled differently:

- **`::class` argument.** It is a `ClassConstFetch`, so name-based extractors record an edge to the abstract. That is the same edge a constructor type hint would give, so it is "seen", but the concrete bound behind it is not.
- **Constant-string key** (`'mailer'`). Recoverable through core aliases plus parsed bindings.
- **Computed key.** Class D.

Pest records the function itself (`app`, `resolve`), which is why a ban on those functions is expressible (§1.2). Larastan types constant keys by calling `make()` on the booted container (§1.5).

### 5.3 Class names in config files

```php
// config/auth.php (framework default)
'providers' => ['users' => ['driver' => 'eloquent', 'model' => env('AUTH_MODEL', App\Models\User::class)]],
// consumer, somewhere in app/
$model = config('auth.providers.users.model');
```

- **The example is real.** It is the framework's own default ([config/auth.php L65](https://github.com/laravel/framework/blob/v13.37.0/config/auth.php#L65)). Providers and aliases also live in config ([config/app.php L166, L186](https://github.com/laravel/framework/blob/v13.37.0/config/app.php#L166-L186)), and since Laravel 11 providers are also listed in `bootstrap/providers.php` and registered via `withProviders()` ([ApplicationBuilder.php L82](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/Foundation/Configuration/ApplicationBuilder.php#L82)).
- **What extractors see.** The `::class` in a config file is visible syntax, but no class owns it, and the consumer holds only a string key.
- **Static recovery.** Parse `config/*.php`, which returns arrays, into a key → class map, then resolve `config('...')` calls that have constant keys. `env()` makes the value environment-dependent, but its default is visible.
- **Boot recovery.** `config:show` ([ConfigShowCommand.php](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/Foundation/Console/ConfigShowCommand.php)) or the config repository. Both give the answer for *one* environment.

### 5.4 Auto-discovered events and listeners

```php
// app/Listeners/SendShipmentNotification.php: no registration anywhere
class SendShipmentNotification {
    public function handle(OrderShipped $event): void { /* ... */ }   // seen: Listener → OrderShipped
}
// app/Http/Controllers/OrderController.php
OrderShipped::dispatch($order);                                       // seen: Controller → OrderShipped
```

- **How discovery works.** Laravel's own docs: "By default, Laravel will automatically find and register your event listeners by scanning your application's `Listeners` directory", for methods beginning with `handle` or `__invoke`, with union types for several events ([laravel.com/docs/13.x/events#event-discovery](https://laravel.com/docs/13.x/events#event-discovery)). The implementation reflects each class in the discovery paths and maps `handle*`/`__invoke` methods to the class type of their first parameter. It honours `ShouldBeDiscovered::shouldBeDiscovered()` and an overridable class-name guess ([DiscoverEvents.php L32–131](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/Foundation/Events/DiscoverEvents.php#L32-L131)).
- **Where else registrations live.**
  - The default path is `app/Listeners`, and `withEvents(discover: [...])` changes it ([EventServiceProvider.php L166–171](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/Foundation/Support/Providers/EventServiceProvider.php#L166-L171), [ApplicationBuilder.php L100–106](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/Foundation/Configuration/ApplicationBuilder.php#L100-L106)).
  - Explicit `$listen`, `$subscribe` and `$observers` arrays are merged in ([EventServiceProvider.php L20–73](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/Foundation/Support/Providers/EventServiceProvider.php#L20-L73)).
  - In production the whole result can come from `bootstrap/cache/events.php` instead ([L105–117](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/Foundation/Support/Providers/EventServiceProvider.php#L105-L117)).
- **What's missing.** Both class edges are syntactic. What is missing is *event → listener*: what runs when an event fires. That is a runtime wiring, not a compile-time dependency of the dispatcher; whether the architecture spec cares is [#10](https://github.com/kfriars/laravel-visual-spec/issues/10)'s call.
- **Static recovery.** Replaying discovery with the AST is deterministic: public methods named `handle*` or `__invoke`, first parameter's type, union types, in the configured paths. It breaks only where discovery depends on runtime code: `shouldBeDiscovered()`, `guessClassNamesUsing()` callbacks, and `Event::listen()` calls with closures or computed names.
- **Boot recovery.** `event:list --json` ([EventListCommand.php L21–23](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/Foundation/Console/EventListCommand.php#L21-L23)).

### 5.5 Policies

```php
// No registration. Laravel guesses App\Policies\PostPolicy for App\Models\Post.
class PostController {
    public function update(Request $r, Post $post) { $this->authorize('update', $post); /* edge to PostPolicy: invisible */ }
}
```

- **How `Gate::getPolicyFor()` resolves a policy**, in order ([Gate.php L653–690](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/Auth/Access/Gate.php#L653-L690)):
  1. Explicit `Gate::policy()` registrations.
  2. The `#[UsePolicy(PostPolicy::class)]` attribute on the model ([L699–716](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/Auth/Access/Gate.php#L699-L716); [UsePolicy.php](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/Database/Eloquent/Attributes/UsePolicy.php)).
  3. A naming guess that tries `…\Policies\{Model}Policy` at each namespace depth, plus the `Models` → `Policies` swaps, keeping the first that `class_exists`. The guess can be replaced with `guessPolicyNamesUsing()` ([L724–755](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/Auth/Access/Gate.php#L724-L755)).
  4. A subclass match.
  5. A parent's attribute.
- **What extractors see.** Call sites name the ability as a string (`'update'`) and the model as an object or `::class`. The model → policy relation is invisible to every extractor.
- **Static recovery.** Replay the guess against the set of declared classes, plus parse `Gate::policy()` calls and `#[UsePolicy]`. This breaks on a custom guess callback.
- **Boot recovery.** `Gate::getPolicyFor()`, which `model:show` uses ([ModelInspector.php L210–215](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/Database/Eloquent/ModelInspector.php#L210-L215)).

### 5.6 Observers

```php
#[ObservedBy([UserObserver::class])]                // seen: User → UserObserver (inverted)
class User extends Model {}
class UserObserver { public function created(User $user): void {} }   // seen: UserObserver → User
// or, in a provider: User::observe(UserObserver::class);             // seen: Provider → both
```

- **What extractors see.** Registration is mostly visible: the `ObservedBy` attribute ([ObservedBy.php](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/Database/Eloquent/Attributes/ObservedBy.php)), `observe()` calls, or the `$observers` array on the event provider ([EventServiceProvider.php L34, L70–72](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/Foundation/Support/Providers/EventServiceProvider.php#L34-L72)).
- **The trap is direction.** `#[ObservedBy]` makes the *model* reference the observer. A rule such as "Models may depend only on Domain" would flag a wiring annotation as a dependency. The same happens with `#[UsePolicy]`, `#[UseFactory]`, `#[CollectedBy]`, `#[UseEloquentBuilder]` and the container's `#[Bind]`, all of which exist at v13.37.0 ([Eloquent attributes](https://github.com/laravel/framework/tree/v13.37.0/src/Illuminate/Database/Eloquent/Attributes), [Container attributes](https://github.com/laravel/framework/tree/v13.37.0/src/Illuminate/Container/Attributes)). Treating them is a semantic question for [#10](https://github.com/kfriars/laravel-visual-spec/issues/10). Extracting them is easy.
- **Boot recovery.** Observers are listeners keyed `eloquent.{event}: {Model}` ([ModelInspector.php L240–247](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/Database/Eloquent/ModelInspector.php#L240-L247)).

### 5.7 Blade components and views

```blade
{{-- resources/views/orders/show.blade.php --}}
<x-order-summary :order="$order" />          {{-- → App\View\Components\OrderSummary --}}
<x-dynamic-component :component="$widget" /> {{-- unknowable --}}
@inject('metrics', App\Services\Metrics::class)
```

```php
return view('orders.show', compact('order'));   // controller → template, by string
```

- **How a tag resolves** ([ComponentTagCompiler.php L276–318, L424–433](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/View/Compilers/ComponentTagCompiler.php#L276-L433)):
  1. Registered aliases.
  2. Registered component namespaces.
  3. The guess `{AppNamespace}View\Components\{Studly}` and its nested-index variant.
  4. Anonymous component views.
- **What extractors see.** Nothing: no PHP extractor reads Blade.
- **Static recovery.** Two routes:
  - *Replay the resolution rules.* Aliases and namespaces come from `Blade::component()` / `Blade::componentNamespace()` calls in providers.
  - *Compile and parse.* Compiled output contains the class as a string and as a static call, `\App\View\Components\OrderSummary::ignoredParameterNames()` ([ComponentTagCompiler.php L235–265](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/View/Compilers/ComponentTagCompiler.php#L235-L265)). `@inject` compiles to `app(...)` ([CompilesInjections.php](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/View/Compilers/Concerns/CompilesInjections.php)). So running `view:cache` (which boots the app) and feeding the compiled PHP to the same name-based extractor recovers component and `@inject` edges. They would be attributed to a compiled file, not a class.
- **Unrecoverable.** Dynamic components, and `view($computed)`.
- **Open question.** Views are not classes, so whether template edges belong in a class-level architecture spec at all is open.

### 5.8 Livewire components

```blade
{{-- resources/views/components/⚡order-table.blade.php (Livewire 4 single-file component) --}}
<?php
use Livewire\Component;
use App\Models\Order;
new class extends Component { public function orders() { return Order::latest()->get(); } };
?>
<div>...</div>
```

- **The default layout.** Livewire 4 looks for components in `resources/views/components` and `resources/views/livewire`, and for class components under `App\Livewire` ([config/livewire.php L16–19, L92](https://github.com/livewire/livewire/blob/v4.4.7/config/livewire.php#L16-L92)). Its finder resolves single-file (`⚡name.blade.php`), multi-file and class components ([Finder.php L127–224](https://github.com/livewire/livewire/blob/v4.4.7/src/Finder/Finder.php#L127-L224)).
- **The blind spot.** The PHP of a single-file component is an **anonymous class inside a Blade file** ([Finder fixtures](https://github.com/livewire/livewire/tree/v4.4.7/src/Finder/Fixtures)). It is outside PSR-4 and has no name, so **no** extractor in this study sees the component or what it depends on. That is a class with dependencies missing from the picture entirely, not a missing edge. The multi-file fixture's `.php` file declares an un-namespaced class beside its view.
- **How components are referenced.** From Blade (`<livewire:order-table />`, `@livewire('order-table')`), and from routes via the `Route::livewire()` macro ([HandleRouting.php L16](https://github.com/livewire/livewire/blob/v4.4.7/src/Mechanisms/HandleRouting/HandleRouting.php#L16)).
- **Static recovery.** The `<?php … ?>` block is ordinary PHP that PHP-Parser can parse, and the component's name follows from its path under the configured locations.
- **Boot recovery.** Livewire's compiler writes the compiled classes to `{view.compiled}/livewire/classes/<hash>.php` ([CacheManager.php L73](https://github.com/livewire/livewire/blob/v4.4.7/src/Compiler/CacheManager.php#L73), [LivewireServiceProvider.php L43–47](https://github.com/livewire/livewire/blob/v4.4.7/src/LivewireServiceProvider.php#L43-L47)).

### 5.9 Routes that reference controllers

```php
// routes/web.php
Route::get('/orders/{order}', [OrderController::class, 'show']);
Route::resource('photos', PhotoController::class);
Route::post('/checkout', ProcessCheckout::class)->middleware('auth');   // 'auth' → an alias class
Route::livewire('/dashboard', 'pages::dashboard');
```

- **What extractors see.** Route files are not classes and not PSR-4, so Pest never reads them. Deptrac's `file` type would attribute the `::class` references to the file. Without route edges, every controller looks unused, and nothing records which middleware guards what.
- **Where the wiring lives.** Routing files are registered by `withRouting()` and middleware aliases by `withMiddleware()` ([ApplicationBuilder.php L155, L287](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/Foundation/Configuration/ApplicationBuilder.php#L155-L287); [Middleware.php L398, L803](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/Foundation/Configuration/Middleware.php#L398-L803)).
- **Static recovery.** Parsing `Route::` calls with `::class` arguments covers the common forms. It degrades on string actions with group namespaces, macros, and routes registered by packages.
- **Boot recovery.** `route:list --json` gives every route's resolved `action` and middleware ([RouteListCommand.php L28–29, L163–164](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/Foundation/Console/RouteListCommand.php#L28-L164)).

### 5.10 Eloquent relationship methods

```php
class Post extends Model {
    public function comments(): HasMany { return $this->hasMany(Comment::class); }   // seen: Post → Comment
    public function author() { return $this->belongsTo('App\Models\User'); }         // string: not seen
}
// elsewhere
foreach ($post->comments as $c) { $c->author->notify(/* ... */); }  // caller → Comment, User: not seen
Post::with('comments.author')->get();                                   // relation names as strings
```

- **What extractors see.** Declarations written with `::class` are visible to every name-based extractor. Two things are not: string targets, and *callers* reaching related models through magic properties or `with()` strings.
- **PHPStan with Larastan** recovers both. It types relation methods through stubs that take a class string, and types properties from `TResult` (§1.5).
- **Boot recovery.** Laravel's `ModelInspector` finds relation methods by reflection plus a source-text heuristic, then **invokes** each one to read `getRelated()` ([ModelInspector.php L160–202](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/Database/Eloquent/ModelInspector.php#L160-L202)), exposed through `model:show --json` ([ShowModelCommand.php L33–35](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/Database/Console/ShowModelCommand.php#L33-L35)).
- **Factories follow the same pattern.** They are resolved by convention, `Database\Factories\{Model}Factory` ([Factory.php L1086–1099](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/Database/Eloquent/Factories/Factory.php#L1086-L1099)), unless `#[UseFactory]` is used.

### 5.11 Morph maps

```php
// AppServiceProvider::boot()
Relation::enforceMorphMap(['post' => Post::class, 'video' => Video::class]);   // seen: Provider → Post, Video
class Comment extends Model { public function commentable() { return $this->morphTo(); } }  // target: data
```

- **Why the target is unknowable.** `morphTo()` reads the type column from the row it loads ([HasRelationships.php L416–432](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/Database/Eloquent/Concerns/HasRelationships.php#L416-L432)) and maps it through the morph map, or uses it as a raw class name when no map applies ([HasRelationships.php L499–505](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/Database/Eloquent/Concerns/HasRelationships.php#L499-L505); [Relation.php L542–592](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/Database/Eloquent/Relations/Relation.php#L542-L592)).
- **Static recovery.** The *possible* targets can be gathered from `morphMap()`/`enforceMorphMap()` calls and from the inverse `morphMany(Comment::class, 'commentable')` declarations.
- **Boot recovery.** `Relation::morphMap()` gives the map.
- **Unrecoverable.** Which target a given call actually reaches is data. The honest rendering is "one of {Post, Video}", or "unknown" when no map is enforced.

### 5.12 Container bindings in service providers

```php
class PaymentServiceProvider extends ServiceProvider {
    public $bindings = [PaymentGateway::class => StripeGateway::class];                  // seen, as constants
    public function register(): void {
        $this->app->singleton(Ledger::class, fn ($app) => new DbLedger($app['db']));    // seen: new DbLedger
        $this->app->when(PhotoController::class)->needs(Filesystem::class)
                  ->give(fn () => Storage::disk('photos'));                              // contextual
        if ($this->app->environment('local')) { $this->app->bind(Mailer::class, LogMailer::class); }
    }
}
#[Bind(StripeGateway::class)] interface PaymentGateway {}   // seen: interface → implementation (inverted)
```

- **What extractors see.** Name-based extractors see provider → abstract and provider → concrete. What the picture needs is *consumer → abstract → bound concrete*: "the implementation behind each contract".
- **Where bindings come from:**
  - provider `$bindings` and `$singletons` properties ([Application.php L906–916](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/Foundation/Application.php#L906-L916));
  - `bind()` calls, which wrap a class-name concrete in a closure ([Container.php L359–415](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/Container/Container.php#L359-L415));
  - contextual `when()` ([L211](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/Container/Container.php#L211));
  - `#[Bind]`, `#[Singleton]` and `#[Scoped]` attributes read lazily on first resolution, with an optional environment list ([Container.php L1000–1029](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/Container/Container.php#L1000-L1029); [Bind.php](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/Container/Attributes/Bind.php));
  - deferred providers, whose bindings exist only once something asks for them ([ProviderRepository.php L144–148](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/Foundation/ProviderRepository.php#L144-L148)).
- **Static recovery.** Works well for the constant forms:
  - `$bindings` and `$singletons` defaults, which BetterReflection can evaluate (§1.4);
  - `bind()`, `singleton()` and `scoped()` with `::class` arguments;
  - `#[Bind]`;
  - closures whose body is a single `new X`.

  It must label environment-conditional and closure-built bindings as conditional or unknown.
- **Booting is weaker than it looks:**
  - `getBindings()` returns closures, not class names ([Container.php L1670–1673](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/Container/Container.php#L1670-L1673)), so reading the concrete means resolving it, which instantiates it with side effects. That is what Larastan does.
  - Attribute bindings and deferred providers don't appear until resolution.
  - The result is for one environment.
  - No first-party command lists container bindings: none was found in `src/Illuminate/Foundation/Console/` at v13.37.0.

## 6. Static parsing versus booting

| | Static (parse PHP and config, replay conventions) | Boot (load the app, ask it) |
| --- | --- | --- |
| Determinism | Same code gives the same result | Depends on `.env`, the environment, packages and database state |
| Side effects | None | Runs providers' `register()`/`boot()`; `make()` instantiates services |
| CI requirements | PHP only | A bootable app: config, extensions, sometimes services |
| Covers | Classes A and B | A, B and most of C |
| Fails on | Closures, custom callbacks, `env()`-conditional wiring, macros | Data-driven targets (class D); deferred and attribute bindings until resolved |
| Prior art | Deptrac (`file` type), PHP-Parser; Laravel Brain (sibling [#5](https://github.com/kfriars/laravel-visual-spec/issues/5)) | Larastan; Laravel's own `route:list`, `event:list`, `model:show`, `config:show` |
| Incremental | Per-file, plus re-reading providers, config and routes, which are few | A whole boot per run |

## 7. Inputs for the open decisions (not decisions)

- **For [#10](https://github.com/kfriars/laravel-visual-spec/issues/10), what counts as a dependency.** The candidate reference kinds have clear support. The contentious ones are:
  - unused imports;
  - docblock-only references;
  - references to names that don't exist;
  - inherited dependencies;
  - wiring attributes that point backwards (`#[ObservedBy]`, `#[UsePolicy]`, `#[Bind]`);
  - runtime fan-out (event → listener, model → observer, model → policy).

  Each needs an explicit yes or no, because the tools above answer them differently.
- **For the picture and the plugin to agree.** If the Pest plugin builds on Pest's arch extractor, the picture inherits its specific choices (§4, items 1, 2 and 5). A single extractor library shared by both would avoid the drift. A file-local, PHP-Parser-based extractor is deterministic and cacheable per file, and it can reproduce Pest's existence filter with a symbol table built from the same parse plus Composer's class map. Whether the plugin can use such a library is [#11](https://github.com/kfriars/laravel-visual-spec/issues/11)'s call.
- **For the Laravel container-knowledge fog.** A realistic split has three layers:
  - syntactic edges, for every class;
  - "wiring" edges recovered statically from providers, config, routes, attributes and conventions, each labelled with how it was found;
  - "unknown" markers for class C and D sites.

  Booting can then be an optional enrichment rather than a requirement. PHPStan with Larastan is the only option that sees typed receivers. Its dependency output, though, is internal, per file and over-inclusive, so using it means writing a Collector and paying for a full analysis plus a boot.

## Open questions surfaced

1. Do unused imports, docblock-only references and references to non-existent classes count as dependencies? Pest and Deptrac's code defaults disagree on the first two, and Pest drops the third.
2. Do wiring attributes that point "backwards" (`#[ObservedBy]`, `#[UsePolicy]`, `#[UseFactory]`, `#[Bind]` on an interface) count as dependencies of the class carrying them, or as a separate wiring relation?
3. Is runtime fan-out (event → listener, model → observer, model → policy, abstract → bound concrete) part of the architecture spec's dependency model, or drawn as a separate, labelled relation?
4. Should the Pest plugin and this package share one extractor library, or should the plugin keep Pest's built-in arch extractor, with this package matching its quirks? (Feeds [#11](https://github.com/kfriars/laravel-visual-spec/issues/11).)
5. Are non-class code units in scope as nodes: route files, config, Blade views and Livewire single-file components? Livewire 4 single-file components are anonymous classes in Blade files, and no studied extractor sees them at all.
6. Is booting the app ever required, optional enrichment only, or never? And if it is ever used, for which environment, given that bindings, config and `#[Bind(environments: …)]` vary?
7. How is a "possible but not certain" edge represented: morph targets, environment-conditional bindings, policy guesses that depend on a callback? Unknown, one-of-a-set, or conditional?
8. Is inferred-type extraction through PHPStan (with Larastan) worth its cost and its coupling to PHPStan versions, given it is the only way to see `$post->comments`-style dependencies? Or are those deliberately out of scope?
9. What is the real cost of each extractor on the reference app set from [#3](https://github.com/kfriars/laravel-visual-spec/issues/3)? No code was run for this ticket. Only PHPStan publishes figures.
