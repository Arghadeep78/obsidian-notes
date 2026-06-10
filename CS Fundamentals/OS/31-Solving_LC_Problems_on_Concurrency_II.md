# Solving LeetCode Problems on Concurrency — Part II

*Bonus session: three more LeetCode concurrency problems solved in C++. Same toolkit as Part I — mutex, condition variable, turn variable.*

> Recommendation: Attempt each problem yourself before watching the solution. Concurrency problems build the mental model of how threads interleave and interact.

**Problems covered:**
1. Print FooBar Alternately (LeetCode 1115)
2. Print Zero Even Odd (LeetCode 1116)
3. Building H₂O (LeetCode 1117)

---

## Problem 1: Print FooBar Alternately (LeetCode 1115)

**Problem:** A class `FooBar` has two methods: `foo()` (prints "foo") and `bar()` (prints "bar"). Two threads call them concurrently — one thread calls `foo` in a loop n times, the other calls `bar` in a loop n times. Ensure the output is always `foobarfoobar...` (alternating), not `foofoo...barbar...`.

**Without synchronization:** Thread calling `foo` may complete its entire loop before `bar` thread gets scheduled, producing `foofoofoofoo...barbarbarbbar...`.

**Solution approach:**
- `turn` variable (bool): `false` = foo's turn, `true` = bar's turn.
- `foo` waits while `turn == true` (not its turn); after printing "foo", sets `turn = true` and notifies.
- `bar` waits while `turn == false` (not its turn); after printing "bar", sets `turn = false` and notifies.

**Execution trace (n=2):**

```
Initial: turn = false

T1 (foo): locks → turn==false → condition (turn==true) is false → prints "foo"
          → sets turn=true → cv.notify_all()

T2 (bar): was waiting (turn was false, condition (turn==false) was true)
          → wakes → re-checks: turn==true → (turn==false) is false → exits wait
          → prints "bar" → sets turn=false → cv.notify_all()

T1 (foo): was waiting → wakes → re-checks: turn==false → exits wait
          → prints "foo" → ...
```

*Output: `foo bar foo bar ...` always.*

**Key insight:** The `turn` variable acts as a token that alternates ownership between the two threads after each print.

---

## Problem 2: Print Zero Even Odd (LeetCode 1116)

**Problem:** A class `ZeroEvenOdd` has three methods: `zero()`, `even()`, `odd()`. Three threads call them concurrently. The desired output for n=5 is `0 1 0 2 0 3 0 4 0 5`. Rules:
- Thread A (`zero`): prints `0` before every number.
- Thread B (`even`): prints even numbers.
- Thread C (`odd`): prints odd numbers.

**Shared state:** A loop counter `i` (starts at 1) shared across all threads.

**Turn values:**
- `turn = 0` → zero thread's turn (print 0)
- `turn = 1` → odd thread's turn (print current odd `i`)
- `turn = 2` → even thread's turn (print current even `i`)

**Solution approach:**
- Zero thread runs a loop `while (i <= n)`:
  - Waits while `turn != 0`.
  - Prints 0.
  - If `i` is odd → sets `turn = 1`; if `i` is even → sets `turn = 2`.
  - Calls `cv.notify_all()`.
- Odd thread loops `while (i <= n)`:
  - Waits while `turn != 1`.
  - Prints `i`, does `i++`.
  - Sets `turn = 0`, notifies.
- Even thread loops `while (i <= n)`:
  - Waits while `turn != 2`.
  - Prints `i`, does `i++`.
  - Sets `turn = 0`, notifies.

**Execution trace for i=1,2,3 (n=3):**

```
turn=0: zero prints 0. i=1 (odd) → sets turn=1, notifies.
turn=1: odd prints 1. i becomes 2. sets turn=0, notifies.
turn=0: zero prints 0. i=2 (even) → sets turn=2, notifies.
turn=2: even prints 2. i becomes 3. sets turn=0, notifies.
turn=0: zero prints 0. i=3 (odd) → sets turn=1, notifies.
turn=1: odd prints 3. i becomes 4. sets turn=0, notifies.
turn=0: zero tries; i=4 > n=3 → exits while via break. Notifies.
  → even and odd also wake, check i<=n fails → exit.
```

**Exit mechanism:**
- After `i > n`, each thread's `while (i <= n && ...)` condition fails.
- An inner `if (i > n) break;` ensures clean exit from the wait loop.
- After zero thread's notify_all, the other two threads wake, check `i <= n` (false), and exit.

*Output for n=3: `0 1 0 2 0 3`*

**Key insight:** The zero thread acts as a *dispatcher* — it decides after printing each 0 whether to hand off to the even or odd thread, based on the current value of `i`.

---

## Problem 3: Building H₂O (LeetCode 1117)

**Problem:** There are many hydrogen threads and many oxygen threads. Output must be groups of "HHO" (in any order within a group, e.g., HHO, HOH, OHH are all valid). Each group represents one water molecule. A hydrogen thread calls `hydrogen()`, an oxygen thread calls `oxygen()`. Synchronize so that threads "pass the barrier" only in complete sets of 2H + 1O.

**Example input:** `HOOHHHHH` (4 H threads, 2 O threads)
**Expected output:** Two valid groups: `HHO` and `HHO` (in some interleaving)

**Solution approach:**
- `turn` variable tracks how many hydrogen threads have been released in the current molecule:
  - `turn = 0` → first H can be released
  - `turn = 1` → second H can be released
  - `turn = 2` → O can be released; then reset to 0 for next molecule

- Hydrogen thread:
  - Waits while `turn == 2` (O's turn; H must wait).
  - Calls `releaseHydrogen()`.
  - Does `turn++`, notifies.
- Oxygen thread:
  - Waits while `turn < 2` (not enough H's yet).
  - Calls `releaseOxygen()`.
  - Sets `turn = 0` (start fresh for next molecule), notifies.

**Execution trace (two H threads and one O thread ready):**

```
Initial: turn=0

H1: locks → turn=0, (turn==2) is false → exits wait → releaseHydrogen()
    → turn=1 → notify_all()

H2: wakes → (turn==2) is false → exits wait → releaseHydrogen()
    → turn=2 → notify_all()

O1: wakes → (turn<2) is false → exits wait → releaseOxygen()
    → turn=0 → notify_all()   (molecule complete, ready for next)

H3 (next molecule): wakes → turn=0, not 2 → proceeds → ...
```

**Why deadlock cannot occur:**
- H threads never hold a resource and wait for another — they simply check `turn` and proceed or wait.
- O threads wait until exactly 2 H atoms have been released.
- No circular dependency exists between H and O threads.

*The `turn` counter cleanly enforces the 2H-before-1O constraint.*

---

## Summary: Full Concurrency Problem Toolkit

*All six problems across Part I and Part II reduce to the same four building blocks:*

| Building Block | Purpose |
|---|---|
| `mutex` | Mutual exclusion over shared state |
| `condition_variable + cv.wait()` | Sleep until condition is true; re-check in a `while` loop |
| `turn` variable | Integer or bool encoding "whose turn is it now" |
| `cv.notify_all()` | Wake all sleeping threads so each re-evaluates its condition |

**The `turn` variable is the master coordinator.** Every problem's logic reduces to:
- Defining what value of `turn` corresponds to each thread's active period.
- Having each thread wait while `turn != its_expected_value`.
- Updating `turn` after completing work and notifying.

> *An interview question worth preparing: "How would you implement a semaphore using a mutex and condition variable?"*
> Answer: Decrement count on `wait()`, sleep if `count < 0`; increment count and `notify_one()` on `signal()` — exactly the custom semaphore built in the Dining Philosophers solution.
