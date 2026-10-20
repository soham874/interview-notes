# Bit Manipulation: Study Notes & Workout Plan

Oct 12, 2026 · @Soham Choudhury

## How to use this guide

Bit manipulation has a small vocabulary — about 15 tricks and 8 patterns cover nearly every interview question. Learn the tricks until they are reflexes, then practise recognising which pattern a problem wants.

1. Read Foundations and the Cheat sheet once, slowly, with pen and paper. Trace every example by hand in binary.
2. Follow the 4-week plan: about 60–75 minutes a day, 6 days a week, one rest or review day.
3. Before writing code for any problem, write the bits of a small example on paper. Most bit bugs come from skipping this.
4. Log every miss in the mistake log at the end, and re-solve logged problems after 3 and 7 days.

All code is Java. Numbers are written as 8-bit binary for readability; real `int` is 32 bits and `long` is 64.

## Foundations

Every integer is a row of bits; bit k is worth 2^k, counted from the right starting at 0. Negative numbers use two's complement, which is why `-x == ~x + 1`.

### Reading binary

13 = 8 + 4 + 1 = `0000 1101` (bits 0, 2 and 3 are set). To convert by hand, repeatedly divide by 2 and read the remainders bottom-up, or subtract the largest power of 2 that fits.

### Two's complement in one minute

- The leftmost bit (bit 31 in an `int`) is the sign bit: 1 means negative.
- To negate: flip all bits, then add 1. 5 = `0000 0101` → flip → `1111 1010` → +1 → `1111 1011` = -5.
- -1 is all ones: `1111 1111`. This is why `x & -1 == x` and `~0 == -1`.
- `int` range is -2^31 to 2^31 - 1. `Integer.MIN_VALUE` = `1000…0` has no positive twin, so `-Integer.MIN_VALUE == Integer.MIN_VALUE`.

### The seven operators

| Operator | Name | Rule per bit | Example (8-bit) |
| --- | --- | --- | --- |
| `a & b` | AND | 1 only if both are 1 | `1100 & 1010 = 1000` |
| `a \| b` | OR | 1 if either is 1 | `1100 \| 1010 = 1110` |
| `a ^ b` | XOR | 1 if they differ | `1100 ^ 1010 = 0110` |
| `~a` | NOT | flips every bit | `~0000 1100 = 1111 0011` |
| `a << k` | left shift | moves bits left, fills 0s; equals a × 2^k | `0000 0011 << 2 = 0000 1100` |
| `a >> k` | arithmetic right shift | moves right, copies the sign bit in; equals floor(a / 2^k) | `1111 0000 >> 2 = 1111 1100` |
| `a >>> k` | logical right shift | moves right, always fills 0s | `1111 0000 >>> 2 = 0011 1100` |

### XOR laws (memorise these)

- `a ^ a = 0` and `a ^ 0 = a`
- Commutative and associative, so order never matters: `a ^ b ^ a = b`
- If `a ^ b = c` then `a ^ c = b` (XOR undoes itself)
- XOR is addition without carry; `a + b = (a ^ b) + 2 × (a & b)`

### Java gotchas that cost interviews

- **Precedence**: `==` and `!=` bind tighter than `&`, `^`, `|`. Write `(x & 1) == 0`, never `x & 1 == 0`. Shifts bind looser than `+`: `1 << n - 1` means `1 << (n - 1)`.
- **Shift distance is masked**: for `int`, only the low 5 bits of k are used, so `1 << 32 == 1`. For `long`, the low 6 bits.
- **Use `1L` for long masks**: `1 << 40` is wrong; `1L << 40` is right. Same in `mask & (1L << i)`.
- **`>>` vs `>>>` on negatives**: looping `while (n != 0) n >>= 1;` never ends for negative n. Use `>>>`.
- **Overflow**: `Math.abs(Integer.MIN_VALUE)` is still negative. Widen to `long` when negating or summing.
- **Bytes and chars** are promoted to `int` before any bit operation; `(byte) 0x80 >> 1` sign-extends. Mask with `& 0xFF` to treat a byte as unsigned.

### Built-ins worth knowing (Integer and Long both have them)

