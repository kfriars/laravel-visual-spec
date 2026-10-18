---
status: accepted
---

# Overlapping, fact-described groups, with every rule between them and no tolerated violations

The architecture spec is built from **groups**: named sets of types, app or vendor, each described declaratively by facts a plan can state and the Pest plugin can prove (namespace, path, kind, ancestry, interfaces, traits, attributes). A type may sit in any number of groups and is bound by every rule of every group it is in; groups do not nest. Every rule is between groups, and a type in no group is a **gap** that no rule judges. The spec has no notion of a tolerated violation: a violation is resolved by changing the code or dropping the rule. We chose this because namespace tracks location (layer, domain, module) in every reference app but tracks role (controller, model, action) only in default-skeleton ones, so any single partition fails somewhere; because a plan's classes don't exist yet, so membership can't depend on reflection or file contents; and because the spec states intent, which a list of tolerated violations would blur.

## Considered Options

- **One nested partition** (every type in exactly one innermost group): rejected; it forces one axis to win, and a class with several roles gets one arbitrarily.
- **Two fixed axes** (one location plus any number of roles): subsumed; overlapping groups give any number of axes with one rule semantics.
- **Flat layers, namespace = layer** (the WIP draft, Deptrac-style): rejected; domain-oriented and modular apps would need one layer per module × role.
- **Membership by arbitrary PHP** (closures over reflection, files or the container): rejected; it can't be evaluated for planned classes, explained in the picture, or edited there.
- **Total coverage** (every app type must be in a group): rejected; gaps are allowed and shown, and planned code that lands in one is flagged.
- **Tolerated violations** as a separate baseline file, or as exceptions on a rule: rejected; neither belongs to a statement of intent. Whether the Pest plugin offers its own ignores is the plugin's business.

## Consequences

- The spec states nothing the Pest plugin can't prove, and the plugin must leave gaps out of "may only be used by" checks, which Pest itself applies to every user.
- A plan must state, for each new type, the facts that group descriptions use.
- A type has no single box, so each view chooses which groups it draws.

See the wayfinder ticket "What can the architecture spec express?" (kfriars/laravel-visual-spec#8).
