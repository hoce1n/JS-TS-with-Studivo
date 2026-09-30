# JavaScript Equality Comparisons — Studivo Walkthrough

Casting was **when a type changes**. This note is **when two values count as the same** — operators
you write, and algorithms the engine runs for `===`, `==`, `Object.is`, Map/Set, and `includes`.

MDN splits this into:

```text
Equality Comparisons
├── Value Comparison Operators     ===  !==  ==  !=   (+ relational < > <= >=)
└── Equality Algorithms            IsStrictlyEqual, IsLooselyEqual, SameValue, SameValueZero
```

The operators are the syntax. The algorithms are **which rule** that syntax (or `Map.has`) actually
uses. Mixing them is how `"1" == 1` becomes true, how `NaN === NaN` is false, and how `Set` still
collapses two `NaN`s.

Read this with the cited files open. Predict: _same type? same reference? coerced? Which algorithm?_

Related: `docs/javascript-primitive-types-in-studivo.md`, `docs/javascript-objects-in-studivo.md`,
`docs/javascript-type-casting-in-studivo.md`, `docs/javascript-set-and-map-in-studivo.md`.

---

## 1. The rule that pays rent

**Studivo compares with `===` / `!==` almost everywhere.** Loose `==` appears as `== null` (both
`null` and `undefined`). One UI line uses `== 1` on a length. `Object.is` is unused.
Map/Set/`Array.includes` use **SameValueZero**, which you do not type — the collection does.

| What you write                       | Algorithm       | Coerces? | `NaN` vs `NaN` | `+0` vs `-0`  | In Studivo                              |
| ------------------------------------ | --------------- | -------- | -------------- | ------------- | --------------------------------------- |
| `===` / `!==`                        | IsStrictlyEqual | no       | **not equal**  | equal         | default                                 |
| `==` / `!=`                          | IsLooselyEqual  | **yes**  | not equal      | equal         | `== null` only (plus one `== 1`)        |
| `Object.is(a, b)`                    | SameValue       | no       | **equal**      | **not** equal | unused                                  |
| `Map` / `Set` keys, `Array.includes` | SameValueZero   | no       | **equal**      | equal         | CORS Set, unique seats, MIME `includes` |
| `Array.indexOf`                      | IsStrictlyEqual | no       | not equal      | equal         | string `.indexOf` only, not arrays      |

Objects: `===` is **identity**, not deep equal. Two `{ id: "x" }` literals are never `===`. Seat
occupancy compares **ids and timestamps**, not row objects.

---

## 2. Value comparison operators

### 2.1 `===` / `!==` — same type, same value (strict)

If types differ, result is `false` immediately. No `ToNumber`. No `"5" === 5`.

**String vs string (enums, flags, paths):**

```43:43:app/(dashboard)/_lib/dashboard-utils.ts
  if (assignment.membership.status === "CANCELLED") return false;
```

```64:65:app/(dashboard)/_lib/dashboard-utils.ts
    (assignment.membership.status === "ACTIVE" ||
      assignment.membership.status === "PENDING");
```

Prisma status is a string. `"CANCELLED" === "CANCELLED"`. `"cancelled"` would be false. Literals in
TS (`as const`, types note) keep you from comparing to a typo if the union is named.

```29:29:lib/marketing-leads/public-api.ts
  if (process.env.NODE_ENV === "production") return [];
```

```18:18:lib/db.ts
if (process.env.NODE_ENV !== "production") {
```

Env values are strings. `!== "production"` is true for `"development"` **and** for `undefined`. That
is intended: treat anything but production as “attach Prisma to `global`.”

**Number vs number:**

```54:54:lib/reminders.ts
  if (candidate.daysLeft === 1) return "۱ روز تا انقضا";
```

`daysLeft` is a `number` (calendar math). `=== 1` not `=== "1"`. `=== 1` is false for `1.0`? No —
`1 === 1.0` is true (same IEEE value). False for `"1"`.

```63:63:lib/sms.ts
    record.status === 1 &&
```

