# JavaScript Data Structures — Studivo Walkthrough

Objects were **one bag of properties**. Built-ins were **which constructors the language ships**.
This note is **how values are arranged** — MDN’s three families:

```text
Data Structures
├── Structured Data     JSON, FormData, URLSearchParams, schema.org bags
├── Keyed Collections   Map, Set  (not WeakMap/WeakSet here)
└── Indexed Collections Array, array-like ({ length })  (not TypedArray)
```

A **structure** is not a type in the TS-tree sense. It is a **runtime layout**: ordered slots,
unique keys, or a wire format that becomes objects again after parse.

Read this with the cited files open. Predict: _lookup by index, by key, or by walking JSON? Does
this copy, share, or stringify?_

Related: `docs/javascript-objects-in-studivo.md`, `docs/javascript-built-in-objects-in-studivo.md`,
`docs/javascript-primitive-types-in-studivo.md`, `docs/javascript-type-casting-in-studivo.md`.

---

## 1. The rule that pays rent

Pick the layout from the **access pattern**, not from habit:

| Need                                                                   | Structure                                        | Studivo                                            |
| ---------------------------------------------------------------------- | ------------------------------------------------ | -------------------------------------------------- |
| Ordered list; n-th item; `.map` / `.filter`                            | **Indexed** — `Array`                            | seats, payments, sections, `seatNumbers`           |
| Membership / uniqueness; `.has`                                        | **Keyed** — `Set`                                | CORS origins; unique manual seats                  |
| Lookup by id; keys that must stay strings without `obj[id]` collisions | **Keyed** — `Map`                                | rate limit by IP; duplicate seats by membership id |
| Talk to another process / HTML / SEO                                   | **Structured data** — JSON (or FormData / query) | SMS.ir body, gallery field, `ld+json`              |

Plain `{ NEW: "جدید" }` is still an object (objects note). Use it when the key set is **closed and
known at compile time** (`Record<LeadStatus, string>`). Use `Map` when keys **arrive at runtime**
(IPs, membership ids).

No `TypedArray`, `ArrayBuffer`, `WeakMap`, `WeakSet`, or `structuredClone` in app code.

---

## 2. Indexed collections — `Array`

Indexed means **integer positions** `0 … length-1`. `seatNumbers[0]` is the first label. Arrays are
objects (`typeof [] === "object"`) with a magic `length`.

### 2.1 Build from a length (no source list)

```74:77:app/onboarding/actions.ts
        const seatNumbers =
          sectionInput.mode === "AUTO"
            ? Array.from({ length: sectionInput.seatCount }, (_, index) => String(index + 1))
            : uniqueSeatNumbers(normalizeManualSeatNumbers(sectionInput.manualSeats));
```

`{ length: n }` is **array-like**, not an array: it has `length` and no `.map` until `Array.from`
allocates a real `Array`. Index `0` → `"1"` (string primitive — seats are identifiers). Prisma
`createMany` wants an array of row objects, produced next with `.map`.

Empty hall: `length: 0` → `[]`. The next lines throw if `seatNumbers.length === 0`.

### 2.2 Transform, keep order

```11:16:app/onboarding/actions.ts
function normalizeManualSeatNumbers(value: string) {
  return value
    .split(",")
    .map((seat) => seat.trim())
    .filter(Boolean);
}
```

`split` → indexed collection of strings. `.map` same length, new array. `.filter(Boolean)` **drops**
holes made by `""` (ToBoolean — casting note). Order of what remains is the operator’s comma order.

```60:63:app/(dashboard)/_lib/actions/membership-payments.ts
      const totalPaidBefore = membership.payments.reduce(
        (sum, p) => sum + Number(p.amount),
        0,
      );
```

`.reduce` folds an indexed list of payment **objects** down to a primitive `number`. Index does not
matter; order of Prisma `payments` is whatever the query returned. Convert Decimal **inside** the
reducer (`Number`) so `+` never concatenates.

```166:181:app/(dashboard)/_lib/dashboard-utils.ts
  const initialSeatView = seats.map((seat) => {
    const currentAssignment = getActiveAssignment(seat.assignments);
    // ...
    return {
      ...seat,
      seatNumber: seatNum,
      membership,
      status: getSeatStatus(membership?.endsAt, membership?.status),
      isDuplicate,
      duplicateSeats: isDuplicate ? activeAssignmentsByMembership.get(membership.id!) : undefined,
    };
  });
```

