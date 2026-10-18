# Stress test: group membership facts on the six reference apps

Research for issue #9, How does the architecture spec say which classes belong where?, a ticket on the map issue #1. Written 2026-10-18. Findings only; nothing here is a decision. The decisions are in the ticket's resolution comment and in ADR 0003 on `master`, and they differ from the candidate tested here in places: `origin` became `application | framework | vendor`, and families may have several captures, including captures under `not`.

It tests the candidate fact vocabulary for group membership (`in`, `name`, `is`, `kind`, `extends`, `implements`, `uses`, `has attribute`, `origin`, `member of`, with and / or / not, and one-segment `{capture}` families). The test asks two things: can it describe the groups that each reference app's structure, and its arch tests where it has them, imply? And where can't it?

## Answer in brief

- **The vocabulary describes almost every group these apps imply, and nearly always without an `is` list.** Across about 260 group descriptions, the only `is` lists that are needed are:
  - excluding a base class (1 to 8 types per app);
  - two groups that are lists by nature: Advising App's 185 "legacy UUID" models, and Bagisto's 117 concord-registered models.
- **Location is easy, and fixed-depth `*` is a feature.**
  - Every app's layers, domains, packages or modules are `in X` or a `{capture}` family.
  - Fixed depth *helped* twice:
    - `BookStack\*\Models` correctly skips `BookStack\Exports\ZipExports\Models` (7 non-Eloquent DTOs).
    - `AdvisingApp\*\Actions` correctly skips the 63 Filament UI actions nested deeper.
  - A `**` would have pulled in 144 non-models in Firefly III, through the `Models` segments reused under `Api\V1\Controllers\Models\...`.
- **Role needs ancestry or suffix outside default-skeleton apps, and ancestry often runs through vendor code.**
  - BookStack: 11 of its 37 models and 17 of its 72 controllers sit flat in domain folders. `extends Model` and `name *Controller` get them exactly.
  - Advising App: 393 of its 412 Livewire classes and 196 of its 201 Filament resource pages are members only through vendor ancestry.
  - Bagisto: 47 of its 97 service providers are members only through vendor ancestry.
- **Hard gaps.** These are groups an architect plausibly wants that need `is` or cannot be said at all:
  1. Global functions (4 apps).
  2. Wiring-defined roles (2 apps): Coolify's 106 routed Livewire pages out of 244 components, and Bagisto's concord registry.
  3. "Other members of the same family" (2 apps).
  4. A family whose capture sits inside a class name, such as Koel's `Integrations\{vendor}` paired with `{vendor}Service` (2 apps).
  5. Ancestry into a vendor namespace: "anything Filament-coupled" would need 89 distinct FQNs (1 app).
  6. Grandfathered sets (1 app).
  7. Two-segment captures (1 app).
  8. Package or path membership that disagrees with the namespace (1 app).
- **The commonest awkwardness is family value filtering, in 4 apps.**
  - BookStack's business-domain family needs 11 `not in` clauses.
  - Bagisto's domain-package family needs 18.
  - Coolify's cross-layer family yields junk groups named after roles (`Concerns`, `Channels`, `Dto`).
- **Hygiene in the six apps:**
  - No case-only collisions, and no imports whose case differs from the declaration.
  - One lowercase-initial segment: Koel's `App\Http\Integrations\iTunes`, used by `ITunesService`.
  - 2 files hold two types each (Coolify).
  - 1 type's namespace contradicts its package folder (Advising App).

## Method

