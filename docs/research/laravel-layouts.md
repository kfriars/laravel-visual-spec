# How real Laravel apps are laid out, and a reference app set

Research for issue #3, a ticket on the map issue #1. Written 2026-10-17. These are findings to feed later decisions; nothing here is a decision. The question being tested is the WIP draft's assumption that "namespace = layer" (`docs/wip-spec-2026-10-16.md`, Non-goals and Rule model).

## Answer in brief

- **Laravel enforces no layout.** The docs say: "Laravel imposes almost no restrictions on where any given class is located - as long as Composer can autoload the class." The skeleton ships three classes. Every other folder appears only when a `make:` command creates it. [L-structure], [skeleton]
- **Five layout families exist in maintained public apps:** default skeleton, action-based, domain-oriented, module packages, and large long-lived monoliths. Most real apps mix families: Coolify is action-based *and* Livewire-first, and Advising App is modular with a default-skeleton core.
- **Namespace tracks architectural *role* only for the framework's generator roles, in default-skeleton apps.** Even there, role folders hold other kinds of type. Koel's `App\Models` holds 45 types, of which 13 are traits and 3 are interfaces. Snipe-IT's `App\Models` subtree holds 162 types, of which only 43 are Eloquent models.
- **In the other families, the leading namespace segments carry the domain, module or package, and the role is a later segment, or absent.** BookStack has `BookStack\Settings\Setting` (a model) sitting next to `BookStack\Settings\SettingController`. Advising App has `AdvisingApp\Task\Models\Task`. Bagisto has `Webkul\Sales\Models\Order` next to `OrderProxy`, which is not a model.
- **Many roles can be recognised only by base class, trait, interface or name.** Examples are Eloquent models in flat domain folders, Livewire components, Filament resources, lorisleiva actions (marked by a trait), queued work (`ShouldQueue` also covers notifications and mailables) and Spatie Data objects. Several app-specific roles have no marker at all except their folder: plain action classes, services, values and support code.
- **Controllers are a special case.** Since Laravel 11 the skeleton's base `Controller` extends nothing, so a controller is recognisable only by namespace, `Controller` suffix, or being a route target. Neither of the first two holds universally. Akaunting has 88 controllers and Leantime 160, none with the suffix.
- **Real dependencies also live outside PSR-4 classes:** global helper functions (Coolify has 438), anonymous Livewire 4 components inside Blade files, and vendor namespaces overridden from inside the repo (FreeScout, Akaunting).
- **No module package provides "module-internal".** Neither InterNACHI/modular nor nWidart/laravel-modules has a visibility concept, and Advising App's modules import each other's models freely.
- **Reference set (6 apps):** Koel, Coolify, BookStack, Advising App, Bagisto and Firefly III, pinned below.

## Method

- **Primary sources only.** These are the apps' own repositories, the framework skeleton, official Laravel and Livewire docs, and the source of the module packages, Pest and lorisleiva/laravel-actions. Each app was shallow-cloned at its default branch on 2026-10-17 and only read. No `composer install`, artisan or app code was run.
- **Candidates.** Candidate apps came from well-known open-source Laravel apps and from GitHub code search for `internachi/modular` and `nwidart/laravel-modules` in `composer.json`, for `modules_statuses.json`, and for `src/Domain` autoload mappings.
- **Class counts.** Counts are type declarations (`class`, `interface`, `trait`, `enum`, with modifiers) at the start of a line in `.php` files under each app's source roots. `tests`, `database`, `resources`, `routes`, `config` and `lang` are excluded. Anonymous classes are not counted. Treat every number as rough, as the ticket asks.
- **Role detection.** A small regex scanner resolved `use` imports and walked `extends`, `implements` and in-class `use Trait` through the app's own classes. It matched framework and vendor bases by fully qualified name, for example `Illuminate\Database\Eloquent\Model` and `Lorisleiva\Actions\Concerns\AsAction`.
- **Limitation.** The scanner does not see the ancestry of vendor classes (no `vendor/` was installed). A class extending a vendor class that itself extends `Model` is missed unless noted. That limitation is itself a finding (see "What a declaration would need").

## Sources

Commits are the default-branch HEAD at the time of reading. Links are `https://github.com/<repo>/tree/<sha>`. In-text citations are written `[key] path`.

