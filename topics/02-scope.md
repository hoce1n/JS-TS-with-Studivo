# JavaScript Scope — Studivo Walkthrough

Hoisting asked _when_ a name exists. Scope asks **where** a name is visible.

A binding (variable, function, class, parameter, import) is visible only inside the **scope** that
created it, plus nested scopes that do not shadow it.

Studivo is ES modules + `const` / `let` / `function`. That already decides most of the story:

- each file is its **own module scope**;
- there is **no `var`**, so you almost never get function-wide leakage;
- `const` / `let` are **block-scoped**;
- nested functions **close over** outer bindings (the next note will go deeper on closures).

Read this with the cited files open. Predict: _from this line, which `error` / `user` / `startOfDay`
is this?_

---

## 1. The four scopes you will actually meet

| Scope               | Created by                                           | Lives until                                   | Studivo example                               |
| ------------------- | ---------------------------------------------------- | --------------------------------------------- | --------------------------------------------- |
| **Module**          | one `.ts` / `.tsx` / `.js` file                      | process / tab that loaded the module          | `OTP_EXPIRY_MINUTES` in `lib/otp.ts`          |
| **Function**        | `function`, arrow, method, constructor               | that call returns (unless a closure keeps it) | `reserveSeat(formData)`                       |
| **Block**           | `{ }`, `if`, `for`, `catch`, `switch` case with `{}` | the block finishes                            | `catch (error)`, `for (const candidate of …)` |
| **Script / global** | non-module classic script                            | the page                                      | almost absent here; Next.js bundles modules   |

There is also a **parameter list** scope (defaults cannot see later params). That showed up in the
hoisting note.

Lexical rule, always:

> Inner scopes can read outer names. Outer scopes cannot read inner names. Same name in an inner
> scope **shadows** the outer one.

---

## 2. Module scope: one file, one world

Everything at the top level of a file belongs to **that file only**, unless it is `export`ed.

```12:16:lib/otp.ts
const OTP_EXPIRY_MINUTES = 5;

export function generateOtpCode(): string {
  return String(Math.floor(100000 + Math.random() * 900000));
}
```

`OTP_EXPIRY_MINUTES` is module-private. `createOtpVerification` in the same file can use it.
`reserveSeat` cannot. No other file even sees the name.

Exports punch a hole **by name**, not by “the whole file becomes global”:

```4:6:lib/utils.ts
export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs))
}
```

Importers get `cn` in _their_ module scope. They do not inherit `clsx` from `lib/utils.ts`.

### 2.1 Same name, different modules — not a clash

`generateOtpCode` exists in two files:

```14:16:lib/otp.ts
export function generateOtpCode(): string {
  return String(Math.floor(100000 + Math.random() * 900000));
}
```

```30:32:lib/auth-plugins/phone-otp-login.ts
function generateOtpCode() {
  return String(Math.floor(100000 + Math.random() * 900000));
}
```

These are **two bindings**. The plugin’s copy is not exported, so it cannot collide with
`lib/otp.ts` at runtime. Module scope is the reason duplicate helpers are legal (whether or not they
are a good idea).

`startOfDay` is the same pattern across UI and the server action:

```19:23:app/(dashboard)/_lib/actions/reserve-seat.ts
function startOfDay(date: Date) {
  const normalized = new Date(date);
  normalized.setHours(0, 0, 0, 0);
  return normalized;
}
```

```143:147:app/(dashboard)/_components/reserve-form.tsx
function startOfDay(date: Date) {
  const normalized = new Date(date);
  normalized.setHours(0, 0, 0, 0);
  return normalized;
}
```

Changing one does not change the other. Scope isolation is why copy-paste helpers stay independent.

### 2.2 Module state is still scoped — it is just long-lived

```18:39:lib/push.ts
let vapidConfigured = false;

function ensureVapidConfigured() {
  if (vapidConfigured) {
    return;
  }
  // ...
  vapidConfigured = true;
}
```

`vapidConfigured` is **not** global. It is module-scoped. Every call to `ensureVapidConfigured` in
this process sees the same binding. A different serverless isolate may get a fresh module and a
fresh `false`.

Rate limiting uses the same idea, then hangs the Map on `globalThis` so hot reload does not reset
it:

```22:26:lib/marketing-leads/public-api.ts
const globalForRateLimit = globalThis as GlobalWithMarketingLeadRateLimit;
const rateLimitStore =
  globalForRateLimit.marketingLeadRateLimitStore ?? new Map<string, RateLimitEntry>();

globalForRateLimit.marketingLeadRateLimitStore = rateLimitStore;
```

`rateLimitStore` is still a module binding. `globalThis.marketingLeadRateLimitStore` is the actual
process-wide slot. Mixing those two is a product decision, not a language accident.

`lib/db.ts` does the same for Prisma.

---

## 3. Function scope: parameters and locals

A function creates a new scope every **call**.

```73:75:app/(dashboard)/_lib/actions/reserve-seat.ts
export async function reserveSeat(formData: FormData): Promise<ActionResult> {
  const user = await requireScopedUser();
  const { studyHallId } = user;
```

Per invocation:

- `formData` is a parameter binding;
- `user` and `studyHallId` are function-local `const`s;
- nothing outside `reserveSeat` can read them.

The nested transaction callback is **another** function scope, nested inside:

```116:132:app/(dashboard)/_lib/actions/reserve-seat.ts
    await prisma.$transaction(async (tx) => {
      const seat = await tx.seat.findFirst({
        where: {
          id: seatId,
          isActive: true,
          section: { studyHallId },
        },
```

Lookup for `studyHallId` goes: callback scope (miss) → `reserveSeat` scope (hit). That is **lexical
scope**: decided by where the function is _written_, not by who calls `$transaction`.

`tx` and `seat` are invisible to the outer `reserveSeat` body. After the callback returns, those
names are gone (unless something closed over them).

### 3.1 Parameters are local bindings

```171:177:app/(dashboard)/_components/reserve-form.tsx
function getStatusMessage(
  statusLabel: string,
  memberName: string,
  seatNumber: string,
  studyHallName: string,
): string {
  const baseMessage = `سلام ${memberName} عزیز، از سالن مطالعه ${studyHallName || ""} مزاحمتون میشم. `;
```

`memberName` here is not the React state in `ReserveForm`. It is an argument. Calling
`getStatusMessage(seat.status, membership.memberName, …)` copies a string into this inner binding.

Default parameters are also local:

```1:1:lib/action-errors.ts
export function getActionErrorMessage(error: unknown, fallback = "انجام عملیات ناموفق بود.") {
```

`fallback` exists only inside this function. The default expression runs in the parameter scope of
that call.

---

## 4. Block scope: `const` / `let` stop at the nearest `{ }`

Because Studivo uses `const` / `let`, a block is a real scope.

### 4.1 `if` / `try` / later use

SMS sending declares `response` **outside** the `try` so the rest of the function can see it:

```99:122:lib/sms.ts
  let response: Response;

  try {
    response = await fetch(SMS_IR_VERIFY_URL, {
      method: "POST",
      // ...
    });
  } catch (error) {
    logSmsFailure(
      "Failed to reach SMS.ir API",
      error instanceof Error ? error.message : "Network error",
    );
    throw new Error(SMS_SEND_FAILURE_MESSAGE);
  }

  let payload: unknown;

  try {
    payload = await response.json();
```

If it had been `const response = await fetch(...)` inside `try`, `response.json()` below would not
compile: `response` would be block-scoped to `try`.

That is the usual Studivo pattern when a value must survive a block: declare with `let` in the
**outer** function scope, assign inside.

`payload` repeats it for the second `try`.

### 4.2 `catch (error)` is its own block

```111:117:lib/sms.ts
  } catch (error) {
    logSmsFailure(
      "Failed to reach SMS.ir API",
      error instanceof Error ? error.message : "Network error",
    );
    throw new Error(SMS_SEND_FAILURE_MESSAGE);
  }
```

`error` lives only in that `catch`. The next `catch {` in the same function binds nothing:

```123:129:lib/sms.ts
  } catch {
    logSmsFailure(
      "SMS.ir API returned non-JSON response",
      `HTTP ${response.status}`,
    );
```

Optional catch binding (`catch {`) is a scope with **no** `error` name. You cannot accidentally
reuse the first `error`.

