# TypeScript Installation and Configuration — Studivo Walkthrough

The earlier TS notes were **what types are** (erasure) and **how TS talks to JS** (interop). This
note is **how TypeScript got into this repo and which knobs are on**.

You do not install TypeScript into the browser. You install a **dev-time compiler and a
`tsconfig.json` contract**. Next.js then emits JavaScript. The `typescript` package never ships to
operators on `app.studivo.ir`.

Read this with `package.json` and `tsconfig.json` open. Predict: _if I deleted this package or
flipped this flag, what breaks — `bun dev`, `next build`, the editor, or runtime?_

Related: `docs/typescript-vs-javascript-in-studivo.md`,
`docs/typescript-js-interoperability-in-studivo.md`.

---

## 1. The rule

**Install TypeScript as a project devDependency. Configure it in `tsconfig.json`. Let the app’s
bundler emit JS.**

Textbook `npm i -g typescript` puts `tsc` on your laptop. Studivo does not use a global compiler.
The version that matters is the one in **this** `package.json`, resolved by **this** lockfile,
driven by **Next**, not by you typing `tsc`.

---

## 2. What “installation” actually is here

README local setup:

```37:41:README.md
bun install
bunx prisma generate
bun dev
```

`bun install` reads `package.json` and fills `node_modules`. That is the install step. There is no
`npm install typescript` in day-to-day work because it is already listed.

```54:66:package.json
  "devDependencies": {
    "@tailwindcss/postcss": "^4",
    "@types/node": "^20",
    "@types/react": "^19",
    "@types/react-dom": "^19",
    "@types/three": "^0.185.1",
    "@types/web-push": "^3.6.4",
    "eslint": "^9",
    "eslint-config-next": "16.2.6",
    "prisma": "^7.9.0",
    "tailwindcss": "^4",
    "typescript": "^5"
  }
```

| Package                             | Role in the TS install                                                                                                |
| ----------------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| `typescript`                        | The language service + `tsc`. Editor, ESLint, `next build` typecheck. **Not** a runtime import.                       |
| `@types/node`                       | Descriptions of Node (`process`, `crypto`). Needed because `scripts/reset-pass.ts` and `lib/db.ts` run on the server. |
| `@types/react` / `@types/react-dom` | JSX, `useState`, DOM event types. Matches `react` / `react-dom` **19**.                                               |
| `@types/web-push`                   | `web-push` is JS. Types are a separate DefinitelyTyped package (interop note).                                        |
| `@types/three`                      | Same idea for `three`.                                                                                                |
| `eslint-config-next`                | Pulls `eslint-config-next/typescript` — a **second** consumer of the same `tsconfig`.                                 |
| `next` (in `dependencies`)          | The **emitter**. SWC compiles TS → JS. `typescript` typechecks; Next bundles.                                         |

`zod`, `react`, `next` are `dependencies` because the **running app** imports them. `typescript` is
`devDependencies` because production Node never `require("typescript")`.

Scripts:

```5:11:package.json
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "eslint",
    "reset-pass": "bun scripts/reset-pass.ts"
  },
```

There is **no** `"typecheck": "tsc --noEmit"` script. `next build` typechecks.
`bun scripts/reset-pass.ts` runs a `.ts` file through Bun’s loader — still not global `tsc`.

Package manager is **Bun** (`bun install`, `bunx prisma generate`). Lockfiles for pnpm/bun are
gitignored; do not assume npm.

---

## 3. What you would type in an empty folder (mapped to Studivo)

If this were a greenfield app, the same result looks like:

```bash
bun init
bun add -d typescript @types/node @types/react @types/react-dom
bun add next react react-dom
# Next then writes tsconfig.json when you first run next dev / create-next-app
```

Studivo already has that graph. `create-next-app` (or an equivalent) produced `tsconfig.json` with
the Next plugin. You do **not** run `tsc --init` on top; that would fight Next’s `noEmit` / `jsx` /
`plugins`.

`tsc --init` defaults that Studivo **rejects**:

| `tsc --init` tendency    | Studivo                           |
| ------------------------ | --------------------------------- |
| `"strict"` sometimes off | `"strict": true`                  |
| `"module": "commonjs"`   | `"module": "esnext"`              |
| emit `.js` next to `.ts` | `"noEmit": true`                  |
| no path aliases          | `"@/*"` → `./*`                   |
| no Next plugin           | `"plugins": [{ "name": "next" }]` |

---

## 4. `tsconfig.json` is the configuration

One file. No `tsconfig.build.json`, no `tsconfig.app.json`. App, scripts, and generated Next types
share it.

```1:43:tsconfig.json
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
    "jsx": "react-jsx",
    "incremental": true,
    "plugins": [
      {
        "name": "next"
      }
    ],
    "paths": {
      "@/*": [
        "./*"
      ]
    }
  },
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
}
```

### 4.1 Language level — `target` and `lib`

| Flag     | Studivo choice                  | Meaning                                                                                                                                       |
| -------- | ------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| `target` | `ES2017`                        | Syntax `tsc` _would_ downlevel to, if it emitted. `async`/`await` stay. Next/SWC may emit newer JS for modern browsers anyway.                |
| `lib`    | `dom`, `dom.iterable`, `esnext` | **Type** libraries, not polyfills. `lib` does not add `fetch` to old browsers. It tells the checker that `Document`, `FormData`, `Map` exist. |

Server files (`lib/sms.ts`, `lib/db.ts`) still see `dom` types because there is **one** tsconfig.
That is normal for Next. Do not invent a second config just to hide `window` on the server —
`typeof window === "undefined"` stays a runtime check (`hooks/usePWA.ts`).

### 4.2 Strictness — the product rule

`"strict": true` turns on the whole bundle: `strictNullChecks`, `noImplicitAny`,
`strictFunctionTypes`, and the rest.

This is why `toDate` returns `Date | null`, why `process.env.DATABASE_URL` is `string | undefined`,
and why `payload: unknown` appears at JSON boundaries. Installation of TypeScript **without**
`strict` would still compile; Studivo would not catch `null` on a seat id.

There is no `"strict": false` override per folder.

### 4.3 Emit — Next owns JS

`"noEmit": true` — `tsc` writes **nothing**. No `dist/`. No `.js` beside `lib/otp.ts`.

`"jsx": "react-jsx"` — TS understands `<Button />` as `jsxDEV` runtime, not `React.createElement`
requiring `import React`. Matches React 19.

`"isolatedModules": true` — every file must be transpilable **alone** (SWC/Babel). That is why
`import type { NextConfig }` in `next.config.ts` exists: a type-only name cannot be a fake runtime
import.

`"incremental": true` — `tsc` (when Next invokes it) caches in `*.tsbuildinfo`. Gitignored:

```44:46:.gitignore
# typescript
*.tsbuildinfo
next-env.d.ts
```

### 4.4 Module resolution — how `import` finds files

| Flag                | Studivo   | If you changed it                                                                                        |
| ------------------- | --------- | -------------------------------------------------------------------------------------------------------- |
| `module`            | `esnext`  | `import`/`export` stay ESM. Matches `"type"` not being `"commonjs"`.                                     |
| `moduleResolution`  | `bundler` | Follow how Next/SWC resolve packages, not old Node `require` walks.                                      |
| `esModuleInterop`   | `true`    | `import webpush from "web-push"` works for CJS `web-push`.                                               |
| `resolveJsonModule` | `true`    | JSON imports _allowed_. Studivo still mostly `JSON.parse`s FormData strings.                             |
| `allowJs`           | `true`    | JS _may_ be part of the program. `include` still skips `public/sw.js`.                                   |
| `skipLibCheck`      | `true`    | Do not typecheck `node_modules/**/*.d.ts`. Faster CI. A broken `@types` package will not fail the build. |

### 4.5 Paths — `@/` is configuration, not magic

```25:29:tsconfig.json
    "paths": {
      "@/*": [
        "./*"
      ]
    }
```

```1:3:lib/db.ts
import { PrismaClient } from "@/lib/generated/prisma/client";
import { PrismaPg } from "@prisma/adapter-pg";
```

