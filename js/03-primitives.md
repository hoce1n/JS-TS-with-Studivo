# JavaScript Primitive Types — Studivo Walkthrough

Hoisting was _when_ a name exists. Scope was _where_. This note is **what the value is**.

JavaScript has seven primitives:

| Type        | `typeof` result    | In Studivo?                                                    |
| ----------- | ------------------ | -------------------------------------------------------------- |
| `string`    | `"string"`         | Everywhere: phones, names, OTP codes, Persian copy             |
| `number`    | `"number"`         | Money (after conversion), days, template IDs, timestamps-as-ms |
| `boolean`   | `"boolean"`        | Flags: `isActive`, `hasFixedSeat`, `showEmail`                 |
| `undefined` | `"undefined"`      | Missing optional fields, “no error”, SSR guards                |
| `null`      | `"object"` (quirk) | SQL empty columns, “no date”, CORS origin absent               |
| `symbol`    | `"symbol"`         | **Not used** in this repo                                      |
| `bigint`    | `"bigint"`         | **Not used** — money stays `number` after a Decimal boundary   |

Primitives are **not objects**. They are copied by value. They have no own methods until JavaScript
briefly boxes them (`"09".trim()` still works).

Studivo is TypeScript, so many primitives are _named_ in types. Runtime still only has the seven.
When JSON, FormData, or Prisma hand you `unknown`, the code re-checks with `typeof`.

Read this with the cited files open. Predict: _is this `""`, `null`, `undefined`, `0`, or `NaN` —
and which check would catch it?_

---

## 1. The rule that pays rent

Primitives compare by **value**. Two strings with the same characters are equal. Two numbers with
the same IEEE value are equal.

```javascript
"PASSWORD_RESET" === "PASSWORD_RESET"; // true
5 === 5; // true
true === true; // true
```

Objects compare by **identity**. That is the next note. For this note, every example below is a
primitive (or a check that _rejects_ non-primitives).

---

## 2. `string` — the default business value

Phones, names, SMS templates, seat numbers, and UI copy are strings. Even digits are often strings
until someone needs arithmetic.

### 2.1 String as identity, not as math

OTP codes are six **characters**, not an integer. Leading zeros would vanish in a `number`.

```14:16:lib/otp.ts
export function generateOtpCode(): string {
  return String(Math.floor(100000 + Math.random() * 900000));
}
```

`Math.floor(...)` produces a `number` in `100000…999999`. `String(...)` boxes that into a primitive
string (`"483102"`). The SMS API and the database both store `code` as text.

Seat numbers in onboarding are the same idea: `"1"`, `"2"`, `"A-12"` — identifiers, not quantities.

### 2.2 String methods do not mutate

```17:22:lib/marketing-leads/schema.ts
export function normalizeMarketingPhoneNumber(value: string) {
  return value
    .replace(/[۰-۹]/g, (digit) => String(persianDigits.indexOf(digit)))
    .replace(/[٠-٩]/g, (digit) => String(arabicDigits.indexOf(digit)))
    .replace(/[\s-]/g, "");
}
```

Each `.replace` returns a **new** string. `value` is unchanged. Primitives are immutable.

`persianDigits.indexOf(digit)` returns a `number` (`0`–`9`). `String(...)` turns it back into an
ASCII digit character. Persian `"۰۹۱۲…"` becomes `"0912…"`.

`indexOf` missing a character returns `-1`. `String(-1)` is `"-1"`, which would corrupt the phone.
The regex `[۰-۹]` guarantees a hit, so that path is safe.

### 2.3 Template literals are still strings

```51:55:lib/reminders.ts
function formatDaysLeft(candidate: ReminderCandidate) {
  if (candidate.kind === "expired") return "منقضی شده";
  if (candidate.daysLeft <= 0) return "امروز پایان می‌یابد";
  if (candidate.daysLeft === 1) return "۱ روز تا انقضا";
  return `${candidate.daysLeft} روز تا انقضا`;
}
```

`` `${candidate.daysLeft} روز تا انقضا` `` interpolates a `number` into a `string`. The result is
one primitive string. `daysLeft === 1` is a **number** comparison; `"1" === 1` would be false.

### 2.4 Empty string is a real value

```10:13:app/actions/marketing/lead.ts
function getFormDataString(formData: FormData, key: string) {
  const value = formData.get(key);
  return typeof value === "string" ? value : "";
}
```

FormData can yield a `File` or `null`. This helper refuses those and returns `""` — a string of
length 0, **not** `null` or `undefined`.

Empty string is truthy-false in `if (value)`, but `typeof "" === "string"`. That distinction shows
up next under `undefined`.

