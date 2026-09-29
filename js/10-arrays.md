# JavaScript Arrays — Studivo Walkthrough

`docs/javascript-data-structures-in-studivo.md` was the **map**: indexed vs keyed vs structured.
This note is the **territory** — arrays as objects, index/`length`, create, mutate vs copy, search,
iterate, sort, spread, nested lists, and the mistakes that show up at a front desk.

Set and Map stay in the overview until their own file. Here the question is only: **ordered slots
`0 … length-1`.**

Read this with the cited files open. Predict: _new array or the same one? Did `length` change? Are
the elements shared?_

Related: `docs/javascript-data-structures-in-studivo.md`, `docs/javascript-objects-in-studivo.md`,
`docs/javascript-built-in-objects-in-studivo.md`, `docs/javascript-type-casting-in-studivo.md`.

---

## 1. What an array is (and why it is an object)

```javascript
typeof [1, 2]; // "object"
Array.isArray([1, 2]); // true
Array.isArray({ 0: 1, length: 1 }); // false
```

An array is an object whose **indexed** properties (`"0"`, `"1"`, …) and special `length` are
maintained by the engine. It still has `Array.prototype` (`map`, `filter`, … — prototype note).

```34:36:app/(dashboard)/logs/_lib/activity-normalizer.ts
export function toAuditMetadata(value: unknown): AuditMetadata {
  if (!value || typeof value !== "object" || Array.isArray(value)) return {};
  return value as AuditMetadata;
}
```

JSON `typeof` is `"object"` for both `{}` and `[]`. Metadata must be a **keyed** bag, not a list.
`Array.isArray` is the test. `instanceof Array` can lie across iframes; Studivo uses `isArray`.

Seat rows, payment lists, onboarding sections, gallery URLs — **indexed collections**. CORS origins
and rate-limit IPs are not (data-structures note).

---

## 2. Index and `length`

Indexes are **zero-based** integers. `seats[0]` is the first hall seat in that array, not seat label
`"1"`. `seat.number` is a string identifier (primitives note). Do not confuse **position in the
array** with **printed seat number**.

`length` is `lastIndex + 1` for dense arrays. Empty: `length === 0`.

```79:81:app/onboarding/actions.ts
        if (seatNumbers.length === 0) {
          throw new Error(`برای بخش «${sectionInput.name}» حداقل یک صندلی تعریف کنید.`);
        }
```

```175:177:app/(dashboard)/settings/_components/public-page-settings-form.tsx
    if (galleryImages.length + files.length > 8) {
      setGalleryError("حداکثر ۸ تصویر در گالری مجاز است.");
```

`length` is a number you **read**. Assigning `arr.length = 0` would empty an array (mutate). Studivo
does not do that; it replaces the array (`setGalleryImages(next)`).

Holes: `new Array(3)` is `[empty × 3]`, `length === 3`, `.map` **skips** holes.
`Array.from({ length: 3 }, …)` fills them. Studivo only uses the second form.

`arr[arr.length - 1]` is the last element. The repo uses `seatAssignments[0]` as “first / latest
known,” not `.at(-1)` (`.at` unused).

```115:118:app/(dashboard)/_lib/actions/renew-membership.ts
      // Fallback: fixed-seat memberships should keep the latest known seat
      const seatToKeep =
        occupyingSeat ??
        (current.hasFixedSeat ? current.seatAssignments[0] : undefined);
```

That `[0]` is only safe because the Prisma query ordered assignments. Index `0` means **first in
this array**, not “seat number 0.”

---

## 3. Creating arrays

### 3.1 Literal `[]` — the default

```39:40:app/onboarding/_components/onboarding-wizard.tsx
  sections: [DEFAULT_SECTION],
  plans: [DEFAULT_PLAN],
```

```83:83:app/(dashboard)/logs/_lib/activity-normalizer.ts
  const secondaryFacts: string[] = [];
```

```257:257:lib/finance-range.ts
  const dayKeys: string[] = [];
```

Start empty, then `push`. Or start with one element. Literals are dense, no holes.

