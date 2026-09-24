# TypeScript vs JavaScript — Studivo Walkthrough

You already know JS: hoisting, scope, primitives. TypeScript is **that same language plus a type
layer that is erased before the code runs**.

Studivo is a TypeScript app (`strict: true` in `tsconfig.json`) that still ships **one real
JavaScript file** (`public/sw.js`) and still does **runtime checks** wherever data is untrusted.
That split is the whole topic.

Read this with the cited files open. Predict: _will this line exist after `next build`? Can SMS.ir
or FormData violate this type?_

---

## 1. One sentence

**JavaScript is what the browser and Node execute. TypeScript is JavaScript plus types that exist
only while you type, lint, and compile.**

If you stripped every type annotation, `import type`, `as const`, and `payload is Foo` from Studivo,
the **runtime behavior** of `reserveSeat`, OTP, and SMS would be the same — until a typo slipped
through. Types are a seatbelt, not the engine.

---

## 2. What Studivo actually compiles

```1:12:tsconfig.json
{
  "compilerOptions": {
    "target": "ES2017",
    "lib": [
      "dom",
      "dom.iterable",
      "esnext"
    ],
    "allowJs": true,
    "skipLibCheck": true,
    "strict": true,
    "noEmit": true,
```

Facts from this file:

| Setting                              | Meaning in this repo                                                                                                                                         |
| ------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `"strict": true`                     | `null` is not assignable to `string`. Implicit `any` is an error. This is why `toDate` returns `Date \| null` instead of pretending every value is a `Date`. |
| `"noEmit": true`                     | `tsc` does **not** write `.js` files. Next.js (and SWC/Babel) typecheck + bundle. You never run `tsc -p .` as the production compiler.                       |
| `"allowJs": true`                    | A `.js` file **may** sit next to `.ts`. The service worker does.                                                                                             |
| `"include"` is `**/*.ts`, `**/*.tsx` | Application code is TypeScript. `public/sw.js` is not in that program as a first-class typed module.                                                         |

`package.json` scripts are `next dev` / `next build` / `eslint`. There is no separate `tsc` script.
Types fail the **build**, not a hidden extra step.

After compile, this:

```ts
export function generateOtpCode(): string {
  return String(Math.floor(100000 + Math.random() * 900000));
}
```

is just:

```js
export function generateOtpCode() {
  return String(Math.floor(100000 + Math.random() * 900000));
}
```

The `: string` is gone. Node never sees it.

---

## 3. The one file that is still JavaScript

```1:12:public/sw.js
const CACHE_NAME = "studivo-v2";
const ASSETS_TO_CACHE = [
  "/web-app-manifest-192x192.png",
  "/web-app-manifest-512x512.png",
  "/notification-icon-192.png",
];

const isNextAssetRequest = (request) => new URL(request.url).pathname.startsWith("/_next/");
const isNavigationRequest = (request) => request.mode === "navigate";
```

Why this stays `.js`:

1. The browser fetches `/sw.js` as a **classic worker script**, not a Next.js module graph.
2. No TypeScript toolchain runs inside the service worker at install time.
3. `self`, `caches`, `clients` are worker globals — the same JS you already read in the hoisting
   note.

You _could_ write `sw.ts` and compile it. Studivo did not. So in this repo:

- **App, actions, Prisma, React** → TypeScript.
- **Service worker** → JavaScript.

That is the cleanest TS-vs-JS demo in the tree: same product, two languages, one of them has no
types at all.

If you pass the wrong thing to `isNavigationRequest` in `sw.js`, nothing complains until a phone is
offline. If you pass the wrong thing to `generateOtpCode()` in `lib/otp.ts`, `strict` plus the
editor complain **before** commit.

---

## 4. Types that vanish: `import type` and `export type`

```1:4:lib/marketing-leads/create-marketing-lead.ts
import "server-only";

import { prisma } from "@/lib/db";
import type { MarketingLeadInput } from "@/lib/marketing-leads/schema";
```

```6:8:app/actions/marketing/lead.ts
export type SubmitDemoResult =
  | { success: true; leadId: string }
  | { success: false; error: string };
```

```1:1:next.config.ts
import type { NextConfig } from "next";
```

`import type` / `export type` are **erased**. The bundler must not emit a runtime `import` for
`MarketingLeadInput`. That type is `z.output<typeof marketingLeadInputSchema>` — a shape, not a
function.

