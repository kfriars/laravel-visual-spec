---
status: accepted
---

# A dependency is what PHPStan with Larastan infers, in an app booted for testing

A type depends on another when either:

- its own code names that type, in code or PHPDoc; or
- static analysis infers, for some expression in its code, a type in which the other appears, generic arguments and union members included.

The analysis is PHPStan with Larastan. It is one analysis, shared by the picture and the specification test (amended by ADR 0005). It uses the package's own pinned configuration and runs in an app booted in the `testing` environment.

The analyser's answers are taken as they are. The package never overrides its inference or adds inference rules of its own. So an edge is only as precise as the code's types, and where it isn't precise, the fix is in the code.

We chose this for two reasons:

- The picture, plan checks and the specification test must agree exactly.
- Only type inference sees what Laravel code actually touches: `$post->comments`, `Page::visible()` as `Builder<Page>`, and services fetched from the container. Collecting names would make this a text-matching exercise, with Laravel's magic invisible.

## Considered Options

- **Syntactic names only** (Pest arch, Deptrac, a PHP-Parser visitor): rejected. It is cheap, works file by file and needs no boot. But it can't see inferred receivers, relations, what facade calls return or what container lookups produce.
- **Pest's own arch extractor:** rejected. It counts unused imports, silently drops names that don't autoload, ignores docblocks, scans PSR-4 only, and describes only the first type in a file.
- **PHPStan's own dependency data:** rejected. It is internal, per file, and expands every referenced class into its ancestors.
- **PHPStan without Larastan:** rejected. It needs no boot, but it types Eloquent, facades and the container as `mixed`.
- **Psalm or Mago:** rejected. Psalm's Laravel coverage is weaker, and Mago is young.
- **Overriding Larastan where its answers look inconsistent:** rejected. Two such overrides were considered: typing `app(X::class)` as the abstract, and typing untyped relations from their method bodies. PHPStan merges the answers of every extension on a call, and Larastan's container extensions can't be switched off, so either override would mean fighting Larastan's configuration on every release.
- **Counting unused imports:** rejected. To an analyser, an import is only name resolution.

## Consequences

- **Booting is required.**
  - Analysis boots the app from source config in `testing`, bypassing Laravel's config, route and event caches.
  - If the app can't boot, the analysis fails.
  - Determinism means the same code, spec and environment config give the same edges.
  - A warm run must give exactly a cold run's edges. CI runs cold until that is proven.
- **Service location exposes its concrete.** `app(PaymentGateway::class)` depends on `PaymentGateway` and on the concrete bound in `testing`. Constructor injection depends only on `PaymentGateway`.
- **Unresolved references fail; unknown references are drawn.**
  - A name that resolves to no type the analysis can find fails, whether it appears in code or in PHPDoc.
  - Using a value the analyser types as `mixed` is shown, but never judged.
- **PHP's own types are a fourth origin, PHP.** They come from PHPStan's stubs, so they are the same on every machine. This amends ADR 0003.
- **Traits own their edges.**
  - The edges found in a trait's code belong to the trait, taken as the union across every class that uses it.
  - A trait nothing uses has no edges.
  - A parent class's edges are not inherited by its subclasses.
- **Enforcement can't build on Pest's arch extractor.** Since ADR 0005, enforcement is this package's specification test, which runs this analysis.
- **Functions stay outside the spec.** About 9% of real fixes forbid only a global function. Those rules stay ordinary Pest arch tests.

See the wayfinder ticket "What counts as a dependency?" (kfriars/laravel-visual-spec#10). The evidence is in [research/larastan-inference](https://github.com/kfriars/laravel-visual-spec/blob/research/larastan-inference/docs/research/larastan-inference.md) and [research/kill-condition-recount](https://github.com/kfriars/laravel-visual-spec/blob/research/kill-condition-recount/docs/research/kill-condition-recount.md).
