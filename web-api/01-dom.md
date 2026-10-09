# DOM — Studivo Walkthrough

This note uses real Studivo code to explain the **DOM** (Document Object Model): the live tree of
nodes the browser builds from HTML, then lets JavaScript read and change.

Studivo is TypeScript + Next.js + React. That is useful, because the repo already shows the modern
rules you will live with:

- React **owns** most of the tree. Application code almost never calls `document.createElement`;
- `"use client"` files may touch `document` / `window`. Server Components may not;
- when Studivo _does_ reach into the tree, it is for things React does not model well: focus,
  geometry, scroll, observers, hidden file inputs, cookies, print.

Read this file with the cited sources open. The goal is not to memorize method names. The goal is to
predict **which node** a line talks to, **when** that node exists, and **what** reading or writing
it does.

This is Web API, not language. JavaScript does not have a DOM. The browser does.

---

## 1. What the DOM actually is

The browser parses HTML into a tree. Every tag becomes an **element node**. Text becomes a **text
node**. The tree lives on `document`. Scripts read and mutate that tree; the engine then paints.

```js
const input = document.getElementById("phone");
input?.focus();
```

Facts that survive every framework:

1. Nodes are **objects**. `HTMLInputElement` is a node with extra fields (`value`, `files`,
   `focus()`).
2. The tree is **live**. A `querySelectorAll` NodeList from an element reflects later children. A
   React ref points at whatever is currently mounted.
3. There is **one document per browsing context**. An iframe has its own `contentWindow.document`.
4. `window` is the global. `document` is `window.document`. `document.documentElement` is `<html>`.
5. If there is no browser, there is no DOM. Node, `tsc`, and Next.js Server Components do not have
   `document`.

Studivo's UI is JSX. JSX is **not** the DOM. JSX is a description React later commits into the DOM.
You still need the Web API when the product needs the _live_ node.

---

## 2. Server vs browser — there is no `document` on the server

Next.js runs the same file in two worlds. On the server, `window` and `document` are missing.
Reading them throws `ReferenceError`.

### 2.1 Guard the first paint

`useIsMobile` must not crash during SSR. The initial state function runs on the server _and_ on the
client.

```5:12:hooks/use-mobile.ts
export function useIsMobile() {
  const [isMobile, setIsMobile] = React.useState(() => {
    if (typeof window === "undefined") {
      return false
    }

    return window.innerWidth < MOBILE_BREAKPOINT
  })
```

Why this is legal:

1. `typeof window` is safe even when `window` does not exist. It returns `"undefined"`.
2. The server paints `false`. After hydration, the effect below can read `matchMedia` for real.
3. `window.innerWidth` is a Window property, not a DOM node — but it is still a browser API, same
   rule.

The effect that follows only runs in the browser, so it can skip the guard:

```14:21:hooks/use-mobile.ts
  React.useEffect(() => {
    const mql = window.matchMedia(`(max-width: ${MOBILE_BREAKPOINT - 1}px)`)
    const onChange = () => {
      setIsMobile(window.innerWidth < MOBILE_BREAKPOINT)
    }
    mql.addEventListener("change", onChange)
    return () => mql.removeEventListener("change", onChange)
  }, [])
```

`useEffect` is the Studivo default for "this needs the DOM / Window." Effects do not run on the
server.

### 2.2 `"use client"` is necessary, not sufficient

`FadeIn`, `CommandPalette`, lead capture, the gallery — all start with `"use client"`. That only
means the module _may_ run in the browser. The first render of a client component can still happen
on the server. Put DOM reads in effects, event handlers, or `typeof window` guards — not in the
function body of the component.

If I misunderstand it: I write `document.getElementById(...)` at the top of a client component and
the server render throws.

---

## 3. React refs are pointers into the live tree

A ref is not a CSS selector. After React commits, `ref.current` is the **element node** (or `null`
if unmounted).

### 3.1 Hold an input, then focus it

The command palette opens on `Ctrl/⌘+K`. The search field must receive keyboard focus after the
sheet mounts.

```29:29:components/command-palette.tsx
  const inputRef = useRef<HTMLInputElement | null>(null);
```

