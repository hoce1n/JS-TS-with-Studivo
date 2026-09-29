# JavaScript Set and Map — Studivo Walkthrough

`docs/javascript-data-structures-in-studivo.md` was the **map** of layouts.
`docs/javascript-arrays-in-studivo.md` was **indexed** slots. This note is **keyed collections**:
`Set` (unique values) and `Map` (key → value), plus when a plain object is still the right map.

MDN: keys compared with `SameValueZero` (`===`, except `NaN` equals `NaN`). Keys are **not**
stringified. Insert order is kept.

Read this with the cited files open. Predict: _membership test, lookup by id, or a closed `Record`?
Does `.set` copy the value or share the object?_

Related: `docs/javascript-data-structures-in-studivo.md`, `docs/javascript-arrays-in-studivo.md`,
`docs/javascript-objects-in-studivo.md`, `docs/javascript-built-in-objects-in-studivo.md`.

---

## 1. Why not a plain object (or an array)?

| Question                              | Array                    | Object / `Record`                | `Set`              | `Map`                     |
| ------------------------------------- | ------------------------ | -------------------------------- | ------------------ | ------------------------- |
| n-th seat, `.map`                     | yes                      | no                               | no                 | no                        |
| Is this origin allowed?               | `.includes` (tiny lists) | `obj[origin]` (coerces key)      | **`.has`**         | `.has` (ignore the value) |
| Unique seat labels                    | `filter` + seen object   | awkward                          | **`new Set(arr)`** | overkill                  |
| Group seats by membership id          | nested loops             | `obj[id]` if id is a safe string | no                 | **`.get` / `.set`**       |
| Closed enum → Persian label           | no                       | **`Record<LeadStatus, string>`** | no                 | no                        |
| Runtime keys (IP, day key, plan name) | no                       | works until `"__proto__"`        | no                 | **yes**                   |

```javascript
const o = {};
o["__proto__"]; // Object.prototype — inherited
const m = new Map();
m.get("__proto__"); // undefined — no prototype walk
```

Studivo Map keys are ids, IPs, plan names, section ids, error messages, civil dates — all
**strings**. `Map` is still chosen when the key set is **not known at compile time**.
`Record<ExpenseCategory, string>` is chosen when it **is**.

No `WeakMap` / `WeakSet`. Keys here are strings, not objects you want to GC.

---

## 2. `Set` — unique values, membership

A `Set` holds each value at most once. No indexes. `.size` not `.length`. Iteration is insertion
order of **first** add.

### 2.1 Create from a list, then back to an array

```18:20:app/onboarding/actions.ts
function uniqueSeatNumbers(numbers: string[]) {
  return Array.from(new Set(numbers));
}
```

Pipeline (data-structures note):

```text
"1, 1, 2" → split/map/filter → string[] → Set → Array.from → createMany
```

`"1"` twice → one slot. Order of first occurrence kept (`"1"`, `"2"`). `"1"` and `"1 "` are
different if trim did not run first — `normalizeManualSeatNumbers` trims **before** the Set.

`createMany` wants an **array**. The Set is a step, not the persistence shape.

### 2.2 Closed allowlist — `.has`

```3:6:lib/marketing-leads/public-api.ts
const PRODUCTION_ALLOWED_ORIGINS = new Set([
  "https://studivo.ir",
  "https://www.studivo.ir",
]);
```

```47:53:lib/marketing-leads/public-api.ts
export function isAllowedMarketingLeadOrigin(origin: string | null): origin is string {
  if (!origin) return false;

  return (
    PRODUCTION_ALLOWED_ORIGINS.has(origin) ||
    getConfiguredDevelopmentOrigins().includes(origin)
  );
}
```

Production: `Set.has` (exact origin string). Dev: **array** `.includes` on a parsed localhost list.
Same question, two structures — prod list is a constant club; dev list is built each call from env.

`"https://studivo.ir/"` (trailing slash) is a **different** value. CORS `Origin` has no path; a
slash would fail `.has`. Do not `origins.some(o => origin.startsWith(o))`.

Constructor: `new Set(iterable)`. Duplicate literals in the array would collapse; these two URLs are
unique.