`[values.sections[0] ?? DEFAULT_SECTION]` wraps a **single object** in a new one-element array
(wizard `visibleSections` when the hall has no sections UI).

### 3.2 `Array()` / `new Array()` — **not in this repo**

`new Array(3)` → holes. `Array(1, 2, 3)` → `[1, 2, 3]` (different overload). Easy to confuse. House
style: `[]` or `Array.from`.

### 3.3 `Array.from()` — array-like, iterables, indexed factory

**From `{ length }` + mapFn** (onboarding AUTO seats, settings bulk add):

```74:76:app/onboarding/actions.ts
        const seatNumbers =
          sectionInput.mode === "AUTO"
            ? Array.from({ length: sectionInput.seatCount }, (_, index) => String(index + 1))
```

```312:316:app/(dashboard)/settings/actions.ts
        : Array.from(
            { length: parsed.data.count ?? 0 },
            (_, index) =>
              `${parsed.data.prefix ?? ""}${(parsed.data.start ?? 1) + index}`,
          );
```

`{ length: n }` is **array-like**, not an array. `from` allocates `n` slots and calls the function
with `(undefined, index)`. Index `0` → seat `"1"` or `"A1"` with prefix.

**From a `Set` (iterable) → unique list:**

```18:20:app/onboarding/actions.ts
function uniqueSeatNumbers(numbers: string[]) {
  return Array.from(new Set(numbers));
}
```

**From `FileList` (array-like host object):**

```173:173:app/(dashboard)/settings/_components/public-page-settings-form.tsx
    const files = Array.from(e.target.files ?? []);
```

```508:508:app/(dashboard)/settings/_components/settings-tabs.tsx
    const selected = Array.from(files ?? []);
```

`<input type="file" multiple>` gives `FileList`: has `length` and `[0]`, but no `.map`. Always
`Array.from` before mapping uploads.

**From `Map` values / entries** (still array at the UI edge):

```465:472:app/(dashboard)/finance/_data/get-finance-dashboard.ts
    const topPlans = Array.from(topPlanMap.values())
      .map((plan) => ({
        name: plan.name,
        count: plan.membershipIds.size,
        revenue: plan.revenue,
      }))
      .sort((left, right) => right.revenue - left.revenue)
      .slice(0, 5);
```

### 3.4 `Array.of()` — **not in this repo**

`Array.of(3)` is `[3]`; `Array(3)` is holes. We write `[3]` or
`[parsed.data.number?.trim()].filter(Boolean)` instead.

```309:311:app/(dashboard)/settings/actions.ts
    const numbers =
      parsed.data.mode === "single"
        ? ([parsed.data.number?.trim()].filter(Boolean) as string[])
```

One optional string, wrapped in a literal, empty string dropped. That is `Array.of`-shaped without
the name.

---

## 4. Adding and removing

### 4.1 What Studivo actually mutates: `push`

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

Local accumulator, not React state. `push` **mutates** and returns the new `length` (ignored). Fine
when the array was born on the previous line.

```83:87:app/(dashboard)/logs/_lib/activity-normalizer.ts
  const secondaryFacts: string[] = [];
  // ...
    if (money) secondaryFacts.push(`مبلغ: ${money}`);
```

Same: build a list, return it. Nobody else holds `secondaryFacts` yet.

### 4.2 `pop` / `shift` / `unshift` / `splice` — **not in this repo**

| Method              | Mutates | Typical use                    | House alternative                 |
| ------------------- | ------- | ------------------------------ | --------------------------------- |
| `push`              | yes     | append to a **new** local list | used                              |
| `pop`               | yes     | remove last                    | unused — do not pop from state    |
| `shift` / `unshift` | yes     | queue at front (O(n))          | unused                            |
| `splice`            | yes     | delete/insert in the middle    | `filter` / `map` / spread (below) |

React state and server snapshots are **replaced**, not spliced.

### 4.3 Non-mutating add / remove (the onboarding / gallery pattern)

**Append:** spread, new array.

```72:79:app/onboarding/_components/onboarding-wizard.tsx
  function addSection() {
    setValues((current) => ({
      ...current,
      sections: [
        ...current.sections,
        { ...DEFAULT_SECTION, name: `بخش ${current.sections.length + 1}` },
      ],
    }));
  }
```

