# JavaScript Loops and Iteration — Studivo Walkthrough

Arrays and Map/Set were **the collections**. Equality was **when two values match**. This note is
**walking those collections** — when Studivo uses a loop, when it uses `.map` / `.filter`, and what
it never writes (`while`, `for…in`, `do`).

MDN’s split:

```text
Loops and iteration
├── for          index + condition + step
├── for…of       values of an iterable
├── for…in       enumerable keys of an object  (not in this repo)
├── while / do   condition-first / body-first  (not in this repo)
└── Array extras forEach / map / filter / reduce  (arrays note)
```

Read this with the cited files open. Predict: _does this produce a new array, mutate an accumulator,
`await` in order, or exit early?_

Related: `docs/javascript-arrays-in-studivo.md`, `docs/javascript-set-and-map-in-studivo.md`,
`docs/javascript-data-structures-in-studivo.md`, `docs/javascript-scope-in-studivo.md`.

---

## 1. The rule that pays rent

| Need                                                   | Construct                             | Studivo                                     |
| ------------------------------------------------------ | ------------------------------------- | ------------------------------------------- |
| Transform list → list                                  | `.map` / `.filter`                    | seats view, gallery, Zod issues             |
| Side effect / accumulate                               | `for…of` or `.forEach`                | Map index, revenue, rate-limit prune        |
| `await` **in order** (one row depends on the previous) | `for…of` + `await`                    | onboarding sections, reminder notifications |
| `await` **in parallel**                                | `.map` + `Promise.all` / `allSettled` | gallery uploads, push batch                 |
| Need the **index** or a **byte array**                 | C-style `for (let i = 0; …)`          | VAPID decode; timezone offset twice         |
| Walk until a civil date                                | C-style `for` with date as “index”    | finance day keys                            |
| Keys of a plain object                                 | `Object.entries` / `.keys`            | not `for…in`                                |
| Infinite / unknown bound                               | `while`                               | **unused** — ranges are finite              |

`for…of` is the default loop. `forEach` is the default **array** side-effect helper. `.map` is not a
loop you pick for “do something”; it is for **new arrays**.

---

## 2. `for…of` — iterate values (house loop)

`for (const x of iterable)` takes **values** from anything with `[Symbol.iterator]`: arrays, `Map`,
`Set`, `String`, `results` from `Promise.allSettled`. Binding is `const` per iteration (scope note —
a new binding each round).

### 2.1 Array of work, sequential `await`

```64:90:app/onboarding/actions.ts
      for (const sectionInput of sectionInputs) {
        const section = await tx.section.create({
          data: {
            studyHallId: studyHall.id,
            name: sectionInput.name,
            isActive: true,
          },
          select: { id: true },
        });

        const seatNumbers =
          sectionInput.mode === "AUTO"
            ? Array.from({ length: sectionInput.seatCount }, (_, index) => String(index + 1))
            : uniqueSeatNumbers(normalizeManualSeatNumbers(sectionInput.manualSeats));
        // ...
        await tx.seat.createMany({
          data: seatNumbers.map((number) => ({
```

Each section must exist before its seats (`section.id`). **Order matters.**
`sectionInputs.map(async …)` would start all creates without waiting unless you `await` a
`Promise.all` — and even then the transaction still needs one section id at a time. Inner
`seatNumbers.map` is **not** a loop; it builds the `createMany` payload (arrays note).

Reminder fan-out is the same shape, nested:

```185:232:lib/reminders.ts
  for (const candidate of candidates) {
    const recipients = await prisma.staffAssignment.findMany({
      // ...
    });
    // ...
    for (const recipient of recipients) {
      try {
        await prisma.notification.create({
          // ...
        });
      } catch (error) {
        if (isNotificationClaimConflict(error)) {
          counters.duplicatesSkipped += 1;
          continue;
        }

        throw error;
      }
```

Outer: each membership candidate. Inner: each staff user. `continue` skips **this recipient** after
a duplicate notification, not the whole candidate list. Unrelated errors `throw` — they leave
**both** loops.

```240:241:lib/reminders.ts
      if (recipient.user.pushSubscriptions.length === 0) {
        continue;
```

Second `continue`: in-app notification already created; no push devices → next recipient.

### 2.2 `continue` vs `break`

`continue` — skip the rest of **this** iteration. Studivo uses it.

`break` — leave the loop entirely. **No `break` in app `.ts`.** The service worker uses `return`
inside `.then` instead (early exit from the callback, which ends the walk):