### 2.3 `Set` as a uniqueness counter inside a Map value

```450:468:app/(dashboard)/finance/_data/get-finance-dashboard.ts
    const topPlanMap = new Map<
      string,
      { name: string; membershipIds: Set<string>; revenue: number }
    >();
    paymentTrend.forEach((payment) => {
      const planKey = payment.membership.planName;
      const current = topPlanMap.get(planKey) ?? {
        name: payment.membership.planName,
        membershipIds: new Set<string>(),
        revenue: 0,
      };
      current.membershipIds.add(payment.membershipId);
      current.revenue += toDisplayAmount(payment.amount);
      topPlanMap.set(planKey, current);
    });
    const topPlans = Array.from(topPlanMap.values())
      .map((plan) => ({
        name: plan.name,
        count: plan.membershipIds.size,
```

Outer **Map**: plan name → aggregate object. Inner **Set**: which membership ids already paid toward
that plan. `.add` is a no-op if the id is already present. `.size` is distinct members, not payment
rows.

Two payments from the same membership → revenue **adds** twice, count stays `1`. That is the product
meaning of “top plans by distinct memberships.”

### 2.4 Set API used vs unused

| Method               | In Studivo?                                           |
| -------------------- | ----------------------------------------------------- |
| `new Set(iterable)`  | yes                                                   |
| `.add`               | yes (membership ids)                                  |
| `.has`               | yes (CORS)                                            |
| `.size`              | yes (top plans; rate-limit prune gate is Map `.size`) |
| `.delete` / `.clear` | no on Sets                                            |
| `set.forEach`        | no                                                    |
| `Set` iteration      | no — always `Array.from(set)` when a list is needed   |

No `union` / `intersection` / `difference` (newer Set methods). Unique seats are “construct from
array,” not set algebra.

---

## 3. `Map` — key to value, runtime keys

A `Map` is not an object with extra methods. `typeof map === "object"`, `map instanceof Map`. **No**
`map.foo` for keys; only `.get` / `.set` / `.has` / `.delete`.

`.get(missing)` → `undefined` (not `null`, not throw). `.set` returns the Map (chainable; unused).
`.size` is entry count.

### 3.1 Process-local store — get / set / iterate / delete

```22:26:lib/marketing-leads/public-api.ts
const globalForRateLimit = globalThis as GlobalWithMarketingLeadRateLimit;
const rateLimitStore =
  globalForRateLimit.marketingLeadRateLimitStore ?? new Map<string, RateLimitEntry>();

globalForRateLimit.marketingLeadRateLimitStore = rateLimitStore;
```

HMR: reuse the Map on `globalThis` so limits survive Next reloads (same idea as Prisma on `global` —
objects / interop notes).

```73:78:lib/marketing-leads/public-api.ts
function pruneExpiredRateLimitEntries(now: number) {
  if (rateLimitStore.size < 1_000) return;

  for (const [key, entry] of rateLimitStore) {
    if (entry.resetAt <= now) rateLimitStore.delete(key);
  }
}
```

`for…of` on a Map yields `[key, value]` pairs **in insert order**. `.delete` during iteration is
specified-safe for that key. `.size < 1000` skips walking a small map (cheap until it is not).

```91:101:lib/marketing-leads/public-api.ts
  const key = getClientIp(request);
  const existing = rateLimitStore.get(key);

  if (!existing || existing.resetAt <= now) {
    const resetAt = now + RATE_LIMIT_WINDOW_MS;
    rateLimitStore.set(key, { attempts: 1, resetAt });
    return { allowed: true, retryAfterSeconds: 0 };
  }

  existing.attempts += 1;
  rateLimitStore.set(key, existing);
```

Miss or expired window → `.set` a **new** object. Same window → mutate `existing.attempts` (same
reference the Map already holds) then `.set` again (redundant for identity, documents “write”). Next
`.get` would see `attempts` even without the second `.set` because the value is a shared object
(objects note).

Comment in file: not a distributed limiter. Two Node processes would each have a Map. Structure idea
stays (key = IP); implementation would move to Redis.

