# JavaScript Object Prototype — Studivo Walkthrough

Objects were bags of own properties. **The prototype is the next bag JavaScript looks in when a key
is missing.**

Every object has an internal link to another object (or `null`). That link is _the prototype_.
Method calls, `instanceof`, and `extends` are all that link in a costume.

Studivo never writes `__proto__`, `Object.create`, or `Foo.prototype.bar =`. A repo-wide search
finds **zero** of those. The prototype chain still runs on every `.trim()`, `.map()`,
`instanceof Error`, and `new UploadValidationError()`.

Read this with the cited files open. Predict: _is this property own, or inherited? What does
`instanceof` walk?_

---

## 1. The lookup rule

When you read `obj.key`:

1. If `obj` has an **own** property `key`, use it.
2. Else follow `obj`’s prototype and try again.
3. Repeat until a hit, or the chain ends at `null` → `undefined`.

```javascript
const payload = { title: "یادآوری تمدید اشتراک", body: "..." };
payload.title; // own
payload.toString; // not own → Object.prototype.toString
payload.notARealKey; // chain ends → undefined
```

Writing `obj.key = x` usually creates or replaces an **own** property. It does not change the
prototype (unless you assign to `obj.__proto__`, which Studivo does not).

---

## 2. Built-in prototypes: why primitives have methods

Strings, numbers, and booleans are primitives (previous note). They still answer `.replace` and
`.trim` because JS **boxes** them for that one call: a temporary `String` / `Number` object whose
prototype is `String.prototype` / `Number.prototype`.

```17:21:lib/marketing-leads/schema.ts
export function normalizeMarketingPhoneNumber(value: string) {
  return value
    .replace(/[۰-۹]/g, (digit) => String(persianDigits.indexOf(digit)))
    .replace(/[٠-٩]/g, (digit) => String(arabicDigits.indexOf(digit)))
    .replace(/[\s-]/g, "");
}
```

`value` is a primitive. `.replace` is **not** stored on the string. Lookup:

`value` → boxed `String` → `String.prototype.replace`.

Each `.replace` returns a **new** primitive string. The prototype is not mutated.
`persianDigits.indexOf` is `String.prototype.indexOf` on another primitive.

Seat parsing is the same chain plus `Array.prototype`:

```11:15:app/onboarding/actions.ts
function normalizeManualSeatNumbers(value: string) {
  return value
    .split(",")
    .map((seat) => seat.trim())
    .filter(Boolean);
}
```

| Call          | Prototype that owns the method |
| ------------- | ------------------------------ |
| `value.split` | `String.prototype`             |
| `.map`        | `Array.prototype`              |
| `seat.trim`   | `String.prototype`             |
| `.filter`     | `Array.prototype`              |

`ALLOWED_TYPES.includes(file.type)` in `lib/upload.ts` is `Array.prototype.includes`.
`error.message.includes("seat_id")` is `String.prototype.includes`. **Same method name, different
prototypes.** The receiver (`this`) decides which chain you walk.

`Number.isFinite` is different: it lives on the **`Number` function object**, not on
`Number.prototype`. `templateId.isFinite` would be `undefined`. That is why `getTemplateId` writes
`Number.isFinite(templateId)`, not a method on the number.

```45:47:lib/sms.ts
  const templateId = Number(templateIdRaw);
  if (!Number.isFinite(templateId) || templateId <= 0) {
    throw new Error(`${environmentVariable} is invalid.`);
```

---

## 3. `new` wires the prototype

`new Date(...)` creates an object whose prototype is `Date.prototype`. Then `.getTime()` is
inherited:

```21:24:lib/date.ts
export function toDate(value: DateInput): Date | null {
  if (value == null) return null;
  const date = value instanceof Date ? value : new Date(value);
  return Number.isNaN(date.getTime()) ? null : date;
}
```

If `value` is already a `Date`, it already has that prototype. `instanceof Date` means:
**`Date.prototype` appears on this object’s chain.**

A plain `{ getTime() { return 0 } }` is **not** a `Date`. It has an own `getTime`, but
`instanceof Date` is false. `toDate` would wrap it in `new Date(value)`, which becomes an Invalid
Date. Prototype identity is not “has a method with this name.”