| Method | Returns | Example for 12 = `1100` |
| --- | --- | --- |
| `Integer.bitCount(x)` | number of set bits | 2 |
| `Integer.numberOfTrailingZeros(x)` | zeros below the lowest 1 | 2 |
| `Integer.numberOfLeadingZeros(x)` | zeros above the highest 1 (out of 32) | 28 |
| `Integer.highestOneBit(x)` | value of the highest set bit | 8 |
| `Integer.lowestOneBit(x)` | value of the lowest set bit | 4 |
| `Integer.reverse(x)` | bits mirrored | — |
| `Integer.toBinaryString(x)` | binary as a string, for debugging | "1100" |

Use built-ins in real code, but be ready to implement bitCount and lowest-set-bit by hand; interviewers often ask.

## Core tricks cheat sheet

These 20 one-liners are the building blocks of every pattern below. Trace each with x = 12 (`1100`) until you can predict the result without paper.

### Single-bit operations (bit i, counted from 0)

| Goal | Java | Why it works |
| --- | --- | --- |
| Check bit i | `(x >> i) & 1` or `(x & (1 << i)) != 0` | Mask isolates one position |
| Set bit i | `x \| (1 << i)` | OR with 1 forces a 1 |
| Clear bit i | `x & ~(1 << i)` | AND with 0 forces a 0 |
| Toggle bit i | `x ^ (1 << i)` | XOR with 1 flips |
| Set bit i to value v (0/1) | `(x & ~(1 << i)) \| (v << i)` | Clear, then OR in v |

### Lowest set bit (the most-used family)

| Goal | Java | x = 12 (`1100`) → |
| --- | --- | --- |
| Clear the lowest set bit | `x & (x - 1)` | `1000` (8) |
| Isolate the lowest set bit | `x & -x` | `0100` (4) |
| Is x a power of two? | `x > 0 && (x & (x - 1)) == 0` | false |
| Set all bits below the lowest set bit | `x \| (x - 1)` | `1111` (15) |
| Clear trailing ones | `x & (x + 1)` | for `0111` → `0000` |

Why `x - 1` works: subtracting 1 flips the lowest set bit to 0 and every 0 below it to 1, leaving higher bits untouched. Why `x & -x` works: `-x = ~x + 1`, so every bit above the lowest 1 is flipped and cancels out under AND.

### Ranges and masks

| Goal | Java |
| --- | --- |
| Mask of the low k bits (k < 32) | `(1 << k) - 1` |
| Keep only the low k bits of x | `x & ((1 << k) - 1)` |
| Clear bits i..0 | `x & ~((1 << (i + 1)) - 1)` (or `x & (-1 << (i + 1))` for i < 31) |
| Flip every bit within the low k bits | `x ^ ((1 << k) - 1)` |

### Arithmetic shortcuts

| Goal | Java | Note |
| --- | --- | --- |
| Is x odd? | `(x & 1) == 1` | Safe for negatives, unlike `x % 2 == 1`, which is false for -3 |
| Multiply or divide by 2^k | `x << k`, `x >> k` | `>>` floors toward -infinity, `/` truncates toward 0 |
| Swap a and b without a temp | `a ^= b; b ^= a; a ^= b;` | Breaks if a and b are the same variable |
| Same sign? | `(a ^ b) >= 0` | Sign bits match → XOR is non-negative |
| Average without overflow | `(a & b) + ((a ^ b) >> 1)` | Shared bits + half of the differing bits |

### Counting set bits by hand (Brian Kernighan)

```java
int countBits(int n) {
    int count = 0;
    while (n != 0) {
        n &= n - 1;   // drop the lowest set bit
        count++;
    }
    return count;     // runs once per set bit; works for negatives too
}
```

## Problem patterns with Java templates

Nearly every interview bit problem is one of these eight patterns. Learn each template, then the problems in the plan become variations.

### 1. XOR cancellation

**Idea:** pairs cancel under XOR (`a ^ a = 0`), so whatever appears an odd number of times survives. Use it for "every element appears twice except…" and for missing numbers (XOR the indices with the values).

**Two-unique variant:** XOR everything to get `a ^ b`. Any set bit in that result is a position where a and b differ; split the array by that bit and XOR each half.