---

## 3. `number` — IEEE 754, not “integer”

JavaScript has one numeric primitive (ignoring `bigint`, which Studivo does not use). It is a 64-bit
float. Integers are a convention, not a type.

### 3.1 Parsing env / JSON into a number

SMS template IDs arrive as strings from the environment:

```38:50:lib/sms.ts
function getTemplateId(environmentVariable: string): number {
  const templateIdRaw = process.env[environmentVariable];

  if (!templateIdRaw) {
    throw new Error(`${environmentVariable} is missing.`);
  }

  const templateId = Number(templateIdRaw);
  if (!Number.isFinite(templateId) || templateId <= 0) {
    throw new Error(`${environmentVariable} is invalid.`);
  }

  return templateId;
}
```

Three number facts in one function:

1. `Number("12")` → `12`.
2. `Number("abc")` → `NaN`. `NaN` is still `typeof "number"`.
3. `Number.isFinite(NaN)` is `false`. So is `Number.isFinite(Infinity)`.

Never use `typeof x === "number"` alone if `NaN` would be a disaster. Audit logs do both:

```43:46:app/(dashboard)/logs/_lib/activity-normalizer.ts
function numberValue(value: unknown): number | undefined {
  if (typeof value === "number" && Number.isFinite(value)) return value;
  if (typeof value === "string" && value.trim() && Number.isFinite(Number(value))) return Number(value);
  return undefined;
}
```

`Number("")` is `0`, which is finite. The `value.trim()` guard stops empty strings becoming a fake
zero amount.

### 3.2 Money: Decimal in the DB, `number` on the screen

Prisma `Decimal` is an **object**. The finance boundary converts it once:

```9:16:app/(dashboard)/finance/_lib/money.ts
export function toDisplayAmount(value: DecimalLike | number | null | undefined) {
  if (typeof value === "number") {
    return Number.isFinite(value) ? value : 0;
  }

  const amount = Number(value?.toString() ?? 0);
  return Number.isFinite(amount) ? amount : 0;
}
```

If it is already a `number`, keep it (unless `NaN` / `Infinity` → `0`). Otherwise call `.toString()`
on the Decimal-like object, then `Number(...)`.

This is the honest Studivo trade: **JavaScript `number` cannot represent all toman amounts with
integer cents if they get huge**, but hall prices fit. `bigint` was not introduced. If you need
exact currency later, that is a new type decision — not a silent `number` hope.

### 3.3 Arithmetic and integer-looking results

```146:152:lib/date.ts
export function formatDurationFa(hours: number): string {
  const totalMinutes = Math.round(hours * 60);
  const h = Math.floor(totalMinutes / 60);
  const m = totalMinutes % 60;
```

`hours` can be fractional (`1.5`). `*` `/` `%` all return `number`. `Math.round` / `Math.floor`
still return `number` (`1` not `1n`).

Reminder day math uses `Number` on split strings:

```40:47:lib/reminders.ts
function getTehranCalendarDayDifference(fromKey: string, toKey: string) {
  const [fromYear, fromMonth, fromDay] = fromKey.split("-").map(Number);
  const [toYear, toMonth, toDay] = toKey.split("-").map(Number);

  return Math.round(
    (Date.UTC(toYear, toMonth - 1, toDay) -
      Date.UTC(fromYear, fromMonth - 1, fromDay)) /
      DAY_IN_MS,
  );
}
```

`"2026-09-23".split("-").map(Number)` → `[2026, 9, 23]`, all primitives of type `number`.
`toMonth - 1` is calendar math (JS `Date` months are 0-based). `Math.round` because dividing
milliseconds can land on `30.999999` — float residue, not a second calendar.

### 3.4 Falsy numbers: `0` vs “missing”

```33:37:lib/reminders.ts
function clampReminderDaysBefore(value: number) {
  return Math.min(
    MAX_REMINDER_DAYS_BEFORE,
    Math.max(MIN_REMINDER_DAYS_BEFORE, value || DEFAULT_RENEWAL_THRESHOLD_DAYS),
  );
}
```

`value || DEFAULT` treats `0`, `NaN`, and `undefined` (if it leaked) as “use default”. `0` days
before expiry is not a valid product value here, so the falsy trap is intentional.

If `0` were a legal amount, this would be a bug. Payment code uses `Number(x) || remainingBalance`
in `ReserveForm` for the same reason: `0` is not a payment they want to record. Know when
falsy-number collapse is a feature.

---

## 4. `boolean` — only `true` and `false`

Booleans are the smallest primitive. Almost every operational flag in Schema V2 is one: `isActive`,
`hasFixedSeat`, `publicPageEnabled`.

