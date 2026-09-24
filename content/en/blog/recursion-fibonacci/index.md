---
title: "Recursion, explained gently (with Fibonacci and friends)"
description: "A friendly, beginner-oriented tutorial on recursion: how it works, why Fibonacci is the perfect example, and how to fix its hidden cost."
date: 2026-09-24T10:00:00Z
draft: false
type: "blog"
author: "Axel-Cleris Gailloty"
tags: [algorithms, recursion, python, programming]
---

If you have ever opened a book about algorithms, you have met recursion. And if the word alone makes you nervous, that is completely normal: recursion is one of those ideas that feels alien until the day it suddenly clicks, and then you cannot unsee it everywhere.

This article is written for that journey. No prerequisites beyond basic programming (I will use Python, but the ideas apply to any language). Take your time. Nobody understands recursion on the first pass.

## What is recursion, really?

A recursive function is a function that calls itself. That is the whole definition, and also the whole problem: it sounds circular. A function that calls itself, like a snake eating its tail — how can that ever produce anything useful?

The trick is that each call works on a slightly smaller version of the same problem. Imagine you are standing in a long queue and you want to know your position. You ask the person in front of you: "what is your position?" They ask the person in front of them, and so on. The person at the front says "I am first". Then the answers travel back: second, third, ... until your neighbour tells you their number and you add one.

That is recursion in the wild:

- Somebody has to stop the chain and give a plain answer. That person is the base case.
- Everybody else solves the problem using a smaller version of it. That is the recursive case.

## The anatomy of a recursive function

Almost every recursive function you will ever write follows this shape:

```python
def solve(problem):
    if problem is small enough:      # base case
        return obvious answer
    smaller = reduce(problem)         # make progress
    return combine(solve(smaller))    # recursive case
```

Three rules, and they are worth memorising:

1. A base case that returns without calling itself.
2. A recursive case that calls itself on a strictly smaller problem.
3. Progress: every call must move towards the base case.

Break rule 1 and you get infinite recursion and a stack overflow. Break rule 2 or 3 and the function never converges. Keep them and, remarkably, the logic just holds together.

## A gentle first example: factorial

The factorial of n (written n!) is the product of all integers from 1 to n. So 5! = 5 x 4 x 3 x 2 x 1 = 120.

There is a loop version, and there is a recursive one:

```python
def factorial_loop(n):
    result = 1
    for i in range(2, n + 1):
        result *= i
    return result

def factorial(n):
    if n <= 1:            # base case
        return 1
    return n * factorial(n - 1)  # recursive case
```

Read the recursive version out loud: "the factorial of n is n times the factorial of n minus 1, and the factorial of 1 is 1." That is literally the mathematical definition. This is the quiet superpower of recursion: it lets you translate a definition directly into code, with no translation effort. The loop version computes; the recursive version declares.

Trace it once by hand for factorial(3), on paper:

```text
factorial(3)
= 3 * factorial(2)
= 3 * (2 * factorial(1))
= 3 * (2 * 1)
= 6
```

If you can do this trace, you understand recursion better than you think.

## Fibonacci: the classic, and its hidden trap

The Fibonacci sequence is the poster child of recursion. Each term is the sum of the two previous ones:

```text
0, 1, 1, 2, 3, 5, 8, 13, 21, 34, ...
```

As a definition:

- F(0) = 0
- F(1) = 1
- F(n) = F(n - 1) + F(n - 2) for n >= 2

Translated to Python, almost word for word:

```python
def fib(n):
    if n < 2:                 # base cases
        return n
    return fib(n - 1) + fib(n - 2)  # two recursive calls
```

Six lines. Elegant. It looks like the definition fell straight out of the textbook into your editor. Try fib(10): instant. fib(25): fine. fib(40)? You can go make coffee. On a typical laptop, fib(40) takes seconds and fib(50) takes longer than you will be willing to wait. Something is deeply wrong, and it is worth understanding precisely what, because it is one of the most valuable lessons in all of algorithms.

### Why is it so slow?

The problem is that fib calls itself twice, and those calls redo the same work over and over. Look at what happens for fib(5):

```text
                fib(5)
              /        \
          fib(4)        fib(3)
         /      \       /    \
     fib(3)   fib(2)  fib(2) fib(1)
     /   \     /  \    /  \
 fib(2) fib(1) ...  ... ...
```

fib(3) is computed twice. fib(2) is computed three times. And the further down you go, the worse it gets: the number of calls grows exponentially, roughly multiplying by 1.6 at each level. Computing fib(50) naively takes about 20 billion function calls. To produce one number.

This is not a Python problem or a syntax problem. It is a design problem, and it teaches the general lesson: a recursive function that branches into several calls with overlapping subproblems will explode unless you manage the overlap.

### The fix: memoization

The observation is simple: the answer to fib(2) never changes. Once you have computed it, why compute it again? Memoization means keeping a little notebook (a cache) of results you have already calculated:

```python
def fib_memo(n, cache=None):
    if cache is None:
        cache = {}
    if n in cache:            # already computed?
        return cache[n]
    if n < 2:
        result = n
    else:
        result = fib_memo(n - 1, cache) + fib_memo(n - 2, cache)
    cache[n] = result          # write it down
    return result
```

Now each value from 0 to n is computed exactly once. fib_memo(50) returns instantly. fib_memo(1000) too. We went from billions of calls to a few thousand, with three extra lines. In Python you can even get this for free with the standard library:

```python
from functools import lru_cache

@lru_cache(maxsize=None)
def fib(n):
    if n < 2:
        return n
    return fib(n - 1) + fib(n - 2)
```

### The other fix: go bottom-up

Memoization is top-down: you start from n and trust the cache on the way down. The alternative is bottom-up: start from the base cases and build your way up. At that point you are writing a loop again, but a smart one that only keeps what it needs:

```python
def fib_iter(n):
    if n < 2:
        return n
    previous, current = 0, 1
    for _ in range(2, n + 1):
        previous, current = current, previous + current
    return current
```

Two variables, one loop, constant memory, no recursion at all. This technique is called dynamic programming, and Fibonacci is its gentlest introduction. The key insight is the same in both versions: never compute the same thing twice.

So which one should you use? The naive recursive version is wonderful for learning and for explaining the definition. The memoized or iterative versions are what you use in real life. Knowing which context you are in is part of the craft.

## Other places recursion shines

Fibonacci is a toy in one sense: nobody computes it recursively in production. But the shape of the problem it illustrates, "a thing defined in terms of smaller versions of itself", is everywhere. Here are three realistic examples.

### Binary search

Given a sorted list, find an element. Compare with the middle: either you found it, or you can discard half the list and search again in the remaining half. A smaller version of the same problem, by definition:

```python
def binary_search(items, target, low=0, high=None):
    if high is None:
        high = len(items) - 1
    if low > high:                       # base case: empty range
        return -1
    mid = (low + high) // 2
    if items[mid] == target:            # base case: found
        return mid
    if items[mid] < target:             # search right half
        return binary_search(items, target, mid + 1, high)
    return binary_search(items, target, low, mid - 1)  # left half
```

Each call halves the search space. One million items means about twenty calls, not one million comparisons. Notice how the recursive structure makes the "discard half" logic explicit and readable.

### Walking a tree (or a folder)

Loops are natural for lists, which are flat. But some data is naturally nested: folders containing folders, JSON documents containing objects containing arrays, comments containing replies containing replies. For nested data, recursion is not a clever trick — it is the honest translation of the shape of the data.

```python
import os

def list_files(folder):
    for entry in os.listdir(folder):
        path = os.path.join(folder, entry)
        if os.path.isfile(path):
            print(path)                        # base case: a file
        else:
            list_files(path)                   # recursive case: a folder
```

Try writing this with only loops and no recursion, handling arbitrary depth. You will need to manage a stack manually. The recursive version lets the call stack do that bookkeeping for you. That is the general truth here: recursion and nested data are made for each other.

### The Tower of Hanoi

The classic puzzle: move a stack of disks from one peg to another, one disk at a time, never placing a big disk on a smaller one. Iteratively, the solution is confusing. Recursively, it is almost embarrassing:

```python
def hanoi(n, source, target, auxiliary):
    if n == 0:                 # base case: nothing to move
        return
    hanoi(n - 1, source, auxiliary, target)   # move the stack above
    print(f"Move disk {n} from {source} to {target}")
    hanoi(n - 1, auxiliary, target, source)   # move it back on top
```

Read it as a story: to move n disks, move the n - 1 disks above them out of the way, move the big disk, then move the stack back on top of it. Three lines of logic that would take a page of loop code. Some problems are simply recursive by nature, and fighting it is pointless.

## Common mistakes (everyone makes them)

- Forgetting the base case. The function calls itself forever and the program crashes with a stack overflow. If you see a recursion error, check the base case first; nine times out of ten it is missing or never reached.
- A base case that is never reached. Beware of conditions that skip over the base case, like subtracting 2 each time from an odd number while testing for equality with 0. Prefer inequalities (n <= 1) over strict equality (n == 1) to be safe.
- No progress. If the recursive call passes the same problem, or a bigger one, you will never reach the base case.
- Redundant branching calls. If a call spawns several copies that solve overlapping subproblems (hello, naive Fibonacci), you need memoization or a different strategy.

## When should you reach for recursion?

Use recursion when the problem itself is self-similar: trees, nested structures, divide-and-conquer (search, sort, traversal), puzzles defined in terms of smaller versions of themselves. Use loops when the data is flat and sequential. Neither is "better"; they are tools for different shapes of problems. And remember that a clean recursive solution that is too slow can almost always be turned into a fast one with memoization or a bottom-up rewrite, without losing the clarity of the idea.

## Practice, gently

If you want to make this stick, here are a few exercises in increasing order of difficulty:

1. Write a recursive function that counts down from n to 0, printing each number.
2. Write a recursive sum of a list of numbers (hint: the sum is the first element plus the sum of the rest).
3. Write a recursive function that reverses a string.
4. Compute the depth of a nested list (how many levels of nesting it contains).
5. Go back to naive Fibonacci and add memoization yourself, without looking at this article.

Do them on paper first, tracing the calls like we did for factorial. The tracing, not the typing, is where the understanding lives.

## Closing thought

Recursion is not a technique to memorise; it is a shift of perspective. Stop asking "how do I compute this step by step" and start asking "how is this problem built out of smaller versions of itself". Fibonacci, trees, Hanoi — they all reward the same question. Ask it often enough and it becomes a reflex, and one day you will catch yourself seeing a nested JSON document as a friendly little tree, waiting to be walked.
