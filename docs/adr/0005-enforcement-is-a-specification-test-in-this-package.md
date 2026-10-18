---
status: accepted
---

# Enforcement is a specification test in this package, not a Pest plugin

laravel-visual-spec is one package, and it enforces the architecture spec itself, through a **specification test** in the app's own suite. That test runs unchanged under PHPUnit and under Pest. It reads the spec file when it runs, always runs its own analysis, and uses only the runners' stable public API. The runners only run it and report its result.

The single interpretation of the spec is one seam inside the package, made of three operations:

- read the spec, giving a spec or its spec errors;
- analyse the app, giving a type graph;
- judge a spec against a type graph, giving memberships, violations, gaps and spec errors.

The specification test, the picture, plan validation, impact preview, conformance and the agent interface all call this seam. None of them parses or writes the spec file, or works out a verdict, on its own. The judge knows nothing about plans. A plan reaches it only as changes to a type graph, whose types need no file.

We chose this for three reasons:

- After ADRs 0003 and 0004, nothing of Pest's arch semantics is left to build on: membership, dependencies, gaps, cycles and spec errors all have their own meaning.
- A required Pest would lock PHPUnit-only apps out of the whole package. Half the reference set uses PHPUnit only. Koel and BookStack can't install Pest 5 at all, because both pin PHP 8.3, and every Pest release pins one exact PHPUnit patch version. The `laravel/laravel` skeleton ships PHPUnit, and the installer keeps it when an agent runs it.
- Pest runs PHPUnit test classes unchanged, so one test serves both runners.

## Considered Options

- **A separate core package that both this package and a Pest plugin require:** rejected. It gives three packages to version for one interpretation.
- **A Pest plugin that requires this package:** rejected. Installing enforcement would pull in the picture's dependencies.
- **Two implementations checked against a shared conformance suite:** rejected. It means two copies of segment matching, ancestry, the inference setup, gaps, cycles and spec errors.
- **This package requires a separate Pest plugin that holds the semantics:** chosen first, then dropped. It forces Pest, with Pest's PHP floor and its exact PHPUnit pin, on every app that wants the picture.
- **A Pest plugin inside this package, with Pest optional:** superseded. It still leaves PHPUnit-only apps without enforcement.
- **Tests generated from the spec:** rejected. They are a second copy of the spec that goes stale, and since Pest's arch expectations can't carry the semantics, they would only call the judge anyway.
- **A compiled form of the spec:** rejected. It adds a build step and buys nothing.

## Consequences

- **Versioning.** There is nothing to keep compatible between packages. The seam is internal and carries no semver promise; only the spec format is versioned.
- **Enforcement without the picture** means installing the package, writing the spec by hand, and never opening the picture. It is still the whole package.
- **The package is `require-dev` only.** Nothing it does runs in production.
- **Running PHPUnit classes under Pest is behaviour, not a promise.** It is shown by Pest's loader source and by Coolify's suite, but Pest's docs don't promise it. A fixture runs the specification test under both runners on every upgrade.
- **The app must add the specification test to its suite.** How it is structured is a separate question.
- **Ignores are now this package's call.** ADR 0002 left ignores outside the spec to "the plugin", which this package now is.
- **Earlier ADRs are amended.** Where ADRs 0001 to 0004 say "the Pest plugin", they now mean the specification test.

See the wayfinder ticket "Where is the boundary between this package and the Pest plugin?" (kfriars/laravel-visual-spec#11).