```183:183:app/(dashboard)/settings/_components/public-page-settings-form.tsx
      setGalleryImages((prev) => [...prev, ...urls]);
```

`{ ...DEFAULT_SECTION }` so the new row is not the shared default object (objects note).

**Remove by index:** `filter` with `!== index`.

```82:88:app/onboarding/_components/onboarding-wizard.tsx
  function removeSection(index: number) {
    setValues((current) => ({
      ...current,
      sections: current.sections.length > 1
        ? current.sections.filter((_, sectionIndex) => sectionIndex !== index)
        : current.sections,
    }));
  }
```

**Remove by value:** `filter` with `!== url`.

```192:193:app/(dashboard)/settings/_components/public-page-settings-form.tsx
  function removeGalleryImage(url: string) {
    setGalleryImages((prev) => prev.filter((u) => u !== url));
  }
```

```550:550:app/(dashboard)/settings/_components/settings-tabs.tsx
                onClick={() => onChange(images.filter((item) => item !== url))}
```

If two gallery slots ever shared the same URL, **both** would go. Uniqueness is implied by upload
paths (`Date.now()` in the filename).

**Replace one index:** `map`, not `arr[i] = next`.

```63:69:app/onboarding/_components/onboarding-wizard.tsx
  function updateSection(index: number, next: OnboardingValues["sections"][number]) {
    setValues((current) => ({
      ...current,
      sections: current.sections.map((section, sectionIndex) =>
        sectionIndex === index ? next : section,
      ),
    }));
  }
```

Unchanged indexes keep the **same object reference**. Changed index gets `next`. That is enough for
React if `next` is a new object (`{ ...section, seatCount }`).

---

## 5. Searching

### 5.1 `includes` — primitive membership (small closed lists)

```35:35:lib/upload.ts
  if (!ALLOWED_TYPES.includes(file.type)) {
```

```63:65:app/(dashboard)/logs/_lib/activity-normalizer.ts
  if (["EXPENSE", "PAYMENT"].includes(log.entityType)) return "finance";
  if (log.entityType === "MEMBERSHIP") return "membership";
  if (["SEAT", "SEAT_ASSIGNMENT"].includes(log.entityType)) return "seats";
```

```40:40:lib/marketing-leads/public-api.ts
        return url.protocol === "http:" && ["localhost", "127.0.0.1"].includes(url.hostname);
```

`includes` uses `===`. File types, entity enums, hostnames — **primitives**. It will not find an
object unless it is the same reference.

`ALLOWED_TYPES` is a tiny array. A `Set` would also work; the data-structures note said Set when the
question is “in the club” at scale. Four MIME types: array `includes` is the house choice.

**Not array `includes`:** CORS production origins use `Set.has`. Same question, keyed collection.

### 5.2 `indexOf` — **on strings here, not arrays**

```19:19:lib/marketing-leads/schema.ts
    .replace(/[۰-۹]/g, (digit) => String(persianDigits.indexOf(digit)))
```

`persianDigits` is a **string**. `String.prototype.indexOf`, not `Array.prototype.indexOf`. No
`arr.indexOf` in app TS. For arrays, Studivo prefers `includes` (boolean) or `find` (element).

`indexOf` on arrays returns `-1` when missing — easy to treat as a valid index if you forget.
`includes` cannot.

### 5.3 `find` — first element that matches, or `undefined`

```106:114:app/(dashboard)/_lib/actions/renew-membership.ts
      const occupyingSeat = current.seatAssignments.find((assignment) =>
        isOccupyingAssignment({
          endsAt: assignment.endsAt,
          membership: {
            status: current.status,
            endsAt: current.endsAt,
          },
        }),
      );
```

```72:73:lib/date.ts
  const hour = parts.find((part) => part.type === "hour")?.value ?? "00";
  const minute = parts.find((part) => part.type === "minute")?.value ?? "00";
```

`find` returns the **element** (an object / part), not an index. Chain `?.` when it may miss.
Occupancy: first open assignment. `Intl` parts: first `{ type: "hour" }`.

