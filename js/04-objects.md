# JavaScript Objects — Studivo Walkthrough

Primitives were values you copy. **Objects are bags of properties you share by reference.**

Almost everything that is not one of the seven primitives is an object: literals `{ title, body }`,
arrays, `Date`, `Error`, `Map`, functions, Prisma rows, `globalThis`.

This note is **plain objects**: create, read, copy, mutate, nest, and detect. Arrays, `Date`, `Map`,
and classes can be later Data Types steps. You will still meet them here when Studivo uses them as
objects.

Read this with the cited files open. Predict: _is this a new object or the same one in memory? What
happens if I change a property?_

---

## 1. The rule that pays rent

```javascript
const a = { ok: true };
const b = a;
b.ok = false;
a.ok; // false — same object
```

```javascript
const a = "ok";
let b = a;
b = "fail";
a; // "ok" — primitives copy
```

Studivo lives on that difference. Reminder **counters** are one object mutated in a loop. Onboarding
**state** is copied with `{ ...current }` so React sees a new object. Prisma is one client hung on
`global`.

---

## 2. Object literals — the default bag

A literal creates **one** object in memory. Keys are strings (or symbols). Values can be primitives
or more objects.

```5:16:lib/push.ts
export type PushPayload = {
  title: string;
  body: string;
  url?: string;
};

export type StoredPushSubscription = {
  id: string;
  endpoint: string;
  p256dh: string;
  auth: string;
};
```

Those `export type` lines are TypeScript shapes (erased). The **runtime** object is built later:

```44:50:lib/push.ts
  return {
    endpoint: record.endpoint,
    keys: {
      p256dh: record.p256dh,
      auth: record.auth,
    },
  };
```

Two objects: the outer subscription, and a nested `keys` object. `record.endpoint` is a **string
primitive** copied into the new object. `record` itself is not nested in; only its fields were read.

Shorthand: if the variable name matches the key, you write the name once.

```15:16:lib/expense-categories.ts
  (Object.entries(EXPENSE_CATEGORY_LABELS) as [ExpenseCategory, string][]).map(
    ([value, label]) => ({ value, label }),
```

`{ value, label }` is `{ value: value, label: label }`.

Onboarding defaults are literals sitting in **module scope** (see the scope note). Every wizard that
forgets to clone them shares those objects:

```17:41:app/onboarding/_components/onboarding-wizard.tsx
const DEFAULT_SECTION: OnboardingValues["sections"][number] = {
  name: "سالن اصلی",
  mode: "AUTO",
  seatCount: 30,
  manualSeats: "",
};
// ...
const initialValues: OnboardingValues = {
  name: "",
  gender: "FEMALE",
  // ...
  sections: [DEFAULT_SECTION],
  plans: [DEFAULT_PLAN],
};
```

`sections: [DEFAULT_SECTION]` does **not** copy the section. The array holds a _reference_ to
`DEFAULT_SECTION`. That is why `cloneSection` exists.

---

## 3. Nested objects and property access

Dot access: `payload.title`. Bracket access: `nextErrors[name]` when the key is in a variable.

```62:66:lib/push.ts
      JSON.stringify({
        title: payload.title,
        body: payload.body,
        url: payload.url ?? "/",
      }),
```

`payload.url` may be `undefined` (optional property). `?? "/"` supplies a primitive default. Missing
key and `undefined` value look the same to `??`.

Optional chaining stops if the left side is `null` or `undefined`:

```50:50:app/(dashboard)/logs/_lib/activity-normalizer.ts
  return text(log.actor?.name) ?? text(metadata.operatorName) ?? "سیستم";
```

`log.actor` can be `null` (Prisma: no actor). `log.actor.name` would throw. `log.actor?.name` yields
`undefined`, then `??` walks to the next string.

Same idea on request objects:

```43:43:lib/auth-plugins/phone-otp-login.ts
  const origin = ctx.request?.headers.get("origin");
```

`ctx.request` may be absent. `?.` is still **object** access, not a primitive operator.

Computed keys in a copy:

```41:45:hooks/use-marketing-lead-capture.ts
    setFieldErrors((currentErrors) => {
      const nextErrors = { ...currentErrors };
      if (error) nextErrors[name] = error;
      else delete nextErrors[name];
      return nextErrors;
    });
```

`name` is `"fullName" | "phoneNumber" | …`. `nextErrors[name]` uses that string as a key. `delete`
removes a property from **that** object. They copied first, so React state is not mutated in place.