- **Clones.** All six apps were shallow-fetched at the pinned SHAs from `docs/research/laravel-layouts.md` (branch `research/laravel-layouts`). Every pin fetched; no fallback to a default branch was needed. Nothing in them was run.
- **Source roots** come from each `composer.json` PSR-4 map, plus path-repository module `composer.json` files for Advising App. `Database\` and `Tests\` roots are excluded.
- **Type counts:**

  | App | Types |
  | --- | --- |
  | Koel | 1,040 |
  | Coolify | 932 |
  | BookStack | 502 |
  | Advising App | 2,072 |
  | Bagisto | 1,337 |
  | Firefly III | 1,217 |

  Bagisto's figure includes about 100 seeders and factories under `packages/Webkul/*/src/Database`.
- **Scanner.** A throwaway Python lexer:
  - strips comments and strings;
  - tracks namespaces, `use` imports (including group use) and brace depth;
  - records each declaration's kind, modifiers, `extends`, `implements`, in-class `use Trait` and class-level `#[...]` attribute names, all resolved through imports.
- **Closures.** A query layer over the scan computes:
  - transitive ancestors, through app classes and then a small assumed vendor map;
  - all interfaces, including interface inheritance;
  - all traits, including traits of traits and traits of parents.
- **The vendor map is hand-written from framework knowledge, not read from `vendor/`.** It covers, for example, `Authenticatable`→`Model`, `Pivot`→`Model`, the Filament page and widget chains→`Livewire\Component`, the provider subclasses, PHP's exception tree→`Throwable`, and about 15 vendor model bases. Any membership that used one of these edges is counted as "vendor-dependent".
- **Oracles.** Each description is compared with an independent oracle where one exists. The oracles may use facts outside the vocabulary:
  - **Controllers:** route targets, scraped from route files. These are `X::class` references, plus `'Foo@bar'` strings matched by FQN suffix.
  - **Models:** `extends Model` with the vendor map.
  - **Jobs:** `ShouldQueue`.
  - **Others:** the app's own lists, such as Advising App's legacy file and Bagisto's concord `$models`.
- **Outputs and scripts** were throwaway and are not kept. The tables below are the record; to reproduce, clone at the pins and re-scan.
- **Notation.** Prefixes: L = location group, F = family, R = role group, A = from an arch test. "False" means the type matches but does not play the role; "missed" means it plays the role but does not match.

## Koel (default skeleton, `App\`)

| Group | Description | Count | False / missed | `is` / `not` |
| --- | --- | --- | --- | --- |
| HTTP entry layer | `in App\Http and not in App\Http\Integrations` | 361 | `App\Http\Integrations` holds outbound Saloon clients, not entry code | 1 `not` |
| Outbound integrations | `in App\Http\Integrations` | 43 | — | — |
| Services / Repositories / Values / Models ns / AI / Console | `in App\Services` etc. | 144 / 25 / 117 / 45 / 59 / 31 | `App\Models` holds 13 traits and 3 interfaces | — |
| Subsonic API slice | `in App\*\Subsonic or in App\Http\*\Subsonic` | 108 | — | or of 2 depths |
| Controllers | `in App\Http\Controllers and kind class and not is App\Http\Controllers\Controller` (or `extends` that base, or `name *Controller`) | 173 | 0 / 0 against 173 route targets | `is` (base, which is abstract) |
| Eloquent models | `extends Illuminate\Database\Eloquent\Model` | 29 | 0 / 0. **5 are vendor-dependent:** `User` (Authenticatable), `PodcastUserPivot` (Pivot), and `Audit`, `Passkey`, `PersonalAccessToken`, which extend vendor models | — |
| Models (alternative) | `in App\Models and kind class` | 29 | identical, with no vendor dependence | — |
| Model concerns | `in App\Models\Concerns` | 13 | — | — |
| Form requests | `extends Illuminate\Foundation\Http\FormRequest` (= `in App\Http\Requests and kind class`) | 107 | 0 / 0 | — |
| API resources | `extends Illuminate\Http\Resources\Json\JsonResource` | 31 | 4 `*ResourceCollection` classes are vendor-dependent | — |
| Queued work | `implements Illuminate\Contracts\Queue\ShouldQueue` | 23 | — | — |
| Jobs | `in App\Jobs` | 16 | misses 7 queued listeners, if "job" means queued | — |
| Broadcast events, mailables, policies, observers, casts, rules, builders, facades, middleware | `implements ShouldBroadcast`, `extends Mailable`, `in App\Policies`, … | 12, 7, 13, 10, 14, 13, 7, 6, 19 | — | — |
| Providers | `extends Illuminate\Support\ServiceProvider` | 14 | 3 are vendor-dependent (Auth/Event/Route providers) | — |
| Commands | `extends Illuminate\Console\Command` | 30 | — | — |
| Repositories | `implements App\Repositories\Contracts\Repository` | 18 | 0 / 0 | — |
| Services by suffix | `name *Service` | 54 | 79 missed of 131 classes in `App\Services`; use `in` instead | — |
| AI tools, Saloon requests, Saloon connectors | `implements Laravel\Ai\Contracts\Tool`, `extends Saloon\Http\Request`, `extends Saloon\Http\Connector` | 38, 27, 11 | — | — |
| Exceptions | `implements Throwable` | 44 | via PHP-core ancestry. `in App\Exceptions` adds 2 false (an interface and an error renderer) | — |
| Plus-only features | `has attribute App\Attributes\RequiresPlus` | 19 | 18 controllers and 1 service | — |
| Disabled in demo | `has attribute App\Attributes\DisabledInDemo` | 10 | — | — |

- Gaps with this set: 26, mostly in `App\Pipelines\Encyclopedia`, `App\Helpers` and `App\Hooks`.
- **Can't say:** "each `App\Http\Integrations\{vendor}` may only be used by `App\Services\Integrations\{vendor}Service`".
  - Static scan confirms that pattern holds for Lastfm, Spotify, YouTube, iTunes, Ticketmaster and ListenBrainz.
  - It needs a capture inside a class name, and the capture would also have to cross case: segment `iTunes`, class `ITunesService`.
  - Without it, it is 10 hand-written group pairs (`name LastfmService`, …). No `is` is needed, but no family either.
- 30 global functions live in `app/Helpers.php`.

## Coolify (action-based, Livewire-first, `App\`)

| Group | Description | Count | False / missed | `is` / `not` |
| --- | --- | --- | --- | --- |
| Livewire UI / Actions / HTTP / Models ns / Jobs ns / MCP / Notifications ns / Services / Support / Traits | `in App\Livewire` etc. | 250 / 90 / 82 / 69 / 64 / 55 / 49 / 44 / 22 / 29 | — | — |
| Livewire area family | `in App\Livewire\{area}` | 19 groups (Project 113, Server 47, …) | 18 classes directly in `App\Livewire` belong to no area | — |
| Action area family | `in App\Actions\{area}` | 14 groups (Server 19, Database 15, Proxy 11, …) | — | — |
| Cross-layer domain family | `in App\*\{domain} and (in App\Actions or in App\Livewire or in App\Notifications)` | 37 groups | **Junk groups named after roles:** `Channels` 12, `Concerns` 4, `Dto` 3, `TransactionalEmails` 4. Domains also don't align: `Livewire\Project` (113) holds `Application`/`Database`/`Service`, which are separate areas under `Actions` | needs value filtering (see the gaps list) |
| Livewire components | `extends Livewire\Component` | 244 | `in App\Livewire and kind class` gives 245; the extra is `MonacoEditor extends Illuminate\View\Component`, a Blade view component in the Livewire folder | — |
| lorisleiva actions | `uses Lorisleiva\Actions\Concerns\AsAction` | 71 | all of them are in `App\Actions` | — |
| Plain actions | `in App\Actions and not uses …AsAction` | 19 | Fortify 3, Stripe 6, User 5, … | — |
| MCP tools | `extends Laravel\Mcp\Server\Tool` | 45 | `in App\Mcp\Tools` also holds `MultipleContainersException`, declared in `GetLogs.php` | — |
| Jobs | `in App\Jobs` | 64 | all are queued; `ShouldQueue` gives 99, adding 33 notifications and 2 listeners | — |
| Notifications / queued notifications | `extends Illuminate\Notifications\Notification` [`and implements ShouldQueue`] | 34 / 33 | — | — |
| Models | `extends Model` (= `in App\Models and kind class`) | 69 | 2 are vendor-dependent (`User`, `PersonalAccessToken`); 44 extend `App\Models\BaseModel` | — |
| Controllers | `in App\Http\Controllers and kind class and not is …\Controller` | 49 | 1 false: `Api\OpenApi`, an OpenAPI attribute holder. Against route targets, 0 missed | `is` (base, which is not abstract) |
| Controllers by suffix | `name *Controller` | 43 | misses the 5 webhook controllers `Webhook\{Github,Gitlab,Gitea,Bitbucket,Stripe}` | — |
| Webhook controllers | `in App\Http\Controllers\Webhook and kind class` | 5 | — | — |
| Policies, events, broadcast events, commands, Spatie Data, view components, rules, exceptions, enums | the usual facts | 28, 25, 18, 41, 5, 7, 13, 17, 17 | — | — |
| OpenAPI schemas | `has attribute OpenApi\Attributes\Schema` | 17 | — | — |
| Providers | `extends ServiceProvider` | 8 | 4 vendor-dependent | — |

- Gaps: 15.
- **Can't say: Livewire pages.**
  - 106 of the 244 components are full-page route targets.
  - Nothing but route wiring separates them. `name Index or name Show` catches 45 pages, with 2 false members.
  - The only other attributes on routed pages are 1 `#[Lazy]` and 1 `#[Title]`.
- **Can't say: one capture across Livewire sub-areas.** A single `App\Livewire\*\{sub}` family collides:
  - `New` appears under `Destination`, `Project` and `Server`;
  - `PrivateKey` and `CloudProviderToken` appear under both `Security` and `Server`.

  Per-area sub-families (`App\Livewire\Server\{sub}`, …) work, but there would be 19 of them. This is the one case found where **one-segment captures are not enough** for a family.
- **Can't say: groups of functions.** Coolify has 438 global functions in `bootstrap/helpers/*.php`, and the mistakes research cites rules such as "Livewire may not call `instant_remote_process()`". Neither the functions nor rules about them can be expressed as groups of types.

## BookStack (domain-oriented, `BookStack\`)

| Group | Description | Count | False / missed | `is` / `not` |
| --- | --- | --- | --- | --- |
| Top-level folders | `in BookStack\{seg}` | 21 groups | mixes domains with technical folders: `Http`, `Console`, `Exceptions`, `Util`, `View`, `App`, `Api`, `Facades`, `Theming`, `Translation` | — |
| Business domains | `in BookStack\{domain} and not in BookStack\Http and not in …` (**11 `not in`**) | 11 groups (Entities 85, Access 54, Activity 54, Exports 30, …) | — | long `not` chain |
| HTTP kernel / platform `App` / Util / View | `in BookStack\Http` etc. | 29 / 17 / 23 / 42 | — | — |
| Controllers | `name *Controller and not is BookStack\Http\Controller, BookStack\Http\ApiController` | 72 | 0 / 0 against route targets | `is` (2 abstract bases) |
| Controllers by base | `extends BookStack\Http\Controller` | 73 | 1 false: `ApiController` (base) | — |
| Controllers by namespace | `in BookStack\*\Controllers and kind class` | 55 | **17 missed**, sitting flat, e.g. `Settings\SettingController`, `App\HomeController`, `Search\SearchController` | — |
| API controllers | `extends BookStack\Http\ApiController` | 20 | `name *ApiController` gives 19 | — |
| Models | `extends Model` | 37 | 0 vendor-dependent; 20 go through the app's `BookStack\App\Model` | — |
| Models by namespace | `in BookStack\*\Models and kind class` | 28 | 2 false (`EntityQueryBuilder`, `EntityScope`); **11 missed** flat (`Settings\Setting`, `Uploads\Image`, `Access\SocialAccount`, `App\Model`, …) | — |
| Models, depths 3 and 4 | `… or in BookStack\*\*\Models` | 35 | adds 7 false: `ZipExports\Models\ZipExport*` DTOs | — |
| Entities | `extends BookStack\Entities\Models\Entity` | 5 | — | — |
| Repositories | `name *Repo` | 13 | `in BookStack\Entities\Repos` gives 7 and misses 6 flat `*Repo` classes | — |
| Services / Queries / Tools | `name *Service`; `in BookStack\*\Queries or name *Queries`; `in BookStack\*\Tools` | 18 / 16 / 44 | none of the 18 services is in a `Services` namespace | — |
| View blocks | `implements BookStack\View\ViewBlockInterface` | 37 | — | — |
| Exceptions | `implements Throwable` | 35 | `in BookStack\Exceptions` gives 32: 3 false (handler, page, interface) and 6 missed (Oidc exceptions and others in domain folders) | — |
| Providers | `in BookStack\App\Providers` | 8 | `extends ServiceProvider` finds 7: it misses `TranslationServiceProvider`, whose vendor parent `Illuminate\Translation\TranslationServiceProvider` is unresolved, and 2 more members are vendor-dependent | — |
| Notifications, queued work, commands, ZIP DTOs, loggable models | the usual facts | 10, 11, 16, 6, 16 | — | — |

- Gaps: 0.
- The mistakes research's candidate rules are all expressible:
  - "controllers may not use `BookStack\Entities\Models`";
  - "Entities controllers may not use References": `member of controllers and in BookStack\Entities`.

## Advising App (InterNACHI modules `AdvisingApp\<Module>\` + `App\` core)

| Group | Description | Count | False / missed | `is` / `not` |
| --- | --- | --- | --- | --- |
| Core | `in App` | 348 | — | — |
| Modules | `in AdvisingApp\{module}` | 37 groups (Ai 222, StudentDataModel 182, …) | 1 type is placed by namespace in the wrong package (see the hygiene section) | — |
| Integration modules | or of 6 × `in AdvisingApp\Integration…` | 117 | — | **needs a segment glob** (`Integration*`); written as an or of 6 |
| Filament UI by location | `in App\Filament or in AdvisingApp\*\Filament` | 683 | against Filament ancestry (747): **93 false** (concerns, form and schema builders) and **157 missed** (71 enums implementing `HasLabel`, 36 `*Plugin`, presets) | — |
| Per-module models / Filament | `in AdvisingApp\{module}\Models and kind class`; `in AdvisingApp\{module}\Filament` | 28 and 36 groups | — | — |
| Models | `extends Model` | 210 | **46 vendor-dependent:** 23 pivots, `Authenticatable` subclasses, 9 core models on Filament/Spatie vendor bases | — |
| Models by namespace | `(in App\Models or in AdvisingApp\*\Models) and kind class` | 235 | **34 false**: scopes, `StateMachine`, attribute class; **9 missed**: `Alert\Configurations\*`, `*\Histories\*`, `*\Settings\SettingsProperties\*` | — |
| … minus Scopes/Attributes | add `not in App\Models\Scopes and not in AdvisingApp\*\Models\Scopes and not in App\Models\Attributes` | 204 | 3 false, 9 missed | 3 `not` |
| Filament resources / relation managers / UI actions / importers / exporters | `extends Filament\Resources\Resource` etc. | 53 / 23 / 23 / 17 / 15 | direct parents, so no vendor dependence | — |
| Filament resource pages | `extends Filament\Resources\Pages\Page` | 201 | **196 vendor-dependent** (via `EditRecord`, `ListRecords`, …); a static-only alternative is an or of 6 direct bases | — |
| Livewire (includes Filament) | `extends Livewire\Component` | 412 | **393 vendor-dependent** | — |
| Plain Livewire | `(in App\Livewire or in AdvisingApp\*\Livewire) and kind class` | 18 | of the 19 classes whose direct parent is `Livewire\Component`, misses `Report\Filament\Forms\Components\…\LiveFilterBuilderComponent` | — |
| Controllers | `name *Controller and not is App\Http\Controllers\Controller` (= the namespace form) | 88 | 2 not in route files (Filament `LogoutController`, `CalendarController`). **`extends` a base does not work:** 46 controllers extend nothing, 37 the app base, 2 extend `CalendarController`, and the remaining 3 extend `Illuminate\Routing\Controller` or vendor classes | `is` (base, which is not abstract) |
| Domain actions | `(in App\Actions or in AdvisingApp\*\Actions) and kind class` | 88 | the depth-3 rule keeps out the 63 Filament `…\Filament\…\Actions\*` UI actions; only 39 have an `*Action` suffix | — |
| Jobs | `(in App\Jobs or in AdvisingApp\*\Jobs) and kind class` | 111 | 3 false: job middleware in `*\Jobs\Middleware`. `ShouldQueue` (159) adds 51 notifications, listeners and 3 actions | needs a `not` for middleware |
| DTOs | `extends Spatie\LaravelData\Data` | 58 | the namespace form (58) differs by 4/4: it misses `App\Multitenancy\DataTransferObjects\*`, a role segment at depth 3 in core, and includes 2 casts, a concern and a non-Data class | or of depths |
| Settings, policies, observers, notifications, enums | the usual facts | 36, 69, 55, 30, 80 | — | — |
| Providers | `(in App\Providers or in AdvisingApp\*\Providers)` (= `extends ServiceProvider`) | 52 | 5 vendor-dependent via `extends` | — |
| No-permissions models / observed models | `has attribute App\Models\Attributes\NoPermissions` / `…\ObservedBy` | 14 / 52 | — | — |
| A: audit-restricted scope | `in App\Models or in AdvisingApp\StudentDataModel or …` (7 `in`s) | 508 | mirrors `AuditTraitUsageTest` | — |
| A: audit wrapper concern | `is AdvisingApp\Audit\Models\Concerns\Auditable` | 1 | the test's `ignoring(...)` | `is` (1) |
| A: legacy V4-UUID models | `is` 185 FQNs, read from `tests/Tenant/Unit/legacy-v4-uuid-models.php` | 185 | **not describable by any fact**; see the gaps list | **`is` (185)** |
| A: models that must not use V4 UUIDs | `extends Model and not member of legacy` | 25 | per module this is a family, as the arch test generates it | — |

- Gaps: 0.

## Bagisto (in-repo packages `Webkul\<Package>\`, concord)

| Group | Description | Count | False / missed | `is` / `not` |
| --- | --- | --- | --- | --- |
| Packages | `in Webkul\{pkg}` | 42 groups | — | — |
| Domain packages | `in Webkul\{pkg} and not in Webkul\Admin and …` (**18 `not in`**: layer and infrastructure packages plus 6 gateways) | 24 groups | — | long `not` chain |
| UI layers | `in Webkul\Admin or in Webkul\Shop` | 311 | — | — |
| Payment gateways | or of 6 `in` | 35 | no shared name to glob on | or of 6 |
| Admin area family | `in Webkul\Admin\Http\Controllers\{area}` | 10 groups | area names don't match package names: `Catalog` covers Product, Category and Attribute; `Customers` vs `Customer` | — |
| Per-package models / repositories / contracts | `in Webkul\{pkg}\Models and kind class and not extends …ModelProxy`; `in Webkul\{pkg}\Repositories`; `in Webkul\{pkg}\Contracts` | 25 groups each | — | — |
| Models | `extends Model` | 127 | 2 vendor-dependent (`Customer`, `Admin` via Authenticatable) | — |
| Models by namespace | `in Webkul\*\Models and kind class and not name *Proxy` (or `not extends …ModelProxy`) | 126 | misses the abstract base `Core\Eloquent\TranslatableModel`; 8 enums in `MagicAI\Enums\Models` are correctly excluded by depth | — |
| Model proxies / model contracts | `extends Konekt\Concord\Proxies\ModelProxy`; `in Webkul\*\Contracts and kind interface` | 119 / 128 | 126 of 127 models implement a same-named contract | — |
| Repositories | `extends Webkul\Core\Eloquent\Repository` (= `in Webkul\*\Repositories`) | 103 | — | — |
| Controllers | `in Webkul\*\Http\Controllers and kind class` | 131 | **8 false: concrete base controllers**, one per package (`Controller`, `APIController`, `Reporting\Controller`, …). Use `and not name Controller` plus 1 `is` | `not` + `is` |
| Admin / shop controllers | `extends Webkul\Admin\Http\Controllers\Controller` / `…Shop…` | 78 / 40 | — | — |
| Controllers outside Admin/Shop | `… and not in Webkul\Admin and not in Webkul\Shop` | 14 | the mistakes research's placement fix (`7206988b0e`) targets this group | — |
| DataGrids, factories, seeders, mailables, listeners, facades, form requests, API resources, payment methods, product types, enums | the usual facts | 51, 63, 33, 38, 50, 17, 24, 28, 10, 7, 28 | — | — |
| Providers | `extends ServiceProvider` | 97 | **47 vendor-dependent**, via `Konekt\Concord\BaseModuleServiceProvider` and `EventServiceProvider` | — |
| Queued work / jobs | `implements ShouldQueue` / `in Webkul\*\Jobs and kind class` | 50 / 18 | 32 of the queued classes are mailables in `*\Mail` | — |
| Concord-registered models | `is` 117, read from each `ModuleServiceProvider::$models` | 117 | differs from the 127 models by 10 (abstract bases, translations, Sitemap subclasses); wiring only | **`is` (117)** |

- Gaps: 0.
- **Notifications depend on vendor ancestry.** Bagisto's only notifications are 3 classes extending `Illuminate\Auth\Notifications\ResetPassword`. `extends Illuminate\Notifications\Notification` finds 0 of them without that vendor edge.

## Firefly III (long-lived monolith, `FireflyIII\`)

| Group | Description | Count | False / missed | `is` / `not` |
| --- | --- | --- | --- | --- |
| API v1 / web HTTP / both trees | `in FireflyIII\Api\V1`; `in FireflyIII\Http`; their `or` | 228 / 184 / 412 | — | — |
| Support / Repositories / Services / Internal services / domain factories / transaction rules | `in FireflyIII\Support` etc. | 150 / 69 / 32 / 25 / 18 / 38 | — | — |
| Root namespace only | `in FireflyIII and not in FireflyIII\*` | 1 (`FireflyIII\User`) | — | the "directly in" idiom works |
| Repository per entity | `in FireflyIII\Repositories\{entity}` | 22 groups | — | — |
| API / web controller areas | `in FireflyIII\Api\V1\Controllers\{area}`; `in FireflyIII\Http\Controllers\{area}` | 10 / 24 groups | the API's biggest area is literally `Models` (89 controllers). Area names drift between the trees (`Webhook`/`Webhooks`, `TransactionCurrency`/`Currency`) | — |
| Controllers | `name *Controller and not is …Http…\Controller, …Api\V1…\Controller` | 267 | 5 have no route-file reference found (possibly dead code, or a missed string form) | `is` (2 abstract bases) |
| Web / API controllers | `extends FireflyIII\Http\Controllers\Controller` / `extends FireflyIII\Api\V1\Controllers\Controller` | 125 / 141 | — | — |
| Form requests | `extends FormRequest` (= `(in …Http\Requests or in …Api\V1\Requests) and kind class`) | 123 | 0 / 0 | — |
| Models | `extends Model` | 51 | 1 vendor-dependent (`User`) | — |
| Models by namespace | `in FireflyIII\Models or (in FireflyIII and not in FireflyIII\*)` | 51 | `in FireflyIII\Models` alone misses the root `User` | — |
| Any `Models` segment (depths 2–5) | or of 4 depths | 194 | **144 false** (API controllers and requests under `…\Models\…`, `Support\Models` calculators). Evidence that `**` would hurt | — |
| Repository interfaces / implementations | `in FireflyIII\Repositories and kind interface` / `… kind class` | 33 / 33 | the container binding (wiring) is not needed: kind and namespace capture the pair | — |
| Events | `extends FireflyIII\Events\Event` | 42 | `in FireflyIII\Events` gives 50: 8 false (the abstract base and helper or value classes) | — |
| Observers | `in FireflyIII\Handlers\Observer` (= `name *Observer`) | 19 | — | — |
| Transformers, view components, rules, commands, rule actions, report generators, cronjobs, notifications, mailables, facades, route binders | the usual facts | 25, 38, 34, 85, 30, 18, 6, 28, 11, 11, 14 | transformers extend the app base, which extends League Fractal (unresolved vendor) | — |
| Queued work / jobs | `implements ShouldQueue` / `in FireflyIII\Jobs` | 38 / 7 | most queued classes are listeners; `Jobs\Job` is an abstract base | — |
| Providers | `extends ServiceProvider` | 21 | 3 vendor-dependent | — |

- Gaps: 30 (Exceptions, Helpers, Notifications leftovers).
- **Plausible want: a per-entity slice across trees.** For example `Repositories\Account` + `Api\V1\Controllers\Models\Account` + `Api\V1\Requests\Models\Account`.
  - It needs the *same* capture name bound in several or-branches at different depths.
  - Even then it breaks where the names drift.
- 33 global functions live in `app/Helpers/Functions/helpers.php`.

## What the vocabulary can't say

### Hard gaps: need `is`, or can't be said at all

Ranked by how many of the six apps are affected.

1. **Global functions** (4 apps: Coolify 438, Firefly III 33, Koel 30, BookStack 4).
   - Rules the mistakes research found, such as "Livewire/controllers may not call `instant_remote_process()`", target functions, not types.
   - Groups have only types as members.
   - **Extra fact needed:** a function member kind, e.g. `function <name glob>` or "declared in file X". The latter is path-like.
2. **Wiring-defined roles** (2 apps).
   - **Coolify Livewire pages:** 106 of 244 components; a name heuristic catches 45 of them.
   - **Bagisto concord-registered models:** 117 of 127 models.
   - **Extra fact needed:** "route target" or "registered in config". Both are excluded by design.
   - The alternative is an `is` list, or a marker the team adds (an attribute or interface) so that `has attribute` works.
3. **"Other members of the same family"** (2 apps).
   - Bagisto: "a package's repositories may not import *another* package's concrete models". This is mistakes-research bucket-A commit `2ca4f7087e`.
   - Advising App: "modules may not use other modules' internals".
   - `member of models and not in Webkul\{pkg}` leaves the capture unbound, because it appears only under `not`.
   - **Extra fact needed:** either a capture shared between a family and its rule targets ("`F repositories {pkg}` may only use `F models {pkg}`, proxies, contracts"), or "same/different capture as".
4. **A capture inside a class name, or a sibling relation** (2 apps).
   - Koel: `Integrations\{vendor}` is used only by `{vendor}Service`, and the case differs (`iTunes`/`ITunesService`).
   - Bagisto: `Models\X` sits alongside `XProxy` and `Contracts\X`.
   - Without it, the description is written per value with `name …`; no `is` is needed, but there is no family.
   - **Extra fact needed:** `name {capture}Service`, ideally matched case-insensitively.
5. **Ancestry into a vendor namespace** (1 app strongly).
   - "Anything Filament-coupled" in Advising App covers 747 types, through 89 distinct Filament FQNs named directly in `extends`/`implements`/`use`.
   - It is technically an or of 89 facts.
   - **Extra fact needed:** `extends|implements|uses member of <group>`, where the group can be a vendor group such as `in Filament and origin vendor`, or a namespace pattern after `extends`.
6. **Grandfathered sets** (1 app).
   - Advising App's 185 legacy V4-UUID models must use `HasVersion4Uuids`, and the other 25 must not.
   - No fact separates them other than the trait the rule is about.
   - **Extra fact needed:** none is natural. This stays an `is` list unless the team adds a marker.
7. **Two-segment captures** (1 app).
   - Coolify's Livewire sub-areas collide under one capture (`New` in 3 areas; `PrivateKey` and `CloudProviderToken` in 2).
   - Without it, the description needs one family per area (19 of them).
   - **Extra fact needed:** `App\Livewire\{area}\{sub}`, or a capture over several segments.
8. **Package or path membership differing from the namespace** (1 app, 2 types).
   - Advising App's `CouldNotFindSmsMessageFromData` file sits in the `integration-twilio` package but declares the `IntegrationAwsSesEventHandling` namespace. Twilio jobs import it under that namespace, so by namespace twilio depends on AWS SES.
   - Advising App's `overrides/OpenAI/…/ThreadRunResponse.php` is app-owned code in a vendor namespace. Whether its `origin` is `app` or `vendor` depends on whether origin means path or namespace.
   - **Extra fact needed:** a path or package fact, or a stated definition of `origin`.

### Describable, but awkward

1. **Filtering family values** (4 apps).
   - BookStack: 11 `not in`.
   - Bagisto: 18 `not in`.
   - Advising App: integration versus domain modules.
   - Coolify: junk cross-layer groups.
   - **Extra fact needed:** a capture restricted to listed values, or with exclusions, e.g. `{domain ∉ Http|Console|…}`, or "family over values …".
2. **A role segment at more than one depth** (3 apps).
   - Koel: the Subsonic slice sits at 2 depths.
   - Advising App: core `App\X` sits at depth 2, while module `AdvisingApp\*\X` and `App\Multitenancy\DataTransferObjects` sit at depth 3.
   - Firefly III: the root `User`.
   - These are written as or-chains.
   - Do not "fix" this with `**`: fixed depth avoided 7 false members in BookStack, 63 in Advising App and 144 in Firefly III.
3. **Excluding a base class with `not is`** (6 apps, 1 to 8 types each).
   - An `abstract` fact would replace these only where the bases are abstract: Koel, BookStack and Firefly III.
   - In Coolify, Advising App and Bagisto the base controllers are concrete.
   - `not name Controller` covers most of Bagisto's.
4. **A kind filter inside role folders** (6 apps). This works as specified: `and kind class` appears in about a third of the role descriptions.
5. **A glob inside a namespace segment** (1 app). Advising App's `Integration*` modules are written as an or of 6.
6. **Capture names drifting between families** (3 apps).
   - Bagisto: Admin `Catalog` versus the packages.
   - Firefly III: `Webhook`/`Webhooks`.
   - Koel: `iTunes`/`ITunes`.
   - No fact fixes this. It needs hand-written per-value groups.

Not needed in any of the six apps:

- a direct-parent fact (transitive `extends` plus a placement rule always sufficed);
- matching on attribute arguments;
- reflection-only facts;
- reading the container. Firefly III's interface/implementation repository pairs are fully described by namespace and kind.

## Vendor ancestry: how much hinges on it

| App | Group: members that need an assumed vendor edge / total |
| --- | --- |
| Koel | models 5/29; resource collections 4/31; providers 3/14 |
| Coolify | models 2/69; providers 4/8 |
| BookStack | providers 2/7, and 1 provider missed outright (unknown vendor parent) |
| Advising App | models 46/210; Livewire 393/412; Filament resource pages 196/201; Filament pages 270/290; widgets 69/77; pivots 4/23; providers 5/52 |
| Bagisto | providers 47/97; models 2/127; notifications 3/3 missed without it |
| Firefly III | models 1/51; providers 3/21 |

- The remaining roles in these apps are found from app source alone. Either the app names the role's base class directly (`extends Model`, `extends FormRequest`, `implements ShouldQueue`), or the chain runs through app-owned bases (`BaseModel`, `App\Http\Requests\Request`).
- So for planned types, a plan that states the direct parent covers most roles.
- Vendor ancestry is unavoidable mainly for Filament and Livewire, for providers, and for models built on vendor or `Authenticatable` bases.

## Case sensitivity, multiple types per file, namespaces vs paths

- **Case.**
  - No FQN or namespace in the six apps differs from another only by case.
  - No `use` import differs in case from the declaration it resolves to.
  - One lowercase-initial segment exists: Koel's `App\Http\Integrations\iTunes`, alongside the class `ITunesService`.
  - PHP resolves class names case-insensitively; segment matching in this test was case-sensitive. The two agree on these apps, but they would diverge on a plan that writes `ITunes`. Pest's own namespace matching should be checked before choosing a rule.
- **Multiple types per file.** There are 2 such files, both in Coolify:
  - `app/Enums/ProxyTypes.php` declares `ProxyTypes` and `ProxyStatus`;
  - `app/Mcp/Tools/GetLogs.php` declares `GetLogs` and `MultipleContainersException`.

  Both secondary types are invisible to PSR-4 autoload until the file is loaded. A reflection-based check can see them differently from a file-based scan. The exception is a member of any `in App\Mcp\Tools` location group.
- **Namespace versus path.**
  - In Advising App, 1 of 2,072 types breaks PSR-4 (`CouldNotFindSmsMessageFromData`, above). Composer cannot autoload it from its namespace, so Pest's reflection may not see it at all, while a static scan does.
  - Advising App's module folders are kebab-case (`student-data-model`) while the namespaces are StudlyCase (`StudentDataModel`). A family keyed on the namespace is unaffected; one keyed on package path would see different names.
  - Every other type in all six apps matches its PSR-4 path exactly, including case.
- **Anonymous and Blade classes.** None of the six apps has `new class` components in Blade. Anonymous migrations are outside the scanned roots.

## Caveats

- **Static approximation.**
  - The lexer handles normal code well, but it is not a PHP parser.
  - Its counts match the earlier layout research within a few types: Bagisto's figure is higher only because `src/Database` is included.
  - Rare string forms (`"{$a["b"]}"`) and conditional declarations could slip through.
- **Assumed vendor map.** Vendor ancestry comes from a hand-written map (84 parent edges and 6 interface edges), not from `vendor/`.
  - Edges I am less sure of: the Filament v4 page and cluster chains, and the vendor model bases.
  - Any count marked vendor-dependent could shift if an edge is wrong.
  - Chains that stop at a vendor class not in the map number 10 to 36 types per app, 199 in Advising App (mostly Filament form blocks and actions). They could hide further members.
- **Route-target oracle.** It is regex-based. Firefly III's `'Foo@bar'` strings are matched by FQN suffix, so they over-match. Filament panel routes, Livewire nested usage and `Mcp::web` registrations are not modelled.
- **Oracles are themselves descriptions.** "False" and "missed" are relative to them. For example, Advising App's "Filament-coupled" oracle counts enums implementing `HasLabel`, which an architect might not call Filament UI.
- **The groups are my reading of each app's structure** and of the arch rules in the mistakes research. They are not the teams' own groups, except where an arch test is mirrored (Advising App's audit and UUID tests). Groups of other plausible architects would differ in places, but the gap list does not rest on any single group choice.
- **Advising App is under the Elastic License 2.0.** Only FQNs and counts are quoted from it; no code was copied.
