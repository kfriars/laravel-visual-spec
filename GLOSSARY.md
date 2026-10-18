# laravel-visual-spec

A Laravel package for writing down an app's intended architecture as a spec that humans review visually and agents plan against. Enforcement lives in a separate Pest plugin that consumes the spec.

## Language

**Architecture spec**:
The single declared description of how an app is meant to be structured, which plans are checked against and the Pest plugin enforces. Provisional term; its final name is still open.
_Avoid_: map, rule map, rules file

**Plan**:
A proposed set of new or changed classes and their intended dependencies, written before the code exists so it can be checked against the architecture spec.
_Avoid_: design, blueprint

**Group**:
A named set of types, from the app or from vendor code, described by stated facts its members share, such as namespace or parent class, so that a planned type is placed as surely as a written one. A type may belong to any number of groups and is bound by the rules of every group it belongs to.
_Avoid_: layer, module, role, collector (as the general term; a project may name its own groups any of these)

**Group family**:
A template that yields one group, with its rules, for each value of a captured part of its description, such as one group per module. It means nothing beyond the groups and rules it yields.
_Avoid_: module (as the general term), group pattern

**Dependency rule**:
A rule on a group about which groups its members may reference (may only use, may not use) or be referenced by (may only be used by, may not be used by). A "may not" rule always wins over a "may only" rule.
_Avoid_: layer rule, allow rule

**Cycle rule**:
A rule that a set of groups must not depend on one another in a cycle, where one group depends on another when any member of the first references any member of the second.
_Avoid_: acyclicity rule

**Placement rule**:
A rule relating the memberships of two groups: every member of one must also be a member of the other, or no member of one may be a member of the other.
_Avoid_: convention, shape rule

**Gap**:
A type in the app's own code that belongs to no group. Rules exist only between groups, so a gap is bound by no rule and an edge into it is not judged; gaps are shown so they can be closed.
_Avoid_: unassigned, unclassified
