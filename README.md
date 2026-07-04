# perl-stack-return-variable

This repository contains an experimental evolution of the Perl Regex Engine (v5.44+ / blead) 
to allow native Chomsky Type 1 processing using dynamic variable scoping.

## Quick Start
1. Clone this repository.
2. Run `make` to compile the debug-enabled custom Perl binary.
3. Run the context-sensitive test suite:
   ```bash
   ./perl -Ilib tests/type1_completeness.t
   ```

## Technical Proposal
For the deep architectural analysis, memory stack impact, and implementation details 
regarding `regexec.c` and `regcomp.c`, please read the formal **[RFC.md](RFC.md)**.

