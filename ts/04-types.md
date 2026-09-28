# TypeScript Types — Studivo Walkthrough

Install/config was **how the checker boots**. This note is **the type tree** TypeScript uses once it
is on:

```text
TypeScript Types
├── Primitive Types
├── Object Types
├── Top Types
├── Bottom Types
└── Type Assertions
```

These are **compile-time names**. After `next build` they are gone (TS vs JS note). Runtime still
has JS primitives, objects, and coercion (casting note). `strict: true` is why the names below
actually bite.

Read this with the cited files open. Predict: _is this a primitive, an object shape, “I don’t know
yet,” “this cannot happen,” or a lie to the checker?_

Related: `docs/javascript-primitive-types-in-studivo.md`, `docs/javascript-objects-in-studivo.md`,
`docs/javascript-type-casting-in-studivo.md`, `docs/typescript-vs-javascript-in-studivo.md`,
`docs/typescript-installation-and-configuration-in-studivo.md`.

---

## 1. The rule that pays rent

Every TypeScript type is one of:

| Kind          | Question it answers                     | Studivo default                                                          |
| ------------- | --------------------------------------- | ------------------------------------------------------------------------ |
| **Primitive** | Which of JS’s seven (plus TS’s `void`)? | Annotate what OTP, days, flags _are_                                     |
| **Object**    | Which properties / call signatures?     | `type Foo = { … }` almost everywhere; one `interface` nest in legal copy |
| **Top**       | Could this be anything?                 | `unknown` at JSON / `catch` / FormData. **`any` is unused**              |
| **Bottom**    | Can this exist?                         | `never` to make a union exclusive; almost no `never` functions           |
| **Assertion** | Trust me, checker                       | `as const`, `as unknown as`, `as File \| null` — not conversion          |

A **union** (`string | null`) is still built from these. A **generic** (`ActionResult<T = unknown>`)
is a hole you fill with one of these.

---

## 2. Primitive types

JS has seven primitives. TypeScript names six of them as types (`symbol` / `bigint` unused here). It
also names **literal** primitives (`1`, `"ACTIVE"`) and the nullish ones (`null`, `undefined`).

| TS type             | JS `typeof`   | In Studivo                                |
| ------------------- | ------------- | ----------------------------------------- |
| `string`            | `"string"`    | Phones, OTP, names, Persian copy          |
| `number`            | `"number"`    | Days, template IDs, money _after_ Decimal |
| `boolean`           | `"boolean"`   | `hasFixedSeat`, `ok`                      |
| `undefined`         | `"undefined"` | Optional fields, “no error”               |
| `null`              | `"object"`    | SQL empty, “no date”                      |
| `void`              | —             | Not used as an annotation in this repo    |
| `symbol` / `bigint` | —             | **Absent**                                |

### 2.1 Named primitives on functions

```14:16:lib/otp.ts
export function generateOtpCode(): string {
  return String(Math.floor(100000 + Math.random() * 900000));
}
```

Return type `string` is a **primitive type**. `Math.floor` is a `number`. `String(...)` is a
**runtime conversion** (casting note). The `: string` is a **compile-time promise**. If you returned
`483102` without `String`, `tsc` would error under `strict`.

```38:50:lib/sms.ts
function getTemplateId(environmentVariable: string): number {
  const templateIdRaw = process.env[environmentVariable];
  // ...
  const templateId = Number(templateIdRaw);
  if (!Number.isFinite(templateId) || templateId <= 0) {
    throw new Error(`${environmentVariable} is invalid.`);
  }

  return templateId;
}
```

Parameter `string`, return `number`. `process.env[...]` is `string | undefined` — a **union of
primitives**. The `if (!templateIdRaw)` narrows away `undefined`. `Number` + `isFinite` turn the
remaining `string` into a `number` the checker already expected.

### 2.2 Literal primitive types

```15:15:lib/sms.ts
  status: 1;
```

`status: 1` is the number **literal** `1`, not `number`. SMS.ir success is _exactly_ `1`. A
`status: number` field would accept `0`.