`new Error("...")` sets the prototype to `Error.prototype`, which provides `.message`, `.name`,
`.stack` (engine-dependent), and `.toString`.

```42:42:lib/sms.ts
    throw new Error(`${environmentVariable} is missing.`);
```

```2:3:lib/action-errors.ts
  if (error instanceof Error && error.message.trim()) {
    return error.message;
```

`.message` is typically an **own** property created by the `Error` constructor. `.trim` is still
`String.prototype`. Two chains in one expression: Error instance → string primitive →
String.prototype.

---

## 4. The only class in the app: `extends` is prototype wiring

```19:19:lib/upload.ts
export class UploadValidationError extends Error {}
```

An empty class body. At runtime this still does real work:

1. `UploadValidationError.prototype` is a new object.
2. That object’s prototype is `Error.prototype`.
3. `UploadValidationError.prototype.constructor` points at the class.
4. `new UploadValidationError("...")` creates an object whose prototype is
   `UploadValidationError.prototype`.

Chain of an instance:

```text
instance
  → UploadValidationError.prototype   (empty)
    → Error.prototype                 (.toString, name default, …)
      → Object.prototype              (.hasOwnProperty, .toString fallback, …)
        → null
```

Throw sites pass a string into `Error`’s constructor (via `extends`):

```35:44:lib/upload.ts
  if (!ALLOWED_TYPES.includes(file.type)) {
    throw new UploadValidationError(
      "فقط فایل‌های JPEG، PNG و WebP مجاز هستند.",
    );
  }

  if (file.size > MAX_BYTES) {
    throw new UploadValidationError(
      "حجم فایل نمی‌تواند بیشتر از ۵ مگابایت باشد.",
    );
  }
```

The instance gets `.message` from `Error`. The subclass exists so **callers can tell validation from
disk failure** without parsing Persian strings.

```35:40:app/api/upload/image/route.ts
  } catch (error) {
    if (error instanceof UploadValidationError) {
      return NextResponse.json({ error: error.message }, { status: 400 });
    }

    throw error;
  }
```

Avatar upload is the same check (`app/api/upload/avatar/route.ts`).
`instanceof UploadValidationError` is true only if `UploadValidationError.prototype` is on the
chain. A generic `new Error("فقط فایل‌های...")` would **not** match. A disk `throw` from `writeFile`
would not match. That is the whole product reason for a subclass.

`instanceof Error` is **also** true for `UploadValidationError`, because `Error.prototype` sits
further up the same chain. Order in a catch block matters: check the **subclass first**, then
`Error`. The upload routes do that. `actionError` only needs `Error` / Prisma, so it has no upload
branch.

---

## 5. `instanceof` walks the chain — Studivo’s real use of prototypes

There is no `Object.getPrototypeOf` in this repo. **`instanceof` is the prototype API they actually
call.**

### 5.1 Subclass vs parent

```javascript
error instanceof UploadValidationError; // own class
error instanceof Error; // parent, still true
error instanceof Object; // usually true too
```

`null` and primitives: `instanceof` is false (or a TypeError if the right-hand side is not a
function). Always useful after `catch (error)` where the binding is `unknown`.

### 5.2 Prisma’s generated error class

```81:85:lib/reminders.ts
function isNotificationClaimConflict(error: unknown) {
  return (
    error instanceof Prisma.PrismaClientKnownRequestError &&
    error.code === "P2002"
  );
}
```

```36:38:features/audit/server/helpers.ts
  if (
    error instanceof Prisma.PrismaClientKnownRequestError &&
    error.code === "P2002"
```

`PrismaClientKnownRequestError` **extends** `Error` (Prisma’s code, not yours). Chain:

```text
instance → PrismaClientKnownRequestError.prototype → Error.prototype → Object.prototype → null
```

So:

- `instanceof PrismaClientKnownRequestError` → this is a known Prisma request failure.
- `error.code` → own (or prototype) string `"P2002"`.
- `error.message.includes("seat_id")` → inherited `Error.message` + `String.prototype.includes`.
- `instanceof Error` → still true.

A throw `new Error("P2002")` would pass `instanceof Error` and fail the Prisma check. Prototype
identity is the discriminator; the string `"P2002"` is extra data on that kind of object.

### 5.3 Host objects: Date and DOM

