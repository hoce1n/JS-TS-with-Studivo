# JavaScript Prototypal Inheritance — Studivo Walkthrough

The last note was **lookup**: missing keys walk a hidden link. This note is **inheritance**: that
same link used as _is-a_.

- A `UploadValidationError` **is an** `Error`.
- A `PrismaClientKnownRequestError` **is an** `Error`.
- A `File` **is a** `Blob`.
- An `HTMLInputElement` **is an** `HTMLElement`.
- A seat `{ id, number }` is **not** a subclass of anything. It is a bag plus functions.

JavaScript implements “is-a” by putting the parent’s `.prototype` on the child’s prototype chain.
That is prototypal inheritance. There is no separate class system at runtime — `class extends` is
syntax for wiring those links.

Studivo has **one** `class extends` in application code. Everything else is either a host/library
chain you _consume_, or **composition** (plain objects + functions). That choice is the lesson.

Read this with the cited files open. Predict: _is this an is-a relationship, or a has-a bag of
fields?_

The prototype note stays useful for “where did `.trim` come from?” This note is “why does this
object count as that kind?”

---

## 1. Two sentences that must not blur

|                   | Object prototype (previous)             | Prototypal inheritance (this)                 |
| ----------------- | --------------------------------------- | --------------------------------------------- |
| Question          | Where is this property?                 | What _kind_ of thing is this?                 |
| Mechanism         | Follow `[[Prototype]]` until a key hits | The same walk, used as taxonomy               |
| Operator you feel | `.map`, `.getTime`                      | `instanceof`, `extends`                       |
| Failure           | `undefined`                             | wrong branch (`400` vs `500`, Date vs string) |

Lookup is the engine. Inheritance is the **product meaning** of that engine.

```javascript
child instanceof Parent;
```

means: `Parent.prototype` appears on `child`’s chain. That is the entire definition of inheritance
in JS.

---

## 2. Delegation, not copy

Classical textbooks (Java, C++) _copy_ method tables into the subclass. JavaScript **does not
copy**. The child object **delegates**:

```text
instance.ownFields
  → Child.prototype          (maybe empty)
    → Parent.prototype       (shared methods)
      → Object.prototype
        → null
```

One `Error.prototype` serves every `new Error` and every `new UploadValidationError`. Changing a
method on `Error.prototype` would change all of them (Studivo never does that).

Own data stays on the instance (`error.message`, `error.code`, `file.size`). Shared behavior stays
on a prototype (`Error.prototype.toString`, `Blob.prototype.arrayBuffer`). That split _is_
prototypal inheritance.

---

## 3. The only app-level `extends`: an is-a Error

```19:19:lib/upload.ts
export class UploadValidationError extends Error {}
```

Empty body. Inheritance still did four jobs:

1. **Constructor** — `new UploadValidationError("…")` still runs `Error`’s constructor (implicit
   `super` in a derived class), so `.message` and `.name` exist.
2. **Prototype link** — `UploadValidationError.prototype` → `Error.prototype`.
3. **Identity** — `instanceof UploadValidationError` is true only for this kind.
4. **Substitutability** — anything that accepts `Error` also accepts this object (`instanceof Error`
   is true).

Throw:

```35:38:lib/upload.ts
  if (!ALLOWED_TYPES.includes(file.type)) {
    throw new UploadValidationError(
      "فقط فایل‌های JPEG، PNG و WebP مجاز هستند.",
    );
```

Catch, **most specific type first**:

```35:40:app/api/upload/image/route.ts
  } catch (error) {
    if (error instanceof UploadValidationError) {
      return NextResponse.json({ error: error.message }, { status: 400 });
    }

    throw error;
  }
```

That `if` is inheritance as HTTP status:

| Kind                            | Chain                  | Status                         |
| ------------------------------- | ---------------------- | ------------------------------ |
| `UploadValidationError`         | child → Error → Object | **400** (user’s file is wrong) |
| `Error` from `writeFile` / disk | Error → Object         | rethrow → **500**              |
| random `{ message: "…" }`       | Object                 | not `instanceof` either class  |

