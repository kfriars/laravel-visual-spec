---
status: accepted
---

# Membership by namespace segments and type facts, never by path, wiring or pattern language

A type's membership of a group is decided only from **facts** about the type. The facts are:

- its namespace, matched whole segment by whole segment, where `*` stands for exactly one segment;
- its short name, by a glob;
- an explicit list of types;
- its kind;
- its ancestors, interfaces and traits, followed transitively;
- its directly declared attributes;
- its origin (application, framework or vendor), decided by where its file lives;
- its membership of other groups.

Ancestors, interfaces, traits and attributes may each name either one type or a group. File paths, wiring, regular expressions, `**` and partial-segment globs are not facts.

Some faults are **spec errors**, not warnings:

- a description that names a type or a namespace that doesn't exist;
- a group with no members;
- a rule that can never be judged.

We chose fixed-depth segments because any-depth matching put wrong members into groups in three reference apps: 144 non-models in Firefly III's models, 7 export data classes in BookStack's, and 63 Filament UI actions in Advising App's domain actions. The cost is an `or` for each extra depth, and in return every match can be explained in the picture. We chose errors over warnings because gaps are allowed and Pest passes an empty selector silently, so without errors a typo quietly makes a rule vacuous.

## Considered Options

- **Any-depth wildcards (`**`) or regex:** rejected because of the wrong members above, and because a regex can't be explained or edited in the picture.
- **Path facts:** rejected. Under PSR-4, path follows namespace for application code. Paths appear only in the spec's excludes, which decide what counts as application code and never decide membership.
- **Wiring facts** (route targets, container bindings, registries such as Bagisto's concord `$models`): rejected for now. Only two groups across the six reference apps needed them, and an `is` list or a marker attribute covers both. To be revisited if "How does the architecture spec treat Laravel's wiring?" (kfriars/laravel-visual-spec#16) makes wiring a fact.
- **Origin by namespace** (`Illuminate\` means framework): rejected. FreeScout and Akaunting override vendor namespaces in the repo, and those files are the team's to change. Framework means exactly the files of `laravel/framework`.
- **Closed lists of family values:** rejected. Values are the namespace segments that exist, and a family's yielded group that comes out empty is dropped.
- **Warnings for empty groups and unknown names** (Sloppy's choice): rejected, for the reason above.
- **Shape facts** (`abstract`, `final`, `readonly`): rejected. Class shape is out of scope, and `abstract` would have replaced the base-class exclusions in only 3 of the 6 apps.

## Consequences

- **The Pest plugin must not inherit Pest's matching.** Pest compares raw text prefixes, so `App\Features\Billing` also matches `App\Features\BillingReports`. Its negated trait check sees only direct traits, and it passes empty selectors.
- **Vendor ancestry needs an installed `vendor/`.** Following ancestry through vendor classes requires one.
- **Only application code is judged as the user end of an edge.** Framework and vendor members are only ever used.
- **A group can't be added ahead of its first member.** How planned types count towards that is left to "What is a plan?" (kfriars/laravel-visual-spec#12).

See the wayfinder ticket "How does the architecture spec say which classes belong where?" (kfriars/laravel-visual-spec#9). The evidence is in [research/membership-stress-test](https://github.com/kfriars/laravel-visual-spec/blob/research/membership-stress-test/docs/research/membership-stress-test.md).
