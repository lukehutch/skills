---
name: deep-code-review
description: Checklist for reviewing or auditing code beyond "does it compile and do the tests pass" — bugs, structural and architectural issues, API inconsistencies, behavioral oddities, wrong comments and docs, wrong assumptions, and surprises for callers, each with a concrete failure scenario and a proposed fix. Use whenever asked to review, audit, or critique code, a diff, a PR, or a branch.
---

# Deep code review

When asked to review or audit code, do not stop at "does it compile and do the tests pass". Read the code for all of the following, and report each with the evidence that supports it:

- **Bugs** — wrong results, unhandled cases, off-by-one, resource leaks, concurrency hazards, silently swallowed failures.
- **Structural, architectural and conceptual issues** — a responsibility in the wrong place, a leaky or redundant abstraction, a layering violation, two mechanisms doing the same job, a design that will not extend where it obviously needs to.
- **API inconsistencies** — the same idea named or shaped differently in different places, asymmetric pairs (an `enableX` with no `disableX`), inconsistent nullability, argument order, or return types, a public method returning an internal type.
- **Behavioral oddities** — surprising defaults, order dependence, silent truncation or coercion, an error path that changes behavior rather than reporting.
- **Wrong or incomplete comments and docs** — anything that names an identifier that no longer exists, describes behavior the code does not have, or omits a constraint the caller has to obey.
- **Wrong assumptions** — about the platform, the JDK version, the filesystem, encodings, locales, the environment, or what a caller will pass.
- **Code that likely does not line up with users' expectations** — behavior that is defensible but will surprise the people who call it, whatever the docs say.

Give every finding a concrete failure scenario (the input or state, and the wrong result it produces), and propose the fix.