| Key | Repo | Commit (date) |
| --- | --- | --- |
| skeleton | laravel/laravel | `9ffef3bc4241e3cc333ef267258a404d0d95f194` (2026-10-14, branch 13.x) |
| koel | koel/koel | `7c34114c6e8d099278c732b85853f720c2546d82` (2026-10-16) |
| coolify | coollabsio/coolify | `6964714b5da26d600621eb137c9845e6240f7d52` (2026-10-16) |
| bookstack | BookStackApp/BookStack | `d3db80004b51a395a9dabdd406f5fbda3d4d76e4` (2026-10-15) |
| advising | canyongbs/advisingapp | `d945da2b74325e5ecdbc3cc7210a46768ea4fe1e` (2026-10-16) |
| bagisto | bagisto/bagisto | `748d3ea117dcf121bfbf57a4d8d996c81144510f` (2026-10-16) |
| firefly | firefly-iii/firefly-iii | `b9c31006e8d3008df3d12e23de86b66088c9480e` (2026-10-17) |
| lychee | LycheeOrg/Lychee | `89a42fc32613224400033e304f151f388d4175fc` (2026-10-17) |
| vito | vitodeploy/vito | `8598e4dc0bce12d2b2f915766a100b445b7d9e22` (2026-10-07) |
| pinkary | pinkary-project/pinkary.com | `f4a528d4e5f46045aa3175d117615c0e1c14270b` (2026-10-15) |
| laravelio | laravelio/laravel.io | `d119d796abeb6fec17fffa8ff9265e6bf95b24e9` (2026-10-16) |
| snipeit | grokability/snipe-it | `d98baec311438e0818988c130eefa8e813a34c22` (2026-10-17) |
| osuweb | ppy/osu-web | `2b41065adeaebc5e918ef8599e815ab81ed27140` (2026-10-16) |
| pixelfed | pixelfed/pixelfed | `6cea5a4e9688471e8cb4d8ff744dc56518496669` (2026-10-16) |
| pterodactyl | pterodactyl/panel | `f9bc3e5e70510774e5daa1c90b6f5b624c99d2bc` (2026-10-16) |
| akaunting | akaunting/akaunting | `06dfa473e0d71a5262a58f6f0759c6badf34d5d2` (2026-10-08) |
| freescout | freescout-help-desk/freescout | `9d11ff01fc7604b87813be4a23fcc2e642817239` (2026-10-10) |
| monica | monicahq/monica | `6e6ec217457e946e330ba7cd3b28107f4332c7b1` (2026-09-24) |
| leantime | Leantime/leantime | `9d09091fefc85fb910a145a7cb9c56d03fd9dda0` (2026-10-15) |
| vfm | VueFileManager/vuefilemanager | `5ba4bf3478803b5e698afedb1c577aa0edd4ce93` (2023-10-07) |
| coloredcow | ColoredCow/portal | `b6d806748a8015cbbcc055fbb5333dff85fa9fab` (2026-10-14) |
| laravelcm | laravelcm/laravel.cm | `8f97bd1e089fd4b1bc12a5b103b39819cc0fba39` (2026-08-24) |
| modular | InterNACHI/modular | `428b63ec001acbf78225f132c25ebc5442621aa1` (2026-05-21; latest release 3.0.2) |
| nwidart | nWidart/laravel-modules | `965931ef67f5ec1d1fa90998f8d1b10f912a1d71` (2026-04-13; latest release v13.0.0) |
| actions | lorisleiva/laravel-actions | `c28e8fb0573324d28fdfdf780425329646bf28a1` (2026-10-15) |
| ddd | lunarstorm/laravel-ddd | `09317395e99b79eb950dfd7a8efc46fab0f9b8d4` |
| pest | pestphp/pest | tag `v5.3.2` (file blob `5ba3e23a` for `src/ArchPresets/Laravel.php`, same on branch 5.x) |
| pest-arch | pestphp/pest-plugin-arch | `244465d5dc9ea9fb5ac5c9badb7eb47d1594e4c1` (branch 5.x) |
| L-structure | https://laravel.com/docs/13.x/structure | read 2026-10-17 |
| L-events | https://laravel.com/docs/13.x/events#event-discovery | read 2026-10-17 |
| L-authz | https://laravel.com/docs/13.x/authorization#policy-discovery | read 2026-10-17 |
| LW-components | https://livewire.laravel.com/docs/4.x/components | read 2026-10-17 |
| php-class-uses | https://www.php.net/manual/en/function.class-uses.php | read 2026-10-17 |

## Baseline: what the framework and tools assume

- **The skeleton.** `app/` contains three files: `Http/Controllers/Controller.php`, `Models/User.php` and `Providers/AppServiceProvider.php` [skeleton]. The base controller is `abstract class Controller { // }` and extends nothing [skeleton] `app/Http/Controllers/Controller.php`. `User` carries Eloquent attributes (`#[Fillable([...])]`, `#[Hidden([...])]`) [skeleton] `app/Models/User.php`.
- **Composer decides where classes live; Laravel does not.** "Laravel imposes almost no restrictions on where any given class is located". Directories other than `Http`, `Models` and `Providers` "will be generated inside the app directory as you use the make Artisan commands" [L-structure].
- **The framework has a few location conventions of its own:**
  - Listener discovery scans "your application's `Listeners` directory" for methods beginning with `handle` or `__invoke` [L-events].
  - Policy discovery requires policies "in a `Policies` directory at or above the directory that contains your models", named after the model with a `Policy` suffix. A `#[UsePolicy]` attribute or `Gate::guessPolicyNamesUsing` overrides it [L-authz].
- **Pest's Laravel arch preset hard-codes the default skeleton.** For example, it expects `App\Models` classes to extend `Illuminate\Database\Eloquent\Model` (ignoring `App\Models\Scopes`). It also says `App` must not extend `Model` outside `App\Models`, `App\Http\Controllers` classes must have the `Controller` suffix, and `App\Http` may only be used in `App\Http` and `App\Providers` [pest] `src/ArchPresets/Laravel.php`.
- **Pest can filter targets by type relationship, at runtime.** It offers `classes()`, `interfaces()`, `traits()`, `enums()`, `implementing()`, `extending()` and `using()`. The last three are implemented with `class_implements`, `is_subclass_of` and `class_uses`, so they need the classes autoloadable [pest-arch] `src/PendingArchExpectation.php`. `class_uses` "does however not include any traits used by a parent class" [php-class-uses].

## The layout families

### 1. Default skeleton

Apps that grow the `make:`-generated folders under `App\`. Examples are Koel, Pixelfed, osu-web, Snipe-IT, Pterodactyl, Laravel.io and Pinkary.

Koel, abridged, with type counts per folder [koel] `app/`:

```
app/                      1,040 types, all under App\
  Http/          404      Controllers (174 controllers), Requests (107), Resources (31), Middleware
  Services/      144      App\Services\..., incl. Integrations, Playlist, Auth, Subsonic
  Values/        117      value objects, e.g. App\Values\Branding (final, implements Arrayable)
  Ai/             59
  Exceptions/     46
  Models/         45      27 Eloquent models + 2 vendor-model subclasses + 13 traits + 3 interfaces
  Console/        31      30 commands in App\Console\Commands
  Repositories/   25      18 *Repository classes
  Jobs/16  Enums/17  Casts/14  Providers/14  Policies/13  Rules/13  Observers/10  Builders/8 ...
