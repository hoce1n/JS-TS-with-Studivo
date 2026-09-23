# JS-and-Studivo

Learn JavaScript by reading a real product: [Studivo](https://github.com/hoce1n/studivo).

This repository used to be **JS-and-Programming** — a first-principles map from computational
problems to JavaScript features. That map still matters. What changed is the method.

I have started diving deep into JavaScript many times and given up halfway. Syntax-first study does
not stick. Abstract mental models without a codebase also do not stick. This time every topic is
studied in two passes:

1. **Concept** — what problem the language is solving, independent of any app.
2. **Studivo** — where that same behavior shows up in a multi-tenant study-hall product: seats,
   memberships, staff, finance.

> The question is no longer only “What is this syntax?” It is “Where does this actually happen in
> Studivo, and what would break if I misunderstood it?”

## Why Studivo

Studivo is not a toy. It is production software:

- App: [app.studivo.ir](https://app.studivo.ir)
- Site: [studivo.ir](https://studivo.ir)
- Source: [github.com/hoce1n/studivo](https://github.com/hoce1n/studivo)

Next.js, TypeScript, Prisma, and PostgreSQL sit on top of JavaScript. The language still decides how
names resolve, when values exist, how work is scheduled, and how failures propagate. Studying those
rules against Studivo keeps the learning attached to code I already care about.

## Two Layers

| Layer             | Role                                                                 | Lives in  |
| ----------------- | -------------------------------------------------------------------- | --------- |
| **Foundations**   | First-principles notes: problem, computational need, concept, syntax | `notes/`  |
| **Topic studies** | One JavaScript topic, explained, then proven with Studivo examples   | `topics/` |

Foundations answer _why the language has this idea_. Topic studies answer _where I have already used
it, often without noticing_.

The original progression is unchanged:

| Layer                  | Guiding question                                                 |
| ---------------------- | ---------------------------------------------------------------- |
| **Problem**            | What real-world or programming difficulty needs to be addressed? |
| **Computational Need** | What capability must a computer have to address that difficulty? |
| **Concept**            | What general programming idea satisfies that need?               |
| **Language Feature**   | How does JavaScript model or implement the concept?              |
| **Syntax**             | How is that feature expressed in JavaScript code?                |
| **Studivo**            | Where does this appear in the Studivo repository, and why?       |

## How a Topic Is Studied

1. Pick one JavaScript topic (hoisting, closures, `this`, promises, …).
2. Write or receive a conceptual note for that topic.
3. Search the Studivo repo for matching patterns — not toy snippets, production code.
4. Compile those findings into a documented Markdown file under `topics/`.
5. Verify surprising claims in the personal REPL at
   [runtimejs.hoce1n.ir](https://runtimejs.hoce1n.ir/).

Reusable shapes live in `templates/`. Polished writing can later move to `drafts/medium/` or
`drafts/instagram/`.

## Progress

| Topic                        | Status                  | Notes / study                |
| ---------------------------- | ----------------------- | ---------------------------- |
| **State & Memory**           | Foundations in progress | `notes/01-state-and-memory/` |
| **Hoisting**                 | Documented              | `topics/01-hoisting/`        |
| **Control Flow**             | Not started             | —                            |
| **Data Organization**        | Not started             | —                            |
| **Abstraction**              | Not started             | —                            |
| **Composition**              | Not started             | —                            |
| **Errors & Failure**         | Not started             | —                            |
| **Concurrency & Asynchrony** | Not started             | —                            |
| **Communication & I/O**      | Not started             | —                            |
| **Identity & Scope**         | Not started             | —                            |
| **Execution Runtime**        | Not started             | —                            |

Hoisting is the first topic study: `topics/01-hoisting/01-javascript-hoisting.md`.

## Repository Structure

```text
.
├── notes/                 # First-principles conceptual notes
│   └── 01-state-and-memory/
├── topics/                # JS topics grounded in Studivo
│   └── 01-hoisting/
├── drafts/
│   ├── instagram/
│   └── medium/
├── templates/
│   ├── concept-template.md
│   └── topic-study-template.md
└── README.md
```

## References

- [Studivo source](https://github.com/hoce1n/studivo)
- [MDN JavaScript](https://developer.mozilla.org/en-US/docs/Web/JavaScript)
- [runtimejs](https://runtimejs.hoce1n.ir/)