```15:15:lib/finance-range.ts
export type FinancePreset = "month" | "quarter" | "year";
```

Three **string literal** primitives. `"Month"` is not in the type. This is still the primitive layer
— a union of strings, not an object.

```13:13:lib/reminders.ts
type ReminderKind = "renewal" | "expired";
```

Same idea for reminder SMS.

### 2.3 `null` and `undefined` are primitive types

```7:7:lib/date.ts
type DateInput = Date | string | number | null | undefined;
```

`Date` is an **object** type (built-in). `string` and `number` are primitives. `null` and
`undefined` are primitive types of their own. `strictNullChecks` (inside `strict`) makes `string`
**not** include `null`. That is why `toDate` returns `Date | null` instead of pretending every call
yields a `Date`.

```12:12:app/(dashboard)/logs/_lib/activity-normalizer.ts
  actor: { id: string; name: string; email: string | null } | null;
```

`email: string | null` — DB column. Outer `| null` — maybe no actor (system). Two different
absences, both primitive `null`.

Optional `end?: string` in `FinanceRangeParams` is `string | undefined`. Optional **property** ≠
`null`. Prisma uses `null`; query params use `undefined` / omitted. Do not mix them in a type unless
the runtime mix exists (`DateInput`).

### 2.4 What primitive types are _not_

They do not run. `phoneNumber: string` does not prove the string is `09xxxxxxxxx`. Zod / `typeof`
still sit at the boundary (interop note).

---

## 3. Object types

If it is not a primitive (or top/bottom), it is some **shape**: properties, arrays, callables,
`Date`, Prisma rows, `Record<K, V>`.

JS objects were bags of properties (objects note). TS object types **name the bag**.

### 3.1 Type aliases for shapes — the house style

```3:26:lib/sms.ts
type SmsIrVerifyParameter = {
  name: string;
  value: string;
};

type SmsIrVerifyRequest = {
  mobile: string;
  templateId: number;
  parameters: SmsIrVerifyParameter[];
};

type SmsIrVerifySuccessResponse = {
  status: 1;
  message: string;
  data: {
    messageId: number;
    cost: number;
  };
};
```

- Nested object types (`data: { messageId; cost }`).
- Arrays are object types too: `SmsIrVerifyParameter[]`.
- Property types are primitives (`string`, `number`) plus the literal `1`.

```5:9:lib/push.ts
export type PushPayload = {
  title: string;
  body: string;
  url?: string;
};
```

`url?` — optional property (`string | undefined`). Notification payload is composition, not a class.

```17:38:lib/finance-range.ts
export type FinanceRangeParams = {
  start?: string;
  end?: string;
};

type JalaliCivilDate = {
  year: number;
  month: number;
  day: number;
};

export type FinanceRange = {
  start: string;
  end: string;
  rangeStart: Date;
  rangeEndExclusive: Date;
};
```

`Date` in `FinanceRange` is the **built-in object type** from `lib: ["dom", "esnext"]` (install
note). `JalaliCivilDate` is three primitive fields — still an object type because it has properties.

### 3.2 `interface` — rare, used for a document tree

```1:16:lib/legal/contract-template.ts
export interface ContractContentBlock {
  type: 'paragraph' | 'list';
  content: string[];
}

export interface ContractSection {
  title: string;
  blocks: ContractContentBlock[];
}

export interface ContractTemplate {
  title: string;
  subtitle: string;
  sections: ContractSection[];
  footer: string;
}
```

Most of Studivo uses `type Foo = { … }`. Legal copy uses `interface`. For these shapes they are
equivalent. `interface` can be **merged** if declared twice; `type` cannot. Prefer `type` unless you
need merge or a public class-like contract. Do not invent `class Member` — domain models stay object
types + functions (prototypal inheritance note).

### 3.3 `Record<K, V>` — object as dictionary

