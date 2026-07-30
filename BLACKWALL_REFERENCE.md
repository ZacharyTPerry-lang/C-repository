# Blackwall — Compile-Time Macro Isolation: Reference Documentation

**Formal name:** `compile_time_macro_isolation`
**Informal name:** Blackwall (used in prose only, never in structural or folder contexts)
**Document version:** 1.0 (reference-grade)
**Supersedes:** `blackwall_specification.md` v0.1.0 (2026-03-07, pre-implementation) and `compile_time_macro_isolation_specification.md` v0.3.0 (mid-implementation cliff-notes)
**Status of the system:** Feature-complete and runnable end to end. 16 production modules, all strict-clean under `-std=c99 -Wall -Wextra -Werror -pedantic`. 8 unit suites and 3 acceptance tests green. ~4,076 lines of production C.

---

## How to read this document

This is written to be read the way you'd read a compiler's internals documentation: front to back it builds an argument, but each section is also a reference you can jump to. The order is deliberate:

1. **What and why** — the problem, and why it is not solvable with ordinary C hygiene.
2. **The governing idea** — the single principle every mechanism descends from.
3. **The empirical foundation** — the six primitives, each a tested fact about GCC, not an assumption.
4. **The proofs** — the two load-bearing arguments (reverse-dependency by complement; the token-paste biconditional) that let the tool claim completeness.
5. **The architecture** — the 16 modules, what each does, and which are critically load-bearing.
6. **The mechanisms in depth** — collision detection, the State Gate, State A's safety analysis, the independent validator, the build pipeline. Each with the *why*, not just the *what*.
7. **How it is used** — the developer's actual workflow and the whitelist language.
8. **Holes, limits, and honest failure modes** — what it does not do, where it errs, and why those choices are correct.
9. **Divergence from the original design** — what changed since v0.1.0 and why the change was forced.

A note on epistemics that pervades the whole system: **the compiler is the single source of truth.** Nothing in this tool predicts what GCC *will* do; everything reads what GCC *did*. When you see a claim like "the vendor uses this macro," it means a specific GCC stage output said so — not that a heuristic guessed it. This is the thread that ties every design decision together, and it is why the tool can make strong soundness claims about a language as adversarial to static reasoning as the C preprocessor.

---

## 1. What this is and why it exists

### 1.1 The problem in one sentence

The C preprocessor has one flat, global, mutable symbol table with no scopes and no namespaces, so a `#define` anywhere in a translation unit silently rewrites every matching token after it — including tokens in code that a *different author* wrote and never intended to expose.

### 1.2 Why this is dangerous specifically

Most bugs announce themselves. A macro collision does not. Consider the canonical case, which the tool's test suite reproduces against real GCC:

```c
/* vendor/vendor.h */
#define index vendor_internal_index

/* your code */
#include "vendor/vendor.h"
static int lookup(int index) { return index + 1; }
```

Run this through GCC's preprocessor and the output is:

```c
static int lookup(int vendor_internal_index) { return vendor_internal_index + 1; }
```

Your parameter named `index` — a perfectly ordinary local name — was renamed out from under you. The code still compiles. It may still pass tests. If `vendor_internal_index` happens to collide with *another* symbol, or if the vendor's macro expands to something with side effects, the failure surfaces far from the cause, weeks later, as data corruption with no diagnostic. This is the failure mode the whole system exists to make impossible.

### 1.3 Why ordinary hygiene does not solve it

The instinct is "just prefix all your macros" or "just use include guards." Neither is sufficient, and understanding *why* is the key to understanding why this tool is shaped the way it is:

- **Prefixing protects your macros, but not your identifiers.** You can name your macro `MYPROJECT_INDEX`, but you cannot rename the parameter `index` in a function signature to `MYPROJECT_index` without making your own code unreadable — and even if you did, the vendor could define *that*. The set of ordinary identifiers you use (parameters, locals, struct members) is unbounded and not under a prefix discipline. A vendor macro can capture any of them. This asymmetry — macros are protectable, identifiers are not — is the reason the tool must reason about *identifiers* and not just macros.

- **Include guards are cooperative.** `#ifndef X / #define X / #endif` only protects `X` if the *redefining* party checks first. A bare `#define` from a vendor overwrites whatever came before, guard or no guard. You cannot lock a macro in standard C.

- **The last definition always wins, silently.** GCC warns on redefinition-with-different-value, but always accepts it. There is no mechanism to make a definition authoritative.

So the problem is real, common (any project vendoring header-only libraries hits it), and not fixable by convention. It needs a *tool* that (a) knows every macro the vendor actually defines, (b) knows every identifier the project actually uses, and (c) can prove, per build, that no unsanctioned collision between the two can occur. That tool is Blackwall.

### 1.4 What it does for you, concretely

- **It refuses to let a vendor macro silently capture your code.** An unguarded collision stops the build with a message naming the exact symbol, before a single object file is produced.
- **It lets you deliberately share a symbol across the boundary when you want to**, in a direction you choose, and it enforces that choice.
- **It catches the reverse coupling too** — when a vendor's compiled object secretly depends on one of *your* symbols, which the linker will bind whether you intended it or not.
- **It does all of this by asking the compiler what actually happened**, so its verdicts are as trustworthy as GCC itself, and it never modifies vendor source.
- **It is free at runtime.** Everything happens at build time; the resulting binary is byte-for-byte what you'd get without the tool. Only the macro environment during compilation differs.

---

## 2. The governing idea

Everything below descends from one principle, stated two ways:

> **Convenience is the abstraction of understanding.**
> **The compiler is the single source of truth; read what it did, never predict what it will do.**