### 5.4 `findIndex` — **not in this repo**

Would return a number (`-1` if missing). The wizard already has the `index` from
`.map((x, index) => …)` in the UI, so it never searches for an index.

### 5.5 `some` / `every`

**`some` — true if any element matches** (occupancy conflict — product-critical):

```148:150:app/(dashboard)/_lib/actions/reserve-seat.ts
      if (existingAssignments.some(isOccupyingAssignment)) {
        throw new Error("این صندلی در حال حاضر دارای رزرو فعال است.");
      }
```

Stops early. `isOccupyingAssignment` is the predicate. Same check on swap (`manage-seat.ts`). This
is how double-booking is rejected **in the array of assignments already loaded**, before insert (DB
unique is still the backstop).

```128:128:app/(dashboard)/logs/_lib/activity-normalizer.ts
  if (seat && !secondaryFacts.some((fact) => fact.includes(seat))) secondaryFacts.push(`صندلی: ${seat}`);
```

`some` + string `includes` (not array `includes`) — “did we already mention this seat?”

**`every` — not in this repo.** “All assignments cancelled” would be `.every(...)`. Today the code
uses `find` / `some` / `filter`.

---

## 6. Iteration and transformation

None of `map` / `filter` / `reduce` / `forEach` change the source array. They may still share
**element objects**.

### 6.1 `forEach` — side effects, no return

```158:163:app/(dashboard)/_lib/dashboard-utils.ts
  seats.forEach((seat) => {
    const active = getActiveAssignment(seat.assignments);
    if (active?.membership.id) {
      const existing = activeAssignmentsByMembership.get(active.membership.id) || [];
      activeAssignmentsByMembership.set(active.membership.id, [...existing, seat.number]);
    }
  });
```

```207:212:app/(dashboard)/_lib/dashboard-utils.ts
  seatView.forEach((s) => {
    if (s.membership) {
      const price = Number(s.membership.planPrice) || 0;
      if (s.status === "reserved") activeRevenue += price;
      if (s.status === "renewal") atRiskRevenue += price;
    }
  });
```

`forEach` when you **accumulate elsewhere** (Map, numbers). Prefer `map` when you want an array
result. Do not `forEach` + `push` into React state.

### 6.2 `map` — same length, new array

```14:14:app/onboarding/actions.ts
    .map((seat) => seat.trim())
```

```84:88:app/onboarding/actions.ts
          data: seatNumbers.map((number) => ({
            sectionId: section.id,
            number,
            isActive: true,
          })),
```

```217:247:app/(dashboard)/_data/get-dashboard-overview.ts
    const seats: DashboardSeat[] = rawSeats.map((seat) => ({
      id: seat.id,
      number: seat.number,
      // ...
      assignments: seat.assignments.map((assignment) => ({
        // nested map — Date → ISO string, Decimal → Number
        payments: assignment.membership.payments.map((payment) => ({
```

Nested `map` is a nested array (section 11). Each level returns **new** objects. Prisma `Date` /
`Decimal` become JSON-safe primitives.

Wizard `map` with index: replace one slot, keep others (section 4.3).

Callback return is the new element. Forgetting `return` in a block body yields `undefined` slots — a
real-world hole that `tsc` often catches (`(seat) => { id: seat.id }` is a labeled block, not an
object; always wrap objects as `({ … })`).

### 6.3 `filter` — maybe shorter, new array

```15:15:app/onboarding/actions.ts
    .filter(Boolean);
```

Empty strings out (ToBoolean).

```249:251:app/(dashboard)/_data/get-dashboard-overview.ts
    const availableSeats: DashboardAvailableSeat[] = rawSeats
      .filter((seat) => seat.isActive && !getActiveAssignment(seat.assignments))
      .map((seat) => ({
```

**Filter then map** — do not map everything and then drop. Occupancy map stats:

```197:201:app/(dashboard)/_lib/dashboard-utils.ts
  const stats = {
    available: seatView.filter((s) => s.status === "available").length,
    reserved: seatView.filter((s) => s.status === "reserved").length,
    renewal: seatView.filter((s) => s.status === "renewal").length,
    expired: seatView.filter((s) => s.status === "expired").length,
  };
```

