# JavaScript Control Flow and Exceptions — Studivo Walkthrough

Loops were **walking a collection**. This note is **choosing a path**, then **what happens when a
path fails**.

MDN’s split:

```text
Control flow
├── if / else if / else     truthy test, early return
├── switch                  discrete labels, fall-through unless return/break
├── ? :  &&  ||  ??  ?.     expression-level branches
└── Exceptions
    ├── throw               leave the current stack
    ├── try / catch / finally
    └── Error objects       Error, subclasses, instanceof
```

Read this with the cited files open. Predict: _does this return a value, skip the rest of the
function, abort a Prisma transaction, show a toast, or rethrow?_

Studivo’s product rule (AGENTS.md): **never swallow errors** that affect booking, payments, auth, or
permissions. Expected business failures become `{ success: false, error }`. Unexpected failures
`throw`. Navigation errors (`redirect` / `notFound`) must be rethrown, not converted into a toast.

---

## 1. The rule that pays rent

| Need                               | Construct                          | Studivo                                       |
| ---------------------------------- | ---------------------------------- | --------------------------------------------- |
| Guard, then stop                   | `if (!x) return` / `throw`         | occupancy, OWNER checks, OTP validation       |
| Classify a small set of labels     | `switch` + `return`                | SMS status copy — **one** `switch` in the app |
| Pick a value inline                | `? :` / `??` / `\|\|` / `?.`       | seat status, gallery JSON, optional names     |
| Operator did something invalid     | `throw new Error("…")` inside `tx` | reserved seat, duplicate phone, empty section |
| Expected failure the UI can render | `return { success: false }`        | Zod parse, slug taken, onboarding catch       |
| Distinguishing failure kinds       | `instanceof` then branch           | `UploadValidationError` → 400, else rethrow   |
| Cleanup that must run either way   | `finally`                          | loading flags, hidden file input reset        |
| Must not hide Next.js navigation   | `catch` + `throw error`            | `ActionForm`                                  |

`if` + early `return` / `throw` is the house branch. `switch` is rare. `try/catch` is a
**boundary**, not a blanket.

---

## 2. `if` — the house branch

`if (condition) statement`. Condition is converted with ToBoolean. Studivo almost always uses
**explicit** checks (`===`, `== null`, `.success`) rather than bare truthiness on domain values.

### 2.1 Early return — stop the function

```21:25:app/(dashboard)/_lib/dashboard-utils.ts
function toTime(value: Date | string | null | undefined): number | null {
  if (value == null) return null;
  const time = new Date(value).getTime();
  return Number.isNaN(time) ? null : time;
}
```

`== null` is the one place `==` is the tool: it matches both `null` and `undefined`. After that
line, `value` is a `Date` or string.

`getSeatStatus` is a **ladder** of early returns. First match wins. No `else`.

```89:101:app/(dashboard)/_lib/dashboard-utils.ts
export function getSeatStatus(
  endsAt?: Date | string | null,
  membershipStatus?: string
): SeatStatus {
  const endTime = toTime(endsAt ?? null);
  if (endTime == null) return "available";
  if (membershipStatus === "CANCELLED") return "available";
  if (membershipStatus === "EXPIRED") return "expired";

  const diffDays = Math.ceil((endTime - Date.now()) / DAY_IN_MS);
  if (diffDays < 0) return "expired";
  if (diffDays <= 3) return "renewal";
  return "reserved";
}
```

Order is the algorithm. Cancelled must beat “has an end date.” Expired membership must beat the
day-count. Swap two `if`s and the occupancy map lies.

`isOccupyingAssignment` is the same shape: cancelled → open `endsAt` → unreadable dates → legacy
equality.

```42:67:app/(dashboard)/_lib/dashboard-utils.ts
export function isOccupyingAssignment(assignment: {
  endsAt: Date | string | null;
  membership: { status: string; endsAt: Date | string };
}): boolean {
  if (assignment.membership.status === "CANCELLED") return false;

  // Canonical V2: Open assignments are active.
  if (assignment.endsAt == null) return true;

  const assignmentEnd = toTime(assignment.endsAt);
  const membershipEnd = toTime(assignment.membership.endsAt);

  if (assignmentEnd == null || membershipEnd == null) return false;
  // ...
  return isLegacyActive;
}
```

