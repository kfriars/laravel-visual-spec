# Kill-condition recount: which kind of reference does each class-level row need?

Research for "What counts as a dependency?" ([#10](https://github.com/kfriars/laravel-visual-spec/issues/10)) on the map [#1](https://github.com/kfriars/laravel-visual-spec/issues/1). Written 2026-10-18. It decides nothing.

[research/mistakes-that-matter](https://github.com/kfriars/laravel-visual-spec/blob/research/mistakes-that-matter/docs/research/mistakes-that-matter.md) counted global functions as class-level references. The spec leaves functions out. This recount reclassifies every class-level row there by the kind of reference a spec rule would need to see.

It was done under a names-only candidate definition. The definition #10 settled on also counts docblocks and inferred types. No row here needed either, and inference only adds edges, so these figures are a **floor**. Inference could still lift some method-body rows into edges, or make some rules less realistic by adding edges to their peers. Neither effect is measured here.

Input: `docs/research/mistakes-that-matter.md` on `research/mistakes-that-matter`.
Scope: 59 C1 + 36 C2 commits, 12 C3 commits in buckets A/B/P, 91 C1+C2 hand-written rules.

## Method and judgement rules

- Worked from the table rows. I opened the diff (`gh api repos/<o>/<r>/commits/<sha> -H 'Accept: application/vnd.github.diff'`) for every row that mentioned helpers, `auth()`, `app()`, `env()`, `config()`, aliases, `\DB::`, `use Cache;` and similar, traits, or functions. Where a diff showed a facade call with no import in the hunk, I read the file's imports at the parent commit (`contents?ref=<parent>`). I read the rule test files at the pinned SHAs for every function, trait, Throwable and migration rule.
- Diffs read: Koel 3e747101a9, 27933d49cf, 47c36f289e, 4fc5117509, 8f7654a220 (plus Song.php at parent), a837432a4e, bf1d158c90, b227ece517, 351efe4092, 9ab2bae1da, 1d5c8e84b6; Coolify 3e0d48faea, 0ef386b4a8, e891318396, d27070b215, cbee84353e, 469e404725, 8c5c249c6a, ff3d0b29e3, e74c464857; BookStack c157dc3490 (plus 6 controllers at parent), 9d7ce59b18, 5b0cb3dd50, 349b4629be, 3d18a04c39, abc283fc64, e287d965f5, a641b4da2c, f4388d5e4a, 8e274a5a84; Advising 5e00f9f7fb; Bagisto 2a5e8b1ddd, 42e3a94b98, 51a7b9ddc2, 6619109cb8; Firefly 8008311d9c, 16a511cf79, 0b33e1ff09 (plus 2 files at parent), cc88d5962e (plus AuthController at parent), f9e2a677d9, 9028ad36ad, c51df8cd83, bee219ebf7, d4e14dd262.
- **Primary category = the weakest requirement on the definition.** If one kind of reference alone lets a realistic rule flag the before-state, the row takes that kind, and the other kinds go in the note. For example, Guzzle `new Client()` plus `use Cache;` is named-type, and `use Cache;` plus `cache()` is root-alias.
- **Placement versus named-type** (both expressible). For C1 placement commits I put named-type when the evidence names the cross-reference that made the location wrong: a cycle, a core-to-module import, or a domain-to-Admin import. I put placement when only the namespace showed it.
- **"Folder X holds only subclasses of Y"** (C2 placement commits and rules) is named-type. It is a placement rule over an ancestor or interface fact, and the spec follows those transitively (ADR 0003).
- **From-side not a type.** Three commits and one rule start in code that is not a named type: a global helper file, a Blade view, or anonymous migration classes. The given categories have no slot for this, so I filed them under the nearest one, marked "(from-side)". None is expressible in any scenario below.
- **Under the candidate definition, unused imports don't count.** This helps in several places. The leftover imports in Coolify ff3d0b29e3, BookStack 546cfb0dcc, and Firefly c51df8cd83 and 0b33e1ff09 no longer flag the after-state.

## Count table

| Category | Commits C1 (59) | Commits C2 (36) | C3 A (5) | C3 B (3) | C3 P (4) | Rules C1 (53) | Rules C2 (38) |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 1 named-type | 32 | 20 | 3 | 2 | 3 | 52 | 28 |
| 2 placement | 21 | 0 | 0 | 0 | 0 | 0 | 0 |
| 3 function-only | 2 (both from-side) | 9 | 1 | 1 | 1 | 0 | 5 |
| 4 function-or-type | 1 | 2 | 1 | 0 | 0 | 0 | 2 |
| 5 root-alias | 2 | 4 | 0 | 0 | 0 | 0 | 0 |
| 6 docblock-only | 0 | 0 | 0 | 0 | 0 | 0 | 0 |
| 7 string-class | 1 (from-side view) | 0 | 0 | 0 | 0 | 0 | 0 |
| 8 inferred-type | 0 | 0 | 0 | 0 | 0 | 0 | 0 |
| 9 via-trait | 0 | 0 | 0 | 0 | 0 | 0 | 0 |
| 10 php-builtin | 0 | 1 | 0 | 0 | 0 | 0 | 3 |
| 11 unclear | 0 | 0 | 0 | 0 | 0 | 1 | 0 |

Rules note: the 53/38 split counts the Pest "mixed" rows the doc's way. The 91 rules are Pinkary 18+17, Vito 35+17 and Advising 0+4.

## Recomputed headline shares

Expressible = categories 1, 2 and 4, plus whichever optional kinds the scenario allows. From-side rows are never expressible.

Commits (out of 153). The doc's figures were 95 (62%) as counted and 107 (70%) with A, B and P.

| Scenario | As counted (C1+C2) | A, B and P count |
| --- | --- | --- |
| Candidate: functions out, aliases resolve, docblock/string/inferred/trait/builtin not counted | 82 (53.6%) | 91 (59.5%) |
| Same, aliases do NOT resolve | 76 (49.7%) | 85 (55.6%) |
| Same as candidate, trait references DO count | 82 (53.6%) | 91 (59.5%) |
| Same as candidate, PHP built-ins DO count | 83 (54.2%) | 92 (60.1%) |

Rules (out of 94). The doc's figure was 91 (97%). Rules have no A/B/P buckets, so only one basis applies.

| Scenario | C1+C2 rules expressible |
| --- | --- |
| Candidate | 82 (87.2%) |
| Aliases do NOT resolve | 82 (87.2%) |
| Trait references DO count | 82 (87.2%) |
| PHP built-ins DO count | 85 (90.4%) |

Sensitivities:
- Koel 20282ed633 (P) is named-type, but its only realistic rule is a static-call-only rule (9/175 controllers at HEAD). A type-to-type edge can't tell a static call from an import (132/175). If it is dropped, the A/B/P basis falls by 1 in every scenario: 90, 84, 90 and 91.
- Coolify d27070b215 is a borderline function-or-type. The only type it removed is a fully qualified `\App\Models\ApplicationDeploymentQueue::where` in one webhook controller. A one-model rule is realistic: 1 controller references the model on today's default branch. If that rule is rejected, the row is function-only and every commit figure drops by 1.
- Vito MigrationsTest:25 (unclear) adds 1 rule, giving 83 (88.3%), if migration files count as application types.

## Rows in categories 3 and 5 to 11

### 3 function-only (commits 11 + A/B/P 3; rules 5)

| App | Commit / rule | Class | Reason |
| --- | --- | --- | --- |
| Koel | 3e747101a9 | C2 | Interaction model's `auth()->user()->id` removed ×4. No type removed; `DB::table` stays |
| Koel | 27933d49cf | C2 | `env()` replaced by `config()` in 9 classes |
| Koel | 47c36f289e | C2 | `auth()->user()` removed from Song::scrobble |
| Coolify | 0ef386b4a8 | C2 | Only global helpers were removed: `instant_remote_process`, `getCurrentApplicationContainerStatus`, `start/stopDatabaseProxy` |
| Coolify | e891318396 | C2 | `config()->set(...)` removed from EmailChannel, Server, Boarding and helpers |
| Coolify | cbee84353e | C2 | `auth()->user()->currentTeam()` removed from 14 event constructors |
| Coolify | 3e0d48faea | C1 | (from-side) The removed `use PrepareCoolifyTask`/`CoolifyTaskArgs` sat in `bootstrap/helpers/remoteProcess.php`, a function file. No type-to-type edge exists |
| BookStack | 5b0cb3dd50 | C1 | (from-side) Before the fix there was no type at all: a global `sortUrl()` in `app/App/helpers.php`, called from a Blade view |
| Advising | 5e00f9f7fb | C2 | `app(AIInterface::class)` replaced by method injection. `AIInterface` is named before and after |
| Bagisto | 2a5e8b1ddd | C2 | Vendor global helpers `country()` and `currency()` from laravel-intl; no vendor class was named |
| Firefly | 16a511cf79 | C2 | `trans()` removed from BillTransformer; Carbon stays |
| Coolify | 469e404725 | C3-P | `instant_remote_process`, `config()` in the Livewire Proxy component |
| Coolify | 8c5c249c6a | C3-B | `instant_remote_process` in the Github webhook controller |
| BookStack | 9d7ce59b18 | C3-A | `error_log()` and `config()` in LoginController |
| Pinkary | GlobalTest.php:5 | C2 | The architectural bans are all functions: `sleep`, `dispatch`, `dispatch_sync`; the rest are debug functions |
| Pinkary | GlobalTest.php:9 | C2 | `session`, `auth`, `request` only in Http/Rules/Livewire |
| Pinkary | LivewireTest.php:19 | C2 | `redirect`, `to_route`, `back` |
| Vito | FoundationTest.php:24 | C2 | `env` |
| Vito | FoundationTest.php:28 | C2 | `sleep`, `usleep` |

### 5 root-alias (commits 6)

| App | Commit | Class | Reason |
| --- | --- | --- | --- |
| Koel | 8f7654a220 | C1 | Song model: `use Lastfm;`, resolved through the config/app.php alias to App\Facades\Lastfm; `Lastfm::scrobble` |
| Koel | a837432a4e | C1 | `use MediaCache;` in Init command, DataController and ClearMediaCache listener; the service also has `use Cache;` |
| Koel | b227ece517 | C2 | iTunesService: `use Cache;` + `Cache::remember`. LastfmService has only `cache()`, which is function-only |
| Bagisto | 42e3a94b98 | C2 | `\DB::update(...)` in the ProductFlat listener, with no import |
| Bagisto | 51a7b9ddc2 | C2 | `\DB::table(...)` in the Velocity ConfigurationController, with no import |
| Firefly | 8008311d9c | C2 | `use Auth;` + 12× `Auth::user()` in AccountRepository |

These rows are named-type and do not depend on aliases, although an alias also appears in them: Koel 4fc5117509 (Album imports App\Facades\Lastfm), bf1d158c90 (`AwsClient` hint; `AWS`/`Cache` via alias), 351efe4092 (Guzzle; Cache/Log via alias); BookStack 349b4629be (ApiToken; `use Hash;`), c157dc3490 (5 controllers import BookStack\Facades\Activity; the web controllers use `use Activity;`); Firefly f9e2a677d9 (HasMany; Auth alias), cc88d5962e (`Illuminate\Mail\Message` hint and `Swift_TransportException` catch; Mail only via `use Mail;`).

### 7 string-class (1)

| App | Commit | Class | Reason |
| --- | --- | --- | --- |
| Bagisto | 6619109cb8 | C1 | (from-side) Blade views did `app('Webkul\Core\Repositories\LocaleRepository')`: a string class name inside a view, which is not a type. Not expressible even if strings counted |

### 10 php-builtin (commits 1; rules 3)

| App | Commit / rule | Class | Reason |
| --- | --- | --- | --- |
| BookStack | e287d965f5 | C2 | `use ZipArchive` + `new ZipArchive()` (ext-zip) in BookExportController |
| Pinkary | ExceptionsTest.php:7 | C2 | `App\Exceptions` `toImplement('Throwable')` |
| Vito | FoundationTest.php:46 | C2 | `App\Exceptions` implement `Throwable` |
| Vito | FoundationTest.php:52 | C2 | Nothing outside `App\Exceptions` implements `Throwable` |

The Throwable rules need a built-in interface as a fact. Built-ins have no file, so ADR 0003's origin fact can't place them.

### 11 unclear (rules 1)

| App | Rule | Class | Reason |
| --- | --- | --- | --- |
| Vito | MigrationsTest.php:25 | C1 | A token search over `database/migrations`. The tokens are mostly named types (`App\Jobs`, `App\Facades\SSH`, `SocketEvent::`, facades) plus `dispatch(` and `broadcast(`. The from-side is migration files: 132 at the pin, the newest are `return new class extends Migration` (anonymous) and the oldest are named global-namespace classes. Migrations are also the typical excluded code. The rule is expressible only if top-level anonymous classes, or migrations, count as application types |

Categories 6, 8 and 9 are empty. No row needs a docblock, an inferred type, or a reference reached only through a used trait. Advising 62e6a4a05a involves traits, but the forbidden references sit in the ai-module traits themselves, which are members, so it is named-type.

## Rows whose original class looks wrong

- **Advising 5e00f9f7fb (C2).** `app(AIInterface::class)` became an `AIInterface $ai` parameter, so the same type is named before and after. By the lead audit's own rule (Koel 65765e8357, Firefly 835a421909 and 1284f9cecc) it is C3 (form).
- **All 5 bucket-A commits (C3)**: Koel 1d5c8e84b6 and 9d9dc0b397, BookStack 9d7ce59b18 and abc283fc64, Bagisto 2ca4f7087e. Each fix removed a reference that 0 or 1 peers hold at HEAD. The doc's C3 test says a commit is C1 or C2 when "the fix removed a reference the role does not normally hold". By the doc's own rubric they are C1 or C2.
- **Coolify 3e0d48faea (C1) and BookStack 5b0cb3dd50 (C1).** Neither from-side is a class: one is a helper file, the other a global function plus a Blade view. The doc's "Blade views and global helper files" caveat lists neither. 3e0d48faea is also a reverse-direction collapse.
- **Coolify 83a7ef21f3 (C1 placement).** Half of it is an enum moved to App\Enums, which the doc classes C4 ("placement by language kind") for Vito's rule. The other half, an invokable moved to Actions, keeps it C1.
- **Firefly bee219ebf7 (C1).** Borderline C3-form. The point is a static-singleton factory becoming a container singleton, though the named type does change because the factory is deleted.
- **Coolify ff3d0b29e3 (C3-B).** Only unused imports survived, so under the candidate definition it behaves as bucket A.

## Per-row table: commits

Legend: N = named-type, PL = placement, FN = function-only, FT = function-or-type, RA = root-alias, SC = string-class, BI = php-builtin.

| App | Commit | Class | Cat | Note |
| --- | --- | --- | --- | --- |
| Koel | 3e747101a9 | C2 | FN | `auth()` in Interaction model |
| Koel | 27933d49cf | C2 | FN | `env()` → `config()` |
| Koel | 3049804811 | C1 | PL | Streamers App\Http → App\Services, namespace-only |
| Koel | 47c36f289e | C2 | FN | `auth()` in Song model |
| Koel | bca8668ace | C2 | N | `Illuminate\Http\Request` hints → FormRequests |
| Koel | 4fc5117509 | C1 | N | Album `use App\Facades\Lastfm;` (explicit); Log via alias, `file_get_contents` incidental |
| Koel | 8f7654a220 | C1 | RA | `use Lastfm;` alias |
| Koel | a837432a4e | C1 | RA | `use MediaCache;` alias ×3; service `use Cache;` |
| Koel | 7c7693179d | C1 | N | `use App\Services\Streamers\*` + `new` |
| Koel | bf1d158c90 | C2 | N | `Aws\AwsClient` parameter hint; `AWS::`/`Cache::` via aliases only |
| Koel | b227ece517 | C2 | RA | `use Cache;` (iTunesService); `cache()` fn (LastfmService) |
| Koel | 351efe4092 | C2 | N | `GuzzleHttp\Client` + `new`; Cache/Log via aliases |
| Koel | 096eb7e610 | C1 | N | `App\Models\User`, UserPreferenceService |
| Koel | 84b05c449f | C1 | PL | Web controllers out of Controllers\API |
| Koel | a4366fe3b1 | C1 | PL | SpotifyClient → Http\Integrations (alternative: named-type on SpotifyWebAPI) |
| Koel | a42eae8b77 | C2 | N | App\Models member not extending Eloquent Model (ancestor fact) |
| Koel | 9ab2bae1da | C1 | PL | Stateless helper Services/Facades → App\Helpers |
| Koel | 1d5c8e84b6 | C3-A | FT | `event(new App\Events\SongLikeToggled)`; import removed |
| Koel | 20282ed633 | C3-P | N | Static model calls; realistic only as a call-form rule (see sensitivity) |
| Koel | 9d9dc0b397 | C3-A | N | `Illuminate\Support\Facades\DB`, `JoinClause` |
| Coolify | e74c464857 | C2 | N | DB facade, Process facade, InvokedProcess imports |
| Coolify | 83a7ef21f3 | C1 | PL | Enum → Enums; invokable → Actions |
| Coolify | cf121062a1 | C1 | PL | CoolifyProcess Services → Actions |
| Coolify | 0ef386b4a8 | C2 | FN | SSH global helpers only |
| Coolify | ba636a95dc | C2 | N | `phpseclib3\Crypt\PublicKeyLoader` |
| Coolify | e891318396 | C2 | FN | `config()->set` |
| Coolify | d27070b215 | C2 | FT | `instant_remote_process` + FQN `\App\Models\ApplicationDeploymentQueue::where` (Github webhook only); borderline |
| Coolify | 3e0d48faea | C1 | FN (from-side) | Helper file referenced the deleted action and DTO |
| Coolify | cbee84353e | C2 | FN | `auth()` in event constructors |
| Coolify | ff3d0b29e3 | C3-B | N | Lcobucci `new Builder`, `Http::`; leftovers are unused imports |
| Coolify | 469e404725 | C3-P | FN | `instant_remote_process`, `config()` |
| Coolify | 8c5c249c6a | C3-B | FN | `instant_remote_process` |
| BookStack | 3d18a04c39 | C2 | FT | `Laravel\Socialite\Contracts\Factory` + `env()` |
| BookStack | 8e274a5a84 | C1 | N | Injected Role/Permission models |
| BookStack | be2ca9d4bb | C1 | N | `use LdapRepo` + `new` |
| BookStack | 919660678b | C1 | PL | Feature-folder reorganisation |
| BookStack | 349b4629be | C1 | N | `BookStack\Api\ApiToken`; Hash via alias |
| BookStack | e287d965f5 | C2 | BI | `ZipArchive` |
| BookStack | c7a2d568bf | C2 | N | Model subclasses gathered into Entities\Models (ancestor fact) |
| BookStack | a641b4da2c | C2 | N | `Illuminate\Database\Connection` → explicit DB facade (reverse direction) |
| BookStack | 837fd74bf6 | C1 | PL | Search domain |
| BookStack | f4388d5e4a | C2 | N | Direct laravel/ui trait use |
| BookStack | 295cd01605 | C1 | PL | Role → domain folders |
| BookStack | 8e3f8de627 | C1 | PL | Notifications into domains |
| BookStack | 42b9700673 | C1 | PL | Exports domain |
| BookStack | 5b0cb3dd50 | C1 | FN (from-side) | Global `sortUrl()` → class |
| BookStack | ac0cd9995d | C1 | PL | Sorting domain |
| BookStack | 9d7ce59b18 | C3-A | FN | `error_log()` |
| BookStack | c157dc3490 | C3-P | N | 5 controllers explicit `BookStack\Facades\Activity`; web controllers `use Activity;` |
| BookStack | abc283fc64 | C3-A | N | WebSafeMimeSniffer, StreamedResponse |
| BookStack | 546cfb0dcc | C3-P | N | Static `Page::visible()`; leftover imports unused |
| Advising | 8a861c5ad4 | C1 | PL | Case models App\Models → module |
| Advising | 0b697a5ebe | C1 | N | Core page imported Audit module settings |
| Advising | 5e00f9f7fb | C2 | FN | `app()` → injection (should be C3 form) |
| Advising | b409c0f602 | C1 | N | Imported Engagement Timeline page |
| Advising | 21850503fc | C1 | PL | Email template resource core → engagement |
| Advising | 725a52feca | C2 | N | Extends Filament Cluster (ancestor fact) |
| Advising | 5a664e6d3e | C1 | PL | Tenancy classes → Multitenancy |
| Advising | 26795aa641 | C2 | N | Extends Filament ImportCsv (ancestor fact) |
| Advising | 62e6a4a05a | C1 | N | ai traits imported Assistant classes (cycle) |
| Advising | 5f5ff17e70 | C1 | N | AiService contract imported OpenAI DTO |
| Advising | 835b8b7b61 | C1 | N | Report page imported Prospect widgets |
| Advising | 8e88bcaf0f | C1 | N | BaseNotification, DatabaseChannelTrait, DatabaseNotification |
| Advising | f2a011da2b | C2 | N | ShouldQueue in Actions (interface fact) |
| Advising | a633b045ff | C1 | PL | MFA → authorization |
| Advising | 57d8f299cc | C1 | N | Notification enum imported Engagement model |
| Advising | 71d3612f00 | C1 | N | Core scope imported CampaignAction |
| Advising | d0ef09d1d1 | C2 | N | ShouldQueue in Actions |
| Advising | d9699f8975 | C1 | N | CampaignAction import + implements contract |
| Advising | 86eddf147e | C2 | N | ShouldQueue in Actions |
| Advising | 61fedc19dc | C1 | N | Kernel `App\Models\Tenant` |
| Bagisto | f236da61ab | C1 | N | Admin/Role models in controller |
| Bagisto | 21a16b8cf0 | C1 | PL | View composers → DataGrids (wired by string class name) |
| Bagisto | 2a5e8b1ddd | C2 | FN | `country()`, `currency()` |
| Bagisto | 8030b097e7 | C1 | PL | Commands → packages |
| Bagisto | 42e3a94b98 | C2 | RA | `\DB::update` |
| Bagisto | 5db4797158 | C1 | N | Attribute, ProductFlat models |
| Bagisto | 51a7b9ddc2 | C2 | RA | `\DB::table` |
| Bagisto | f5d9365dc6 | C1 | N | Customer → Webkul\Shop\Mail |
| Bagisto | 7206988b0e | C1 | N | 15/29 referenced Webkul\Admin (cycle); the rest are placement |
| Bagisto | ea67b95f6d | C2 | N | Explicit DB facade, Carbon |
| Bagisto | af7ac1a1cf | C1 | N | Admin controllers imported domain FormRequests |
| Bagisto | 0a99fe4311 | C1 | N | Shop imported Checkout request |
| Bagisto | 98d5b761c5 | C1 | N | Injected Inventory indexer |
| Bagisto | 32986ba8b0 | C1 | PL | Namespace-only move |
| Bagisto | 62b8d7267f | C1 | N | Admin/FPC → Shop repository |
| Bagisto | 04fe8a4cae | C1 | N | Admin → Shop listener |
| Bagisto | 789da165e2 | C1 | N | DataGrid controller extends Admin Controller |
| Bagisto | 6619109cb8 | C1 | SC (from-side) | Blade `app('Webkul\...\LocaleRepository')` |
| Bagisto | 2316f8d9ff | C1 | N | DataGrid → Admin export |
| Bagisto | 8b6b036d1b | C2 | N | Explicit DB facade |
| Bagisto | 81d1b5a58c | C1 | N | Injected ElasticSearchRepository |
| Bagisto | cce091fc93 | C1 | N | `Webkul\Sales\Models\Order` |
| Bagisto | 54dbbd514b | C1 | N | Core → other packages' concrete models |
| Bagisto | 2ca4f7087e | C3-A | N | `Webkul\Attribute\Models\Attribute` in repository |
| Firefly | 9028ad36ad | C2 | N | `Grumpydictator\Gchart\GChart` |
| Firefly | f9e2a677d9 | C2 | N | `HasMany` closure hints; Auth alias incidental |
| Firefly | 50e5c21735 | C1 | PL | Export → Repositories\Journal (caller used a container string id) |
| Firefly | 8008311d9c | C2 | RA | `use Auth;` |
| Firefly | 194fe178c0 | C1 | PL | Support → Services |
| Firefly | 16a511cf79 | C2 | FN | `trans()` |
| Firefly | bee219ebf7 | C1 | N | Static `ExpressionLanguageFactory::get()` |
| Firefly | d4e14dd262 | C1 | N | ExchangeRateConverter, Support\Facades\Steam |
| Firefly | 0b33e1ff09 | C2 | N | Explicit DB facade; leftover import unused |
| Firefly | c51df8cd83 | C1 | FT | `event(new CreatedSingleTransactionGroup)`, `new TransactionGroupEventFlags` |
| Firefly | 1893a33d84 | C2 | N | DB facade, Query\Builder |
| Firefly | cc88d5962e | C3-B | N | `Illuminate\Mail\Message` hint + `Swift_TransportException` catch; Mail via `use Mail;` |

## Per-row table: rules (91 C1+C2)

Pinkary (tests/ArchTest.php, tests/Arch/*):

| # | Where | Class | Cat | Note |
| --- | --- | --- | --- | --- |
| 8 | ConsoleTest:8 | C2 | N | Extends Illuminate Command (ancestor fact) |
| 11 | ConsoleTest:11 | C2 | N | Only SignalableCommandInterface |
| 12 | ConsoleTest:12 | C1 | N | Commands not used |
| 16 | EnumsTest:9 | C1 | N | `toUseNothing` |
| 18 | EnumsTest:11 | C1 | N | Only-used-in |
| 19 | ExceptionsTest:7 | C2 | BI | Throwable |
| 20 | ExceptionsTest:8 | C1 | N | Only-used-in |
| 21 | FactoriesTest:7 | C2 | N | Extends Eloquent Factory (factories must not be excluded code) |
| 22 | FactoriesTest:9 | C1 | N | Trait RefreshOnCreate (trait fact) |
| 24 | FactoriesTest:12 | C1 | N | Only used in App\Models |
| 25 | GlobalTest:5 | C2 | FN | `sleep`, `dispatch`, `dispatch_sync` (+ debug functions) |
| 26 | GlobalTest:9 | C2 | FN | `session`, `auth`, `request` |
| 32 | HttpTest:8 | C1 | N | Controllers not used |
| 34 | HttpTest:13 | C2 | N | Middleware must use Request (a "must use" dependency rule kind) |
| 35 | HttpTest:14 | C1 | N | Middleware not used |
| 36 | HttpTest:18 | C2 | N | Extends FormRequest |
| 38 | HttpTest:20 | C1 | N | `toBeUsedIn` (must be used by) |
| 41 | HttpTest:26 | C1 | N | `toBeUsedIn` (must be used by) |
| 45 | JobsTest:10 | C2 | N | ShouldQueue |
| 47 | LivewireTest:9 | C2 | N | Extends Livewire\Component |
| 49 | LivewireTest:13 | C1 | N | Only-used-in |
| 50 | LivewireTest:19 | C2 | FN | `redirect`, `to_route`, `back` |
| 53 | MailablesTest:8 | C2 | N | Extends Mailable |
| 55 | ModelsTest:10 | C2 | N | Extends Model |
| 56 | ModelsTest:13 | C1 | N | Only-used-in |
| 60 | NotificationsTest:8 | C2 | N | Extends Notification |
| 61 | NotificationsTest:9 | C1 | N | Only-used-in |
| 62 | ProvidersTest:7 | C2 | N | Extends ServiceProvider |
| 63 | ProvidersTest:8 | C1 | N | Not used |
| 65 | RulesTest:8 | C2 | N | ValidationRule |
| 66 | RulesTest:9 | C1 | N | Only-used-in |
| 67 | ServicesTest:7 | C1 | N | Implements app contract |
| 68 | ServicesTest:8 | C1 | N | Only-used-in |
| 69 | ViewTest:7 | C2 | N | Extends View Component |
| 71 | ViewTest:9 | C1 | N | Not used |

Pinkary totals: C1 18 N; C2 13 N, 3 FN, 1 BI.

Vito (tests/Arch/*):

| # | Where | Class | Cat | Note |
| --- | --- | --- | --- | --- |
| 1 | ActionsTest:5 | C2 | N | Inertia, RedirectResponse, Redirector |
| 2 | ActionsTest:13 | C2 | FT | `Illuminate\Http\Request` + `request` |
| 3 | ActionsTest:18 | C1 | N | |
| 4 | ConventionsTest:18 | C1 | N | SocketEvent |
| 5 | ConventionsTest:28 | C1 | N | Includes explicit DB/Http facade FQCNs |
| 7 | ConventionsTest:43 | C1 | N | |
| 10 | EnumsTest:10 | C1 | N | VitoEnum |
| 11 | EnumsTest:14 | C1 | N | |
| 14–22 | ExtensionPointsTest:23–64 | C1 | N ×9 | Extends app abstract (ancestor fact) |
| 23 | ExtensionPointsTest:70 | C2 | N | |
| 24 | ExtensionPointsTest:75 | C2 | N | |
| 25 | ExtensionPointsTest:80 | C2 | N | |
| 27 | ExtensionPointsTest:90 | C2 | N | |
| 30–34 | ExtensionPointsTest:104–120 | C1 | N ×5 | |
| 36 | FoundationTest:24 | C2 | FN | `env` |
| 37 | FoundationTest:28 | C2 | FN | `sleep`, `usleep` |
| 41 | FoundationTest:46 | C2 | BI | Throwable |
| 42 | FoundationTest:52 | C2 | BI | Throwable |
| 46 | HttpTest:17 | C1 | N | |
| 47 | HttpTest:23 | C2 | N | `Illuminate\Support\Facades\Validator` |
| 48 | HttpTest:28 | C2 | N | `Illuminate\Support\Facades\DB` |
| 49 | HttpTest:33 | C1 | N | |
| 50 | HttpTest:38 | C2 | N | FormRequest |
| 52 | HttpTest:46 | C2 | N | JsonResource |
| 53 | HttpTest:48 | C1 | N | |
| 54 | HttpTest:58 | C1 | N | |
| 56 | JobsTest:8 | C2 | N | |
| 59 | JobsTest:23 | C1 | N | |
| 60 | JobsTest:31 | C1 | N | `class_uses_recursive` = transitive trait fact |
| 63 | MigrationsTest:25 | C1 | unclear | From-side is migration files (anonymous classes) |
| 64 | ModelsTest:9 | C1 | N | |
| 65 | ModelsTest:19 | C2 | N | |
| 67 | ModelsTest:30 | C1 | N | |
| 68 | ModelsTest:39 | C1 | N | |
| 73 | PoliciesTest:17 | C1 | N | |
| 74 | PoliciesTest:27 | C1 | N | |
| 75 | PoliciesTest:31 | C1 | N | `class_uses_recursive` = transitive trait fact |
| 83 | SecurityTest:20 | C2 | FT | Symfony Process + Process facade (+ `proc_open`, `popen`) |
| 84 | SecurityTest:30 | C1 | N | App\Facades\SSH |
| 85 | SecurityTest:34 | C1 | N | App\Helpers\SSH |

Vito totals: C1 34 N + 1 unclear; C2 11 N, 2 FT, 2 FN, 2 BI.

Advising App: ArchTest:49, ArchTest:66 and ArchTest:74 (HasVersion4Uuids/HasUuids, trait facts) and AuditTraitUsageTest:37 (direct `OwenIt\Auditing\Auditable` use) are all C2 N. AuditTraitUsageTest:37 needs a *direct* trait edge: the app's wrapper concern itself uses the vendor trait, so a transitive trait fact, or "via-trait counts", would flag every model that uses the wrapper. Counting trait references for the using class would therefore *break* this rule, not help it.

## Side notes for the decision

- **Trait references help nothing.** Counting what a class's traits reference rescues no commit or rule in this sample, and it would break Advising AuditTraitUsageTest:37 unless direct and transitive use stay distinguishable.
- **Built-in types matter mainly for Throwable.** That is 3 rules plus 1 commit (ZipArchive).
- **Alias resolution matters only for commits.** It decides 6 commits (6 of 95 C1+C2), all from 2016–2021 code: Koel 2018, Bagisto 2020–21, Firefly 2016. No rule depends on it.
- **The function loss is the large one.** It is 11 C1+C2 commits plus 3 A/B/P commits, and 5 rules. Three commits fail because their from-side is not a type (helper file, view), not because of the reference kind.
- **Three Pinkary rules are "must use / must be used by" rules** (HttpTest:13, HttpTest:20, HttpTest:26). The glossary's dependency rules are may/may-not only, so these are named-type but may lack a rule kind. Trait must-use rules (Pinkary FactoriesTest:9, Vito JobsTest:31, Vito PoliciesTest:31) fit as placement rules over trait facts.