```23:23:lib/date.ts
  const date = value instanceof Date ? value : new Date(value);
```

```64:66:hooks/use-marketing-lead-capture.ts
    const phoneInput = form.elements.namedItem("phoneNumber");
    if (phoneInput instanceof HTMLInputElement) {
      phoneInput.value = normalizedPhoneNumber;
```

`namedItem` can return a `RadioNodeList`, an `HTMLElement`, or `null`. Only
`HTMLInputElement.prototype` on the chain means `.value` is a text field. That is prototype checking
on **browser-provided** constructors. You did not write `HTMLInputElement.prototype`; the engine
did.

`File` in `saveUploadedFile(file: File)` is the same family: instances of the host `File` class,
methods like `.arrayBuffer()` on `Blob.prototype` / `File.prototype`.

---

## 6. Own properties vs inherited vs `"in"`

`"key" in obj` is true for **own and inherited** keys.

```6:8:lib/action-errors.ts
  if (typeof error === "object" && error !== null && "message" in error) {
    const message = Reflect.get(error, "message");
    if (typeof message === "string" && message.trim()) {
```

```16:19:lib/action-errors.ts
export function isNextNavigationError(error: unknown) {
  if (typeof error !== "object" || error === null || !("digest" in error)) {
    return false;
  }
```

Why `"in"` and not `.message` directly?

- Reading `.message` on `null` throws — they already guarded `typeof` + `!== null`.
- `"toString" in error` is true for almost every object, because `Object.prototype.toString` is
  inherited. That would be a useless check.
- `"message" in error` is true for `Error` instances (own `message`) **and** for a plain
  `{ message: "..." }` (own). It is also true if something put `message` on a prototype.

`Reflect.get(error, "message")` reads the same lookup chain as `error.message`, without assuming a
type.

Second branch of `getActionErrorMessage`: objects that are **not** `instanceof Error` but still have
a string `message`. Duck typing after the prototype check failed. Next.js navigation errors may not
pass `instanceof Error` in every runtime, so `digest` is detected with `"digest" in error` rather
than a class they do not control.

`toAuditMetadata` rejected arrays because `typeof [] === "object"` and arrays inherit
`Array.prototype`. Different prototype, wrong shape. That was the objects note; the reason arrays
look like objects is **this** note: `Array.prototype` → `Object.prototype`.

---

## 7. What Studivo refuses to do (and why tutorials diverge)

Textbook prototype lessons mutate `Foo.prototype` or set `__proto__`. That would be a red flag in
this codebase:

- Shared methods on a prototype are a **mutable global**. One `Array.prototype.sort` monkey-patch
  would affect every seat list.
- Next.js bundles many modules; prototype pollution is a real attack class (`__proto__` in JSON).
- The empty `UploadValidationError` class is the **structured** way to share behavior: inherit
  `Error`, add a unique prototype for `instanceof`.

So when a JS course says “put methods on `Person.prototype`,” map it to what Studivo actually ships:

| Tutorial pattern                          | Studivo equivalent                                   |
| ----------------------------------------- | ---------------------------------------------------- |
| `function Foo() {}` + `Foo.prototype.bar` | `class UploadValidationError extends Error`          |
| `obj.__proto__ === Error.prototype`       | `error instanceof Error`                             |
| `Object.create(proto)`                    | not used; literals `{}` (proto = `Object.prototype`) |
| `hasOwnProperty`                          | not used; `"key" in obj` after a null check          |
| Extending `Array.prototype`               | never; `.map` / `.filter` as-is                      |

Plain literals in `actionError` returns (`{ success: false, error: "..." }`) still have
`Object.prototype`. They inherit `.toString`. They are **not** `instanceof Error`. Callers branch on
`.success`, an **own** boolean.

---

## 8. Methods vs own data on the same object

`file.type` and `file.size` in `saveUploadedFile` are own (or host-defined) data properties on the
`File`. `file.arrayBuffer` is a method inherited from `Blob.prototype`.

```47:50:lib/upload.ts
  const destination = path.join(getUploadDir(), subpath);

  await mkdir(path.dirname(destination), { recursive: true });
  await writeFile(destination, Buffer.from(await file.arrayBuffer()));
```

You do not copy `arrayBuffer` onto every file. One function on the prototype, many instances. That
is the original point of prototypes: **shared behavior, per-object data.**