`||` in a condition is “either date unreadable.” That is still an `if`, not a fallback value.

### 2.2 Early return in the UI — validation before I/O

```104:107:app/(auth)/login/_components/login-form.tsx
    if (!isValidMobileNumber) {
      setMobileError("شماره موبایل باید ۱۱ رقم و با ۰۹ شروع شود.");
      return;
    }
```

```168:179:app/(dashboard)/profile/_components/profile-settings.tsx
    if (newPassword !== confirmPassword) {
      const message = "رمز عبور جدید و تکرار آن یکسان نیستند.";
      setPasswordError(message);
      toast.error(message);
      return;
    }

    if (newPassword.length < 8) {
      const message = "رمز عبور جدید باید حداقل ۸ کاراکتر باشد.";
      setPasswordError(message);
      toast.error(message);
      return;
    }
```

These are **not** exceptions. The operator can fix the input. `return` skips the `try`. Throwing
here would force a `catch` that does the same toast.

Lead capture: invalid fields → focus + `return`, never `submitLead`.

```84:89:hooks/use-marketing-lead-capture.ts
    if (firstInvalidField) {
      document
        .getElementById(getMarketingLeadFieldId(idPrefix, firstInvalidField))
        ?.focus();
      return;
    }
```

### 2.3 `if` then `throw` — abort work that must not commit

Inside `prisma.$transaction`, `throw` rolls the transaction back. That is why reserve/swap/release
**throw** instead of `return { success: false }` from inside the callback.

```130:150:app/(dashboard)/_lib/actions/reserve-seat.ts
      if (!seat) {
        throw new Error("صندلی مورد نظر در این سالن مطالعه یافت نشد.");
      }
      // ...
      if (existingAssignments.some(isOccupyingAssignment)) {
        throw new Error("این صندلی در حال حاضر دارای رزرو فعال است.");
      }
```

```196:203:app/(dashboard)/_lib/actions/reserve-seat.ts
      if (existingActiveMembership) {
        const occupiedSeat =
          existingActiveMembership.seatAssignments[0]?.seat.number;
        throw new Error(
          occupiedSeat
            ? `این شماره تلفن هم‌اکنون صندلی ${occupiedSeat} را در این سالن دارد.`
            : "این شماره تلفن هم‌اکنون یک اشتراک فعال در این سالن دارد.",
        );
      }
```

Nested ternary **inside** `throw new Error(...)` picks the message. The control-flow fact is still
the `if`: membership exists → leave the transaction.

Release is one compound guard:

```68:70:app/(dashboard)/_lib/actions/manage-seat.ts
      if (!assignment || !isOccupyingAssignment(assignment)) {
        throw new Error("تخصیص صندلی فعال برای تخلیه یافت نشد.");
      }
```

OWNER-only settings:

```140:145:app/(dashboard)/settings/actions.ts
async function requireOwner() {
  const user = await requireScopedUser();
  if (user.role !== "OWNER") {
    throw new Error("فقط مدیر سالن (OWNER) اجازه تغییر تنظیمات سالن را دارد.");
  }
  return user;
}
```

A **returned** 403 is the HTTP cousin of the same `if` (no throw — the route chooses the status):

```15:17:app/api/upload/image/route.ts
  if (user.role !== "OWNER") {
    return NextResponse.json({ error: "دسترسی غیر مجاز." }, { status: 403 });
  }
```

```5:17:app/api/cron/renewal-reminders/route.ts
function isAuthorized(request: NextRequest) {
  const cronSecret = process.env.CRON_SECRET;
  if (!cronSecret) {
    return process.env.NODE_ENV !== "production";
  }
  // ...
}
export async function GET(request: NextRequest) {
  if (!isAuthorized(request)) {
    return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
  }
```

Same idea, different channel: throw (server action / tx) vs return a Response (Route Handler).

---

## 3. `switch` — one file, fall-through on purpose

Studivo has **one** `switch` in application code: SMS copy for a seat’s status.