```

Where classes live, and whether namespace tracks role:

- **Framework roles: yes, almost perfectly.** Koel's 27 Eloquent models are all in `App\Models`, its 174 `*Controller` classes all in `App\Http\Controllers`, its 107 form requests all in `App\Http\Requests`, and its 30 commands all in `App\Console\Commands` [koel]. Pixelfed is similar: 109 of 109 models in `App\Models` and 113 of 113 controllers in `App\Http\Controllers` [pixelfed].
- **Role folders are not pure.** They contain traits, interfaces, scopes and builders:
  - Koel's `App\Models` is 45 types, of which 13 are traits and 3 are interfaces [koel].
  - Snipe-IT's `App\Models` subtree is 162 types, only 43 of them Eloquent models. The others include 24 Eloquent builders in `App\Models\Builders`, label classes and scopes (`App\Models\CompanyableScope`) [snipeit] `app/Models/`.
  - osu-web's `App\Models` subtree is 224 types, of which 189 are models and 17 are traits [osuweb].
  - Pinkary's own arch test has to carve out `App\Models\Concerns` and `App\Models\Scopes` before asserting that `App\Models` extends `Model` [pinkary] `tests/Arch/ModelsTest.php`.
- **App-specific roles are marked only by their folder.** Examples are Koel's `App\Values` (117 types), `App\Services` (144) and `App\Builders` [koel], osu-web's `App\Libraries` (213) [osuweb], and Pixelfed's `App\Util` (63) and `App\Services` (108) [pixelfed]. The suffix does not help: only 53 of the 148 types in Koel's `Services` namespaces are named `*Service`. Examples of the rest are `App\Services\MediaBrowser` and `App\Services\LibraryManager` [koel].
- **`Jobs` does not mean queued, and queued does not mean `Jobs`:**
  - Laravel.io has 42 classes in `App\Jobs`, but only 9 implement `ShouldQueue`. The rest are synchronous commands such as `final class CreateThread`, dispatched with `$this->dispatchSync(...)` from controllers [laravelio] `app/Jobs/CreateThread.php`, `app/Http/Controllers/ReplyController.php`. The Laravel docs anticipate this: synchronous jobs "are sometimes referred to as 'commands'" [L-structure].
  - Snipe-IT's 25 `ShouldQueue` classes are all notifications in `App\Notifications` [snipeit].

Roles recognisable only by base class, interface, attribute or suffix:

- **Controllers.** Koel's base `App\Http\Controllers\Controller` is `abstract class Controller` with no parent, following the skeleton [koel] `app/Http/Controllers/Controller.php`. So nothing in a controller's ancestry says "controller" except the app's own base class. Controllers are recognisable by namespace, suffix or route registration.
- **Models through vendor bases.** `App\Models\PersonalAccessToken` extends `Laravel\Sanctum\PersonalAccessToken`, and `App\Models\Passkey` extends `Laravel\Passkeys\Passkey` [koel] `app/Models/`. They are models only if vendor ancestry is resolved. The scanner used here missed both.
- **Attributes configure roles but rarely identify them.** Koel uses Eloquent class attributes (`#[ObservedBy]` ×9, `#[Guarded]`, `#[Unguarded]`, `#[UseEloquentBuilder]`, `#[Table]`) and its own `App\Attributes\RequiresPlus` (×19) [koel]. The skeleton's `User` uses `#[Fillable]` and `#[Hidden]` [skeleton].

What a declaration would need: namespace prefixes rooted at the app's PSR-4 root; exclusion of sub-namespaces or a kind filter (classes only, not traits or interfaces); and, for framework roles, an optional ancestry check to catch strays. This family is the case where "namespace = layer" mostly works.

### 2. Action-based

Apps that put use-case logic in single-purpose action classes, either plain or via `lorisleiva/laravel-actions`. Examples are Coolify, Lychee, Vito, Pinkary, Snipe-IT (partly) and Akaunting ("Jobs" used as actions).

Coolify, abridged [coolify] `app/`, `bootstrap/`:

```
app/                       932 types, all under App\
  Livewire/      250       244 Livewire components in 61 namespaces (Project/ 113, Server/ 47, ...)
  Actions/        90       Server/, Database/, Proxy/, Application/, Service/, Docker/, Stripe/, Fortify/ ...
  Http/           82       44 controllers
  Models/         69       68 Eloquent models
  Jobs/           64
  Mcp/            55
  Notifications/  49
  Services/44  Console/42  Traits/29  Policies/28  Events/25  Support/22 ...
bootstrap/helpers/*.php    22 files, 438 global functions, loaded via composer "files"
```

Where classes live, and whether namespace tracks role:

- **Actions are recognisable in three different ways:**
  - **By trait.** In Coolify, 71 of the 91 classes in `Actions` namespaces `use Lorisleiva\Actions\Concerns\AsAction`, for example `App\Actions\Server\StartSentinel` [coolify] `app/Actions/Server/StartSentinel.php`.
  - **By nothing but the namespace.** Lychee has 253 classes in `App\Actions\...`, such as `App\Actions\Album\Create`, a plain `class Create` with no base class or trait, and only 1 of them has an `Action` suffix [lychee] `app/Actions/Album/Create.php`. Vito has 237 types in `App\Actions` with no marker [vito], and Pinkary has 42 [pinkary].
  - **By suffix.** Snipe-IT names 15 of its 17 action classes `*Action`, for example `App\Actions\Suppliers\DestroySupplierAction` [snipeit].
- **One class can hold several roles.** A lorisleiva action is written "as anything you want": `asController`, `asJob`, `asListener`, `asCommand` [actions] `README.md`. In Coolify, `App\Actions\Database\StartDatabase` and `RestartDatabase` also define `asJob` [coolify].
- **Actions may live under a framework role's folder.** Akaunting keeps 92 classes in `App\Jobs`, but only 4 are queued. `App\Jobs\Banking\CreateTransaction extends Job implements HasOwner, HasSource, ShouldCreate` is an action [akaunting] `app/Jobs/Banking/CreateTransaction.php`. None of the 88 classes in Akaunting's `Controllers` namespaces has a `Controller` suffix either, for example `App\Http\Controllers\Banking\Accounts extends Controller` [akaunting] `app/Http/Controllers/Banking/Accounts.php`.
- **The HTTP entry layer is often Livewire, not controllers.** Coolify has 244 Livewire components against 44 controllers [coolify]. Livewire 4's default component format is single-file. Such a component lives in `resources/views/components/...` as `new class extends Component { ... }` inside a Blade file [LW-components]. Laravel.cm already ships these, for example `resources/views/livewire/components/discussion/top-contributors.blade.php`, an anonymous class that queries `App\Models\User` [laravelcm]. Those components are classes outside `app/` and have no name.
- **Dependencies flow through global functions.** Coolify's `bootstrap/includeHelpers.php` requires every `bootstrap/helpers/*.php`. Those files define 438 functions, import `App\Models\*`, and are called from `app/`: `instant_remote_process(` appears in 96 files [coolify] `bootstrap/helpers/remoteProcess.php`, `bootstrap/helpers/shared.php`.

Roles recognisable only by base class, interface, attribute or suffix: lorisleiva actions (trait), Livewire components (`Livewire\Component`), queued work (`ShouldQueue`), and MCP tools. Coolify keeps MCP tools in `App\Mcp`, and Leantime has 57 subclasses of `Laravel\Mcp\Server\Tool` spread across domain `Tools` folders [coolify], [leantime].