The first is a design ethic: abstraction is admitted only where technically necessary, boundaries are exposed rather than hidden, and the tool never trades understanding for ergonomic shortcuts that would make its behavior opaque. The second is an epistemic discipline: the C preprocessor is too dynamic to reason about by static inspection of source in the general case (macros expand transitively, paste tokens together, and gate code on conditions), so instead of *modeling* the preprocessor, the tool *runs* it and reads the results.

This second principle is what makes the tool sound. Every place where a naive implementation would guess — "does the vendor use this macro?", "what does this macro expand to?", "does this vendor object need a project symbol?" — Blackwall instead consumes a specific GCC or `nm` output that answers the question factually. The recurring lesson from the tool's own development (documented in §8.5) is that every attempt to *reason about* the preprocessor from the outside was falsified by a case that reasoning missed, and every mechanism that *asked the compiler* survived.

---

## 3. The empirical foundation — six primitives

These are not axioms in the sense of "assumed true." Each was verified by running GCC/`nm`/`ld` and observing the result, and each is re-verified by the tool's tests. If GCC's documented behavior changed, these would need re-verification — but they are stable, documented behavior.

### P1 — Identifier capture is real, and prefixing cannot stop it

A vendor `#define index vendor_internal_index` rewrites a project's `index` parameter at preprocessing time (demonstrated in §1.2). Prefixing protects project *macros* but is powerless for project *identifiers*, because identifiers are not and cannot be under a prefix discipline. **This is the tool's entire reason to exist**: the one thing convention cannot fix.

### P2 — Per-translation-unit state selection causes silent ODR divergence

If you let different translation units select different macro states, you can produce a single binary in which the *same* vendor inline function was compiled at two different macro values, with the linker silently picking one. This is undefined behavior with no diagnostic. **Consequence:** the State Gate is *one build-wide choice*, never per-file and never per-vendor. This is why `ISOLATION_STATE` is a single Makefile variable, not a per-source annotation.

### P3 — Only file-scope external-linkage names cross the link boundary

`nm` classifies symbols: `T`/`D`/`B` are defined with external linkage, `U` is undefined (demanded). Parameters and block-locals get *no* linker symbol at all. **Consequence:** reverse-dependency detection needs only the file-scope symbol facts that `nm` computes soundly — but collision detection needs *more* than `nm` can see (it must catch capture of a parameter like `index`, which has no symbol), which is why the tool has *two* identifier sources (§4, `symbol_set_from_object` vs the lexer).

### P4 — `gcc -dM -E` inherits the vendor's own includes

When you dump a vendor header's macros with `gcc -dM -E`, GCC resolves the vendor's own `#include`s first, so the dump cannot miss a macro the vendor pulls in from its internal headers. **Consequence:** the vendor macro table is complete by construction, without the tool having to chase includes itself.

### P5 — Post-preprocessing token presence reveals internal consumption

After full preprocessing, whether a name still appears as a token in the output tells you whether the vendor consumed it internally. **Consequence:** the static macro-table check can be *confirmed* by the compiler's actual output — the two-layer design of §5.2.

### P6 — GCC internals are disqualified as a data source; stage outputs are not

GCC's internal AST is body-centric, lossy, and unstable across versions. The tool consumes *stage outputs* — `-E` text, `-dM`/`-dU` dumps, `nm` symbols — never internal state. This is the same disqualification applied to libclang, which is admitted for *verification only*, never as a production dependency. **Consequence:** the tool is deterministic and version-robust in a way that a plugin reading compiler internals could never be.

---

## 4. The two load-bearing proofs

Two arguments carry the system's soundness claims. Everything else is machinery; these are the reasons the machinery is *sufficient*.

### 4.1 Reverse dependencies, by complement

**The claim:** the tool finds *every* case where a vendor object demands a project symbol, with no false negatives.

**The argument.** There are exactly two possibilities, and they are exhaustive: either the vendor references a project symbol, or it does not. Rather than trying to enumerate references (which requires understanding vendor code), the tool tests the complement:

```
U ∩ P = ∅ ?
```

where **U** = the vendor object's undefined symbols (`nm`, selector `U`) and **P** = the project's defined-external symbols (`nm`, `T`/`D`/`B`). This is computed in `boundary_crossing_discovery.c` by walking U and keeping every name present in P — a literal set intersection.

- If `U ∩ P = ∅`: the vendor demands nothing the project defines. Isolation is safe.
- If `U ∩ P ≠ ∅`: each symbol in the intersection is a genuine reverse dependency. The linker *will* bind it (P3). Each must be declared in the whitelist or the build errors.

**Why there are no false positives or coincidences:** at link scope there is one global namespace, so a name in both U and P *is* the same symbol — the linker binds it by name. There is no "coincidental" collision to filter out. The answer is *computed from compiler outputs*, not *guessed from source*.

**Why there are no false negatives:** U is computed by `nm` from the actual compiled vendor object, so it cannot miss a reference the vendor really makes. Incompleteness could only *inflate* U (e.g., if a symbol is undefined for an unrelated reason), which errs toward "declare or error" — a loud failure, never a silent capture. **The failure mode is always safe.**

### 4.2 The token-paste biconditional

**The apparent gap:** token pasting (`##`) constructs names that are not literally in any macro body. `#define MAKE(x) project_ ## x` builds `project_helper` only when invoked as `MAKE(helper)`. The constructed token `project_helper` appears in no macro table. How can a static tool claim to catch it?

**The resolution — a biconditional, verified against GCC:**

> A pasted collision exists in the compiled output **⟺** the macro fired **⟺** the invocation (with its argument) is present in the code we compile **⟺** we tokenize that code and see the constructed token.

Read the chain in both directions:

- **If the paste never fires** with the constructing argument, the collision never occurs — there is nothing to catch, because the token is never built.
- **If the paste does fire**, the invocation is by definition present in the source being compiled, so the constructed token appears in the compiler's preprocessed output, where tokenizing it reveals the collision.