`@/lib/db` → `/workspace/lib/db.ts`. Next’s bundler honors the same map. shadcn repeats it for
codegen:

```15:21:components.json
  "aliases": {
    "components": "@/components",
    "utils": "@/lib/utils",
    "ui": "@/components/ui",
    "lib": "@/lib",
    "hooks": "@/hooks"
  },
```

Those aliases are **not** a second TypeScript config. They tell `shadcn` what to write in new files.
If `tsconfig` paths and `components.json` drifted, generated UI would not typecheck.

### 4.6 Next plugin and generated types

```20:24:tsconfig.json
    "plugins": [
      {
        "name": "next"
      }
    ],
```

The `next` plugin teaches the editor about App Router (`page.tsx` props, `metadata`). After
`next dev` / `next build`, Next writes:

- `next-env.d.ts` — triple-slash refs to Next’s types. **Gitignored**; regenerated.
- `.next/types/**/*.ts` — route types. Listed in `include`.

`include` names `next-env.d.ts` even though git ignores it. Clone + `bun dev` recreates it. Do not
hand-write that file.

### 4.7 `include` / `exclude` — what is in the program

**In:** `**/*.ts`, `**/*.tsx`, `**/*.mts`, Next generated types.

**Out:** `node_modules` (always), `scripts/migrate-v2.ts` (one-off migration kept off the checker).

**Not in `include` even with `allowJs`:** `public/sw.js`, `eslint.config.mjs`, `postcss.config.mjs`.
Config as ESM JS is a valid choice. `next.config.ts` and `prisma.config.ts` **are** in the program —
they are TypeScript.

Prisma **output** is generated TS/JS under `lib/generated/prisma`, gitignored, produced by
`bunx prisma generate`. Until that runs, `@/lib/generated/prisma/client` is a broken import.
Generate is part of **install**, not of `tsc`.

---

## 5. Other config files that _are_ TypeScript

### 5.1 Next config — typed, type-only import

```1:4:next.config.ts
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  allowedDevOrigins: ["studivo.ir"],
```

`import type` + `NextConfig` annotation. After compile this file is still consumed as config; the
type is gone. This is configuration **written in** TS, not `tsconfig` itself.

### 5.2 Prisma config — TS, not in `compilerOptions`

```1:11:prisma.config.ts
import "dotenv/config";
import { defineConfig } from "prisma/config";

export default defineConfig({
  schema: "prisma/schema.prisma",
  migrations: {
    path: "prisma/migrations",
  },
  datasource: {
    url: process.env["DATABASE_URL"],
  },
});
```

`defineConfig` types the object. `process.env["DATABASE_URL"]` is still `string | undefined` under
`strict`. Prisma’s own CLI reads this file; `tsconfig` only typechecks it because it is `*.ts`.

### 5.3 ESLint — JS config, TS plugin

```1:16:eslint.config.mjs
import { defineConfig, globalIgnores } from "eslint/config";
import nextVitals from "eslint-config-next/core-web-vitals";
import nextTs from "eslint-config-next/typescript";

const eslintConfig = defineConfig([
  ...nextVitals,
  ...nextTs,
  // Override default ignores of eslint-config-next.
  globalIgnores([
    // Default ignores of eslint-config-next:
    ".next/**",
    "out/**",
    "build/**",
    "next-env.d.ts",
  ]),
]);
```

`bun lint` → `eslint` → `eslint-config-next/typescript` uses the **same** `tsconfig.json`.
Installing `typescript` without this plugin would still typecheck on `next build`, but the editor’s
red squiggles from ESLint would miss type-aware rules.

`next-env.d.ts` is ignored by ESLint (generated noise) and still `include`d by `tsc`.

---

## 6. What is _not_ configured (on purpose)

