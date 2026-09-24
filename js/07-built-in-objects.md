# JavaScript Built-in Objects — Studivo Walkthrough

You already know **your** objects (`{ title, body }`) and **prototypes** (`Error`, `Date`, `File`).
**Built-in objects** are the constructors and namespaces the language (and host) ship so you do not
reinvent maps, dates, JSON, or math.

They are still objects. `Date` is a function object. `Math` is a namespace object (you never
`new Math()`). `JSON.stringify` is a function on the `JSON` object.

Studivo uses a **working set**, not the whole spec. This note is that set, with a short “not in this
repo” list at the end.

Read this with the cited files open. Predict: _constructor vs static helper? Does this copy, mutate,
or throw?_

---

## 1. Two families

| Family                 | You typically                                  | Studivo examples                                                                                                             |
| ---------------------- | ---------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| **Constructable**      | `new X(...)` → instance, then instance methods | `new Date()`, `new Map()`, `new Set()`, `new URL()`, `new Error()`, `new FormData()`                                         |
| **Namespace / static** | `X.method(...)` never `new`                    | `Math.floor`, `JSON.parse`, `Number.isFinite`, `Object.keys`, `Array.from`, `Date.now`, `Reflect.get`, `Intl.DateTimeFormat` |

`Intl.DateTimeFormat` is both: you `new` a formatter, then `.format(date)`.

`String` / `Number` in this repo are almost always **called as functions** (`String(n)`,
`Number(raw)`), not `new String()` (which would box and make `typeof "object"`).

---

## 2. `Object` — keys of a plain bag

`Object.keys` / `Object.entries` / `Object.hasOwn` are **statics**. They do not live on `{ }.keys`.

```14:16:lib/expense-categories.ts
export const EXPENSE_CATEGORY_OPTIONS: { value: ExpenseCategory; label: string }[] =
  (Object.entries(EXPENSE_CATEGORY_LABELS) as [ExpenseCategory, string][]).map(
    ([value, label]) => ({ value, label }),
```

`Object.entries` → array of `[key, value]` pairs from **own enumerable** string keys.

```81:83:hooks/use-marketing-lead-capture.ts
    const firstInvalidField = Object.keys(errors)[0] as
      | MarketingLeadFieldName
      | undefined;
```

Empty object → `keys[0]` is `undefined`. Insertion order is the order fields were assigned.

```132:133:app/(dashboard)/logs/_lib/activity-normalizer.ts
export function activityGroupForFilter(group: string | undefined): ActivityGroup | undefined {
  return group && Object.hasOwn(activityGroupLabels, group) ? (group as ActivityGroup) : undefined;
```

`Object.hasOwn(obj, key)` is own-property only. `"toString" in activityGroupLabels` would be true
(inherited). `hasOwn` would be false. That is why this filter cannot be spoofed with an inherited
name.

The objects note used `"message" in error` on purpose (own **or** inherited). Built-in choice:
**`in` vs `Object.hasOwn`** is a product decision.

---

## 3. `Array` — lists, still objects

Array literals `[]` are `new Array` with `Array.prototype`. Statics you will actually see:

**Dedup via `Set`, back to array:**

```18:19:app/onboarding/actions.ts
function uniqueSeatNumbers(numbers: string[]) {
  return Array.from(new Set(numbers));
}
```

`Set` stores unique values. `Array.from` copies any iterable into a real array (`createMany` wants
an array).

**Length + index factory (no loop):**

```74:76:app/onboarding/actions.ts
        const seatNumbers =
          sectionInput.mode === "AUTO"
            ? Array.from({ length: sectionInput.seatCount }, (_, index) => String(index + 1))
```

`Array.from({ length: n }, mapFn)` is a **array-like** (has `.length`), not an array, until `from`
allocates. Index `0 … n-1` → seat labels `"1"`, `"2"`, … (strings, primitives note).

**Type guard, not a constructor:**