Compare a **value** import:

```ts
import { marketingLeadInputSchema } from "@/lib/marketing-leads/schema";
```

That **stays**. `safeParse` runs in production.

Rule of thumb in Studivo:

| Syntax                                 | Survives `next build`?                                      |
| -------------------------------------- | ----------------------------------------------------------- |
| `import type { Metadata } from "next"` | No                                                          |
| `export type ActionResult = { ... }`   | No                                                          |
| `import { prisma } from "@/lib/db"`    | Yes                                                         |
| `import { z } from "zod"`              | Yes                                                         |
| `: string`, `: number`, generics       | No                                                          |
| `as const` (see below)                 | The **value** stays; the literal-narrowing is a type effect |
| `"use server"`                         | Yes — Next.js directive, not a TS type                      |

If you wrote `import { MarketingLeadInput } from "..."` **without** `type`, and `isolatedModules` is
on, TS may still require `import type` because the symbol is types-only. Studivo already writes
`import type` explicitly. Copy that.

---

## 5. TypeScript does not validate SMS.ir or FormData

This is the sentence that saves careers:

> **A type annotation is a promise you made to the compiler. The network did not sign it.**

JSON from SMS.ir is `unknown` until checked:

```53:67:lib/sms.ts
function isSmsIrSuccessResponse(
  payload: unknown,
): payload is SmsIrVerifySuccessResponse {
  if (typeof payload !== "object" || payload === null) {
    return false;
  }

  const record = payload as Record<string, unknown>;

  return (
    record.status === 1 &&
    typeof record.data === "object" &&
    record.data !== null &&
    typeof (record.data as Record<string, unknown>).messageId === "number"
  );
}
```

Three layers:

1. **`unknown`** — TS forbids `payload.status` until you narrow. That is TS helping JS.
2. **`typeof` / `=== null`** — real JS from the primitives note. This runs in production.
3. **`payload is SmsIrVerifySuccessResponse`** — a **type predicate**. After
   `if (!isSmsIrSuccessResponse(payload)) throw`, the compiler treats `payload` as the success
   shape. The predicate **function** remains; the `is` clause is erased.

`as Record<string, unknown>` is an assertion: you telling TS “trust me.” It does **not** convert the
value. Abuse of `as` is how TS becomes lying JS.

FormData is the same hole:

```10:13:app/actions/marketing/lead.ts
function getFormDataString(formData: FormData, key: string) {
  const value = formData.get(key);
  return typeof value === "string" ? value : "";
}
```

`.get` is typed as `FormDataEntryValue | null` (`string | File | null`). TS already knows it might
not be a string. The `typeof` check is JS. Then Zod runs:

```22:32:app/actions/marketing/lead.ts
  const parsed = marketingLeadInputSchema.safeParse({
    fullName: getFormDataString(formData, "fullName") || getFormDataString(formData, "name"),
    // ...
  });
```

**Zod is JavaScript.** Schemas execute. Types like `MarketingLeadInput` are inferred _from_ the
schema (`z.output<typeof ...>`), not the other way around.

Onboarding makes that explicit:

```79:79:lib/validations/onboarding.ts
export type OnboardingValues = z.infer<typeof onboardingSchema>;
```

`z.infer<typeof onboardingSchema>` is a **TS** query on a **JS** runtime object. Two `typeof`s, two
worlds:

- `typeof onboardingSchema` in a **type position** → the TypeScript type of that value.
- `typeof payload !== "object"` in a **value position** → the JavaScript operator from the
  primitives note.

If you only added `function submitOnboarding(data: OnboardingValues)` and skipped
`onboardingSchema.safeParse(data)`, a crafted client could send `{ seatCount: -1 }`. Types would be
happy at the authoring site; the server would not have checked. Studivo parses. That is the house
style: **TS for internal certainty, Zod/`typeof` for the boundary.**

---

## 6. What TypeScript adds on top of the JS you already know

### 6.1 Annotations on bindings you already understood

```ts
function getTemplateId(environmentVariable: string): number {
```

JS: `function getTemplateId(environmentVariable) {`. TS: parameter must be a string; return must be
a number. `Number.isFinite` still needed because `Number("x")` is `NaN`, and `NaN` is a `number`.