If `UploadValidationError` did **not** extend `Error`, it would still be an object you could
`throw`. It would **not** be an `Error`. `actionError`’s `error instanceof Error` would skip it. The
upload route’s `instanceof UploadValidationError` would still work if you checked that class — but
you would have invented a parallel universe of failures. Inheritance here means: **validation
failures participate in the Error taxonomy, with one extra tag.**

Substitutability in the other direction is false: `new Error("فقط فایل…")` is **not** a
`UploadValidationError`. Persian text on `.message` is data, not kind. Kind is the prototype.

There is no `constructor()` and no `super()` in this file. For an empty subclass, the default
derived constructor is `constructor(...args) { super(...args); }`. That is why the string argument
reaches `Error`. If you added a constructor and forgot `super()`, the engine would throw
(`Must call super constructor`). Studivo avoided that footgun by leaving the class empty.

---

## 4. Library inheritance you did not write

Prisma ships a real class hierarchy. Studivo only _tests_ it.

```81:85:lib/reminders.ts
function isNotificationClaimConflict(error: unknown) {
  return (
    error instanceof Prisma.PrismaClientKnownRequestError &&
    error.code === "P2002"
  );
}
```

```36:59:features/audit/server/helpers.ts
  if (
    error instanceof Prisma.PrismaClientKnownRequestError &&
    error.code === "P2002"
  ) {
    if (error.message.includes("seat_id")) {
      return {
        success: false,
        error: "این صندلی همین الان توسط یک درخواست دیگر رزرو شد. لطفاً دوباره تلاش کنید.",
      };
    }
    // ...
  }

  if (error instanceof Error && error.message.trim()) {
    return { success: false, error: error.message };
  }
```

Intended chain (Prisma’s, not yours):

```text
PrismaClientKnownRequestError
  is-a Error
    is-a Object
```

`actionError` uses inheritance as a **funnel**:

1. Next.js navigation fake-errors (duck-typed, not this hierarchy) — rethrow.
2. Known Prisma unique-violation **subclass** — Persian conflict copy.
3. Any **Error** — `.message`.
4. Anything else — `fallback` string.

Step 2 must run before step 3. A P2002 **is an** Error. If you tested `instanceof Error` first,
double-booking would leak English Prisma text to the front desk. **Child before parent** is the
inheritance rule in every catch ladder.

`.code === "P2002"` is **not** inheritance. It is a property on that subclass of failures. Two
P2002s (seat vs membership) share a kind (`PrismaClientKnownRequestError`) and split on data
(`message` text). Kind vs data again.

---

## 5. Host inheritance: Date, File/Blob, DOM

You inherit from the platform the same way.

### 5.1 Date

```21:24:lib/date.ts
export function toDate(value: DateInput): Date | null {
  if (value == null) return null;
  const date = value instanceof Date ? value : new Date(value);
  return Number.isNaN(date.getTime()) ? null : date;
}
```

`instanceof Date` asks: is this already a Date instance (chain includes `Date.prototype`)? If you
pass an ISO string, it is a primitive — not a Date — so `new Date(value)` **constructs** a child of
`Date.prototype`.

A duck `{ getTime() { return Date.now() } }` is **not** a Date. It has-a `getTime`. It is-not-a
Date. `toDate` will not treat it as already converted. Prototypal inheritance is not “looks like.”

### 5.2 File is-a Blob

```31:50:lib/upload.ts
export async function saveUploadedFile(
  file: File,
  subpath: string,
): Promise<{ url: string }> {
  if (!ALLOWED_TYPES.includes(file.type)) {
    throw new UploadValidationError(
      "فقط فایل‌های JPEG، PNG و WebP مجاز هستند.",
    );
  }
  // ...
  await writeFile(destination, Buffer.from(await file.arrayBuffer()));
```

In browsers / undici:

```text
File.prototype → Blob.prototype → Object.prototype
```

- Own / File fields: `name`, `lastModified`, `type` (also on Blob).
- Inherited Blob behavior: `arrayBuffer()`, `size`, `slice()`.

`saveUploadedFile` never calls a Studivo method on `file`. It relies on **File inheriting Blob**. If
someone passed a plain `{ type, size, arrayBuffer }`, TypeScript might be silenced with `as File`,
and runtime might even work — until a real `File` check appears. Inheritance is the contract
`FormData.get("file")` is supposed to honor.