```35:35:app/(dashboard)/logs/_lib/activity-normalizer.ts
  if (!value || typeof value !== "object" || Array.isArray(value)) return {};
```

`Array.isArray` is the reliable test. `instanceof Array` can fail across frames; Studivo still often
uses `isArray` at boundaries.

Instance methods (`.map`, `.filter`, `.includes`) are prototype inheritance from the previous notes
— not repeated here.

---

## 4. `Date` — instants, not calendar pages

A `Date` instance stores **one millisecond offset from Unix epoch (UTC)**. Display in Tehran is a
_formatter_ problem (`Intl`), not a different kind of Date.

```21:24:lib/date.ts
export function toDate(value: DateInput): Date | null {
  if (value == null) return null;
  const date = value instanceof Date ? value : new Date(value);
  return Number.isNaN(date.getTime()) ? null : date;
}
```

- `new Date(value)` accepts string | number | Date-like.
- Invalid parse → `getTime()` is `NaN`. `Number.isNaN` (static on `Number`).
- `instanceof Date` — prototypal inheritance note.

**Statics:**

```26:26:lib/otp.ts
  const expiresAt = new Date(Date.now() + OTP_EXPIRY_MINUTES * 60 * 1000);
```

`Date.now()` → number (ms). Adding `5 * 60 * 1000` is primitive math. `new Date(number)` wraps that
instant.

```45:46:lib/reminders.ts
    (Date.UTC(toYear, toMonth - 1, toDay) -
      Date.UTC(fromYear, fromMonth - 1, fromDay)) /
```

`Date.UTC(y, m0, d)` → number. Month is **0-based** (`toMonth - 1`). Used as a day-difference in
civil keys, not as a Tehran-local clock.

`new Date()` with no args = now, in `sendRenewalReminders`, `toDate` fallbacks, shift forms.

Host trap: `Date` is mutable (`setHours` in `startOfDay`). `toDate` does not freeze. If you mutate a
Date that Prisma also holds, you share that object (objects note). Prefer `new Date(existing)` when
you will `setHours`.

---

## 5. `Math` — namespace, never constructed

```14:15:lib/otp.ts
export function generateOtpCode(): string {
  return String(Math.floor(100000 + Math.random() * 900000));
}
```

| Call                       | Role                                   |
| -------------------------- | -------------------------------------- |
| `Math.random()`            | `[0, 1)` float — **not** cryptographic |
| `Math.floor`               | toward −∞; here makes `100000…999999`  |
| `Math.round` / `Math.ceil` | duration and day diffs                 |
| `Math.min` / `Math.max`    | clamp reminder days, pagination        |
| `Math.abs`                 | gallery card distance                  |

```33:36:lib/reminders.ts
  return Math.min(
    MAX_REMINDER_DAYS_BEFORE,
    Math.max(MIN_REMINDER_DAYS_BEFORE, value || DEFAULT_RENEWAL_THRESHOLD_DAYS),
  );
```

OTP uses `Math.random`. That is fine for a 6-digit hall login code with server-side expiry and
hashing-by-lookup; it is **not** `crypto.getRandomValues`. Do not copy this into password reset
entropy without a security review.

`Number.POSITIVE_INFINITY` in the gallery is a **Number static**, used as a Math-ish sentinel for
“closest so far.”

---

## 6. `Number` and `String` as functions

```45:47:lib/sms.ts
  const templateId = Number(templateIdRaw);
  if (!Number.isFinite(templateId) || templateId <= 0) {
    throw new Error(`${environmentVariable} is invalid.`);
```

`Number("12")` → `12`. `Number("abc")` → `NaN`. `typeof NaN === "number"`. **`Number.isFinite`**
rejects `NaN` and `Infinity`. `isFinite` (global) coerces first — Studivo uses the `Number.` form.

`Number.isInteger` on Jalali parts in `finance-range.ts`. `Number.isNaN(date.getTime())` in
`toDate`. `Number.parseInt(value, 10)` in `ReserveForm` quantity (always pass radix `10`).

