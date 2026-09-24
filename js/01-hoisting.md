# JavaScript Hoisting — Studivo Walkthrough

This note uses real Studivo code to explain **hoisting**: JavaScript moving some declarations to the
top of their scope _before_ that scope runs.

Studivo is TypeScript + Next.js. That is useful, because the repo already shows the modern rules you
will live with:

- there is **no `var`** in application code;
- helpers are usually **`function` declarations** or **`const` arrows**;
- modules start with **`import`**, which is itself a hoisting story.

Read this file with the cited sources open. The goal is not to memorize jargon. The goal is to
predict what runs, and what throws.

---

## 1. What hoisting actually does

Before a function or module body runs, the engine does a pass over that scope and:

1. records every **declaration** (`function`, `var`, `let`, `const`, `class`, `import`);
2. **initializes** some of them immediately;
3. leaves others in the **Temporal Dead Zone (TDZ)** until the line that assigns them executes.

| Declaration               | Hoisted?                | Usable before its line?        |
| ------------------------- | ----------------------- | ------------------------------ |
| `function foo() {}`       | Yes, fully              | Yes                            |
| `var x = 1`               | Name only (`undefined`) | Yes, but value is `undefined`  |
| `let` / `const`           | Name only               | **No** — TDZ, `ReferenceError` |
| `class Foo {}`            | Name only               | **No** — TDZ                   |
| `import { x } from "..."` | Yes, fully              | Yes, anywhere in the module    |
| `const foo = () => {}`    | Name only               | **No** — TDZ                   |

Studivo never uses `var`, so you will not see the classic `undefined` surprise. You _will_ see the
difference between `function` and `const`.

---

## 2. Function declarations are fully hoisted

A `function` declaration creates the function **before any line in that scope runs**. Callers can
sit above or below the definition. The module still works.

### 2.1 Helpers declared under the export that uses them

`usePWA` calls `urlBase64ToUint8Array` inside a callback. In the source file, the helper is
**below** the hook.

```16:91:hooks/usePWA.ts
export function usePWA(options?: UsePWAOptions) {
  // ...
  const subscribeToPushNotifications = useCallback(async (vapidKey: string) => {
    try {
      const registration = await navigator.serviceWorker.ready;

      const subscription = await registration.pushManager.subscribe({
        userVisibleOnly: true,
        applicationServerKey: urlBase64ToUint8Array(vapidKey),
      });
      // ...
    }
  }, []);
  // ...
}

// Helper function to convert VAPID key
function urlBase64ToUint8Array(base64String: string): BufferSource {
  const padding = '='.repeat((4 - (base64String.length % 4)) % 4);
  // ...
}
```

Why this is legal:

1. The whole module is parsed.
2. Both `usePWA` and `urlBase64ToUint8Array` are function declarations, so both exist immediately.
3. The call happens later, when a component actually subscribes to push.

The same “define helpers at the bottom” pattern appears in settings UI. `SettingsTabs` is exported
first. `Field` is declared after the export, then used from JSX inside `SettingsTabs`.

```83:108:app/(dashboard)/settings/_components/settings-tabs.tsx
export function SettingsTabs({
  hall,
  sections,
  unassignedSeats,
  plans,
  initialTab = "general",
}: {
  // ...
}) {
  const router = useRouter();
  const refresh = () => router.refresh();
  // ...
  async function saveGeneral(formData: FormData) {
    formData.set("heroImage", heroImage ?? "");
    formData.set("galleryImages", JSON.stringify(galleryImages));
    return updateStudyHallSettings(formData);
  }
```

```359:372:app/(dashboard)/settings/_components/settings-tabs.tsx
function Field({
  label,
  children,
}: {
  label: string;
  children: React.ReactNode;
}) {
  return (
    <div className="grid gap-2">
      <Label>{label}</Label>
      {children}
    </div>
  );
}
```

This is the everyday reason Studivo (and most codebases) can put small presentational helpers after
the main export.

### 2.2 Same-file helpers called from later exports

