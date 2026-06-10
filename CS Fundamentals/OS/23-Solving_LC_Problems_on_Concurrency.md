# Solving LeetCode Problems on Concurrency

*Hands-on session: three LeetCode concurrency problems solved in C++, demonstrating thread synchronization using mutex, condition variables, and a `turn` variable.*

> Concurrency problems don't appear directly in DSA coding rounds, but Dining Philosophers and similar synchronization questions come up in system design/discussion rounds.

---

## Core Tools Used Across All Problems

- **`mutex` (mtx):** Protects shared state. Ensures only one thread is in the critical section at a time.
- **Condition variable (cv):** Allows a thread to *wait* on a condition and be *notified* when the condition may have changed.
- **`turn` variable:** A shared integer (or bool) that explicitly controls *which thread's turn it is* to proceed.

**Pattern in every solution:**
```css
lock(mtx)
while (turn != my_expected_turn):
    cv.wait(lock)   // releases lock, sleeps; re-acquires lock on wake
// do work
turn = next_turn
cv.notify_all()
unlock(mtx)
```

---

## Problem 1: Print in Order (LeetCode 1114)

**Problem:** A class `Foo` has three methods: `first()`, `second()`, `third()`. Each is called by a separate thread concurrently. Ensure `first` always prints before `second`, and `second` always before `third`.

**Why it's non-trivial:** Three threads are scheduled independently. Without synchronization, any ordering can occur.

**Solution approach:**
- Use a `turn` variable initialized to `0`.
  - `turn == 0` → first's turn
  - `turn == 1` → second's turn
  - `turn == 2` → third's turn
- Each thread waits in a `while` loop until `turn` matches its expected value.
- After printing, the thread increments `turn` and calls `cv.notify_all()`.

**Execution trace (t=0: all three threads launched):**

```css
T2 (second): locks → sees turn=0, turn≠1 → cv.wait() → releases lock
T3 (third):  locks → sees turn=0, turn≠2 → cv.wait() → releases lock
T1 (first):  locks → sees turn=0, turn==0 → condition false → prints "first"
             → sets turn=1 → cv.notify_all()

T2 wakes: checks turn=1 → turn==1 → exits while → prints "second"
          → sets turn=2 → cv.notify_all()

T3 wakes: checks turn=2 → turn==2 → exits while → prints "third"
```

*Output is always: `first → second → third`, regardless of scheduler order.*

---

## Problem 2: Fizz Buzz Multithreaded (LeetCode 1195)

**Problem:** Implement FizzBuzz with four threads:
- Thread A (`fizz`): prints "fizz" when `i % 3 == 0` and `i % 5 != 0`
- Thread B (`buzz`): prints "buzz" when `i % 5 == 0` and `i % 3 != 0`
- Thread C (`fizzbuzz`): prints "fizzbuzz" when `i % 3 == 0` and `i % 5 == 0`
- Thread D (`number`): prints `i` otherwise

**Shared state:** The loop counter `i` (starts at 1) is shared across all four threads.

**Solution approach:**
- Shared variable `i` (initialized to 1), protected by mutex + condition variable.
- Each thread runs a `while (i <= n)` loop.
- Each thread checks whether the *current value of `i`* satisfies its condition.
- If not its turn, the thread calls `cv.wait()`.
- The thread that *should* act on the current `i` is the only one whose while-condition evaluates to false (exits the wait loop).
- After printing, that thread does `i++` then `cv.notify_all()`.

**Condition logic (each thread waits while its condition is NOT met):**
```cpp
// fizz thread: active when i%3==0 && i%5!=0  →  condition value = (i%3==0 && i%5!=0)
// wait while condition == 0
while ((i%3==0 && i%5!=0) == 0) cv.wait(lock);
```css

**Execution trace for i=1:**
- fizz: `1%3 != 0` → condition=0 → waits
- buzz: `1%5 != 0` → condition=0 → waits
- fizzbuzz: neither → waits
- number: `1%3 != 0 && 1%5 != 0` → condition=1 → proceeds, prints `1`, increments `i` to 2, notifies all

**For i=3:**
- All threads wake on notify_all; each re-evaluates.
- fizz: `3%3==0 && 3%5!=0` → condition=1 → proceeds, prints "fizz", increments to 4, notifies.
- Others re-evaluate and go back to wait.

*The `turn` variable here is `i` itself — each thread's condition directly encodes "is it my turn?"*

---

## Problem 3: Dining Philosophers (LeetCode 1226)

**Background:** Five philosophers sit at a round table. Between each pair is one fork (5 forks total). A philosopher needs *both* the left and right fork to eat. If all philosophers pick up their left fork simultaneously, no one can get the right fork → **deadlock**.

**Recall the three tweaks from the Dining Philosophers lecture (Lecture 20):**
1. Make the picking-up action atomic (critical section) — a philosopher acquires *both* forks or *neither*.
2. Allow at most 4 philosophers to be simultaneously trying to pick up forks.
3. Odd-numbered philosophers pick left first, even-numbered pick right first.

**Tweak used here:** Tweak 1 — make `pickUp` a critical section.

**Solution approach:**
- Five binary semaphores (one per fork), implemented as custom semaphores using condition variables (since C++ doesn't have a built-in semaphore in older standards).
- The entire `pickLeftFork + pickRightFork` sequence is made a critical section using a mutex.
- Only one philosopher can be in the "picking up" phase at a time.
- If philosopher Ph0 is in the critical section picking up both forks, Ph1 waits at the mutex.
- Ph2, whose adjacent forks are both free, can proceed *after* Ph0 exits the critical section.

**Custom Semaphore (C++):**
```cpp
struct Semaphore {
    int count;
    mutex mtx;
    condition_variable cv;

    void wait() {
        unique_lock<mutex> lock(mtx);
        count--;
        if (count < 0) cv.wait(lock);
    }

    void signal() {
        unique_lock<mutex> lock(mtx);
        count++;
        cv.notify_one();
    }
};
```

**`wantsToEat` logic (simplified):**
```cpp
void wantsToEat(int philosopher,
                function<void()> pickLeftFork,
                function<void()> pickRightFork,
                function<void()> eat,
                function<void()> putLeftFork,
                function<void()> putRightFork) {
    // Critical section: pick up BOTH forks atomically
    mutex_lock.lock();
    forks[left].wait();
    pickLeftFork();
    forks[right].wait();
    pickRightFork();
    mutex_lock.unlock();

    eat();

    // Put down forks
    putLeftFork();
    forks[left].signal();
    putRightFork();
    forks[right].signal();
}
```css

**Why deadlock cannot occur:**
- Only one philosopher at a time is in the critical section attempting to pick up forks.
- That philosopher will always successfully acquire both forks (since no other philosopher can grab them while it is in the critical section).
- The circular wait condition cannot form because simultaneous fork acquisition by multiple philosophers is prevented.

> *Homework: Try the other two tweaks (limit philosophers to 4, or odd/even fork order) and verify they also pass on LeetCode.*

---

## Conclusion: Pattern Summary

*Every concurrency problem in this session used the same small toolkit:*

| Tool | Role |
|---|---|
| `mutex` | Protect shared state; enable critical sections |
| `condition_variable` | Thread waits until a condition is true; another thread notifies |
| `turn` variable | Explicitly decides which thread proceeds next |
| `while` loop + `cv.wait()` | Robust wait: re-checks condition after each notification (handles spurious wakeups) |

- The `turn` variable is the central coordinator — it encodes "whose turn is it" and drives the synchronization logic.
- `cv.notify_all()` wakes all waiting threads; only the one whose condition is satisfied proceeds; the rest go back to waiting.