```javascript
String(Math.floor(...))  // otp.ts — primitive string, not new String()
```

`new String("x")` would be an object wrapper. Never in this repo.

---

## 7. `JSON` — wire format, identity dies

```109:109:lib/sms.ts
       body: JSON.stringify({ mobile, templateId, parameters }),
```

```62:66:lib/push.ts
      JSON.stringify({
        title: payload.title,
        body: payload.body,
        url: payload.url ?? "/",
      }),
```

`JSON.stringify` walks enumerable own properties → **string** primitive. `undefined` values are
dropped. `Date` becomes ISO string. Functions vanish. After `parse`, you have **new** plain objects
(`Object.prototype` only).

Settings round-trip gallery URLs through FormData as text:

```151:155:app/(dashboard)/settings/actions.ts
    galleryImages = JSON.parse(
      formData.get("galleryImages")?.toString() || "[]",
    );
  } catch (error) {
    return actionError(error, "تصاویر گالری معتبر نیستند.");
```

Invalid JSON **throws** `SyntaxError` (an `Error` subclass). They catch and return a Persian
`ActionResult`. `JSON.parse("[]")` → a new array object.

Pretty-print audit metadata in the log table: `JSON.stringify(log.metadata, null, 2)` — second arg
replacer unused (`null`), third is indent.

PWA code sometimes `JSON.parse(JSON.stringify(sub))` to clone a `PushSubscription` into a plain
object. That is “structured clone via JSON,” losing methods — same identity rule.

---

## 8. `Map` and `Set` — keyed collections that are not plain objects

Plain objects coerce keys to strings. `Map` keeps key identity. `Set` is unique values.

**CORS allowlist — small frozen set of strings:**

```3:6:lib/marketing-leads/public-api.ts
const PRODUCTION_ALLOWED_ORIGINS = new Set([
  "https://studivo.ir",
  "https://www.studivo.ir",
]);
```

`.has(origin)` is O(1) intent and avoids `array.includes` on a growing list. Values are primitives;
equality is `===`.

**Rate limit store — Map of IP → entry object:**

```23:24:lib/marketing-leads/public-api.ts
const rateLimitStore =
  globalForRateLimit.marketingLeadRateLimitStore ?? new Map<string, RateLimitEntry>();
```

`.get` / `.set` / `.delete` / iteration `for (const [key, entry] of rateLimitStore)`. The **entry**
is a mutable plain object (`attempts += 1`). The Map holds references (objects note).

**Duplicate memberships on the seat map:**

```157:162:app/(dashboard)/_lib/dashboard-utils.ts
  const activeAssignmentsByMembership = new Map<string, string[]>();
  seats.forEach((seat) => {
    const active = getActiveAssignment(seat.assignments);
    if (active?.membership.id) {
      const existing = activeAssignmentsByMembership.get(active.membership.id) || [];
      activeAssignmentsByMembership.set(active.membership.id, [...existing, seat.number]);
```

Key = membership id (string). Value = **new** array each `set` (`[...existing, seat.number]`), not
`existing.push`, so they do not accidentally share one array across keys… they replace the value
each time. Fine for a one-pass build.

`Set` for unique seat numbers: see `Array.from(new Set(numbers))`. Insert order is preserved.

No `WeakMap` / `WeakSet` in app code.

---

## 9. `URL` — parse, do not regex hosts

```8:8:public/sw.js
const isNextAssetRequest = (request) => new URL(request.url).pathname.startsWith("/_next/");
```

`new URL(absolute)` → object with `.pathname`, `.origin`, `.protocol`, `.hostname`. Invalid input
**throws**. Dev origins:

```38:42:lib/marketing-leads/public-api.ts
      try {
        const url = new URL(origin);
        return url.protocol === "http:" && ["localhost", "127.0.0.1"].includes(url.hostname);
      } catch {
        return false;
      }
```

`catch` turns a throw into “not allowed.” `URL` is the built-in parser so
`"https://evil.example/?x=http://localhost"` does not pass a sloppy `includes("localhost")`.