```17:25:lib/lead-labels.ts
export const STATUS_LABELS: Record<LeadStatus, string> = {
  NEW: "جدید",
  CONTACTED: "تماس گرفته شده",
  DEMO_SCHEDULED: "دمو برنامه‌ریزی‌شده",
  DEMO_COMPLETED: "دمو برگزار شده",
  NEGOTIATION: "در حال مذاکره",
  CONVERTED: "تبدیل‌شده",
  LOST: "از دست رفته",
};
```

`Record<LeadStatus, string>` means **every** `LeadStatus` key must be present. Forget `LOST` and
`strict` fails. That is an object type indexed by a union of string literals.

```3:3:app/(dashboard)/logs/_lib/activity-normalizer.ts
export type AuditMetadata = Record<string, unknown>;
```

Open dictionary: any string key, values are the **top type** `unknown`. JSON metadata is not a
closed shape.

```13:15:lib/marketing-leads/schema.ts
export type MarketingLeadFieldErrors = Partial<
  Record<MarketingLeadFieldName, string>
>;
```

`Partial<…>` makes every key optional — object type transformer. Missing field errors are
`undefined`, not empty string.

### 3.4 Intersection — object types glued

```18:20:lib/marketing-leads/public-api.ts
type GlobalWithMarketingLeadRateLimit = typeof globalThis & {
  marketingLeadRateLimitStore?: RateLimitStore;
};
```

`typeof globalThis` (object type of the host) **and** an extra optional property. Intersection `&`
is still an object type.

```6:9:components/pwa/InstallPrompt.tsx
type BeforeInstallPromptEvent = Event & {
  prompt: () => Promise<void>
  userChoice: Promise<{ outcome: "accepted" | "dismissed"; platform: string }>
}
```

DOM `Event` plus methods Chrome adds. `prompt: () => Promise<void>` is a **function** object type;
`void` here is the promise’s inner primitive-like “no useful value.”

### 3.5 Inline object types on parameters

```18:24:lib/otp.ts
export async function createOtpVerification({
  phoneNumber,
  purpose,
}: {
  phoneNumber: string;
  purpose: OtpPurpose;
}) {
```

Destructured parameter annotated with an anonymous object type. Same as naming
`type CreateOtpInput = { … }`. Studivo inlines when the shape is used once.

### 3.6 Discriminated object unions (objects + literals)

```65:69:lib/otp.ts
  if (!record) {
    return { ok: false as const, error: "کد تایید نامعتبر یا منقضی شده است." };
  }

  return { ok: true as const, record };
```

Two object types: `{ ok: false; error: string }` vs `{ ok: true; record: … }`. `ok` is a **literal
primitive** used as a tag. After `if (result.ok)`, `error` does not exist on the true branch. That
is object types plus literals, not a class hierarchy.

---

## 4. Top types — `unknown` and `any`

A **top type** is a supertype of everything. Every value is assignable _to_ it. You may not use it
as a `string` until you narrow.

### 4.1 `unknown` — Studivo’s only top type

```1:1:lib/action-errors.ts
export function getActionErrorMessage(error: unknown, fallback = "انجام عملیات ناموفق بود.") {
```

```31:31:features/audit/server/helpers.ts
export function actionError<T = unknown>(error: unknown, fallback: string): ActionResult<T> {
```

```53:55:lib/sms.ts
function isSmsIrSuccessResponse(
  payload: unknown,
): payload is SmsIrVerifySuccessResponse {
```

```10:10:app/(dashboard)/logs/_lib/activity-normalizer.ts
  metadata: unknown;
```

Use `unknown` when the value **crossed a seam**: `catch`, `JSON.parse`, `FormData`, Prisma JSON,
server-action results, marketing POST body.

Operations allowed on `unknown` without narrowing: pass it to another `unknown` parameter, compare
with `===`, use it as `unknown`. Not `.message`, not `.trim()`, not `+`.

Narrowing (runtime, stays in JS):

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

`instanceof Error` → object type `Error`. `typeof === "object"` + `!== null` → still not a named
shape, so `Reflect.get` returns `unknown`/`any`-like until `typeof message === "string"`.

`ActionResult<T = unknown>` — generic default is the top type. Callers specialize
(`ActionResult<Seat>`) or leave data as `unknown`.

