# TypeScript and JavaScript Interoperability — Studivo Walkthrough

The earlier note was **TS vs JS**: types erase, `sw.js` stays JS. This note is **how they live in
one repo**: importing a JS package from TS, typing `global`, parsing JSON, and why `allowJs` does
not mean the service worker is typechecked.

Interop is not a third language. It is the seams: **TS files talking to JS values the compiler did
not author.**

Read this with the cited files open. Predict: _who invented this type — our `.ts`, `@types/*`,
Prisma generate, or a runtime check?_

---

## 1. The rule

At runtime there is only JavaScript. Interop is how TypeScript **describes** JS that already exists:

| Seam                                              | In Studivo                                                           |
| ------------------------------------------------- | -------------------------------------------------------------------- |
| App TS → app TS                                   | Normal `import`. Types travel with values.                           |
| App TS → npm package written in JS                | `esModuleInterop` + `@types/web-push` (or the package’s own `.d.ts`) |
| App TS → generated JS/TS                          | Prisma client in `lib/generated/prisma/client`                       |
| App TS → host JS (`global`, `window`, `FormData`) | `lib: ["dom", …]` + assertions / `unknown`                           |
| App TS → JSON / SMS.ir / FormData                 | **No types on the wire.** Zod / `typeof` / predicates                |
| Browser → `public/sw.js`                          | Plain JS. Not in the TS `include` graph                              |

If the other side is untyped, TS will not invent safety. You add types, or you check at runtime.

---

## 2. Compiler flags that _are_ the interop policy

```1:17:tsconfig.json
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
    "esModuleInterop": true,
    "module": "esnext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
```

```31:42:tsconfig.json
  "include": [
    "next-env.d.ts",
    "**/*.ts",
    "**/*.tsx",
    ".next/types/**/*.ts",
    ".next/dev/types/**/*.ts",
    "**/*.mts"
  ],
  "exclude": [
    "node_modules",
    "scripts/migrate-v2.ts"
  ]
```

| Flag                             | Interop meaning here                                                                                                  |
| -------------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| `"strict": true`                 | JS values must be narrowed. `null` is not `string`.                                                                   |
| `"allowJs": true`                | TS _may_ import `.js` and type-check it loosely if it were in the program.                                            |
| `"include"` is `**/*.ts(x)` only | **`public/sw.js` is not in the program.** `allowJs` does not typecheck the worker.                                    |
| `"esModuleInterop": true`        | `import webpush from "web-push"` works even if the package is CommonJS (`module.exports`).                            |
| `"moduleResolution": "bundler"`  | Next/SWC resolves packages; matches how the app actually loads JS.                                                    |
| `"resolveJsonModule": true`      | You _could_ `import x from "./foo.json"` with a type. Studivo mostly `JSON.parse`s strings instead.                   |
| `"isolatedModules": true`        | Every file must be transpilable alone. Forces `import type` for type-only names (TS vs JS note).                      |
| `"skipLibCheck": true`           | Do **not** typecheck `.d.ts` inside `node_modules`. Faster builds; a bad `@types` package will not fail `next build`. |
| `"noEmit": true`                 | `tsc` does not write JS. Next emits JS from TS. The interop output is the bundle, not `tsc --outDir`.                 |
| `"lib": ["dom", "esnext"]`       | Built-ins (`Date`, `Map`, `FormData`) get TS descriptions of **engine JS**.                                           |

`eslint.config.mjs` is ESM JavaScript consumed by ESLint, not by `tsc`. Config files often stay
JS/MJS on purpose.

---

## 3. JS in the repo that TS does not see

```1:9:public/sw.js
const CACHE_NAME = "studivo-v2";
const ASSETS_TO_CACHE = [
  "/web-app-manifest-192x192.png",
  "/web-app-manifest-512x512.png",
  "/notification-icon-192.png",
];

const isNextAssetRequest = (request) => new URL(request.url).pathname.startsWith("/_next/");
```