### 3.2 Index over an array — group by id

```157:163:app/(dashboard)/_lib/dashboard-utils.ts
  const activeAssignmentsByMembership = new Map<string, string[]>();
  seats.forEach((seat) => {
    const active = getActiveAssignment(seat.assignments);
    if (active?.membership.id) {
      const existing = activeAssignmentsByMembership.get(active.membership.id) || [];
      activeAssignmentsByMembership.set(active.membership.id, [...existing, seat.number]);
    }
  });
```

Same pattern in `study-hall-seats-map.tsx`. **Array is source of truth; Map is the index.**

Copy-on-write `[...existing, seat.number]` so two memberships never share one array. `|| []` on
miss: `.get` is `undefined`, falsy, new empty list (do not mutate a cached `[]` — there is none
yet).

Then occupancy:

```171:180:app/(dashboard)/_lib/dashboard-utils.ts
    const isDuplicate =
      membership?.id && (activeAssignmentsByMembership.get(membership.id)?.length ?? 0) > 1;
    // ...
      duplicateSeats: isDuplicate ? activeAssignmentsByMembership.get(membership.id!) : undefined,
```

`.get` twice is cheap. `?.length ?? 0` if the id was never indexed.

### 3.3 `reduce` into a Map (mutate the accumulator)

```225:236:app/(dashboard)/_components/study-hall-seats-map.tsx
  const groupedSeats = visibleSeats.reduce((groups, seat) => {
    const key = seat.sectionId ?? "unassigned";
    const current = groups.get(key);
    if (current) current.seats.push(seat);
    else {
      groups.set(key, {
        name: seat.sectionId ? seat.sectionName : "صندلی‌های عمومی",
        seats: [seat],
      });
    }
    return groups;
  }, new Map<string, { name: string; seats: ReserveFormSeat[] }>());
```

Initializer is a **new Map**. Hit: `current.seats.push` — mutate the array **inside** the Map value
(local grouping, not React state). Miss: `.set` a new `{ name, seats: [seat] }`.
`sectionId ?? "unassigned"` so `null` does not become the string `"null"` and does not collide with
a real id.

Render: keyed → indexed

```313:313:app/(dashboard)/_components/study-hall-seats-map.tsx
            {[...groupedSeats.entries()].map(([sectionId, group]) => {
```

`[...map.entries()]` is an array of pairs. React needs an array. Insert order = first time that
section appeared in `visibleSeats` (already sorted).

### 3.4 Seed a Map from pairs, then update copies

```406:420:app/(dashboard)/finance/_data/get-finance-dashboard.ts
    const trendMap = new Map<string, Omit<ChartData, "date">>(
      getFinanceRangeDayKeys(range).map((date) => [
        date,
        { cashCollected: 0, recordedExpenses: 0 },
      ]),
    );
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

`new Map(iterableOfPairs)` fills every day in the range with zeros so the chart has **dense**
x-axis. Payment on a day outside the range: `.get` miss → `return` (drop). Update replaces the value
object (`{ ...current, cashCollected }`) instead of mutating — either is fine here; replace matches
“immutable entry” style.

`expenseCategoryMap` is the sparse variant: no seed, `.get ?? 0` then `.set`.

```438:444:app/(dashboard)/finance/_data/get-finance-dashboard.ts
    const expenseCategoryMap = new Map<string, number>();
    expenseTrend.forEach((expense) => {
      expenseCategoryMap.set(
        expense.category,
        (expenseCategoryMap.get(expense.category) ?? 0) +
          toDisplayAmount(expense.amount),
      );
    });
```

Values are **primitives** (`number`). No shared-object bug. `.set` must run every time because
`number` is copied by value (primitives note). Contrast rate-limit `attempts += 1` on an object.

### 3.5 Deduplicate by key, last write wins

```193:195:components/ui/field.tsx
    const uniqueErrors = [
      ...new Map(errors.map((error) => [error?.message, error])).values(),
    ]
