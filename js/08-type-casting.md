# Type Casting: Conversion, Coercion, and `as` — Studivo Walkthrough

Primitives were **what the value is**. This note is **when JavaScript (or you) change that type**.

Three different operations get called “casting.” They are not the same:

| Name                    | When                                                             | Changes the runtime value? | In Studivo                                   |
| ----------------------- | ---------------------------------------------------------------- | -------------------------- | -------------------------------------------- |
| **Implicit coercion**   | Engine, because an operator wanted another type                  | Yes                        | `if (!x)`, `\|\|`, `+` on numbers, `== null` |
| **Explicit conversion** | You called `Number`, `String`, `Boolean`, `parseInt`, `z.coerce` | Yes                        | Forms, env vars, Prisma Decimal, dates       |
| **TypeScript `as`**     | Compiler only. Erased                                            | **No**                     | `as const`, `as unknown as`, predicates      |

Conversion is a **new primitive** (or object). Coercion is conversion you did not write. `as` is a
**lie or a hint** to `tsc`. Mixing them is how `"09"` becomes `9`, how `0` becomes “missing,” and
how a checkbox string `"on"` becomes a boolean.

Read this with the cited files open. Predict: _what type is this after the line runs — and did
anyone check `NaN` / `""` / `0`?_

Related: `docs/javascript-primitive-types-in-studivo.md`,
`docs/typescript-vs-javascript-in-studivo.md`, `docs/typescript-js-interoperability-in-studivo.md`.

---

## 1. The rule that pays rent

JavaScript will convert for you when an operator **needs** a type:

```javascript
"5" * 2; // 10   — * wants numbers
"5" + 2; // "52" — + prefers strings if either side is a string
if ("") {
} // skipped — ToBoolean("") is false
0 || 7; // 7    — 0 is falsy
null ?? 7; // 7    — ?? only treats null/undefined as missing
```

Studivo almost never uses `"5" + 2`. It **does** use truthiness, `||`, `??`, `Number(...)`, and Zod
coerce. Money, days, and seat counts must not silently become `NaN` or `0`.

TypeScript `strict` does **not** stop runtime coercion. It only complains when you pass the wrong
_named_ type. `"5"` as `string` times `2` is still a type error in TS — unless you convert first.

---

## 2. Implicit coercion — the engine converted

### 2.1 ToBoolean — every `if`, `!`, `&&`, `||`

Falsy primitives (from the primitives note):

`false`, `0`, `-0`, `NaN`, `""`, `null`, `undefined`

Everything else is truthy: `"0"`, `"false"`, `[]`, `{}`, `new Date()`.

```24:26:lib/marketing-leads/schema.ts
function emptyStringToUndefined(value: unknown) {
  return typeof value === "string" && !value.trim() ? undefined : value;
}
```

`!value.trim()` is ToBoolean on the trimmed string. `"   "` becomes `""`, which is falsy, so the
function returns `undefined`. That is **implicit**. The `typeof value === "string"` guard is
**explicit** (a type check, not a conversion).

```202:203:lib/finance-range.ts
  const hasStart = Boolean(params.start);
  const hasEnd = Boolean(params.end);
```

Here the conversion is **explicit** `Boolean(...)`. Same ToBoolean algorithm, written on purpose.
Prefer this when the result is stored as a real `boolean`.

```66:70:lib/marketing-leads/public-api.ts
function getClientIp(request: NextRequest) {
  const forwardedFor = request.headers.get("x-forwarded-for");
  if (forwardedFor) return forwardedFor.split(",")[0]?.trim() || "unknown";

  return request.headers.get("x-real-ip")?.trim() || "unknown";
}
```

`if (forwardedFor)`: `null` from `headers.get` is falsy. `?.trim() || "unknown"`: empty string after
trim is also falsy, so you get `"unknown"`. That is the intended use of `||` — **fallback for
missing-or-empty string**, not for numbers.

### 2.2 `||` vs `??` — the `0` trap