Four allocations for four counts. Clear at hall scale. A single `forEach` could count without
arrays; the house style prefers readable `filter`.

`filter` keeps the **same element references** that passed. Mutating a filtered seat still mutates
the object inside `seatView`.

### 6.4 `reduce` — fold to one value

```60:63:app/(dashboard)/_lib/actions/membership-payments.ts
      const totalPaidBefore = membership.payments.reduce(
        (sum, p) => sum + Number(p.amount),
        0,
      );
```

Start at `0` (number). Convert **inside** so `+` is addition (casting note). Missing initial value
would use `payments[0]` as `sum` — a Decimal object — and concatenate. **Always pass the
initializer.**

```79:88:lib/marketing-leads/schema.ts
  return error.issues.reduce<MarketingLeadFieldErrors>((fieldErrors, issue) => {
    const field = issue.path[0];
    if (
      typeof field === "string" &&
      ["fullName", "phoneNumber", "studyhallName", "email", "notes"].includes(field) &&
      !fieldErrors[field as MarketingLeadFieldName]
    ) {
      fieldErrors[field as MarketingLeadFieldName] = issue.message;
    }
    return fieldErrors;
  }, {});
```

`reduce` into a **plain object**, mutating the accumulator that `reduce` created (`{}`). Allowed
because the object was born in this call. First error per field wins (`!fieldErrors[field]`).
`issue.path` is itself an array (`path[0]`).

---

## 7. Sorting

### 7.1 `sort` mutates — copy first

```184:194:app/(dashboard)/_lib/dashboard-utils.ts
  const seatView = sortByRenewal
    ? [...initialSeatView].sort((a, b) => {
        const timeA = a.membership
          ? new Date(a.membership.endsAt).getTime()
          : Infinity;
        const timeB = b.membership
          ? new Date(b.membership.endsAt).getTime()
          : Infinity;
        if (timeA !== timeB) return timeA - timeB;
        return a.seatNumber - b.seatNumber;
      })
    : initialSeatView;
```

`[...initialSeatView]` copies **indexes** (same seat objects). `.sort` reorders the copy.
Comparator: negative → `a` first. `Infinity` for empty seats so they sink. Tie-break numeric
`seatNumber` (`parseInt` earlier).

```490:491:app/(dashboard)/finance/_data/get-finance-dashboard.ts
      .sort((left, right) => right.date.getTime() - left.date.getTime())
      .slice(0, RECENT_TRANSACTION_LIMIT);
```

Here the array was **just created** by `[...payments, ...expenses]` — no other owner — so in-place
`sort` is safe. Then `slice` takes the first 10 of the mutated array.

**Default `sort()` without comparator** is lexicographic (`"10"` before `"2"`). Never sort seat
labels that way without `parseInt` / numeric fields.

### 7.2 `toSorted` / `reverse` / `toReversed` — **not in this repo**

`toSorted` would copy+sort without `[...]`. Same idea, newer. `reverse` mutates; `toReversed`
copies. Occupancy map already does `[...].sort`. Do not `.reverse()` a Prisma result you still need
in query order.

---

## 8. `slice` vs `splice`

**`slice(start, end)` — copy a window, original unchanged. `end` excluded.**

Used on **strings** more than arrays in UI (`name.slice(0, 2)` avatars). On arrays:

```471:472:app/(dashboard)/finance/_data/get-finance-dashboard.ts
      .sort((left, right) => right.revenue - left.revenue)
      .slice(0, 5);
```

Top 5 after sort. `slice(0, 5)` on a 3-element list returns 3 — no throw.

**`splice` — mutate: remove/insert at an index.** Unused. Removing a gallery image is `filter`, not
`splice(i, 1)`.

|            | `slice`         | `splice`             |
| ---------- | --------------- | -------------------- |
| Mutates?   | no              | **yes**              |
| Returns    | copied subarray | **removed** elements |
| In Studivo | yes (top N)     | no                   |