SMS sending is built as a stack of function declarations. The public exports at the bottom call
private helpers declared above them. That is readable, not mandatory: because they are declarations,
reversing the order would still run.

```28:36:lib/sms.ts
function getSmsApiKey(): string {
  const apiKey = process.env.SMS_IR_API_KEY;

  if (!apiKey) {
    throw new Error("SMS configuration is missing.");
  }

  return apiKey;
}
```

```148:157:lib/sms.ts
export async function sendVerificationCode(
  mobile: string,
  code: string,
): Promise<void> {
  await sendVerificationTemplate({
    mobile,
    templateId: getTemplateId("SMS_IR_TEMPLATE_ID"),
    parameters: [{ name: "CODE", value: code }],
  });
}
```

`sendVerificationCode` → `sendVerificationTemplate` → `getSmsApiKey` / `getTemplateId` /
`logSmsFailure`. Every name is a function declaration, so the call graph does not depend on source
order.

Lead intake does the same: `OPTIONS` / `POST` call `jsonResponse` and `invalidOriginResponse`
declared in the same file.

```16:50:app/api/public/leads/route.ts
function jsonResponse(
  body: Record<string, unknown>,
  status: number,
  origin?: string,
  headers?: HeadersInit,
) {
  return NextResponse.json(body, {
    status,
    headers: {
      ...(origin ? getMarketingLeadCorsHeaders(origin) : {}),
      ...headers,
    },
  });
}

function invalidOriginResponse() {
  return jsonResponse(
    {
      ok: false,
      code: "INVALID_ORIGIN",
      message: "مبدأ درخواست مجاز نیست.",
    },
    403,
  );
}

export function OPTIONS(request: NextRequest) {
  const origin = request.headers.get("origin");
  if (!isAllowedMarketingLeadOrigin(origin)) return invalidOriginResponse();
  // ...
}
```

---

## 3. `const` / `let` are not usable before their line (TDZ)

A `const` binding is hoisted as a _name_, but it is uninitialized until that line runs. Reading it
earlier throws `ReferenceError`.

### 3.1 Arrow functions stored in `const`

The service worker is plain JavaScript, not TypeScript. Predicates are `const` arrows, then used in
`fetch`:

```1:10:public/sw.js
const CACHE_NAME = "studivo-v2";
const ASSETS_TO_CACHE = [
  "/web-app-manifest-192x192.png",
  "/web-app-manifest-512x512.png",
  "/notification-icon-192.png",
];

const isNextAssetRequest = (request) => new URL(request.url).pathname.startsWith("/_next/");
const isNavigationRequest = (request) => request.mode === "navigate";
```

```40:47:public/sw.js
self.addEventListener("fetch", (event) => {
  if (event.request.method !== "GET") {
    return;
  }

  if (isNavigationRequest(event.request) || isNextAssetRequest(event.request)) {
    event.respondWith(fetch(event.request));
```

This works **because the `fetch` listener runs later**, after those `const` lines have executed.

If someone wrote this instead, it would crash on install / parse of the worker:

```js
// Would throw ReferenceError: Cannot access 'isNavigationRequest' before initialization
self.addEventListener("fetch", (event) => {
  if (isNavigationRequest(event.request)) {
    /* ... */
  }
});

const isNavigationRequest = (request) => request.mode === "navigate";
```

Wait — that last snippet is subtle. The _listener callback_ runs later, so a `const` below
`addEventListener` would still be initialized by the time `fetch` fires. The crash happens only if
you **call** the arrow during the TDZ, at module evaluation time:

```js
isNavigationRequest({ mode: "navigate" }); // ReferenceError
const isNavigationRequest = (request) => request.mode === "navigate";
```

A function declaration would survive that immediate call.

### 3.2 Component stored in `const`, then exported

```7:21:components/theme-toggle.tsx
const ThemeToggle = () => {
  const { resolvedTheme, setTheme } = useTheme();

  return (
    <Button
      variant="ghost"
      size="icon"
      onClick={() => setTheme(resolvedTheme === "dark" ? "light" : "dark")}
    >
      {resolvedTheme === "dark" ? <SunIcon /> : <MoonIcon />}
    </Button>
  );
};

export default ThemeToggle;
```

