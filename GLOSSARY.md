# Whiteboard

A Laravel package for writing down an app's intended architecture as a spec that humans review visually, agents plan against, and the app's own test suite enforces.

## Language

**Architecture spec**:
The single declared description of how an app is meant to be structured, which plans are checked against and the specification test enforces. Provisional term; its final name is still open.
_Avoid_: map, rule map, rules file

**Plan**:
A proposed set of new or changed classes and their intended dependencies, written before the code exists so it can be checked against the architecture spec.
_Avoid_: design, blueprint

**Group**:
A named set of types, from application, framework or vendor code, described by facts its members share, so that a planned type is placed as surely as a written one. A type may belong to any number of groups and is bound by the rules of every group it belongs to.
_Avoid_: layer, module, role, collector (as the general term; a project may name its own groups any of these)

**Fact**:
A statement about a single type that a plan can state and the specification test can prove: its namespace, short name, kind, ancestors, interfaces, traits, attributes, origin, or membership of another group. Group descriptions are built only from facts, combined with and, or and not.
_Avoid_: selector, matcher, criterion

**Group family**:
A template that yields one group, with its rules, for each combination of values of its captured namespace segments, such as one group per module. It means nothing beyond the groups and rules it yields.
_Avoid_: module (as the general term), group pattern

**Dependency**:
A type depends on another when its own code names that type, or when static analysis infers, for some expression in its code, a type in which the other appears. Imports alone, functions and the type itself are not dependencies.
_Avoid_: reference (as the general term), import, coupling

**Dependency rule**:
A rule on a group about which groups its members may depend on (may only use, may not use) or be depended on by (may only be used by, may not be used by). A "may not" rule always wins over a "may only" rule.
_Avoid_: layer rule, allow rule

**Cycle rule**:
A rule that a set of groups must not depend on one another in a cycle, where one group depends on another when any member of the first depends on any member of the second.
_Avoid_: acyclicity rule

**Placement rule**:
A rule relating the memberships of two groups: every member of one must also be a member of the other, or no member of one may be a member of the other.
_Avoid_: convention, shape rule

**Gap**:
A type in application code that belongs to no group. Rules exist only between groups, so a gap is bound by no rule and an edge into it is not judged; gaps are shown so they can be closed.
_Avoid_: unassigned, unclassified

**Unresolved reference**:
A name in application code that resolves to no type that static analysis can find, such as a deleted or misspelled class. It fails, and is fixed in the code, not the spec.
_Avoid_: missing class, dangling edge

**Unknown reference**:
A place where application code uses a value, by calling it, reading it or treating it as a class, whose type static analysis can't infer. It is shown so it can be typed, but no rule judges it.
_Avoid_: unresolved reference, dynamic dependency

**Type graph**:
The types the architecture spec is judged against, with their facts and the dependencies between them, as read from the code or as changed by a plan. A type in it need not exist in code yet.
_Avoid_: model, codebase, snapshot

**Spec error**:
A fault in the architecture spec itself rather than in the code, such as a description naming a type or namespace that doesn't exist, a group with no members, or a rule that can never be judged. It fails as a violation does, but is fixed by editing the spec.
_Avoid_: warning, invalid rule

**Specification test**:
The test in the app's own suite that fails when application code breaks the architecture spec, or when the spec has a spec error. It judges code exactly as the picture and plan checks do.
_Avoid_: architecture test, arch test, enforcement test, Pest plugin

## Origins of code

**Application code**:
The named types in the app's own repository, outside installed packages and excluded code. It is the only code whose dependencies the spec judges.
_Avoid_: app code, user code, first-party code

**Framework code**:
The types shipped by Laravel's framework package itself. Laravel's other packages, such as Sanctum or Horizon, are vendor code.
_Avoid_: core, Illuminate (as the general term)

**Vendor code**:
The types of every installed package other than the framework. They may be group members, but only ever as the used end of a dependency, because the team can't change them.
_Avoid_: third-party code, dependencies

**PHP code**:
The types PHP itself and its extensions provide, such as `PDO`, `Redis` or `DateTime`, which have no file in the app or its packages. Like vendor code, they may be group members, but only ever as the used end of a dependency.
_Avoid_: built-ins, internal classes, core

**Excluded code**:
Code in folders the spec is told to ignore, such as database factories and seeders. It is not application code, so it is neither a group member nor a gap.
_Avoid_: ignored code, baseline