```67:70:components/command-palette.tsx
  useEffect(() => {
    if (!open) return;
    setTimeout(() => inputRef.current?.focus(), 50);
  }, [open]);
```

```106:112:components/command-palette.tsx
            <Input
              ref={inputRef}
              placeholder="جستجو: صندلی، نام یا شماره"
              value={query}
              onChange={(e) => setQuery(e.target.value)}
              onKeyDown={onKeyDown}
              aria-label="Command palette search"
            />
```

What this teaches:

- `useRef` starts as `null`. The node exists **after** commit, not during the first render of an
  unopened sheet.
- `HTMLInputElement.focus()` is a DOM method. React does not focus because `open` became `true`. You
  ask the node.
- The `50ms` delay is waiting for the sheet to put the input into the document. `focus()` on a node
  that is not connected is a no-op or fights the overlay.

The calendar day button does the same without a timeout, because the button is already in the tree
when `modifiers.focused` flips:

```203:206:components/ui/calendar.tsx
  const ref = React.useRef<HTMLButtonElement>(null);
  React.useEffect(() => {
    if (modifiers.focused) ref.current?.focus();
  }, [modifiers.focused]);
```

### 3.2 Optional chaining is the unmounted-node story

`inputRef.current?.focus()` — if the ref is `null`, skip. That is the DOM equivalent of "the node is
not in the tree." Do not assume a ref is set because the component function ran.

---

## 4. Finding nodes: id, selector, namedItem

When you do not already hold a ref, you ask `document` or a subtree.

### 4.1 `getElementById` + `focus` on the first invalid field

Lead capture validates, then moves the operator's cursor to the broken field. Ids are built from a
prefix so two forms on one page do not collide.

```80:87:lib/marketing-lead-capture.ts
export function getMarketingLeadFieldId(
  idPrefix: string,
  name: MarketingLeadFieldName
) {
  return `${idPrefix}-${name.replace(
    /[A-Z]/g,
    (letter) => `-${letter.toLowerCase()}`
  )}`;
}
```

```81:88:hooks/use-marketing-lead-capture.ts
    const firstInvalidField = Object.keys(errors)[0] as
      | MarketingLeadFieldName
      | undefined;
    if (firstInvalidField) {
      document
        .getElementById(getMarketingLeadFieldId(idPrefix, firstInvalidField))
        ?.focus();
      return;
    }
```

Why `getElementById` and not a ref map:

1. The hook does not render the inputs. The venue form does. The contract is the **id string**.
2. `getElementById` searches the whole `document`. Duplicate ids would return the first match — that
   is why `idPrefix` exists.
3. `?.focus()` — missing node is not an exception. The form still returns; it just does not move
   focus.

`getElementById` is `document`-scoped. There is no `element.getElementById`. For a subtree, use
`querySelector`.

### 4.2 `querySelector` / `querySelectorAll` on a subtree

The public venue gallery does not search `document`. It searches the scroll container it already
holds.

```22:27:app/[slug]/_components/scroll-stack-gallery.tsx
  const updateScrollBounds = useCallback(() => {
    const el = scrollRef.current;
    if (!el) return;

    const cards = el.querySelectorAll<HTMLElement>("[data-gallery-card]");
    if (cards.length === 0) return;
```

```63:70:app/[slug]/_components/scroll-stack-gallery.tsx
  const scrollByCard = (direction: "prev" | "next") => {
    const el = scrollRef.current;
    if (!el) return;
    const card = el.querySelector<HTMLElement>("[data-gallery-card]");
    const gap = 20;
    const distance = (card?.offsetWidth ?? el.clientWidth * 0.78) + gap;
    const delta = direction === "next" ? distance : -distance;
    el.scrollBy({ left: delta, behavior: "smooth" });
  };
```

`querySelector` returns the **first** match or `null`. `querySelectorAll` returns a static
`NodeList` of every match (this API returns a static list in modern browsers). The selector
`[data-gallery-card]` is an **attribute selector**. The JSX puts the marker on each card:

```126:129:app/[slug]/_components/scroll-stack-gallery.tsx
              <div
                key={url}
                data-gallery-card
                className={cn(
```

`data-*` attributes are the DOM's namespaced metadata. In JS they also show up as `element.dataset`,
but Studivo queries the attribute, it does not read `dataset.galleryCard`.

Dots in the gallery jump to a card by index:

```172:175:app/[slug]/_components/scroll-stack-gallery.tsx
              const el = scrollRef.current;
              const card = el?.querySelectorAll<HTMLElement>("[data-gallery-card]")[i];
              card?.scrollIntoView({ behavior: "smooth", inline: "center", block: "nearest" });
```

`HTMLElement.scrollIntoView` is a method on the **node**, not on `window`. `inline: "center"`
matters because this scroller is horizontal.

### 4.3 `form.elements.namedItem` and `instanceof`

After submit, lead capture normalizes the phone in `FormData`, then writes it back onto the actual
input so the user sees `09xxxxxxxxx`.

```54:67:hooks/use-marketing-lead-capture.ts
  function handleSubmit(event: React.FormEvent<HTMLFormElement>) {
    event.preventDefault();

    const form = event.currentTarget;
    const formData = new FormData(form);
    const normalizedPhoneNumber = normalizeMarketingPhoneNumber(
      formData.get("phoneNumber")?.toString() ?? ""
    );
    formData.set("phoneNumber", normalizedPhoneNumber);

    const phoneInput = form.elements.namedItem("phoneNumber");
    if (phoneInput instanceof HTMLInputElement) {
      phoneInput.value = normalizedPhoneNumber;
    }
```

What this teaches:

- `event.currentTarget` is the element the listener is attached to — here, the `<form>`.
  `event.target` might be the submit button. Use currentTarget for the form.
- `HTMLFormElement.elements` is a `HTMLFormControlsCollection`. `namedItem("phoneNumber")` looks up
  the control whose `name` is `phoneNumber`. That is the DOM, not React state.
- The lookup can return an `HTMLInputElement`, a `RadioNodeList`, or `null`.
  `instanceof HTMLInputElement` is the runtime type check the DOM actually gives you.
- `FormData` is a snapshot of named controls. Mutating `formData` does **not** mutate the input. The
  `phoneInput.value = ...` line is the DOM write. `form.reset()` later restores default values.

`ActionForm` is the same snapshot pattern, without writing back:

```49:52:components/action-form.tsx
  function handleSubmit(event: React.FormEvent<HTMLFormElement>) {
    event.preventDefault();
    const form = event.currentTarget;
    const formData = new FormData(form);
```

`preventDefault()` stops the browser's default navigation POST. Studivo sends the `FormData` to a
server action instead. That is the everyday reason almost every Studivo form calls it.

---

## 5. Events: default, bubble, target

The DOM dispatches events along the tree: capture down, target, bubble up. Listeners can cancel the
default action, stop the walk, or both.

### 5.1 `preventDefault` — keep the browser from doing the native thing

| Call site                                       | Default that must not happen   |
| ----------------------------------------------- | ------------------------------ |
| `ActionForm` / login / onboarding / lead submit | Form navigation                |
| Command palette `Ctrl+K`                        | Browser search / location bar  |
| Sidebar `Ctrl/⌘+B`                              | Browser bookmark               |
| Pipeline card `Enter` / `Space`                 | Scroll the page                |
| Dock item `Enter` / `Space`                     | Scroll the page                |
| Settings image `dragover` / `drop`              | Browser opens the file         |
| PWA `beforeinstallprompt`                       | Browser's own install mini-bar |

Command palette:

```53:64:components/command-palette.tsx
  useEffect(() => {
    function onKey(e: KeyboardEvent) {
      const isMac = navigator.platform.toUpperCase().includes("MAC");
      if ((isMac && e.metaKey && e.key === "k") || (!isMac && e.ctrlKey && e.key === "k")) {
        e.preventDefault();
        setOpen((v) => !v);
      }
      if (e.key === "Escape") setOpen(false);
    }

    window.addEventListener("keydown", onKey);
    return () => window.removeEventListener("keydown", onKey);
  }, []);
```