Same length as `seats`. Each element is a **new object** (`...seat` plus extra fields). The source
`seats` array is unchanged. Nested `assignments` arrays are still the **same references** (objects
note — spread is shallow).

### 2.3 Copy then sort — do not mutate the mapped list in place unless you mean to

```184:195:app/(dashboard)/_lib/dashboard-utils.ts
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

`[...initialSeatView]` copies the **index list** (same object elements). `.sort` mutates that copy.
Occupancy map order is an indexed-collection problem (compare `seatNumber`, timestamps), not a Map.

```197:201:app/(dashboard)/_lib/dashboard-utils.ts
  const stats = {
    available: seatView.filter((s) => s.status === "available").length,
    reserved: seatView.filter((s) => s.status === "reserved").length,
    renewal: seatView.filter((s) => s.status === "renewal").length,
    expired: seatView.filter((s) => s.status === "expired").length,
  };
```

Four passes over the same index range. Hall scale is small. Each `.filter` allocates a new array
just to read `.length`.

### 2.4 Destructuring indexes

```40:42:lib/reminders.ts
function getTehranCalendarDayDifference(fromKey: string, toKey: string) {
  const [fromYear, fromMonth, fromDay] = fromKey.split("-").map(Number);
  const [toYear, toMonth, toDay] = toKey.split("-").map(Number);
```

`split("-")` → indexed `["2026","09","26"]`. `.map(Number)` → `[2026, 9, 26]`. Tuple destructure is
still array access `arr[0]`, `arr[1]`, `arr[2]`. Wrong shape (`"2026-09"`) → `fromDay === undefined`
→ `Number(undefined)` is `NaN` (casting note).

### 2.5 `Array.isArray` — indexed vs plain object

```34:36:app/(dashboard)/logs/_lib/activity-normalizer.ts
export function toAuditMetadata(value: unknown): AuditMetadata {
  if (!value || typeof value !== "object" || Array.isArray(value)) return {};
  return value as AuditMetadata;
}
```

JSON arrays are indexed collections (`typeof "object"` **and** `Array.isArray`). Audit metadata must
be a **keyed** plain object. An array payload is rejected, not walked as `{ 0: first }`.

### 2.6 What is _not_ an indexed collection here

`TypedArray` (`Uint8Array`, etc.) and `ArrayBuffer` are indexed **bytes**. Studivo does not parse
binary; uploads go through `File` / `FormData`. Do not reach for `Buffer` in Next server actions
unless you are actually hashing bytes.

---

## 3. Keyed collections — `Set` and `Map`

Keyed means **lookup by value identity** (`===`), not by `0..n` and not by coerced object-property
strings.

Plain objects (`STATUS_LABELS.NEW`) coerce keys to string. `obj[1]` and `obj["1"]` are the same
slot. `Map` / `Set` do not stringify keys.

### 3.1 `Set` — uniqueness, then maybe back to an array

```18:20:app/onboarding/actions.ts
function uniqueSeatNumbers(numbers: string[]) {
  return Array.from(new Set(numbers));
}
```

Pipeline: **indexed** (`string[]`) → **keyed** (`Set`, drop duplicate labels) → **indexed**
(`Array.from`) because `createMany` wants an array. Insert order of first occurrence is kept. `"12"`
and `"12 "` are different if trim already ran; if not, they are two seats.

```3:6:lib/marketing-leads/public-api.ts
const PRODUCTION_ALLOWED_ORIGINS = new Set([
  "https://studivo.ir",
  "https://www.studivo.ir",
]);
```

Closed allowlist. `.has(origin)` — membership test, not `origins.includes(origin)` on a long list.
Values are primitives; `"https://studivo.ir/"` (slash) is a **different** key. That is why CORS
compares exact origin strings, not “starts with.”

A `Set` is the right structure when the question is **is this in the club?** An array is fine for
two URLs; the type documents the intent.

### 3.2 `Map` — runtime keys, mutable values

```22:26:lib/marketing-leads/public-api.ts
const globalForRateLimit = globalThis as GlobalWithMarketingLeadRateLimit;
const rateLimitStore =
  globalForRateLimit.marketingLeadRateLimitStore ?? new Map<string, RateLimitEntry>();

globalForRateLimit.marketingLeadRateLimitStore = rateLimitStore;
```

Key = client IP (`string`). Value = `{ attempts, resetAt }` **object**. The Map stores a
**reference**. Later:

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

`.get` miss → `undefined` (not `null`). New window → `.set` a **new** object. Same window → mutate
`existing.attempts` then `.set` the same reference (redundant but clear). A plain `{ [ip]: entry }`
would also work for string IPs; `Map` avoids `__proto__` / `toString` as accidental keys and has
`.size`.

The store is process-local (comment in file). Horizontal scale would replace this keyed collection
with Redis — same _idea_ (key = IP), different _structure_.

### 3.3 `Map` as a grouping index over an array

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

Walk an **indexed** list of seats; build a **keyed** index membershipId → seat numbers. Each `.set`
replaces the array (`[...existing, seat.number]`) instead of `existing.push`, so values are not
shared across keys. Duplicate occupancy is then `.get(id)?.length > 1` — O(1) after the one pass.

This is the classic pattern: **array is the source of truth; Map is the index.**

### 3.4 `Map` to dedupe while keeping the last (or first) object

```193:195:components/ui/field.tsx
    const uniqueErrors = [
      ...new Map(errors.map((error) => [error?.message, error])).values(),
    ]
```

`errors` is indexed. Each element becomes a `[key, value]` pair. `new Map(iterableOfPairs)` — **last
write wins** for the same `message`. `.values()` is keyed-collection iteration in insert order.
Spread back to an indexed array for React.

```153:159:app/platform/_components/leads-table.tsx
        : Array.from(
            new Map(
              leads.flatMap((lead) =>
                lead.owner ? [[lead.owner.id, lead.owner.name] as const] : []
              )
            ).entries()
          ).map(([id, name]) => ({ id, name })),
```

`flatMap` → indexed list of pairs. `Map` keyed by owner `id` (dedupe). `.entries()` → pairs again.
`Array.from` + `.map` → indexed `{ id, name }[]` for a `<select>`. Same triangle: array → Map →
array.

### 3.5 When a plain object is the keyed collection

```3:17:lib/expense-categories.ts
export const EXPENSE_CATEGORY_LABELS: Record<ExpenseCategory, string> = {
  RENT: "اجاره",
  // ...
};

export const EXPENSE_CATEGORY_OPTIONS: { value: ExpenseCategory; label: string }[] =
  (Object.entries(EXPENSE_CATEGORY_LABELS) as [ExpenseCategory, string][]).map(
    ([value, label]) => ({ value, label }),
  );
```

Closed enum → **object** as the map. `Object.entries` is how you turn that keyed object into an
**indexed** list of pairs for a dropdown. Do not use `new Map` for Prisma enums;
`Record<Enum, string>` is the TS object type (types note) and the runtime structure is a plain
object.

`STATUS_LABELS`, `activityGroupLabels` — same. Runtime keys known. `Object.hasOwn` / `Object.keys`
belong here, not `map.get`.

### 3.6 Absent: `WeakMap` / `WeakSet`

Weak collections hold object keys without preventing GC. Studivo keys rate limits by **string** IP
and seats by **string** id. Nothing to weakly hold. Skip them until you cache something keyed by a
DOM node or a Prisma result you do not own.

---

## 4. Structured data — shapes that cross a boundary

**Structured data** is a layout meant to be **copied, logged, or sent**, then reconstituted. In
browsers/Node that is mostly **JSON**. Host cousins: `FormData`, `URLSearchParams`. Schema.org
JSON-LD is JSON with agreed keys.

After `JSON.parse` you always get primitives + plain objects + arrays — never `Date`, `Map`, `Set`,
or class instances.

### 4.1 JSON as the wire for objects and arrays

```109:109:lib/sms.ts
       body: JSON.stringify({ mobile, templateId, parameters }),
```

Plain object + nested **indexed** `parameters` array. `JSON.stringify` walks enumerable own keys →
**string** primitive (structured payload). `undefined` fields dropped. The HTTP body is not a JS
object anymore.

```62:66:lib/push.ts
      JSON.stringify({
        title: payload.title,
        body: payload.body,
        url: payload.url ?? "/",
      }),
```

Notification payload: closed object → string → browser `push` event → `JSON.parse` in `sw.js`
(built-ins / interop). Identity dies. `{ title, body }` in the worker is a **new** object.

```151:155:app/(dashboard)/settings/actions.ts
    galleryImages = JSON.parse(
      formData.get("galleryImages")?.toString() || "[]",
    );
```

HTML FormData cannot send a nested array. The client does
`formData.set("galleryImages", JSON.stringify(galleryImages))` — **indexed** `string[]` →
**structured string** → parse back to array. Invalid JSON throws `SyntaxError`; `catch` turns it
into `ActionResult`. Missing field → `"[]"` → empty indexed collection, not `null`.

`JSON.parse` does not typecheck. `galleryImages` is annotated `string[]` but a parsed `{ foo: 1 }`
would still assign at runtime. Zod belongs at this seam if the array is untrusted beyond “we just
stringified it in our form.”

### 4.2 JSON as clone (lossy)

```88:90:components/pwa/PushNotificationManager.tsx
      setSubscription(sub)
      const serializedSub = JSON.parse(JSON.stringify(sub))
      const result = await subscribeUser(serializedSub)
```

`PushSubscription` is a host object (methods, prototype). JSON clone keeps enumerable data
(`endpoint`, keys) and **drops methods**. The server action receives a plain object — structured
data, not the live Push API object. Same pattern as “structured clone via JSON.” Real
`structuredClone(sub)` would still fail on some host objects; JSON is the intentional subset.

### 4.3 JSON-LD — structured data for machines that are not your app

```166:187:app/[slug]/page.tsx
  const structuredData = {
    "@context": "https://schema.org",
    "@type": "LocalBusiness",
    name: hall.name,
    description:
      hall.description?.trim() || `سالن مطالعه ${hall.name} در Studivo.`,
    url: canonicalUrl,
    ...(address ? { address: { "@type": "PostalAddress", streetAddress: address, addressCountry: "IR" } } : {}),
    ...(phoneNumber ? { telephone: phoneNumber } : {}),
    ...(monthlyFee > 0
      ? { makesOffer: { "@type": "Offer", price: monthlyFee, priceCurrency: "IRR" } }
      : {}),
  };
  // ...
        dangerouslySetInnerHTML={{ __html: JSON.stringify(structuredData) }}
```

This is the MDN phrase **structured data** in the SEO sense: a plain object whose keys Google agreed
on, serialized into `<script type="application/ld+json">`. Nested `address` is an object; optional
fields use spread so JSON omits them (better than `"address": null` for crawlers). `price` is a
**number** in the object; stringify emits JSON number, not `"120000"` unless you converted wrong.

Not a `Map`. Crawlers do not run your `Set`. They parse JSON.

### 4.4 FormData and URLSearchParams — structured, not JSON

`FormData` is a **keyed** host collection of string/File entries (duplicates allowed per key).
Access is `.get` / `.getAll`, not `[0]`. Checkboxes absent → `null` (casting note). Nested arrays of
gallery URLs do not fit; hence JSON-in-a-field.

```37:41:app/(dashboard)/logs/_components/log-filters.tsx
  function updateFilters(updates: Record<string, string | null>) {
    const params = new URLSearchParams(searchParams.toString());
    params.set("page", "1");
    Object.entries(updates).forEach(([key, value]) => value ? params.set(key, value) : params.delete(key));
    router.push(`${pathname}?${params.toString()}`);
```

`URLSearchParams` is a keyed list of query pairs (order preserved, stringify with `&`).
`Object.entries(updates)` bridges a plain object to that collection. `page` is a string in the query
(`"1"`); `Number(params.page)` happens later (indexed? no — one key). Query strings are structured
data for **navigation**, not for seat rows.

### 4.5 Prisma rows vs JSON

A membership from Prisma is an **object** in memory (`Decimal`, `Date` instances). Putting it on the
wire (`toISOString()`, `Number(planPrice)` in dashboard overview) is an explicit **structure
transform** into JSON-safe primitives. Do not `JSON.stringify` a Prisma Decimal and expect a number
back without `toDisplayAmount` (casting note).

`AuditMetadata = Record<string, unknown>` is “this was structured data (JSON column) and we have not
indexed it yet.” `numberValue` / `text` then pull primitives out.

---

## 5. Crossing the three families (typical Studivo pipelines)

### Onboarding seats

```text
string "1, 1, 2"     structured (form field)
  → split/map/filter   indexed string[]
  → new Set            keyed unique
  → Array.from         indexed again
  → createMany         rows in Postgres (relational structure, not JS)
```

### Occupancy map

```text
seats[]                    indexed
  → Map<membershipId, []>  keyed index
  → seats.map(...)         indexed view
  → [...view].sort         indexed, different order
```

### Lead owners dropdown

```text
leads[]                    indexed
  → flatMap pairs          indexed
  → Map(id → name)         keyed dedupe
  → Array.from entries     indexed options
```

### Public hall SEO

```text
Prisma hall                object + Dates
  → { @type, name, ... }   plain object (structured data)
  → JSON.stringify         string
  → HTML script            crawler parses JSON
```

### Rate limit

```text
x-forwarded-for            structured header string
  → IP string              primitive key
  → Map.get/set            keyed collection in process memory
```

If you store a `Map` in JSON you get `{}`. If you need to persist it, convert:
`Object.fromEntries(map)` (plain object / JSON) or an array of pairs.

---

## 6. Mental model

1. **Indexed (`Array`)** — position and order. Seats, payments, `.map` / `.filter` / `.reduce` /
   `length`.
2. **Keyed (`Map` / `Set`)** — identity lookup and uniqueness. Runtime keys. Convert back to arrays
   at UI/DB edges.
3. **Plain object** — keyed, but **fixed** keys (`Record<Enum, string>`). Not a `Map`.
4. **Structured data** — JSON / FormData / query / JSON-LD. Survives a process boundary;
   `Date`/`Map`/`Set` do not.
5. **Array-like** `{ length }` is not an array until `Array.from`.
6. No typed arrays / weak collections in this app — do not import them for halls.

---

## 7. Exercises on this repo

Predict; do not edit production code.

1. `uniqueSeatNumbers(["1", "1", "2", "1"])` — Set size? `Array.from` order?
2. `Array.from({ length: 3 }, (_, i) => String(i + 1))` vs `[...Array(3)]` — holes? `.map` on
   `Array(3)`?
3. Rate-limit `Map` value mutated with `existing.attempts += 1` without `.set` — does the next
   `.get` see it? Why is `.set` still there?
4. Replace `activeAssignmentsByMembership` with a plain object. What IP-like key would be a bad
   membership id? (`"__proto__"`)
5. `JSON.stringify` a `Map` of IPs. Parse it. What do you get?
6. `JSON.parse('["a","b"]')` into `toAuditMetadata` — return value? Why?
7. Gallery: `JSON.stringify` then parse of `["https://…"]` — same array _identity_? Same strings?
8. `new Set(["https://studivo.ir", "https://studivo.ir/"]).size` — CORS implications.
9. `field.tsx` Map keyed by `error?.message`. Two errors with `undefined` message — how many unique?
10. Why `EXPENSE_CATEGORY_LABELS` is an object and `rateLimitStore` is a Map — phrase the access
    pattern for each.

---

## Files cited

| File                                                  | Structure                                        |
| ----------------------------------------------------- | ------------------------------------------------ |
| `app/onboarding/actions.ts`                           | Array.from length; split/map/filter; Set → array |
| `app/(dashboard)/_lib/dashboard-utils.ts`             | Array map/filter/sort; Map index by membership   |
| `app/(dashboard)/_lib/actions/membership-payments.ts` | Array.reduce                                     |
| `lib/reminders.ts`                                    | split → indexed tuple                            |
| `lib/marketing-leads/public-api.ts`                   | Set allowlist; Map rate limit                    |
| `lib/expense-categories.ts`                           | Object as closed keyed map; entries → array      |
| `lib/sms.ts` / `lib/push.ts`                          | JSON.stringify wire objects                      |
| `app/(dashboard)/settings/actions.ts`                 | JSON.parse array from FormData                   |
| `app/[slug]/page.tsx`                                 | schema.org structured data                       |
| `components/pwa/PushNotificationManager.tsx`          | JSON clone of host object                        |
| `components/ui/field.tsx`                             | Map dedupe → array                               |
| `app/platform/_components/leads-table.tsx`            | Map dedupe owners                                |
| `app/(dashboard)/logs/_lib/activity-normalizer.ts`    | Array.isArray vs object                          |
| `app/(dashboard)/logs/_components/log-filters.tsx`    | URLSearchParams                                  |

Related: objects (plain `{}` vs reference), built-ins (`Array.from`, `Map`, `JSON` catalog),
primitives (string keys vs numbers), casting (`Number` inside `reduce`).

Next JS: **`this` / constructors**, or a deeper **Array methods** drill. Next TS: unions and
narrowing, generics, or `typeof` / `keyof`.