JSON number `1` vs literal `1`. If SMS.ir sent `"1"` (string), this is **false**. The predicate then
fails; you do not coerce. That is the point of `unknown` + `=== 1`.

```76:76:lib/push.ts
    if (statusCode === 404 || statusCode === 410) {
```

After `Number(error.statusCode)`. If conversion failed, `statusCode` is `undefined`;
`undefined === 404` is false (no throw).

**`typeof` results are strings — compare them strictly:**

```56:57:lib/sms.ts
  if (typeof payload !== "object" || payload === null) {
    return false;
```

```6:6:lib/action-errors.ts
  if (typeof error === "object" && error !== null && "message" in error) {
```

`typeof null === "object"` (primitives quirk). So you **also** `=== null` / `!== null`. Two strict
checks, two meanings: “is it the object type tag” vs “is it the null primitive.”

**Nullish as its own primitive, not via `==`:**

```65:65:lib/sms.ts
    record.data !== null &&
```

After `typeof record.data === "object"`, still exclude `null`. Strict: `undefined !== null` is
**true**, so a missing `data` (`undefined`) already failed `typeof === "object"`.

### 2.2 `==` / `!=` — loose, coerces (almost banned)

IsLooselyEqual: if types differ, `ToNumber` / `ToPrimitive` until they match. Highlights:

```javascript
null == undefined     // true
"" == 0               // true
"5" == 5              // true
false == 0            // true
[] == false           // true
[] == 0               // true
```

None of those belong next to a seat id or a phone.

**The allowed idiom — `== null`:**

```21:22:lib/date.ts
export function toDate(value: DateInput): Date | null {
  if (value == null) return null;
```

```21:22:app/(dashboard)/_lib/dashboard-utils.ts
function toTime(value: Date | string | null | undefined): number | null {
  if (value == null) return null;
```

```46:46:app/(dashboard)/_lib/dashboard-utils.ts
  if (assignment.endsAt == null) return true;
```

`value == null` is `value === null || value === undefined`. One operator, both absences. `0 == null`
is **false**. `"" == null` is **false**. That is why this is the one loose check that does not eat
empty string or zero.

Open assignment in Schema V2: `endsAt` is SQL `NULL` → JS `null`. Missing argument might be
`undefined`. Occupying if either.

```51:51:app/(dashboard)/_lib/dashboard-utils.ts
  if (assignmentEnd == null || membershipEnd == null) return false;
```

`toTime` returned `null` for invalid dates (`NaN` getTime). Here `== null` is only `null` (the
function never returns `undefined`), but the idiom stays consistent.

**The slip — `== 1` on a length:**

```197:197:components/ui/field.tsx
    if (uniqueErrors?.length == 1) {
```

`length` is a number. `== 1` and `=== 1` are the same **if** `length` is actually a number. Optional
chaining: `undefined == 1` is false. House style would still write `=== 1`. This is shadcn UI, not
hall occupancy. Do not copy `==` into `reserve-seat`.

**Checkbox boolean is `=== "on"`, not `== true`:**

```164:164:app/(dashboard)/settings/actions.ts
    publicPageEnabled: formData.get("publicPageEnabled") === "on",
```

`== true` would coerce `"on"` → NaN → false. Strict string compare is the conversion (casting note).

### 2.3 Relational operators — not equality, but they compare

`<` `>` `<=` `>=` coerce via `ToPrimitive` / `ToNumber` for mixed types. Same-type strings compare
lexicographically. Same-type numbers compare numerically. `Date` compares via `valueOf()` (ms).

```47:47:app/(dashboard)/_lib/actions/reserve-seat.ts
    if (data.startsAt >= data.endsAt) {
```

Zod already `z.coerce.date()` — these are `Date` objects. `>=` uses timestamps. Invalid Date → `NaN`
comparisons are **false** (so `>=` would not catch “both invalid” the way you hope). Coerce+validate
first.

```158:159:lib/reminders.ts
      membership.endsAt < now &&
      endsAtTehranKey === todayTehranKey
```