What a declaration would need: trait-based selectors, plus "plain class in this namespace" selectors for unmarked actions. It also needs a way to say one class plays several roles. And it needs a stance on code outside classes: functions, Blade-embedded classes and route closures. Those code units could be graph nodes, explicitly "unknown" edges, or out of scope.

### 3. Domain-oriented

Apps whose first namespace segment below the root is a business domain, with roles nested inside or mixed flat. Examples are BookStack, Monica, Leantime, the "Laravel Beyond CRUD" split (seen in VueFileManager) and the defaults of `lunarstorm/laravel-ddd`.

BookStack, abridged; the root namespace is `BookStack\`, not `App\` [bookstack] `composer.json`, `app/`:

```
app/                        502 types under BookStack\
  Entities/      85         Controllers/ (12), Models/ (Book, Page, ... + interfaces/traits), Queries/, Repos/, Tools/
  Access/        54         Controllers/, Guards/, Mfa/, Oidc/, LoginService, LdapService, SocialAccount (a model) ...
  Activity/      54         Controllers/, Models/, Notifications/, DispatchWebhookJob ...
  View/42  Exceptions/32  Exports/30  Util/23  Uploads/18  Console/18  App/17 (incl. App/Providers/)
  Users/17  Search/14  Permissions/14  References/15  Settings/9  Sorting/9  Theming/9 ...
  Settings/   flat: Setting.php (model), SettingController.php, SettingService.php, MaintenanceController.php ...
```

Monica, abridged [monica] `app/Domains/`:

```
app/Domains/                       582 of the app's 746 types
  Contact/ManageContact/Services/CreateContact.php          App\Domains\Contact\ManageContact\Services\CreateContact
  Contact/ManageContact/Web/Controllers/ContactController.php
  Contact/ManageContact/Web/ViewHelpers/ContactShowViewHelper.php
  Contact/ManageContact/Dav/ImportContact.php
  Contact/ManageCalls/{Services,Web}  Contact/ManageDocuments/{Events,Listeners,Services,Web} ...
  Settings/...  Vault/...
app/Models/                        68 of 70 Eloquent models stay central
```

Where classes live, and whether namespace tracks role:

- **BookStack: the namespace tracks the domain; the role is sometimes a sub-namespace and sometimes absent:**
  - 74 controllers are spread over 16 top-level folders. 55 are in a `Controllers` sub-namespace, and 19 sit flat beside models and services: `BookStack\Settings\SettingController`, `BookStack\App\HomeController`, `BookStack\Search\SearchController` [bookstack].
  - 37 Eloquent models live in 14 namespaces, 11 of them outside any `Models` segment. Examples are `BookStack\Settings\Setting extends Model`, `BookStack\Access\SocialAccount`, `BookStack\Uploads\Attachment` and the base `BookStack\App\Model` [bookstack] `app/Settings/Setting.php`, `app/App/Model.php`.
  - 18 `*Service` classes exist and none is in a `Services` namespace (for example `BookStack\Access\LoginService`). `ShouldQueue` work sits in domain folders, for example `BookStack\Activity\DispatchWebhookJob implements ShouldQueue` [bookstack] `app/Activity/DispatchWebhookJob.php`.
  - Service providers are in `BookStack\App\Providers` [bookstack].
- **A role-folder name can mean something else.** `BookStack\Exports\ZipExports\Models\*` are export data shapes (`final class ZipExportBook extends ZipExportModel`), not Eloquent. Of the 45 types in BookStack's `Models` segments, 26 are Eloquent models [bookstack] `app/Exports/ZipExports/Models/ZipExportBook.php`.
- **Monica: the role sits 4 to 5 segments deep, under Area and Feature.** An example is `App\Domains\Contact\ManageContact\Web\Controllers\ContactController`, which extends `App\Http\Controllers\Controller`. HTTP code lives in `Web`, not `Http`.
- **Monica's services are recognisable by base class and interface, not name.** `CreateContact extends BaseService implements ServiceInterface` [monica] `app/Domains/Contact/ManageContact/Services/CreateContact.php`. 263 types sit in `Services` namespaces, and only 2 are named `*Service` [monica].
- **Leantime: role segments are present, but names repeat and the framework roles differ:**
  - The pattern is `Leantime\Domain\<Module>\{Controllers,Hxcontrollers,Services,Repositories,Models,...}` across 45 domain folders. `Leantime\Domain\Tickets\Models\Tickets`, `...\Repositories\Tickets` and `...\Services\Tickets` share a short name [leantime] `app/Domain/Tickets/`.
  - None of the 160 classes in `Controllers` namespaces has a `Controller` suffix. They extend the app's own `Leantime\Core\Controller\Controller` [leantime] `app/Domain/Tickets/Controllers/ShowAll.php`.
  - The app has **zero** Eloquent models. Its `Models` are plain classes, so "model" does not mean Eloquent in every Laravel app [leantime].
- **The textbook split is rare in maintained public apps.** VueFileManager's `composer.json` maps `"App\\": "src/App/"`, `"Domain\\": "src/Domain/"` and `"Support\\": "src/Support/"`. That is the Beyond CRUD split.
  - VueFileManager still puts its 126 controllers in `Domain\*\Controllers`, not the application layer [vfm] `composer.json`, `src/Domain/`.
  - Its last commit is 2023-10-07, on Laravel `^9.2`, so it is not maintained.
  - `lunarstorm/laravel-ddd` defaults to `domain_path => 'src/Domain'` and `domain_namespace => 'Domain'`. Its application layer is `App\Modules` holding `controller`, `request` and `middleware`, and its role-to-sub-namespace table (`model => Models`, `data_transfer_object => Data`, `action => Actions`, ...) is configurable [ddd] `config/ddd.php`.
  - GitHub code search for `src/Domain` autoload mappings turned up demos, course projects and unmaintained apps, but no maintained, sizeable app.
  - The book's own text could not be read: the stitcher.io chapter now redirects to a host that failed TLS. The attribution of the split to the book is therefore not verified here.

Roles recognisable only by base class, interface, attribute or suffix: models and controllers in flat domain folders (BookStack); services via `BaseService` or `ServiceInterface` (Monica); controllers via an app-owned base class with no suffix (Leantime); permission attributes such as `Leantime\Core\Auth\Permissions\RequiresPermission` (Leantime).

What a declaration would need: namespace patterns with a wildcard or capture for the domain segment, such as `BookStack\{domain}\Controllers` or `App\Domains\{area}\{feature}\Web\Controllers`. Inside flat folders, role selectors by ancestry and suffix. Role names must be configurable (`Web` vs `Http`, `Repos` vs `Repositories`). A "domain" needs to be a first-class grouping, distinct from a role.

### 4. Module packages

Apps split into modules, each with its own namespace root, usually a Composer package. Examples are Advising App and Laravel.cm (InterNACHI/modular), ColoredCow portal (nWidart/laravel-modules) and Bagisto (in-repo packages wired by `konekt/concord`). Akaunting, FreeScout, Laradashboard and solidtime also use module packages, but as plugin systems with no or few modules committed in the repo.

What the packages prescribe:

- **InterNACHI/modular.** Modules live in `app-modules/<name>/` with `composer.json`, `src/`, `tests/`, `routes/`, `resources/` and `database/`. They are loaded as Composer path repositories plus package discovery. The default namespace is `Modules\`, and the docs recommend an organisation namespace instead. Modules "follow existing Laravel conventions", and most `make:` commands gain `--module=` [modular] `README.md`, `config/app-modules.php`. Nothing in the README describes public versus internal classes.
- **nWidart/laravel-modules v13.**
  - The default namespace is `Modules`, with `app_folder => 'app/'`. A generator table maps each role to a folder, for example `model => app/Models`, `controller => app/Http/Controllers`, `resource => app/Transformers`, `emails => app/Emails`, `scopes => app/Models/Scopes` [nwidart] `config/config.php`.
  - The module `composer.json` stub maps `$MODULE_NAMESPACE$\$STUDLY_NAME$\` to the app folder [nwidart] `src/Commands/stubs/composer.stub`.
  - Modules can be switched on and off at runtime through an activator such as `modules_statuses.json` [nwidart] `config/config.php`.
  - InterNACHI's README summarises the difference: nwidart has "its own directory structure", and suits apps with "3rd-party modules that can be dynamically enabled and disabled" [modular] `README.md`.

Advising App, abridged [advising] `composer.json`, `config/app-modules.php`, `app-modules/`:

```
app/                          348 types under App\  (default skeleton core: Models, Filament, Http, Jobs, ...)
app-modules/  37 modules      1,724 types under AdvisingApp\<Module>\
  task/composer.json          psr-4 "AdvisingApp\\Task\\": "src/", provider AdvisingApp\Task\Providers\TaskServiceProvider
  task/src/{Enums,Filament/{Actions,Concerns,RelationManagers,Resources},Histories,Models/Scopes,Notifications,Observers,Policies,Providers}
  prospect/src/{Actions,Concerns,DataTransferObjects,Enums,Filament,Http,Imports,Models,Observers,Policies,Providers,Sorts}