**"Execution is complete where enumeration is not."** You cannot enumerate every token a paste *could* build (the argument space is unbounded), but you do not need to: the compiler builds exactly the tokens that *actually* occur, which is precisely the set that matters. This is why the collision check has two layers (§5.2): a static layer that catches everything enumerable from the macro table, and a post-preprocessing layer that catches constructed tokens by reading GCC's real output. Between them, **every collision that can occur in the build is seen.**

The static layer additionally over-approximates pasting without waiting for the invocation, via affix reasoning (§5.3), so that many constructed captures are caught early with a specific diagnostic rather than only at the post-preprocessing stage.

---

## 5. Architecture — the sixteen modules

The tool is built leaf-to-orchestrator. Every module is a `.h` contract (with five-term parameter annotations, §7.3), a `.c` implementation, and a `_test.c` unit test, each compiling clean under the strict flag set. Dependencies point strictly from leaves toward the orchestrator; there are no cycles.

```
DEPENDENCY ORDER (leaves first)

lexical_scanner                    ── pure leaf, no project deps
symbol_set_from_object             ── wraps nm
parameter_contract_system/
    parameter_contract_vocabulary  ── grammar as data
    parameter_contract_recognizer  ── depends on vocabulary + lexer
    parameter_contract_translator  ── depends on recognizer
boundary_crossing_system/
    boundary_crossing_vocabulary   ── crossing kinds as data
    boundary_crossing_enforcement  ── depends on vocabulary
    boundary_crossing_discovery    ── depends on symbol_set_from_object
compile_time_token_rewriter/
    vendor_macro_table             ── wraps gcc -dM -E
    project_macro_set              ── depends on lexer
    project_identifier_set         ── depends on lexer
    collision_detection            ── depends on all the above sets
    compile_time_token_rewriter    ── ORCHESTRATOR
compile_time_macro_isolation/
    state_crossing_generation      ── State A/B generation + gcc -dU gate
    state_validation               ── INDEPENDENT verifier
```

### 5.1 The load-bearing modules (read these first)

Three modules carry disproportionate weight. If you read only three, read these:

**`lexical_scanner`** — the foundation everything textual stands on. It is a boundary-exact C99 tokenizer with a crucial design choice: it is *keyword-free and grammar-free*. It classifies bytes into coarse categories (identifier, number, string/char literal, single-byte punctuator, whitespace, comment, preprocessor-line, invalid) and nothing finer. It does not know that `int` is a keyword, and it emits `>>=` as three separate punctuator tokens, because combining punctuators into operators is a *grammatical* act that depends on a particular C standard's operator table — knowledge the scanner deliberately refuses to hold.

*Why this matters:* a pure leaf with no grammar is *write-once*. Grammar ages — new operators, new keywords — but a byte-classifier does not. When a consumer needs multi-byte operators or keyword awareness, it reconstructs them downstream, where that knowledge belongs. This is the "convenience is the abstraction of understanding" principle applied at the lowest level: the scanner classifies, it does not interpret. Its invariant is total coverage — every byte of input belongs to exactly one token, and concatenating all token ranges reproduces the input byte-for-byte. This total-coverage guarantee is what lets every downstream pass trust that it is seeing all the source.

**`collision_detection`** — the largest module (~28 KB of source) and the heart of the static safety check. It takes the vendor macro table, the project identifier set, the project macro set, and the whitelist, and produces a *sorted report* of every collision. For each vendor macro it checks the macro's name and every identifier in its expansion against project space, and additionally performs `##` affix reasoning (§5.3). Each collision is sorted into one of four dispositions (§5.4). Its correctness is the correctness of the static half of the paste biconditional (§4.2).

**`compile_time_token_rewriter`** (the orchestrator) — composes the leaves into the actual transformation and runs the two jobs (§5.5). It is where the pieces become a tool. It also contains the emit logic, including the subtle directive-prefixing step (§5.6) that a real end-to-end compile forced into existence.

### 5.2 Collision detection is two layers, not one

The static layer (`collision_detection`) catches every collision *enumerable from the macro table*: name collisions, single- and multi-token expansion collisions, transitive chains (via the fully-expanded macro table). What it *cannot* enumerate is tokens constructed only at expansion — the paste case. Those are caught by the **post-preprocessing verification**, which tokenizes GCC's actual `-E` output; by the biconditional (§4.2), a token that is never constructed is a collision that never fires. The two layers together are complete. This module is the static layer; the post-preprocessing layer is realized as the validation pass (§5.8) run in with/without-boundary mode.

### 5.3 The `##` affix reasoning (static over-approximation)

The static layer does not wait for a paste to fire. A macro body containing `PREFIX ## x` can construct *any* project identifier that starts with `PREFIX`; `x ## SUFFIX` any that ends with `SUFFIX`. So the tool takes each *fixed* (non-parameter) affix adjacent to a `##` and asks the project identifier set whether any member starts-with (respectively ends-with) that affix. `symbol_set_from_object` provides `any_starts_with` (binary search, since the set is sorted) and `any_ends_with` (linear scan) for exactly this.

This is **sound** (no false negatives — every project id the paste could build is caught) and **over-checking** (it may flag a paste that never actually fires with the constructing argument, which the whitelist then resolves). Parameter affixes are *excluded*, because a parameter is a placeholder replaced by the argument, not a fixed prefix — a subtlety that, when first omitted, caused builtin macros like `__INT64_C(c) → c ## L` to false-positive against any project name starting with `c`. Excluding parameter affixes fixed it while preserving soundness.

### 5.4 The four dispositions

Every detected collision is sorted, so the caller knows how to treat it:

| Disposition | Meaning | Response |
|---|---|---|
| `WHITELISTED` | Declared to cross under the State Gate | Permitted; the crossing is sanctioned |
| `IDENTIFIER_CAPTURE` | Collides with a project identifier, not whitelisted | **Error** — prefixing cannot fix an identifier (P1) |
| `MACRO_SHADOW` | Collides with a project macro, not whitelisted | Informational — the prefix resolves it (the project reference gets prefixed, the vendor macro does not) |
| `CONSTRUCTED_CAPTURE` | A `##` paste could construct a project identifier | **Error** — same class as identifier capture |

Reverse dependencies are *not* sorted here; they are a separate discovery-and-enforcement path (§5.7). This module concerns macro-level collisions only.

### 5.5 The orchestrator's two jobs

`compile_time_token_rewriter_run` executes, in order:

1. **Job 2 — reverse-dependency enforcement.** Compile the project sources to obtain P, walk the vendor root for vendor objects (prebuilt `.o` or `.c` compiled on demand), discover `U ∩ P` per vendor object (§4.1), and enforce each discovered crossing against the whitelist. An undeclared *compelled* crossing is a build error naming the symbol. (It runs first because it is the cheapest way to fail a fundamentally broken build.)

2. **Job 1a — collision guard.** For each project source, run collision detection against every vendor header's macros. Any `IDENTIFIER_CAPTURE` or `CONSTRUCTED_CAPTURE` fails the run with a diagnostic naming the symbol.

3. **Job 1b — translate.** For each source, emit a transformed file into the output tree with project macros prefixed and contract annotations stripped — valid C for GCC to compile. The tool does *not* compile; GCC does, from the output tree.

The naming ("Job 1"/"Job 2") reflects that collision prevention is the primary purpose and reverse-dependency enforcement is the secondary guarantee, though enforcement runs first operationally.

### 5.6 The directive-prefixing subtlety (a real-integration lesson)

When the orchestrator prefixes a project macro, it must prefix *both* the macro's definition and its uses, in step, or the output references an undeclared name. But the lexer emits a `#define` directive as a *single opaque preprocessor-line token* (directives are not C grammar — a correct choice for the pure leaf). So the ordinary identifier-prefixing path never sees the macro name *inside* its own `#define` line.

The fix keeps the leaf pure and puts the knowledge where it belongs: the orchestrator, which already understands macros (it parses directive tokens to *collect* the macro set), also handles the directive on *emit* — it scans inside a `#define` token for the defined name and prefixes it there too. This is symmetric with how the macro set is collected. It was surfaced only by compiling the tool's actual output with GCC (§8.5), which is why the pipeline acceptance test now compiles the transformed tree as its decisive assertion.

### 5.7 Reverse-dependency discovery and enforcement

`boundary_crossing_discovery` computes `U ∩ P` (§4.1). `boundary_crossing_enforcement` parses the whitelist (declarations of the form `KEYWORD(symbol)`), looks up each discovered crossing's *kind* in the vocabulary, reads that kind's *modality*, and applies the modality rule: a `CHOSEN`-modality crossing absent from the whitelist falls to the default (isolation); a `COMPELLED`-modality crossing absent from the whitelist is an error. The reverse-dependency kind (`VENDOR_LINKS_PROJECT`) is `COMPELLED`, so an undeclared reverse dependency stops the build.

### 5.8 The independent validator (`state_validation`)

This module deserves special attention because its *independence* is a hard requirement, not a nicety. It verifies that a transformation produced the behavior its state defines — but it does so by a route that *shares no logic with the transformer*. If the validator asked the transformer "what did you do?", a bug in the shared logic would pass both the transform and its own check; the validator would inherit the transformer's blind spots and be worthless as a check.

Instead, the validator derives its verdict from the *compiler's actual output*, checked against each state's *defined behavior*:

- **State C:** preprocess the project source; assert the project identifier survives unchanged (no capture).
- **State B:** independently read the vendor's value from the vendor header's own `gcc -dM` dump, read what project code sees from `gcc -E`, assert they match.
- **State A:** read the vendor-space value from the preprocessed wrapper output, assert it equals the project value, not the vendor's original.

