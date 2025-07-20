+++
title = 'Roman to Integer — Clean, Fast, and Functional Approaches Compared'
date = 2025-07-20T12:09:04+01:00
draft = false
description = "LeetCode 13 — Parsing Roman numerals in Python using a clean single-pass solution and a functional overengineered version. Benchmarks and complexity included."
tags = ["python", "leetcode", "clean code", "algorithms", "interview prep", "functional programming"]
categories = ["python"]
slug = "roman-to-integer"

[cover]
  image = "images/cover/roman-cover.png"
  alt = "Python Code Cover Image"
+++

## 🧩 Problem: Roman to Integer ([LeetCode 13](https://leetcode.com/problems/roman-to-integer/))

Convert a string representing a Roman numeral into its integer value.
Roman numerals are typically written from largest to smallest (left to right),
except for six well-known subtractive cases:

- I before V or X → 4, 9
- X before L or C → 40, 90
- C before D or M → 400, 900

## ✅ Clean Idiomatic Solution (One-Pass)

```python
class Solution:
    SYMBOLS = {
        'I': 1, 'V': 5, 'X': 10,
        'L': 50, 'C': 100,
        'D': 500, 'M': 1000
    }

    def romanToInt(self, s: str) -> int:
        total = 0
        prev_value = 0
        for c in s:
            value = self.SYMBOLS[c]
            total += value - 2 * prev_value if value > prev_value else value
            prev_value = value
        return total
```

- Single loop
- Detects subtractive patterns with `value > prev_value`
- Clean inline logic

✅ Runtime: 3 ms (faster than 79.17%)  
✅ Memory: 17.6 MB (less than 90.19%)

## 🔁 Overengineered (but fun!) Functional Version

```python
def romanToGenerator(s):
    from operator import add, sub
    SYMBOLS = {
        'I': 1, 'V': 5, 'X': 10,
        'L': 50, 'C': 100,
        'D': 500, 'M': 1000
    }
    total = 0
    prev_val = 0
    for c in s:
        val = SYMBOLS[c]
        op = sub if prev_val < val else add
        delta = val if op is add else val - 2 * prev_val
        total += delta
        yield total
```

- Uses `operator.add` / `operator.sub`
- Yields intermediate results step-by-step
- Great for teaching or tracing, but slower

## 📊 Benchmarks

| Version                | Avg. time (100K runs) | Notes                        |
| ---------------------- | --------------------- | ---------------------------- |
| Idiomatic version      | ≈145 ms               | Fastest and cleanest         |
| Overengineered (list)  | ≈205 ms               | Stores operation pairs       |
| Overengineered (yield) | ≈195 ms               | Generator-based, incremental |

## 🧮 Complexity Analysis

- Time Complexity:
  - Best / Worst / Average: O(n) — single pass

- Space Complexity:
  - Idiomatic: O(1) (constant variables)
  - Overengineered (list): O(n) (stores ops)
  - Overengineered (yield): O(1) (no intermediate structure)

## ⏱️ Time Invested

- Reading and planning: 20 minutes
- Coding and analysis: 50 minutes
- **Total:** 1h10min

## 🧠 What I Learned

- Subtractive notation in Roman numerals can be generalized with `value > prev_value`.
- The difference between clean idiomatic solutions and overengineered abstractions in Python.
- How to benchmark Python code using `timeit`, and what tradeoffs come with generators and operator functions.
- The value of exploring multiple angles of a problem, even if the first solution works.

## 🌍 Real-World Applications

- Token-based parsing of custom numeric notations or symbolic expressions.
- Building interpreters or compilers that evaluate streams of symbols (e.g., financial codes, scientific notation).
- Teaching clean code versus functional design tradeoffs.
- Performance benchmarking and complexity analysis in educational content.

## 💬 Personal Note

What started as a simple Roman numeral conversion turned into an exploration of Python idioms, operator functions, generator design, and performance benchmarking.

Even the simplest problems can reveal new learning paths if you're curious enough 🧠

---

### 📂 Code Repository
You can find my full source code for this solution on GitHub:
* 📦 Repo: [github.com/fredericogago/leetcode](https://github.com/fredericogago/leetcode.git)
* 📄 File: [`Roman to Integer.py`](https://github.com/fredericogago/leetcode/blob/main/src/leetcode/editor/en/%5B13%5DRoman%20to%20Integer.py)

---

👨‍💻 Like this post? 
Let’s connect on [LinkedIn](https://www.linkedin.com/in/frederico-gago-5849281aa)
or check out more on [GitHub](https://github.com/fredericogago)!