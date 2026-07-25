---
name: python-house-rules
description: Use for any Python code change, review, refactor, debugging task, test
  update, or pyproject/tooling edit when the user's standing Python rules should
  be applied consistently across repositories, including Django, DRF, Celery,
  migrations, testing, typing, exceptions, logging, and service structure.
---

# Python House Rules

Apply these rules on every Python task unless the user explicitly overrides them.
When choosing between two valid Python approaches, prefer the approach documented
here. When reviewing code, flag deviations with a short explanation.

These rules take precedence over generic Python/Django style skills when both are
relevant.

## Reference Files

This skill is split for progressive disclosure. The core (this file) always
applies. Before writing or reviewing code, **read the reference file(s) that match
the task** — they hold the detailed rules and the calibration examples:

- `references/python-style.md` — **any non-trivial Python.** PEP 8, typing, style,
  good/bad examples, formatting, imports, naming, control flow, strings/comments,
  logging, exceptions, classes/services, API clients, defensive parsing,
  concurrency, and non-Django service layout.
- `references/django.md` — **Django / DRF / Celery work.** App layout, models,
  querysets/ORM, enums, forms, admin, celery/tasks, signals, migrations & system
  checks, settings, DRF/viewsets, throttling/authz/flags, management
  commands/middleware.
- `references/testing.md` — **writing or reviewing tests.** Testing principles,
  what *not* to test (shape/config/tautology anti-patterns), and Django test
  patterns.

When unsure, read `python-style.md` first. For a Django change that also adds
tests, read all three.

## First Pass

Before editing:

- Read the repo's `AGENTS.md` if present.
- Check `pyproject.toml`, formatter config, lint config, pytest config, and type-checker config.
- Match the existing package style and architecture before introducing a new pattern.
- Discover the package layout, dependency direction, and local conventions before editing.

If repo-local rules conflict with these house rules, follow the stricter or more
specific rule and mention the conflict briefly.

## The Short List

Thirteen rules that decide most reviews. Apply them while writing, not after.

1. **Bare `return`, never `return None`.** `None` is a return *value* only when the
   caller distinguishes it from absence.
2. **Test truthiness, not `is None`.** `if items:` — spell out `is None` only where
   `0`, `""`, `[]`, or `{}` are meaningful values distinct from absence.
3. **Don't guard values your own code produced.** Validate at the trust boundary —
   external input, user data, a third-party response — and past it, read the value.
   Prefer one declarative filter at the boundary over an `if` at the top of every
   consumer.
4. **No data model without a boundary to justify it.** Define a schema where the
   shape is untrusted or contractual: external input, a response you publish, an
   LLM's structured output. Everywhere else pass dicts, arguments, and rows.
5. **Comments describe the end state.** No note about what the code used to do, no
   "previously", no pointer at adjacent code. What survives is a *why* the code
   cannot show.
6. **Spell names out** (`workflow`, not `wf`). No helper with one caller — inline it.
7. **Application code calls the framework's public surface, never its internals.**
   Reaching for a framework's persistence, event, or lifecycle primitive from
   application code means bypassing the seam that exists for it.
8. **Derive state from its owner; never stamp a second copy.** Whoever owns the
   fact — the upstream service, the lifecycle, the webhook — is its only writer.
9. **Pass identity, not a pre-assembled payload.** A capable consumer fetches its
   own data and writes its own output. Threading denormalized metadata through the
   call chain to save it the trip couples both sides to a shape neither owns.
10. **One noun per concept.** Inputs carry identity; state carries what was learned.
    Never compose a bag that is both.
11. **Failure raises a typed error; it never returns a sentinel.** No `Value | str`
    error union, no `X | None` standing in for ok/fail, no `isinstance` on the error
    arm at the call site.
12. **Keep the wire boundary thin.** A single-field request body is one embedded
    field, not a one-field model. A response object is a projection over data handed
    to it — the caller does the fetching; a schema method never queries.
13. **Write the positive condition.** `if value: do` — not `if not value: return`
    followed by the happy path a single positive branch could hold. Flip it and let
    the miss fall through. Negative guards stay for raises and genuine multi-exit
    chains.

## Core Philosophy

- Code should read like prose written by someone who already understands the domain.
  If a reader needs your explanation to follow it, the code is wrong: fix the code
  and delete the explanation.
- Minimize the time the next reader spends building a mental model.
- Names carry the meaning. A function's name plus its signature should make its body
  predictable before you read it. Name things for what they ARE in the domain's
  words, never for their mechanism, their pattern, or their position in a pipeline.
  If you cannot name it cleanly, you do not understand the concept yet — stop and
  re-derive it.
- One idea per function, at one altitude. Policy sitting next to plumbing is the tell.
  Behavior lives on the type that owns the data, not in a helper module.
