# C2Y Migration — All-in-One Structured Document

This document bundles four deliverables you requested:

* **Migration checklist for developers**
* **Compiler implementer checklist**
* **MISRA and CERT-C compliance impact table**
* **Side-by-side annotated PDF diff references**

Sources:

* N3220 (C23 baseline): [https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3220.pdf](https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3220.pdf)
* N3685 (C2Y draft):  [https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3685.pdf](https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3685.pdf)

# 1. Migration Checklist for Developers

## Overview

Purpose: give developers a practical, prioritized list of actions to make codebases ready for C2Y (draft N3685) while preserving compatibility with C23 (N3220).

### Priority levels

* **P0** — Must do before shipping (high risk / breaking)
* **P1** — Strongly recommended before wide adoption
* **P2** — Nice-to-have, improve readability/maintenance

### Checklist (CSV-friendly rows shown below)

| Priority | Area        | Item                                                                | Action                                                                      | Tooling / Notes                                   |
| -------: | ----------- | ------------------------------------------------------------------- | --------------------------------------------------------------------------- | ------------------------------------------------- |
|       P0 | Syntax      | Legacy leading-zero octal literals (e.g. `0755`)                    | Replace with `0o755` or decimal; mark occurrences in code review            | Regex: `\b0[0-7]{2,}\b` ; run automated scanner   |
|       P0 | Types       | `_Imaginary` usage                                                  | Replace with `_Complex` or implement wrapper types                          | Search for `\b_Imaginary\b`                       |
|       P0 | Parsing     | Numeric parsers relying on leading-zero octal recognition           | Audit parsing code, if base==0 ensure explicit semantics or restrict input  | Unit tests for edge cases like `"077"`, `"0o77"`  |
|       P0 | Build       | `__COUNTER__` usage in handwritten macros                           | Prohibit in source; allow only in auto-gen with reproducible build controls | Add linter rule                                   |
|       P1 | Strings     | Legacy octal escape sequences in string literals                    | Review; prefer braced escapes only where supported and documented           | `\\[0-7]{1,3}` detection                          |
|       P1 | Libraries   | Replace custom `strnlen`-like helpers with standardized `strnlen()` | Refactor and add unit tests                                                 | Feature-detection macros for freestanding targets |
|       P1 | CI          | Add C2Y-aware compiler flags and static-analysis checks             | Add matrix for `-std=c2y` (or vendor equivalent)                            | Track toolchain adoption                          |
|       P2 | Readability | Adopt `0o` consistently and document in style guide                 | Update coding standards and examples                                        | Update code review checklist                      |

### Suggested automated scans (tools & commands)

* **Legacy octal detection**: `grep -RIn --exclude-dir={.git,build} -E '\b0[0-7]{2,}\b' *.c *.h`
* **Imaginary type detection**: `grep -RIn '\b_Imaginary\b' *.c *.h`
* **Deprecated escapes**: `grep -RIn -E '\\\\[0-7]{1,3}' *.c *.h`
* **Macro scan**: `grep -RIn '__COUNTER__' *.c *.h`

### Quick CI additions

* Add compile jobs using latest GCC/Clang with `-std=c2y` (or `-std=gnu2x` if vendor) and warnings:

```
-Wall -Wextra -Wpedantic -Wdeprecated -Wliteral-suffix
```

* Static analysis: `clang-tidy`, `clang-analyzer`, `cppcheck`, or vendor tools.
* Sanitizers: Address UB with `-fsanitize=undefined,address` in regression runs.

# 2. Compiler Implementer Checklist

Goal: give compiler and toolchain maintainers a concise recipe for adding C2Y features while retaining C23 compatibility.

## Implementation priorities (short-term → long-term)

### Short-term (must implement / fix before user adoption)