`export default ThemeToggle` **must** come after the `const`. This would be invalid:

```js
export default ThemeToggle; // TDZ
const ThemeToggle = () => {
  /* ... */
};
```

Compare with a function declaration, which _can_ be default-exported in either order:

```js
export default function ThemeToggle() {
  /* ... */
}
```

or even:

```js
export default ThemeToggle;
function ThemeToggle() {
  /* ... */
}
```

Studivo's onboarding wizard uses the declaration form, so the name exists for the whole module:

```51:55:app/onboarding/_components/onboarding-wizard.tsx
export function OnboardingWizard() {
  const router = useRouter();
  const [values, setValues] = React.useState<OnboardingValues>(initialValues);
```

### 3.3 Module-level `const` values

Prisma is created as a `const`. Nothing in `lib/db.ts` may read `prisma` above line 12.

```8:16:lib/db.ts
const adapter = new PrismaPg({
  connectionString: process.env.DATABASE_URL!,
});

export const prisma =
  globalForPrisma.prisma ??
  new PrismaClient({
    adapter,
  });
```

`adapter` is also `const`. Swapping those two blocks would throw: `PrismaClient` would try to close
over `adapter` while `adapter` is still in the TDZ.

Marketing lead rate-limit constants follow the same rule: declared first, then closed over by later
functions.

```8:26:lib/marketing-leads/public-api.ts
const RATE_LIMIT_WINDOW_MS = 10 * 60 * 1000;
const RATE_LIMIT_MAX_ATTEMPTS = 5;
// ...
const globalForRateLimit = globalThis as GlobalWithMarketingLeadRateLimit;
const rateLimitStore =
  globalForRateLimit.marketingLeadRateLimitStore ?? new Map<string, RateLimitEntry>();

globalForRateLimit.marketingLeadRateLimitStore = rateLimitStore;
```

Those `const`s are initialized top-to-bottom. `consumeMarketingLeadRateLimit` can use them because
it only _runs_ after the module has finished evaluating.

---

## 4. Nested declarations vs nested `const` — same component

`ReserveForm` is the best Studivo file for this contrast. Inside one function you get both styles.

### 4.1 Function declarations nested in a component

These are hoisted to the **top of `ReserveForm`**, not the module.

```448:460:app/(dashboard)/_components/reserve-form.tsx
  function handleSendStatusMessage() {
    if (!seat || !membership) return;
    const message = getStatusMessage(
      seat.status,
      membership.memberName,
      seat.seatNumber,
      studyHallName,
    );
    window.open(
      `sms:${membership.phoneNumber}?body=${encodeURIComponent(message)}`,
      "_blank",
    );
  }
```

`handleRenew`, `handleSwap`, `handleRelease`, and `handleRecordPayment` in the same file are the
same shape. You could theoretically call `handleSwap()` on the first line of `ReserveForm` and it
would exist.

Onboarding does this for field updaters:

```59:61:app/onboarding/_components/onboarding-wizard.tsx
  function updateField<K extends keyof OnboardingValues>(key: K, value: OnboardingValues[K]) {
    setValues((current) => ({ ...current, [key]: value }));
  }
```

Nested `function` declarations are re-created every render. They are hoisted _per invocation_ of the
component.

### 4.2 `const` handlers in the same component — not hoisted

A few lines above those declarations, the same component uses arrow functions assigned to `const`:

```418:446:app/(dashboard)/_components/reserve-form.tsx
  const handleStartDateChange = (selectedDate: Date | undefined) => {
    if (!selectedDate) {
      setStartDate(undefined);
      return;
    }
    const adjusted = startOfDay(selectedDate);
    setStartDate(adjusted);
    applyPlanDates(selectedPlan, adjusted, getNormalizedQuantity(quantity));
  };

  const handlePlanChange = (planId: string) => {
    setSelectedPlanId(planId);
    const plan = membershipPlans.find((item) => item.id === planId);
    applyPlanDates(plan, startDate, getNormalizedQuantity(quantity));
  };
  // ...
  const handleSuccess = () => {
    onOpenChange(false);
  };
```

