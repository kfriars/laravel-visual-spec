# laravel-visual-spec

A Laravel package for writing down an app's intended architecture as a spec that humans review visually and agents plan against. Enforcement lives in a separate Pest plugin that consumes the spec.

## Language

**Architecture spec**:
The single declared description of how an app is meant to be structured, which plans are checked against and the Pest plugin enforces. Provisional term; its final name is still open.
_Avoid_: map, rule map, rules file

**Plan**:
A proposed set of new or changed classes and their intended dependencies, written before the code exists so it can be checked against the architecture spec.
_Avoid_: design, blueprint