That listener is on **`window`**, not on a node. Key events that do not target an input still reach
`window` by bubbling (and some are fired on the document). `preventDefault` here is what makes `⌘K`
Studivo's shortcut instead of the browser's.

Arrow keys inside the palette also call `preventDefault` so the page does not scroll while the
selected row moves.

### 5.2 `stopPropagation` — keep the parent from seeing the event

A pipeline card is a clickable `article`. The stage-change menu sits inside it. Without stopping the
bubble, opening the menu would also open the lead.

```64:75:app/platform/_components/lead-pipeline-board.tsx
    <article
      aria-label={`پرونده لید ${lead.fullName}`}
      className="po-pipeline-card"
      onClick={() => onOpenLead(lead.id)}
      onKeyDown={(event) => {
        if (event.key === "Enter" || event.key === " ") {
          event.preventDefault();
          onOpenLead(lead.id);
        }
      }}
      role="button"
      tabIndex={0}
    >
```

```100:104:app/platform/_components/lead-pipeline-board.tsx
      <div
        className="po-pipeline-card-actions"
        onClick={(event) => event.stopPropagation()}
        onKeyDown={(event) => event.stopPropagation()}
      >
```

`stopPropagation` does not cancel the menu's own click. It only cuts the walk toward the `article`.
`preventDefault` on `Enter` / `Space` is a different job: the card is `role="button"`, so Space
would otherwise scroll.

### 5.3 `currentTarget` vs `target` vs `files`

File inputs fire `change` on the `<input>`. The file list lives on that element.

```63:76:app/(dashboard)/settings/_components/public-page-settings-form.tsx
  async function handleFile(e: React.ChangeEvent<HTMLInputElement>) {
    const file = e.target.files?.[0];
    if (!file) return;
    // ...
      if (inputRef.current) inputRef.current.value = "";
```

```121:123:app/(dashboard)/profile/_components/profile-settings.tsx
  async function handleAvatarFile(event: React.ChangeEvent<HTMLInputElement>) {
    const file = event.target.files?.[0];
    if (!file) return;
```

`input.value = ""` is a DOM write with a special rule: for `type="file"`, the value is read-only
except you may set it to empty string. That is how Studivo lets the operator pick the **same** file
again. Clearing React state would not be enough; the input still holds the previous `FileList`.

Drag-and-drop is a different event surface. The file list is on `dataTransfer`, not `target.files`:

```468:473:app/(dashboard)/settings/_components/settings-tabs.tsx
        onDragOver={(event) => event.preventDefault()}
        onDrop={(event) => {
          event.preventDefault();
          void handleFiles(event.dataTransfer.files);
        }}
```

`dragover` + `preventDefault` is required. Without it, the browser will not fire `drop`; it will
navigate to the file.

Phone blur writes the normalized string onto the node that blurred:

```49:51:hooks/use-marketing-lead-capture.ts
  function handlePhoneBlur(event: React.FocusEvent<HTMLInputElement>) {
    event.currentTarget.value = normalizeMarketingPhoneNumber(event.currentTarget.value);
    updateFieldError("phoneNumber", event.currentTarget.value);
  }
```

Here `currentTarget` is the input the handler is bound to. Same node as `target` for a direct
listener, but the code is stating the contract: "the element I attached to."

---

## 6. Listening, cleaning up, `passive`

`addEventListener` registers a function on a target (`window`, `document`, or an element). You must
remove the **same function reference** or the listener leaks across navigations.

### 6.1 Window scroll — sticky CTA on the public hall page

```15:21:app/[slug]/_components/sticky-cta-bar.tsx
  useEffect(() => {
    const onScroll = () =>
      setVisible(window.scrollY > window.innerHeight * 0.4);
    onScroll();
    window.addEventListener("scroll", onScroll, { passive: true });
    return () => window.removeEventListener("scroll", onScroll);
  }, []);
```

- `window.scrollY` is the document's vertical scroll offset. The sticky bar appears after ~40% of
  one viewport.
