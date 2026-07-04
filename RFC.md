# RFC: Native Dynamic Scoping and Return-Effect Capture Variables in Perl Regex Core

* **Author:** jpfrx
* **Status:** Draft / Research
* **Target Version:** Perl v5.44.0+ (blead)
* **Repository:** perl-stack-return-variable

## 1. Abstract
This RFC proposes an architectural evolution for the Perl regular expression engine (`regcomp.c` / `regexec.c`). By introducing a novel dynamic scoping mechanism via the `(?<$name>...)` syntax, we enable capture registers to persist their values across recursive subroutine boundaries (`GOSUB`/`RECURSE`). This extends the engine's capabilities to achieve native **Chomsky Type 1 (Context-Sensitive)** completeness in a purely declarative manner, resolving historical memory limits (stack exhaustion) and avoiding non-portable code embedding hacks `(?{...})`.

## 2. The Architectural Problem
Currently, when the regex engine executes a recursive group call `(?&router)`, it encounters a structural scoping conflict:
1. At the subroutine entry, the engine takes a full snapshot of the match data (all capture registers) and pushes it onto the backtracking stack (`regexec.c`).
2. The subroutine executes, writing over the flat capture registry array.
3. Upon hitting the `TAIL` node (subroutine exit), a strict LIFO pop occurs, executing a brute memory restore that completely overwrites any modifications done inside the subroutine with the parent's historical snapshot.

This makes it impossible to implement a reusable, state-mutating router or lexical tokenizer within a regex. To achieve context-sensitivity today, developers must choose between:
* Inline code duplication (ruining maintainability).
* Deeply nested recursive configurations that rapidly hit the physical interpreter limit of **65,534 stack frames**, crashing with resource exhaustion.

## 3. Proposed Solution: Return-Effect Variables

### 3.1 Syntax Expansion
We propose the addition of the `(?<$name>...)` construct. The inclusion of the Perl variable sigil `$` tells the compiler that this specific named group operates under a mutable dynamic scope with a return effect.

```perl
# Example Syntax Concept
/(?<router> ... (?<\$state> [a-z]+) ... ) (?&router) /x
```

### 3.2 Core Engine Changes (Perl C Source)

#### A. Compiler Phase (`regcomp.h` / `regcomp.c`)
* Intercept the `$` character during named capture parsing inside `S_reganysubstr()`.
* Map this metadata to the compiled `regnode`. We will introduce a bit-flag macro: `RE_NAMED_BUFF_DYNAMIC_SCOPE`.

#### B. Execution Phase (`regexec.c`)
* Modify the `TAIL` evaluation branch within the main `S_regmatch()` switch-case statement.
* Instead of performing an unconditional restore of the captured buffers from the parent stack frame, insert a conditional filter loop:

```c
/* Conceptual patch logic inside regexec.c during GOSUB/RECURSE pop */
for (I = 0; I < total_captures; i++) {
    if (capture_registry[i].flags & RE_NAMED_BUFF_DYNAMIC_SCOPE) {
        /* SKIP restoration: Allow the deeply computed value to bubble up */
        continue; 
    }
    /* Otherwise, perform standard historical restoration */
    restore_capture_slot(i); 
}
```

## 4. Engineering Impact
* **Chomsky Type 1 Native Processing:** Turns regex capture groups into genuine accumulators/state registers.
* **O(1) Memory Space for Context Mutations:** Reusable loops no longer need to allocate thousands of nested frames on the heap just to bypass the LIFO destruction. 
* **Backtracking Mitigation:** Empowers deterministic, pipe-less pattern designs (like the SHORT Multiplexer) to achieve linear processing time without performance degradation.

