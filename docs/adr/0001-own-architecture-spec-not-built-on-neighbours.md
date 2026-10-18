---
status: accepted
---

# Our own architecture spec, not built on Sloppy or Laravel Architecture Kit

laravel-visual-spec defines its own architecture spec instead of building on Sloppy (`heyosseus/sloppy`) or Laravel Architecture Kit (`gracjankubicki/laravel-architecture-kit`), although both already declare architecture for Laravel apps and serve agents over MCP. The core bet needs things neither can give without upstream rework. Plans are validated before code exists, and the app's specification test enforces the same spec with the same meaning (amended by ADR 0005). Neither declaration can be read by another tool with one meaning. Sloppy's profile is executed PHP that rejects unknown keys, and its allow-list skips unclassified classes, which is exactly what a plan's new classes are. Kit's vocabulary is closed to eight roles, and it must be a runtime dependency. We borrow their ideas freely but take no dependency on either.

## Considered Options

- **Build on Sloppy**: add plan validation and pictures over its profile. Rejected because the profile can't carry plans, reasons or layout without upstream changes from a single maintainer. (Its enforcement being its own analyser rather than Pest no longer counts against it: since ADR 0005, ours isn't Pest's either.)
- **Contribute the plan side upstream** to Sloppy or Kit. Rejected for the same format limits, and because neither is designed to be consumed by a second engine.
- **Interoperate** (import or emit their formats, or `deptrac.yaml`): not rejected. It is left open as a question of spec format.

See the wayfinder ticket "Is a new package warranted, and what is its unique value?" (kfriars/laravel-visual-spec#7) and its research inputs (#2, #5, #15).