```171:188:app/(dashboard)/_components/reserve-form.tsx
function getStatusMessage(
  statusLabel: string,
  memberName: string,
  seatNumber: string,
  studyHallName: string,
): string {
  const baseMessage = `سلام ${memberName} عزیز، از سالن مطالعه ${studyHallName || ""} مزاحمتون میشم. `;

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
}
```

Facts:

1. `case "renewal":` with **no** `break` falls through to `case "نیازمند تمدید":`. Both labels share
   one `return`. That is intentional fall-through.
2. `return` exits the function. No `break` needed after it.
3. `default` is required here because `statusLabel` is a `string`, not a closed union. Without it,
   TypeScript still compiles; runtime would return `undefined`.
4. Seat **state** itself is an if-ladder (`getSeatStatus`), not a switch. Switch is for **copy**,
   not occupancy rules.

`getSeatStatus` could have been a switch on `membershipStatus`. It is not, because the last two
branches need `diffDays` arithmetic, not a label match.

---

## 4. Expression branches: `? :`, `||`, `??`, `?.`

These are still control flow. They do not `throw`. They pick a value.

| Operator    | When the right side runs                   | Studivo                                         |
| ----------- | ------------------------------------------ | ----------------------------------------------- |
| `a ? b : c` | `b` if `a` is truthy, else `c`             | duplicate-phone message; sort times             |
| `a \|\| b`  | `b` if `a` is falsy (`""`, `0`, `null`, …) | `studyHallName \|\| ""`; `parseInt(...) \|\| 0` |
| `a ?? b`    | `b` if `a` is `null` or `undefined` only   | `endsAt ?? null`; `payload.url ?? "/"`          |
| `a?.b`      | skip `b` if `a` is nullish                 | `assignments?.length`; `occupiedSeat` optional  |

```93:93:app/(dashboard)/_lib/dashboard-utils.ts
  const endTime = toTime(endsAt ?? null);
```

```169:172:app/(dashboard)/_lib/dashboard-utils.ts
    const seatNum = parseInt(seat.number, 10) || 0;
    const isDuplicate =
      membership?.id && (activeAssignmentsByMembership.get(membership.id)?.length ?? 0) > 1;
```

`|| 0` is deliberate: `NaN` is falsy, so a junk seat number becomes `0` for sort. `?? 0` would
**keep** `NaN` (`NaN` is not nullish). Different operators, different occupancy sorts.

Onboarding’s “has sections?” is a ternary that **builds the list** the `for…of` later walks:

```53:62:app/onboarding/actions.ts
      const sectionInputs = values.hasSections
        ? values.sections
        : [
            {
              name: "سالن اصلی",
              mode: "AUTO" as const,
              seatCount: values.seatCount,
              manualSeats: "",
            },
          ];
```

Settings JSON parse uses `|| "[]"` so empty form fields become a parseable array, not
`JSON.parse("")` which throws:

```160:164:app/(dashboard)/settings/actions.ts
    galleryImages = JSON.parse(
      formData.get("galleryImages")?.toString() || "[]",
    );
  } catch (error) {
    return actionError(error, "تصاویر گالری معتبر نیستند.");
```

`?.toString()` — missing field is `undefined`, then `|| "[]"`.

---

## 5. `throw` — leave this stack

`throw expression` stops the current function. The engine looks for the nearest `catch`. If none,
the promise rejects (async) or the process / overlay handles it.

You can throw anything. Studivo throws **`Error` instances** (or a subclass). That keeps
`instanceof Error` and `.message` working.

### 5.1 `throw new Error(message)` — operator-visible, Persian

Reserve, swap, release, payments, staff, onboarding empty section — all `new Error("…")` with a
message the UI can show.

```79:81:app/onboarding/actions.ts
        if (seatNumbers.length === 0) {
          throw new Error(`برای بخش «${sectionInput.name}» حداقل یک صندلی تعریف کنید.`);
        }
```

The message is interpolated **before** throw. `sectionInput.name` is the per-iteration `const` from
the loops note — safe to close over.

### 5.2 Config / invariant throws — English, not for the operator

```28:33:lib/sms.ts
function getSmsApiKey(): string {
  const apiKey = process.env.SMS_IR_API_KEY;

  if (!apiKey) {
    throw new Error("SMS configuration is missing.");
  }
```