* **Lexical updates**

  * Recognize `0o` / `0O` as octal literal prefix in integer literal tokenizer.
  * Update string escape parsing to accept braced escapes (`\x{..}`, `\u{..}`, `\o{..}`) per N3685 grammar.
  * Continue to accept legacy leading-zero octal for backward compatibility but emit `-Wdeprecated` / `-Wconversion` warnings by default if obsolescence policy applies.

* **Preprocessor**

  * Implement `__COUNTER__` macro semantics: an expanding integer that increments each expansion.
  * Ensure macro expansion reproducibility across translation units documentation.

* **Parser & semantic analysis**

  * Prepare to reject `_Imaginary` declarations (or provide a warning mode then error mode depending on compatibility policy).
  * Add parse support for proposed constructs in draft (e.g., case ranges, optional loop labels) under feature flags.

### Medium-term (enhanced correctness and optimizations)

* **Constant expression evaluation**

  * Strengthen compile-time constant evaluation for new literal forms and complex literals if implemented.

* **Floating point & math library conformance**

  * Update math library wrappers to comply with refined rounding and FP exception semantics; add targeted tests.

* **Atomic & memory model**

  * Align relaxations in atomic alignment rules; ensure frontend/back-end agree on representation and ABI.

### Long-term (developer ergonomics & standard evolution)

* Add `-std=c2y` mode and warn/error levels for obsolescent features.
* Update diagnostics to include suggestions for migration (e.g., “use 0oNNN instead of leading-zero octal”).

## Test cases & validation

* Extend compiler test-suite with cases derived from N3220 and N3685 examples.
* Add cross-ABI tests for zero-length memcpy/memmove on null pointers if draft allows.
* Add fuzz harnesses for lexer/tokenizer to ensure new escape forms don't regress existing behavior.

## Documentation for users

* Migration guide: list of incompatible changes and recommended code transforms.
* Feature gate flags: `-fno-c2y-deprecations`, `-fallow-legacy-octal` for transitional uses.

# 3. MISRA and CERT-C Compliance Impact Table

This table maps notable C2Y proposals to likely impacts on MISRA-C and CERT-C guidance.

| C2Y Proposal                                  |                                                                                                          MISRA Impact |                                                              CERT-C Impact | Recommended Rule/Action                                                               |
| --------------------------------------------- | --------------------------------------------------------------------------------------------------------------------: | -------------------------------------------------------------------------: | ------------------------------------------------------------------------------------- |
| `0o` octal prefix; implicit octal obsolescent |                                                 Positive: encourages explicit notation; legacy octal likely forbidden |                 CERT: warn on ambiguous notation; recommend explicit parse | Ban legacy leading-zero octal; allow `0o` with style rule                             |
| Braced escape sequences (`\x{}`, `\u{}`)      |                                      Requires lexing/encoding rule updates; MISRA may restrict use to documented code |              CERT: accept but require validation in input/output functions | Define approved escape styles in coding standard                                      |
| `_Imaginary` removal                          |                                                              MISRA: neutral/positive (discouraged constructs removed) |                                                              CERT: neutral | Replace with `_Complex`                                                               |
| `__COUNTER__` preprocessor macro              | High impact: macros generating identifiers conflict with deterministic builds; MISRA generally forbids complex macros |                    CERT: flagged for maintainability and security concerns | Forbid in hand-written code; allow only for auto-generated code under strict controls |
| `strnlen()` standardization                   |                                                                                          Low impact; positive (safer) |                                     Positive; reduces buffer overrun risks | Prefer `strnlen()` over `strlen()` in untrusted contexts                              |
| Zero-length ops on null pointers allowed      |                                           Medium impact: MISRA typically avoids UB reliance; clarifications reduce UB | CERT: reduces a class of UB-related vulnerabilities if properly documented | Ensure tests for zero-length memory ops; document behavior for portability            |
| Switch ranges (case 1...4)                    |                                                       MISRA may disallow novel syntactic sugar that can obscure logic |                             CERT: potential for off-by-one bugs if misused | If accepted, require explicit comments and unit tests for ranges                      |

