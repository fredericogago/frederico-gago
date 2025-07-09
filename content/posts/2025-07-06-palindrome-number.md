+++
title = "Is It a Palindrome Number? Three Ways to Solve It Without Converting to String"
date = 2025-07-06T16:00:00+01:00
draft = false
description = "I explore three different approaches to check if an integer is a palindrome — from the most direct solution to a mathematical reversal without string conversion."
tags = ["python", "leetcode", "clean code", "algorithms", "interview prep"]
categories = ["python"]
slug = "palindrome-number"

[cover]
  image = "images/cover/Python-palindrome.png"
  alt = "Python Code Cover Image"
+++


## 🧩 The Problem

Given an integer `x`, determine whether it's a palindrome — that is, whether it reads the same forwards and backwards.

🔗 Problem Link: [LeetCode 9. Palindrome Number](https://leetcode.com/problems/palindrome-number/description/)

### Examples

```text
Input:  x = 121   → Output: True
Input:  x = -121  → Output: False
Input:  x = 10    → Output: False
```

> Bonus: Can you solve it **without converting the integer to a string**?

---

## 🔍 The Three Approaches I Explored

### 1. 💡 Direct Solution with String Conversion

```python
def isPalindrome(x: int) -> bool:
    return str(x) == str(x)[::-1]
```

* Simple and fast (4 ms on LeetCode)
* But does **not** satisfy the follow-up constraint of avoiding strings

---

### 2. 📐 Mathematical Reversal Using `log10` and Closed-Form Formula

Inspired by [this StackExchange post](https://math.stackexchange.com/q/480068), this solution uses a formula to reverse digits without strings:

```math
reverse(x) = x × 10ⁿ − 99 × Σₖ floor(x / 10ᵏ) × 10ⁿ⁻ᵏ
```

* ✅ No string conversion
* ✅ Mathematically elegant
* ⚠️ Slightly slower due to `log10()` and exponentiation
* ⚠️ Can suffer from floating-point rounding issues

---

### 3. ⚙️ Iterative Arithmetic Reversal (Preferred)

```python
def reverse(x: int) -> int:
    result = 0
    while x > 0:
        result = result * 10 + x % 10
        x //= 10
    return result
```

* ✅ No strings
* ✅ Fast and stable
* ✅ Easy to understand and debug

---

## ⏱️ Benchmark Results

Tested on values like `11, 101, 1001, ..., 10000001`

```text
Iterative method:    ~0.000009 seconds
Math (log10) method: ~0.000033 seconds
```

✅ Both produce the correct result
📌 **Iterative** is about **3.7× faster** and more robust for real-world use

---

## 🧠 What I Learned

* How to reverse integers purely with arithmetic
* How to benchmark small Python functions using `time.perf_counter()`
* That **elegance** can sometimes come at a **performance cost**
* How to choose the right tradeoff between **readability**, **correctness**, and **efficiency**

---

## 🌍 Real-World Applications

Palindrome logic pops up in places like:

* Input validation (e.g. palindromic IDs)
* Data compression using symmetry
* Embedded systems that don’t allow string types
* Constraints in coding interviews or systems-level code (like drivers/firmware)

---

## 📂 Code Repository

You can find my full source code for this solution on GitHub:

* 📦 Repo: [github.com/fredericogago/leetcode](https://github.com/fredericogago/leetcode.git)
* 📄 File: [`Palindrome Number.py`](https://github.com/fredericogago/leetcode/blob/main/src/leetcode/editor/en/%5B9%5DPalindrome%20Number.py)

---

## ✍️ Final Thoughts

Solving the same problem in multiple ways helps me grow not only as a developer, but as a **technical communicator**. Being able to solve is great. Being able to **explain clearly** — that’s where the real value is.

> *"Hope it helps! I’m learning to write more elegant solutions with Python 3.13 and Clean Code."* 🚀

---

👨‍💻 Like this post?
Let’s connect on [LinkedIn](https://www.linkedin.com/in/frederico-gago-5849281aa)
or check out more on [GitHub](https://github.com/fredericogago)!