config/app-modules.php        'modules_namespace' => 'AdvisingApp', 'modules_vendor' => 'canyon-gbs'
composer.json                 repositories: { type: path, url: "app-modules/*" }
```

Bagisto, abridged [bagisto] `composer.json`, `config/concord.php`, `packages/Webkul/`:

```
app/                          3 PHP files
packages/Webkul/  42 packages 1,236 types under Webkul\<Package>\, each mapped in the ROOT composer.json psr-4
  Admin/src/Http/Controllers/{Sales,Catalog,Customers,...}   79 controllers: the admin UI for every domain
  Shop/src/Http/Controllers/...                               38 controllers: the storefront for every domain
  Sales/src/{Contracts,Models,Repositories,Providers,Transformers,Generators,...}
    Models/Order.php          class Order extends Model implements OrderContract
    Models/OrderProxy.php     class OrderProxy extends ModelProxy {}
    Contracts/Order.php       interface Order {}
config/concord.php            'modules' => [ ...\Providers\ModuleServiceProvider::class, ... ]
```

ColoredCow portal, the one sizeable public app with real in-repo nWidart modules found [coloredcow] `composer.json`, `Modules/`:

- It runs on Laravel `^8.0` with `nwidart/laravel-modules ^7.0`, and has 18 module folders mapped by `"Modules\\": "Modules/"`.
- It uses the pre-v11 convention: models in `Entities`, mail in `Emails`. Of its 108 Eloquent models, 87 are in `Modules\*\Entities` and only 21 in a `Models` namespace (`App\Models`).

Where classes live, and whether namespace tracks role:

- **The role is the third segment: `<Root>\<Module>\<Role>\...`.** For example `AdvisingApp\Task\Models\Task`, `Webkul\Sales\Repositories\OrderRepository` and `Modules\HR\Http\Controllers\...`.
- **Modules are not boundaries in practice.** `AdvisingApp\Task\Models\Task` imports `AdvisingApp\Prospect\Models\Prospect` and `AdvisingApp\StudentDataModel\Models\Student` [advising] `app-modules/task/src/Models/Task.php`. The engagement module has 37 imports of `AdvisingApp\StudentDataModel\Models\...` and 29 of `AdvisingApp\Prospect\Models\...` [advising] `app-modules/engagement/src/`. In Bagisto, `Webkul\Shop`, `Webkul\User`, `Webkul\Theme` and `Webkul\Product` import `Webkul\Admin\...` [bagisto].
- **A package can be a layer, not a domain.** Bagisto's `Admin` and `Shop` packages hold controllers for every domain and call domain repositories, for example `Webkul\Admin\Http\Controllers\Sales\OrderController` uses `Webkul\Sales\Repositories\OrderRepository` [bagisto] `packages/Webkul/Admin/src/Http/Controllers/Sales/OrderController.php`.
- **The `Models` folder holds non-models.** Bagisto's `Models` segments hold 253 types: 126 Eloquent models, 119 concord `*Proxy` classes and 8 enums [bagisto].
- **Membership is declared in code, not derived.** Bagisto lists its modules in `config/concord.php`, and each `ModuleServiceProvider` lists its models in `protected $models = [...]` [bagisto] `packages/Webkul/Sales/src/Providers/ModuleServiceProvider.php`. Advising App derives modules from `app-modules/*/composer.json`, and its own arch test reads them through `InterNACHI\Modular\Support\ModuleRegistry` [advising] `tests/Tenant/Unit/ArchTest.php`.

Roles recognisable only by base class, interface, attribute or suffix:

- Filament classes, recognised by ancestry: 747 classes in Advising App, across 287 namespaces, extend, implement or use a `Filament\` class, interface or trait. The top bases are `Filament\Resources\Resource` (53), `Filament\Resources\Pages\EditRecord` (53) and `ListRecords` (47).
- `Spatie\LaravelData\Data` (53) and `Spatie\LaravelSettings\Settings` (31) subclasses [advising].
- Bagisto's marker interfaces (`Webkul\Sales\Contracts\Order`) and `ModelProxy` subclasses [bagisto].
- Advising App's 51 `#[ObservedBy]` and 14 `#[App\Models\Attributes\NoPermissions]` attributes [advising].

What a declaration would need:

- A module defined by a namespace capture (`AdvisingApp\{module}\...`) or by Composer package (path repository or root PSR-4 entry), or listed explicitly.
- Multiple source roots: `app-modules/*/src`, `packages/*/*/src`, `Modules/*`.
- Role selectors that work within each module.
- An explicit way to say what is module-internal, since no package supplies one.
- A module that is a layer (Bagisto's `Admin`) must be expressible as well as a module that is a domain.

### 5. Large, long-lived monoliths

Big apps that have absorbed several eras of conventions. Examples are Firefly III (repo created 2014), osu-web, Pixelfed, Snipe-IT, Pterodactyl, Akaunting and FreeScout.

Firefly III, abridged; the root namespace is `FireflyIII\` [firefly] `composer.json`, `app/`:

```
app/                          1,217 types under FireflyIII\
  Api/V1/       228           Controllers/ (142 controllers), Requests/ (85 form requests), Middleware/
  Http/         184           Controllers/ (127 controllers), Requests/ (38), Middleware/
  Support/      150           Amount, Steam, Navigation, Models/ (calculators), Repositories/, Http/, Search/ ...
  Console/88  Repositories/69 (interface + implementation pairs)  Models/50  Events/50  Listeners/38
  TransactionRules/38  View/38  Rules/34  Notifications/32  Services/32  Transformers/26  Generator/24
  Handlers/Observer/          BillObserver, AutoBudgetObserver, ... (observers outside Observers)
  User.php                    FireflyIII\User extends Authenticatable (a model at the root)
composer.json                 "Domain\\": "domain/" is mapped, but no domain/ directory exists at this commit
```

Where classes live, and whether namespace tracks role:

- **The root namespace is often not `App`.** It is `FireflyIII\`, and also `BookStack\`, `Pterodactyl\`, `Leantime\`, `Webkul\`, `AdvisingApp\` and `Modules\` [firefly], [bookstack], [pterodactyl], [leantime], [bagisto], [advising], [coloredcow].
- **One role sits in several trees.** Firefly's controllers live in `FireflyIII\Api\V1\Controllers` (142) and `FireflyIII\Http\Controllers` (127). Its form requests live in `Api\V1\Requests` (85) and `Http\Requests` (38) [firefly].
- **Segment names are reused for other roles.** Firefly's `Models` segments hold 194 types, but only 50 are Eloquent models. The others include `FireflyIII\Support\Models\BillDateCalculator` and `FireflyIII\Api\V1\Requests\Models\TransactionCurrency\StoreRequest`. The `User` model sits at the root, and observers live in `FireflyIII\Handlers\Observer` [firefly] `app/Support/Models/BillDateCalculator.php`, `app/User.php`.
- **Grab-bag namespaces grow large:** Firefly's `Support` (150 types), osu-web's `Libraries` (213), Pixelfed's `Util` (63) [firefly], [osuweb], [pixelfed].
- **Contracts are wired in providers.** `FireflyIII\Repositories\Account\AccountRepositoryInterface` is bound in `AccountServiceProvider` with `$this->app->bind(static function (Application $app): AccountRepositoryInterface { ... })` [firefly] `app/Providers/AccountServiceProvider.php`.
- **Vendor code is overridden from inside the repo:**
  - FreeScout pins `laravel/framework v5.5.40` and maps 124 vendor namespace prefixes to `overrides/`, including `Illuminate\Foundation\`, `Illuminate\Database\Eloquent\` and `Symfony\Component\Console\` [freescout] `composer.json`.
  - Akaunting maps `Illuminate\Translation\`, `Livewire\`, `Maatwebsite\Excel\` and `Symfony\Component\Process\` to `overrides/` [akaunting] `composer.json`.
  - Advising App loads `overrides/OpenAI/...` through `files` and `exclude-from-classmap` [advising] `composer.json`.
  - So a namespace prefix does not say whether code is the app's or a vendor's; the file path does.

What a declaration would need: configurable roots read from `composer.json`, not an assumed `App\`. Patterns that can match several subtrees for one role. Per-class exceptions. And for contract rules, knowledge of container bindings, which are registered in closures.

## Namespace vs role, side by side

Counts come from the scanner described under Method. "In segment" means the class's namespace contains that segment anywhere (for example `...\Models\...`).

| App | Eloquent models: in a `Models` segment / total | Types in `Models` segments / of which models | `*Controller` classes / in a `Controllers` segment | Actions: types in `Actions` / with a marker |
| --- | --- | --- | --- | --- |
| Koel | 27 / 27 (+2 via vendor bases) | 45 / 27 (+2) | 174 / 174 | none |
| Coolify | 68 / 68 | 69 / 68 | 44 / 44 | 91 / 71 (trait) |
| BookStack | 26 / 37 | 45 / 26 | 74 / 55 | none |
| Advising App | 201 / 210 | 268 / 201 | 89 / 89 | 153 / 52 (suffix) |
| Bagisto | 126 / 127 | 253 / 126 | 132 / 132 | none |
| Firefly III | 50 / 51 | 194 / 50 | 269 / 269 | 31 / 0 (`TransactionRules\Actions`) |
| Lychee | 41 / 42 (one in `App\Http\Controllers\Admin\Maintenance\Model`) | 137 / 41 | 99 / 99 (+23 unsuffixed) | 261 / 1 |
| Snipe-IT | 43 / 43 | 162 / 43 | 110 / 110 | 17 / 15 (suffix) |
| Akaunting | 33 / 34 | 35 / 33 | 0 (88 unsuffixed in `Controllers`) | n/a (92 action-like classes in `Jobs`) |
| Leantime | 0 (no Eloquent) | 38 / 0 | 0 (160 unsuffixed in `Controllers`) | none |
| ColoredCow | 21 / 108 (87 in `Entities`) | 21 / 21 | 94 / 94 | none |

Sources for the table: [koel], [coolify], [bookstack], [advising], [bagisto], [firefly], [lychee], [snipeit], [akaunting], [leantime], [coloredcow] at the commits listed. Lychee's stray model is `app/Http/Controllers/Admin/Maintenance/Model/Album.php` (`class Album extends Model implements Node`) [lychee].

The pattern:

- Framework roles mostly sit in their conventional folder in default-skeleton, module and monolith apps.
- They do not in domain-oriented apps (BookStack), where a module package renames folders (ColoredCow's `Entities`), or where an app uses its own naming (Akaunting's and Leantime's controllers).
- A conventional folder name rarely means only that role.
- App-specific roles (actions, services, values) have no reliable marker other than their folder.

## What a declaration would need, across families

These are the requirements the evidence imposes on whatever the architecture spec ends up being. They are not proposals for its format.

- **Two independent questions per class.**
  - *Where does it belong?* A layer, domain or module, or a module that is itself a layer.
  - *What is it?* A controller, model, action, Livewire component, and so on.
  - In default-skeleton apps the namespace answers both. In BookStack the namespace answers the first only. In Advising App and Bagisto, different segments of the namespace answer each.
- **Selector kinds that real apps need**, each with at least one app where nothing else works:
  - **Namespace pattern with a wildcard or capture for a segment:** `AdvisingApp\{module}\Models`, `BookStack\{domain}\Controllers`, `App\Domains\{area}\{feature}\Web\Controllers` [advising], [bookstack], [monica].
  - **Kind filter** (class, interface, trait, enum, abstract), because role folders contain traits and interfaces [koel], [snipeit], [pinkary].
  - **Ancestry, transitive and including vendor classes:** `Model` in flat folders [bookstack], vendor model bases [koel], `Livewire\Component` [coolify], Filament bases [advising], `ModelProxy` to exclude [bagisto].
  - **Trait use:** `AsAction` [coolify]. Pest's `using()` relies on `class_uses`, which does not include traits used by parent classes [pest-arch], [php-class-uses].
  - **Interface:** `ShouldQueue`, marker contracts, `ServiceInterface` [snipeit], [bagisto], [monica].
  - **Attribute**, where an app uses one as a role marker: `#[RequiresPermission]`, `#[NoPermissions]` [leantime], [advising].
  - **Name suffix or pattern**, which is reliable in some apps (Koel's `*Controller`) and absent in others (Akaunting, Leantime) [koel], [akaunting], [leantime].
  - **Explicit class lists and exceptions**, for the strays found in every app above, such as Firefly's root `User` and Lychee's model under `Controllers`.
- **Roles are not exclusive.** A lorisleiva action can be controller, job, listener and command at once [actions], and a notification can implement `ShouldQueue` [snipeit]. Whether a class may carry several roles while belonging to exactly one location is a modelling choice the evidence forces.
- **Source roots come from `composer.json`, not from `app/`.** Real roots include `app-modules/*/src` (path repositories), `packages/Webkul/*/src` (root PSR-4), `Modules/` and `src/`. Files loaded via `"files"` contain functions [advising], [bagisto], [coloredcow], [vfm], [coolify]. Autoload maps can name directories that no longer exist [firefly], and can point vendor namespaces at app-owned files [freescout], [akaunting].
- **Some code is not a named PSR-4 class:**
  - global functions (Coolify, 438);
  - anonymous Livewire 4 components in Blade (Laravel.cm);
  - anonymous migration classes;
  - route closures.

  A class-level architecture spec must either include these or report them as outside its view [coolify], [laravelcm], [LW-components].
- **Ancestry needs vendor knowledge.** Recognising Koel's `Passkey` as a model, or Advising App's Filament pages as Filament pages, means resolving vendor classes. The scanner would need either an installed `vendor/` (as Pest's runtime filters do) or bundled knowledge of framework and package bases [koel], [pest-arch].
- **"Module-internal" has to be declared.** Neither InterNACHI/modular nor nWidart/laravel-modules documents a visibility concept, and Advising App's modules import each other's models freely [modular], [nwidart], [advising].

How this bears on the WIP draft's assumptions (each still an open question):

- **"namespace = layer".** This works as *namespace pattern = location* if patterns can wildcard the module or domain segment. It does not deliver *namespace = role* outside default-skeleton apps.
- **"Every class under `app/` must belong to a layer".** This misses module roots, `src/`, helper files and Blade-embedded classes.
- **`Platform: App\Providers`.** Providers live in `BookStack\App\Providers`, and Advising App has 50 providers spread over 38 namespaces, one or more per module [bookstack], [advising].

## Reference app set

Chosen to cover the spread: one clean default-skeleton app, and one strong representative of each other family. Every app is public, maintained (commits in October 2026), mid-size or large, on Laravel 12 or 13, and each one stresses a different selector. "Size" is type declarations under the source roots, counted as in Method.

| App | Repo | Pinned commit | Size | Family | Laravel | Why it is in the set |
| --- | --- | --- | --- | --- | --- | --- |
| Koel | [koel/koel](https://github.com/koel/koel/tree/7c34114c6e8d099278c732b85853f720c2546d82) | `7c34114c6e8d099278c732b85853f720c2546d82` | ~1,040 (`app/`) | Default skeleton | `^13.0` | **Best case for namespace = role.** It is large and clean, so a spec that is verbose here is too verbose. Its base controller has no parent class (Laravel 11+ style). It has app-specific roles (`Values`, `Services`, `Repositories`, `Builders`), impure `Models`, and models with vendor ancestry. MIT licence. |
| Coolify | [coollabsio/coolify](https://github.com/coollabsio/coolify/tree/6964714b5da26d600621eb137c9845e6240f7d52) | `6964714b5da26d600621eb137c9845e6240f7d52` | ~932 (`app/`) + 438 functions | Action-based (lorisleiva) and Livewire-first | `12.69.3` | Its roles are marked by a trait (`AsAction`). Livewire components outnumber controllers more than five to one, some actions double as jobs, global helper functions carry real dependencies, and it has MCP tools. Apache-2.0 licence. |
| BookStack | [BookStackApp/BookStack](https://github.com/BookStackApp/BookStack/tree/d3db80004b51a395a9dabdd406f5fbda3d4d76e4) | `d3db80004b51a395a9dabdd406f5fbda3d4d76e4` | ~502 (`app/`) | Domain-oriented | `^v13.35.0` | **The namespace is the domain, not the role.** Roles appear as sub-namespaces in some domains and flat in others. It has a `BookStack\` root, models found only by base class, and a `Models` folder holding non-Eloquent DTOs. MIT licence. |
| Advising App | [canyongbs/advisingapp](https://github.com/canyongbs/advisingapp/tree/d945da2b74325e5ecdbc3cc7210a46768ea4fe1e) | `d945da2b74325e5ecdbc3cc7210a46768ea4fe1e` | ~2,072 (348 in `app/`, 1,724 in `app-modules/*/src`) | Module packages (InterNACHI/modular) | `^13.35` | It has 37 modules as Composer path packages with a custom module namespace, plus a default-skeleton core. Cross-module coupling is heavy. It is Filament-heavy (747 classes with Filament ancestry), uses Spatie Data and Settings, and is the largest app in the set. **Licence: Elastic License 2.0 (source-available)**; reading and testing against it is fine, but copying its code into fixtures needs a check. |
| Bagisto | [bagisto/bagisto](https://github.com/bagisto/bagisto/tree/748d3ea117dcf121bfbf57a4d8d996c81144510f) | `748d3ea117dcf121bfbf57a4d8d996c81144510f` | ~1,239 (3 in `app/`, 1,236 in `packages/Webkul/*/src`) | Module packages (in-repo packages, konekt/concord) | `^13.0` | **Its modules are both layers (`Admin`, `Shop`) and domains (`Sales`, `Product`).** Membership is declared in config and providers, every model has a marker interface and a proxy, there are repository classes, and almost nothing lives in `app/`. MIT licence. |
| Firefly III | [firefly-iii/firefly-iii](https://github.com/firefly-iii/firefly-iii/tree/b9c31006e8d3008df3d12e23de86b66088c9480e) | `b9c31006e8d3008df3d12e23de86b66088c9480e` | ~1,217 (`app/`) | Large long-lived monolith | `^13` | Its repo dates from 2014 and its root is `FireflyIII\`. One role is split across two trees (`Api\V1` and `Http`), a `Models` segment is reused for non-models, there is a big `Support` grab-bag, and repository interface/implementation pairs are bound in providers. There is also a model at the root and a stale autoload entry. AGPL-3.0 licence. |

Commit dates: Koel 2026-10-16, Coolify 2026-10-16, BookStack 2026-10-15, Advising App 2026-10-16, Bagisto 2026-10-16, Firefly III 2026-10-17. Laravel constraints are from each app's `composer.json` at the pinned commit. To reproduce: `git clone <repo> && git checkout <sha>`, then read only.

**Alternates, if a later ticket needs a specific stressor:**

- **ColoredCow portal** (`b6d806748a8015cbbcc055fbb5333dff85fa9fab`, ~518 types in `app/` and `Modules/`). This is the in-repo nWidart case, with `Entities` and `Emails` naming. It is on Laravel 8, so it was not chosen. Shape-wise, nWidart v11+ (`Modules\X\Http\Controllers` from `Modules/X/app/`) matches what Advising App already exercises.
- **Lychee** (`89a42fc32613224400033e304f151f388d4175fc`, ~1,647 types). It has 253 unmarked action classes, 200 Spatie Data classes acting as HTTP resources, and an Eloquent model under `Controllers`.
- **Leantime** (`9d09091fefc85fb910a145a7cb9c56d03fd9dda0`, ~689 types). It is domain-oriented with no Eloquent and no suffixes, and its controllers extend an app-owned base class.
- **Pinkary** (`f4a528d4e5f46045aa3175d117615c0e1c14270b`, ~293 types). It is small, but carries 17 Pest arch test files (`tests/ArchTest.php` plus 16 in `tests/Arch/`). That makes it a ready target for "can the architecture spec express what this app already enforces?" [pinkary]. Vito has a similar arch suite with its own exemption list (`ArchTestCase::except(...)`) [vito] `tests/Arch/ConventionsTest.php`.
- **Monica** (`6e6ec217457e946e330ba7cd3b28107f4332c7b1`, ~746 types). It has the deepest domain slicing (`App\Domains\Area\Feature\{Services,Web}`), but only sporadic commits since August 2025.

None of the 23 surveyed apps uses Deptrac, PHPArkitect or phpat; no config file or Composer requirement for them was found. Pinkary, Vito, Advising App and Coolify (a single file) have Pest arch tests.

## Open questions surfaced

1. **One axis or two?** Does the architecture spec separate *location* (layer, domain or module) from *role* (controller, model, action), or treat role as a sub-layer? The evidence shows the two vary independently.
2. **Which selector kinds does the first release support?** The candidates are namespace pattern with captures, path, kind, ancestry, trait, interface, attribute, name pattern and explicit lists. Ancestry and trait need vendor resolution: does the scanner require an installed `vendor/` and reflection, as Pest does, or parse vendor code statically?
3. **Can a class carry several roles** (lorisleiva actions, queued notifications)? If so, which one governs its allowed dependencies?
4. **How is a module identified** (namespace capture, Composer package, or declared list)? Does the architecture spec define a module's public surface versus module-internal classes, given that no module package does and real apps cross-import freely?
5. **What happens to code that is not a named class:** global helper functions, anonymous Livewire 4 components in Blade, route closures, anonymous migrations? Are they nodes, "unknown", or out of scope?
6. **Are source roots read from `composer.json`** (PSR-4, path repositories, merge-plugin includes, `files`) or declared? And how are vendor namespaces overridden from inside the repo treated?
7. **Is "controller" defined as "route target"?** If so, it needs route files, statically parsed or read from a booted app. This ties into the "Laravel container knowledge" item under "Not yet specified" in map issue #1, as do Firefly III's closure-registered bindings.
8. **Supported Laravel versions.** The reference set is on Laravel 12–13, while real long-lived apps run on 8 (ColoredCow) or 5.5 (FreeScout). Does the first release care?
9. **Licensing of fixtures.** Advising App is source-available (Elastic License 2.0). Can fixtures derived from reference apps be committed, or should tests clone at the pinned SHAs?
10. **Glossary gaps.** "Layer", "role", "module" and "domain" are used throughout this research and are not yet in `GLOSSARY.md`; they need resolving (via `domain-modeling`) before the spec format is designed.