- Shape functions top-down: the happy path is the spine, exits are early, nesting is
  shallow, and the ending is the interesting case.
- Prefer no new surface. A parameter or an inline beats a new function; a new function
  beats a new class; a new class beats a new package. Cheap to write is not a reason
  to exist.
- Prefer clean current design over compatibility layers unless compatibility is explicitly required.
- Abstractions earn their place by being used three times or more; before that, duplication is often cheaper than indirection.
- Build only what the task asks for. Do not add config knobs, options, alternate code paths, or speculative resilience that were not requested — unused flexibility is complexity no one is paying for. When a "while I'm here" extension is tempting, leave it out (or raise it separately).
- Guard clauses live at the top of functions; main logic lives in the middle. A guard
  earns its place by raising or by opening a genuine second path — not by wrapping the
  one branch rule 13 would have you write positively.
- Keep functions focused and side effects explicit.
- Reuse existing project patterns before introducing new abstractions or dependencies.
- Be skeptical of generic abstraction layers that only move data around; keep abstractions only where there is real behavioral variation.
- Prefer explicit configuration and visible setup over import-time side effects.
- Prefer explicit wire contracts over defensive parsing for owned service contracts.
- Do not invent resilience for undocumented alternate payload shapes unless the task explicitly requires it.
- Prefer domain-owned path builders and identifiers over settings-driven path glue.
- Avoid projection or mirror data models unless there is a hard product requirement for separate copies.
- Keep I/O, subprocess, network, and database boundaries easy to spot.

## Public Surfaces

When the code is something other people will call — a library, an SDK, a plugin API,
a published client — the surface is the product, not the plumbing. Design it by
writing the example first: the obvious call is the correct one, the correct one is
short, and a newcomer gets it right without reading the source or the docs. No
exposed internals, no required boilerplate, no ceremony, no knowledge of framework
mechanics leaking into caller code. If the example needs a paragraph of setup or a
caveat, the surface is wrong — redesign it.

## Debugging Rules

- Do root-cause analysis before fixing a bug.
- Do not patch symptoms if the underlying cause is still unclear.
- When the cause is inferred rather than proven, say so explicitly.
- Keep fixes tightly scoped to the root cause.

## Verification

- Prefer the smallest useful verification set first, then broaden if risk is high.
- Run repo-standard checks when feasible and report exactly what ran.
- Typical Python verification is `uv run pytest`, `uv run ruff check`, and formatter check such as `uv run ruff format --check` or the repo's configured equivalent.
- Do not run formatters or linters in rewrite mode unless implementation work calls for it and the user has not restricted edits.
- For Django behavior changes, include migrations checks and focused tests when relevant.
- Never pipe a test run that decides whether you commit. Run it as its own command so
  you read the real exit status, not the tail of a pipeline.
- After any scripted or bulk edit, grep to confirm it landed.

## Before You Commit

Re-read The Short List against **every file the change touches, end to end — not the
diff**. A smell on an untouched line in a file you edited is yours the moment you
edit that file. Make the pass mechanical: grep the touched files for the tells.

- `from … import` inside a `def` — a function-level import with no real cycle to break
- `isinstance(` applied to data your own code produced
- `-> … | str` or `| None` used as an ok/fail union
- `class X(BaseModel)` with a single field
- `def _helper` with one call site
- a query (`.get(`, `.all(`, `session`) inside a schema or DTO method

Then read the files once more as prose. If a line makes you wince, that is the
finding — fix it, or say it out loud in your final response.

## Decision Defaults

- Greenfield project work can prefer clean breaking changes over compatibility layers when the user has not requested compatibility.
- Preserve backwards compatibility only when the task, repo, or product requirement calls for it.
- Prefer incremental patches over large rewrites unless a cleaner design clearly requires a wider change.
- Leave concise comments only where the code would otherwise be hard to follow.
- Prefer upstream-system state of record over local mirror state. If GitHub/Linear/Slack can store the flag, label, or field that answers "has X happened?", use it. Local DB rows are for state we genuinely own: audit, retries, in-flight work. Do not maintain a hash/snapshot/cached-projection of upstream content just to compute "has this changed" — let the upstream system own that.
- Avoid intermediate projection types between an upstream response and the place that uses the data. A `LinearIssueSnapshot` dataclass built from a Linear GraphQL dict, then consumed by functions that read the same fields, is one layer of indirection too many. Operate on the upstream dict directly or model it as a typed object that lives at the boundary.

## Final Response Expectations

In final responses for Python work:

- Lead with what changed.
- Mention verification status.
- Call out important assumptions.
- Surface risks or follow-ups if something could not be fully validated.
- Keep the response short and useful.