- `{ passive: true }` tells the browser this listener will **not** call `preventDefault`. The
  browser can scroll without waiting. Use it for `scroll` / `touchstart` that only read.
- The effect return is the DOM cleanup. React runs it on unmount. Forgetting it stacks listeners
  every time the venue page remounts.

The gallery listens on **the element**, not on `window`, for `scroll`, and on `window` for `resize`:

```50:60:app/[slug]/_components/scroll-stack-gallery.tsx
  useEffect(() => {
    const el = scrollRef.current;
    if (!el) return;

    updateScrollBounds();
    el.addEventListener("scroll", updateScrollBounds, { passive: true });
    window.addEventListener("resize", updateScrollBounds);
    return () => {
      el.removeEventListener("scroll", updateScrollBounds);
      window.removeEventListener("resize", updateScrollBounds);
    };
  }, [updateScrollBounds]);
```

`el.scroll` does not bubble the way people expect in every browser for every element. Listen on the
scroller you care about.

### 6.2 Geometry: `getBoundingClientRect`, `offsetWidth`, `clientWidth`

`getBoundingClientRect()` returns a `DOMRect` in **viewport** coordinates: `top`, `left`, `width`,
`height`, plus `right` / `bottom`. Layout, not React state.

The gallery finds which card is closest to the container's center:

```29:43:app/[slug]/_components/scroll-stack-gallery.tsx
    const containerRect = el.getBoundingClientRect();
    const containerCenter = containerRect.left + containerRect.width / 2;

    let closestIndex = 0;
    let closestDistance = Number.POSITIVE_INFINITY;

    cards.forEach((card, index) => {
      const rect = card.getBoundingClientRect();
      const cardCenter = rect.left + rect.width / 2;
      const distance = Math.abs(cardCenter - containerCenter);
      if (distance < closestDistance) {
        closestDistance = distance;
        closestIndex = index;
      }
    });
```

The dock uses the same API to measure an item relative to the mouse:

```67:72:components/Dock.tsx
  const mouseDistance = useTransform(mouseX, (val) => {
    const rect = ref.current?.getBoundingClientRect() ?? {
      x: 0,
      width: baseItemSize,
    };
    return val - rect.x - baseItemSize / 2;
  });
```

Useful distinctions Studivo actually relies on:

| Property                  | Meaning                                            |
| ------------------------- | -------------------------------------------------- |
| `getBoundingClientRect()` | Viewport box after transform                       |
| `offsetWidth`             | Layout width including border, integer             |
| `clientWidth`             | Inner width including padding, excluding scrollbar |
| `scrollBy({ left })`      | Move this element's scroll offset                  |
| `window.scrollY`          | Document scroll offset                             |

`scrollByCard` uses `offsetWidth` of one card plus a gap, then `el.scrollBy`. That is layout math on
live nodes.

---

## 7. Observers: the DOM telling you about the tree

Polling `getBoundingClientRect` in a `scroll` listener works. `IntersectionObserver` is the API for
"did this node enter a root?"

`FadeIn` reveals a public-page section once ~12% of it is visible, then **disconnects** — it only
needs to happen once.

```23:38:app/[slug]/_components/fade-in.tsx
  useEffect(() => {
    const node = ref.current;
    if (!node) return;

    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting) {
          setVisible(true);
          observer.disconnect();
        }
      },
      { threshold: 0.12, rootMargin: "0px 0px -40px 0px" },
    );

    observer.observe(node);
    return () => observer.disconnect();
  }, []);
```

What this teaches:

1. Construct the observer, `observe(node)`, disconnect on unmount. Same lifecycle as
   `addEventListener`.
2. The callback receives `IntersectionObserverEntry` objects. `isIntersecting` is the boolean.
3. `threshold: 0.12` — fire when 12% of the target is visible.
4. `rootMargin: "0px 0px -40px 0px"` shrinks the root from the bottom, so the fade happens a little
   before the block hits the very bottom edge.
5. After the first intersect, Studivo disconnects. Leaving the observer running would keep callbacks
   on every scroll for a one-shot animation.

The observed node is the `div` the component renders, held by `ref`. No `querySelector` needed.

---

