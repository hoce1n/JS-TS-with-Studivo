# Bindings, Scope, `var`, `let`, and `const`

## Problem

After solving the problem of preserving State, a language needs tools to define which name is
connected to which value, where that connection can be accessed, when it is created, and whether it
can be assigned to another value.

## Computational Need

A program needs to manage State bindings with explicit rules for access and mutability. Those rules
should prevent ambiguous access and errors caused by using a declaration too early.

## Concept

A **binding** is the relationship between a name and a value. **Scope** is the region in which a
binding can be accessed. **Mutability** describes how a binding or the data associated with it can
change.

This layer follows the conceptual path:

```text
Language
   ↓
Variable Declaration
   ↓
Binding
   ↓
Scope + Lifetime + Mutability
```

## Language Feature

`var` has function scope; this model does not always fit the block structure of modern code. `let`
and `const`, introduced in ES6, have block scope, and their bindings are in the **Temporal Dead Zone
(TDZ)** before initialization.

Hoisting is not inherently a failure. It describes how declarations are created or registered in
their environment before ordinary code execution. The important difference is that `var` has the
value `undefined` before assignment, while `let` and `const` cannot be accessed before
initialization.

`const` does not make an object immutable; it prevents reassignment of the binding. Internal
mutation of the object is still possible:

```js
const user = { age: 20 };
user.age = 21; // Allowed
// user = { age: 21 }; // Error
```

Restricting property changes is a separate topic, such as `Object.freeze()`, which is shallow by
default.

## Syntax

```js
var functionScoped = 1;

{
  let blockScoped = 2;
  const fixedBinding = { value: 3 };
  fixedBinding.value = 4;
}
```

Comparison of the three declarations:

| Feature              | `var`               | `let`         | `const`       |
| -------------------- | ------------------- | ------------- | ------------- |
| Scope                | Function            | Block         | Block         |
| Reassignment         | Allowed             | Allowed       | Not allowed   |
| TDZ                  | No                  | Yes           | Yes           |
| Hoisted declaration  | Yes                 | Yes, with TDZ | Yes, with TDZ |
| Typical modern usage | Usually discouraged | Appropriate   | Appropriate   |

## Personal Notes & REPL Verification

`var`, `let`, and `const` are language-level tools for managing State bindings, not memory
management itself. The end of a scope also does not necessarily mean immediate memory release; if a
closure or another reference can access the State, it may remain alive.

| Question or hypothesis                                 | REPL experiment                            | Observed result       | Updated understanding                                                         |
| ------------------------------------------------------ | ------------------------------------------ | --------------------- | ----------------------------------------------------------------------------- |
| Does `const` make an object immutable?                 | `const user = { age: 20 }; user.age = 21;` | The property changed. | `const` protects a binding from reassignment, not an object from mutation.    |
| Does `let` contain `undefined` before its declaration? | `console.log(value); let value = 10;`      | A TDZ error occurred. | The binding exists conceptually, but it cannot be used before initialization. |

Hoisting study: `topics/01-hoisting/01-javascript-hoisting.md`.