Runtime checks often _produce_ booleans with `===` / `typeof`:

```53:57:lib/sms.ts
function isSmsIrSuccessResponse(
  payload: unknown,
): payload is SmsIrVerifySuccessResponse {
  if (typeof payload !== "object" || payload === null) {
    return false;
  }
```

`typeof payload !== "object"` is a boolean. `payload === null` is a boolean. `||` combines them.

Lead CORS:

```47:48:lib/marketing-leads/public-api.ts
export function isAllowedMarketingLeadOrigin(origin: string | null): origin is string {
  if (!origin) return false;
```

`!origin` is true for `null`, `undefined`, and `""`. The return type `origin is string` is
TypeScript (a type predicate). At runtime this function still just returns a boolean.

Form options are real boolean primitives, not `"true"` strings:

```54:57:lib/marketing-lead-capture.ts
  options: {
    showEmail: boolean;
    showNotes: boolean;
    fixedStudyhallName?: string;
  }
```

```64:65:lib/marketing-lead-capture.ts
    ...(options.showEmail ? ["email" as const] : []),
    ...(options.showNotes ? ["notes" as const] : []),
```

A HTML checkbox that submits `"on"` is a **string**. Someone must convert it before it becomes this
boolean. That conversion is object/FormData territory (next notes). Here, once you are in
`showEmail`, it is `true` or `false` only.

---

## 5. `undefined` — “nobody wrote this”

`undefined` means the binding or property was never given a value (or was explicitly set to
`undefined`).

### 5.1 Optional result: no error message

```16:19:lib/marketing-lead-capture.ts
export function validateMarketingLeadField(
  name: MarketingLeadFieldName,
  value: string
): string | undefined {
```

```48:49:lib/marketing-lead-capture.ts
  return undefined;
}
```

Valid field → `undefined`, not `""` and not `null`. Callers test `if (error)`. Both `undefined` and
`""` are falsy, but `undefined` means “skip this key”:

```73:76:lib/marketing-lead-capture.ts
    const error = validateMarketingLeadField(field, value);

    if (error) errors[field] = error;
    return errors;
```

They only assign when the value is a non-empty string. The map stays a `Partial<Record<…>>` of real
messages.

### 5.2 Empty string → `undefined` for optional Zod fields

```24:26:lib/marketing-leads/schema.ts
function emptyStringToUndefined(value: unknown) {
  return typeof value === "string" && !value.trim() ? undefined : value;
}
```

Product rule: blank email is “not provided”, not “invalid empty email”. Zod’s `.optional()` treats
`undefined` as absent and `""` as a string that fails `.email()`. This helper is the bridge.

`typeof value === "string"` first: if someone passed `null`, it is left as `null` (Zod will fail).
Only empty **strings** become `undefined`.

### 5.3 Missing globals: `typeof` so the name itself is safe

```20:20:hooks/usePWA.ts
    typeof Notification !== 'undefined' ? Notification.permission : 'default'
```

```7:8:hooks/use-mobile.ts
    if (typeof window === "undefined") {
      return false
```

If you write `Notification.permission` on the server, you get `ReferenceError`.
`typeof Notification` is legal even when the binding does not exist: the result is the string
`"undefined"`.

That string is a primitive. The comparison is string vs string.

### 5.4 Optional parameters

```78:79:lib/date.ts
  locale: string | undefined = APP_JALALI_LOCALE,
) {
```

Omitted argument → `undefined` → default kicks in. Explicit `undefined` also triggers the default.
Explicit `null` would **not** (and the type forbids it).

---

## 6. `null` — “we know it is empty”

`null` is an assigned empty value. Databases and APIs use it on purpose.

### 6.1 `== null` catches both

```7:24:lib/date.ts
type DateInput = Date | string | number | null | undefined;
// ...
export function toDate(value: DateInput): Date | null {
  if (value == null) return null;
  const date = value instanceof Date ? value : new Date(value);
  return Number.isNaN(date.getTime()) ? null : date;
}
```

`value == null` is the rare `==` you _want_: it is true for `null` **and** `undefined`, false for
`0`, `""`, and `false`.

Invalid dates (`new Date("nope")`) have `getTime() === NaN`. `Number.isNaN` turns that into `null`
too. Three inputs, one output primitive/object story: missing → `null`, unparsable → `null`, valid →
a `Date` **object** (not a primitive).

Display layer maps `null` to the string `"—"`:

```31:32:lib/date.ts
  const date = toDate(value);
  if (!date) return "—";
```

`!date` is true for `null`. The UI never prints the word `null`.

### 6.2 SQL-shaped nulls