Crucially, the validator catches *violations*, not just confirms successes — its tests prove it detects a wrong State B value (99 ≠ vendor's 32) and a failed State A supersede (a wrapper missing its redefine). The transformer and validator share only the *compiler*, which is the neutral oracle. This makes agreement between them real evidence and makes the validator a genuine check rather than theater. (It also subsumes what would otherwise be a separate "rewriter self-verification" module: "no miss, no overzealous action" is exactly what the with/without-boundary comparison detects when checked against the state spec.)

### 5.9 The supporting modules, briefly

- **`symbol_set_from_object`** — wraps `nm -P`, extracting P (defined-external) and U (undefined) as sorted sets, and provides membership, enumeration, and the prefix/suffix affix queries the paste reasoning needs. Also offers a public builder (begin/add/finalize) for non-`nm` population. It is sound where a hand-rolled parser is not: it correctly handles comma-lists, function-pointer names, and internal-linkage exclusion, because `nm` does.
- **`vendor_macro_table`** — wraps `gcc -dM -E`, capturing each vendor macro's name, full expansion, and function-like flag. Complete by P4.
- **`project_macro_set`** — the set of names the project `#define`s (the names to prefix), extracted by finding `#define` directives in the lexer's preprocessor-line tokens.
- **`project_identifier_set`** — *every* identifier the lexer emits from project source (source-scoped), which catches parameters and locals like `index` that `nm` cannot see (P3). This is the collision layer's identifier source, distinct from `nm`'s linkage-scoped P.
- **`parameter_contract_{vocabulary,recognizer,translator}`** — the parameter-contract annotation language (§7.3): a five-axis grammar defined purely as data, a recognizer with zero hardcoded tokens (extensibility proven by test — adding a token to the data extends the language with no code change), and a translator that strips the annotation and emits the fifth (diagnostic) axis according to build mode (dropped in release, kept in debug).
- **`boundary_crossing_vocabulary`** — the crossing kinds as data (§5.7), with modality and discovery mechanism per kind. Adding a crossing kind is a data row, not an engine change.

---

## 6. The State Gate in depth

One build-wide state (P2) governs how *whitelisted* symbols cross the boundary. Non-whitelisted vendor macros never cross; that is the default. The universal prefix, defined identically in both the rewriter and the generator, is:

```
COMPILE_TIME_ENFORCEMENT_FRAMEWORK_
```

chosen long enough to be globally unique (no vendor or system header will define a symbol starting with it) and tab-completable so its length costs the developer nothing. It also makes the codebase greppable: `grep COMPILE_TIME_ENFORCEMENT_FRAMEWORK_` returns every project macro.

### 6.1 State C — Full Isolation (the default)

Nothing crosses. Project code uses its own names and prefixed names; vendor macros never enter a translation unit holding project identifiers. A project identifier that would collide survives untouched, and the project's own macros are prefixed into the project namespace. This is the common case and needs no generated header — it is the *absence* of a crossing. Demonstrated: `index` (a parameter) survives, `MY_BUFFER_SIZE` (a project macro) becomes `COMPILE_TIME_ENFORCEMENT_FRAMEWORK_MY_BUFFER_SIZE`, validator confirms `HOLDS`.

### 6.2 State B — Vendor Supersedes

You want the vendor's value, under your prefixed name, without the vendor header entering your TU. `state_crossing_generation` reads the vendor's value via `gcc -dM -E` and writes a generated header:

```c
#define COMPILE_TIME_ENFORCEMENT_FRAMEWORK_BLAKE3_OUT_LEN 32
```

Your code includes the *generated* header (never the vendor's) and reads the prefixed name. GCC confirms it resolves to the vendor value. State B is **always safe**: it introduces a new prefixed name and changes nothing the vendor sees. Demonstrated: `my_hash_size = 32`, validator confirms vendor's value (32) equals what the project sees (32).

### 6.3 State A — Project Supersedes (the "lazy path," made robust)

You want *your* value to win in vendor space, so vendor code compiles against your definition. `state_crossing_generation` generates a wrapper that includes the vendor header, then `#undef`s the macro and redefines it to the project value:

```c
#include "vendor/lib.h"
#undef BUFFER_LIMIT
#define BUFFER_LIMIT 64
```

Vendor code compiled *through this wrapper* sees 64. Crucially, **the developer writes no guard by hand** — the transformation manufactures it. This is the "lazy path" the design deliberately pursued. But it is only *safe* under a condition, and that condition is what §6.4 is about.

### 6.4 State A's safety condition — the crux, and why `-dU` is load-bearing

**The hazard.** If the vendor uses the macro *in its own declarations* — say `static char buf[BUFFER_LIMIT]` or `int fill(char out[BUFFER_LIMIT])` — those declarations fixed the *old* value (32) at the point they were preprocessed, *before* the wrapper's `#undef` runs. So vendor code sees 32 and your code sees 64: the same symbol, two values, one binary. This is the ODR-class split-brain of P2, reappearing inside a single translation unit. Auto-superseding here would be silently wrong — exactly the failure the tool exists to prevent.

**The safety condition, stated exactly:** auto-superseding a vendor macro is safe **if and only if the vendor does not use that macro itself.**

**How the condition is decided — soundly, by the compiler.** The tool reads `gcc -dU`, which lists every macro the preprocessor *expanded or whose definedness it tested* during preprocessing. A macro *absent* from `-dU` output was defined but never used → **safe** (`DEFINE_ONLY`). A macro *present* was used somehow → **unsafe** (`SELF_USED`), and generation returns `UNSAFE_SUPERSEDE`, writing nothing, requiring the developer to define the override explicitly.

**Why `-dU` specifically, and nothing weaker.** This is the single most important robustness result in the system, and it was reached by *falsifying* three plausible alternatives (§8.5 has the full history):

- A **source scan** for the macro name outside its `#define` line misses *transitive* use: `#define SZ BUFFER_LIMIT` then `buf[SZ]` — `BUFFER_LIMIT` appears only on its own `#define` line, so a source scan calls it "define-only" and would auto-guard it, splitting the value.
- A **sentinel-value grep** (define the macro to a unique value, preprocess, look for the value in the output) misses `#if` use: `#if BUFFER_LIMIT > 16` *consumes* the value into a condition; it never appears as a token, so the grep says "safe" wrongly.
- A **two-value diff** (preprocess under two different values, diff the outputs) misses *exact-match* `#if`: `#if BUFFER_LIMIT == 32` — any two probe values other than 32 both take the `#else` branch, outputs identical, "safe" wrongly. And you cannot pick probe values that trip an *unknown* predicate; value-probing is fundamentally unsound here.

`-dU` catches *all* of these — direct, transitive, function-like, and every `#if` form including `== 32` — because it is not a probe that can miss an unknown predicate; it is *the compiler's own record of what it read*. Its only imperfection errs on the **safe** side: a macro mentioned only inside a string literal may be reported used, which asks the developer to be explicit unnecessarily — never the silent false-safe that matters. This is P6 in action: ask the compiler what it did, don't model what it might do.

Demonstrated on both paths: a `DEFINE_ONLY` vendor macro auto-guards and the project value 64 wins (validator `HOLDS`); a `SELF_USED` vendor macro returns `UNSAFE_SUPERSEDE` and writes nothing.

### 6.5 The modality axis (why reverse dependencies are not a fourth state)

The State Gate is one axis (which value crosses). *Modality* is a separate axis (what an absent declaration means), and conflating them was an early error the design corrected:

- **CHOSEN** — the developer selects; absence falls to a default. The three states live here: a symbol may cross with the project value, the vendor value, or not at all, and silence means isolation.
- **COMPELLED** — the toolchain proves it; absence is an error. A reverse dependency lives here: the linker binds it regardless, so silence is a build failure.

A reverse dependency is *not* a fourth state — it is a crossing of a different *modality*. This is why the enforcement engine reads a `modality` column from the vocabulary and branches, rather than having separate engines: adding a crossing kind is a data row, and the engine does not grow a case (`boundary_crossing_vocabulary.c` is literally a three-row table).

---

## 7. How it is used

### 7.1 The developer's workflow

1. **Write ordinary project source.** Use whatever identifiers you like. Prefix your macros or don't — the tool prefixes project macros for you during translation.
2. **Vendor your dependencies** under a `vendor/` root. The path *is* the boundary: everything under the vendor root is vendor, everything else is project. (Location = ownership.)
3. **Choose one build-wide state** via the `ISOLATION_STATE` Makefile variable (`A`, `B`, or `C`; `C` is the default and the common case).
4. **Declare any deliberate crossings** in the whitelist (§7.2). Most projects have a handful of entries or none.
5. **Build.** The Makefile runs the three-stage pipeline (§7.4). If there is an unguarded collision or an undeclared reverse dependency, the build stops with a message naming the symbol. Otherwise you get compiled objects and analysis reports, and the `.intermediate/` tree is the exact source GCC compiled.

You never hand-write guards, never modify vendor source, and never reason per-vendor about whether superseding is safe — the tool decides that from `-dU`.

### 7.2 The whitelist language

The whitelist is data: declarations of the form `KEYWORD(symbol)`, one per crossing. The keywords are exactly the crossing kinds in the vocabulary:

```
PROJECT_VALUE_SUPERSEDES(BLAKE3_OUT_LEN)   /* State A: project value into vendor space */
VENDOR_VALUE_SUPERSEDES(BLAKE3_KEY_LEN)    /* State B: vendor value into project space (prefixed) */
VENDOR_LINKS_PROJECT(project_allocator)    /* reverse dep: vendor object may bind this project symbol */
```

A symbol absent from the whitelist does not cross (State C default) if its modality is CHOSEN, or errors the build if its modality is COMPELLED (a reverse dependency you failed to acknowledge). The whitelist is the *single source of truth* for what crosses; it contains names and directions only, no logic.

### 7.3 The parameter-contract annotation language (adjacent, optional)

The tooling also carries a parameter-contract annotation system used in the project's own headers. Each parameter is annotated with a five-term grammar, always in order, always all five present:

```
STATE  ACTION  CONTRACT  CONTROL  LAYER_TWO_DIAGNOSTIC
```

with `NONE` the explicit default for the fifth term. The legal tokens per axis are pure data (`parameter_contract_vocabulary.c`): STATE ∈ {VALID, GARBAGE}; ACTION ∈ {CONSUME, POPULATE, MUTATE, FORCE, COERCE, INVOKE, VERIFY}; CONTRACT ∈ {ALWAYS, NEVER, IF_NONZERO, …} (the `IF_*` forms take a peer-parameter argument); CONTROL ∈ {KEEP, GIVE, NONE}; LAYER_TWO_DIAGNOSTIC ∈ {NONE, PROVENANCE, TRANSIT, ANOMALY, SHADOW}. Axes 1–4 are compile-time documentation for the analyzer and are stripped before GCC sees the code; axis 5 drives runtime diagnostic instrumentation and compiles out entirely in release builds. Extending the language is a data edit — the recognizer has zero hardcoded tokens, proven by a test that adds a token and observes it recognized.

### 7.4 The build pipeline

`source_transformation_pipeline.makefile` wires three stages, with the root Makefile as the single authority:

```
1. transform : source/*.c        → .intermediate/*.c       (the rewriter)
2. compile   : .intermediate/*.c  → build/*.o               (GCC)
3. analyze   : .intermediate/*.c  → analysis/*.report       (static analyzer)
```

Compile and analysis both consume `.intermediate/`, so both see the same transformed source under the same configuration (this is the soundness move that makes analysis configuration equal to build configuration).

Three Make mechanics are load-bearing and were each verified against real `make`:

- **`.intermediate/` is a real tracked output, not `.PHONY`.** Directory creation uses *order-only* prerequisites (`| $(INTERMEDIATE_DIR)`) so the directory existing does not force a rebuild.
- **`.SECONDARY` retains the intermediates.** Without it, Make deletes "chained intermediate" files after the build, which would force a full re-transform every time and break the analyzer's ability to read the tree. `.SECONDARY` keeps them while leaving them fully tracked (unlike `.PHONY`).
- **The vendor macro table is a build-wide prerequisite** of every transform. So touching a vendor header re-transforms *all* sources (the macro environment changed), while touching one project source re-transforms *only* that file. This is exactly the correct incremental behavior.

`ISOLATION_STATE` is validated inline (`$(error …)` if not A/B/C) — no silent acceptance of a bad state.

---

## 8. Holes, limits, and honest failure modes

This section is deliberately unsparing. A reference document that hides a system's limits is worse than useless.

### 8.1 Toolchain dependence

The tool depends on GCC (`-dM -E`, `-dU`, `-E`) and `nm`. These are the mechanisms by which it reads the compiler's truth; they are not incidental. Porting to a toolchain without equivalents (or with different flags — MSVC's `/EP`, `/FI`, etc.) requires changes to the query wrappers (`vendor_macro_table`, `state_crossing_generation`, `symbol_set_from_object`) and the Makefile, though not to the core logic. This is an accepted, documented dependency, not a hole — but it is a constraint.

### 8.2 State A on self-using vendor macros is refused, not solved

When a vendor uses a macro in its own declarations, State A *cannot* safely supersede it (§6.4), and the tool refuses with `UNSAFE_SUPERSEDE`. This is the correct behavior (refusing is safe; auto-guarding would be silently wrong), but it *is* a limit: you cannot get project-value-wins for such a symbol via the lazy path. You must define the override explicitly and accept responsibility for the consequences. This is a genuine boundary of what auto-generation can safely do, and the tool draws it honestly rather than papering over it.

### 8.3 The post-preprocessing verification layer is designed but run via the validator, not as a standalone always-on pass

The paste biconditional's runtime half (§4.2, §5.2) is realized by running the validator in with/without-boundary mode against the compiler's `-E` output. The static affix reasoning (§5.3) catches most constructed captures early. The two together are complete in principle. What is *not* yet built is a standalone, always-on post-preprocessing scanner wired as a mandatory pipeline stage — it exists as a validation capability rather than an unconditional gate. For the common cases the static layer suffices; the completeness argument holds; but a paranoid deployment would want the post-preprocessing scan wired in unconditionally, and that wiring is future work.

### 8.4 The `-dU` string-literal over-report

`-dU` may report a macro as "used" when the only occurrence is inside a string literal (where it does not actually expand). This causes State A to refuse a supersede it could technically have allowed. This is a *deliberately accepted* imperfection because it errs on the safe side — a false "unsafe" costs a manual override; a false "safe" would cost a silent split-brain. Documented, not fixed, because fixing it would trade a safe error for a risk.

### 8.5 The development history is itself a hole-finding record

The tool's development repeatedly demonstrated that reasoning *about* the preprocessor from outside is unsound, and each such attempt was caught only by testing against GCC before building on it. Concretely, within this system's construction:

- Three separate State-A detection methods were **falsified** before `-dU` was adopted (source scan → misses transitive; sentinel grep → misses `#if`; two-value diff → misses `#if ==`).
- The command-line `-U`/`-D` override was reached for **three times** to try to beat a header's own `#define`, and failed every time (a header's textual `#define` is later, so it wins). The durable lesson: never override the preprocessor from the command line; scan source with the lexer *or* read the compiler's actual output.
- A real end-to-end compile surfaced two bugs unit tests missed: the `--output` file-vs-directory mismatch, and the define/use prefixing inconsistency (§5.6). Both were invisible until GCC actually compiled the transformed output.