### 4.2 `any` — not in this repo

A grep for `: any` in `*.ts(x)` is empty. `any` is the _other_ top type: assignable **to and from**
everything, with **no** narrowing required. It turns `strict` off locally.

Do not add `any` to silence FormData or JSON. Use `unknown` + Zod / `typeof` / predicates.
`skipLibCheck` already avoids fighting `any` inside `node_modules`.

If you wrote `payload: any` in `isSmsIrSuccessResponse`, `payload.status` would typecheck even if
SMS.ir returned a string. That is the bug `unknown` exists to prevent.

### 4.3 `unknown` vs object vs primitive

`AuditMetadata = Record<string, unknown>` is an **object** type whose values are **top**.
`metadata: unknown` on the log row is “could even be a string or null.” `toAuditMetadata` narrows to
the object form or returns `{}`.

---

## 5. Bottom types — `never`

A **bottom type** is assignable _to_ every type, but only an empty set of values inhabits it.
`never` means **this cannot happen**.

### 5.1 Exclusive union fields

```54:56:app/(dashboard)/finance/_data/get-finance-dashboard.ts
export type FinanceAttention =
  | { data: FinanceAttentionData; error?: never }
  | { data: null; error: string };
```

Success branch: `data` is present; `error` must **not** be a `string`. `error?: never` means “you
may omit `error`, but you cannot set it.” Failure branch: `data: null` (primitive) +
`error: string`.

That is how TS encodes **XOR** on object types without a `class`. Compare OTP’s
`{ ok: true as const }` / `{ ok: false as const }` — same idea with a discriminant instead of
`never`.

### 5.2 What Studivo does not do with `never`

No `function fail(msg: string): never { throw … }`. Throws are unannotated; inferred return is
`void` or the success type.

No exhaustive `switch` with a `default: satisfies never` helper in app code. Literal unions are
small and handled with `if` / `Record` maps (`STATUS_LABELS` already forces every key).

### 5.3 `as never` is not a bottom-type design — it is an assertion

```51:51:app/platform/_components/venues-table.tsx
    <Badge variant={variant as never}>
```

The computed `variant` string is wider than `Badge`’s variant union. `as never` tells the checker
“treat this as the empty type so it is assignable to the prop.” Runtime still passes a string. That
is a **type assertion** (next section), not modeling impossibility. Prefer narrowing the ternary to
the Badge union, or a `satisfies` map.

---

## 6. Type assertions

Assertions (`as T`, `as const`, angle brackets unused in TSX) **do not convert**. Casting note:
`Number("12")` changes the value; `as number` does not.

### 6.1 `as const` — widen less

```7:10:lib/otp.ts
export const OTP_PURPOSE = {
  PASSWORD_RESET: "PASSWORD_RESET" as OtpPurpose,
  SIGNUP: "SIGNUP" as OtpPurpose,
} as const;
```

Without `as const`, object values would be `string`. With it, they are the literals
`"PASSWORD_RESET"` and `"SIGNUP"`. The inner `as OtpPurpose` asserts those literals match the Prisma
enum type.

```7:15:lib/lead-labels.ts
export const ALL_STATUSES = [
  "NEW",
  "CONTACTED",
  "DEMO_SCHEDULED",
  "DEMO_COMPLETED",
  "NEGOTIATION",
  "CONVERTED",
  "LOST",
] as const satisfies readonly LeadStatus[];
```

`as const` → readonly tuple of literals. `satisfies readonly LeadStatus[]` **checks** the tuple
against Prisma’s enum **without** widening to `LeadStatus[]`. Assertion + extra check. If Prisma
adds a status, `STATUS_LABELS: Record<LeadStatus, string>` fails until you add a label. If you add a
typo to `ALL_STATUSES`, `satisfies` fails.

```69:69:lib/push.ts
    return { ok: true as const };
```

Literal `true`, so the union discriminant is not `boolean`.

### 6.2 `as` after you already narrowed — cheap widening of the _name_