```13:14:lib/marketing-leads/create-marketing-lead.ts

```

Optional email/notes go into Prisma as `input.email ?? null`. `??` only replaces `null`/`undefined`.
Empty string would pass through — that is why the schema converted `""` to `undefined` first. Chain:

1. `""` (string primitive)
2. preprocess → `undefined`
3. `?? null` → `null`
4. PostgreSQL NULL

Three primitives, one column.

CORS origin from the Request header is `string | null` (Fetch standard: missing header → `null`, not
`undefined`).

```47:48:lib/marketing-leads/public-api.ts
export function isAllowedMarketingLeadOrigin(origin: string | null): origin is string {
  if (!origin) return false;
```

### 6.3 `typeof null === "object"`

This is the oldest JS bug, still shipped.

```56:57:lib/sms.ts
  if (typeof payload !== "object" || payload === null) {
    return false;
```

```6:6:lib/action-errors.ts
  if (typeof error === "object" && error !== null && "message" in error) {
```

Every careful object guard in Studivo is **two** checks: `typeof === "object"` **and** `!== null`.
Skip the second, and `null` looks like a successful payload.

`typeof null` is the string `"object"`. The value `null` is still a primitive. Do not confuse the
`typeof` lie with the type.

---

## 7. `symbol` and `bigint` — absent on purpose

A repo-wide search finds **no** `Symbol(` and **no** `BigInt` / `bigint`.

That is a curriculum fact:

- You do not need `symbol` to understand Studivo. React uses symbols internally (`react.element`);
  application code does not create them.
- You do not need `bigint` until money or IDs exceed `Number.MAX_SAFE_INTEGER` (`2^53 - 1`). Hall
  prices and seat counts do not.

When a tutorial dives into `Symbol.iterator`, come back here and notice the product still ships
without it.

---

## 8. `typeof` is a string, not a type tag object

`typeof` always returns one of:
`"undefined" | "boolean" | "number" | "bigint" | "string" | "symbol" | "function" | "object"`.

Studivo uses it as a **runtime** filter on `unknown`:

```1:13:lib/action-errors.ts
export function getActionErrorMessage(error: unknown, fallback = "انجام عملیات ناموفق بود.") {
  if (error instanceof Error && error.message.trim()) {
    return error.message;
  }

  if (typeof error === "object" && error !== null && "message" in error) {
    const message = Reflect.get(error, "message");
    if (typeof message === "string" && message.trim()) {
      return message;
    }
  }

  return fallback;
}
```

Walk:

1. `error` might be anything (catch bindings are `unknown`-ish).
2. If it is an `Error` **object**, use `.message` (a string primitive).
3. Else if it is a non-null object with a string `message`, use that.
4. Else return `fallback` — another string primitive.

`instanceof` is for objects. `typeof` is for primitives (and the `"object"` / `"function"` buckets).
You will need both.

JSON success payload:

```62:66:lib/sms.ts
  return (
    record.status === 1 &&
    typeof record.data === "object" &&
    record.data !== null &&
    typeof (record.data as Record<string, unknown>).messageId === "number"
  );
```

`status === 1` is number equality (JSON numbers are JS numbers). `messageId` must be
`typeof "number"` — a string `"123"` from a sloppy API would fail, which is the point.

---

## 9. Truthiness vs type

Falsy primitives: `false`, `0`, `-0`, `0n`, `""`, `null`, `undefined`, `NaN`.

Studivo `if (x)` checks mix these. Know which one you meant.

| Check                                                 | Allows `""`   | Allows `0`   | Allows `null` | Typical use                            |
| ----------------------------------------------------- | ------------- | ------------ | ------------- | -------------------------------------- |
| `if (value)`                                          | no            | no           | no            | “has a message”, `if (error)`          |
| `value == null`                                       | yes           | yes          | **catches**   | `toDate`                               |
| `value === null`                                      | yes           | yes          | only `null`   | SMS payload                            |
| `typeof value === "string"`                           | **yes**       | n/a          | no            | FormData, Zod preprocess               |
| `typeof value === "number" && Number.isFinite(value)` | n/a           | **yes**      | no            | amounts                                |
| `value \|\| default`                                  | replaces `""` | replaces `0` | replaces      | `clampReminderDaysBefore`, IP fallback |
| `value ?? default`                                    | keeps `""`    | keeps `0`    | replaces      | Prisma `email ?? null`                 |

IP helper:

```66:70:lib/marketing-leads/public-api.ts
function getClientIp(request: NextRequest) {
  const forwardedFor = request.headers.get("x-forwarded-for");
  if (forwardedFor) return forwardedFor.split(",")[0]?.trim() || "unknown";

  return request.headers.get("x-real-ip")?.trim() || "unknown";
}
```