```20:30:lib/push.ts
function ensureVapidConfigured() {
  if (vapidConfigured) {
    return;
  }
  // ...
  if (!publicKey || !privateKey) {
    throw new Error("VAPID keys are not configured.");
  }
```

```24:26:lib/upload.ts
  if (process.env.NODE_ENV === "production" && !uploadDir) {
    throw new Error("UPLOAD_DIR must be set in production.");
  }
```

These mean “the server is misconfigured.” Login maps the SMS one to a Persian return value (below).
Hooks that require a React context throw English too — a programmer error, not a hall-ops error:

```49:49:components/ui/sidebar.tsx
    throw new Error("useSidebar must be used within a SidebarProvider.")
```

### 5.3 `throw error` — rethrow, do not invent a new one

After handling the cases you understand, pass the same object on. Stack and identity stay intact.

```229:235:lib/reminders.ts
      } catch (error) {
        if (isNotificationClaimConflict(error)) {
          counters.duplicatesSkipped += 1;
          continue;
        }

        throw error;
      }
```

```71:84:lib/push.ts
  } catch (error) {
    const statusCode =
      error && typeof error === "object" && "statusCode" in error
        ? Number(error.statusCode)
        : undefined;

    if (statusCode === 404 || statusCode === 410) {
      await prisma.pushSubscription.deleteMany({
        where: { endpoint: record.endpoint },
      });

      return { ok: false as const, stale: true };
    }

    throw error;
  }
```

404/410 → expected, return a result. Anything else → still a failure, rethrow.

Upload route: validation → 400 JSON. Unknown → rethrow (Next.js error overlay / 500).

```32:41:app/api/upload/image/route.ts
  try {
    const result = await saveUploadedFile(file, pathname);
    return NextResponse.json(result);
  } catch (error) {
    if (error instanceof UploadValidationError) {
      return NextResponse.json({ error: error.message }, { status: 400 });
    }

    throw error;
  }
```

`ActionForm` rethrows Next navigation errors so `redirect()` still redirects:

```78:81:components/action-form.tsx
      } catch (error) {
        if (isNextNavigationError(error)) {
          throw error;
        }
```

Catching `redirect` and turning it into `toast.error` would be the AGENTS.md bug.

### 5.4 Wrap — new Error, old fact lost on purpose

SMS never leaks provider text to the client. Network / HTTP / JSON failures all become one constant:

```111:116:lib/sms.ts
  } catch (error) {
    logSmsFailure(
      "Failed to reach SMS.ir API",
      error instanceof Error ? error.message : "Network error",
    );
    throw new Error(SMS_SEND_FAILURE_MESSAGE);
  }
```

Log the real reason. Throw a **new** Error the UI is allowed to show. Login then matches that
message:

```92:94:app/(auth)/login/actions.ts
    } catch (error) {
      if (error instanceof Error && error.message === SMS_SEND_FAILURE_MESSAGE) {
        return { ok: false, error: error.message };
```

String-matching `.message` is brittle. It works here because `lib/sms.ts` owns the constant and
login imports the same one.

---

## 6. `try` / `catch` / `finally`

```javascript
try {
  // might throw
} catch (error) {
  // runs if throw / reject
} finally {
  // always, even if catch throws or try returns
}
```

`catch` binding is optional (`catch { }`). Studivo uses that when the value does not matter.

### 6.1 Boundary: convert throw → action result

Onboarding: anything the transaction throws becomes `{ success: false }`. The UI never sees a raw
exception from this action.

```39:117:app/onboarding/actions.ts
  try {
    await prisma.$transaction(async (tx) => {
      // ... throw new Error(`برای بخش «${sectionInput.name}» …`)
    });
  } catch (error) {
    return {
      success: false,
      error: error instanceof Error ? error.message : "راه‌اندازی سالن با خطا مواجه شد.",
    };
  }
```

The inner `throw` is still the right tool **inside** `tx` (rollback). The outer `catch` is the
HTTP/action boundary.

`getActionErrorMessage` is that conversion in one helper:

```1:14:lib/action-errors.ts
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

`catch (error)` types `error` as `unknown` (strict TS). You do **not** read `.message` until
`instanceof Error` or a duck check. `throw "string"` would miss the first branch and hit fallback —
another reason Studivo throws `Error`.

Cron route is the Route Handler version of the same boundary:

```20:36:app/api/cron/renewal-reminders/route.ts
  try {
    const result = await sendRenewalReminders();
    return NextResponse.json({ ok: true, /* ... */ });
  } catch (error) {
    const message =
      error instanceof Error ? error.message : "Failed to send renewal reminders";

    return NextResponse.json({ ok: false, error: message }, { status: 500 });
  }
```

### 6.2 `catch` without a binding — swallow a **known cheap** failure

`JSON.parse` of operator-controlled origin strings: bad URL → skip that origin, do not crash boot.

```38:43:lib/marketing-leads/public-api.ts
      try {
        const url = new URL(origin);
        return url.protocol === "http:" && ["localhost", "127.0.0.1"].includes(url.hostname);
      } catch {
        return false;
      }
```

`new URL(origin)` throws `TypeError` on garbage. The catch returns `false` from `.filter`. That is
allowed: one bad env token must not take down lead intake.

Lead submit: network/action throw → generic Persian line. The thrown value is unused.

```94:99:hooks/use-marketing-lead-capture.ts
      try {
        result = await submitLead(formData);
      } catch {
        setErrorMessage("مشکلی پیش آمد. لطفاً دوباره تلاش کنید.");
        return;
      }
```

SMS JSON body: `response.json()` can throw. Same pattern — log, throw the public Error, ignore the
parse exception’s text.

```121:128:lib/sms.ts
  try {
    payload = await response.json();
  } catch {
    logSmsFailure(
      "SMS.ir API returned non-JSON response",
      `HTTP ${response.status}`,
    );
    throw new Error(SMS_SEND_FAILURE_MESSAGE);
  }
```

Do **not** copy this for reserve/payment. Those failures must keep `.message`.

### 6.3 `finally` — loading and DOM reset

`finally` runs if `try` returns, if `catch` runs, and if `catch` throws. Studivo uses it for **UI
state that must clear**.

Avatar upload: success, throw, or early — the spinner dies and the file input clears (DOM note: same
file can be chosen again).

```125:147:app/(dashboard)/profile/_components/profile-settings.tsx
    setIsAvatarUploading(true);
    try {
      const url = await uploadAvatar(file);
      // ...
    } catch (error) {
      const message =
        error instanceof Error ? error.message : "خطا در آپلود تصویر پروفایل.";
      toast.error(message);
    } finally {
      setIsAvatarUploading(false);
      if (avatarInputRef.current) avatarInputRef.current.value = "";
    }
```

Password change: `try / catch / finally` with `setIsLoading(false)` in `finally`.

Email login: **`try / finally` with no `catch`**. Errors are handled in Better Auth’s `onError`.
`finally` still clears `loading` so a failed sign-in does not leave the button stuck.

```75:97:app/(auth)/login/_components/login-form.tsx
    try {
      await authClient.signIn.email(
        { /* ... */ },
        {
          onRequest: () => {
            setLoading(true);
          },
          onSuccess: () => {
            toast.success("خوش اومدی...");
          },
          onError: (ctx) => {
            toast.error(ctx.error.message);
          },
        },
      );
    } finally {
      setLoading(false);
    }
```

Phone OTP login uses the full trio: `try` / `catch` (toast + field error) / `finally`
(`setLoading(false)`).

`return` inside `try` still runs `finally` first:

```121:123:app/(auth)/login/_components/login-form.tsx
        setOtpSent(true);
        toast.success("اگر این شماره به حسابی متصل باشد، کد ورود برای آن ارسال می‌شود.");
        return;