`slice()` with no args copies the whole list — same as `[...arr]` for dense arrays.

---

## 9. Mutation vs non-mutating methods

**Mutating (avoid on shared / React state):** `push`, `pop`, `shift`, `unshift`, `splice`, `sort`,
`reverse`, index assign `arr[i] = x`, `arr.length = 0`.

Studivo mutates only:

- `push` on a **fresh** local array (`dayKeys`, `secondaryFacts`);
- `sort` on a **fresh** array or an explicit `[...]` copy.

**Non-mutating (preferred):** `map`, `filter`, `reduce`, `slice`, `concat` / spread, `includes`,
`find`, `some`, `Array.from`, `toSorted` (if you add it later).

React: `setX(old => old.filter(...))` always a **new** array. Server: return new lists from `map` so
the Prisma result is not rewritten.

---

## 10. Shallow copy (same rule as objects)

```javascript
const copy = [...seats];
copy[0] === seats[0]; // true — same seat object
copy[0].status = "expired"; // seats[0] changed too
```

`[...arr]`, `arr.slice()`, `arr.map(x => x)` all **shallow**. Nested `assignments` arrays inside a
seat are still shared unless you `map` them too (dashboard overview does nested `map` on purpose).

Wizard:

```43:48:app/onboarding/_components/onboarding-wizard.tsx
function cloneSection(section: OnboardingValues["sections"][number]) {
  return { ...section };
}
```

```121:124:app/onboarding/_components/onboarding-wizard.tsx
    const payload: OnboardingValues = {
      ...values,
      sections: values.hasSections ? values.sections : [cloneSection(values.sections[0] ?? DEFAULT_SECTION)],
      plans: values.plans.map(clonePlan),
```

New **array**, new **section/plan objects** (`clonePlan`). Deep enough because those objects have no
nested arrays of objects that will be mutated later. `manualSeats` is a string (primitive copy).

Gallery: `[...prev, ...urls]` — strings are primitives, so shallow is deep.

JSON clone (`JSON.parse(JSON.stringify(arr))`) is structured-data copy: loses `Date`, `undefined`,
methods. Occupancy views use explicit `map` instead.

---

## 11. Nested arrays

Prisma: `seats[]` → each `assignments[]` → each `payments[]`. That is an array of objects of arrays.

```224:244:app/(dashboard)/_data/get-dashboard-overview.ts
      assignments: seat.assignments.map((assignment) => ({
        // ...
          payments: assignment.membership.payments.map((payment) => ({
```

`getActiveAssignment(seat.assignments)` searches the **inner** array. Outer `filter` uses that.
Flattening too early (`allAssignments = seats.flatMap(s => s.assignments)`) would lose “which seat.”
Occupancy keeps nesting.

`flatMap` when you **do** want one list:

```155:157:app/platform/_components/leads-table.tsx
              leads.flatMap((lead) =>
                lead.owner ? [[lead.owner.id, lead.owner.name] as const] : []
              )
```

Each lead → `[]` or a one-element array of a pair; `flatMap` concatenates one level. `map` +
`filter` + `flat` would be the long form. **`flat` unused** as a named call; `flatMap` is the one
nested-array helper in the repo.

`flat(2)` would collapse two levels — do not flatten seat → assignment → payment unless a report
truly wants a denormalized table.

---

## 12. Destructuring and spread

### 12.1 Index destructure

```41:42:lib/reminders.ts
  const [fromYear, fromMonth, fromDay] = fromKey.split("-").map(Number);
  const [toYear, toMonth, toDay] = toKey.split("-").map(Number);
```

`split` returns an array. Holes / short strings → later bindings `undefined` → `Number(undefined)`
is `NaN`.

React: `const [values, setValues] = useState(...)` destructures the **tuple** React returns (still
an array).

### 12.2 Rest / spread

```75:77:app/onboarding/_components/onboarding-wizard.tsx
      sections: [
        ...current.sections,
        { ...DEFAULT_SECTION, name: `بخش ${current.sections.length + 1}` },
      ],
```