```

`errors.map` → `[message, error]` pairs. `new Map(pairs)` — same message overwrites. `.values()` —
Map iterator of **error objects**. Spread → array for React. Two errors with `message: undefined`
collapse to **one** (one `undefined` key).

```153:159:app/platform/_components/leads-table.tsx
        : Array.from(
            new Map(
              leads.flatMap((lead) =>
                lead.owner ? [[lead.owner.id, lead.owner.name] as const] : []
              )
            ).entries()
          ).map(([id, name]) => ({ id, name })),
```

Key = owner id. Value = name string. Duplicate id: **last** lead in `leads` wins the name.
`Array.from(map.entries())` same as `[...map.entries()]`.

```395:400:app/(dashboard)/_components/reserve-form.tsx
  const swapSections = React.useMemo(() => {
    const map = new Map<string, string>();
    for (const option of availableSeats) {
      map.set(option.sectionId, option.sectionName);
    }
    return [...map.entries()].map(([id, name]) => ({ id, name }));
  }, [availableSeats]);
```

Loop `.set` instead of `new Map(pairs)`. First section id keeps first name unless a later seat
overwrites (same id should mean same name). `useMemo` rebuilds when `availableSeats` changes — Map
is ephemeral, not state.

### 3.6 Map API used vs unused

| Method                         | In Studivo?                            |
| ------------------------------ | -------------------------------------- |
| `new Map()` / `new Map(pairs)` | yes                                    |
| `.get` `.set`                  | yes                                    |
| `.delete`                      | yes (rate-limit prune)                 |
| `.size`                        | yes (prune threshold)                  |
| `.has`                         | no — miss is `.get` then `undefined`   |
| `.clear`                       | no                                     |
| `.keys()`                      | no                                     |
| `.values()`                    | yes (field errors, top plans)          |
| `.entries()` / `for…of map`    | yes                                    |
| `map.forEach`                  | no (arrays `forEach` to **fill** maps) |

`.has` would distinguish “key present, value `undefined`” from miss. Values here are objects or
numbers, never stored `undefined`, so `.get` is enough.

---

## 4. Iteration: Map/Set → array at the edge

UI, Prisma `createMany`, Recharts, `<select>` want **arrays**.

| Need                       | Code                                                                       |
| -------------------------- | -------------------------------------------------------------------------- |
| Pairs → objects            | `Array.from(map.entries()).map(([k, v]) => …)` or `[...map.entries()].map` |
| Values only                | `Array.from(map.values())` / `[...map.values()]`                           |
| Unique list                | `Array.from(set)`                                                          |
| Chart days in insert order | `Array.from(trendMap.entries())` after seeding from `dayKeys[]`            |

`JSON.stringify(map)` is `{}`. `JSON.stringify(set)` is `{}`. Persist via `Object.fromEntries(map)`
or `Array.from(set)` first (structured-data note).

---

## 5. Plain object as the keyed collection

```3:16:lib/expense-categories.ts
export const EXPENSE_CATEGORY_LABELS: Record<ExpenseCategory, string> = {
  RENT: "اجاره",
  // ...
};

export const EXPENSE_CATEGORY_OPTIONS: { value: ExpenseCategory; label: string }[] =
  (Object.entries(EXPENSE_CATEGORY_LABELS) as [ExpenseCategory, string][]).map(
    ([value, label]) => ({ value, label }),
  );