Onboarding’s outer catch cannot see `studyHall` from inside the transaction:

```113:118:app/onboarding/actions.ts
  } catch (error) {
    return {
      success: false,
      error: error instanceof Error ? error.message : "راه‌اندازی سالن با خطا مواجه شد.",
    };
  }
```

Here the object field `error:` does not clash with the catch binding `error` in a confusing way at
runtime: the shorthand is `{ error: error.message }`. The **parameter** `error` is the catch
binding. The **property name** `error` is just a key on the returned object. Different kinds of
name.

### 4.3 `for (const … of …)` — a fresh binding per iteration

```185:215:lib/reminders.ts
  for (const candidate of candidates) {
    const recipients = await prisma.staffAssignment.findMany({
      where: {
        studyHallId: candidate.studyHallId,
        // ...
      },
      // ...
    });

    const message = buildReminderMessage(candidate);
    const dedupeKey = getDedupeKey(
      candidate.kind,
      candidate.membershipId,
      todayTehranKey,
    );

    for (const recipient of recipients) {
```

Each loop trip gets its own `candidate`, `recipients`, `message`, `dedupeKey`. The inner loop gets
its own `recipient`.

`todayTehranKey` is not declared in the loop. Lookup walks out to `sendRenewalReminders`. That outer
`const` is shared by every iteration — one day key, many candidates.

`counters` sits in the function scope, so inner loops can mutate it:

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

Mutation of an outer object is allowed. Rebinding `const counters = …` inside the loop would be a
**new** inner binding, and the outer counters would stay at zero.

`sendPushToMany` uses the same loop-local `result`:

```96:111:lib/push.ts
  let sent = 0;
  let stale = 0;
  let failed = 0;

  for (const result of results) {
    if (result.status === "fulfilled") {
      if (result.value.ok) {
        sent += 1;
      } else if (result.value.stale) {
        stale += 1;
      }
      continue;
    }

    failed += 1;
  }
```

`sent` / `stale` / `failed` are function-scoped `let` so they accumulate. `result` is block-scoped
to one iteration.

Classic `var i` in a `for` would have been **one** `i` for the whole function. Studivo never does
that. `const` in `for…of` is the safe default, especially if you later pass `candidate` into an
async callback: each iteration already has its own binding.

### 4.4 Nested `if` blocks

```21:38:lib/push.ts
  if (vapidConfigured) {
    return;
  }

  const publicKey = process.env.NEXT_PUBLIC_VAPID_PUBLIC_KEY;
  const privateKey = process.env.VAPID_PRIVATE_KEY;

  if (!publicKey || !privateKey) {
    throw new Error("VAPID keys are not configured.");
  }

  webpush.setVapidDetails(
    process.env.VAPID_SUBJECT ?? "mailto:1hccein@gmail.com",
    publicKey,
    privateKey,
  );
```

`publicKey` is function-scoped in `ensureVapidConfigured`, not scoped to the first `if`. It is
declared after the early return, so the “already configured” path never creates those bindings.

If they had been written inside `if (!publicKey || !privateKey) { const publicKey = … }`, the outer
`setVapidDetails` call would not see them.

### 4.5 `switch` without extra blocks

Studivo has one `switch`, and every `case` only `return`s. No extra `const` inside a case:

```179:188:app/(dashboard)/_components/reserve-form.tsx
  switch (statusLabel) {
    case "renewal":
    case "نیازمند تمدید":
      return `${baseMessage}اشتراک صندلی شماره ${seatNumber} شما رو به اتمام است. لطفاً جهت تمدید و حفظ صندلی خود اقدام کنید.`;
    case "expired":
    case "منقضی":
      return `${baseMessage}اشتراک صندلی شماره ${seatNumber} شما به اتمام رسیده است. در صورت تمایل به ادامه حضور، لطفاً نسبت به تمدید آن اقدام فرمایید.`;
    default:
      return `${baseMessage}خواستار ارتباط با شما در خصوص صندلی شماره ${seatNumber} بودم.`;
  }
```

If two `case`s both declared `const message = …` **without** wrapping `{ }`, they would share one
switch-block scope and collide. The fix is `case "expired": { const message = …; return message; }`.
This file sidesteps that by not declaring in cases.