This is not a defect list — it is evidence for *why the design is shaped as it is*. Every one of these was a case where the "ask the compiler" discipline won and the "reason about it" shortcut lost. The tool is sound because it was built by trying to break it and keeping only what survived.

### 8.6 What the tool explicitly does not attempt

- It does not parse C. It classifies tokens and reads compiler outputs. It has no C grammar and no type system, by design.
- It does not evaluate macro values (beyond reading them as text from `-dM`). It does not need to.
- It does not handle a project that *wants* per-TU state divergence — that is disqualified by P2 as inherently unsafe, not merely unsupported.
- It does not protect against a vendor that is actively adversarial in ways beyond macro definition (e.g., a vendor that ships a `.o` deliberately crafted to mislead `nm`) — it trusts the toolchain's own outputs, which is the correct trust boundary.

---

## 9. How this diverged from the original design, and why

The pre-implementation spec (v0.1.0) and the built system share a purpose, a prefix, and the three-state concept, but the *mechanism* changed substantially. The changes were forced by implementation reality, and understanding them clarifies the final design.

**v0.1.0 was header-and-flag driven; the built system is a tokenizing transformer.** The original plan used state *header files* that redefined a `BLACKWALL_PERMIT` macro, a hand-authored whitelist *header*, sentinel `#define`s marking code ownership, and `-imacros` injection. The built system instead has a real tokenizer, a transformation pass producing an `.intermediate/` tree, generated crossing headers, and a whitelist *data file*. The shift happened because the header-macro approach could not *detect* collisions — it could only *arrange* macros and hope. Detecting that a vendor macro captures a project *identifier* (P1) requires tokenizing project source and comparing identifier sets, which a header of `#define`s cannot do. The tool had to be able to *see* the source, so it grew a lexer.