The `"in"` operator asks if a key exists (own or inherited):

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

Order matters: `"message" in error` throws if `error` is `null`. The primitives note’s
`typeof === "object" && !== null` guard always comes first.

`Reflect.get` reads the property without assuming a type. The value is still unknown until
`typeof message === "string"`.

---

## 4. Detecting an object at runtime

```javascript
typeof null === "object"; // the lie
typeof [] === "object"; // arrays are objects
typeof new Date() === "object";
typeof function () {} === "function";
```

Studivo’s audit metadata helper refuses the impostors:

```34:37:app/(dashboard)/logs/_lib/activity-normalizer.ts
export function toAuditMetadata(value: unknown): AuditMetadata {
  if (!value || typeof value !== "object" || Array.isArray(value)) return {};
  return value as AuditMetadata;
}
```

Walk:

1. `!value` drops `null`, `undefined`, `""`, `0`, `false`.
2. `typeof !== "object"` drops primitives and functions.
3. `Array.isArray` drops arrays (still objects, wrong shape for metadata).
4. Remaining: a plain-ish object. Then property reads (`metadata.memberName`) are `unknown` until
   `text()` / `numberValue()`.

SMS JSON used the same `typeof` + `!== null` pair (primitives note). Objects begin where that guard
**passes**.

`instanceof` tests the prototype chain, not `typeof`:

```58:59:features/audit/server/helpers.ts
  if (error instanceof Error && error.message.trim()) {
    return { success: false, error: error.message };
```

```36:38:features/audit/server/helpers.ts
  if (
    error instanceof Prisma.PrismaClientKnownRequestError &&
    error.code === "P2002"
```

`Error` is an object with a `.message` string. Prisma’s error is a **subclass** — still an object,
extra `.code`. `instanceof` is how JS distinguishes kinds of objects. Types do not run here.

---

## 5. Copy by reference — mutation of a shared object

Reminder sending allocates **one** counters object, then inner loops write properties:

```176:183:lib/reminders.ts
  const counters: ReminderCounters = {
    candidates: candidates.length,
    notificationsCreated: 0,
    duplicatesSkipped: 0,
    pushDeliveries: 0,
    stale: 0,
    failed: 0,
  };
```

```230:232:lib/reminders.ts
        if (isNotificationClaimConflict(error)) {
          counters.duplicatesSkipped += 1;
          continue;
        }
```

`counters.duplicatesSkipped += 1` mutates the object. Every nested `for` sees the same reference.
That is why totals work (scope note, now with the object rule).

Replacing the object (`counters = { ... }`) would be a different binding; `const` forbids it.
Mutating **properties** of a `const` object is legal. `const` protects the binding, not the innards.

Prisma client: one object, reused.

```12:19:lib/db.ts
export const prisma =
  globalForPrisma.prisma ??
  new PrismaClient({
    adapter,
  });

if (process.env.NODE_ENV !== "production") {
  globalForPrisma.prisma = prisma;
}
```

`globalForPrisma` is `global` (an object). Assigning `.prisma` mutates that global object so hot
reload does not open a second connection. Two module evaluations, **one** client object.

Rate limit store:

```22:26:lib/marketing-leads/public-api.ts
const globalForRateLimit = globalThis as GlobalWithMarketingLeadRateLimit;
const rateLimitStore =
  globalForRateLimit.marketingLeadRateLimitStore ?? new Map<string, RateLimitEntry>();

globalForRateLimit.marketingLeadRateLimitStore = rateLimitStore;
```

`globalThis` is an object. `Map` is an object. Entries `{ attempts, resetAt }` are objects stored
**inside** the Map. `existing.attempts += 1` later mutates that inner object in place.

---

## 6. Copy by spread — a new object, same nested refs

Spread (`{ ...obj }`) creates a **shallow** clone: new outer object, copied property values. Nested
objects are still shared.

Onboarding clones a section so editing one row does not mutate `DEFAULT_SECTION`:

```43:48:app/onboarding/_components/onboarding-wizard.tsx
function cloneSection(section: OnboardingValues["sections"][number]) {
  return { ...section };
}

function clonePlan(plan: OnboardingValues["plans"][number]) {
  return { ...plan };
}
```

These section objects are flat (primitives only), so shallow is enough. If a section ever nested
another object, `{ ...section }` would still share that nested one.

State updates always spread the **root**:

```59:61:app/onboarding/_components/onboarding-wizard.tsx
  function updateField<K extends keyof OnboardingValues>(key: K, value: OnboardingValues[K]) {
    setValues((current) => ({ ...current, [key]: value }));
  }
```

`{ ...current, [key]: value }` = new object, all old keys, one key replaced. React compares the root
reference. Mutating `current.name = x` in place would often **not** re-render.

Adding a section copies the default and overrides `name`:

```77:77:app/onboarding/_components/onboarding-wizard.tsx
        { ...DEFAULT_SECTION, name: `بخش ${current.sections.length + 1}` },
```

Right-hand keys win. New object; `DEFAULT_SECTION` stays `{ name: "سالن اصلی", ... }`.

Auth copies a Prisma user then overrides `email` with a narrowed string:

```146:151:lib/auth-plugins/phone-otp-login.ts
          const userEmail = user?.email;
          if (!user || !userEmail) {
            return ctx.json({ ok: false, error: GENERIC_OTP_ERROR });
          }

          const authUser = { ...user, email: userEmail };
```

`user` is an object from the database (or `null`). After the guard, spread copies enumerable own
properties onto a new object. `email: userEmail` replaces a `string | null` field with a `string`.
The original `user` row object is not mutated.

Field errors: copy, then mutate the **copy**:

```42:45:hooks/use-marketing-lead-capture.ts
      const nextErrors = { ...currentErrors };
      if (error) nextErrors[name] = error;
      else delete nextErrors[name];
      return nextErrors;
```

Pattern: clone → change clone → return clone. Never change the object React (or a caller) still
holds.

---

## 7. Objects as JSON, Prisma `data`, and HTTP bodies

`JSON.stringify` walks enumerable own properties and emits a **string** (primitive). Nested objects
become nested JSON.

```62:66:lib/push.ts
      JSON.stringify({
        title: payload.title,
        body: payload.body,
        url: payload.url ?? "/",
      }),
```

`JSON.parse` on the other side (the service worker) produces **new** objects, not the same
references. Serialization is a boundary: identity is lost, shape is kept (plus: `undefined` keys are
dropped, `Date` becomes a string, functions disappear).

Prisma `create({ data: { ... } })` is the same idea: a plain object describing columns.

```219:227:lib/reminders.ts
        await prisma.notification.create({
          data: {
            userId: recipient.userId,
            studyHallId: candidate.studyHallId,
            type: "MEMBERSHIP_EXPIRING",
            title: message.title,
            message: message.body,
            dedupeKey,
          },
        });
```

`message` is `{ title, body }` from `buildReminderMessage`. Two property reads copy **primitives**
into a new `data` object. Nested `message` is not stored as an object in SQL — two strings are.

Action results are objects the UI branches on:

```41:44:features/audit/server/helpers.ts
      return {
        success: false,
        error: "این صندلی همین الان توسط یک درخواست دیگر رزرو شد. لطفاً دوباره تلاش کنید.",
      };
```

```69:69:lib/push.ts
    return { ok: true as const };
```

A new object every return. Callers should not mutate them; they are messages, not stores.

---

## 8. Walking keys: `Object.keys` / `Object.entries`

Objects are not iterable with `for...of` unless they implement an iterator (arrays, Map). For a
plain label map, convert to an array of pairs:

```3:16:lib/expense-categories.ts
export const EXPENSE_CATEGORY_LABELS: Record<ExpenseCategory, string> = {
  RENT: "اجاره",
  ELECTRICITY: "برق",
  WATER: "آب",
  INTERNET: "اینترنت",
  CLEANING: "نظافت",
  EQUIPMENT: "تجهیزات",
  SALARY: "حقوق",
  OTHER: "سایر",
};

export const EXPENSE_CATEGORY_OPTIONS: { value: ExpenseCategory; label: string }[] =
  (Object.entries(EXPENSE_CATEGORY_LABELS) as [ExpenseCategory, string][]).map(
    ([value, label]) => ({ value, label }),
  );
```

`Object.entries` → array of `[key, value]` (that array is an object too). `.map` builds new
`{ value, label }` objects for selects.

`Object.keys(errors)[0]` in the lead hook is “first property name,” used to focus the first invalid
field. Key order is insertion order for string keys. An empty object → `keys[0]` is `undefined`.

---

## 9. Equality: two literals are not one object

```javascript
{ ok: true } === { ok: true }  // false
```