`a || b` uses ToBoolean. `a ?? b` only replaces `null` and `undefined`.

```33:37:lib/reminders.ts
function clampReminderDaysBefore(value: number) {
  return Math.min(
    MAX_REMINDER_DAYS_BEFORE,
    Math.max(MIN_REMINDER_DAYS_BEFORE, value || DEFAULT_RENEWAL_THRESHOLD_DAYS),
  );
}
```

`value || DEFAULT` treats **`0` as missing**. A hall that wanted “remind on expiry day” (`0`) would
be clamped to the default instead. That is implicit coercion paying the wrong rent.

```227:229:app/(dashboard)/_lib/actions/reserve-seat.ts
      const membershipStatus =
        paymentStatus === "COMPLETED" ? "ACTIVE" : "PENDING";
      const paymentAmount = amount ?? Number(plan.price);
```

`amount ?? Number(plan.price)`: if `amount` is omitted (`undefined`), use the plan price. If
`amount` were `0`, `??` would keep `0`. `||` would not. For money, **`??` is the right operator**.

```161:165:app/(dashboard)/settings/actions.ts
    phoneNumber: formData.get("phoneNumber")?.toString() || undefined,
    address: formData.get("address")?.toString() || undefined,
    description: formData.get("description")?.toString() || undefined,
    publicPageEnabled: formData.get("publicPageEnabled") === "on",
    slug: formData.get("slug")?.toString() ?? "",
```

Empty optional fields: `|| undefined` (empty string is “not provided”). Slug: `?? ""` (only
`null`/`undefined` become `""`; an empty slug string is kept so Zod can reject it). Two operators,
two meanings.

### 2.3 `==` vs `===` — Studivo almost never coerces equality

Loose `==` runs `ToNumber` / `ToPrimitive` until types match. `"5" == 5` is `true`.
`null == undefined` is `true`. `"" == 0` is `true`.

Strict `===` does not convert. Different types are unequal.

The repo’s allowed exception:

```21:24:lib/date.ts
export function toDate(value: DateInput): Date | null {
  if (value == null) return null;
  const date = value instanceof Date ? value : new Date(value);
  return Number.isNaN(date.getTime()) ? null : date;
}
```

```21:22:app/(dashboard)/_lib/dashboard-utils.ts
function toTime(value: Date | string | null | undefined): number | null {
  if (value == null) return null;
```

`value == null` is the idiomatic check for **both** `null` and `undefined`. It is the one loose
equality you should recognize on sight. `value === null || value === undefined` is the same, longer.

Do not write `seat.number == 1` when `seat.number` is a string `"1"`. That would coerce. Seat labels
stay strings (primitives note). Compare with `===` after an explicit `parseInt` if you need math.

### 2.4 `+` is addition **or** concatenation

Studivo’s `+` on values is almost always **numbers**:

```60:64:app/(dashboard)/_lib/actions/membership-payments.ts
      const totalPaidBefore = membership.payments.reduce(
        (sum, p) => sum + Number(p.amount),
        0,
      );
      const totalPaidAfter = totalPaidBefore + amount;
```

`Number(p.amount)` first, _then_ `+`. If someone wrote `sum + p.amount` and Prisma Decimal
stringified, you could get `"012000"` instead of `12000`. Explicit conversion before `+` is the fix.

String building uses template literals or `.toString()`, not `"" + n`.

Unary `+value` (same as `Number(value)`) does not appear as a house style. Prefer `Number(...)`.

---

## 3. Explicit conversion — you asked for a type

### 3.1 `Number(...)` — env, query params, Decimal, input[type=number]

```45:48:lib/sms.ts
  const templateId = Number(templateIdRaw);
  if (!Number.isFinite(templateId) || templateId <= 0) {
    throw new Error(`${environmentVariable} is invalid.`);
  }
```

`process.env.*` is `string | undefined`. `Number("12")` → `12`. `Number("")` → `0`. `Number("abc")`
→ `NaN`. `Number(undefined)` → `NaN`.