### Practical recommendations

* Update MISRA deviation records where necessary and include migration rationale.
* Update CERT-C checklists to include checks for `0o` usage and `__COUNTER__` avoidance.
* Add new static-analysis rules to detect disallowed patterns and produce actionable messages.

---

# 4. Side-by-side Annotated PDF Diff References

This section provides a compact annotated diff map linking relevant clauses in N3220 (C23) and N3685 (C2Y draft). Use these references to guide precise wording changes and test-case generation.

> **How to use:** for each entry below, locate the clause in N3220 and the corresponding (or new) clause in N3685; add test-cases that exercise the behavior and verify compiler diagnostics.

## 4.1 Lexical grammar (literals & escapes)

* **N3220**: See Lexical elements / 6.4 Integer constants / Character constants sections. (search `integer constant`, `escape sequence`)
* **N3685**: See updated Lexical grammar where `0o` prefix and braced escapes are introduced. Tests:

  * Tokenize `0o755`, `0755`, `0x1F`, ensure tokens recognized and warning for legacy octal.
  * Parse `"\x{41}"` and `"\123"` verify both produce 'A' but braced form preferred.

## 4.2 Type system

* **N3220**: clauses on arithmetic types, complex, imaginary.
* **N3685**: search for `_Imaginary` removal, complex literal proposals. Tests:

  * Attempt to compile `float _Imaginary a;` — expect deprecation warning or error depending on implementation flag.
  * Ensure `_Complex` support unchanged.

## 4.3 Preprocessor behavior

* **N3220**: defined macro list; no `__COUNTER__` standard.
* **N3685**: `__COUNTER__` addition in the preprocessor section. Tests:

  * `#define ID __COUNTER__` used across expansions yields incrementing numbers.
  * Reproducibility: ensure value sequence deterministic per translation unit.

## 4.4 Library changes

* **N3220**: standard library headers and function definitions.
* **N3685**: additions like `strnlen`, changes to `strtol` behavior for `0o` prefix. Tests:

  * `assert(strnlen("abc", 2) == 2);`
  * `assert(strtol("0o77", NULL, 0) == 63);` (if implemented)

## 4.5 Undefined behavior clarifications

* **N3220**: baseline UB descriptions.
* **N3685**: proposals to define certain previously UB behaviors (e.g., zero-length memops). Tests:

  * `memcpy(NULL, NULL, 0);` ensure it either compiles and runtime is allowed (documented) or flagged if not supported.

## 4.6 Grammar-cleanup and obsolescence sections

* **N3220**: existing obsolescent features list.
* **N3685**: extended obsolescence including legacy octal. Tests:

  * Build with `-std=c2y -Wdeprecated` and ensure legacy octal emits specific diagnostics.

# Appendix: Example transformation snippets

## Replace legacy octal with `0o` (automated using `sed`)

```bash
# naive: replace leading-zero octal like 0755 -> 0o755 (ensure not to replace 0x.. or single 0)
sed -E -i 's/\b0([0-7]{2,})\b/0o\1/g' $(git ls-files '*.c' '*.h')
```

## Sample unit test for `strtol` new behavior

```c
#include <assert.h>
#include <stdlib.h>
int main(void) {
    long v = strtol("0o77", NULL, 0);
    assert(v == 63);
    return 0;
}
```

# Closing notes & next steps

* The content here is intended as an actionable package for teams moving from C23 to C2Y drafts. Because N3685 is a draft, treat changes as *proposals* and maintain a compatibility mode in toolchain and project policy.
* If you want, I can now:

  * Export this document as a PDF and attach it to the canvas, or
  * Create a GitHub repo with the scanner, CI templates and sample tests, or
  * Generate the JSON machine-readable diff of features.

Choose: **Export PDF / Create Repo / JSON diff / All**.

# Reference

N3220andN3685.md