### 5.3 HTMLInputElement is-a HTMLElement is-a Node

```64:66:hooks/use-marketing-lead-capture.ts
    const phoneInput = form.elements.namedItem("phoneNumber");
    if (phoneInput instanceof HTMLInputElement) {
      phoneInput.value = normalizedPhoneNumber;
```

`namedItem` returns several host types. Only one is-a `HTMLInputElement`. That class inherits
HTMLElement (style, `id`, `focus`) and Node (`parentNode`, …). Checking the **leaf** type is the
same child-before-parent idea: you need `.value` as a text control, not “any element.”

`event.preventDefault()` works because the event object is-a `Event` (here `React.FormEvent`, a TS
view of that). You did not copy `preventDefault` onto the event literal.

---

## 6. What is _not_ prototypal inheritance in this repo

### 6.1 TypeScript `extends` on generics

```70:72:app/(dashboard)/_lib/dashboard-utils.ts
export function getActiveAssignment<T extends AssignmentLike>(
  assignments: T[] | undefined
): T | undefined {
```

```59:59:app/onboarding/_components/onboarding-wizard.tsx
  function updateField<K extends keyof OnboardingValues>(key: K, value: OnboardingValues[K]) {
```

This `extends` is **constraint**, erased at compile time (TypeScript vs JavaScript note). No
prototype is wired. `getActiveAssignment` receives a plain array of plain objects. Occupancy is a
**function** `isOccupyingAssignment(assignment)`, not `assignment.isOccupying()`.

If you confuse the two `extends`, you will look for a class tree of seats that does not exist.

### 6.2 Spread copies — has-a, not is-a

```javascript
const authUser = { ...user, email: userEmail };
```

New object, `Object.prototype`, extra own fields. It does **not** become a `User` class. Prisma
models are not constructors in app code. Copying fields is composition.

### 6.3 Module “singletons”

`prisma` on `global` is one object shared by reference (objects note). Sharing a reference is not
inheritance. There is no `PrismaClient.prototype` involvement in `globalForPrisma.prisma = prisma`
beyond the client already being a class instance from `new PrismaClient()`.

### 6.4 React components

`function OnboardingWizard()` does not extend `React.Component`. No `Component.prototype.render`.
Function components **compose** hooks. That is the dominant Studivo UI pattern: inheritance not
invited.

---

## 7. Why domain models use composition

A membership in reserve-seat / reminders is:

```javascript
{
  id, status, startsAt, endsAt,
  user: { name, phoneNumber },
  payments: [ ... ],
}
```

No `class Membership extends SeatOccupant`. Operations:

- `isOccupyingAssignment(assignment)` — standing function
- `sendRenewalReminders()` — walks candidates
- `reserveSeat` — transaction

**Has-a** nested objects. **Uses-a** functions. **Is-a** only for Errors, Dates, Files, DOM nodes.

Reasons this matches prototypal reality in a Next.js app:

1. Prisma already returns plain objects. Wrapping them in your class would copy or hide the row.
2. JSON over the wire **drops** prototypes. A `Membership` instance serialized to the client comes
   back as `{ }`. Kind would vanish; data remains.
3. Server Actions should stay serializable. Class instances are a bad boundary.
4. Inheritance trees get stale (`MemberWithFixedSeatWithDebt`). Occupancy rules already have a
   legacy branch in `isOccupyingAssignment` — a function can encode that without a subclass per era.

When _do_ they inherit? When the **platform already has a taxonomy** you must tap: Error status
codes, Date math, Blob bytes, DOM controls. You extend or `instanceof` those. You do not rebuild ORM
rows as classes.

---

## 8. Constructor pattern vs `class` (what the empty class desugars to)

You will see this in older JS books. Same runtime as `class UploadValidationError extends Error {}`:

```javascript
function UploadValidationError(message) {
  Error.call(this, message); // borrow parent constructor (incomplete vs real super)
  this.name = "UploadValidationError";
}
UploadValidationError.prototype = Object.create(Error.prototype);
UploadValidationError.prototype.constructor = UploadValidationError;
```