**Always pair `Number` with `Number.isFinite`.** `typeof NaN === "number"` (primitives note).
Conversion succeeded at the type level and failed at the value level.

```20:20:app/(dashboard)/logs/page.tsx
  const page = Math.max(1, Number(params.page) || 1);
```

Query `page` is a string. `Number(undefined)` is `NaN`, which is falsy, so `|| 1`. `Number("0")` is
`0`, also falsy, so page `0` becomes `1` — then `Math.max(1, …)` would have done that anyway. The
`|| 1` is coercion used as “invalid page → 1.”

```246:246:app/onboarding/_components/onboarding-wizard.tsx
                onChange={(event) => updateField("seatCount", Number(event.target.value))}
```

`<input type="number">` still yields a **string** in `event.target.value`. Clearing the field yields
`""` → `Number("")` → `0`. The server schema then uses `z.coerce.number().min(1)`, so `0` is
rejected. Client conversion is convenience; **Zod is the boundary**.

### 3.2 `parseInt` vs `Number`

```169:169:app/(dashboard)/_lib/dashboard-utils.ts
    const seatNum = parseInt(seat.number, 10) || 0;
```

`parseInt("12A", 10)` → `12` (prefix). `Number("12A")` → `NaN` (entire string). Seat labels like
`"12"` sort numerically; junk becomes `0` via `||`.

**Always pass radix `10`.** `parseInt("09")` without radix is `9` in modern engines, but the habit
is non-negotiable.

`parseInt("10.9", 10)` → `10` (truncates). `Number("10.9")` → `10.9`. Money and prices want `Number`
(or Zod). Integers-from-labels want `parseInt`.

### 3.3 `String(...)` — keep digits as text

```14:16:lib/otp.ts
export function generateOtpCode(): string {
  return String(Math.floor(100000 + Math.random() * 900000));
}
```

`String(483102)` → `"483102"`. Template `` `${n}` `` is the same conversion. OTP must not stay a
`number` (leading zeros / SMS API).

```17:21:lib/marketing-leads/schema.ts
export function normalizeMarketingPhoneNumber(value: string) {
  return value
    .replace(/[۰-۹]/g, (digit) => String(persianDigits.indexOf(digit)))
    .replace(/[٠-٩]/g, (digit) => String(arabicDigits.indexOf(digit)))
```

`indexOf` returns a `number`. `String(0)` is `"0"`, a digit **character**. That is conversion with
intent, not display formatting.

`.toString()` on a value that might be `null` throws. `String(null)` is `"null"` (the four-character
string — usually wrong). Studivo uses `formData.get(...)?.toString()` so `null` short-circuits, then
`|| undefined` / `?? ""`.

### 3.4 `Boolean(...)` and `!!`

```103:103:app/(dashboard)/staff/_components/add-staff-form.tsx
                onCheckedChange={(checked) => setCreateNewUser(!!checked)}
```

Radix `onCheckedChange` can pass `boolean | "indeterminate"`. `"indeterminate"` is truthy, so `!!`
becomes `true`. `!!` is ToBoolean, written twice (`!` then `!`). Same as `Boolean(checked)` except
`"indeterminate"` is still `true` — if that matters, compare `checked === true` instead.

Prefer `=== "on"` for HTML checkboxes (next section) over `Boolean(formData.get(...))`, because a
missing checkbox is `null` (falsy) and a checked one is `"on"` (truthy) — but an input whose value
is `"false"` would be **truthy**.

### 3.5 `new Date(...)` — string/number → Date object

```21:24:lib/date.ts
export function toDate(value: DateInput): Date | null {
  if (value == null) return null;
  const date = value instanceof Date ? value : new Date(value);
  return Number.isNaN(date.getTime()) ? null : date;
}
```

This is conversion to an **object**, not a primitive. Invalid input (`""`, `"nope"`) produces a Date
whose `getTime()` is `NaN`. The function converts that failure into `null`. Never skip the
`Number.isNaN` check after `new Date`.