---

## 5. Nested functions: inner scope + outer lookup

A nested `function` is both a new scope and a use of lexical lookup.

Onboarding wizard:

```51:61:app/onboarding/_components/onboarding-wizard.tsx
export function OnboardingWizard() {
  const router = useRouter();
  const [values, setValues] = React.useState<OnboardingValues>(initialValues);
  const [error, setError] = React.useState<string | null>(null);
  const [pending, startTransition] = React.useTransition();

  function updateField<K extends keyof OnboardingValues>(key: K, value: OnboardingValues[K]) {
    setValues((current) => ({ ...current, [key]: value }));
  }
```

`updateField` does not take `setValues` as an argument. It **looks up** `setValues` in
`OnboardingWizard`. That lookup is fixed at write time.

`current` is a parameter of the inner arrow. It shadows nothing important here. `value` in
`updateField` is the new field value, not React’s `values` state. Different names, on purpose.

Lead capture does the same:

```39:47:hooks/use-marketing-lead-capture.ts
  function updateFieldError(name: MarketingLeadFieldName, value: string) {
    const error = validateMarketingLeadField(name, value);
    setFieldErrors((currentErrors) => {
      const nextErrors = { ...currentErrors };
      if (error) nextErrors[name] = error;
      else delete nextErrors[name];
      return nextErrors;
    });
  }
```

Scopes in that snippet:

1. hook function — `setFieldErrors`, `updateFieldError`;
2. `updateFieldError` — `name`, `value`, `error`;
3. updater arrow — `currentErrors`, `nextErrors`.

The inner `error` (string | undefined from validation) does **not** touch any component
`errorMessage` state. Inner name, inner scope.

Mobile breakpoint is module-scoped, read from nested arrows:

```3:21:hooks/use-mobile.ts
const MOBILE_BREAKPOINT = 768;

export function useIsMobile() {
  const [isMobile, setIsMobile] = React.useState(() => {
    if (typeof window === "undefined") {
      return false
    }

    return window.innerWidth < MOBILE_BREAKPOINT
  })

  React.useEffect(() => {
    const mql = window.matchMedia(`(max-width: ${MOBILE_BREAKPOINT - 1}px)`)
    const onChange = () => {
      setIsMobile(window.innerWidth < MOBILE_BREAKPOINT)
    }
```

`MOBILE_BREAKPOINT` is found by walking: `onChange` → `useEffect` callback → `useIsMobile` →
**module**. `mql` is only inside the effect. `onChange` can use `mql` because it is nested in that
effect callback.

### 5.1 Tiny inner arrows as local scopes

```101:107:lib/date.ts
  const get = (type: Intl.DateTimeFormatPartTypes) =>
    parts.find((part) => part.type === type)?.value;

  return {
    year: get("year") ?? "",
    month: get("month") ?? "",
    day: get("day") ?? "",
  };
```

`get` is scoped to `formatTehranParts`. `parts` is in that same function. The arrow closes over
`parts`. Callers of `formatTehranDateKey` cannot call `get`.

`lib/finance-range.ts` repeats this as `valueFor` inside `getTimeZoneOffsetMs`. Two files, two
private helpers, same idea: shrink the name’s scope to the function that needs it.

---

## 6. Shadowing: the inner name wins

Shadowing is not a bug. It is the rule.

### 6.1 Catch parameter vs returned field

Already noted in onboarding: `catch (error)` then `{ error: error.message }`. The binding `error` is
the exception. The key `'error'` is unrelated.

`ActionForm` shadows in a more interesting way:

```46:60:components/action-form.tsx
  const [errorMessage, setErrorMessage] = React.useState<string | null>(null);
  const [pending, startTransition] = React.useTransition();

  function handleSubmit(event: React.FormEvent<HTMLFormElement>) {
    event.preventDefault();
    const form = event.currentTarget;
    const formData = new FormData(form);

    setErrorMessage(null);
    startTransition(async () => {
      try {
        const result = await action(formData);

        if (isServerActionResult(result) && result.success === false) {
          const message = result.error || "انجام عملیات ناموفق بود.";
```