`new URL("https://app.studivo.ir")` in `app/layout.tsx` is metadata, not request parsing.

`URLSearchParams` (sibling built-in) shows up in log filters:
`new URLSearchParams(searchParams.toString())` then `.set` / `.delete`. Query strings are not JSON.

---

## 10. `Intl` — locale as an object, not a string concat

Persian UI amounts:

```149:149:app/(dashboard)/_lib/dashboard-utils.ts
  return new Intl.NumberFormat("fa-IR").format(value);
```

Jalali _display_ of an instant:

```34:37:lib/date.ts
  return new Intl.DateTimeFormat(APP_JALALI_LOCALE, {
    timeZone: APP_TIME_ZONE,
    ...options,
  }).format(date);
```

`APP_JALALI_LOCALE = "fa-IR-u-ca-persian"`. Calendar and time zone are **options on the formatter**,
not mutations of the `Date`.

`.formatToParts(date)` returns an array of `{ type, value }` objects so finance-range can read
`year` / `month` without parsing a localized string.

`Intl` constructors are heavy; creating one per cell in a hot list is a cost. Several dashboard
cards do `new Intl.NumberFormat("fa-IR")` inline anyway — fine at hall scale.

---

## 11. `Error` — the built-in you throw

Covered as inheritance. As a **built-in constructor**:

```javascript
throw new Error("SMS configuration is missing.");
```

Subclasses the engine provides: `TypeError`, `SyntaxError` (`JSON.parse`), `RangeError`, `URIError`.
Studivo mostly uses `Error` and `UploadValidationError extends Error`. Prisma’s class is a library
subclass, not a language built-in.

`error.message` is a string. `instanceof Error` is the kind check.

---

## 12. `Promise` — async as an object

A Promise is a built-in object with `then`. You rarely `new Promise` in Studivo. You **await** and
use statics:

```92:93:lib/push.ts
  const results = await Promise.allSettled(
    records.map((record) => sendPushToSubscription(record, payload)),
```

`allSettled` → array of `{ status: "fulfilled", value }` | `{ status: "rejected", reason }`. One
dead device does not reject the whole batch. `Promise.all` (dashboard
`Promise.all([getStaffList(), getShifts()])`) **fails fast**.

That is a built-in API choice: occupancy map vs notifications. Same `Promise` constructor, different
static.

`async function` always returns a Promise. That is syntax on top of this object.

---

## 13. `FormData` — host built-in for multipart / actions

```javascript
const formData = new FormData(form);
formData.get("phoneNumber");
formData.set("galleryImages", JSON.stringify(galleryImages));
```

`.get` returns `string | File | null`. Not JSON. Settings put a JSON **string** in one field, then
`JSON.parse` on the server. Two built-ins stacked.

---

## 14. `Reflect` — property access as a function

```7:7:lib/action-errors.ts
    const message = Reflect.get(error, "message");
```

`Reflect.get(obj, key)` is the same lookup chain as `obj[key]`, callable without assuming a type.
Used after `"message" in error`. Rare in this repo; not a second object model.

No `Proxy` in application code.

---

## 15. `RegExp` — patterns as objects

```17:17:lib/auth-plugins/phone-otp-login.ts
  .refine((value) => /^09\d{9}$/.test(value), {
```

`/^09\d{9}$/` is a `RegExp` instance. `.test` returns boolean. `.replace(/[۰-۹]/g, ...)` on strings
uses `RegExp.prototype` plus `String.prototype.replace`. Flags `g` mean global. Literals are
compiled once per evaluation of that module expression.

---

## 16. What Studivo does not use (yet)

| Built-in                                  | Status here                                 |
| ----------------------------------------- | ------------------------------------------- |
| `Symbol`                                  | not created in app code                     |
| `BigInt`                                  | money stays `number` after Decimal boundary |
| `Proxy` / `Reflect.set` traps             | not used                                    |
| `WeakMap` / `WeakSet`                     | not used                                    |
| `Atomics` / `SharedArrayBuffer`           | not used                                    |
| `eval` / `Function` constructor           | not used                                    |
| `new Object()` / `Object.assign` as clone | spread `{ ... }` instead                    |
| `Object.create(null)`                     | not used; dictionaries are `{}` or `Map`    |