```56:60:lib/sms.ts
  if (typeof payload !== "object" || payload === null) {
    return false;
  }

  const record = payload as Record<string, unknown>;
```

After the guard, TS still types `payload` as `object`. `object` does not allow `record.status`.
Assert to `Record<string, unknown>` (object type of string keys → top values), then **runtime**
`typeof` on fields. The assertion does not prove `status === 1`; the comparisons do. The predicate
return `payload is SmsIrVerifySuccessResponse` then narrows callers.

```34:36:app/(dashboard)/logs/_lib/activity-normalizer.ts
export function toAuditMetadata(value: unknown): AuditMetadata {
  if (!value || typeof value !== "object" || Array.isArray(value)) return {};
  return value as AuditMetadata;
}
```

Same pattern: guards, then `as` to the dictionary type.

### 6.3 Double assertion — incompatible object types

```4:6:lib/db.ts
const globalForPrisma = global as unknown as {
  prisma: PrismaClient;
};
```

`global` (Node) and `{ prisma: PrismaClient }` do not overlap enough for a single `as`. **Up to
top** (`as unknown`) **then down** to the shape. Runtime: still `global`. First import assigns
`globalForPrisma.prisma`. Without this, HMR would construct extra Prisma clients.

Prefer `typeof globalThis & { … }` (rate-limit store) when the extra fields are optional and the
intersection is honest.

### 6.4 Assertions at untyped hosts — FormData, enums, DOM

```20:20:app/api/upload/image/route.ts
  const file = formData.get("file") as File | null;
```

`FormData.get` is `FormDataEntryValue | null` (`File | string | null`). Assertion skips
distinguishing string vs File. `if (!file)` treats `null` and would treat `""` as truthy if a string
slipped in. Safer: `instanceof File` (interop exercises). The `as` is a boundary shortcut, not
conversion to `File`.

```70:70:app/(dashboard)/_lib/actions/membership-payments.ts
          method: method as PaymentMethod,
```

Form/Zod string asserted to Prisma enum. If Zod already `z.enum([...])`, the `as` is redundant or
hiding a mismatch. Prefer the Zod-inferred type.

```20:20:components/pwa/InstallPrompt.tsx
    !(navigator as NavigatorWithMSStream).MSStream
```

Assert `navigator` to an intersection that _might_ have `MSStream?: unknown`. Reading an optional
top-typed field. IE11 quirk. Runtime: property missing is `undefined`.

### 6.5 Assertion vs annotation vs conversion vs `satisfies`

| Syntax                | Changes value?      | Checks assignability?                   | Example                                    |
| --------------------- | ------------------- | --------------------------------------- | ------------------------------------------ |
| `: string` annotation | No                  | Yes (the expression must be `string`)   | `generateOtpCode(): string`                |
| `as string`           | No                  | Weak (will double-assert via `unknown`) | `global as unknown as { prisma }`          |
| `Number(x)`           | **Yes**             | Runtime                                 | `Number(templateIdRaw)`                    |
| `satisfies T`         | No                  | Yes, keeps inferred literal type        | `as const satisfies readonly LeadStatus[]` |
| `z.coerce.number()`   | **Yes** + validates | Runtime                                 | onboarding `seatCount`                     |

If you need a `number`, convert. If you need the checker to believe a JSON object is
`SmsIrVerifySuccessResponse`, predicate. If you need a literal discriminant, `as const`. If you need
to attach a property to `global`, prefer intersection; use `as unknown as` only when the types
refuse to overlap.

---

## 7. How the tree shows up in one file

`lib/sms.ts` is the whole diagram:

| Line                                    | Kind                             |
| --------------------------------------- | -------------------------------- |
| `name: string`                          | Primitive                        |
| `status: 1`                             | Primitive literal                |
| `SmsIrVerifyRequest`                    | Object type                      |
| `parameters: SmsIrVerifyParameter[]`    | Object (array)                   |
| `payload: unknown`                      | Top                              |
| `payload as Record<string, unknown>`    | Assertion to object-of-top       |
| `payload is SmsIrVerifySuccessResponse` | Predicate — narrows top → object |
| `templateId: number`                    | Primitive after runtime `Number` |