Hooks like `usePWA` register this file by **URL**, not by `import "../public/sw.js"`. No types cross
that boundary. The worker talks JSON from `push` events — same as SMS.ir: **shape by convention, not
by compiler.**

If you `import sw from "@/public/sw.js"` inside a `.ts` file, `allowJs` would start mattering.
Studivo does not. Interop pattern: **same origin, different language, no import.**

---

## 4. Importing a JavaScript npm package

`web-push` is published as JS. Types are a **separate** DefinitelyTyped package:

```54:61:package.json
  "devDependencies": {
    "@tailwindcss/postcss": "^4",
    "@types/node": "^20",
    "@types/react": "^19",
    "@types/react-dom": "^19",
    "@types/three": "^0.185.1",
    "@types/web-push": "^3.6.4",
```

```1:1:lib/push.ts
import webpush, { type PushSubscription as WebPushSubscription } from "web-push";
```

Three interop tricks in one line:

1. **Default import** of a CJS module — needs `esModuleInterop` (synthesizes `module.exports` as
   `default`).
2. **`import { type PushSubscription }`** — type-only; erased. The runtime `webpush` object stays.
3. **`as WebPushSubscription` rename** — the library’s type name would clash with the DOM
   `PushSubscription`. TS name vs JS global of the same word.

`@types/node` types `process.env`, `Buffer`, `URL`. Without it, `process.env.SMS_IR_API_KEY` would
be an error (or `any`, depending on config). **Node is JS; `@types/node` is the TS view.**

`@types/react` types JSX and hooks. React itself is JS. Your `.tsx` is TS describing that JS.

`zod` ships its **own** types in the package (`zod` is TS-first). No `@types/zod`. Rule: prefer
packages with bundled `.d.ts`; use `@types/*` when the author shipped JS only.

`skipLibCheck` means if `@types/web-push` disagrees with `@types/node`, the build may still pass.
Your app code is still `strict`.

---

## 5. Generated TS sitting next to handwritten TS

```1:3:lib/otp.ts
import { prisma } from "@/lib/db";
import { sendVerificationCode } from "@/lib/sms";
import type { OtpPurpose } from "@/lib/generated/prisma/client";
```

Prisma **generates** a client (TS/JS) from `schema.prisma`. That is interop with a **code
generator**, not with a human JS file.

- `import type { OtpPurpose }` — enum/union type, erased.
- `import { prisma }` — value: a JS class instance.
- `export const OTP_PURPOSE = { … } as const` — your JS object **branded** with the generated type
  so the two worlds agree.

If generate is stale, TS fails on `.studyHall` / `SeatAssignment` even though production JS might
still query old SQL. Interop here is **schema → types → queries**. Keep `prisma generate` in the
pipeline.

`PrismaClient` in `lib/db.ts` is that generated class. `new PrismaClient({ adapter })` is JS
construction with a TS constructor signature.

---

## 6. Host objects: `global` is JS, the annotation is a lie you document

```4:6:lib/db.ts
const globalForPrisma = global as unknown as {
  prisma: PrismaClient;
};
```

`global` (Node) / `globalThis` is a JS object. `@types/node` does **not** declare `.prisma`. Strict
TS rejects `global.prisma = prisma`.

The escape hatch is a **double assertion**:

1. `as unknown` — throw away the Node type (any value can be `unknown`).
2. `as { prisma: PrismaClient }` — claim a shape.

Nothing at runtime changes. Hot reload still mutates the real global object (objects note). TS just
stops arguing.

This is the honest interop story for **expanding a JS global**. Prefer a typed wrapper over
sprinkling `as any`. `unknown` in the middle makes the lie visible.

`process.env.DATABASE_URL!` is another host lie: Node types say `string | undefined`. `!` asserts “I
deployed it.” Production `getUploadDir` **checks** instead of `!`. Interop quality = assertion vs
runtime check.

---

## 7. Data that was never TypeScript: JSON, FormData, `unknown`