```java
int[] twoSingles(int[] nums) {
    int xor = 0;
    for (int x : nums) xor ^= x;          // xor = a ^ b
    int diff = xor & -xor;                // a bit where a and b differ
    int a = 0;
    for (int x : nums) if ((x & diff) != 0) a ^= x;
    return new int[]{a, xor ^ a};
}
```

### 2. Count bits per position

**Idea:** when XOR cannot cancel (things appear 3 times), count how many numbers have bit i set. The count mod k tells you the odd one's bit. Works for negatives because all 32 positions are handled.

```java
int singleAmongTriples(int[] nums) {
    int result = 0;
    for (int i = 0; i < 32; i++) {
        int count = 0;
        for (int x : nums) count += (x >> i) & 1;
        if (count % 3 != 0) result |= (1 << i);
    }
    return result;
}
```

### 3. Bitmask as a subset

**Idea:** with n ≤ 20 items, an `int` from 0 to 2^n - 1 represents every subset; bit j set means item j is chosen. Use it for "generate all subsets" and brute force over choices.

```java
for (int mask = 0; mask < (1 << n); mask++) {
    for (int j = 0; j < n; j++) {
        if ((mask & (1 << j)) != 0) { /* item j is in this subset */ }
    }
}

// All non-empty submasks of a given mask, largest first
for (int sub = mask; sub > 0; sub = (sub - 1) & mask) { /* ... */ }
```

### 4. Bitmask DP

**Idea:** the DP state is "which items are already used", stored as a mask. Signal: n ≤ 20 and you need an optimal ordering or assignment (TSP, assign n tasks to n people, partition into k groups). Cost is O(2^n × n).

```java
// Assign task j to person i (i = number of tasks already assigned), minimise total cost
int minCost(int[][] cost) {
    int n = cost.length, full = 1 << n;
    int[] dp = new int[full];
    Arrays.fill(dp, Integer.MAX_VALUE);
    dp[0] = 0;
    for (int mask = 0; mask < full; mask++) {
        if (dp[mask] == Integer.MAX_VALUE) continue;
        int person = Integer.bitCount(mask);
        if (person == n) continue;
        for (int task = 0; task < n; task++) {
            if ((mask & (1 << task)) == 0) {
                int next = mask | (1 << task);
                dp[next] = Math.min(dp[next], dp[mask] + cost[person][task]);
            }
        }
    }
    return dp[full - 1];
}
```

### 5. Greedy bit by bit, high to low (often with a binary trie)

**Idea:** a higher bit outweighs all lower bits combined (2^k > 2^k - 1). So to maximise an XOR, decide bits from the top and always try to make the current bit 1. A trie of bits answers "is the opposite bit available?" in O(32).

```java
class TrieNode { TrieNode[] child = new TrieNode[2]; }

int findMaximumXOR(int[] nums) {
    TrieNode root = new TrieNode();
    int best = 0;
    for (int x : nums) {
        TrieNode ins = root, q = root;
        int cur = 0;
        for (int i = 30; i >= 0; i--) {          // inputs are non-negative
            int b = (x >> i) & 1;
            if (ins.child[b] == null) ins.child[b] = new TrieNode();
            ins = ins.child[b];
            if (q.child[1 - b] != null) { cur |= (1 << i); q = q.child[1 - b]; }
            else q = q.child[b];
        }
        best = Math.max(best, cur);
    }
    return best;
}
```

### 6. Prefix XOR

**Idea:** like prefix sums, `xor(l..r) = px[r + 1] ^ px[l]`. To count subarrays with XOR k, store prefix XOR frequencies: a previous prefix p works when `p == cur ^ k`.

```java
long countSubarraysWithXor(int[] nums, int k) {
    Map<Integer, Integer> freq = new HashMap<>();
    freq.put(0, 1);
    int px = 0;
    long count = 0;
    for (int x : nums) {
        px ^= x;
        count += freq.getOrDefault(px ^ k, 0);
        freq.merge(px, 1, Integer::sum);
    }
    return count;
}
```

### 7. Bits as a fast set

**Idea:** 26 lowercase letters fit in one `int`. Two words share no letter when `(m1 & m2) == 0`; a state of "parity of each vowel" fits in 5 bits. Turns O(L) set checks into O(1).