Knowing they exist matters so you do not invent a unique-key `{}` when `Set` is the built-in, and so
you do not reach for `Proxy` to debug Prisma.

---

## 17. Mental model

For each built-in call, ask:

1. **`new` or static?** `new Date` vs `Date.now`; `new Map` vs `Object.keys`.
2. **Does it throw?** `JSON.parse`, `new URL`, invalid `Date` is _not_ a throw (`Invalid Date`).
3. **Does it mutate?** `Map.set`, `Date.setHours`, `FormData.set` yes. `Object.keys`,
   `JSON.stringify`, `Array.from` no.
4. **Instance or primitive result?** `new Intl.NumberFormat().format(n)` returns a **string**.
   `Date.now()` returns a **number**.

---

## 18. Exercises on this repo

1. `uniqueSeatNumbers(["1","1","2"])` — what built-ins run, in order? Are the strings interned or
   just `===`?
2. Replace `Array.from({ length: n }, …)` with a `for` loop. Same seats? Any holey array risk?
3. `JSON.parse(formData.get("galleryImages"))` without `|| "[]"` when the field is missing. What
   throws?
4. Why `Object.hasOwn(activityGroupLabels, group)` instead of `group in activityGroupLabels`?
5. `PRODUCTION_ALLOWED_ORIGINS.has("https://studivo.ir/")` — trailing slash. Set or URL?
6. `Promise.all` vs `allSettled` for push: which user-visible failure mode does each create?
7. `Math.random` for OTP: name one attack that `crypto.getRandomValues` would address that this code
   does not claim to.
8. `new Date("2026-09-24")` vs `Date.UTC(2026, 8, 24)` — local vs UTC. Which does
   `getTehranCalendarDayDifference` use?

---

## Files cited

| File                                               | Built-ins                                                                |
| -------------------------------------------------- | ------------------------------------------------------------------------ |
| `lib/expense-categories.ts`                        | `Object.entries`                                                         |
| `hooks/use-marketing-lead-capture.ts`              | `Object.keys`                                                            |
| `app/(dashboard)/logs/_lib/activity-normalizer.ts` | `Object.hasOwn`, `Array.isArray`, `Number.isFinite`, `Intl.NumberFormat` |
| `app/onboarding/actions.ts`                        | `Array.from`, `Set`, `Error`                                             |
| `lib/date.ts`                                      | `Date`, `Number.isNaN`, `Intl.DateTimeFormat`                            |
| `lib/otp.ts`                                       | `Math`, `String`, `Date.now`                                             |
| `lib/sms.ts`                                       | `Number`, `JSON.stringify`, `Error`                                      |
| `lib/reminders.ts`                                 | `Math.min/max/round`, `Date.UTC`                                         |
| `lib/push.ts`                                      | `JSON.stringify`, `Promise.allSettled`                                   |
| `lib/marketing-leads/public-api.ts`                | `Set`, `Map`, `URL`, `Date.now`                                          |
| `app/(dashboard)/_lib/dashboard-utils.ts`          | `Map`, `Intl.NumberFormat`                                               |
| `app/(dashboard)/settings/actions.ts`              | `JSON.parse`, `FormData`                                                 |
| `public/sw.js`                                     | `URL`                                                                    |
| `lib/action-errors.ts`                             | `Reflect.get`                                                            |
| `lib/auth-plugins/phone-otp-login.ts`              | `RegExp`, `Math`                                                         |

Related: primitives (`Number` vs `number`), objects (plain `{}` vs `Map`), prototype
(`Date.prototype`), inheritance (`Error` subclasses).

Next in Data Types: a dedicated **Array** note, or **`this` / constructors** if you want to stay on
built-in construction (`new`).
