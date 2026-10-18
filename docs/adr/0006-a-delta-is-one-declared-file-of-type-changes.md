---
status: accepted
---

# A delta is one declared file of type changes, apart from the spec, which the specification test never reads

Planning new work edits two files. Groups and rules are edited in the architecture spec itself. The types to add, change or remove are written in a **delta**: one declared file next to the spec, standing in for code changes not yet made. A delta states what each type's declaration would (full name, kind, direct parent, interfaces, traits and attributes) and the dependencies it is meant to have. It never states which groups a type is in; those are worked out from its facts. A delta is checked by judging the spec in the working tree against the built type graph with the delta applied, so its types count as group members and towards spec errors. The specification test never reads it: code must pass the spec on its own, and no line in a delta can make a test pass. A delta has no author, approval, status or spec version. There is one at a time, and the user or the team's workflow clears it before the next piece of work is planned.

We chose this for four reasons:

- The code is the truth for types, as the spec is for groups and rules. A delta exists only because the code doesn't yet, so it holds changes and nothing else.
- Editing the spec in place keeps one spec file with one reader and one writer (ADR 0005). A delta that carried spec changes would need a second syntax for them, a step that applies them, and an "applied or not" state.
- Stored states such as draft, approved and implemented go stale as soon as the code or the spec moves. Whether a delta passes is worked out whenever it is asked.
- Agents already write their own "plans": prose documents of steps. Calling this file a plan would get an agent to write the wrong thing.

## Considered Options

- **A plan that carries its own spec changes**, applied to the spec later: rejected for the second reason above. Two plans touching one group would also conflict as deltas instead of as a normal merge.
- **A specification test that accounts for the delta**, so that a group whose only members are planned isn't empty: rejected. Tests must never pass because of a line in a delta file.
- **Plans as records**, many at a time, kept with lifecycle states so that impact preview can list the in-flight plans a spec change breaks: rejected for the third reason above.
- **Plans as inputs handed to each command, with no fixed place:** rejected. A delta is a declared file, like the spec.
- **Every dependency the built code will have**, or every type its signatures will name: rejected. Inferred dependencies can't be known before the code exists, so a delta lists the dependencies that are intended.
- **Expected group memberships stated on each type**, as the WIP's `layer:` did: rejected, because they would be a second source of truth beside the facts.
- **A delta embedded in a markdown plan or an issue:** rejected. It is a standalone file.
- **Other names:** plan, sketch, changeset, blueprint.

## Consequences

- **Two versioned schemas.** The delta's schema is versioned apart from the spec's. The two share one syntax and the same fact names. This amends ADR 0005's "only the spec format is versioned".
- **A delta can pass while its code fails.** Built code carries inferred dependencies the delta never listed, and only the specification test judges those.
- **Spec edits that run ahead of code fail the specification test** with spec errors, such as a group with no members yet, until that code is built.
- **What fails a delta:**
  - violations it introduces, while violations already in the code are reported and those it removes are reported as fixed;
  - its types that land in no group, unless the delta marks them as meant to be ungrouped;
  - **delta errors**, which are faults in the delta itself: a name that resolves to no existing or added type, adding a type that exists, changing or removing one that doesn't, moving onto a taken name, dropping a dependency the code doesn't have, and removing a type that built code still depends on.
- **Delta types have no file,** so they are application code by definition, and settings that work by folder, such as excludes, never apply to them.
- **Earlier ADRs are amended.** Where ADRs 0001 to 0005 speak of a plan, they now mean a delta.

See the wayfinder ticket "What is a plan?" (kfriars/whiteboard#12).