```104:110:public/sw.js
      for (const client of windowClients) {
        if (client.url.includes(targetUrl) && "focus" in client) {
          return client.focus();
        }
      }

      return clients.openWindow(targetUrl);
```

First matching window: `return` from the `.then` callback (not `break` then open). No match: fall
through to `openWindow`. That is “search then default,” the job `break` + flag would do, done with
`return` because the loop lives in a Promise callback.

### 2.3 Classify results — no await

```100:111:lib/push.ts
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

`Promise.allSettled` already ran in parallel (`.map` of sends). This loop only **counts**.
`continue` after fulfilled skips the `failed += 1` path. Could be `else { failed += 1 }`; `continue`
keeps the rejected branch visually at the bottom.

### 2.4 Fill a Map / a record

```395:400:app/(dashboard)/_components/reserve-form.tsx
  const swapSections = React.useMemo(() => {
    const map = new Map<string, string>();
    for (const option of availableSeats) {
      map.set(option.sectionId, option.sectionName);
    }
    return [...map.entries()].map(([id, name]) => ({ id, name }));
```

Side effect on a local Map, then convert to an array. `forEach` would also work; `for…of` is the
same walk without a callback.

```123:134:app/(dashboard)/staff/_data/get-staff-data.ts
  for (const id of staffAssignmentIds) {
    hoursByAssignment[id] = 0;
  }

  for (const shift of shifts) {
    const startMs = Math.max(shift.startsAt.getTime(), rangeStartMs);
    const endMs = Math.min(shift.endsAt.getTime(), rangeEndMs);
    if (endMs <= startMs) continue;
    hoursByAssignment[shift.staffAssignmentId] =
      (hoursByAssignment[shift.staffAssignmentId] ?? 0) +
      (endMs - startMs) / (1000 * 60 * 60);
  }
```

First loop: seed zeros so every id appears. Second: add hours; `continue` when the clipped interval
is empty (`<=` — equality note). Two simple passes beat one clever `reduce`.

### 2.5 `for…of` on a Map

```76:78:lib/marketing-leads/public-api.ts
  for (const [key, entry] of rateLimitStore) {
    if (entry.resetAt <= now) rateLimitStore.delete(key);
  }
```

Maps are iterable as `[key, value]` pairs (Set/Map note). Destructure in the `const`. `.delete`
during iteration is allowed for that key. This is **not** `for…in` on the Map object (that would
walk the Map’s **method names** if you tried — another reason `for…in` is banned).

---

## 3. C-style `for` — index, fixed count, or a sliding date

```javascript
for (initialization; condition; afterthought) {
  statement;
}
```

Three slots, all optional. Studivo always fills all three. Uses `let` (the index **changes**). Never
`var`.

### 3.1 Byte index — VAPID key

```98:103:hooks/usePWA.ts
  const rawData = window.atob(base64);
  const outputArray = new Uint8Array(rawData.length);

  for (let i = 0; i < rawData.length; ++i) {
    outputArray[i] = rawData.charCodeAt(i);
  }
```

Same loop in `PushNotificationManager.tsx`. Need `i` to write `outputArray[i]`. `for…of` on a string
would give **code units as strings**, not indexes into a `Uint8Array`. `++i` vs `i += 1` — same for
numbers here. This is the rare TypedArray exception (data-structures note said app domain does not
use them; PWA crypto keys do).

### 3.2 Run **exactly twice** (offset convergence)

```123:125:lib/finance-range.ts
  for (let index = 0; index < 2; index += 1) {
    instant = utcMidnight - getTimeZoneOffsetMs(new Date(instant));
  }
```

Not iterating a collection. The collection is “two passes.” A DST/offset boundary can depend on the
instant you ask about, so they apply the offset, then apply it again. Unrolling
`instant = …; instant = …;` would be two copies of the same line; the loop is the documentation:
**twice**.

`log-date-range.ts` repeats the pattern on one line. Same algorithm.

### 3.3 Iterator that is not an integer — civil days

```257:264:lib/finance-range.ts
  const dayKeys: string[] = [];
  for (
    let currentDay = startDay;
    compareCivilDates(currentDay, endDay) <= 0;
    currentDay = addCivilDays(currentDay, 1)
  ) {
    dayKeys.push(formatTehranDateKey(toTehranBoundary(currentDay)));
  }
```

Initialization: `{ year, month, day }`. Condition: still on or before `endDay`. Afterthought: add
one civil day (not `i++` — months are not 32 days). Body: `push` onto a **fresh** array (arrays
note: `push` OK when you own the list).

`while (compare(...) <= 0) { push; currentDay = add... }` would work. The `for` header keeps the
three moving parts in one place. No `while` in the repo; this is the substitute.

If `startDay > endDay`, condition fails immediately → `[]`.

---

## 4. What Studivo does **not** write

| Loop                         | Why it is absent                                                                                                                                                 |
| ---------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `while` / `do…while`         | Bounds are known (N seats, 2 passes, date range, list length). `while (true)` is a hang risk.                                                                    |
| `for…in`                     | Walks **enumerable keys**, including inherited if you are sloppy. Arrays: `"0"`, `"1"`, `"length"` surprises. Objects: use `Object.keys` / `entries` / `hasOwn`. |
| `for await…of`               | No async iterables (no streaming DB cursor in app code). Sequential await is `for…of` + `await`.                                                                 |
| Labeled `break` / `continue` | Nested reminder loops use inner `continue` only.                                                                                                                 |
| `var i` in a `for`           | TDZ / function-scope bugs (hoisting note). Always `let` / `const`.                                                                                               |

React lists are **`.map` in JSX**, not `for` pushing to an array of elements.

---

## 5. `forEach` and the array extras (iteration without `for`)

Covered as methods in the arrays note. Here they are **iteration style**.

**`forEach` — visit every element, return `undefined`, cannot `break` / `continue` / `await`
usefully** (the callback can `return` to skip the rest of **that** call, like `continue`).

```158:164:app/(dashboard)/_lib/dashboard-utils.ts
  seats.forEach((seat) => {
    const active = getActiveAssignment(seat.assignments);
    if (active?.membership.id) {
      const existing = activeAssignmentsByMembership.get(active.membership.id) || [];
      activeAssignmentsByMembership.set(active.membership.id, [...existing, seat.number]);
    }
  });
```

```207:213:app/(dashboard)/_lib/dashboard-utils.ts
  seatView.forEach((s) => {
    if (s.membership) {
      const price = Number(s.membership.planPrice) || 0;
      if (s.status === "reserved") activeRevenue += price;
      if (s.status === "renewal") atRiskRevenue += price;
    }
  });
```

```412:420:app/(dashboard)/finance/_data/get-finance-dashboard.ts
    paymentTrend.forEach((payment) => {
      const date = formatTehranDateKey(payment.paidAt);
      const current = trendMap.get(date);
      if (!current) return;

      trendMap.set(date, {
        ...current,
        cashCollected: current.cashCollected + toDisplayAmount(payment.amount),
      });
    });
```

`return` inside `forEach` = `continue`, **not** `break`. Early-exit of the whole list needs
`for…of` + `break`/`return`. Occupancy and finance never stop early; they visit all.

**`map` / `filter` / `reduce`** — iteration that **returns a value**. Prefer them when the output
**is** the list (dashboard seat DTO, available seats, `createMany` rows). Prefer `forEach` /
`for…of` when the output is a Map, a number, or a void side effect.

**`Object.entries(…).forEach`** — iterate a plain object without `for…in`:

```40:40:app/(dashboard)/logs/_components/log-filters.tsx
    Object.entries(updates).forEach(([key, value]) => value ? params.set(key, value) : params.delete(key));
```

---

## 6. Parallel vs sequential (the await trap)

```javascript
// WRONG — forEach does not wait
records.forEach(async (r) => {
  await send(r);
});

// Sequential — onboarding sections, reminder creates
for (const r of records) {
  await send(r);
}

// Parallel — independent I/O
await Promise.all(records.map((r) => send(r)));
await Promise.allSettled(records.map((r) => send(r)));
```

Studivo:

- **Sequential:** `submitOnboarding` sections; `sendRenewalReminders` candidates/recipients
  (dedupe + create then push).
- **Parallel:** `sendPushToMany` (`allSettled` so one dead endpoint does not abort the batch);
  gallery `Promise.all(files.map(uploadImage))`.

`forEach` + `async` looks sequential and is not. The reminder code uses `for…of` **because**
`await prisma.notification.create` must finish (and maybe `continue`) before push.

---

## 7. Iteration protocol (why `for…of` works)

`for…of` calls `iterable[Symbol.iterator]()`. Arrays, Map, Set, and `allSettled` results implement
it. Plain objects do **not**:

```javascript
for (const x of { a: 1 }) {
} // TypeError at runtime
```

That is why labels use `Object.entries` and why `toAuditMetadata` rejects arrays but does not
`for…of` an unknown object.

`Array.from({ length: n }, fn)` is iteration by **index 0…n-1** without writing `for`. Factory, not
a side-effect loop.

Spread `[...map.entries()]` is also iteration (consumes the iterator into an array).

---

## 8. Closures: `let` vs `var` in loops

```javascript
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0); // 3, 3, 3
}
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0); // 0, 1, 2
}
```

`let` is per-iteration binding. Studivo’s `for (let i = 0; i < rawData.length; ++i)` does not close
over `i` asynchronously; the rule still holds: never `var`. `for (const section of …)` is already a
fresh `const` each time — safe to `await` and capture `sectionInput.name` in an Error.

---

## 9. Nested loops and complexity

Reminder: `candidates × recipients` Prisma creates. Hall scale (tens of memberships × few staff) is
fine. A nested `for` over every seat × every assignment would be the wrong occupancy algorithm —
they **index with a Map** in one `forEach`, then `.map` (arrays / Set-Map notes).

If you need “does any assignment occupy,” that is `.some`, not a nested loop with a flag (arrays
note). `.some` **is** a loop with an internal `break`.

---

## 10. Mental model

1. **Output is a list?** `.map` / `.filter` / `Array.from`.
2. **Output is a total, Map, or void, visit all?** `forEach` or `for…of`.
3. **Need `await` in order or `continue`?** `for…of`.
4. **Need `i` or a non-list counter?** C-style `for`.
5. **Never** `for…in`, `while (true)`, `forEach(async …)` without `Promise.all`.
6. `return` in `forEach` = `continue`. `return` in `for…of` inside a function leaves the
   **function**. `break` leaves the **loop** (unused here).

---

## 11. Exercises on this repo

Predict; do not edit production code.

1. Replace onboarding `for…of` with
   `sectionInputs.forEach(async (sectionInput) => { await tx.section.create… })`. Do seats still
   attach to the right section? Does the transaction wait?
2. `continue` after duplicate notification — does the outer `for (const candidate …)` skip remaining
   recipients or remaining candidates?
3. `sendPushToMany`: swap `continue` for `else`. Same counts?
4. `for (const client of windowClients)` + `return client.focus()` — if you used `break` instead of
   `return`, would `openWindow` still run?
5. Finance `for (let currentDay = startDay; compare <= 0; add 1)` when start equals end — how many
   `push`?
6. VAPID `++i` vs `for…of rawData` — what type is each character? Can you assign it to
   `Uint8Array[i]` without `charCodeAt`?
7. `hoursByAssignment` first loop vs `Object.fromEntries(ids.map(id => [id, 0]))` — same structure?
   Why two `for…of`?
8. `paymentTrend.forEach` with `return` when `!current` — is that `break` or `continue`?
9. Why prune uses `for…of rateLimitStore` and not `rateLimitStore.forEach` if you want `.delete` —
   does `forEach` allow delete? (It does.) Why `for…of` anyway?
10. Write a `while` that matches `index < 2` offset loop. Why did they still choose `for`?

---

## Files cited

| File                                                             | Loop                                        |
| ---------------------------------------------------------------- | ------------------------------------------- |
| `app/onboarding/actions.ts`                                      | `for…of` + sequential `await`; inner `.map` |
| `lib/reminders.ts`                                               | nested `for…of`; `continue` twice           |
| `lib/push.ts`                                                    | `for…of` settled results; `continue`        |
| `lib/marketing-leads/public-api.ts`                              | `for…of` Map; `.delete`                     |
| `lib/finance-range.ts`                                           | `for` twice; `for` civil days + `push`      |
| `hooks/usePWA.ts` / `components/pwa/PushNotificationManager.tsx` | C-style index into `Uint8Array`             |
| `app/(dashboard)/staff/_data/get-staff-data.ts`                  | two `for…of`; `continue` on empty interval  |
| `app/(dashboard)/_components/reserve-form.tsx`                   | `for…of` fill Map                           |
| `app/(dashboard)/_lib/dashboard-utils.ts`                        | `forEach` side effects; `.map` transform    |
| `app/(dashboard)/finance/_data/get-finance-dashboard.ts`         | `forEach` + `return` as continue            |
| `app/(dashboard)/logs/_components/log-filters.tsx`               | `Object.entries.forEach`                    |
| `public/sw.js`                                                   | `for…of` + `return` from callback           |

Series: collections → equality → **this file**. Next JS: **`this` / constructors**. Next TS: unions
and narrowing, generics, or `typeof` / `keyof`.