```517:517:app/(dashboard)/settings/_components/settings-tabs.tsx
      onChange([...images, ...(await Promise.all(selected.map(uploadImage)))]);
```

Spread of two arrays copies **indexes** into a new list. `Promise.all` on
`selected.map(uploadImage)` is an array of Promises — not nested arrays; `all` returns an array of
URLs, then spread.

```162:162:app/(dashboard)/_lib/dashboard-utils.ts
      activeAssignmentsByMembership.set(active.membership.id, [...existing, seat.number]);
```

Copy-on-write for the Map **value** array so keys do not share one list.

Spread is **not** `push`. It allocates. Huge galleries would copy each add; max 8 images,
irrelevant.

---

## 13. Reference nuances (common Studivo bugs)

1. **Seat label ≠ index.** `seats[0].number` might be `"12"` or `"A-1"`. Sort by
   `parseInt(seat.number, 10) || 0` for numeric-looking labels; keep the string for identity.

2. **`filter(Boolean)` on objects.** Always truthy. Use a real predicate (`isOccupyingAssignment`).
   On strings it drops `""` — that is why manual seats trim then filter.

3. **Shared default in an array.** `sections: [DEFAULT_SECTION]` — if you mutate
   `values.sections[0].name` in place, `DEFAULT_SECTION` changes for the next add. Wizard updates
   replace via `map` + new objects; `addSection` spreads `DEFAULT_SECTION`. Do not
   `values.sections[0].seatCount = 40`.

4. **`sort` without copy** on a list still rendered elsewhere — occupancy map copies. Finance
   `topPlans` sorts a brand-new `Array.from` result.

5. **`some` vs `find` vs `filter`.** Conflict check: `some` (boolean, throw). Need the row: `find`.
   Need all matches: `filter`. Do not `filter` then `[0]` when `find` is enough.

6. **`includes` on objects.** `seats.includes(seatFromPrisma)` is false if it is a different object
   with the same `id`. Compare `seat.id` or use `some(s => s.id === id)`.

7. **`reduce` without initial value** on an empty `payments[]` throws. Always `reduce(..., 0)`.

8. **`Array.from({ length: count ?? 0 })`.** `count` optional; `0` → `[]` →
   `"شماره صندلی را وارد کنید."` Empty is not `undefined`.

9. **Nested map forgot inner copy.** Spreading a seat (`...seat`) keeps the same `assignments`
   array. Dashboard overview maps assignments explicitly when sending to the client.

10. **String `.slice` vs array `.slice`.** `member.name.slice(0, 2)` is characters, not staff[0],
    staff[1]. Same method name, different prototype.

11. **`length` after `filter`.** Gallery cap uses `galleryImages.length + files.length` **before**
    upload. Race: two pickers could exceed 8; UI-only. Not an array-API bug.

12. **Holes vs `undefined`.** Not produced here. If you ever `arr[5] = x` on a short list, `map`
    skips holes but visits `undefined`. Prefer `push` / `from`.

---

## 14. API cheat sheet (this repo vs the language)

| Method                            | In Studivo?                 | Mutates?                 |
| --------------------------------- | --------------------------- | ------------------------ |
| `[]` literal                      | yes                         | n/a                      |
| `Array.from`                      | yes                         | no (new array)           |
| `Array()` / `Array.of`            | no                          | —                        |
| `push`                            | yes, local builders         | **yes**                  |
| `pop` `shift` `unshift` `splice`  | no                          | yes                      |
| `includes`                        | yes                         | no                       |
| `indexOf`                         | string only                 | no                       |
| `find`                            | yes                         | no                       |
| `findIndex`                       | no                          | no                       |
| `some`                            | yes (occupancy)             | no                       |
| `every`                           | no                          | no                       |
| `forEach`                         | yes                         | no (callback may mutate) |
| `map` `filter` `reduce`           | yes                         | no                       |
| `sort`                            | yes, after copy or on fresh | **yes**                  |
| `toSorted` `reverse` `toReversed` | no                          | —                        |
| `slice`                           | yes (top N; strings too)    | no                       |
| `flatMap`                         | yes (leads owners)          | no                       |
| `flat` `.at()`                    | no                          | no                       |

