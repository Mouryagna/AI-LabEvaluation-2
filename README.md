# Lab 2 -- Cryptarithmetic Puzzle Using Constraint Satisfaction Problems (CSP)

## 1. Problem Overview

The notebooks solve the cryptarithmetic puzzle:

``` text
  SEND
+ MORE
------
 MONEY
```

Each distinct letter represents one digit from 0 to 9. The same letter
must always represent the same digit, and different letters must
represent different digits.

### Constraints

1.  **Unique digits (AllDifferent):** No two letters can have the same
    digit.
2.  **No leading zero:** `S` and `M` cannot be zero.
3.  **Arithmetic validity:** The numerical value of `SEND` plus `MORE`
    must equal `MONEY`.
4.  **Carry constraints:** In the versions that model columns, each
    column must satisfy its addition equation.

## 2. Notebook 1 -- `labeval2.ipynb`: Basic CSP Solver

This is the simplest implementation. It models the eight letters as CSP
variables, each with a digit domain.

### Main components

-   **Initial domains:** Every letter receives digits 0--9. Zero is
    removed from the domains of `S` and `M`.
-   **`check_constraints()`:** Checks digit uniqueness and leading-zero
    restrictions. It verifies the complete arithmetic equation after all
    eight letters have been assigned.
-   **MRV (Minimum Remaining Values):** Selects the unassigned letter
    with the smallest remaining domain.
-   **LCV (Least Constraining Value):** Tries values that appear in
    fewer other unassigned variables' domains first.
-   **Forward checking:** After assigning a digit, removes that digit
    from the domains of the other unassigned letters. If any domain
    becomes empty, the branch is rejected.
-   **Constraint propagation:** Repeatedly removes already assigned
    digits from unassigned letter domains until no more changes occur.
-   **Backtracking:** Assigns a value, checks constraints, propagates
    domain reductions, and recursively searches. If a branch fails, it
    removes the assignment and tries another value.

### Key limitation

The arithmetic equation is checked only when all letters have been
assigned. There are no explicit carry variables or column-wise
arithmetic constraints, so arithmetic information is not propagated
early.

## 3. Notebook 2 -- `labeval2_harcoded_constraints.ipynb`: Hardcoded Column Constraints

This version improves the basic solver by representing the addition
column by column for the specific `SEND + MORE = MONEY` puzzle.

### Additional variables

It introduces carry variables `C1`, `C2`, `C3`, and `C4`, each with a
domain of `{0, 1}`. Since the result is five digits, it fixes `C4 = 1`
and `M = 1`.

### Column equations

  Column      Constraint
  ----------- --------------------------
  Units       `D + E = Y + 10*C1`
  Tens        `N + R + C1 = E + 10*C2`
  Hundreds    `E + O + C2 = N + 10*C3`
  Thousands   `S + M + C3 = O + 10*C4`

### Additional functions

-   **`direct_deductions()`:** When enough domains are singletons,
    calculates the forced digit or carry for a column. It also includes
    a reverse deduction for `M`.
-   **`revise()`:** Enumerates combinations of values in a constraint's
    scope. It retains only values that participate in at least one
    satisfying combination.
-   **GAC (Generalized Arc Consistency):** Applies `revise()` to each
    arithmetic constraint so unsupported domain values are removed.
-   **`propagate()`:** Repeats direct deductions, GAC revision, and
    removal of assigned digits until domains stop changing or a domain
    becomes empty.

The notebook retains MRV, LCV, forward checking, and recursive
backtracking from the basic solver.

### Key limitation

The column equations and deductions are written specifically for
`SEND + MORE = MONEY`. Changing the words requires manually changing the
variables, domains, and arithmetic constraints.

## 4. Notebook 3 -- `lab2_dynamic_generator.ipynb`: Dynamic CSP Generator

This is the most general implementation. Instead of manually writing the
column constraints for one puzzle, it builds them from the two input
words and the result word.