Two different comparisons: **order** (`<` on Date vs now) and **equality** (`===` on civil
**strings** `"2026-09-30"`). Same calendar day in Tehran can still be `endsAt < now` (this morning
vs this afternoon). The string key is the equality that “today” cares about — not `endsAt === now`
(almost never true to the ms).

```69:69:lib/finance-range.ts
  if (parsed.year !== year || parsed.month !== month || parsed.day !== day) {
```

Round-trip: parse `"1404-01-01"` → Date → parts. If the civil triple does not **`!==` match**, the
date was invalid (e.g. month 13). Strict number equality, no coerce.

`<= 0` on `daysLeft` (reminders) is relational on numbers, not `== 0`. Negative leftover days still
“ends today or earlier” copy.

### 2.4 What is _not_ an equality operator

- `=` assignment
- `=>` arrow
- `instanceof` — prototype chain (inheritance note), not `===`
- `"message" in error` — key existence, not value equality
- `Array.isArray` — brand check

```javascript
assignment instanceof Object; // true for arrays too
assignment === Object; // false
```

---

## 3. Equality algorithms (what the engine actually runs)

### 3.1 IsStrictlyEqual — `===`

Informal:

1. If types differ → `false`.
2. If both `number`: `NaN` is not equal to anything, including itself; `+0 === -0` is `true`.
3. If both `string` / `boolean` / `undefined` / `null` / `bigint` / `symbol`: compare values.
4. If both objects: **same reference**.

```javascript
NaN === NaN; // false
0 === -0; // true
"PASSWORD_RESET" === OTP_PURPOSE.PASSWORD_RESET; // true (same characters)
```

```7:10:lib/otp.ts
export const OTP_PURPOSE = {
  PASSWORD_RESET: "PASSWORD_RESET" as OtpPurpose,
  SIGNUP: "SIGNUP" as OtpPurpose,
} as const;
```

The object `OTP_PURPOSE` is one reference. The **string values** compare with `===` to Prisma enum
strings. You never need `OTP_PURPOSE === something`; you compare `purpose === OTP_PURPOSE.SIGNUP` or
`purpose === "SIGNUP"`.

**Why `Number.isNaN` / `Number.isFinite` exist:** you cannot write `x === NaN`.

```24:24:lib/date.ts
  return Number.isNaN(date.getTime()) ? null : date;
```

```46:46:lib/sms.ts
  if (!Number.isFinite(templateId) || templateId <= 0) {
```

`getTime()` of invalid Date is `NaN`. `NaN === NaN` is false, so the check is a **function**, not
`===`. `Number.isNaN` does not coerce (`Number.isNaN("foo")` is false). Global `isNaN("foo")`
coerces first — Studivo uses the `Number.` form.

### 3.2 IsLooselyEqual — `==`

If types are the same, it **calls IsStrictlyEqual**. If not, it walks a coerce table (`null` ↔
`undefined`, string ↔ number, boolean → number, object → primitive, …).

The only intended use in hall code is the `null` ↔ `undefined` row of that table (`== null`).
Everything else in that table is a bug next to money, seats, or phones.

`uniqueErrors?.length == 1`: types already match (`number` vs `number`) so loose and strict coincide
— still write `===` in new code.

### 3.3 SameValue — `Object.is`

```javascript
Object.is(NaN, NaN); // true
Object.is(+0, -0); // false
Object.is(1, 1); // true
```

Same as `===` except those two number edge cases. **Not used** in Studivo. You do not distinguish
`+0` and `-0` for seat counts. You detect `NaN` with `Number.isNaN` / `isFinite`, not
`Object.is(x, NaN)`.

If you added `Object.is(templateId, templateId)` as a NaN test, that is the same as `Number.isNaN`
only when `templateId` is a number. Prefer `Number.isFinite`.

### 3.4 SameValueZero — Map, Set, `Array.includes`

Same as SameValue **except** `+0` and `-0` are equal (like `===`). `NaN` equals `NaN` (like
`Object.is`).