---

## 4. FormData and HTML — everything is a string

`FormData.get` returns `File | string | null`. Text fields are strings. Checkboxes that are
unchecked are **absent** (`null`), not `false`.

```164:164:app/(dashboard)/settings/actions.ts
    publicPageEnabled: formData.get("publicPageEnabled") === "on",
```

```456:458:app/(dashboard)/settings/actions.ts
    hasFixedSeat: formData.get("hasFixedSeat") === "on" || formData.get("hasFixedSeat") === "true",
    description: formData.get("description")?.toString() || undefined,
    isActive: formData.get("isActive") === "on" || formData.get("isActive") === "true",
```

`=== "on"` is **explicit boolean conversion** without ToBoolean. The string `"on"` is HTML’s default
checkbox value. Hidden fields in the staff form send `"true"` / `"false"` as strings, so the plan
action also accepts `"true"`.

`Boolean("false")` is `true`. `z.coerce.boolean()` treats `"false"` as **false** (Zod’s rule). Know
which converter you are using.

```33:36:app/(dashboard)/settings/_components/hall-settings-form.tsx
const notificationPreferencesClientSchema = z.object({
  renewalRemindersEnabled: z.coerce.boolean(),
  expiryRemindersEnabled: z.coerce.boolean(),
  reminderDaysBefore: z.coerce
```

Client schema: coerce. Server action for the same flags: `=== "on"`. Both are explicit. They must
agree on what the form actually posts.

---

## 5. `z.coerce` — Studivo’s boundary converter

Zod coerce runs **runtime** conversion, then validates.

| Schema                    | Input examples                      | Result                                     |
| ------------------------- | ----------------------------------- | ------------------------------------------ |
| `z.coerce.number()`       | `"12"`, `12`, `""`                  | `12`, `12`, `0` then often fails `.min(1)` |
| `z.coerce.number().int()` | `"3.2"`                             | may fail `.int()` after becoming `3.2`     |
| `z.coerce.date()`         | `"2026-09-26"`, `Date`              | `Date` or issue `"تاریخ … معتبر نیست"`     |
| `z.coerce.boolean()`      | `"on"`, `"true"`, `"false"`, `true` | boolean per Zod’s table                    |

```9:13:lib/validations/onboarding.ts
  seatCount: z.coerce
    .number()
    .int("تعداد صندلی باید عدد صحیح باشد.")
    .min(1, "تعداد صندلی باید حداقل ۱ باشد.")
    .max(500, "تعداد صندلی در هر بخش نمی‌تواند بیشتر از ۵۰۰ باشد."),
```

```36:42:app/(dashboard)/_lib/actions/reserve-seat.ts
    startsAt: z.coerce.date("تاریخ شروع معتبر نیست."),
    endsAt: z.coerce.date("تاریخ پایان معتبر نیست."),
    paymentMethod: z.enum(["CASH", "POS", "CARD_TO_CARD", "ONLINE"]),
    paymentStatus: z.enum(["COMPLETED", "PENDING"]),
    amount: z.coerce
      .number()
      .positive("مبلغ باید بزرگتر از صفر باشد.")
      .optional(),
```

Forms and JSON give strings. Operators need numbers and Dates. **`z.coerce` is where Studivo
converts at the edge**, then the rest of the action uses typed numbers/dates. That is better than
sprinkling `Number(formData.get("price"))` and hoping.

`z.number()` **without** coerce would reject `"12"`. Coerce exists because HTML is strings.

`z.preprocess` is conversion you write yourself:

```24:30:lib/marketing-leads/schema.ts
function emptyStringToUndefined(value: unknown) {
  return typeof value === "string" && !value.trim() ? undefined : value;
}

const optionalEmailSchema = z.preprocess(
  emptyStringToUndefined,
  z
```

Empty email field → `undefined` → optional schema passes. That is explicit conversion **before**
validation, not coercion inside `+` or `if`.

---

## 6. Decimal and unknown — convert, then prove