Push returns `{ ok: true as const }` on success. Callers check **`result.ok`**, a boolean primitive,
not `result === { ok: true }`.

Same for action results: `if (!result.success)`. Never compare objects with `===` unless you mean
**the same reference** (the Prisma singleton: `globalForPrisma.prisma ?? new PrismaClient` uses
reference equality on purpose).

---

## 10. Objects hiding in everyday APIs

These are objects, even when they feel like values:

| Value                   | Why it is an object                                            |
| ----------------------- | -------------------------------------------------------------- |
| `new Date()`            | `typeof === "object"`, methods on the prototype                |
| `new Error("...")`      | `.message`, `.digest` on Next navigation errors                |
| `formData`              | `FormData` instance; `.get` / `.set`                           |
| `event.currentTarget`   | DOM element; `.value` is a string primitive **on** that object |
| `process.env`           | object of strings (or `undefined`)                             |
| `globalThis` / `global` | the global object                                              |
| `[DEFAULT_SECTION]`     | array object holding a reference                               |
| `rateLimitStore`        | `Map` object                                                   |
| `generateOtpCode`       | functions are objects; this note does not need that yet        |

Lead blur writes a primitive onto an object:

```49:51:hooks/use-marketing-lead-capture.ts
  function handlePhoneBlur(event: React.FocusEvent<HTMLInputElement>) {
    event.currentTarget.value = normalizeMarketingPhoneNumber(event.currentTarget.value);
```

The phone **string** is copied and replaced. The **input element** is mutated. Two different rules
in one line.

---

## 11. Mental model

For every value, after the primitives checklist, ask:

1. **`typeof === "object"` and not `null`?** If no, it is not this note.
2. **Is it an array?** `Array.isArray` — treat as a list (next topic), still a reference.
3. **Who else holds this reference?** Module singleton, React state, loop accumulator, Prisma row.
4. **Do I need a new object?** Spread / literal. **Or in-place mutation?** `+=`, `delete`,
   `global.prisma =`.
5. **Is the property missing, `null`, or `undefined`?** `?.` and `??` and `"key" in obj` answer
   different questions.

---

## 12. Exercises on this repo

1. Remove `cloneSection` and push `DEFAULT_SECTION` into two halls’ forms. Change one name. What
   does the other show, and why?
2. In `sendRenewalReminders`, replace `counters.duplicatesSkipped += 1` with a new object each time,
   without assigning back. What totals do you return?
3. `{ ...current, [key]: value }` vs `current[key] = value` in `updateField`. Which breaks React,
   and which rule is that?
4. `toAuditMetadata([])` — `typeof []`, `Array.isArray`, return value?
5. `log.actor.name` without `?.` when `actor` is `null`. Which error? Is that TDZ, scope, or
   objects?
6. `JSON.stringify({ url: undefined })` vs `{ url: payload.url ?? "/" }`. What does the worker
   receive?
7. Why is `result.ok` the check, not `result === { ok: true }`?
8. `const authUser = { ...user, email: userEmail }` — does `user.email` on the Prisma object change?

---

## Files cited

| File                                               | Why it is here                                                      |
| -------------------------------------------------- | ------------------------------------------------------------------- |
| `lib/push.ts`                                      | Nested literal; optional property; `JSON.stringify`; result objects |
| `lib/expense-categories.ts`                        | Literal map; shorthand; `Object.entries`                            |
| `app/onboarding/_components/onboarding-wizard.tsx` | Shared default vs `{ ... }` clone; immutable state updates          |
| `hooks/use-marketing-lead-capture.ts`              | Spread + `delete` + computed key; DOM object vs string              |
| `lib/action-errors.ts`                             | `typeof` + `null` + `"in"`; `Reflect.get`                           |
| `app/(dashboard)/logs/_lib/activity-normalizer.ts` | Object vs array vs null; `?.`                                       |
| `lib/reminders.ts`                                 | One mutated counters object; Prisma `data` object                   |
| `lib/db.ts`                                        | Singleton object on `global`                                        |
| `lib/marketing-leads/public-api.ts`                | `globalThis` + Map of entry objects                                 |
| `lib/auth-plugins/phone-otp-login.ts`              | `?.`; shallow copy with override                                    |
| `features/audit/server/helpers.ts`                 | `instanceof`; returned plain objects                                |

Next in Data Types: **arrays** (lists that are still objects) — or `Date` / `Map` if you want
special built-ins first.