## 8. Programmatic `click()` on a hidden file input

The visible UI is a button or avatar. The real control is a visually hidden `<input type="file">`.
Operators never see the native control. JavaScript clicks it.

Profile:

```231:264:app/(dashboard)/profile/_components/profile-settings.tsx
                  <button
                    type="button"
                    onClick={() => avatarInputRef.current?.click()}
                    disabled={isAvatarBusy}
                    className="group relative rounded-full ..."
                    aria-label="تغییر تصویر پروفایل"
                  >
                    {/* Avatar ... */}
                  </button>
                  <input
                    ref={avatarInputRef}
                    type="file"
                    accept="image/jpeg,image/png,image/webp"
                    className="hidden"
                    onChange={handleAvatarFile}
                    aria-label="انتخاب تصویر پروفایل"
                  />
```

Public page hero image is the same idea: `inputRef.current?.click()`.

`HTMLElement.click()` synthesizes a click. For `type="file"` that opens the OS picker. It must run
from a user gesture (the button's `onClick`). Calling `click()` from an effect after a timeout is
likely to be ignored.

The hidden input is still in the tree. `className="hidden"` is CSS. `display: none` inputs can still
be clicked from script in this pattern; they just are not shown.

---

## 9. Other `document` / browsing-context APIs Studivo actually uses

These are still DOM / Web API, even when they are not element nodes.

### 9.1 `document.cookie` — sidebar open state

```84:85:components/ui/sidebar.tsx
      // This sets the cookie to keep the sidebar state.
      document.cookie = `${SIDEBAR_COOKIE_NAME}=${openState}; path=/; max-age=${SIDEBAR_COOKIE_MAX_AGE}`
```

Writing `document.cookie` is a setter with special semantics: it **adds or updates one cookie**, it
does not replace the whole cookie string. There is no element involved. The document is the storage
surface.

### 9.2 iframe `contentWindow` — print a contract

Platform contracts render a hidden iframe with its own HTML document (`srcDoc`). Print must target
**that** document, not the app shell.

```70:77:app/platform/_components/contract-view.tsx
  const printFrameRef = React.useRef<HTMLIFrameElement>(null);

  const handlePrint = () => {
    const frame = printFrameRef.current;
    if (!frame || !frame.contentWindow) return;
    frame.contentWindow.focus();
    frame.contentWindow.print();
  };
```

`HTMLIFrameElement.contentWindow` is the iframe's `Window`. `contentWindow.document` would be its
DOM. `print()` is on `Window`. `focus()` first is the usual requirement so the browser prints the
framed document.

Same-origin here because `srcDoc` is created by this page. A cross-origin iframe would throw on
`contentWindow.document` and may restrict `print`.

### 9.3 `window.open` / `window.location` — leave or reload this document

Reserve form opens the device SMS app with a prefilled body:

```456:459:app/(dashboard)/_components/reserve-form.tsx
    window.open(
      `sms:${membership.phoneNumber}?body=${encodeURIComponent(message)}`,
      "_blank",
    );
```

Login assigns the location after success (`window.location.assign("/")`). Error recovery reloads
(`window.location.reload()`). Those replace or reload **this** browsing context's document. They are
Window APIs that destroy or replace the current DOM tree.

---

## 10. Mental model for reading Studivo

When you open a client file, classify each DOM touch:

1. **SSR-safe?** If the line runs during render, it needs `typeof window !== "undefined"` or it must
   live in an effect / handler.
2. **How do I hold the node?** Ref (React gave it to me), `getElementById` (global id contract),
   `querySelector` on a subtree I already hold, `event.currentTarget` / `event.target` (the event
   gave it to me), `form.elements.namedItem` (the form gave it to me).
3. **Read or write?** `getBoundingClientRect`, `scrollY`, `files` are reads. `.value =`, `.focus()`,
   `.click()`, `scrollBy`, `scrollIntoView`, `document.cookie =` are writes.
4. **Default action?** If the browser would navigate, scroll, open a file, or focus the location
   bar, you probably need `preventDefault`.
5. **Bubble?** If a parent also listens for the same event, you may need `stopPropagation`.
6. **Cleanup?** Every `addEventListener` / `observe` in an effect needs a matching remove /
   `disconnect` in the effect return.
7. **Which document?** App shell vs iframe `contentWindow`.

That is why Studivo can:

- skip `createElement` almost everywhere and still be a DOM app;
- put `document.getElementById(...).focus()` in lead validation;
- measure gallery cards with `getBoundingClientRect` instead of storing pixel widths in React state;
- hide a file input and `click()` it from an avatar button;
- print a contract without printing the platform chrome.

---

## 11. Exercises on this repo

Do these in your head (or a scratch file). Do not change production code.

1. In `hooks/use-mobile.ts`, delete the `typeof window === "undefined"` branch. Which render throws,
   server or client, and on which identifier?
2. In `hooks/use-marketing-lead-capture.ts`, replace `event.currentTarget` with `event.target` on
   submit. When does `new FormData(form)` throw, and why?
3. In the gallery, change `el.querySelectorAll("[data-gallery-card]")` to
   `document.querySelectorAll("[data-gallery-card]")`. What breaks if two venues' markup were ever
   on one page? What stays fine today?
4. In `FadeIn`, remove `observer.disconnect()` inside the callback but keep the effect cleanup. What
   extra work happens on scroll after the first reveal?
5. In profile settings, skip `avatarInputRef.current.value = ""` after a successful upload. Pick the
   same file again. Does `onChange` fire? Why?
6. In the pipeline card, remove `stopPropagation` on the actions `div`. Click the stage menu. Which
   two handlers run, in what order?
7. In `sticky-cta-bar.tsx`, drop `{ passive: true }` and add `onScroll.preventDefault()`. What does
   the browser do with a `scroll` listener that tries to cancel? (Predict, then check MDN: `scroll`
   is not cancelable.)
8. In `contract-view.tsx`, call `window.print()` instead of `frame.contentWindow.print()`. What
   document gets the print dialog?

---

## Files cited

| File                                                                 | Why it is here                                                      |
| -------------------------------------------------------------------- | ------------------------------------------------------------------- |
| `hooks/use-mobile.ts`                                                | No `window` on the server; `matchMedia`                             |
| `components/command-palette.tsx`                                     | `window` keydown, ref, `focus()`                                    |
| `components/ui/calendar.tsx`                                         | Focus a mounted button via ref                                      |
| `lib/marketing-lead-capture.ts`                                      | Stable element ids                                                  |
| `hooks/use-marketing-lead-capture.ts`                                | `getElementById`, `FormData`, `namedItem`, `instanceof`, blur write |
| `components/action-form.tsx`                                         | `preventDefault` + `FormData` from a form                           |
| `app/[slug]/_components/scroll-stack-gallery.tsx`                    | Subtree query, `data-*`, geometry, `scrollBy`, `scrollIntoView`     |
| `app/[slug]/_components/fade-in.tsx`                                 | `IntersectionObserver`                                              |
| `app/[slug]/_components/sticky-cta-bar.tsx`                          | `window.scrollY`, passive listener, cleanup                         |
| `app/platform/_components/lead-pipeline-board.tsx`                   | `preventDefault` vs `stopPropagation`                               |
| `app/(dashboard)/profile/_components/profile-settings.tsx`           | Hidden file input, programmatic `click()`, reset `.value`           |
| `app/(dashboard)/settings/_components/public-page-settings-form.tsx` | Same file-input pattern                                             |
| `app/(dashboard)/settings/_components/settings-tabs.tsx`             | `dragover` / `drop` / `dataTransfer`                                |
| `components/Dock.tsx`                                                | `getBoundingClientRect` for pointer math                            |
| `components/ui/sidebar.tsx`                                          | `document.cookie`                                                   |
| `app/platform/_components/contract-view.tsx`                         | iframe `contentWindow`                                              |
| `app/(dashboard)/_components/reserve-form.tsx`                       | `window.open`                                                       |

When you pick the next Web API topic (events in depth, `fetch`, Storage, Service Worker, Canvas, …),
hunt Studivo, then write the next note beside this one.
