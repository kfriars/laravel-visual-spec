# What PHPStan with Larastan infers, for extracting dependencies

Research done 2026-10-18 against the releases current that day. The owner has decided that a dependency between two types is defined by static analysis with type inference: PHPStan with Larastan, read through a custom Collector. This note checks, from source, what that inference actually yields in a Laravel app. It is input to "What counts as a dependency?" ([#10](https://github.com/kfriars/laravel-visual-spec/issues/10)) and "How does the architecture spec treat Laravel's wiring?" ([#16](https://github.com/kfriars/laravel-visual-spec/issues/16)). It decides nothing.

It builds on [static-extraction §1.5 and §5](https://github.com/kfriars/laravel-visual-spec/blob/research/static-extraction/docs/research/static-extraction.md) (branch `research/static-extraction`), and doesn't repeat them: PHPStan's internal dependency data, the result-cache basics, and the Laravel blind spots.

**Inferred** marks a claim derived from reading code or docs rather than stated by them or shown by a test. No code was run for this note. Cost figures come from PHPStan's own blog and from public CI logs.

## Answer in brief

1. **Container lookups give the concrete class, not the abstract.**
   - **Constant keys.** With a constant-string argument, Larastan calls `make()` on the booted container and types the expression as the class of the object that comes back. In Larastan's own tests, `$app->make(\Illuminate\Contracts\Config\Repository::class)` is `Illuminate\Config\Repository`.
   - **Same for every form:** `app()`, `resolve()`, `App::make()`, `app()->make()` and `$this->app->make()`, and also `$app['key']`. String keys resolve the same way: `app('auth')` is `AuthManager`.
   - **When resolution fails.** If it throws, a class-string argument falls back to the requested class, and any other string becomes `*ERROR*`, which holds no class. A non-constant argument is `mixed`, even when it is a `class-string<Foo>`.
   - **Side effects.** Resolving instantiates the service.
   - **The abstract is still visible**, but only as the `Foo::class` argument node, not in the call's type.
2. **Facades and aliases.**
   - **What `Cache::get()` names.** Three classes are involved:
     - the facade `Illuminate\Support\Facades\Cache`, as the call's class;
     - `Illuminate\Cache\Repository`, the method's declaring class, reached through a `@mixin` that Larastan's stub puts on the facade;
     - none in the return type, which is `mixed`.

     No inferred type holds the runtime root.
   - **How facades are typed.** Framework facades are typed from PHPDoc: `@method` tags, or Larastan's `@mixin` stubs. Only a facade with neither is typed by calling `getFacadeRoot()`.
   - **Root aliases and real-time facades resolve to real classes.** Larastan's boot registers Laravel's `AliasLoader`, and PHPStan runs autoloaders that bootstrap files registered whenever it can't find a class. `\Cache` reflects as `Illuminate\Support\Facades\Cache`.
   - **But this path is fragile.** It broke twice in 2026: in PHPStan 2.1.34–2.1.39, and in 2.2.9. Larastan's maintainer declined to support real-time facades (in 2021).
3. **Traits are analysed only inside each analysed class that uses them.**
   - **How often.** Once per using class, and not at all if no analysed class uses the trait.
   - **Telling them apart.** Inside trait code:
     - `$scope->getClassReflection()` is the using class;
     - `$scope->getTraitReflection()` is the trait;
     - `$scope->getFile()` is the using class's file.
   - **Types can differ per using class.**
4. **Unknown classes.**
   - **When PHPStan reports them.** Level 0 reports unknown classes in code and in native types. Unknown classes in PHPDoc types are reported from level 2.
   - **A Collector can still see them.** An unknown name stays in the inferred type (`new Missing()` is `Missing`), so a Collector can test every name with `ReflectionProvider::hasClass()` at any level.
   - **What counts as declared.** All of these:
     - analysed paths;
     - `scanFiles` and `scanDirectories`;
     - Composer `autoload` and `autoload-dev` of the root package, plus `autoload` of every installed package;
     - PHP's own symbols, from phpstorm-stubs;
     - anything the bootstrapped runtime can load, which under Larastan means the booted app.

     Stub files declare nothing.
   - **IDE Helper mixin files.** Users add `_ide_helper_models.php` to `scanFiles`, or write the PHPDoc onto the models instead.
5. **PHP built-ins come from JetBrains' phpstorm-stubs, bundled with PHPStan, whether or not the extension is loaded.** `Redis`, `Imagick` and `ZipArchive` are all in the stub map. The result is the same on every machine for a given PHPStan version and PHP version, with three exceptions:
   - symbols from core extensions follow the PHP version;
   - `couchbase` and `ds` follow the installed extension's major version;
   - internal classes missing from the stubs are found only where their extension is loaded.
6. **Collectors run at every level, but some configuration does change inferred types.** Levels only register PHPStan's own rules and collectors, and set rule flags.
   - **PHPStan settings that change types:**
     - `phpVersion`;
     - the discovery settings;
     - `stubFiles`;
     - `inferPrivatePropertyTypeFromConstructor`;
     - `treatPhpDocTypesAsCertain`, through which code counts as reachable;
     - the scope-pollution settings and `earlyTerminating*Calls`;
     - `universalObjectCratesClasses`;
     - some bleeding-edge toggles.
   - **Larastan settings that change types:**
     - the migration and schema settings;
     - `parseModelCastsMethod`;
     - `checkConfigTypes`;
     - `generalizeEnvReturnType`.
   - **Model properties need no database.** They come from parsing migration files and schema dumps, plus a model instance created without calling its constructor.
7. **Larastan boots the app through the console kernel, in the main process and in every worker.**
   - **What it needs:**
     - `bootstrap/app.php`;
     - an installed `vendor/`;
     - providers whose `register()` and `boot()` don't throw. Any exception there stops the analysis with exit code 1.

     A `.env` file is optional.
   - **Which environment.** `APP_ENV` from the process, else from `.env`, else `production`.
   - **Forcing one.** Setting `APP_ENV=testing` in the process forces it, and loads `.env.testing` if present, unless the config is cached.
   - **In CI.** Larastan's own CI analyses Monica, a real app, with no `.env` and no database or Redis service.
8. **The Collector API is as expected.**
   - **Types:** `$scope->getType($expr)` gives an expression's type.
   - **Class names:** `Type::getReferencedClasses()` gives every class in a type, including generic arguments, unions and callables. `getObjectClassNames()` gives only the direct object types.
   - **Class strings.** A constant class string such as `Foo::class` references no class. It needs `getConstantStrings()` plus `isClassString()`.
   - **Receivers.** Use `$scope->getType($node->var)`, or `resolveName()` for a static call. Then `getMethodReflection()->getDeclaringClass()` gives the declaring class.
   - **Relation properties.** `$post->comments` is typed from the relation method's `TResult` generic. So it is `Collection<int, Comment>` only if the method has a native `Relation` return type and `@return HasMany<Comment, $this>`. Otherwise it falls back to `Model`, or isn't a relation at all.
   - **Scopes.** `Page::visible()` is `Builder<Page>`.
9. **Cost.**
   - **PHPStan's published figures.** Drupal (11,194 files) takes 22 s cold and 1.6–6.2 s warm.
   - **Larastan's public CI** ([§9](#9-cost-and-the-result-cache)), cold, on GitHub-hosted runners:

     | App | Files | Level | Time |
     | --- | --- | --- | --- |
     | Monica | 813 | 5 | 19–20 s |
     | Filament | 4,923 | 6 | 17–25 s |

   - **The result cache.** Collector data is stored per file in the result cache, and survives it. But the cache doesn't track Larastan's runtime-derived types (bindings, facade roots, aliases, migrations, config), so cached data can go stale until the next full run.

## Sources examined

Shallow clones at the tag current on 2026-10-18. No newer tags exist for PHPStan or Larastan than the ones static-extraction pinned. All code links point at these tags.

| Project | Tag | Commit | Role here |
| --- | --- | --- | --- |
| phpstan/phpstan-src | 2.3.1 | `cb3b7e99489b` | Scope, types, reflection, source locators, Collectors, result cache |
| phpstan/phpstan | 2.3.1 | `6155239b488e` | Website docs and blog under `website/src/` |
| larastan/larastan | v3.13.0 | `24ee69a7705f` | Container, facade and Eloquent extensions, boot, stubs, type tests |
| laravel/framework | v13.37.0 | `685f8d6c2ab4` | Container, facades, `AliasLoader`, environment loading |
| ondrejmirtes/BetterReflection | 6.73.0.6 | `b79afc9c50dd` | PHPStan's fork; the phpstorm-stubs stubber (pinned in phpstan-src's `composer.lock`) |
| JetBrains/phpstorm-stubs | (dev-master) | `e4f5f6c3de39` | Stubs for built-in classes (pinned in [phpstan-src composer.json L29](https://github.com/phpstan/phpstan-src/blob/2.3.1/composer.json#L29)) |
| barryvdh/laravel-ide-helper | v3.7.0 | (README only) | The `--write-mixin` mode |

Also read:

- GitHub issues on [larastan/larastan](https://github.com/larastan/larastan/issues) and [phpstan/phpstan](https://github.com/phpstan/phpstan/issues), cited inline;
- the [PHPStan 2.2.10 release notes](https://github.com/phpstan/phpstan/releases/tag/2.2.10);
- Larastan's public E2E workflow runs.

Larastan's type tests (`tests/Type/data/*.php`, which use `assertType`) are cited wherever they pin down an inferred type. A test is the strongest source available short of running code.

## 1. Container lookups

### 1.1 What Larastan returns

Every form ends in one helper, `AppMakeHelper::resolveTypeFromCall()` ([AppMakeHelper.php L25–64](https://github.com/larastan/larastan/blob/v3.13.0/src/ReturnTypes/AppMakeHelper.php#L25-L64)). It works like this:

- **No argument:** `*ERROR*`. The `app()` helper with no argument is handled earlier and returns `Illuminate\Foundation\Application` ([AppExtension.php L37–39](https://github.com/larastan/larastan/blob/v3.13.0/src/ReturnTypes/Helpers/AppExtension.php#L37-L39)).
- **Constant string or union of constant strings:**
  - Each value is passed to `$container->make($value)` on the booted app ([HasContainer.php L32–41](https://github.com/larastan/larastan/blob/v3.13.0/src/Concerns/HasContainer.php#L32-L41)).
  - The type is `new ObjectType($resolved::class)`: the class of the object actually returned ([AppMakeHelper.php L54](https://github.com/larastan/larastan/blob/v3.13.0/src/ReturnTypes/AppMakeHelper.php#L54)).
  - A union of keys gives a union of types ([L37–60](https://github.com/larastan/larastan/blob/v3.13.0/src/ReturnTypes/AppMakeHelper.php#L37-L60)).
- **If `make()` throws:**
  - `HasContainer::resolve()` catches `Throwable` and returns `null`.
  - A class string then falls back to the requested class: `getClassStringObjectType()` ([L45–48](https://github.com/larastan/larastan/blob/v3.13.0/src/ReturnTypes/AppMakeHelper.php#L45-L48)).
  - Any other string returns `ErrorType` ([L51, L55–56](https://github.com/larastan/larastan/blob/v3.13.0/src/ReturnTypes/AppMakeHelper.php#L51-L56)).
  - A string counts as a class string if PHPStan's reflection knows the class, so `'App\Foo'` written as a literal qualifies too ([ConstantStringType.php L100–118](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Type/Constant/ConstantStringType.php#L100-L118)).
- **Anything else** (a variable, a computed key, a non-constant `class-string<Foo>`): `MixedType` ([L63](https://github.com/larastan/larastan/blob/v3.13.0/src/ReturnTypes/AppMakeHelper.php#L63)).
  - This return value wins over the framework's own generic PHPDoc on `app()`, `resolve()` and `make()` ([helpers.php L123–132, L813–826](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/Foundation/helpers.php#L123-L132), [Container.php L855–865](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/Container/Container.php#L855-L865)).
  - So `app($classString)` is `mixed`, not `Foo` ([application-make.php L26–28](https://github.com/larastan/larastan/blob/v3.13.0/tests/Type/data/application-make.php#L26-L28)).

Which extension handles each form:

| Form | Extension | Matches on |
| --- | --- | --- |
| `app(X)`, `resolve(X)` | [AppExtension](https://github.com/larastan/larastan/blob/v3.13.0/src/ReturnTypes/Helpers/AppExtension.php#L27-L42) | function name |
| `App::make(X)`, `App::makeWith(X)` | [AppMakeDynamicReturnTypeExtension](https://github.com/larastan/larastan/blob/v3.13.0/src/ReturnTypes/AppMakeDynamicReturnTypeExtension.php#L21-L34) | the `App` facade |
| `app()->make(X)`, `$this->app->make(X)`, `->makeWith`, `->resolve` | [ApplicationMake…](https://github.com/larastan/larastan/blob/v3.13.0/src/ReturnTypes/ApplicationMakeDynamicReturnTypeExtension.php#L23-L36) and [ContainerMake…](https://github.com/larastan/larastan/blob/v3.13.0/src/ReturnTypes/ContainerMakeDynamicReturnTypeExtension.php#L23-L36) | `Illuminate\Contracts\Foundation\Application` and `Illuminate\Contracts\Container\Container` |
| `$app['key']` | [ContainerArrayAccess…](https://github.com/larastan/larastan/blob/v3.13.0/src/ReturnTypes/ContainerArrayAccessDynamicMethodReturnTypeExtension.php#L39-L79), registered for four container classes ([extension.neon L212–235](https://github.com/larastan/larastan/blob/v3.13.0/extension.neon#L212-L235)) | `offsetGet` |

Notes on the table:

- **Subtypes match.** An extension registered for a class or interface also applies to every subtype, because PHPStan looks up extensions for the receiver class plus all its parents and interfaces ([DynamicReturnTypeExtensionRegistry.php L79–96](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Type/DynamicReturnTypeExtensionRegistry.php#L79-L96), [ExtensionClassHelper.php L26–34](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Type/ExtensionClassHelper.php#L26-L34)).
- **`$this->app` in a service provider** is typed `@var \Illuminate\Contracts\Foundation\Application` ([ServiceProvider.php L19–24](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/Support/ServiceProvider.php#L19-L24)), so it is covered by the third row.
- **Array access differs on failure.** A failed lookup gives `ErrorType`, but a value that isn't an object is typed from the value itself ([L63–75](https://github.com/larastan/larastan/blob/v3.13.0/src/ReturnTypes/ContainerArrayAccessDynamicMethodReturnTypeExtension.php#L63-L75)).

### 1.2 Abstract or concrete

The type is the **concrete** class. Larastan's tests assert this:

| Expression | Asserted type | Test |
| --- | --- | --- |
| `$app->make(Illuminate\Contracts\Config\Repository::class)` | `Illuminate\Config\Repository` | [application-make.php L16](https://github.com/larastan/larastan/blob/v3.13.0/tests/Type/data/application-make.php#L16) |
| same, with `$app2` typed as the Application contract | same | [L21](https://github.com/larastan/larastan/blob/v3.13.0/tests/Type/data/application-make.php#L21) |
| `app('auth')`, `resolve('auth')` | `Illuminate\Auth\AuthManager` | [helpers.php L22–26](https://github.com/larastan/larastan/blob/v3.13.0/tests/Type/data/helpers.php#L22-L26) |
| `$app['auth']`, `$app['translator']` | `Illuminate\Auth\AuthManager`, `Illuminate\Translation\Translator` | [container-array-access.php L14–16](https://github.com/larastan/larastan/blob/v3.13.0/tests/Type/data/container-array-access.php#L14-L16) |
| `$app->resolve(FailsAtRuntime::class)`, whose constructor throws | `FailsAtRuntime` | [application-make.php L19](https://github.com/larastan/larastan/blob/v3.13.0/tests/Type/data/application-make.php#L19), [FailsAtRuntime.php](https://github.com/larastan/larastan/blob/v3.13.0/tests/Type/data/FailsAtRuntime.php) |

What that means for each case:

- **An interface `Foo` bound to `Bar`.** `app(Foo::class)` is `Bar`. `Foo` appears in no inferred type of the call. A Collector still sees `Foo` on the argument node, a `ClassConstFetch` typed as the constant class string `'Foo'` (see §8 on reading class strings).
- **An interface with no binding.** `make()` throws "not instantiable", so the type is `Foo` (inferred from the fallback code).
- **An instantiable class with no binding.** Laravel autowires it, so the type is the class itself (inferred).
- **String keys.** These map through the container's aliases and bindings. For example:
  - `'cache'` is a singleton built as `new CacheManager($app)` ([CacheServiceProvider.php L18–20](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/Cache/CacheServiceProvider.php#L18-L20)). The core aliases also map `CacheManager` and the cache `Factory` contract to that key ([Application.php L1653](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/Foundation/Application.php#L1653)), so `app('cache')` and `app(Factory::class)` both resolve to `CacheManager` (inferred);
  - `'mailer'` is bound to a closure returning `MailManager::mailer()`, from a deferred provider ([MailServiceProvider.php L8, L28–34](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/Mail/MailServiceProvider.php#L28-L34)).

  So `app('mailer')` is whatever class that closure returns in the booted environment: `Illuminate\Mail\Mailer` by default (inferred).
- **Unbound keys** (`app('nope')`). `make()` throws, and `'nope'` isn't a class, so the type is `*ERROR*`.
- **Computed keys** (`app("reports.$type")`). `mixed`, unless PHPStan has narrowed `$type` to a finite set of constants, in which case each is resolved.

### 1.3 Instantiation, failure and environment

- **Resolving instantiates.** `Container::make()` builds the object and runs its constructor and any resolving callbacks; singletons are memoised in the container.
- **In every process** (inferred). Larastan's boot runs once per PHPStan process: the main process and every worker (§7). So each worker builds its own instances.
- **Failure degrades the type, not the run.** Any `Throwable` from `make()` is swallowed:
  - a class-string lookup degrades to the abstract;
  - a string-key lookup degrades to `*ERROR*`.

  Larastan's own facade extension does the same: an exception from `getFacadeRoot()` is ignored ([FacadesMethodsExtension.php L105–120](https://github.com/larastan/larastan/blob/v3.13.0/src/Methods/FacadesMethodsExtension.php#L105-L120)).
- **The concrete depends on the environment** (inferred). It is whatever the booted environment binds, including bindings made under `if ($this->app->environment(...))`, and whatever constructors manage to do without services. A service whose constructor needs a database that CI lacks would show its abstract in CI and its concrete locally.
- **The result cache doesn't track bindings.** Larastan doesn't track any of this for the result cache. There are no `DependencyTracker`, `trackFileDependency` or `ResultCacheMetaExtension` uses in `src/` at v3.13.0. So changing a binding in a provider doesn't re-analyse the files that call `app(Foo::class)` (inferred; see §9).

## 2. Facades and aliases

### 2.1 How a facade method is found

PHPStan finds a method by trying these in order ([ClassReflection.php L626–647](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Reflection/ClassReflection.php#L626-L647), [LazyClassReflectionExtensionRegistryProvider.php L51–58](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/DependencyInjection/Reflection/LazyClassReflectionExtensionRegistryProvider.php#L51-L58)):

1. the native method;
2. extension-provided methods, Larastan's among them;
3. `@method` annotations;
4. `@mixin` types.

Larastan's `FacadesMethodsExtension` **steps aside** whenever the facade, an ancestor, or one of its mixins already provides the method ([FacadesMethodsExtension.php L50–53](https://github.com/larastan/larastan/blob/v3.13.0/src/Methods/FacadesMethodsExtension.php#L50-L53), [ReflectionHelper.php L55–81](https://github.com/larastan/larastan/blob/v3.13.0/src/Reflection/ReflectionHelper.php#L55-L81)). Only otherwise does it:

- call `getFacadeRoot()`, which resolves and instantiates the root, memoised per process;
- use the root class's native method ([L57–75, L105–120](https://github.com/larastan/larastan/blob/v3.13.0/src/Methods/FacadesMethodsExtension.php#L57-L120)).

So there are three cases:

| Facade has | Method comes from | Declaring class | Root consulted? |
| --- | --- | --- | --- |
| A Larastan stub with `@mixin` (`Cache`, `DB`, `Redis`, `Event`, `Queue`, `Log`, `Request`, `Artisan`, `Cookie`, `Config`, `Http`, `View`, `Route`, …) ([Facades.stub](https://github.com/larastan/larastan/blob/v3.13.0/stubs/common/Facades.stub)) | The mixin class's method, made static because `Facade` has `__callStatic` ([MixinMethodsClassReflectionExtension.php L49–81](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Reflection/Mixin/MixinMethodsClassReflectionExtension.php#L49-L81)) | The mixin class ([MixinMethodReflection.php L18–21](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Reflection/Mixin/MixinMethodReflection.php#L18-L21)) | No |
| `@method static` tags and no stub (most other framework facades; app facades written that way) | The tag | The facade ([AnnotationMethodReflection.php L40–43](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Reflection/Annotations/AnnotationMethodReflection.php#L40-L43)) | No |
| Neither (typical hand-written app facade) | The root's native method, through `getFacadeRoot()` | The root's runtime class | Yes, and it is instantiated |

The stubbed facades behave differently because of how stubs work. A stub's class PHPDoc **replaces** the original class PHPDoc entirely ([ClassReflection.php L2123–2132](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Reflection/ClassReflection.php#L2123-L2132), [StubPhpDocProvider.php L73–96](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/PhpDoc/StubPhpDocProvider.php#L73-L96)). So for `Cache`, the framework's 65 `@method static` tags are not read. That includes `@method static mixed get(...)` ([Facades/Cache.php L24](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/Support/Facades/Cache.php#L24)). Instead, Larastan's stub gives `Cache` `@mixin \Illuminate\Cache\CacheManager` and `@mixin \Illuminate\Cache\Repository` ([Facades.stub L14–18](https://github.com/larastan/larastan/blob/v3.13.0/stubs/common/Facades.stub#L14-L18)). The stub set itself depends on the booted Laravel version ([LarastanStubFilesExtension.php L19–39](https://github.com/larastan/larastan/blob/v3.13.0/src/LarastanStubFilesExtension.php#L19-L39)).

### 2.2 The classes in `Cache::get('k')`

The chain, inferred from the precedence code above:

1. `Cache` has no native or annotated `get`.
2. Its first mixin, `CacheManager`, has no native `get` either ([CacheManager.php](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/Cache/CacheManager.php)), but its own stub mixes in `Repository` ([CacheManager.stub](https://github.com/larastan/larastan/blob/v3.13.0/stubs/common/Cache/CacheManager.stub)).
3. `Repository::get()` declares a native `mixed` return type ([Repository.php L122–128](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/Cache/Repository.php#L122-L128)).

So the classes a Collector can see are:

- **the call's class:** `Illuminate\Support\Facades\Cache`;
- **the method's declaring class:** `Illuminate\Cache\Repository`;
- **the return type:** `mixed`, which holds no class.

Neither the object actually bound under `'cache'` nor the store-specific repository behind it is the type of anything. The `Cache` facade has no root fixed by code, because the container decides it at runtime ([Cache.php L74–87](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/Support/Facades/Cache.php#L74-L87)). Larastan's facade test asserts the neighbours of this case ([facades.php](https://github.com/larastan/larastan/blob/v3.13.0/tests/Type/data/facades.php)):

| Expression | Asserted type | Line |
| --- | --- | --- |
| `Redis::get('foo')` | `mixed` | [L48](https://github.com/larastan/larastan/blob/v3.13.0/tests/Type/data/facades.php#L48) |
| `DummyFacade::foo()`, with `@method` tags | the tag types | [L51–52](https://github.com/larastan/larastan/blob/v3.13.0/tests/Type/data/facades.php#L51-L52) |
| `Cache::driver()` | `Illuminate\Contracts\Cache\Repository` | [L72](https://github.com/larastan/larastan/blob/v3.13.0/tests/Type/data/facades.php#L72) |
| `Cache::remember(..., fn (): string => '123')` | `'123'`, the callback's return type | [L73–81](https://github.com/larastan/larastan/blob/v3.13.0/tests/Type/data/facades.php#L73-L81) |

### 2.3 Root aliases, `use Cache;` and app aliases

**How Laravel creates aliases.** The console kernel's `RegisterFacades` bootstrapper registers `AliasLoader` ([Kernel.php L127–135](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/Foundation/Console/Kernel.php#L127-L135), [RegisterFacades.php L24–27](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/Foundation/Bootstrap/RegisterFacades.php#L24-L27)):

- with `config('app.aliases')` merged with package-discovered aliases;
- **prepended** to the autoloader stack ([AliasLoader.php L160–177](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/Foundation/AliasLoader.php#L160-L177)).

Its `load()` either calls `class_alias()` or builds a real-time facade ([L70–81](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/Foundation/AliasLoader.php#L70-L81)).

**How PHPStan finds them** under Larastan:

1. **Capturing the autoloaders.** PHPStan records every autoloader that bootstrap files register ([BootstrapFilesRunner.php L49–57, L67–87](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Command/BootstrapFilesRunner.php#L49-L87)). Larastan's `bootstrap.php` is such a file ([extension.neon L10–11](https://github.com/larastan/larastan/blob/v3.13.0/extension.neon#L10-L11)).
2. **Running them early.** `AutoloadFunctionsSourceLocator` comes right after a bundled `Attribute` stub in PHPStan's locator chain, before every static locator ([BetterReflectionSourceLocatorFactory.php L106–112](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Reflection/BetterReflection/BetterReflectionSourceLocatorFactory.php#L106-L112)). For a class that doesn't exist yet, it actually runs each captured autoloader ([AutoloadFunctionsSourceLocator.php L27–53](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Reflection/BetterReflection/SourceLocator/AutoloadFunctionsSourceLocator.php#L27-L53)). For `Cache`, that is `class_alias('Illuminate\Support\Facades\Cache', 'Cache')`, which autoloads the facade for real.
3. **Locating the file.** It then reflects the now-defined class at runtime and parses its file statically ([AutoloadSourceLocator.php L317–330](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Reflection/BetterReflection/SourceLocator/AutoloadSourceLocator.php#L317-L330)). PHP's `ReflectionClass` on an alias reports the original name, so `\Cache` and `use Cache;` reflect as `Illuminate\Support\Facades\Cache`.
4. **Canonical names.** `ReflectionProvider::getClassName('Cache')` returns that canonical name ([BetterReflectionProvider.php L172–183](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Reflection/BetterReflection/BetterReflectionProvider.php#L172-L183)).

PHPStan's own E2E suite pins this shape down: "The shape Illuminate\Foundation\AliasLoader creates: a prepended autoloader that resolves a short alias to a real class with class_alias()". The test analyses `function (File $file)` at level 8 and expects no errors ([e2e/bug-15102b](https://github.com/phpstan/phpstan-src/tree/2.3.1/e2e/bug-15102b), [e2e-tests.yml L243–245](https://github.com/phpstan/phpstan-src/blob/2.3.1/.github/workflows/e2e-tests.yml#L243-L245)).

Aliases an app defines in `config/app.php` go through the same loader, so they resolve the same way (inferred).

**Caveat for a Collector** (inferred, not checked). A type built directly from the written name may carry the alias string, for example `ObjectType('Cache')` from `\Cache::get()`. Mapping names through `ReflectionProvider::getClassName()` or `getClass()->getName()` before matching them against groups avoids treating `Cache` and the facade as different types.

**This path has been fragile:**

- **PHPStan 2.1.34 to 2.1.39 (Fibers).** Users saw nondeterministic "unknown class Queue" errors for `use Queue;`, and the set of failing aliases changed with the worker count. Reported fixed from 2.1.40 ([larastan#2435](https://github.com/larastan/larastan/issues/2435), [phpstan#14036](https://github.com/phpstan/phpstan/issues/14036)).
- **PHPStan 2.2.9** demoted bootstrap-registered autoloaders below the static locators, which broke `Cache::lock()` on the alias ([larastan#2541](https://github.com/larastan/larastan/issues/2541), [phpstan#15102](https://github.com/phpstan/phpstan/issues/15102)). The change was reverted in 2.2.10 ([release notes](https://github.com/phpstan/phpstan/releases/tag/2.2.10)).

Larastan contributors' advice in both threads is to import the full facade class instead. A Rector set, `LARAVEL_FACADE_ALIASES_TO_FULL_NAMES`, does that rewrite ([#2435 comments](https://github.com/larastan/larastan/issues/2435)).

### 2.4 Real-time facades

`use Facades\App\Billing\Invoicer;` reaches the same `AliasLoader::load()`. That method writes `storage/framework/cache/facade-<sha1>.php` from a stub and `require`s it ([AliasLoader.php L89–121](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/Foundation/AliasLoader.php#L89-L121)). The generated class extends `Facade` and carries `@mixin \App\Billing\Invoicer` ([facade.stub](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/Foundation/stubs/facade.stub)).

By the mechanism in §2.3, PHPStan should then:

- locate the generated file;
- type calls through PHPStan's own mixin extension, with `App\Billing\Invoicer` as the declaring class.

This is inferred. No Larastan or PHPStan test covers real-time facades, and a Larastan maintainer declined to support them in 2021 ("I don't plan to support this", [larastan#795](https://github.com/larastan/larastan/issues/795)). Two side effects also follow (inferred):

- resolving one writes into the app's `storage/`;
- whether it works depends on that directory being writable.

## 3. Traits

**When traits are analysed.** PHPStan's blog states the rule directly ([how-phpstan-analyses-traits.md L50–56](https://github.com/phpstan/phpstan/blob/2.3.1/website/src/_posts/how-phpstan-analyses-traits.md#L50-L56)):

> PHPStan does not analyse traits in isolation, it analyses them only as part of a class that uses them. In order for a trait to be analysed, both the using class and the trait need to be in analysed paths. Traits that are used zero times will not be analysed at all. Trait that is used 10 times will be analysed 10 times.

The code agrees. When a class's `use T;` statement is processed, the handler does the following ([TraitUseHandler.php L120–162, L216–227](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Analyser/StmtHandler/TraitUseHandler.php#L120-L162)):

1. It skips the trait unless its file is an analysed file (L140).
2. It re-parses the trait file and walks the trait's statements with `$scope->enterTrait($traitReflection)`.
3. It fires a virtual `InTraitNode`, which is `@api` and carries the trait and the implementing class ([InTraitNode.php](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Node/InTraitNode.php)).

PHPStan's level-4 rule reports the unanalysed case as "Trait X is used zero times and is not analysed" ([NotAnalysedTraitRule.php L17, L50–61](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Rules/Traits/NotAnalysedTraitRule.php#L17-L61)).

Two cases follow from that rule (inferred):

- a vendor trait used by application code is never walked, because its file isn't analysed;
- a trait used only by classes outside the analysed paths is never walked either.

**Telling the using class from the trait.** Inside trait code, a Collector's `Scope` reports the following ([Scope.php L64–87](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Analyser/Scope.php#L64-L87), [ScopeContext.php L43–53](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Analyser/ScopeContext.php#L43-L53)):

| Call | Returns |
| --- | --- |
| `isInTrait()` | `true` |
| `getTraitReflection()` | the trait ("Returns the trait itself, not the class using the trait") |
| `getClassReflection()` | the using class |
| `getFile()` | the using class's file ("When analysing a trait, returns the file where the trait is used, not the trait file itself") |
| `getFileDescription()` | e.g. `"TraitFile.php (in context of class MyClass)"` |

Collected data is filed under `$scope->getFile()` ([FileAnalyserCallback.php L98, L242](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Analyser/FileAnalyserCallback.php#L242)), so trait-code data lands under the using class's file. Node line numbers still refer to the trait file (inferred). A Collector that wants the trait file must record `getTraitReflection()->getFileName()` itself.

**Per-class types.** The same trait line can have different types in different using classes:

- `$this`, `self`, `static` and `$this->prop` resolve against the using class kept in the scope context. The blog's example is exactly a property that exists in one user and not another.
- Trait PHPDoc is resolved in the implementing class's context ([ClassReflection.php L2151](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Reflection/ClassReflection.php#L2151)).

So a Collector can see different referenced classes for the same trait line, once per using class (inferred).

## 4. Unknown classes, and what PHPStan counts as declared

### 4.1 Reporting by level

The [rule-levels doc](https://github.com/phpstan/phpstan/blob/2.3.1/website/src/user-guide/rule-levels.md#L28-L30) lists "unknown classes" at level 0 and "validating PHPDocs" at level 2. The rules agree:

| Where the name appears | First level reported | Source |
| --- | --- | --- |
| `new X`, `X::m()`, `X::C`, `instanceof X`, `catch (X $e)`, extends, implements, `use T` (trait), native parameter, return and property types | 0 | `#[RegisteredRule(level: 0)]` on [InstantiationRule](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Rules/Classes/InstantiationRule.php#L49), [CallStaticMethodsRule](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Rules/Methods/CallStaticMethodsRule.php#L25), [ClassConstantRule](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Rules/Classes/ClassConstantRule.php#L36), [ExistingClassInInstanceOfRule](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Rules/Classes/ExistingClassInInstanceOfRule.php#L25), [ExistingClassesInTypehintsRule](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Rules/Methods/ExistingClassesInTypehintsRule.php#L18) and others |
| `@param`, `@return` | 2 | The same typehint rules check only native types while `checkThisOnly` is true (the default, [config.neon L82](https://github.com/phpstan/phpstan-src/blob/2.3.1/conf/config.neon#L82)). Level 2 sets it to false ([config.level2.neon L6](https://github.com/phpstan/phpstan-src/blob/2.3.1/conf/config.level2.neon#L6)), adding PHPDoc types ([FunctionDefinitionCheck.php L740–781](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Rules/FunctionDefinitionCheck.php#L740-L781)) |
| `@var` | 2 | [InvalidPhpDocVarTagTypeRule L27](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Rules/PhpDoc/InvalidPhpDocVarTagTypeRule.php#L27) |
| `@mixin`, `@property`, `@method` | 2 | [MixinRule](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Rules/Classes/MixinRule.php#L15), [PropertyTagRule](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Rules/Classes/PropertyTagRule.php#L15), [MethodTagRule](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Rules/Classes/MethodTagRule.php#L15) |
| `use X;` import on its own | never, for existence | The rule checks only the name's case and forbidden names ([ExistingNamesInUseRule.php L63, L137–144](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Rules/Namespaces/ExistingNamesInUseRule.php#L137-L144), [ClassNameCheck.php L33–80](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Rules/ClassNameCheck.php#L33-L80)) |

### 4.2 Detecting "declared nowhere" from a Collector

A Collector can detect undeclared names at any level:

- **Unknown names stay in types.** `new Missing()` is typed `ObjectType('Missing')` ([NewHandler.php L514–527](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Analyser/ExprHandler/NewHandler.php#L514-L527)). PHPDoc names resolve to object types the same way.
- **The check is public API.** `ReflectionProvider` is `@api` and has `hasClass()` ([ReflectionProvider.php L9–18](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Reflection/ReflectionProvider.php#L9-L18)). A Collector registered as a service can take it through its constructor, which is PHPStan's normal dependency injection.
- **Levels don't matter.** Nothing about this depends on the level, because levels only switch rules on (§6).

Caveat (inferred). `hasClass()` runs the whole locator chain in §4.3, including the app's runtime autoloaders. For a `Facades\…` name, that means writing a real-time facade file (§2.4).

### 4.3 What counts as declared

The source-locator chain, in order ([BetterReflectionSourceLocatorFactory.php L90–197](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Reflection/BetterReflection/BetterReflectionSourceLocatorFactory.php#L90-L197)):

1. A bundled stub for `Attribute`.
2. **Runtime autoloaders registered by bootstrap files**, actually executed, then located by reflection (§2.3). Under Larastan this is the booted app's `AliasLoader` and any other autoloader the app registers.
3. **Static file locators** ([L114–183](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Reflection/BetterReflection/BetterReflectionSourceLocatorFactory.php#L114-L183)):
   - analysed files and directories;
   - `scanFiles` and `scanDirectories`;
   - Composer. This reads `composer.json` and `vendor/composer/installed.json`, not the generated autoloader. It covers the root package's `autoload`; the root package's `autoload-dev`, when dev packages were installed; and every installed package's `autoload`, including PSR-4, PSR-0, classmap and `files` ([ComposerJsonAndInstalledJsonSourceLocatorMaker.php L58–117](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Reflection/BetterReflection/SourceLocator/ComposerJsonAndInstalledJsonSourceLocatorMaker.php#L58-L117)).
4. **PHP's own symbols from phpstorm-stubs** (§5).
5. **Any runtime autoloader**, run inside a trap that records which file it would include ([AutoloadSourceLocator.php L332–388](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Reflection/BetterReflection/SourceLocator/AutoloadSourceLocator.php#L332-L388)).
6. **Internal classes known only to the running PHP**, by runtime reflection, and code created with `eval` (L193–194).

PHPStan's docs say the same at user level ([discovering-symbols.md L6–28, L60–99](https://github.com/phpstan/phpstan/blob/2.3.1/website/src/user-guide/discovering-symbols.md#L6-L99)): analysed paths, Composer dependencies, `scanFiles`/`scanDirectories`, and class aliases or custom autoloaders through `bootstrapFiles`, which "will actually be executed by the PHP runtime".

**Stub files declare nothing.** "Stub files aren't a replacement for discovering symbols" ([stub-files.md L13](https://github.com/phpstan/phpstan/blob/2.3.1/website/src/user-guide/stub-files.md#L13)). They only override the PHPDoc of classes found some other way.

Consequence (inferred). Under Larastan, "declared" means statically locatable, or loadable by the booted app at runtime. That is wider than "has a file in application code or `vendor/`", and it can differ by environment: for example, a class only autoloadable when a dev package is installed.

### 4.4 IDE Helper `@mixin IdeHelperPost`

**What the IDE Helper does.** laravel-ide-helper's `--write-mixin` mode adds only `@mixin IdeHelperPost` to the model, and writes the generated PHPDoc into `_ide_helper_models.php` ([README L54, L125–130, L166–170](https://github.com/barryvdh/laravel-ide-helper/blob/v3.7.0/README.md)). That file is not autoloaded.

**Without configuration** (inferred from §4.1 and §4.3):

- `IdeHelperPost` is an unknown class;
- level 2 reports the `@mixin`, and levels 0–1 say nothing;
- the generated `@property` types are invisible, so Larastan falls back to migrations (§6.3).

**What users do** ([larastan#965](https://github.com/larastan/larastan/issues/965)):

- **Add the file to `scanFiles`.** The reporter said this first failed on its reference to `\Eloquent`, and that an IDE Helper fix had been proposed ([barryvdh/laravel-ide-helper#1268](https://github.com/barryvdh/laravel-ide-helper/pull/1268)).
- **Write the PHPDoc onto the models instead**, which "will just work", per a contributor.

A Larastan collaborator's position in the same thread is that "Larastan is not expected to work with `barryvdh/laravel-ide-helper`". Larastan itself does two related things:

- it excludes `Eloquent` from mixin resolution, the alias IDE Helper's `@mixin \Eloquent` names ([extension.neon L8–9](https://github.com/larastan/larastan/blob/v3.13.0/extension.neon#L8-L9));
- its docs suggest IDE Helper PHPDocs on the models for PostgreSQL schema dumps it can't parse ([custom-config-parameters.md L72–74](https://github.com/larastan/larastan/blob/v3.13.0/docs/custom-config-parameters.md#L72-L74)).

## 5. PHP built-ins

**The extension doesn't have to be loaded.** Built-in classes are located in phpstorm-stubs by `SkipClassAliasSourceLocator(CachedPhpInternalSourceLocator(PhpInternalSourceLocator(phpstormStubs)))` ([factory L184–190](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Reflection/BetterReflection/BetterReflectionSourceLocatorFactory.php#L184-L190)). It is consulted for any name that is not an already-loaded user class with a file ([SkipClassAliasSourceLocator.php L22–42](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Reflection/BetterReflection/SourceLocator/SkipClassAliasSourceLocator.php#L22-L42)). So `Redis` comes from the stub whether `ext-redis` is loaded or not. The stub map has all three examples:

- `'Imagick' => 'imagick/imagick.php'` ([L609](https://github.com/JetBrains/phpstorm-stubs/blob/e4f5f6c3de39f3bab3e9f3fca4b8cdb8b061e681/PhpStormStubsMap.php#L609));
- `'Redis' => 'redis/Redis.php'` ([L924](https://github.com/JetBrains/phpstorm-stubs/blob/e4f5f6c3de39f3bab3e9f3fca4b8cdb8b061e681/PhpStormStubsMap.php#L924));
- `'ZipArchive' => 'zip/zip.php'` ([L1343](https://github.com/JetBrains/phpstorm-stubs/blob/e4f5f6c3de39f3bab3e9f3fca4b8cdb8b061e681/PhpStormStubsMap.php#L1343)).

The stubber never checks `extension_loaded()`.

**What can differ by machine:**

- **PHP version, for core extensions.** The stubber filters `@since`/`@removed` by PHP version, but only for "core" extensions, a fixed list that includes `zip` but not `redis` or `imagick` ([PhpStormStubsSourceStubber.php L86–170, L586–612](https://github.com/ondrejmirtes/BetterReflection/blob/6.73.0.6/src/SourceLocator/SourceStubber/PhpStormStubsSourceStubber.php#L586-L612)). The PHP version is `phpVersion` if set, else `config.platform.php` from `composer.json`, else the running PHP ([config-reference.md L967–987](https://github.com/phpstan/phpstan/blob/2.3.1/website/src/config-reference.md#L967-L987)).
- **Extension major version, for two extensions.** Only `couchbase` and `ds` have versioned stubs ([PhpStormStubsMap.php L14997–15394](https://github.com/JetBrains/phpstorm-stubs/blob/e4f5f6c3de39f3bab3e9f3fca4b8cdb8b061e681/PhpStormStubsMap.php#L14997)). Their version comes from `composer.json` (`config.platform.ext-*`, then `require.ext-*`) and only then from the running extension ([ExtensionVersionProvider.php L44–130](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Reflection/BetterReflection/SourceStubber/ExtensionVersionProvider.php#L44-L130), [stubber L738–806](https://github.com/ondrejmirtes/BetterReflection/blob/6.73.0.6/src/SourceLocator/SourceStubber/PhpStormStubsSourceStubber.php#L738-L806)).
- **Classes missing from the stubs** are found only by the later runtime-reflection locator, which means only where the extension is loaded (inferred from L193).
- **The result cache key.** The result cache is invalidated when the set of loaded PHP extensions changes ([result-cache.md L19–22](https://github.com/phpstan/phpstan/blob/2.3.1/website/src/user-guide/result-cache.md#L19-L22)), so a cache built on one machine doesn't carry over to a machine with different extensions.

So the set of known **PHP code** types is the same on every machine for a fixed PHPStan version, a pinned `phpVersion`, and stubs that cover the extension. The stubs commit is fixed per PHPStan release ([composer.json L29](https://github.com/phpstan/phpstan-src/blob/2.3.1/composer.json#L29)).

## 6. Inference versus configuration

### 6.1 Collectors and levels

**Custom Collectors run at every level.**

- The collector registry takes every service tagged `phpstan.collector` ([RegistryFactory.php L14–31](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Collectors/RegistryFactory.php#L14-L31), [Collector.php L10–43](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Collectors/Collector.php#L10-L43)).
- Levels only register PHPStan's own `#[RegisteredRule]`/`#[RegisteredCollector]` services, such as the trait collectors at level 4 ([TraitDeclarationCollector.php L13](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Rules/Traits/TraitDeclarationCollector.php#L13)).
- Levels set only rule flags ([config.level0.neon … config.level10.neon](https://github.com/phpstan/phpstan-src/tree/2.3.1/conf)): `checkThisOnly`, `checkClassCaseSensitivity`, `reportMagicMethods`, `checkUnionTypes`, `checkNullables`, `checkExplicitMixed` and the like. None of them is an inference setting.
- The rule that reads a Collector's data is an ordinary custom rule, also registered at every level.

One caveat: when only some files are analysed, `CollectedDataNode::isOnlyFilesAnalysis()` is true and the data is partial ([NotAnalysedTraitRule.php L28](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Rules/Traits/NotAnalysedTraitRule.php#L28)).

### 6.2 PHPStan settings that change inferred types

| Setting | Default | Effect on what a Collector sees |
| --- | --- | --- |
| `phpVersion` | `null`: `composer.json` platform, else the runtime | Built-in signatures and stubs (§5) |
| `bootstrapFiles`, `scanFiles`, `scanDirectories` | `[]` | Which names resolve at all (§4.3) |
| `stubFiles` (Larastan adds many) | Larastan's set | Stub class PHPDoc replaces the original (§2.1). Return types and generics change |
| `inferPrivatePropertyTypeFromConstructor` | `false` ([config.neon L97](https://github.com/phpstan/phpstan-src/blob/2.3.1/conf/config.neon#L97)) | Untyped private properties take the constructor parameter's type ([config-reference.md L989–1004](https://github.com/phpstan/phpstan/blob/2.3.1/website/src/config-reference.md#L989-L1004)). Without it, `$this->repo->find()` in older code has a `mixed` receiver (inferred) |
| `treatPhpDocTypesAsCertain` | `true` ([L128](https://github.com/phpstan/phpstan-src/blob/2.3.1/conf/config.neon#L128)) | Decides whether a condition is always true from PHPDoc or native types ([IfHandler.php L60–61](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Analyser/StmtHandler/IfHandler.php#L60-L61)). That changes the merged scope after the branch, and whether later statements are reachable (inferred). Statements after code that always terminates are not walked: only an `UnreachableStatementNode` is emitted ([StatementsHandler.php L290–315](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Analyser/StatementsHandler.php#L290-L315)). So a Collector misses `new Foo` there |
| `earlyTerminatingFunctionCalls`, `earlyTerminatingMethodCalls` | `[]`; Larastan adds `abort`, `dd` ([extension.neon L5–7](https://github.com/larastan/larastan/blob/v3.13.0/extension.neon#L5-L7)) | Same reachability effect |
| `polluteScopeWithLoopInitialAssignments`, `…AlwaysIterableForeach`, `…Block`, `rememberPossiblyImpureFunctionValues` | `true` ([L122–130](https://github.com/phpstan/phpstan-src/blob/2.3.1/conf/config.neon#L122-L130)) | Whether variables, and their types, survive loops, blocks and repeated calls ([config-reference.md L380–396, L537–573](https://github.com/phpstan/phpstan/blob/2.3.1/website/src/config-reference.md#L380-L396)) |
| `universalObjectCratesClasses`, `mixinExcludeClasses` | Larastan: `Request`, `Optional`; `Eloquent` ([extension.neon L2–9](https://github.com/larastan/larastan/blob/v3.13.0/extension.neon#L2-L9)) | Unknown properties on those classes are `mixed`; mixins named there are ignored |
| Bleeding edge | off | Toggles that change inference rather than only adding rules: `stricterFunctionMap`, `unresolvedTemplateArguments`, `closureSignaturesFromUsages`, `staticVariablesFromUsages`, `curlSetOptArrayTypes` ([bleedingEdge.neon](https://github.com/phpstan/phpstan-src/blob/2.3.1/conf/bleedingEdge.neon)) |

Real apps change these settings. Monica's `phpstan.neon`, at the commit Larastan's E2E job pins, sets `inferPrivatePropertyTypeFromConstructor: true` and `treatPhpDocTypesAsCertain: false` ([monica phpstan.neon](https://github.com/monicahq/monica/blob/6e6ec217457e946e330ba7cd3b28107f4332c7b1/phpstan.neon)).

### 6.3 Larastan settings that change inferred types

Larastan's options are listed in [custom-config-parameters.md](https://github.com/larastan/larastan/blob/v3.13.0/docs/custom-config-parameters.md) and wired in [extension.neon L103–113, L488–556](https://github.com/larastan/larastan/blob/v3.13.0/extension.neon#L103-L113).

| Option | Default | Effect |
| --- | --- | --- |
| `databaseMigrationsPath`, `disableMigrationScan`, `squashedMigrationsPath`, `disableSchemaScan` | `database/migrations`, `false`, `database/schema`, `false` | Which columns exist, so which model properties are typed (§6.4) |
| `parseModelCastsMethod` | `false` | Reads the `casts()` body, not only its return type. Cast classes can then appear in property types |
| `checkConfigTypes` (+ `configDirectories`) | `false` | Registers return-type extensions for `config()`, `Config::get()` and `Repository::get()` ([extension.neon L106–111](https://github.com/larastan/larastan/blob/v3.13.0/extension.neon#L106-L111)). These read config files statically into constant types: `config('auth.defaults')` is `array{guard: 'web', passwords: 'users'}` ([config-helper-function.php L9–17](https://github.com/larastan/larastan/blob/v3.13.0/tests/Type/data/config-helper-function.php#L9-L17)). Class names in config then appear as constant class strings. Off, `config('key')` is `mixed` (Helpers.stub) |
| `generalizeEnvReturnType` | `false` | `env('X', 'd')` becomes `string` rather than the literal |
| `checkModelProperties` | `false` | Turns on the `model-property<…>` PHPDoc type and builder checks ([extension.neon L492–496](https://github.com/larastan/larastan/blob/v3.13.0/extension.neon#L492-L496), [BuilderHelper.php L119](https://github.com/larastan/larastan/blob/v3.13.0/src/Methods/BuilderHelper.php#L119)). That narrows string parameter types; it adds no classes (inferred) |

The booted app's own configuration also feeds types:

- **Auth.** The authenticated user's class comes from the booted `auth.guards` and `auth.providers` config ([LoadsAuthModel.php](https://github.com/larastan/larastan/blob/v3.13.0/src/Concerns/LoadsAuthModel.php)).
- **Dates.** The class used for date properties comes from `Date::now()::class` at runtime ([ModelCastHelper.php L199–210](https://github.com/larastan/larastan/blob/v3.13.0/src/Properties/ModelCastHelper.php#L199-L210)).

### 6.4 Model properties: migrations, not a database

**No database is involved.** "Larastan will automatically scan your application's migrations in order to infer the database schema" ([features.md L251–255](https://github.com/larastan/larastan/blob/v3.13.0/docs/features.md#L251-L255)). In code:

- **Migrations.** `MigrationHelper` globs migration paths (default `database_path('migrations')`) and parses each file with PHPStan's parser ([MigrationHelper.php L42–85](https://github.com/larastan/larastan/blob/v3.13.0/src/Properties/MigrationHelper.php#L42-L85)). `if` bodies in migrations are assumed true ([custom-config-parameters.md L24](https://github.com/larastan/larastan/blob/v3.13.0/docs/custom-config-parameters.md#L24)).
- **Schema dumps** are parsed with a MySQL-oriented SQL parser ([L52–75](https://github.com/larastan/larastan/blob/v3.13.0/docs/custom-config-parameters.md#L52-L75)). A PostgreSQL dump the parser can't read contributes no tables ([larastan#2568](https://github.com/larastan/larastan/issues/2568)).
- **The table name** comes from a model instance created without calling its constructor, via `getTable()` and `getKeyName()`. Dates and casts then pick the property type ([ModelPropertyHelper.php L56–100, L117–185](https://github.com/larastan/larastan/blob/v3.13.0/src/Properties/ModelPropertyHelper.php#L56-L185)).
- **`@property` tags win.** A model's `@property` tag, for example one written by IDE Helper, takes precedence: the extension steps aside ([L78–80](https://github.com/larastan/larastan/blob/v3.13.0/src/Properties/ModelPropertyHelper.php#L78-L80)).
- **No connection code.** No PDO or connection use appears in `src/Properties/` (inferred from search).
- **Caching.** Parsed schemas are cached in PHPStan's `tmpDir`, keyed by file paths and modification times ([custom-config-parameters.md L37–50](https://github.com/larastan/larastan/blob/v3.13.0/docs/custom-config-parameters.md#L37-L50)).

## 7. Larastan's boot

**What the boot does.** `bootstrap.php` ([L16–51](https://github.com/larastan/larastan/blob/v3.13.0/bootstrap.php#L16-L51)):

1. requires `getcwd()/bootstrap/app.php`, or the copy found relative to the vendor directory, or a Testbench app when analysing a package ([ApplicationResolver.php L60–106](https://github.com/larastan/larastan/blob/v3.13.0/src/ApplicationResolver.php#L60-L106));
2. runs `$app->make(Kernel::class)->bootstrap()` on the console kernel.

That runs the kernel's bootstrappers in order ([Kernel.php L127–135](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/Foundation/Console/Kernel.php#L127-L135)):

1. `LoadEnvironmentVariables`;
2. `LoadConfiguration`;
3. `HandleExceptions`;
4. `RegisterFacades`;
5. `SetRequestForConsole`;
6. `RegisterProviders`;
7. `BootProviders`.

Any `Throwable` is printed as "Application bootstrap failed" or "Laravel framework bootstrap failed", depending on whether the file is under `vendor/`, followed by `exit(1)` ([BootstrapErrorHandler.php L150–189](https://github.com/larastan/larastan/blob/v3.13.0/src/Support/BootstrapErrorHandler.php#L150-L189)).

**It runs in every process.** PHPStan runs `bootstrapFiles` "once per process": the main thread before an in-process analysis, every worker (spawned or forked), the main thread after the workers, and after a fully cached run ([BootstrapFilesRunner.php L9–28](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Command/BootstrapFilesRunner.php#L9-L28)). Workers stopped inheriting the parent's boot in 2.2.10: "Run bootstrapFiles per forked worker instead of inheriting the parent's" ([release notes](https://github.com/phpstan/phpstan/releases/tag/2.2.10)). So a run with N workers boots the app N + 1 times (inferred).

**What it needs:**

- **`bootstrap/app.php` and an installed `vendor/`**, including whatever packages the providers reference.
- **No `.env`.** It is loaded with `safeLoad()`, which tolerates a missing file ([LoadEnvironmentVariables.php L28–29](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/Foundation/Bootstrap/LoadEnvironmentVariables.php#L28-L29)).
- **Services, only if a provider touches them.** A database, Redis or a queue is needed only if a provider uses it in `register()` or `boot()`. Laravel's connections are lazy (inferred).
- **Evidence.** Larastan's E2E workflow analyses Monica at a pinned commit after only Composer steps (`composer config`, `composer require`):
  - no `.env` file is created, and the repository ships only `.env.example`;
  - no service containers are declared;
  - the job succeeds ([e2e-tests.yml L26–82](https://github.com/larastan/larastan/blob/v3.13.0/.github/workflows/e2e-tests.yml#L26-L82); [run 37635055699](https://github.com/larastan/larastan/actions/runs/37635055699), for the v3.13.0 commit).
- **What fails in CI without services** (inferred):
  - a provider that queries or connects at boot stops the run with exit 1;
  - a service whose constructor connects doesn't stop the run, but silently degrades the container-lookup and facade types that depend on it (§1.3, §2.1).

**The environment:**

- **Default.** `config('app.env')` is `env('APP_ENV', 'production')` ([config/app.php L32](https://github.com/laravel/framework/blob/v13.37.0/config/app.php#L32)). So it comes from the process environment, else from `.env`, else `production` ([LoadConfiguration.php L61](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/Foundation/Bootstrap/LoadConfiguration.php#L61)). Monica's config has the same default ([config/app.php L29](https://github.com/monicahq/monica/blob/6e6ec217457e946e330ba7cd3b28107f4332c7b1/config/app.php#L29)), so Larastan's CI analyses it as `production` (inferred).
- **Forcing `testing`.** Run `APP_ENV=testing vendor/bin/phpstan analyse`:
  - Laravel's env repository is immutable, so a real environment variable beats `.env` ([Env.php L76–93](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/Support/Env.php#L76-L93));
  - a set `APP_ENV` makes Laravel load `.env.testing` instead of `.env` when that file exists ([LoadEnvironmentVariables.php L41–58](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/Foundation/Bootstrap/LoadEnvironmentVariables.php#L41-L58)).
- **Cached config wins.** A cached config (`bootstrap/cache/config.php`) skips environment loading entirely and is used as-is ([LoadEnvironmentVariables.php L22–24](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/Foundation/Bootstrap/LoadEnvironmentVariables.php#L22-L24), [LoadConfiguration.php L35–45](https://github.com/laravel/framework/blob/v13.37.0/src/Illuminate/Foundation/Bootstrap/LoadConfiguration.php#L35-L45)).
- **`--env=testing` isn't usable** (inferred). Laravel reads it from `argv` (L43–47), but `phpstan analyse` defines no such option, so the PHPStan command line would reject it.

## 8. What a Collector can see per expression

### 8.1 The API

| Need | API | Source |
| --- | --- | --- |
| A Collector's signature | `processNode(Node $node, Scope&DependencyTracker $scope)`, returning arrays and scalars, or `null` | [Collector.php L29–43](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Collectors/Collector.php#L29-L43), [collectors.md L58](https://github.com/phpstan/phpstan/blob/2.3.1/website/src/developing-extensions/collectors.md#L58) |
| Type of an expression | `$scope->getType($expr)` (PHPDoc-enhanced), `getNativeType($expr)` (native only) | [Scope.php L146–154](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Analyser/Scope.php#L146-L154) |
| Every class in a type | `$type->getReferencedClasses()`: "all class names referenced anywhere in this type, recursively (including generic arguments, callable signatures, etc.)". Unions merge members; generic objects add their type arguments; `class-string<Foo>` gives `Foo`; `static`/`$this` give the class; `mixed` gives none | [Type.php L53–61](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Type/Type.php#L53-L61), [UnionType.php L177](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Type/UnionType.php#L177), [GenericObjectType.php L99](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Type/Generic/GenericObjectType.php#L99), [GenericClassStringType.php L43–46](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Type/Generic/GenericClassStringType.php#L43-L46), [StaticType.php L110](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Type/StaticType.php#L110) |
| Only the direct object classes | `getObjectClassNames()`, `getObjectClassReflections()` | [Type.php L63–72](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Type/Type.php#L63-L72) |
| A literal class string (`Foo::class`, `'App\Foo'`) | **Not** in `getReferencedClasses()`: constant strings return `[]` ([JustNullableTypeTrait.php L11–14](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Type/JustNullableTypeTrait.php#L11-L14), via `StringType`). Use `getConstantStrings()`, then `isClassString()` and `getClassStringObjectType()` | [ConstantStringType.php L100–118](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Type/Constant/ConstantStringType.php#L100-L118) |
| Receiver of `$a->m()` / `$a->p` | `$scope->getType($node->var)` | [MethodCallHandler.php L395–406](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Analyser/ExprHandler/MethodCallHandler.php#L395-L406), which records exactly this receiver type, the return type and the declaring class for its own dependency data |
| Class of `A::m()` | `$scope->resolveName($node->class)` or `resolveTypeByName()`, when the class is a `Name`; `getType()` when it is an expression | [Scope.php L171–192](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Analyser/Scope.php#L171-L192) |
| Declaring class of the method or property | `$scope->getMethodReflection($type, $name)?->getDeclaringClass()`, `getInstancePropertyReflection()`, `getStaticPropertyReflection()` | [Scope.php L122–126](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Analyser/Scope.php#L122-L126) |
| Whether a name exists | Inject `ReflectionProvider`; `hasClass()`, `getClassName()` | §4.2 |
| Trait context | `isInTrait()`, `getTraitReflection()`, `getClassReflection()`, `InTraitNode` | §3 |

### 8.2 Eloquent magic properties

**Relation properties** (`$post->comments`). Larastan's `ModelRelationsExtension` treats a property as a relation when three conditions hold ([ModelRelationsExtension.php L55–82](https://github.com/larastan/larastan/blob/v3.13.0/src/Properties/ModelRelationsExtension.php#L55-L82)):

- the model has no `@property` tag for it;
- a same-named native method exists;
- that method's declared return type is a `Relation`.

The property's type is then the relation's `TResult` template type, with collections normalised ([L84–109](https://github.com/larastan/larastan/blob/v3.13.0/src/Properties/ModelRelationsExtension.php#L84-L109)). What that gives depends on how the relation is declared:

| Relation declared as | `$post->comments` is | Source |
| --- | --- | --- |
| `/** @return HasMany<Comment, $this> */ public function comments(): HasMany` | `Illuminate\Database\Eloquent\Collection<int, Comment>` | [model-properties-relations.php L96–101](https://github.com/larastan/larastan/blob/v3.13.0/tests/Type/data/model-properties-relations.php#L96-L101) asserts this for `HasMany`, `HasManyThrough`, `HasOneThrough` (`Baz\|null`), `BelongsTo` and `MorphTo` |
| `public function comments(): HasMany` with no generics | `Collection<int, Model>` (inferred from the template bounds). "A relationship without the generic annotations above only exposes the `Model` bound" | [features.md L257–273, L329–331](https://github.com/larastan/larastan/blob/v3.13.0/docs/features.md#L257-L331) |
| `public function comments() { return $this->hasMany(Comment::class); }`, no return type | Not a relation, by design. Larastan 3 stopped reading method bodies: "this approach is slow" | [UPGRADE.md L9–40](https://github.com/larastan/larastan/blob/v3.13.0/UPGRADE.md#L9-L40), [larastan#2178](https://github.com/larastan/larastan/issues/2178) |
| An IDE Helper `@property-read` tag on the model | The tag's type; Larastan steps aside | [L57–59](https://github.com/larastan/larastan/blob/v3.13.0/src/Properties/ModelRelationsExtension.php#L57-L59) |

In every row, the call `$this->hasMany(Comment::class)` inside the relation method is itself typed `HasMany<Comment, $this>`, through Larastan's stubs. So the model's own edge to `Comment` is visible even when callers' edges are not (inferred).

**Column properties** (`$post->title`). These come from migrations (§6.4):

- nullable columns add `null`;
- date columns are the runtime date class, by default `Carbon\Carbon`;
- cast columns take the cast's type.

So **vendor code** such as Carbon, enums and cast classes appears in property types.

**Scopes.** A static or builder call to a local scope is typed as the model's builder:

| Expression | Asserted type | Test |
| --- | --- | --- |
| `Activity::completed()`, a `#[Scope]` method | `Builder<Activity>` | [model-scope-attribute-l12.php L14–19](https://github.com/larastan/larastan/blob/v3.13.0/tests/Type/data/model-scope-attribute-l12.php#L14-L19) |
| `User::query()->whereActive()` | `Builder<App\User>` | [model-scopes.php L45](https://github.com/larastan/larastan/blob/v3.13.0/tests/Type/data/model-scopes.php#L45) |
| `$user->someScope()` | `Builder<App\User>` | [L47–50](https://github.com/larastan/larastan/blob/v3.13.0/tests/Type/data/model-scopes.php#L47-L50) |

So `Page::visible()`, with `scopeVisible()` or `#[Scope] visible()`, is `Builder<Page>`: `getReferencedClasses()` gives `Builder` and `Page`. A model with a custom builder (`HasBuilder`) yields that builder class instead ([features.md L57–94](https://github.com/larastan/larastan/blob/v3.13.0/docs/features.md#L57-L94)).

## 9. Cost and the result cache

**PHPStan's published figures** ([phpstan-2-3-leap-in-performance.md L16, L34–43, L68–99](https://github.com/phpstan/phpstan/blob/2.3.1/website/src/_posts/phpstan-2-3-leap-in-performance.md#L68-L99)), on an M1 Pro:

| Project | Files | Run | Time |
| --- | --- | --- | --- |
| WordPress | — | full, PHPStan 2.3.0, Turbo off / on | 19.47 s / 7.93 s |
| Drupal | 11,194 | cold | 22 s |
| Drupal | 11,194 | warm, nothing changed | 1.6 s |
| Drupal | 11,194 | warm, 10 files re-analysed | 2.8 s |
| Drupal | 11,194 | warm, 10% re-analysed | 6.2 s |

**Laravel figures from Larastan's public CI.** Larastan's E2E job runs `phpstan analyse` on Monica (a Laravel app) and on Filament (a package monorepo, analysed through Testbench), on GitHub-hosted `ubuntu-latest` runners. The workflow has no cache step ([e2e-tests.yml](https://github.com/larastan/larastan/blob/v3.13.0/.github/workflows/e2e-tests.yml)), so every run is cold. "Perform static analysis" step times, which include `composer exec` start-up and the Laravel boot:

| Run (date) | Larastan commit | Monica: 813 files, level 5 | Filament: 4,923 files, level 6 |
| --- | --- | --- | --- |
| [37635055699](https://github.com/larastan/larastan/actions/runs/37635055699) (2026-10-07) | `24ee69a` = v3.13.0, PHPStan 2.3.0 | 19 s | 17 s |
| [38034977943](https://github.com/larastan/larastan/actions/runs/38034977943) (2026-10-10) | `f66505b` | 19 s | 24 s |
| [38401779291](https://github.com/larastan/larastan/actions/runs/38401779291) (2026-10-13) | `89729bb` | 19 s | 19 s |
| [38981985787](https://github.com/larastan/larastan/actions/runs/38981985787) (2026-10-16) | `043139b` | 20 s | 25 s |

File counts come from the progress bar in the first run's log. Two caveats:

- Monica's paths cover `app`, `database/factories`, `database/seeders` and `routes`.
- Neither project's memory use is logged.

These are the closest published figures to "a large Laravel app". A real app with thousands of classes would sit between Monica and Drupal (inferred).

**User reports show the outliers.** These are single cases, not benchmarks:

- One file with a large enum took a run from about 30 s to 9 min on Laravel 11 ([larastan#1943](https://github.com/larastan/larastan/issues/1943)).
- A PostgreSQL schema dump Larastan couldn't parse made every property lookup re-scan migrations. A cold run went from minutes to over 10, and one file took 211 s; with the fix, the same run took 30 s ([larastan#2568](https://github.com/larastan/larastan/issues/2568)).

**What a Collector adds** (inferred, not measured). A Collector that reads `getType()` on each node rides on inference PHPStan does anyway, plus its own per-node work. The boot cost is per process (§7).

**Collector data survives the result cache:**

- PHPStan stores collected data per file in the result cache. For files it doesn't re-analyse, it keeps the cached data; for files it does, it takes the fresh data ([ResultCacheManager.php L1496–1511](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Analyser/ResultCache/ResultCacheManager.php#L1496-L1511)).
- The docs say "Collectors are seamlessly integrated with the result cache" ([collectors.md L13](https://github.com/phpstan/phpstan/blob/2.3.1/website/src/developing-extensions/collectors.md#L13)).
- Collected values must be arrays and scalars.
- A Collector whose data holds file paths can implement `CollectorWithPaths` (`@api`), so the cache survives a change of checkout directory ([CollectorWithPaths.php L7–35](https://github.com/phpstan/phpstan-src/blob/2.3.1/src/Collectors/CollectorWithPaths.php#L7-L35)).

**The whole cache is thrown away** on any of these ([result-cache.md L5, L18–32](https://github.com/phpstan/phpstan/blob/2.3.1/website/src/user-guide/result-cache.md#L18-L32)):

- a new PHPStan version or PHP version;
- a change in loaded extensions or level;
- a change to config file hashes or the analysed path list;
- a change to `composer.lock`, stub files or bootstrap files;
- at least every 7 days.

**Staleness under Larastan** (inferred). The cache knows which files depend on which classes. It doesn't know that a type came from the booted app:

- a binding changed in a provider;
- a facade root;
- an alias;
- a migration;
- a config value.

So after such a change, unchanged consumer files keep stale collected data until a full run. The "bootstrap files" hash covers Larastan's own `bootstrap.php`, not the app's providers. Larastan doesn't use PHPStan 2.3's tracking APIs (§1.3).

**The fix is in PHPStan's 2.3 API.** A Collector can declare these dependencies itself:

- `trackFileDependency()` or `trackDirectoryDependency()` for files such as migrations or `config/*.php`;
- a `ResultCacheValueExtension` for values, such as "the class bound to `Foo`" ([result-cache-meta-extensions.md L45–150](https://github.com/phpstan/phpstan/blob/2.3.1/website/src/developing-extensions/result-cache-meta-extensions.md#L45-L150)).

The 2.3 blog notes phpstan-symfony already does this for its DI container ([L1133](https://github.com/phpstan/phpstan/blob/2.3.1/website/src/_posts/phpstan-2-3-leap-in-performance.md#L1133)).

## Inputs for the open decisions (not decisions)

- **Container lookups give two candidate ends**, and #10 must choose:
  - the abstract the code names (`Foo`, on the `::class` argument);
  - the concrete the booted app binds (`Bar`, the call's type).

  The concrete depends on the environment and fails silently back to the abstract. A Collector can record both, each labelled.
- **Facade edges in inferred types point at the facade and at a declaring class chosen by PHPDoc** (a Larastan stub mixin or a `@method` tag), not at the runtime root. Only app facades with neither reach their root. So "facade → root" wiring (#16) isn't reliably available from inference alone.
- **Trait code is seen once per using class**, with per-class types, and filed under the using class's file. Whether its references count against the trait, the using class, or both is a #10 question.
- **Unknown names can be marked at any level** with `hasClass()`. But "declared" under Larastan includes anything the booted app can autoload, so it can differ by environment.
- **The picture and the Pest plugin must share inference settings**, or their edges diverge:
  - one PHPStan and Larastan configuration (§6);
  - one PHP version pin;
  - one boot environment (§7).

  Real apps already set inference-changing options in their own `phpstan.neon`.
- **Cached runs can be stale for Larastan-derived edges** unless the Collector tracks bindings, migrations and config through the 2.3 dependency APIs.

## Open questions surfaced

1. For `app(Foo::class)`, is the edge to the abstract, to the inferred concrete, or both, labelled? And what if the concrete is environment-dependent?
2. Does the architecture spec fix the boot environment, for example always `APP_ENV=testing`? Does it reject cached config?
3. Does the package bring its own PHPStan configuration, pinning the settings that change inference? Or does it inherit the app's `phpstan.neon`, which may set them differently?
4. Are references in trait code attributed to the trait, to each using class, or to both? Is a trait no analysed class uses a gap with no edges?
5. Relations without generic return types expose only `Model` to callers. Should the plugin require `@return HasMany<Comment, $this>`, treat such call sites as unknown, or fall back to the `hasMany(Comment::class)` call in the relation method?
6. Should root aliases and real-time facades rely on Larastan's runtime alias path, which PHPStan broke twice in 2026? Or should they be resolved statically, or be forbidden?
7. Should the package's Collector track container bindings, migrations and config for the result cache, or accept stale warm runs and require full runs in CI?
8. What is the actual cost of a Collector-based extraction on the reference app set ([#3](https://github.com/kfriars/laravel-visual-spec/issues/3)), cold and warm? Nothing was run for this note.