`headers.get` returns `string | null`. `if (forwardedFor)` drops `null` and `""`.
`?.trim() || "unknown"` drops empty trim results. The fallback `"unknown"` is a string primitive
used as a Map key — a real value, not `null`, so the rate-limit bucket still works.

---

## 10. Copy by value (why primitives stay safe)

```javascript
function clampReminderDaysBefore(value: number) {
  value = 3; // only this binding
}
```

Callers pass a number. Reassigning `value` inside does not change the argument at the call site.
Same for strings:

```javascript
normalizeMarketingPhoneNumber(phone); // phone in the form is unchanged
```

The function **returns** a new string. The lead hook then writes it back:

```javascript
event.currentTarget.value = normalizeMarketingPhoneNumber(event.currentTarget.value);
```

That assignment is a DOM **object** property write (next topic). The primitive itself was copied
into the function, transformed, copied out.

---

## 11. TypeScript names vs runtime primitives

`z.infer<typeof onboardingSchema>` in onboarding is **not** JavaScript `typeof`. It is a type-level
query. After compile, it is gone.

Runtime `typeof payload !== "object"` stays in the shipped JS.

When you see `string | null` in a signature, that is two possible primitives. When you see
`DateInput = Date | string | number | null | undefined`, `Date` is an object; the rest are
primitives (or missing). `toDate` exists to collapse that union to `Date | null`.

Do not expect TypeScript to check JSON from SMS.ir. That is why `isSmsIrSuccessResponse` uses
runtime `typeof`.

---

## 12. Mental model

For every value, ask:

1. **Which of the seven is it?** (`typeof`, plus an extra `=== null`.)
2. **Is `NaN` possible?** If yes, `Number.isFinite`.
3. **Is empty possible?** Decide among `""`, `null`, `undefined` — Studivo uses all three with
   different meanings.
4. **Should this be a string even if it looks numeric?** Phones, OTP, seat labels: yes. Template
   IDs, days, money-after-boundary: no.

---

## 13. Exercises on this repo

1. `generateOtpCode` without `String(...)`: what type is `code`? What happens to a theoretically
   6-digit value under `100000` if the formula ever changed?
2. In `emptyStringToUndefined`, if `value` is `null`, what is returned? Why not turn `null` into
   `undefined` too?
3. `toDisplayAmount(NaN)` and `toDisplayAmount("12.5")` — results?
4. `toDate("")` — `== null`? `new Date("")`? Final return?
5. `typeof null` in `isSmsIrSuccessResponse` if someone deleted `|| payload === null`. What would
   `record.status === 1` do?
6. Why is `daysLeft === 1` not `daysLeft === "1"` in `formatDaysLeft`?
7. Could `clampReminderDaysBefore(0)` mean “remind on the expiry day”? What does the function
   actually do?
8. There is no `bigint` in Studivo. Pick a finance amount that would be unsafe as `number`. (Hint:
   `Number.MAX_SAFE_INTEGER`.)

---

## Files cited

| File                                               | Why it is here                                               |
| -------------------------------------------------- | ------------------------------------------------------------ |
| `lib/otp.ts`                                       | `number` → `string` OTP; numeric expiry math                 |
| `lib/marketing-leads/schema.ts`                    | Immutable string normalize; `""` → `undefined`               |
| `lib/reminders.ts`                                 | Template strings; `=== 1`; falsy `0`; `Number(...)` on dates |
| `app/actions/marketing/lead.ts`                    | `typeof === "string"` vs `""`                                |
| `lib/sms.ts`                                       | `Number` + `Number.isFinite`; `typeof` + `null`              |
| `app/(dashboard)/logs/_lib/activity-normalizer.ts` | Safe number from `unknown`                                   |
| `app/(dashboard)/finance/_lib/money.ts`            | Decimal object vs `number` primitive                         |
| `lib/date.ts`                                      | `== null`; `NaN` dates; duration arithmetic                  |
| `lib/marketing-lead-capture.ts`                    | `string \| undefined` errors; real booleans                  |
| `lib/marketing-leads/public-api.ts`                | `string \| null` origin; falsy IP fallback                   |
| `lib/action-errors.ts`                             | `typeof` on `unknown`; string messages                       |
| `hooks/usePWA.ts` / `hooks/use-mobile.ts`          | `typeof window` / `Notification`                             |
| `lib/marketing-leads/create-marketing-lead.ts`     | `?? null` into the database                                  |

Next in Data Types: **objects** (including arrays, `Date`, Prisma results) — reference values, as
opposed to everything in this note.