```java
int mask = 0;
for (char c : word.toCharArray()) mask |= 1 << (c - 'a');
```

### 8. Bit-level arithmetic and DP

- **Counting bits 0..n:** `bits[i] = bits[i >> 1] + (i & 1)` — i has the bits of i/2 plus its last bit.
- **Bitwise AND of a range \[l, r\]:** the answer is the common binary prefix. `while (l < r) r &= r - 1; return r;`
- **Add without `+`:** XOR is the sum without carry; AND shifted left is the carry.

```java
int add(int a, int b) {
    while (b != 0) {
        int carry = (a & b) << 1;
        a ^= b;
        b = carry;
    }
    return a;   // Java int wraps, so negatives work
}
```

## Recognising a bit problem

The clue is usually in the constraints or the wording, not the topic tag. Scan for these signals before choosing an approach.

| Signal in the problem | Likely pattern |
| --- | --- |
| "appears twice except one", "missing number", O(1) extra space | 1. XOR cancellation |
| "appears three times except one" | 2. Count bits per position |
| n ≤ 20 (sometimes ≤ 15), "all subsets", "choose any group" | 3. Bitmask as a subset |
| n ≤ 20 and "minimum cost to visit/assign/partition all" | 4. Bitmask DP |
| "maximum XOR", "XOR with a query value" | 5. Greedy high-to-low + trie |
| "subarray XOR", "XOR of range l..r" | 6. Prefix XOR |
| Lowercase letters only, "no common letters", "even count of each vowel" | 7. Bits as a fast set |
| "power of two/four", "count 1 bits", "without using + or \*", "reverse bits" | 8. Arithmetic tricks from the cheat sheet |
| AND / OR of a whole range or subarray | 8. Common prefix, or track each bit's count in a sliding window |

When stuck, ask three questions: Can I treat each bit position independently? Does a higher bit dominate all lower ones (then go greedy from the top)? Is n small enough that 2^n states fit (2^20 ≈ 10^6)?

## 4-week workout plan

Four weeks, 60–75 minutes a day, going from single-bit reflexes to hard bitmask DP. Each day: 10 minutes of drills (next section), then the listed problems. Tick a day off only when you can explain every solution out loud without notes.

Rules for each problem: 20 minutes on Easy and 35 on Medium before reading a hint; after a hint, close it and code from scratch; after solving, write the one-line "trick" in the mistake log even if you got it right.

### Week 1 — Reflexes: single-bit tricks and XOR basics (Easy)

Goal: predict the output of any cheat-sheet one-liner in under 5 seconds.