No `any`. No `never`. Throws instead of `: never`.

---

## 8. Mental model

1. **Primitive type** — JS primitive (or literal, or `null`/`undefined`). Copied by value at
   runtime; named at compile time.
2. **Object type** — properties / `Record` / arrays / `Date` / functions. `type` alias by default;
   `interface` only where already used.
3. **Top** — `unknown` until you narrow. Never `any` in app code.
4. **Bottom** — `never` to forbid a field on one union member. Not a substitute for `throw`.
5. **Assertion** — compile-time only. After `as`, ask whether a runtime check still exists.

---

## 9. Exercises on this repo

Predict; do not edit production code.

1. Change `generateOtpCode(): string` to `: number` but keep `String(...)`. What does `tsc` say?
   What runs?
2. `status: 1` → `status: number` in `SmsIrVerifySuccessResponse`. Does `record.status === 1` still
   narrow anything useful?
3. Add `CANCELLED` to `LeadStatus` in Prisma (thought experiment). Which object type breaks first —
   `STATUS_LABELS` or `ALL_STATUSES`?
4. Replace `error: unknown` with `error: any` in `getActionErrorMessage`. Which lines become legal
   that should not be?
5. `FinanceAttention` success with `{ data, error: "x" }` — why does `error?: never` reject it?
6. `formData.get("file") as File | null` when the field is a text string `"hello"`. What is the TS
   type? What is `typeof file` at runtime? What does `file.type` do?
7. Drop `as const` from `{ ok: true as const }`. Is `ok` `boolean` or `true`? Does `if (result.ok)`
   still discriminate `error`?
8. `global as { prisma: PrismaClient }` without `unknown` — what error?
9. Why is `MarketingLeadFieldErrors` `Partial<Record<…>>` an object type and not a primitive union?
10. `variant as never` on `Badge` — name a type-safe alternative using `satisfies` or a `Record`.

---

## Files cited

| File                                                     | Type-tree role                                                       |
| -------------------------------------------------------- | -------------------------------------------------------------------- |
| `lib/otp.ts`                                             | Primitive `: string`; inline object params; `as const` discriminants |
| `lib/sms.ts`                                             | Object aliases; literal `1`; `unknown`; `as Record`; predicates      |
| `lib/date.ts`                                            | Primitive union + `Date` object in `DateInput`                       |
| `lib/finance-range.ts`                                   | String literal union; object aliases                                 |
| `lib/lead-labels.ts`                                     | `Record`; `as const satisfies`                                       |
| `lib/legal/contract-template.ts`                         | `interface` object types                                             |
| `lib/push.ts`                                            | Optional object field; `ok: true as const`                           |
| `lib/action-errors.ts`                                   | `unknown` top; narrowing                                             |
| `lib/db.ts`                                              | `as unknown as`                                                      |
| `lib/validations/onboarding.ts`                          | `z.infer` → object type from Zod                                     |
| `lib/marketing-leads/schema.ts`                          | String literal union; `Partial<Record<>>`                            |
| `lib/marketing-leads/public-api.ts`                      | Intersection object type                                             |
| `features/audit/server/helpers.ts`                       | Generic default `unknown`                                            |
| `app/(dashboard)/finance/_data/get-finance-dashboard.ts` | `never` on exclusive union                                           |
| `app/(dashboard)/logs/_lib/activity-normalizer.ts`       | `unknown` metadata; `as AuditMetadata`                               |
| `app/api/upload/image/route.ts`                          | `as File \| null`                                                    |
| `app/(dashboard)/_lib/actions/membership-payments.ts`    | `as PaymentMethod`                                                   |
| `app/platform/_components/venues-table.tsx`              | `as never`                                                           |
| `components/pwa/InstallPrompt.tsx`                       | Intersection + `as` + `unknown` field                                |

Next TS: unions & narrowing (you already met them), generics, or `typeof` / `keyof`. Next JS:
**arrays** if you want to finish Data Types.