| Missing textbook file                     | Why Studivo skips it                                        |
| ----------------------------------------- | ----------------------------------------------------------- |
| `jsconfig.json`                           | That is for JS projects. This app is TS.                    |
| `tsconfig.node.json` / project references | One Next app, one config.                                   |
| `"types": ["node"]` only                  | `lib` + `@types/*` in `node_modules` are auto-included.     |
| `typeRoots` override                      | Default `@types` lookup is enough.                          |
| `outDir` / `rootDir`                      | `noEmit`.                                                   |
| `"typecheck"` npm script                  | `next build` is the gate.                                   |
| Global `tsc`                              | Version would drift from `typescript` `^5` in the lockfile. |

`postcss.config.mjs` stays JS: it does not need types. That is configuration of **CSS**, not of
TypeScript.

---

## 7. Install vs generate vs typecheck vs emit

Four different jobs people mash together as “setting up TS”:

```text
bun install              → node_modules (typescript + @types + next)
bunx prisma generate     → lib/generated/prisma (client types)
next dev / next build    → typecheck (tsc API) + emit JS (SWC) + next-env.d.ts
eslint                   → extra type-aware lint, same tsconfig
```

If `bun install` was skipped, there is no `typescript` binary. If `prisma generate` was skipped,
`tsconfig` is fine but `lib/db.ts` cannot resolve `PrismaClient`. If you run `tsc` with a different
config, you are not testing Studivo.

---

## 8. Mental model

1. **`typescript` is a devDependency.** Production runs JS that Next emitted.
2. **`@types/*` is also installation.** They are `.d.ts` files, not runtime.
3. **`tsconfig.json` is the contract.** `strict`, `noEmit`, `paths`, `include`.
4. **Next is the driver.** Plugin, `next-env.d.ts`, `noEmit`.
5. **Generate Prisma before the checker can see the database client.**
6. **One program.** `allowJs` does not mean `sw.js` is checked; `include` decides.

---

## 9. Exercises on this repo

Predict; do not edit production config unless you are practicing on a throwaway clone.

1. Move `typescript` from `devDependencies` to `dependencies`. Does `next start` behave differently?
   What about image size?
2. Delete `@types/node`. Which files go red first — `scripts/reset-pass.ts` (`node:crypto`) or a
   client component?
3. Set `"strict": false`. What does `toDate`’s `Date | null` stop catching?
4. Set `"noEmit": false` and run `npx tsc`. Where do `.js` files appear, and would Next still use
   them?
5. Remove `"@/*"` from `paths`. Does `eslint` fail, `next build`, or both?
6. Why is `next-env.d.ts` gitignored but listed in `include`?
7. `allowJs: true` and `public/sw.js` — is the worker typechecked? (Hint: `include`.)
8. `bunx prisma generate` never run on a fresh clone: is that a `tsconfig` bug or an install-step
   miss?
9. `import type { NextConfig }` vs `import { NextConfig }` with `isolatedModules: true`.
10. Could `scripts/reset-pass.ts` use a second `tsconfig` with `"module": "commonjs"`? What would
    break `@/` imports?

---

## Files cited

| File                    | Install / config role                                         |
| ----------------------- | ------------------------------------------------------------- |
| `package.json`          | `typescript` + `@types/*` as devDependencies; no `tsc` script |
| `README.md`             | `bun install` then `prisma generate` then `bun dev`           |
| `tsconfig.json`         | The whole compiler contract                                   |
| `.gitignore`            | `*.tsbuildinfo`, `next-env.d.ts`, generated Prisma            |
| `next.config.ts`        | App config written in TS                                      |
| `prisma.config.ts`      | Prisma CLI config written in TS                               |
| `prisma/schema.prisma`  | `output = "../lib/generated/prisma"` — generate step          |
| `eslint.config.mjs`     | `eslint-config-next/typescript`                               |
| `components.json`       | shadcn aliases must match `paths`                             |
| `lib/db.ts`             | `@/` path + generated client import                           |
| `lib/push.ts`           | `@types/web-push` after installing the JS package             |
| `scripts/reset-pass.ts` | Bun runs `.ts` using `@types/node`                            |

Related: TS vs JS (erasure, `noEmit`), TS/JS interop (`allowJs`, `esModuleInterop`,
`@types/web-push`).

Next TS: annotations, unions/narrowing, or `any` vs `unknown`. Next JS: **arrays** if you want to
finish Data Types.