```

Then `finally` sets loading false, then the function actually returns.

### 6.4 `try` without `finally` — classify and maybe continue

Reminder create: catch uniqueness, `continue` the inner loop, otherwise rethrow. No `finally`.
Nothing to reset.

ActionForm: catch → toast, unless navigation. No `finally`; pending state is `useTransition`.

---

## 7. Error objects

### 7.1 `Error` is a constructor

```javascript
const err = new Error("این صندلی در حال حاضر دارای رزرو فعال است.");
err.name; // "Error"
err.message; // the Persian string
err.stack; // call sites, engine-dependent
```

`throw new Error(msg)` vs `throw msg`: only the first is `instanceof Error`. Studivo’s catch helpers
assume the first.

Built-ins you will meet, even if Studivo rarely constructs them:

| Object           | Typical throw                                      |
| ---------------- | -------------------------------------------------- |
| `Error`          | application `throw new Error`                      |
| `TypeError`      | `new URL("not a url")`; reading property of `null` |
| `RangeError`     | invalid array length / number                      |
| `SyntaxError`    | `JSON.parse` of garbage — settings gallery catch   |
| `ReferenceError` | TDZ / missing binding (hoisting note)              |

`JSON.parse` of bad `galleryImages` throws `SyntaxError`, which **is** an `Error`, so
`actionError(error, …)` can still read `.message`.

### 7.2 Subclass for a **kind** the catch can test

```19:44:lib/upload.ts
export class UploadValidationError extends Error {}