---

## 15. Mental model

1. Array = object + integer indexes + `length`.
2. Prefer **new arrays** (`map` / `filter` / spread) over splice/pop for state and Prisma snapshots.
3. `push` / `sort` only when you **own** the array from the line above (or copied with `[...]`).
4. Search: `includes` primitives, `find` / `some` objects, never `includes` for occupancy.
5. Copies are **shallow** — nested assignment lists need nested `map`.
6. `Array.from` is how FileList, `{ length }`, `Set`, and `Map` become arrays.

---

## 16. Exercises on this repo

Predict; do not edit production code.

1. `Array.from({ length: 3 }, (_, i) => String(i + 1))` vs
   `new Array(3).map((_, i) => String(i + 1))`.
2. `removeSection` with one section left — which array is returned? Same reference as
   `current.sections`?
3. `addSection` without `{ ...DEFAULT_SECTION }` — change section 2’s name; what happens to the next
   add?
4. `existingAssignments.some(isOccupyingAssignment)` vs `.filter(...).length > 0` — occupancy
   correctness? Extra work?
5. `payments.reduce((s, p) => s + Number(p.amount))` **without** `0` on a one-element list and on
   `[]`.
6. `[...initialSeatView].sort(...)` then mutate `seatView[0].status`. Does `initialSeatView[0]`
   change?
7. `images.filter((item) => item !== url)` when `url` appears twice.
8. `["SEAT"].includes(log.entityType)` vs `log.entityType === "SEAT"` — when is the array form worth
   it?
9. `flatMap` of owners: two leads, same owner id, different name strings — which name survives the
   later `Map`?
10. `dayKeys.push` vs `dayKeys = [...dayKeys, key]` inside the finance-range loop — same result? Why
    `push` is acceptable there?

---

## Files cited

| File                                                                 | Array topic                                                        |
| -------------------------------------------------------------------- | ------------------------------------------------------------------ |
| `app/onboarding/actions.ts`                                          | `from({ length })`, `from(Set)`, `map`/`filter`, `length === 0`    |
| `app/onboarding/_components/onboarding-wizard.tsx`                   | literal; spread add; `filter` remove; `map` replace; shallow clone |
| `app/(dashboard)/settings/actions.ts`                                | `from` prefix range; one-element literal + `filter`                |
| `app/(dashboard)/settings/_components/public-page-settings-form.tsx` | `from(FileList)`; spread append; `filter` remove; `length` cap     |
| `app/(dashboard)/settings/_components/settings-tabs.tsx`             | same gallery pattern                                               |
| `app/(dashboard)/_lib/dashboard-utils.ts`                            | `forEach`; `map`; `[...].sort`; `filter` counts                    |
| `app/(dashboard)/_lib/actions/reserve-seat.ts`                       | `some` occupancy                                                   |
| `app/(dashboard)/_lib/actions/renew-membership.ts`                   | `find`; `[0]` fallback                                             |
| `app/(dashboard)/_lib/actions/membership-payments.ts`                | `reduce` + `Number`                                                |
| `app/(dashboard)/_data/get-dashboard-overview.ts`                    | nested `map`; `filter`+`map`                                       |
| `app/(dashboard)/logs/_lib/activity-normalizer.ts`                   | `isArray`; `[]`+`push`; `includes`; `some`                         |
| `app/(dashboard)/finance/_data/get-finance-dashboard.ts`             | `from(Map)`; `sort`+`slice`                                        |
| `lib/finance-range.ts`                                               | `push` in a loop                                                   |
| `lib/upload.ts`                                                      | `includes` MIME                                                    |
| `lib/date.ts`                                                        | `find` on parts                                                    |
| `lib/reminders.ts`                                                   | destructure `split` array                                          |
| `lib/marketing-leads/schema.ts`                                      | `reduce` to object; string `indexOf`                               |
| `app/platform/_components/leads-table.tsx`                           | `flatMap`                                                          |

Series: data-structures overview → **this file** → next **Set and Map** deep-dive. Next TS: unions
and narrowing, generics, or `typeof` / `keyof`.