### `build_csp(word1, word2, result, base=10)`

This function:

1.  Converts the input words to uppercase and validates that they are
    non-empty.
2.  Checks that the base is at least 2 and that the number of distinct
    letters does not exceed the base.
3.  Extracts the distinct letters.
4.  Creates carry variables `C0` through `Cn`.
5.  Creates digit domains for letters and carry domains `{0, 1}`.
6.  Removes zero from the leading letters' domains.
7.  Fixes the initial carry `C0` and final carry `Cn` to zero.
8.  Generates one arithmetic constraint for each column, working from
    right to left.

For each column, it generates the equation:

`digit(a) + digit(b) + carry_in = digit(result) + base * carry_out`

If a word has no character in a particular column, that position
contributes zero.

### Search and propagation

The dynamic notebook uses:

-   **MRV:** Chooses the unassigned letter with the smallest current
    domain.
-   **LCV:** Orders candidate digits by how few other letter domains
    contain them.
-   **Forward checking:** Copies the current domains, fixes the selected
    letter's value, and removes that digit from other unassigned
    letters.
-   **`revise()` and GAC:** Remove values that have no supporting
    combination in a column constraint.
-   **`propagate()`:** Repeatedly fixes assigned variables, enforces GAC
    on all generated column constraints, and removes assigned digits
    from unassigned letters.
-   **Backtracking:** Searches recursively and returns a solution if the
    complete letter assignment satisfies the arithmetic equation.

The final cell sets `word1`, `word2`, and `result`, builds the CSP, runs
initial propagation, and invokes backtracking.

### Important implementation detail

Although carry variables are included in the generated CSP and their
domains are reduced by propagation, the recursive variable selector
selects only the letter variables. The carry values are handled through
the generated column constraints and domain propagation rather than
being added to the returned letter assignment.

## 5. Comparison of the Three Implementations

  Feature                  Basic CSP   Hardcoded constraints   Dynamic generator
  ------------------------ ----------- ----------------------- ---------------------------------------
  Puzzle-specific          Yes         Yes                     No; accepts input words
  Letter variables         Yes         Yes                     Generated automatically
  Carry variables          No          Yes                     Generated automatically
  Column-wise arithmetic   No          Yes                     Yes
  Direct deductions        No          Yes                     No separate direct-deduction function
  GAC                      No          Yes                     Yes
  MRV and LCV              Yes         Yes                     Yes
  Forward checking         Yes         Yes                     Yes
  Backtracking             Yes         Yes                     Yes

## 6. Overall Workflow

1.  Define the puzzle and create variables and domains.
2.  Apply initial restrictions such as no leading zero.
3.  Generate or define arithmetic constraints.
4.  Propagate constraints to reduce domains.
5.  Select an unassigned letter using MRV.
6.  Order its candidate digits using LCV.
7.  Assign a digit and perform forward checking.
8.  Apply constraint propagation/GAC.
9.  If a contradiction occurs, backtrack; otherwise continue
    recursively.
10. Verify the completed numerical equation.

## 7. Main Takeaways

-   **CSP formulation** represents the puzzle using variables, domains,
    and constraints.
-   **MRV** chooses the most constrained variable first.
-   **LCV** tries the value expected to restrict other variables the
    least.
-   **Forward checking** detects empty domains immediately after an
    assignment.
-   **Constraint propagation** reduces domains before deeper search.
-   **GAC** checks whether each domain value has support in its
    associated constraint.
-   **Backtracking** explores alternatives when the current assignment
    leads to a contradiction.
-   **Dynamic generation** makes the solver reusable for different
    cryptarithmetic additions instead of tying it to one fixed puzzle.

## 8. One-Sentence Summary

The three notebooks progress from a basic backtracking CSP solver, to a
solver with hardcoded column-wise arithmetic and GAC, and finally to a
dynamic cryptarithmetic CSP generator that constructs variables,
domains, and column constraints from input words.