`action` the **parameter** of `ActionForm` shadows nothing in the module (the module’s `action` is
not a binding; the prop is). Inside `handleSubmit`, `action(formData)` is that prop, not a DOM
attribute.

Then:

```78:85:components/action-form.tsx
      } catch (error) {
        if (isNextNavigationError(error)) {
          throw error;
        }

        const message = getActionErrorMessage(error);
        setErrorMessage(message);
        toast.error(message);
      }
```

`error` in `catch` is not `errorMessage`. `message` in the `if` block and `message` in the `catch`
block are **two** block-scoped `const`s. They never exist at the same time. Reusing the name is fine
because the scopes do not overlap.

### 6.2 Nested `error` in push send

```70:74:lib/push.ts
  } catch (error) {
    const statusCode =
      error && typeof error === "object" && "statusCode" in error
        ? Number(error.statusCode)
        : undefined;
```

`error` is the catch binding. `error.statusCode` is a property read, not a new scope. After the
`catch` block, that `error` is gone. `throw error` rethrows the **same** binding.

### 6.3 What Studivo does _not_ do

You will not find:

```js
const error = "outer";
function f() {
  const error = "inner";
  // ...
}
```

often enough to quote. The real shadowing is quieter: **parameter vs module**, **catch vs
function**, **loop `const` vs outer `const`**.

Imagine renaming badly in reminders:

```js
const message = "debug";
for (const candidate of candidates) {
  const message = buildReminderMessage(candidate); // shadows outer
}
```

The outer `message` would be untouched. Inner loops would use the object `{ title, body }`. That is
shadowing doing its job — and hiding a leftover debug binding.

---

## 7. Component scope vs module helpers

`ReserveForm` is one function scope per render. Nested handlers live in that scope.

```448:455:app/(dashboard)/_components/reserve-form.tsx
  function handleSendStatusMessage() {
    if (!seat || !membership) return;
    const message = getStatusMessage(
      seat.status,
      membership.memberName,
      seat.seatNumber,
      studyHallName,
    );
```

Name resolution:

| Name                                  | Found in                            |
| ------------------------------------- | ----------------------------------- |
| `seat`, `membership`, `studyHallName` | `ReserveForm` (closure over render) |
| `message`                             | `handleSendStatusMessage`           |
| `getStatusMessage`                    | **module** (sibling function)       |

`getStatusMessage` is not inside the component. It cannot read `seat`. That is why the author passed
four arguments instead of closing over the component. Smaller scope, easier to test, no accidental
stall on stale render data for that particular helper.

Contrast `handleRenew`, which _does_ close over `membership`, `renewDate`, `startDate`, setters, and
`onOpenChange`. Wider scope, more coupling, fewer parameters.

Choosing which names live in which scope is the real design skill. The language only tells you what
is _possible_.

---

## 8. Global scope: treated as a last resort

Browser globals (`window`, `Notification`, `navigator`) are **not** declared in Studivo modules.
They live on the global object.

```7:11:hooks/use-mobile.ts
    if (typeof window === "undefined") {
      return false
    }

    return window.innerWidth < MOBILE_BREAKPOINT
```

`window` is resolved on the global object (or missing on the server). `MOBILE_BREAKPOINT` is
module-scoped. Mixing them is why the `typeof window === "undefined"` guard exists: the module can
load in Node, where that global is absent.

Service worker globals (`self`, `caches`, `clients`) in `public/sw.js` are the worker’s global
scope. `CACHE_NAME` is still a module-level `const` in that file (classic worker script, one file =
one script scope).

Studivo does **not** write `globalSeatCount = 30` without `const`. Sloppy assignment would create a
real global in non-strict classic scripts. ES modules are strict: that would be `ReferenceError`.
Another reason the repo stays in modules.

---

## 9. Scope vs hoisting (how the two notes fit)

Hoisting is a **time** rule inside one scope. Scope is a **place** rule across nested regions.

In `usePWA`, `urlBase64ToUint8Array` is visible throughout the **module** because it is a function
declaration in that module (hoisting). It is **not** visible in `lib/otp.ts` (scope).

In `ReserveForm`:

- `function handleSwap()` is hoisted to the top of **that component invocation**;
- `const handleSuccess = () => {}` is in the same function scope but TDZ until its line;
- neither is visible in `reserveSeat` on the server.