Prisma `Decimal` is an **object**. UI and `+` want a `number` primitive.

```9:15:app/(dashboard)/finance/_lib/money.ts
export function toDisplayAmount(value: DecimalLike | number | null | undefined) {
  if (typeof value === "number") {
    return Number.isFinite(value) ? value : 0;
  }

  const amount = Number(value?.toString() ?? 0);
  return Number.isFinite(amount) ? amount : 0;
}
```

Path: object → `toString()` (explicit string) → `Number(...)` (explicit number) → `Number.isFinite`
(guard). `Number(decimalObject)` can also work for some Decimal impls; going through `toString` is
the documented boundary.

```43:46:app/(dashboard)/logs/_lib/activity-normalizer.ts
function numberValue(value: unknown): number | undefined {
  if (typeof value === "number" && Number.isFinite(value)) return value;
  if (typeof value === "string" && value.trim() && Number.isFinite(Number(value))) return Number(value);
  return undefined;
}
```

Audit metadata is `unknown`. Conversion is allowed only after `typeof` and `isFinite`.
`Number("  ")` is `0`; the `.trim()` check refuses that. `Number("12px")` is `NaN` → `undefined`.
This is the gold standard for untrusted strings.

```209:209:app/(dashboard)/_lib/dashboard-utils.ts
      const price = Number(s.membership.planPrice) || 0;
```

`|| 0` after `Number` maps `NaN` **and** `0` to `0`. Fine for summing display revenue. Wrong if `0`
must mean “free plan” vs “failed parse.” `toDisplayAmount` / `numberValue` distinguish those; this
line does not.

---

## 7. TypeScript `as` is not conversion

```4:6:lib/db.ts
const globalForPrisma = global as unknown as {
  prisma: PrismaClient;
};
```

At runtime, `global` is still `global`. No new object, no `Number`, no Zod. `as` only silences
`tsc`. The double `as unknown as` is required because Node’s `typeof global` does not overlap
`{ prisma }`.

```8:10:lib/otp.ts
  PASSWORD_RESET: "PASSWORD_RESET" as OtpPurpose,
  SIGNUP: "SIGNUP" as OtpPurpose,
} as const;
```

`as const` narrows to literal types (`"PASSWORD_RESET"` not `string`). Runtime value unchanged.

```60:60:lib/sms.ts
  const record = payload as Record<string, unknown>;
```

Safe **only because** the lines above proved `typeof payload === "object" && payload !== null`. The
`as` does not convert JSON. The `typeof` checks did the work.

If you `as number` on a string, `tsc` may allow a double assertion; the engine still has a string.
**Never use `as` where you meant `Number` or `z.coerce`.**

---

## 8. What Studivo does _not_ do

| Pattern                                       | Why it is absent / rare                                          |
| --------------------------------------------- | ---------------------------------------------------------------- |
| `==` except `== null`                         | Coercing equality hides `"1"` vs `1` bugs (seat numbers, phones) |
| Unary `+x`                                    | Prefer `Number(x)` — greppable, obvious                          |
| `parseInt` without radix                      | Footgun; the one `parseInt` passes `10`                          |
| `var` + implicit globals                      | Already banned; not a conversion issue but same era of sloppy JS |
| `Number(formData.get("x"))` as the only check | Zod coerce + `.min` / `.positive` at the action boundary         |
| `as number` on FormData / JSON                | Types are not validation (TS vs JS note)                         |
| `"5" + 2` string concat                       | Templates and `.toString()`                                      |

---

## 9. Mental model

For every line that might change type, ask:

1. **Who converted?** Engine (implicit), you (`Number` / `String` / `Boolean` / `parseInt` /
   `new Date`), Zod (`z.coerce` / `preprocess`), or nobody (`as`)?
2. **What happens to `""`, `0`, `NaN`, `null`, `undefined`?** Those five are the coercion bugs.
3. **Is this a boundary?** FormData, `searchParams`, `process.env`, Prisma Decimal, JSON → convert +
   validate. Interior functions should already have numbers/dates.
