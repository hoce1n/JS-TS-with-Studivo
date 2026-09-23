# Hoisting

Status: documented

JavaScript records some declarations before a scope runs. Studivo makes the modern rules visible: no
`var`, function declarations vs `const` arrows, imports, classes, and default-parameter TDZ.

## Files

| File                        | Role                                    |
| --------------------------- | --------------------------------------- |
| `01-javascript-hoisting.md` | Full walkthrough with Studivo citations |

## Read this when

You need to predict whether a name is usable above its source line, or why a `ReferenceError`
appears during module evaluation.

Related foundation:

- `notes/01-state-and-memory/06-bindings-scope-var-let-const.md`