**State A's "known limitation" became a decided, mechanical behavior.** v0.1.0 documented that State A "requires vendor cooperation" (guarded `#define`s) and told the developer to *manually verify* guard usage and document it. The built system *mechanizes* that verification with `-dU` and makes the tool decide: auto-guard where the vendor is define-only, refuse where it self-uses. What was a manual caveat became an automatic, sound decision. This is the single largest advance over the original design, and it is what makes the lazy path (auto-generated guards) actually safe.

**Reverse dependencies were absent from v0.1.0 entirely.** The original spec's Axiom 3 asserted "project code never needs to define into vendor code" and handled the *forward* configuration-flag case with the whitelist. It did not consider the *reverse* — a vendor object demanding a project symbol at link time. The built system adds the entire `boundary_crossing_*` subsystem and the U ∩ P proof (§4.1) to handle a coupling the original design did not see. This is new capability, not a refinement.

**Sentinels were dropped.** v0.1.0 wrapped every code block in `CODE_ACTIVE`/`VENDOR_CODE_ACTIVE` sentinels for human readability and future tooling. The built system does not use them: the boundary is defined by *path location* (vendor root vs. project source), which is unambiguous and needs no in-source marking, and the tool reasons about ownership from the filesystem rather than from sentinels in the token stream. The sentinels were solving a problem (boundary identification) that path-based ownership solves more cleanly.