```javascript
new Set([NaN, NaN]).size[NaN].includes(NaN) // 1 // true
  [NaN].indexOf(NaN) // -1  (Strict Equality!)
  [0].includes(-0); // true
```

**Set unique seats:**

```18:20:app/onboarding/actions.ts
function uniqueSeatNumbers(numbers: string[]) {
  return Array.from(new Set(numbers));
}
```

Keys are strings. SameValueZero ≡ `===` for strings. `"1"` and `"1"` collapse; `"1"` and `1` (if a
number slipped in) would **both** stay — no coerce. Trim first so `"1"` and `"1 "` stay two seats on
purpose or not.

**Set CORS:**

```51:51:lib/marketing-leads/public-api.ts
    PRODUCTION_ALLOWED_ORIGINS.has(origin) ||
```

`.has` is SameValueZero on the origin string. Not `==`. Not substring.

**Array.includes (SameValueZero on elements):**

```35:35:lib/upload.ts
  if (!ALLOWED_TYPES.includes(file.type)) {
```

```63:65:app/(dashboard)/logs/_lib/activity-normalizer.ts
  if (["EXPENSE", "PAYMENT"].includes(log.entityType)) return "finance";
  if (log.entityType === "MEMBERSHIP") return "membership";
  if (["SEAT", "SEAT_ASSIGNMENT"].includes(log.entityType)) return "seats";
```

```40:40:lib/marketing-leads/public-api.ts
        return url.protocol === "http:" && ["localhost", "127.0.0.1"].includes(url.hostname);
```

Elements are strings. SameValueZero ≡ `===`. `includes` will **not** match a File type object to
`"image/png"` unless `.type` is that string.

**`indexOf` on arrays uses Strict Equality** (`NaN` not found). Studivo’s `indexOf` is on the
**string** `persianDigits` (UTF-16 code units), which uses string search, not the array algorithm.

**Map keys** (rate limit IP, membership id, section id, day key, plan name, error message) —
SameValueZero. Two IPs that stringify the same are one key. Two `{ id }` objects would be two keys
(identity). Studivo uses strings.

### 3.5 Identity vs structural equality

```javascript
const a = { ok: true };
const b = { ok: true };
a === b; // false
a.ok === b.ok; // true
```

Prisma rows from two queries: different objects, same `id` string. Occupancy uses `membership.id` in
a Map key, not `assignment === other`.

```62:63:app/(dashboard)/_lib/dashboard-utils.ts
  const isLegacyActive =
    assignmentEnd === membershipEnd &&
```

`toTime` turned both Dates into **numbers** (ms). Then IsStrictlyEqual on numbers. Same instant →
occupying (legacy). A new Date object with the same ms still matches because you compared
primitives, not `assignment.endsAt === membership.endsAt` (two Date **objects** — false unless same
reference).

That is the recipe: **normalize to primitives, then `===`.**

Gallery remove: `images.filter((item) => item !== url)` — string `!==`, SameValueZero-equivalent for
strings.

### 3.6 Relational vs equality on the same data

| Question          | Operator                               | Example                              |
| ----------------- | -------------------------------------- | ------------------------------------ |
| Same status enum? | `===`                                  | `status === "ACTIVE"`                |
| Missing date?     | `== null`                              | `endsAt == null`                     |
| Same millisecond? | `===` on `getTime()`                   | legacy occupancy                     |
| Same Tehran day?  | `===` on date **keys**                 | `endsAtTehranKey === todayTehranKey` |
| Start before end? | `>=` / `<` on Date                     | reserve form refine                  |
| Invalid Date?     | `Number.isNaN`                         | `toDate`                             |
| In the allowlist? | SameValueZero via `.has` / `.includes` | CORS, MIME                           |

Do not use `==` to mean “same day.” Do not use `===` on two Date objects to mean “same instant.”

---

## 4. TypeScript’s extra layer (erased)

`strict` makes `"1" === 1` a **compile error** (types differ). It does **not** change runtime `==`.
It does not make two objects equal because they have the same shape.