```

Closed Prisma enum → object. `Object.entries` is the analog of `map.entries()`. `STATUS_LABELS`,
`activityGroupLabels` — same.

Do **not** replace these with `Map`. TS `Record<Enum, string>` forces every key at compile time. A
`Map` would not.

`Object.hasOwn(activityGroupLabels, group)` — own keys only (built-ins note). `Map` has no
prototype-key problem; objects do.

---

## 6. Equality (`SameValueZero`) and gotchas

1. **Strings must match exactly.** `"https://studivo.ir"` ≠ `"https://studivo.ir/"`. IPs from
   `x-forwarded-for` are trimmed; still `"unknown"` if missing (one shared bucket).

2. **Objects as Set values / Map keys would be by identity.** Studivo never does `set.add({ id })` —
   two `{ id: "x" }` would both stay. Ids are strings.

3. **`NaN`:** `new Set([NaN, NaN]).size === 1`. Unused here.

4. **Shared value objects.** Rate-limit entry and `topPlanMap` aggregates are mutated in place.
   Finance trend replaces `{ ...current }`. Duplicate-seat lists copy with spread. Pick one style
   and do not `push` onto an array you still consider immutable React state.

5. **`.get` || `[]`.** If you ever stored a real empty array and it was falsy… empty arrays are
   truthy. `|| []` only fires on `undefined` / `null` / `0` / `""`. Fine for `.get` miss.

6. **Last-write-wins Map from pairs.** Owner names, error messages — later duplicate keys replace.
   First-write-wins would need `if (!map.has(k)) map.set(k, v)`.

7. **Nested Set in Map** (`membershipIds`): `.add` then `.set(planKey, current)` — the Set is the
   same object; `.set` is optional after the first insert of `current`. They still `.set` every
   payment for clarity.

8. **Do not use Map for FormData.** `formData.get` is a different keyed host collection (casting /
   data-structures notes).

---

## 7. Mental model

1. **Set** — “have I seen this value?” Unique seats, CORS club, distinct membership ids.
2. **Map** — “given this runtime key, what bag/count/list?” IP, membership id, section id, day, plan
   name, error message.
3. **Object / `Record`** — “given this **compile-time** enum, what label?”
4. Convert to **Array** at UI/DB/JSON edges. Never stringify Map/Set directly.
5. Values that are objects are **references**. Mutate only when you own the Map for this function.

---

## 8. Exercises on this repo

Predict; do not edit production code.

1. `uniqueSeatNumbers(["2", "1", "2", "1"])` — `Array.from` order?
2. `PRODUCTION_ALLOWED_ORIGINS.has("https://www.studivo.ir")` vs `"https://studivo.ir:443"`.
3. Rate-limit: skip `rateLimitStore.set(key, existing)` after `attempts += 1`. Does the next request
   see `2`? Why is `.set` still written?
4. `existing.attempts += 1` if the value were a primitive number stored in the Map — would in-place
   mutate work?
5. Duplicate seats Map: `.set(id, existing.push(seat.number) && existing)` — what goes wrong?
   (`push` returns a number.)
6. `groupedSeats` `current.seats.push(seat)` vs `[...current.seats, seat]`. Mutation OK here? Would
   it be OK in `useState`?
7. `new Map(errors.map(e => [e?.message, e]))` — two errors, both `message` undefined.
   `uniqueErrors.length`?
8. `topPlanMap` Set `.add` vs using an array + `.includes` for membership ids — correctness vs cost.
9. `JSON.stringify(rateLimitStore)` — result? How would you dump it for a debug log?
10. Why `EXPENSE_CATEGORY_LABELS` is not a `Map<ExpenseCategory, string>`.

---

## Files cited

| File                                                     | Set / Map role                                                |
| -------------------------------------------------------- | ------------------------------------------------------------- |
| `app/onboarding/actions.ts`                              | `Set` unique seats → `Array.from`                             |
| `lib/marketing-leads/public-api.ts`                      | `Set.has` CORS; `Map` rate limit get/set/delete/size/`for…of` |
| `app/(dashboard)/_lib/dashboard-utils.ts`                | `Map` membership id → seat numbers                            |
| `app/(dashboard)/_components/study-hall-seats-map.tsx`   | same index; `reduce` into Map; `[...entries()]`               |
| `app/(dashboard)/_components/reserve-form.tsx`           | `Map` section id → name; spread entries                       |
| `app/(dashboard)/finance/_data/get-finance-dashboard.ts` | seeded trend Map; category Map; Map of `{ Set, number }`      |
| `app/platform/_components/leads-table.tsx`               | `new Map(pairs)` last-write owner                             |
| `components/ui/field.tsx`                                | `new Map(pairs)` dedupe errors; `.values()`                   |
| `lib/expense-categories.ts`                              | object `Record` instead of Map                                |

Series: data-structures overview → arrays → **this file**. Next JS: **`this` / constructors**, or
JSON as its own drill. Next TS: unions and narrowing, generics, or `typeof` / `keyof`.
