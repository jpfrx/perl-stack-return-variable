# RFC: Dynamic Scoping and Return-Effect Capture Variables in Perl Regex Core

* **Author:** jpfrx
* **Status:** Draft / Research & Planning Phase
* **Target Version:** Perl v5.44.0+ (blead)
* **Repository:** perl-stack-return-variable

## 1. Abstract
This RFC proposes an architectural evolution for the Perl regular expression engine (`regcomp.c` / `regexec.c`). By introducing a novel dynamic scoping mechanism via a proposed syntax such as `(?<$name>...)`, we aim to allow capture registers to persist their values across recursive subroutine boundaries (`GOSUB`/`RECURSE`). The core objective is to extend engine capabilities toward handling **Chomsky Type 1 (Context-Sensitive)** grammars. Specifically, this evolution ensures that possessive operators and atomic groups can be used in conjunction with stack-return variables to parse context-sensitive structures linearly, without causing catastrophic stack exhaustion or relying on non-portable code embedding hacks `(?{...})`.

## 2. The Architectural Problem
Currently, when the regex engine executes a recursive group call like `(?&router)`, it encounters a structural scoping conflict:
1. At the subroutine entry, the engine takes a full snapshot of the match data (all capture registers) and pushes it onto the backtracking stack (`regexec.c`).
2. The subroutine executes, writing over the flat capture registry array.
3. Upon hitting the `TAIL` node (subroutine exit), a strict LIFO pop occurs, executing a brute memory restore that completely overwrites any modifications done inside the subroutine with the parent's historical snapshot.

This behavior paralyzes state-mutating routers or lexical tokenizers within a regex. As documented in the [`GOMACRO.pl` Case Study](../../rx-design-patterns/src/branch/main/current-work/perl-evo/GOMACRO.pl), when processing a token sequence like `"cd c2"`, the router captures the new context `"c"`, but the subsequent `TAIL` execution forces the capture registry to revert back to the parent's value, breaking deterministic evaluation.

To bypass this today, developers are limited to:
* Inline code duplication (ruining maintainability).
* Deeply nested recursive configurations. As demonstrated across various [Conditional Branching Structures](../../rx-design-patterns/src/branch/main/structures/conditional_branching/), complex context routing rapidly hits the physical interpreter limit of **65,534 stack frames**, crashing the process via resource exhaustion.

## 3. Proposed Solution: Return-Effect Variables

### 3.1 Syntax Expansion
We propose evaluating a construct like `(?<$name>...)`. Incorporating the Perl variable sigil `$` inside the named capture group construct explicitly flags the register as a mutable variable operating under a dynamic scope with a return effect. This syntax approach avoids token collisions in non-Perl engines while remaining intuitive to the Perl ecosystem.

### 3.2 Core Engine Changes (High-Level Specification)

#### A. Compiler Phase (`regcomp.h` / `regcomp.c`)
The compilation phase must be updated to intercept the designated dynamic sigil during named capture parsing (within functions like `S_reganysubstr()`). This metadata will be mapped to the compiled `regnode` using a dedicated internal bit-flag attribute, marking the group for dynamic scope tracking.

#### B. Execution Phase (`regexec.c`)
The main execution loop (`S_regmatch()`) must be updated at the subroutine exit sequence (`TAIL`). Instead of executing an unconditional memory restoration of the captured buffers from the parent stack frame, the engine will evaluate the dynamic scope attribute of each register. If a register is marked for stack-return persistence and its virtual depth counter indicates a mutation intended for a higher stack level, the engine will bypass standard historical restoration for that specific slot, allowing the value to persist past the subroutine pop.

## 4. Engineering Impact and Scope
* **Extended Chomsky Type 1 Capabilities:** Transitions capture registers from static substring containers into mutable state registers, allowing the engine to natively process broader context-sensitive string rules.
* **Stack Exhaustion Mitigation via Possessive Operators:** By combining stack-return variables with possessive quantifiers (`++`, `*+`) or atomic groups `(?>...)`, the engine can instantly discard backtracking checkpoints immediately following a state mutation. This prevents the accumulation of thousands of nested frames to preserve semantic states, achieving O(1) memory space for deterministic context mutations.
* **Linear Performance Integration:** Empowers deterministic, pipe-less pattern designs to execute complex semantic routing fluently without inducing catastrophic engine backtracking.