export async function saveUploadedFile(
  file: File,
  subpath: string,
): Promise<{ url: string }> {
  if (!ALLOWED_TYPES.includes(file.type)) {
    throw new UploadValidationError(
      "فقط فایل‌های JPEG، PNG و WebP مجاز هستند.",
    );
  }

  if (file.size > MAX_BYTES) {
    throw new UploadValidationError(
      "حجم فایل نمی‌تواند بیشتر از ۵ مگابایت باشد.",
    );
  }
```

Empty class body. It inherits `message` / `name` / `stack`. The **name** of the class is the
discriminant:

```36:40:app/api/upload/image/route.ts
    if (error instanceof UploadValidationError) {
      return NextResponse.json({ error: error.message }, { status: 400 });
    }

    throw error;
```

400 vs 500 without string-matching the Persian text. Disk / `mkdir` failures stay generic `Error`
and rethrow.

`instanceof` walks the prototype chain (prototype note). `UploadValidationError` is an `Error`, so
`error instanceof Error` is also true. Test the **narrower** class first.

### 7.3 Duck typing when it is not an `Error`

web-push failures have `.statusCode` without being `Error` in every path:

```71:75:lib/push.ts
    const statusCode =
      error && typeof error === "object" && "statusCode" in error
        ? Number(error.statusCode)
        : undefined;
```

Next.js navigation errors have `.digest`:

```16:22:lib/action-errors.ts
export function isNextNavigationError(error: unknown) {
  if (typeof error !== "object" || error === null || !("digest" in error)) {
    return false;
  }

  const digest = Reflect.get(error, "digest");
  return typeof digest === "string" && (digest.startsWith("NEXT_REDIRECT") || digest.startsWith("NEXT_NOT_FOUND"));
}
```

Check `typeof === "object"` and not `null` **before** `"digest" in error`. `in` on `null` is a
`TypeError`.

---

## 8. What Studivo does **not** write

| Pattern                                  | Why it is absent                                  |
| ---------------------------------------- | ------------------------------------------------- |
| `throw "string"` / `throw 400`           | Breaks `instanceof Error`                         |
| Empty `catch (e) {}` on reserve/payment  | AGENTS.md: no swallowing booking/payment failures |
| `catch` of `redirect` converted to toast | `isNextNavigationError` + rethrow                 |
| `switch` on seat occupancy               | Arithmetic + ordered predicates → `if` ladder     |
| `try/finally` around every Prisma call   | Transactions already abort on throw               |
| `error.stack` in JSON to the client      | Leak; messages are enough                         |

Client-side expected failures (mismatch password, invalid mobile) use `if` + `return`, not `throw`.

---

## 9. Mental model

1. **Can the operator fix it without I/O?** `if` + `return` (and a field error / toast).
2. **Must a transaction not commit?** `if` + `throw new Error` inside `tx`.
3. **HTTP route choosing a status?** `if` + `return NextResponse` (403/401) or `instanceof` in
   `catch` (400 vs rethrow).
4. **Action / UI boundary?** `try/catch` → `{ success: false }` or toast. Rethrow navigation.
5. **Must run on the way out?** `finally` (loading, `input.value = ""`).
6. **Kind of failure matters?** subclass + `instanceof`, or duck-type `statusCode` / `digest`.
7. **Need a value, not a jump?** ternary / `??` / `||` / `?.`.

`throw` is control flow that **skips** every line until `catch`/`finally`. `return` skips the rest
of **this** function only. Do not mix them up in a `forEach` (loops note: `return` there is
`continue`).

---

## 10. Exercises on this repo

Predict; do not edit production code.

1. In `getSeatStatus`, move `if (membershipStatus === "CANCELLED")` below the `diffDays` checks.
   What status does a cancelled membership with a future `endsAt` get?
2. `getStatusMessage`: delete `case "نیازمند تمدید":` but leave `case "renewal":` without
   `return`/`break`, then `case "expired":`. What string do you send for `"renewal"`?
3. `reserveSeat`: `throw` inside `$transaction` vs `return { success: false }` inside the callback.
   Does the occupancy write commit?
4. `ActionForm`: `redirect()` throws. If you remove `isNextNavigationError` and always toast, what
   does the operator see?
5. `UploadValidationError`: is `error instanceof Error` true in the route catch? Why still test the
   subclass first?
6. Avatar `finally`: `uploadAvatar` throws. Does `avatarInputRef.current.value = ""` still run?
7. Phone OTP `return` after a successful `/request`. Does `finally` run before or after
   `setOtpSent(true)` has taken effect?
8. SMS `catch { throw new Error(SMS_SEND_FAILURE_MESSAGE) }` — is the original `fetch` error
   available to login? Where did it go?
9. `JSON.parse(...)` in settings: malformed gallery string. Which built-in Error? Does
   `instanceof Error` succeed?
10. `new URL(origin)` in the origins filter: `catch { return false }`. What if you omitted `try`?
    What happens to hall lead intake at boot?
11. Push `statusCode === 410`: `return { stale: true }` vs `throw error`. Does `sendPushToMany`’s
    `allSettled` see a rejection?
12. `parseInt(seat.number, 10) || 0` vs `?? 0` for `Number.NaN`. Which sort bucket does a junk
    number land in?

---

## Files cited

| File                                                       | Why it is here                              |
| ---------------------------------------------------------- | ------------------------------------------- |
| `app/(dashboard)/_lib/dashboard-utils.ts`                  | if-ladders, `== null`, `??` / `\|\|` / `?.` |
| `app/(dashboard)/_components/reserve-form.tsx`             | the only app `switch`; fall-through         |
| `app/(dashboard)/_lib/actions/reserve-seat.ts`             | `if` + `throw` inside a transaction         |
| `app/(dashboard)/_lib/actions/manage-seat.ts`              | compound guard then throw                   |
| `app/(dashboard)/settings/actions.ts`                      | OWNER throw; `JSON.parse` try/catch         |
| `app/onboarding/actions.ts`                                | throw in tx, catch at action boundary       |
| `lib/upload.ts`                                            | `class UploadValidationError extends Error` |
| `app/api/upload/image/route.ts`                            | `instanceof` then rethrow                   |
| `lib/sms.ts`                                               | config throw; wrap; empty `catch`           |
| `lib/push.ts`                                              | duck-type `statusCode`; rethrow             |
| `lib/reminders.ts`                                         | catch, `continue`, or `throw error`         |
| `lib/action-errors.ts`                                     | `instanceof Error`; navigation digest       |
| `components/action-form.tsx`                               | rethrow Next navigation                     |
| `app/api/cron/renewal-reminders/route.ts`                  | try/catch → 500 JSON                        |
| `app/(auth)/login/_components/login-form.tsx`              | if-return; try/finally; try/catch/finally   |
| `app/(auth)/login/actions.ts`                              | match wrapped SMS Error                     |
| `app/(dashboard)/profile/_components/profile-settings.tsx` | if-return; finally resets DOM               |
| `hooks/use-marketing-lead-capture.ts`                      | if-return; empty catch                      |
| `lib/marketing-leads/public-api.ts`                        | `new URL` try / empty catch                 |
| `components/ui/sidebar.tsx`                                | programmer-error throw                      |

Series: loops → **this file**. Next JS: **`this` / constructors**, or functions (parameters, default
values, rest). Exceptions and `Error.prototype` also continue the prototype notes.