### 6.2 Unions instead of “whatever”

```ts
type DateInput = Date | string | number | null | undefined;
```

JS would accept anything and hope `new Date(value)` works. TS forces `toDate` to handle `null`. That
is the primitives note, now **named**.

Discriminated unions (success vs failure) show up as `as const` on a tag:

```65:69:lib/otp.ts
  if (!record) {
    return { ok: false as const, error: "کد تایید نامعتبر یا منقضی شده است." };
  }

  return { ok: true as const, record };
```

`as const` makes `ok` the literal `false`, not a general `boolean`. Callers can write
`if (result.ok) { result.record }` and TS knows `error` is absent. At runtime this is still
`{ ok: false, error: "..." }` — ordinary JS object.

Push send does the same: `{ ok: true as const }` vs `{ ok: false as const, stale: true }`.

`SubmitDemoResult` is the type-only form of that pattern (no extra runtime tag beyond `success`).

### 6.3 `satisfies` — check without widening

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

JS array of strings. TS extra:

- `as const` → tuple of string **literals**, not `string[]`.
- `satisfies readonly LeadStatus[]` → every element must be a Prisma `LeadStatus`, but the variable
  keeps the literal tuple type.

If Prisma adds a status and you forget the label map, `Record<LeadStatus, string>` on
`STATUS_LABELS` fails the build. That failure is **the point of TS**. JS would ship a missing
Persian label and show `undefined` in the UI.

### 6.4 Structural types vs classes

```5:10:features/audit/server/helpers.ts
export type ActionResult<T = unknown> = {
  success: boolean;
  error?: string;
  message?: string;
  data?: T;
};
```

This is not a JS class. It is a **shape**. Any object with those fields is an `ActionResult`. Erased
at emit.

`export class UploadValidationError extends Error {}` in `lib/upload.ts` **is** JS.
`instanceof UploadValidationError` works at runtime. Prefer `type` for data bags, `class` when you
need `instanceof` (errors).

### 6.5 Generics are deleted

```ts
export function actionError<T = unknown>(error: unknown, fallback: string): ActionResult<T>;
```

`<T>` never runs. It only threads the `data` type through to callers. The function body is
untyped-looking JS: `instanceof`, `error.message`, object literals.

---

## 7. TypeScript `enum` vs Studivo’s choice

Prisma generates a TS enum for `OtpPurpose`. The app **also** writes a JS object:

```7:10:lib/otp.ts
export const OTP_PURPOSE = {
  PASSWORD_RESET: "PASSWORD_RESET" as OtpPurpose,
  SIGNUP: "SIGNUP" as OtpPurpose,
} as const;
```

At runtime you need a real object to pass into Prisma. A
`type OtpPurpose = "PASSWORD_RESET" | "SIGNUP"` alone cannot be used as a value.

Studivo pattern:

- **Union / Prisma enum** for the type.
- **`as const` object** when JS must iterate or pass a value.
- Avoid TypeScript `enum { A, B }` unless Prisma already emitted one — classic TS enums compile to
  extra JS. String unions + `as const` stay closer to JS.

---

## 8. Same syntax, two meanings

You will mix these constantly.

| Written                        | In a type                  | In a value                                              |
| ------------------------------ | -------------------------- | ------------------------------------------------------- |
| `typeof onboardingSchema`      | Type of that const         | —                                                       |
| `typeof payload !== "object"`  | —                          | JS operator, returns a string                           |
| `x as const`                   | Narrow to literal          | Value unchanged                                         |
| `x as Record<string, unknown>` | Trust me                   | Value unchanged — **dangerous** if unchecked            |
| `payload is Foo`               | Narrow after a true return | The function still returns a boolean                    |
| `A \| B`                       | Union type                 | Bitwise OR if used as a value (`status \| 0`)           |
| `<T>`                          | Generic                    | JSX in `.tsx` (`<Button />`)                            |
| `!origin`                      | —                          | JS truthiness                                           |
| `origin!` (non-null assertion) | Trust me, not null         | Value unchanged — Studivo prefers `if (!origin) return` |

If you are unsure, ask: **does this run on the server during a reservation?** If yes, it is JS. If
deleting it would not change `node` output, it is TS.

---

## 9. What TypeScript will _not_ do for you

These remain pure JS problems, even under `strict`:

1. **Hoisting / TDZ** — `const` before initialization still throws. TS may warn on
   use-before-define; the engine still has a TDZ.
2. **Scope** — two `startOfDay` functions in two files are two bindings. TS will not merge them.
3. **`typeof null === "object"`** — SMS guards still need `payload === null`.
4. **`NaN` is a number** — `getTemplateId` still uses `Number.isFinite`.
5. **Mutating objects** — types do not freeze Prisma results.
6. **Race conditions** — seat double-booking is a DB unique index plus `actionError` matching
   `P2002`, not a type.
7. **Wrong assertion** — `as SmsIrVerifySuccessResponse` without checks compiles and then blows up
   on `.data.messageId`.

AGENTS.md already tells you: **UI hiding is not security; types are not validation.** Same idea.

---

## 10. How a Studivo function is both

Take `verifyOtp`:

**TypeScript (author time):** parameters
`{ phoneNumber: string; code: string; purpose: OtpPurpose }`. Return is a discriminated union via
`ok: true | false`. Callers get autocomplete on `result.record` only after `result.ok`.

**JavaScript (run time):** `prisma.otpVerification.findFirst(...)`. If no row, return a plain
object. No type predicate function needed because the `ok` field is a real boolean-like literal the
caller’s `if` already understands — and `as const` made the compiler understand it too.

Take `createMarketingLead(input: MarketingLeadInput)`:

**TS:** you cannot call it with `{ foo: 1 }` from another `.ts` file.

**JS:** a server action must still `safeParse` FormData **before** calling it. The type on `input`
assumes the caller already validated. The public route and `submitLead` do that job.

---

## 11. Mental model

```text
        you type .ts / .tsx
                |
        TypeScript compiler (strict)
        catches: wrong args, missing null checks,
                 incomplete Record<LeadStatus, string>
                |
        types erased
                |
        JavaScript that Next.js runs
                |
        remaining checks: typeof, Zod, Prisma, if (!origin)
```

`public/sw.js` skips the top half.

When learning TS on this repo: read the **types** to see intent; read the **`typeof` / Zod /
`instanceof`** to see what is actually enforced.

---

## 12. Exercises

1. Delete `: string` from `generateOtpCode`. Does OTP still send? What did you lose?
2. Change `import type { MarketingLeadInput }` to a value import. Does the build care? Does the
   bundle include extra code?
3. In `sw.js`, add a type annotation. What happens at install in the browser?
4. Replace `isSmsIrSuccessResponse` with `payload as SmsIrVerifySuccessResponse` and skip the
   `typeof` checks. Does `tsc` pass? What happens when SMS.ir returns HTML?
5. Why is `OnboardingValues = z.infer<typeof onboardingSchema>` better than writing the interface by
   hand?
6. `ALL_STATUSES as const satisfies readonly LeadStatus[]` — remove `as const`. What happens to a
   `switch` that expected `"NEW"` not `string`?
7. Is `"use server"` TypeScript? What if you put it in a `.js` file?

---

## Files cited

| File                                           | Why it is here                                               |
| ---------------------------------------------- | ------------------------------------------------------------ |
| `tsconfig.json`                                | `strict`, `noEmit`, `allowJs`                                |
| `package.json`                                 | Next.js is the compiler driver                               |
| `public/sw.js`                                 | Real JavaScript in a TS app                                  |
| `lib/otp.ts`                                   | Erased return types; `as const` unions; value object vs type |
| `lib/marketing-leads/create-marketing-lead.ts` | `import type`                                                |
| `app/actions/marketing/lead.ts`                | `export type` + runtime `typeof` + Zod                       |
| `next.config.ts`                               | Typed config, `import type`                                  |
| `lib/sms.ts`                                   | Type predicate + JS `typeof`                                 |
| `lib/validations/onboarding.ts`                | `z.infer` — types from runtime schema                        |
| `lib/lead-labels.ts`                           | `as const satisfies`                                         |
| `features/audit/server/helpers.ts`             | Object type vs runtime `instanceof`                          |
| `lib/upload.ts`                                | `class` survives as JS                                       |
| `lib/date.ts`                                  | Union of primitives named in TS                              |

Next TypeScript topics can go file-by-file the same way: annotations, unions, narrowing, then back
to JS **objects** if you want to finish Data Types first.