`handleSuccess` is **not** available above its line. This would throw:

```js
handleSuccess(); // ReferenceError
const handleSuccess = () => {
  onOpenChange(false);
};
```

Settings uses the same `const` style for a tiny local function:

```96:97:app/(dashboard)/settings/_components/settings-tabs.tsx
  const router = useRouter();
  const refresh = () => router.refresh();
```

`refresh` cannot be called before that line, even though `saveGeneral` (a nested function
declaration just below) can.

### 4.3 Module-level helpers used by nested code

`startOfDay` / `endOfDay` are module-level function declarations. `ReserveForm` and
`getSmartRenewalPreview` both call them.

```143:163:app/(dashboard)/_components/reserve-form.tsx
function startOfDay(date: Date) {
  const normalized = new Date(date);
  normalized.setHours(0, 0, 0, 0);
  return normalized;
}

function endOfDay(date: Date) {
  const normalized = new Date(date);
  normalized.setHours(23, 59, 59, 999);
  return normalized;
}

function getDefaultStartDate() {
  return startOfDay(new Date());
}

function addPlanDays(start: Date, durationDays: number) {
  const end = new Date(start);
  end.setDate(end.getDate() + durationDays);
  return endOfDay(end);
}
```

Then the exported component uses them as a lazy state initializer — a function _reference_, not an
immediate call of an uninitialized `const`:

```291:296:app/(dashboard)/_components/reserve-form.tsx
  const [startDate, setStartDate] = React.useState<Date | undefined>(
    getDefaultStartDate,
  );
  const [endDate, setEndDate] = React.useState<Date | undefined>(() =>
    addPlanDays(getDefaultStartDate(), membershipPlans[0]?.durationDays ?? 30),
  );
```

`useState(getDefaultStartDate)` passes the hoisted function. React calls it during this render. That
only works because `getDefaultStartDate` is already initialized (function declaration).

If those helpers were `const getDefaultStartDate = () => ...` _below_ `ReserveForm`, passing them
into `useState` would still work: the component function runs after the module finishes. The failure
mode is only an **immediate** top-level call during TDZ.

---

## 5. Import hoisting

Every Studivo module starts with imports. That is not style. **`import` is hoisted and evaluated
before the module body.**

```1:3:app/api/cron/renewal-reminders/route.ts
import { NextRequest, NextResponse } from "next/server";

import { sendRenewalReminders } from "@/lib/renewal-notifications";
```

```15:21:app/api/cron/renewal-reminders/route.ts
export async function GET(request: NextRequest) {
  if (!isAuthorized(request)) {
    return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
  }

  try {
    const result = await sendRenewalReminders();
```

You can imagine `sendRenewalReminders` being referenced anywhere in this file, including above the
`import` line in source. Bundlers still bind it first. Do not actually write imports at the bottom;
ESLint and humans will hate it. The engine would accept it.

Type-only imports (`import type { Metadata } from "next"` in `app/[slug]/page.tsx`) are erased at
compile time. They never exist at runtime, so they are not a runtime hoisting story.

Side-effect imports _do_ run, in dependency order, before this module's other statements:

```26:26:app/[slug]/page.tsx
import "./venue-public.css";
```

That CSS import is hoisted with the others. It is not delayed until `generateMetadata` runs.

---

## 6. Class declarations: hoisted name, TDZ value

Studivo has one small class:

```19:19:lib/upload.ts
export class UploadValidationError extends Error {}
```

```31:38:lib/upload.ts
export async function saveUploadedFile(
  file: File,
  subpath: string,
): Promise<{ url: string }> {
  if (!ALLOWED_TYPES.includes(file.type)) {
    throw new UploadValidationError(
      "فقط فایل‌های JPEG، PNG و WebP مجاز هستند.",
    );
  }
```

`class` is hoisted like `let`: the name exists, the constructor does not until that line. This would
throw during module init:

```js
throw new UploadValidationError("..."); // ReferenceError
export class UploadValidationError extends Error {}
```

`saveUploadedFile` is safe because it only constructs the class when a request hits it, after the
module has evaluated.

`ALLOWED_TYPES` / `MAX_BYTES` above the class are `const`. Same TDZ rule.

---

## 7. Default parameters are a mini-TDZ

Default values run **at call time**, left to right, in their own TDZ.

```1:1:lib/action-errors.ts
export function getActionErrorMessage(error: unknown, fallback = "انجام عملیات ناموفق بود.") {
```

```168:171:lib/finance-range.ts
export function getFinancePresetRange(
  preset: FinancePreset,
  now: Date = new Date(),
): FinanceRange {
```

```16:16:hooks/usePWA.ts
export function usePWA(options?: UsePWAOptions) {
```

Related facts, using these signatures:

- `fallback` is created only if the caller omits the second argument (or passes `undefined`).
- Inside the parameter list, you cannot use `fallback` before it is initialized. This is illegal:

  ```js
  function getActionErrorMessage(error = fallback, fallback = "...") {}
  ```

- A later default _may_ read an earlier parameter:

  ```js
  function example(error, fallback = String(error)) {}
  ```

- `now: Date = new Date()` in `getFinancePresetRange` runs a fresh `new Date()` on every call that
  omits `now`. It is not hoisted to module load.

Optional object params (`usePWA(options?: UsePWAOptions)`) are TypeScript. At runtime that is just
`options` possibly `undefined`. No default initializer, so no parameter TDZ.

---

## 8. What Studivo deliberately avoids: `var`

A search of `*.js` / `*.ts` / `*.tsx` finds **zero `var` bindings**.

That is the point of the modern default. `var` is function-scoped and initialized to `undefined`, so
this classic trap does not exist in Studivo:

```js
console.log(seatCount); // undefined, not ReferenceError
var seatCount = 30;
```

The Studivo equivalent uses `const` and would throw instead of silently continuing:

```17:22:app/onboarding/_components/onboarding-wizard.tsx
const DEFAULT_SECTION: OnboardingValues["sections"][number] = {
  name: "سالن اصلی",
  mode: "AUTO",
  seatCount: 30,
  manualSeats: "",
};
```

If onboarding code read `DEFAULT_SECTION` above that line, you would get a hard `ReferenceError`.
That is better than `undefined` walking into Prisma.

`var` is also **not** block-scoped. This mental model does not apply to Studivo `const` / `let`
inside `if` / `try`:

```js
if (true) {
  var leaked = 1;
}
leaked; // 1 with var; ReferenceError with let/const
```

When you see a `const` inside `usePWA`'s `useEffect`, it stays inside that effect callback:

```23:35:hooks/usePWA.ts
  useEffect(() => {
    const handleBeforeInstallPrompt = (e: Event) => {
      e.preventDefault();
      setDeferredPrompt(e as BeforeInstallPromptEvent);
      setIsInstallable(true);
    };

    window.addEventListener('beforeinstallprompt', handleBeforeInstallPrompt);

    return () => {
      window.removeEventListener('beforeinstallprompt', handleBeforeInstallPrompt);
    };
  }, []);
```

`handleBeforeInstallPrompt` is block-scoped to the effect. It is not hoisted onto `usePWA`.

---

## 9. Closures are not hoisting

Easy mix-up: a function _closing over_ a later `const` looks like hoisting. It is not.

```87:96:lib/marketing-leads/public-api.ts
export function consumeMarketingLeadRateLimit(request: NextRequest) {
  const now = Date.now();
  pruneExpiredRateLimitEntries(now);

  const key = getClientIp(request);
  const existing = rateLimitStore.get(key);

  if (!existing || existing.resetAt <= now) {
    const resetAt = now + RATE_LIMIT_WINDOW_MS;
```

`consumeMarketingLeadRateLimit` mentions `rateLimitStore` and `RATE_LIMIT_WINDOW_MS`, which are
declared earlier in the file. Even if the function were written _above_ those `const`s:

```js
export function consumeMarketingLeadRateLimit(request) {
  const resetAt = now + RATE_LIMIT_WINDOW_MS; // lookup happens at CALL time
}
const RATE_LIMIT_WINDOW_MS = 10 * 60 * 1000;
```

the lookup happens when the function **runs**, not when it is **created**. By call time the `const`
is initialized. That is a closure over a live binding, not hoisting.

Hoisting is about using a binding **during the TDZ of the current scope**. Closures delay the read
until later.

Same idea in `lib/reminders.ts`: `sendRenewalReminders` calls `clampReminderDaysBefore`,
`getDedupeKey`, and friends. Those helpers can sit above or below the export. The work runs on the
cron request, after the module loaded.

---

## 10. Mental model for reading Studivo

When you open a file, classify each name:

1. **`import`** — available for the whole module.
2. **`function name()` at module scope** — available for the whole module. Source order is for
   humans.
3. **`function name()` inside a component** — available for that render, from the first line of the
   component.
4. **`const name = () => {}` or `const name = value`** — available only **after** that line in the
   same scope.
5. **`class Name`** — like `const`: name exists, value is TDZ until the class line.
6. **Default params** — own tiny TDZ, evaluated on each call.

That is why Studivo can:

- put `Field` / `urlBase64ToUint8Array` / `jsonResponse` under the thing that uses them;
- put `const CACHE_NAME` and `const isNavigationRequest` at the top of `public/sw.js`;
- mix `const handlePlanChange = ...` and `function handleSwap()` in `ReserveForm` without those two
  names behaving the same.

---

## 11. Exercises on this repo

Do these in your head (or a scratch file). Do not change production code.

1. In `hooks/usePWA.ts`, rewrite `urlBase64ToUint8Array` as
   `const urlBase64ToUint8Array = (...) => { ... }` **below** `usePWA`. Does subscribe still work?
   Why?
2. Move that same `const` **above** `usePWA` but call it once at module top level. What happens?
3. In `components/theme-toggle.tsx`, move `export default ThemeToggle` above the `const`. Predict
   the error.
4. In `public/sw.js`, change the two predicates to `function isNavigationRequest(request) { ... }`.
   What new call sites become legal?
5. In `lib/upload.ts`, construct `new UploadValidationError("x")` on line 1. Which rule fires —
   class TDZ or import hoisting?
6. In `ReserveForm`, imagine calling `handleSuccess()` as the first statement of the component, then
   calling `handleSwap()` as the first statement. One throws. Which, and why?

---

## Files cited

| File                                                     | Why it is here                                     |
| -------------------------------------------------------- | -------------------------------------------------- |
| `hooks/usePWA.ts`                                        | Function declaration used above its source line    |
| `app/(dashboard)/settings/_components/settings-tabs.tsx` | Helpers below export; nested `function` vs `const` |
| `lib/sms.ts`                                             | Stack of hoisted private helpers                   |
| `app/api/public/leads/route.ts`                          | Route handlers calling same-file declarations      |
| `public/sw.js`                                           | `const` arrows in plain JS                         |
| `components/theme-toggle.tsx`                            | `const` component + default export order           |
| `lib/db.ts`                                              | Module `const` initialization order                |
| `lib/marketing-leads/public-api.ts`                      | `const` state + later function closures            |
| `app/(dashboard)/_components/reserve-form.tsx`           | Nested declaration vs nested `const`               |
| `app/onboarding/_components/onboarding-wizard.tsx`       | Nested `function` updaters                         |
| `app/api/cron/renewal-reminders/route.ts`                | Import hoisting                                    |
| `lib/upload.ts`                                          | `class` TDZ vs later `throw new ...`               |
| `lib/action-errors.ts`                                   | Default parameter                                  |
| `lib/finance-range.ts`                                   | Default `new Date()` at call time                  |
| `lib/reminders.ts`                                       | Helpers closed over by a later export              |

When you pick the next JavaScript topic (scope, closures, `this`, promises, event loop, modules, …),
we can do the same thing: hunt Studivo, then write the next note beside this one.