4. **Did I want `false` or did I want “missing”?** `||` vs `??` vs `=== "on"` vs
   `z.coerce.boolean()`.

---

## 10. Exercises on this repo

Predict; do not edit production code.

1. `Number("")`, `Number("  ")`, `Number("09")`, `parseInt("09", 10)`, `parseInt("12px", 10)`,
   `Number("12px")`.
2. `clampReminderDaysBefore(0)` — which operator made `0` disappear? What would `??` do instead?
3. `amount ?? Number(plan.price)` if `amount` is `0`. Same line with `||`.
4. Unchecked checkbox: `formData.get("publicPageEnabled")` is `?`. `=== "on"` is `?`.
   `Boolean(formData.get(...))` is `?`.
5. `z.coerce.boolean()` on `"false"`, `"on"`, `""`, `"0"`. Then compare to `!!` of each.
6. `toDate("")` and `toDate(undefined)` — walk `== null` vs `new Date("")`.
7. `Number(params.page) || 1` when `page` is `"NaN"`, `"2"`, `"0"`, missing.
8. Why `String(persianDigits.indexOf(digit))` and not implicit concat in the replace callback?
9. `payload as Record<string, unknown>` without the `typeof` / `null` guards — what does
   `record.status === 1` do if `payload` is `null`?
10. `toDisplayAmount` vs `Number(s.membership.planPrice) || 0` for a Decimal of `0` and for a
    garbage string. Which treats them the same?

---

## Files cited

| File                                                          | Conversion shown                                   |
| ------------------------------------------------------------- | -------------------------------------------------- |
| `lib/sms.ts`                                                  | `Number` + `isFinite`; `as` after `typeof`         |
| `lib/otp.ts`                                                  | `String(number)` OTP; `as const` / `as OtpPurpose` |
| `lib/date.ts`                                                 | `== null`; `new Date`; `NaN` → `null`              |
| `lib/reminders.ts`                                            | `value \|\| default` (falsy `0`)                   |
| `lib/db.ts`                                                   | `as unknown as` — not a conversion                 |
| `lib/validations/onboarding.ts`                               | `z.coerce.number()`                                |
| `lib/marketing-leads/schema.ts`                               | `!trim`; `String(indexOf)`; `z.preprocess`         |
| `lib/marketing-leads/public-api.ts`                           | `if (str)` and `\|\| "unknown"`                    |
| `lib/finance-range.ts`                                        | `Boolean(params.start)`; `Number` on date parts    |
| `app/(dashboard)/finance/_lib/money.ts`                       | Decimal → string → number                          |
| `app/(dashboard)/logs/page.tsx`                               | `Number(params.page) \|\| 1`                       |
| `app/(dashboard)/logs/_lib/activity-normalizer.ts`            | Safe `Number` from `unknown`                       |
| `app/(dashboard)/_lib/dashboard-utils.ts`                     | `== null`; `parseInt(..., 10)`; `Number \|\| 0`    |
| `app/(dashboard)/_lib/actions/reserve-seat.ts`                | `z.coerce.date/number`; `??` for money             |
| `app/(dashboard)/_lib/actions/membership-payments.ts`         | `Number` before `+`                                |
| `app/(dashboard)/settings/actions.ts`                         | `\|\| undefined` vs `?? ""`; `=== "on"`            |
| `app/(dashboard)/settings/_components/hall-settings-form.tsx` | `z.coerce.boolean/number`                          |
| `app/(dashboard)/staff/_components/add-staff-form.tsx`        | `!!checked`                                        |
| `app/onboarding/_components/onboarding-wizard.tsx`            | `Number(event.target.value)`                       |

Related: primitives (`""` / `null` / `0` / `NaN`), TS vs JS (erasure), TS/JS interop
(`as unknown as`, JSON).

Next JS: **arrays** to finish Data Types, or `this` / constructors. Next TS: narrowing, unions, or
`any` vs `unknown`.