SMS.ir returns JSON. `response.json()` is typed as `any` or `unknown` depending on DOM lib. Studivo
treats it as `unknown` and **proves** a shape:

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

Interop layers:

| Layer                                   | Language                                |
| --------------------------------------- | --------------------------------------- |
| HTTP body                               | JS string                               |
| `JSON.parse` (inside `.json()`)         | JS object, no prototype of yours        |
| `unknown`                               | TS: “do not touch yet”                  |
| `as Record<string, unknown>`            | TS: indexable bag, values still unknown |
| `typeof … === "number"`                 | JS check                                |
| `payload is SmsIrVerifySuccessResponse` | TS predicate; the function remains JS   |

**Do not** start at `as SmsIrVerifySuccessResponse`. That is fake interop: you told TS the JS was
already your type.

FormData is a host JS API. `.get` is `FormDataEntryValue | null` (`string | File`).

```20:20:app/api/upload/image/route.ts
  const file = formData.get("file") as File | null;
```

Assertion: “this field is a File.” A crafted request can send a string. The later `saveUploadedFile`
still reads `.type` / `.size`. **TS interop without a runtime `instanceof File` is optimistic.**
Contrast lead capture, which uses `instanceof HTMLInputElement`.

Gallery images:

```151:153:app/(dashboard)/settings/actions.ts
    galleryImages = JSON.parse(
      formData.get("galleryImages")?.toString() || "[]",
    );
```

`JSON.parse` returns `any` in default TS DOM types. Assigning to `string[]` is interop by **hope**,
then Zod/`safeParse` on the rest of the form. A better seam is
`z.array(z.string()).parse(JSON.parse(...))` — JS schema, inferred TS. Onboarding already does that
for the whole wizard:

```79:79:lib/validations/onboarding.ts
export type OnboardingValues = z.infer<typeof onboardingSchema>;
```

**Best Studivo interop:** one JS Zod object owns the boundary; TS infers. Worst: `as File` /
`as string[]` with no parse.

---

## 8. `import type` vs value from the same JS module

```1:1:lib/push.ts
import webpush, { type PushSubscription as WebPushSubscription } from "web-push";
```

```3:5:lib/otp.ts
import type { OtpPurpose } from "@/lib/generated/prisma/client";

export { OtpPurpose };
```

`export { OtpPurpose }` re-exports a **type** (and possibly a runtime enum, depending on Prisma
emit). If Prisma emits a real JS enum object, that export is a value. If it is `type` only, bundlers
must not keep a require. `isolatedModules` + `import type` keep this safe.

From a JS package you can:

- import a **function** (`webpush.sendNotification`) — runtime JS, types from `.d.ts`;
- import a **type** (`PushSubscription`) — `.d.ts` only.

Mixing them in one statement is normal interop, not a special syntax island.

---

## 9. CommonJS vs ESM (what `esModuleInterop` fixes)

Old JS:

```js
module.exports = webpush;
```

TS without interop wanted `import * as webpush from "web-push"` or
`import webpush = require("web-push")`.

With `"esModuleInterop": true` and `"moduleResolution": "bundler"`:

```ts
import webpush from "web-push";
```

Next’s bundler emits a compatible default. You do not write `require` in app TS (grep is clean).
**App code is ESM-shaped TS.** Some dependencies remain CJS. That mismatch is the classic interop
bug (`undefined` default import). If a new package’s default import is `undefined` at runtime but
types look fine, this is the seam — check CJS vs ESM, not your types.

`eslint.config.mjs` uses `export default` — native ESM JS, outside `tsc`.

---

## 10. Declaration files you did not write

When you hover `useState` in a `.tsx` file, you read **`.d.ts`**, not React’s `.js`. Interop
pipeline:

```text
react.js  (runtime)
   ↑
@types/react/index.d.ts  (TS)
   ↑
hooks/usePWA.ts  (your TS, compiles to JS)
```