**The independent validator is new, and embodies a principle the original lacked.** v0.1.0 had "Fail Visible" via `-Werror=macro-redefined`. The built system adds a *separate verifier* that independently confirms each state's behavior from compiler output, sharing no logic with the transformer (§5.8). This is a stronger guarantee: not just "collisions that escape become compile errors," but "an independent process confirms the transform did exactly what its state defines, and catches it when it didn't."

In short: the original was a sound *arrangement* strategy; the built system is a sound *detection-and-verification* strategy. The purpose held; the mechanism was rebuilt around the discovery that the only trustworthy way to reason about the preprocessor is to ask the compiler.

---

## 10. Invariants (the rules the system never violates)

1. **One build-wide state.** Never per-TU, never per-vendor (P2).
2. **Vendor source is never edited.** Generated wrappers and prefixed headers do the work; the vendor header is included, never modified.
3. **Using a vendor is not a violation.** The include happens. An *unguarded* collision is the error, not the use of a library.
4. **No silent failure.** Ambiguity, an unsafe supersede, or an undeclared compelled crossing stops the build with a named cause.
5. **Every decision traces to a compiler stage output.** `-dM`, `-dU`, `-E`, `nm`. Never GCC internals, never a guess.
6. **Validation is independent of transformation.** No shared logic; the verdict comes from compiler output against the state spec.
7. **Determinism is non-negotiable.** Any mechanism that assumes code shape or reads compiler internals is disqualified from the production pipeline.

---

## Appendix A — Complete failure-mode table

| Situation | Detected by | Mechanism | Response |
|---|---|---|---|
| Vendor macro name captures project identifier | `collision_detection` (static) | name vs project identifier set | Error, `IDENTIFIER_CAPTURE`, names symbol |
| Vendor macro expansion hits project symbol | `collision_detection` (static) | expansion tokens vs project space | Error, `IDENTIFIER_CAPTURE` |
| `##` paste could construct project identifier | `collision_detection` (static) | fixed-affix prefix/suffix match | Error, `CONSTRUCTED_CAPTURE` |
| Vendor macro shadows project macro | `collision_detection` (static) | name vs project macro set | Informational; prefix resolves |
| Constructed token collides only at expansion | validation pass (post-preprocessing) | tokenize `gcc -E` output | Caught by biconditional |
| Vendor object demands project symbol | `boundary_crossing_discovery` | `U ∩ P` from `nm` | Error unless whitelisted (`COMPELLED`) |
| State A supersede on vendor-used macro | `state_crossing_generation` | `gcc -dU` use check | Error, `UNSAFE_SUPERSEDE` |
| Whitelist references unknown crossing kind | `boundary_crossing_enforcement` | keyword lookup in vocabulary | Error (malformed whitelist) |
| Malformed vendor header | GCC (during any query) | tool query fails | Surfaced as tool failure, not papered over |
| Wrong state behavior in output | `state_validation` | compiler output vs state spec | `VIOLATED`, names observed vs expected |

## Appendix B — Module map with sizes and roles

| Module | Role | Load-bearing? |
|---|---|---|
| `lexical_scanner` | Boundary-exact C99 tokenizer, grammar-free | **Critically** — foundation of all textual reasoning |
| `symbol_set_from_object` | `nm` wrapper: P/U sets, affix queries | Yes — the link-scope truth source |
| `vendor_macro_table` | `gcc -dM -E` wrapper: vendor macros | Yes — the macro truth source |
| `project_macro_set` | Project `#define` names | Supporting |
| `project_identifier_set` | Every project identifier (source-scoped) | Yes — catches identifier capture (P1) |
| `collision_detection` | The static collision sort | **Critically** — the static safety check |
| `compile_time_token_rewriter` | Orchestrator: two jobs, emit | **Critically** — where it becomes a tool |
| `boundary_crossing_vocabulary` | Crossing kinds as data | Supporting (but defines the model) |
| `boundary_crossing_enforcement` | Whitelist parse + modality rule | Yes — the crossing judge |
| `boundary_crossing_discovery` | `U ∩ P` reverse-dep discovery | Yes — the reverse-coupling proof |
| `state_crossing_generation` | State A/B generation + `-dU` gate | **Critically** — the safe lazy path |
| `state_validation` | Independent verifier | **Critically** — the trust anchor |
| `parameter_contract_vocabulary` | Five-axis grammar as data | Supporting (adjacent language) |
| `parameter_contract_recognizer` | Annotation detection, zero hardcoded tokens | Supporting |
| `parameter_contract_translator` | Strip annotation, emit axis 5 by build mode | Supporting |

## Appendix C — The proof-to-mechanism map

For the reader who wants to see exactly which proof licenses which piece of code:

| Proof / primitive | Licenses | Realized in |
|---|---|---|
| P1 (identifier capture is real) | The need for a source-scoped identifier set | `project_identifier_set`, `collision_detection` |
| P2 (per-TU ODR divergence) | One build-wide state; the State A hazard | `ISOLATION_STATE` (Makefile), §6.4 |
| P3 (only file-scope crosses link) | Reverse-dep needs only `nm`; collision needs more | `boundary_crossing_discovery` vs `project_identifier_set` |
| P4 (`-dM -E` inherits includes) | Vendor macro table completeness | `vendor_macro_table` |
| P5 (residual token presence) | The two-layer collision check | `collision_detection` + validation pass |
| P6 (internals disqualified) | Stage-outputs-only discipline | Every query wrapper |
| Reverse-dep by complement | `U ∩ P` is the whole reverse-dep answer | `boundary_crossing_discovery.c` |
| Token-paste biconditional | Static + post-preprocessing = complete | `collision_detection` (§5.3) + validation pass |
| State A safety condition | `-dU` decides auto-guard vs refuse | `state_crossing_generation.c` |
| Validator independence | Verdict from compiler output, no shared logic | `state_validation.c` |