Discriminants (`ok: true as const`) exist so `if (result.ok)` is a **narrowing** check that still
compiles to `=== true` in spirit (`if (result.ok)` uses ToBoolean on a boolean — not loose equality
with `1`).

`payload is SmsIrVerifySuccessResponse` after `status === 1` — the `===` is the runtime algorithm;
the predicate is TS.

---

## 5. Mental model

1. **Same type, primitives → `===`.** Status, ids, env, days, HTTP codes.
2. **`null` or `undefined` → `== null`.** Only that pair.
3. **Never `==` for numbers/strings/booleans.** Checkboxes are `=== "on"`.
4. **NaN is not equal to NaN under `===`.** Use `Number.isNaN` / `isFinite`.
5. **Objects: identity.** Compare `id` or `getTime()`, not the row.
6. **Map/Set/`includes`: SameValueZero.** For strings you already use, identical to `===`. No
   coerce.
7. **Order in time: `<` `>` on Date or on numbers; equality of a calendar day is a string key.**

---

## 6. Exercises on this repo

Predict; do not edit production code.

1. `"CANCELLED" == "CANCELLED"`, `"CANCELLED" === "CANCELLED"`, `"CANCELLED" == true`.
2. `endsAt == null` when `endsAt` is `0`, `""`, `null`, `undefined`, `false`.
3. `record.status === 1` when JSON has `"1"`. What does the SMS predicate return?
4. `Number.isNaN(NaN)`, `NaN === NaN`, `[NaN].includes(NaN)`, `new Set([NaN, NaN]).size`.
5. Two `new Date("2026-09-30T00:00:00.000Z")` — `===`? `getTime() ===`?
6. `assignment.endsAt === membership.endsAt` if both are Date instances with the same ms but
   different objects. What did `toTime` change?
7. `uniqueErrors?.length == 1` vs `=== 1` when `uniqueErrors` is `undefined`.
8. `ALLOWED_TYPES.includes("image/png")` vs `ALLOWED_TYPES.indexOf("image/png") !== -1` vs a `Set`.
   Algorithms?
9. `PRODUCTION_ALLOWED_ORIGINS.has("https://studivo.ir/")` — SameValueZero result?
10. Why `daysLeft === 1` must not be `daysLeft == true`.

---

## Files cited

| File                                           | Comparison                                         |
| ---------------------------------------------- | -------------------------------------------------- |
| `lib/date.ts`                                  | `== null`; `Number.isNaN`; `instanceof Date`       |
| `app/(dashboard)/_lib/dashboard-utils.ts`      | `== null`; `===` status; `===` on ms               |
| `lib/sms.ts`                                   | `!== null`; `typeof ===`; `status === 1`           |
| `lib/action-errors.ts`                         | `typeof === "object"` + `!== null`                 |
| `lib/reminders.ts`                             | `=== 1`; `=== "expired"`; Date `<`; key `===`      |
| `lib/otp.ts`                                   | string enum identity                               |
| `lib/db.ts`                                    | `!== "production"`                                 |
| `lib/push.ts`                                  | `=== 404` after `Number`                           |
| `lib/finance-range.ts`                         | `!==` on civil parts                               |
| `lib/upload.ts`                                | `includes` (SameValueZero)                         |
| `lib/marketing-leads/public-api.ts`            | `Set.has`; `=== "production"`; `includes` hostname |
| `app/onboarding/actions.ts`                    | `Set` unique strings                               |
| `app/(dashboard)/_lib/actions/reserve-seat.ts` | Date `>=`                                          |
| `app/(dashboard)/settings/actions.ts`          | `=== "on"`                                         |
| `components/ui/field.tsx`                      | `== 1` (avoid in app code)                         |

Related: primitives (`NaN`, `null`), casting (`==` as coerce), objects (identity), Set/Map
(SameValueZero).

Next JS: **`this` / constructors**. Next TS: unions and narrowing, generics, or `typeof` / `keyof`.