When you get a `ReferenceError`, ask two questions in order:

1. Am I still **before** the `const` / `let` / `class` line? → hoisting / TDZ.
2. Am I **outside** the `{ }` / function / file that declared it? → scope.

---

## 10. A lookup walk, start to finish

Take this inner line from reminders:

```javascript
counters.duplicatesSkipped += 1;
```

Engine walk:

1. Is `counters` a binding in the `catch` block? No.
2. In the `try`? No.
3. In the `for (const recipient of recipients)` block? No.
4. In the `for (const candidate of candidates)` block? No.
5. In `sendRenewalReminders`? **Yes** — the object created before the loops.

Then `duplicatesSkipped` is not a scope lookup. It is a **property** of that object. Properties are
not lexical. `counters.foo` would be `undefined`, not a `ReferenceError`, unless you freeze the
object.

Second walk, `isNotificationClaimConflict(error)`:

1. `error` — catch parameter, found immediately.
2. `isNotificationClaimConflict` — not in catch, not in loops, not in `sendRenewalReminders` locals
   → **module** function declaration.

That is the entire algorithm. Nested scopes, then module, then global.

---

## 11. Exercises on this repo

Do these in your head. Do not change production code.

1. In `lib/otp.ts`, could `phone-otp-login.ts` call `OTP_EXPIRY_MINUTES` without importing? What
   error?
2. In `lib/sms.ts`, move `const response = await fetch(...)` inside `try` (no outer `let`). Which
   later line breaks, and is that TypeScript or JS scope?
3. In `lib/reminders.ts`, declare `const counters` **inside** the `candidate` loop. What happens to
   totals?
4. In `use-mobile.ts`, could `onChange` read `mql` if you moved `const mql = …` below the
   `return () => …` cleanup? (Hint: same function scope, TDZ vs scope.)
5. In `ActionForm`, rename the catch binding to `err`. Does `getActionErrorMessage(error)` still
   compile? Why?
6. In `ReserveForm`, could `getStatusMessage` read `seat` directly if you nested it _inside_
   `ReserveForm`? What would that do to every render?
7. Two `startOfDay` functions exist. You fix a timezone bug in the form helper. Does reservation on
   the server change? Which scope rule is that?

---

## Files cited

| File                                               | Why it is here                                               |
| -------------------------------------------------- | ------------------------------------------------------------ |
| `lib/otp.ts`                                       | Module-private `const`; exported vs private functions        |
| `lib/auth-plugins/phone-otp-login.ts`              | Same function name, other module                             |
| `app/(dashboard)/_lib/actions/reserve-seat.ts`     | Function scope + nested transaction callback                 |
| `app/(dashboard)/_components/reserve-form.tsx`     | Duplicate helper; switch; nested handlers vs module helper   |
| `lib/utils.ts`                                     | Exports do not leak imports                                  |
| `lib/push.ts`                                      | Module `let` state; `if` blocks; loop `const` vs outer `let` |
| `lib/marketing-leads/public-api.ts`                | Module binding vs `globalThis`                               |
| `lib/sms.ts`                                       | Outer `let` vs `try` block; `catch (error)` vs `catch {`     |
| `app/onboarding/actions.ts`                        | Catch cannot see inner `studyHall`                           |
| `lib/reminders.ts`                                 | Nested `for…of`; shared `counters`; per-iteration `const`    |
| `app/onboarding/_components/onboarding-wizard.tsx` | Nested function lookup                                       |
| `hooks/use-marketing-lead-capture.ts`              | Inner `error` vs state                                       |
| `hooks/use-mobile.ts`                              | Module const + effect-local bindings + `window`              |
| `lib/date.ts`                                      | Inner helper `get` scoped to one function                    |
| `lib/finance-range.ts`                             | Same pattern as `valueFor`                                   |
| `lib/action-errors.ts`                             | Parameter scope / default                                    |
| `components/action-form.tsx`                       | Overlapping `message` blocks; catch vs state                 |
| `public/sw.js`                                     | Worker global vs file-level `const`                          |

When you pick the next topic (closures, `this`, promises, the event loop, modules, …), we can write
the next note beside this one and the hoisting note.
