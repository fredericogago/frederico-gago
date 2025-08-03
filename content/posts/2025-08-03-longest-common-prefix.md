+++
title = "Longest Common Prefix: Five Algorithms Compared"
date = 2025-08-03T12:33:27+01:00
draft = false
description = "Explore five different strategies to solve LeetCode 14 - Longest Common Prefix, complete with benchmarks, complexity analysis, and guidance on choosing the optimal approach."
tags = ["python", "leetcode", "algorithms", "benchmarking", "clean-code", "interview-prep"]
categories = ["python", "algorithms"]
slug = "longest-common-prefix"


[cover]
image = "images/cover/longest-common-prefix-cover.png"
alt = "Illustration of string prefixes" 
+++


## 🔢 Problem Statement

**LeetCode 14 – Longest Common Prefix**  
Given an array of strings, return the longest common prefix among them. If no common prefix exists, return an empty string (`""`).

- **Example 1**  
  Input: `["flower", "flow", "flight"]` → Output: `"fl"`
- **Example 2**  
  Input: `["dog", "racecar", "car"]`   → Output: `""`

**Constraints**
- `1 ≤ strs.length ≤ 200`
- `0 ≤ strs[i].length ≤ 200`
- `strs[i]` contains only lowercase English letters if non-empty.

---

## 👨‍💻 Solution Strategies

We’ll explore and compare five distinct approaches, from brute force to divide & conquer:

| #  | Approach                        | Time Complexity            | Space       | Bench Rank    |
|----|---------------------------------|----------------------------|-------------|---------------|
| 1  | Horizontal Scan                 | O(S)                       | O(1)        | 4th           |
| 2  | Zip-based Scan                  | O(S)                       | O(1)        | 3rd           |
| 3  | Binary Search on Prefix Length  | O(S·log m)                 | O(1)        | 2nd           |
| 4  | Sort & Compare (first vs last)  | O(n·m + n log n)           | O(1)        | **1st**       |
| 5  | Divide & Conquer                | O(S·log n)                 | O(log n)    | 5th           |

Where:
- n = number of strings
- m = length of the shortest string
- S = total number of characters across all strings

---

## 📝 Detailed Breakdown

### 1. Horizontal Scan
Compare each character position across all strings until a mismatch occurs.

```python
for i in range(min_len):
    ch = strs[0][i]
    if all(s[i] == ch for s in strs):
        prefix += ch
    else:
        break
```

👍 **When to use**: Very small input size or whiteboard interviews—simple and intuitive.

---

### 2. Zip-based Scan
Transpose the list of strings with `zip(*strs)` and use `set` to detect divergence.

```python
for letters in zip(*strs):
    if len(set(letters)) == 1:
        prefix += letters[0]
    else:
        break
```

👍 **When to use**: Clean Pythonic code; moderate input sizes, readability prioritized.

---

### 3. Binary Search on Prefix Length
Binary‐search the possible prefix length range [1..min_len], validating with `startswith()`.

```python
low, high = 1, min_len
while low <= high:
    mid = (low + high) // 2
    if is_common_prefix(mid):
        best = strs[0][:mid]
        low = mid + 1
    else:
        high = mid - 1
```

👍 **When to use**: Strings are long (large m), and individual comparisons are expensive.

---

### 4. Sort & Compare (First vs Last)
Sort `strs` lexicographically; only the first and last items need comparison.

```python
strs.sort()
first, last = strs[0], strs[-1]
for i, (a, b) in enumerate(zip(first, last)):
    if a != b:
        return first[:i]
return first
```

👍 **When to use**: Default choice—smallest code footprint and best benchmark performance.

---

### 5. Divide & Conquer
Recursively split the array, compute LCP for each half, then merge.

```python
def lcp_range(l, r):
    if l == r:
        return strs[l]
    mid = (l + r) // 2
    left = lcp_range(l, mid)
    right = lcp_range(mid + 1, r)
    # merge two prefixes
    return common_prefix(left, right)
```

👍 **When to use**: Academic deep dives or parallel execution frameworks.

---

## 📊 Benchmark Results

Average runtime per call (seconds):

| n    | Sort & Compare | Binary Search | Zip Scan | Horizontal Scan | Divide & Conquer |
|------|----------------|---------------|----------|-----------------|------------------|
| 10   | 0.000012       | 0.000012      | 0.000021 | 0.000078        | 0.000061         |
| 100  | 0.000012       | 0.000034      | 0.000106 | 0.000424        | 0.000692         |
| 500  | 0.000017       | 0.000177      | 0.000546 | 0.002052        | 0.002920         |
| 1000 | 0.000022       | 0.000390      | 0.001079 | 0.005615        | 0.005845         |
| 2000 | 0.000052       | 0.000723      | 0.002315 | 0.008572        | 0.011302         |

---

## 🧑‍🏫 When to Use Each Approach

- Sort & Compare: Go-to for most real-world cases—minimal code, fastest in
  practice.
- Binary Search: When strings are very long and each startswith check is
  expensive.
- Zip-based Scan: Super readable and concise; perfect for moderate input sizes.
- Horizontal Scan: Ultra-clear and memory-light; good for tiny lists or
  interview whiteboard.
- Divide & Conquer: Great for teaching or parallel frameworks—even though
  recursion adds overhead.

---

## 💡 Key Takeaways

- Constant‐time factors matter: two O(S) scans can differ drastically in practice.
- Benchmark on realistic data shapes before optimizing.
- Keep code simple unless a clear bottleneck exists.

---

## ⏱️ Time Invested

- Analysis & design: 15 min
- Implementation & benchmarking: 30 min
- Writing & review: 10 min

Total: ~ 55 min

---

## 🔗 Resources

- [▶ Run on LeetCode](https://leetcode.com/problems/longest-common-prefix/)  
- [🔗 My Submission](https://leetcode.com/problems/longest-common-prefix/submissions/1721806526)  
- [💻 Source Code on GitHub](https://github.com/fredericogago/leetcode/blob/main/src/leetcode/editor/en/%5B14%5DLongest%20Common%20Prefix.py)

---

✍️ *Enjoyed this deep dive? Let’s connect on [LinkedIn](https://www.linkedin.com/in/frederico-gago-5849281aa)* or explore more on [GitHub](https://github.com/fredericogago/leetcode).