- [ ] Day 1 — Read Foundations; check, count, power of two: [191 Number of 1 Bits](https://leetcode.com/problems/number-of-1-bits/) · [231 Power of Two](https://leetcode.com/problems/power-of-two/) · [461 Hamming Distance](https://leetcode.com/problems/hamming-distance/)
- [ ] Day 2 — Masks and flips: [342 Power of Four](https://leetcode.com/problems/power-of-four/) · [476 Number Complement](https://leetcode.com/problems/number-complement/) · [2220 Minimum Bit Flips](https://leetcode.com/problems/minimum-bit-flips-to-convert-number/) · [693 Alternating Bits](https://leetcode.com/problems/binary-number-with-alternating-bits/)
- [ ] Day 3 — Shifts and bit DP: [338 Counting Bits](https://leetcode.com/problems/counting-bits/) · [1356 Sort by Number of 1 Bits](https://leetcode.com/problems/sort-integers-by-the-number-of-1-bits/) · [190 Reverse Bits](https://leetcode.com/problems/reverse-bits/)
- [ ] Day 4 — XOR cancellation: [136 Single Number](https://leetcode.com/problems/single-number/) · [268 Missing Number](https://leetcode.com/problems/missing-number/) · [389 Find the Difference](https://leetcode.com/problems/find-the-difference/) · [1720 Decode XORed Array](https://leetcode.com/problems/decode-xored-array/)
- [ ] Day 5 — XOR undo and carry: [645 Set Mismatch](https://leetcode.com/problems/set-mismatch/) (solve with XOR) · [2433 Original Array of Prefix XOR](https://leetcode.com/problems/find-the-original-array-of-prefix-xor/) · [371 Sum of Two Integers](https://leetcode.com/problems/sum-of-two-integers/)
- [ ] Day 6 — Timed test, 45 minutes, no hints: [2683 Neighboring Bitwise XOR](https://leetcode.com/problems/neighboring-bitwise-xor/) · [1486 XOR Operation in an Array](https://leetcode.com/problems/xor-operation-in-an-array/) · [762 Prime Number of Set Bits](https://leetcode.com/problems/prime-number-of-set-bits-in-binary-representation/)
- [ ] Day 7 — Review: re-solve every logged miss; rewrite the cheat sheet from memory

### Week 2 — Thinking per bit position (Medium)

Goal: when XOR alone fails, switch to "treat each of the 32 bits independently".

- [ ] Day 1 — [137 Single Number II](https://leetcode.com/problems/single-number-ii/) · [260 Single Number III](https://leetcode.com/problems/single-number-iii/)
- [ ] Day 2 — [201 Bitwise AND of Numbers Range](https://leetcode.com/problems/bitwise-and-of-numbers-range/) · [477 Total Hamming Distance](https://leetcode.com/problems/total-hamming-distance/)
- [ ] Day 3 — Prefix XOR: [1310 XOR Queries of a Subarray](https://leetcode.com/problems/xor-queries-of-a-subarray/) · [1442 Count Triplets with Equal XOR](https://leetcode.com/problems/count-triplets-that-can-form-two-arrays-of-equal-xor/)
- [ ] Day 4 — [1734 Decode XORed Permutation](https://leetcode.com/problems/decode-xored-permutation/) · [2275 Largest Combination with AND > 0](https://leetcode.com/problems/largest-combination-with-bitwise-and-greater-than-zero/)
- [ ] Day 5 — [287 Find the Duplicate Number](https://leetcode.com/problems/find-the-duplicate-number/) (solve with per-bit counts) · [29 Divide Two Integers](https://leetcode.com/problems/divide-two-integers/)
- [ ] Day 6 — Timed test, 60 minutes: [1835 XOR Sum of All Pairs AND](https://leetcode.com/problems/find-xor-sum-of-all-pairs-bitwise-and/) · [2401 Longest Nice Subarray](https://leetcode.com/problems/longest-nice-subarray/) · [89 Gray Code](https://leetcode.com/problems/gray-code/)
- [ ] Day 7 — Review: re-solve logged misses; explain 137 and 260 out loud as if in an interview

### Week 3 — Masks as sets and bitmask DP

Goal: see "n ≤ 20" and immediately think "2^n states".

- [ ] Day 1 — Subset enumeration: [78 Subsets](https://leetcode.com/problems/subsets/) (bitmask version) · [1863 Sum of All Subset XOR Totals](https://leetcode.com/problems/sum-of-all-subset-xor-totals/) · [2044 Count Max Bitwise-OR Subsets](https://leetcode.com/problems/count-number-of-maximum-bitwise-or-subsets/)
- [ ] Day 2 — Letters as bits: [318 Maximum Product of Word Lengths](https://leetcode.com/problems/maximum-product-of-word-lengths/) · [1239 Max Length of Concatenated String with Unique Chars](https://leetcode.com/problems/maximum-length-of-a-concatenated-string-with-unique-characters/)
- [ ] Day 3 — Parity masks + prefix: [1371 Vowels in Even Counts](https://leetcode.com/problems/find-the-longest-substring-containing-vowels-in-even-counts/) · [1915 Number of Wonderful Substrings](https://leetcode.com/problems/number-of-wonderful-substrings/)
- [ ] Day 4 — Bitmask DP: [698 Partition to K Equal Sum Subsets](https://leetcode.com/problems/partition-to-k-equal-sum-subsets/) · [1986 Minimum Work Sessions](https://leetcode.com/problems/minimum-number-of-work-sessions-to-finish-the-tasks/)
- [ ] Day 5 — Bitmask DP + BFS: [847 Shortest Path Visiting All Nodes](https://leetcode.com/problems/shortest-path-visiting-all-nodes/) · [1125 Smallest Sufficient Team](https://leetcode.com/problems/smallest-sufficient-team/) (Hard)
- [ ] Day 6 — Timed test, 45 minutes: [464 Can I Win](https://leetcode.com/problems/can-i-win/)
- [ ] Day 7 — Review: re-solve logged misses; write the bitmask DP template from memory

### Week 4 — Greedy high-to-low, subarray OR/AND, mixed mocks

Goal: handle unseen Medium/Hard bit problems under interview time pressure.

- [ ] Day 1 — Binary trie: [421 Maximum XOR of Two Numbers](https://leetcode.com/problems/maximum-xor-of-two-numbers-in-an-array/) · [1707 Maximum XOR With an Element](https://leetcode.com/problems/maximum-xor-with-an-element-from-array/) (Hard)
- [ ] Day 2 — OR/AND over subarrays: [898 Bitwise ORs of Subarrays](https://leetcode.com/problems/bitwise-ors-of-subarrays/) · [2411 Smallest Subarrays With Max OR](https://leetcode.com/problems/smallest-subarrays-with-maximum-bitwise-or/)
- [ ] Day 3 — Submask enumeration: [1178 Number of Valid Words for Each Puzzle](https://leetcode.com/problems/number-of-valid-words-for-each-puzzle/) · [1494 Parallel Courses II](https://leetcode.com/problems/parallel-courses-ii/) (both Hard)
- [ ] Day 4 — [1680 Concatenation of Consecutive Binary Numbers](https://leetcode.com/problems/concatenation-of-consecutive-binary-numbers/) · [2419 Longest Subarray With Max AND](https://leetcode.com/problems/longest-subarray-with-maximum-bitwise-and/)
- [ ] Day 5 — Mock, 60 minutes, talk out loud: [1829 Maximum XOR for Each Query](https://leetcode.com/problems/maximum-xor-for-each-query/) · [2997 Make Array XOR Equal to K](https://leetcode.com/problems/minimum-number-of-operations-to-make-array-xor-equal-to-k/) · [1611 Minimum One Bit Operations](https://leetcode.com/problems/minimum-one-bit-operations-to-make-integers-zero/) (stretch)
- [ ] Day 6 — Re-solve the five hardest entries in your mistake log, cold
- [ ] Day 7 — Rest, or explain one problem from each pattern to a friend (or a rubber duck)

After week 4, keep the skill alive with 2 random bit problems a week from LeetCode's Bit Manipulation tag.

## Drills, review routine and mistake log

Ten minutes of paper drills a day builds the instinct that problems then rely on. Do them before the day's problems, without running code.

### Daily 10-minute drills

1. **Conversion (2 min):** convert 3 random numbers between 0 and 255 to binary and back. Include one negative 8-bit number in two's complement.
2. **Predict (4 min):** pick a number and predict, on paper, `x & (x - 1)`, `x & -x`, `x ^ (x >> 1)`, `x | (x + 1)` and `x >>> 28` for a negative x. Then check with `Integer.toBinaryString`.
3. **Write from memory (4 min):** rotate through one template a day — Kernighan count, two-singles split, per-bit count, submask loop, bitmask DP, trie insert/query, prefix XOR count.

Answer key for 2, with x = 22 (`0001 0110`): `x & (x - 1)` = 20 (`0001 0100`); `x & -x` = 2; `x ^ (x >> 1)` = 29 (`0001 1101`, the Gray code of 22); `x | (x + 1)` = 23 (`0001 0111`).

### Review routine

- Re-solve every logged problem 3 days later, then 7 days later, from a blank editor.
- A problem leaves the log only after two clean re-solves.
- Each Sunday, read your "trick" column top to bottom; patterns you miss twice get an extra problem from the same section.

### Mistake log

Add one row per problem you missed or solved slowly. The "Trick" column is the one sentence you would want whispered to you next time.

| Date | Problem | Pattern | What went wrong | Trick | Re-solve day 3 | Re-solve day 7 |
| --- | --- | --- | --- | --- | --- | --- |
| 2026-10-12 | Example: 260 Single Number III | 1. XOR cancellation | Didn't see how to separate a and b | Any set bit of a ^ b splits the array into two groups |  |  |
|  |  |  |  |  |  |  |
|  |  |  |  |  |  |  |