Prisma rows (`user`, `seat`) are plain objects with **own** enumerable fields (`id`, `name`, …).
They usually do not carry methods. Behavior lives in functions (`toDate`, `actionError`) that
_receive_ those objects. Two styles in one app:

- **Prototype methods** — `Date`, `Error`, `String`, `Array`, `File`.
- **Standing functions** — domain logic on plain objects.

Do not hunt for `membership.renew()` on a Prisma result. There is no such prototype.

---

## 9. `this` is a preview, not the next homework

When `date.getTime()` runs, `this` inside `Date.prototype.getTime` is `date`. When
`value.replace(...)` runs, `this` is the boxed string.

Studivo almost never defines `function` methods that use `this` on your own objects. React
components are functions; server actions are functions. The `this` that matters today is **inside
built-in prototype methods**.

If a borrowed method loses its receiver, it breaks:

```javascript
const getTime = date.getTime;
getTime(); // TypeError: this is not a Date
```

`toDate` never detaches `.getTime`. It always calls `date.getTime()`. Keep receiver and method
together until you study `this` on purpose.

---

## 10. Mental model

For every property read:

1. **Own key?** Data on this object (`error.code`, `payload.title`, `file.size`).
2. **Elsewhere on the chain?** Built-in method (`trim`, `map`, `getTime`, `includes`).
3. **Which constructor’s `.prototype`?** `instanceof C` answers that.
4. **End of chain?** `undefined`, not a throw (unless you used `null.foo`).

For every `throw` / `catch`:

1. `instanceof UploadValidationError` → 400, user message.
2. `instanceof PrismaClientKnownRequestError` + `code` → conflict copy.
3. `instanceof Error` → `.message`.
4. Else duck-type `"message" in error` or a fallback string.

That stack **is** the prototype chain, used as a taxonomy of failures.

---

## 11. Exercises on this repo

1. `new UploadValidationError("x") instanceof Error` — true or false? Why must the upload route test
   the subclass first?
2. Replace `extends Error` with a plain `{ name, message }` object. What happens to
   `instanceof UploadValidationError`?
3. `"trim" in "abc"` vs `"abc".trim()`. One boxes; one may throw. Which, in sloppy vs strict?
4. `[] instanceof Object` and `[] instanceof Array`. Both true. How does `toAuditMetadata` still
   reject arrays?
5. `error.message.includes` vs `ALLOWED_TYPES.includes` — same function? Prove it with prototypes,
   not names.
6. Could `isNotificationClaimConflict` use `"code" in error && error.code === "P2002"` without
   `instanceof`? What extra values would sneak through?
7. `phoneInput instanceof HTMLInputElement` after `namedItem`. What else might that API return, and
   what would `.value =` do on those?
8. There is no `Object.create` in Studivo. What prototype does `{ success: false }` have anyway?

---

## Files cited

| File                                  | Why it is here                                                                |
| ------------------------------------- | ----------------------------------------------------------------------------- |
| `lib/upload.ts`                       | Only app `class`; `extends Error`; `Array.prototype.includes`; `File` methods |
| `app/api/upload/image/route.ts`       | `instanceof UploadValidationError` before rethrow                             |
| `app/api/upload/avatar/route.ts`      | Same prototype check                                                          |
| `lib/date.ts`                         | `instanceof Date`; `Date.prototype.getTime`                                   |
| `lib/marketing-leads/schema.ts`       | `String.prototype.replace` / `indexOf`                                        |
| `app/onboarding/actions.ts`           | String + Array prototype methods chained                                      |
| `lib/sms.ts`                          | `new Error`; `Number.isFinite` not on the prototype                           |
| `lib/action-errors.ts`                | `instanceof Error`; `"in"` vs inherited keys                                  |
| `lib/reminders.ts`                    | Prisma error subclass + `.code`                                               |
| `features/audit/server/helpers.ts`    | Subclass then `Error`; `String.prototype.includes` on `.message`              |
| `hooks/use-marketing-lead-capture.ts` | Host prototype `HTMLInputElement`                                             |

Next in Data Types: **arrays** (objects whose prototype is `Array.prototype`) — or `this` /
constructors if you want to stay on the prototype thread.