`next-env.d.ts` (included) references Next’s types so `Image`, `Metadata`, route `params` exist.
Generated `.next/types/**/*.ts` is Next describing **your** `app/` files back to `tsc`. Framework JS
↔ your TS, both directions.

You do not hand-write `declare module "web-push"` because `@types/web-push` already did. You _would_
`declare module "*.svg"` if you imported assets; Studivo mostly does not.

---

## 11. What “the types lie” looks like in this codebase

| Lie                                    | Where           | Safer interop                      |
| -------------------------------------- | --------------- | ---------------------------------- |
| `global as unknown as { prisma }`      | `lib/db.ts`     | Acceptable; isolated, documented   |
| `formData.get("file") as File \| null` | upload route    | `instanceof File`                  |
| `JSON.parse(...)` as `string[]`        | settings action | Zod array                          |
| `process.env.X!`                       | db adapter      | check like `getSmsApiKey`          |
| `payload as Record<string, unknown>`   | sms.ts          | OK **after** `typeof === "object"` |
| SMS success `as` without predicate     | (avoided)       | `payload is Foo`                   |

TS/JS interop is **where you are allowed to lie**, and whether a JS check backs the lie.

---

## 12. Mental model

```text
          TypeScript (.ts/.tsx)
                    |
     descriptions of JS:
     @types/*, generated Prisma, lib/dom
                    |
     your runtime checks:
     Zod, typeof, instanceof, predicates
                    |
          JavaScript that runs
          (bundle + public/sw.js + node_modules)
```

When a value **crosses** from the bottom up, types start at `unknown` unless a `.d.ts` already
describes that package.

When a value **stays** inside app TS (`clampReminderDaysBefore(n: number)`), interop is not
involved.

---

## 13. Exercises on this repo

1. Remove `@types/web-push`. What fails: `tsc`/editor, or `sendPushToSubscription` at runtime?
2. Move `public/sw.js` into `include` via `allowJs` and add `// @ts-check`. Which `self` type shows
   up?
3. Replace `as File | null` with `instanceof File`. Which request bodies now 400?
4. `JSON.parse` of gallery images: if the string is `"{}"`, does TS complain? Does Prisma?
5. Why `as unknown as` instead of `as { prisma: PrismaClient }` directly on `global`? (Hint: overlap
   of Node’s `Global`.)
6. Is `z.infer<typeof onboardingSchema>` interop with JS or only a TS convenience?
7. `import webpush from "web-push"` with `esModuleInterop: false` — what runtime symptom?
8. Prisma `OtpPurpose`: if generate emits a const object, is `export { OtpPurpose }` a type-only
   re-export?

---

## Files cited

| File                                  | Interop seam                                       |
| ------------------------------------- | -------------------------------------------------- |
| `tsconfig.json`                       | `allowJs`, `esModuleInterop`, `include` vs `sw.js` |
| `package.json`                        | `@types/*` vs packages with bundled types          |
| `public/sw.js`                        | JS not in the TS program                           |
| `lib/push.ts`                         | CJS default import + `import type` rename          |
| `lib/db.ts`                           | `global as unknown as`; generated `PrismaClient`   |
| `lib/otp.ts`                          | generated enum type + value object                 |
| `lib/sms.ts`                          | `unknown` JSON + predicate                         |
| `lib/validations/onboarding.ts`       | `z.infer` from a JS schema                         |
| `app/api/upload/image/route.ts`       | `FormData.get` assertion                           |
| `app/(dashboard)/settings/actions.ts` | `JSON.parse` into `string[]`                       |
| `eslint.config.mjs`                   | ESM JS outside `tsc`                               |

Related: `docs/typescript-vs-javascript-in-studivo.md` (erasure),
`docs/javascript-built-in-objects-in-studivo.md` (`JSON` / `FormData`),
`docs/javascript-primitive-types-in-studivo.md` (`typeof`).

Next TS topics: narrowing, unions, or `any` vs `unknown`. Next JS: **Array** if you want to finish
Data Types.