`class` / `extends` / implicit `super` is the version Studivo uses. `Object.create(Error.prototype)`
is the explicit “child prototype delegates to parent prototype.” The prototype note said Studivo
never calls `Object.create`. **The compiler does**, once, for that class.

`Error.call(this, message)` in old code is a poor man’s `super`: run parent constructor on the child
instance. Native `Error` subclasses really need `new.target` / `super` to set stacks correctly.
Another reason the empty `class extends Error` is the right one-liner.

---

## 9. Multiple “parents”? Mixins vs chain

A prototype chain is **single** inheritance: one `[[Prototype]]` per object.

Studivo needs “validation error **and** HTTP 400 **and** Persian message.” That is not multiple
prototype parents. It is:

- inherit Error (kind),
- catch in the route (policy),
- `.message` string (data).

Need two behaviors on a plain object? **Compose functions**, do not splice two `.prototype`s. There
is no `Membership extends Occupant, Billable` in this repo.

`Object.assign(Child.prototype, mixin)` would be a mixin. Not used. Prefer `isOccupyingAssignment` +
`toDisplayAmount` as functions.

---

## 10. Mental model

When you see an object, ask:

1. **Is-a (prototype)?** Error / Date / File / Element / Prisma error. Test with `instanceof`.
   Branch child → parent.
2. **Has-a (own fields)?** `title`, `studyHallId`, `payments[]`. Read properties. Pass to functions.
3. **Uses-a (module function)?** `toDate`, `actionError`, `cloneSection`. No chain required.
4. **TS `extends` only?** Generic constraint. Delete it mentally; JS still runs.

When you are tempted to write `class Seat extends MapTile`, write `function getSeatStatus(seat)`
instead — unless you are subclassing a platform Error.

---

## 11. Exercises on this repo

1. `new UploadValidationError("x") instanceof Object` — true? Useful? Why does the upload route not
   test that?
2. Swap the two `if`s in `actionError` (Error before Prisma). What user-visible bug appears on a
   seat clash?
3. Implement `UploadValidationError` as a factory `function make(...) { return { message, name } }`
   without `extends`. Which `instanceof` checks die?
4. Is `file.arrayBuffer` own-on-File or inherited-from-Blob? How would you check in DevTools without
   reading spec?
5. `getActiveAssignment<T extends AssignmentLike>` — if you `new` a class
   `class Row implements AssignmentLike`, does occupancy change? Why not?
6. After `JSON.stringify(new UploadValidationError("x"))`, is the parsed value still
   `instanceof UploadValidationError`? What does that say about API boundaries?
7. Could `HTMLInputElement` checks be replaced with `'value' in phoneInput`? Name a host object that
   would then wrongly pass.
8. Draw the chain for a P2002 thrown from `reserveSeat` through `actionError`. Mark which step is
   kind (inheritance) vs data (`.message.includes`).

---

## Files cited

| File                                               | Why it is here                                          |
| -------------------------------------------------- | ------------------------------------------------------- |
| `lib/upload.ts`                                    | Only `class extends`; File is-a Blob; throw child Error |
| `app/api/upload/image/route.ts`                    | Child-before-parent catch → 400                         |
| `app/api/upload/avatar/route.ts`                   | Same ladder                                             |
| `features/audit/server/helpers.ts`                 | Prisma subclass then Error; funnel                      |
| `lib/reminders.ts`                                 | `instanceof` library subclass                           |
| `lib/date.ts`                                      | `instanceof Date` as is-a, not duck                     |
| `hooks/use-marketing-lead-capture.ts`              | DOM inheritance leaf type                               |
| `app/(dashboard)/_lib/dashboard-utils.ts`          | TS `extends` ≠ prototype; composition                   |
| `app/onboarding/_components/onboarding-wizard.tsx` | Generic `extends keyof`; function UI                    |

Related notes: `docs/javascript-object-prototype-in-studivo.md` (lookup),
`docs/javascript-objects-in-studivo.md` (reference bags),
`docs/typescript-vs-javascript-in-studivo.md` (`extends` erasure).

Next: **arrays** (inherit `Array.prototype`, still objects) — or **`this` / constructors** if you
want to stay on the inheritance thread.
