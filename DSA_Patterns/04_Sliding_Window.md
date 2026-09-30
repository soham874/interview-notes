# Sliding Window

Keep a contiguous range `[l, r]` and update its state in O(1) as elements enter on the right and leave on the left. This turns an O(n²) scan over every subarray into O(n), because each index enters and leaves the window at most once.

## When to recognize it

- The question asks about a **contiguous** subarray or substring.
- It wants the longest, shortest, count, or max/min of such a range.
- The validity condition can be updated incrementally when one element enters or leaves.
- Growing the window can only make it "more invalid" (or only "more valid"). That monotonic behaviour is what lets `l` move forward and never come back.

## Fixed-size window

Add `a[r]`; once the window reaches size `k`, record the answer and remove `a[r - k + 1]`.

<details>
<summary>Template (Java)</summary>

```java
int sum = 0, best = Integer.MIN_VALUE;
for (int r = 0; r < n; r++) {
    sum += a[r];                        // add incoming
    if (r >= k - 1) {
        best = Math.max(best, sum);     // window is full
        sum -= a[r - k + 1];            // remove outgoing
    }
}
```

</details>

### Maximum Average Subarray I (LC 643)

[LeetCode 643](https://leetcode.com/problems/maximum-average-subarray-i/) · Easy · O(n) time, O(1) space

**Tags:** Sliding Window

**Hint:** Keep a running sum; subtract the element that falls out of the window.

### Find All Anagrams in a String (LC 438)

[LeetCode 438](https://leetcode.com/problems/find-all-anagrams-in-a-string/) · Medium · O(n) time, O(1) space

**Tags:** Sliding Window

**Hint:** Window of size `|p|`; compare two `int[26]` frequency arrays.

**Trap:** Comparing 26 slots every step is fine, but a `matches` counter makes each step O(1).

<details>
<summary>Why it works</summary>

Each slide changes the count of only two characters (one enters, one leaves), so the window's frequency array can be updated in O(1) instead of being rebuilt for every start index.

</details>

<details>
<summary>Approach</summary>

1. Build `pCount[26]` from `p`.
2. Slide `r` over `s` and increment `sCount[s[r]]`.
3. Once `r ≥ |p|`, decrement `sCount[s[r − |p|]]`.
4. If `sCount` equals `pCount`, add index `r − |p| + 1`.

</details>

### Sliding Window Maximum (LC 239)

[LeetCode 239](https://leetcode.com/problems/sliding-window-maximum/) · Hard · O(n) time, O(k) space

**Tags:** Sliding Window · Monotonic Stack & Queue

**Hint:** Keep a deque of indices whose values decrease from front to back; the front is the window's maximum.

**Glue:** The window decides which indices are still allowed; the monotonic deque answers "max of what's allowed" in O(1).

**Trap:** Pop the front when its index is `≤ r − k`, i.e. it has left the window.

<details>
<summary>Why it works</summary>

If a newer element is at least as big as an older one, the older one can never be the maximum again while both are in the window, so discarding it loses nothing. Every index is pushed once and popped at most once, which gives O(n) overall.

</details>

<details>
<summary>Approach</summary>

1. The deque stores indices; their values stay decreasing from front to back.
2. For each `r`: pop from the back while `a[back] ≤ a[r]`, then push `r`.
3. If the front index is outside the window (`≤ r − k`), pop it.
4. Once `r ≥ k − 1`, the answer for this window is `a[front]`.

</details>

<details>
<summary>Code (Java)</summary>

```java
public int[] maxSlidingWindow(int[] a, int k) {
    int n = a.length;
    Deque<Integer> dq = new ArrayDeque<>();
    int[] res = new int[n - k + 1];
    for (int r = 0; r < n; r++) {
        while (!dq.isEmpty() && a[dq.peekLast()] <= a[r]) dq.pollLast();
        dq.offerLast(r);
        if (dq.peekFirst() <= r - k) dq.pollFirst();
        if (r >= k - 1) res[r - k + 1] = a[dq.peekFirst()];
    }
    return res;
}
```

</details>

## Longest valid window

Expand `r`; while the window is invalid, shrink `l`; then update the answer. The answer is updated **after** the shrinking loop, because only then is the window valid.

<details>
<summary>Template (Java)</summary>

```java
int l = 0, ans = 0;
for (int r = 0; r < n; r++) {
    // add a[r] to the window state
    while (/* window invalid */) {
        // remove a[l] from the window state
        l++;
    }
    ans = Math.max(ans, r - l + 1);     // after shrinking
}
```

</details>

### Longest Substring Without Repeating Characters (LC 3)

[LeetCode 3](https://leetcode.com/problems/longest-substring-without-repeating-characters/) · Medium · O(n) time, O(min(n, alphabet)) space

**Tags:** Sliding Window · Hashing

**Hint:** Map each character to its last index; jump `l` to `max(l, last + 1)`.

**Glue:** The hashmap remembers where each character was last seen, so the window can skip past a repeat in one step instead of shrinking one index at a time.

**Trap:** Use `max()` so `l` never moves backward when the repeated character is already outside the window.

<details>
<summary>Why it works</summary>

Every character in `[l, r]` must be unique. If `s[r]` was last seen inside the window, everything up to and including that position has to leave, so `l` can jump there directly instead of stepping one index at a time.

</details>

<details>
<summary>Approach</summary>

1. Keep a map from character to its last index.
2. For each `r`: if `s[r]` was seen at index `j ≥ l`, set `l = j + 1`.
3. Store `last[s[r]] = r`.
4. `ans = max(ans, r − l + 1)`.

</details>

### Longest Repeating Character Replacement (LC 424)

[LeetCode 424](https://leetcode.com/problems/longest-repeating-character-replacement/) · Medium · O(n) time, O(1) space

**Tags:** Sliding Window

**Hint:** The window is valid while `length − maxFreq ≤ k`.

**Trap:** `maxFreq` does not need to decrease when the window shrinks.

<details>
<summary>Why it works</summary>

The answer can only get longer when some window has a higher `maxFreq` than any seen before. A stale, too-high `maxFreq` only stops the window from shrinking, so the window slides at its current best length instead of reporting a longer wrong answer. Recomputing `maxFreq` exactly is never needed.

</details>

<details>
<summary>Approach</summary>

1. Keep `count[26]`, `maxFreq` and `l = 0`.
2. Add `s[r]` and set `maxFreq = max(maxFreq, count[s[r]])`.
3. If `(r − l + 1) − maxFreq > k`, decrement `count[s[l]]` and move `l` by one. A single `if` is enough, because the window only slides.
4. `ans = max(ans, r − l + 1)`.

</details>

<details>
<summary>Code (Java)</summary>

```java
public int characterReplacement(String s, int k) {
    int[] cnt = new int[26];
    int l = 0, maxF = 0, ans = 0;
    for (int r = 0; r < s.length(); r++) {
        maxF = Math.max(maxF, ++cnt[s.charAt(r) - 'A']);
        if (r - l + 1 - maxF > k) cnt[s.charAt(l++) - 'A']--;
        ans = Math.max(ans, r - l + 1);
    }
    return ans;
}
```

</details>

### Max Consecutive Ones III (LC 1004)

[LeetCode 1004](https://leetcode.com/problems/max-consecutive-ones-iii/) · Medium · O(n) time, O(1) space

**Tags:** Sliding Window

**Hint:** The window is valid while the number of zeros in it is `≤ k`.

### Fruit Into Baskets (LC 904)

[LeetCode 904](https://leetcode.com/problems/fruit-into-baskets/) · Medium · O(n) time, O(1) space

**Tags:** Sliding Window

**Hint:** Longest window with at most 2 distinct values.

**Trap:** Remove a key from the map when its count hits 0, or `map.size()` overcounts.

## Shortest valid window

Expand `r`; while the window is valid, update the answer, then shrink `l`. The answer is updated **inside** the shrinking loop, because every step of it is still a valid window.

<details>
<summary>Template (Java)</summary>

```java
int l = 0, ans = Integer.MAX_VALUE;
for (int r = 0; r < n; r++) {
    // add a[r] to the window state
    while (/* window valid */) {
        ans = Math.min(ans, r - l + 1); // inside the loop
        // remove a[l] from the window state
        l++;
    }
}
return ans == Integer.MAX_VALUE ? 0 : ans;
```

</details>

### Minimum Size Subarray Sum (LC 209)

[LeetCode 209](https://leetcode.com/problems/minimum-size-subarray-sum/) · Medium · O(n) time, O(1) space

**Tags:** Sliding Window

**Hint:** While `sum ≥ target`, record the length and shrink.

**Trap:** This only works because every number is positive.

<details>
<summary>Why it works</summary>

With positive numbers, growing the window always raises the sum and shrinking always lowers it. Once `[l, r]` is valid, no window starting at `l` and ending later can be shorter, so `l` can safely move on.

</details>

### Minimum Window Substring (LC 76)

[LeetCode 76](https://leetcode.com/problems/minimum-window-substring/) · Hard · O(|s| + |t|) time, O(1) space

**Tags:** Sliding Window

**Hint:** Keep `need[]` counts plus a `missing` counter; the window is valid when `missing == 0`.

**Trap:** Only decrement `missing` when `need` was positive; surplus copies push `need` below 0.

<details>
<summary>Why it works</summary>

`need[c]` tracks how many more `c`s the window still owes. It goes negative for surplus copies, so a single counter of still-owed characters tells you whether the window is valid in O(1), without comparing maps.

</details>

<details>
<summary>Approach</summary>

1. Fill `need[]` from `t`; set `missing = t.length()`.
2. Add `s[r]`: if `need[s[r]] > 0`, decrement `missing`. Then decrement `need[s[r]]`.
3. While `missing == 0`: record the window if it's shorter, then remove `s[l]`: increment `need[s[l]]`, and if it becomes `> 0`, increment `missing`. Move `l`.
4. Return the best substring, or `""` if there was none.

</details>

<details>
<summary>Code (Java)</summary>

```java
public String minWindow(String s, String t) {
    int[] need = new int[128];
    for (char c : t.toCharArray()) need[c]++;
    int missing = t.length(), l = 0, start = 0, best = Integer.MAX_VALUE;
    for (int r = 0; r < s.length(); r++) {
        if (need[s.charAt(r)]-- > 0) missing--;
        while (missing == 0) {
            if (r - l + 1 < best) { best = r - l + 1; start = l; }
            if (++need[s.charAt(l++)] > 0) missing++;
        }
    }
    return best == Integer.MAX_VALUE ? "" : s.substring(start, start + best);
}
```

</details>

## Counting subarrays

`exactly(k) = atMost(k) − atMost(k − 1)`. In `atMost`, every `r` adds `r − l + 1` subarrays: all the windows that end at `r` and start anywhere in `[l, r]`.

<details>
<summary>Template (Java)</summary>

```java
int exactly(int[] a, int k) {
    return atMost(a, k) - atMost(a, k - 1);
}

int atMost(int[] a, int k) {
    if (k < 0) return 0;
    int l = 0, count = 0;
    for (int r = 0; r < a.length; r++) {
        // add a[r]
        while (/* more than k */) { /* remove a[l] */ l++; }
        count += r - l + 1;             // subarrays ending at r
    }
    return count;
}
```

</details>

### Subarrays with K Different Integers (LC 992)

[LeetCode 992](https://leetcode.com/problems/subarrays-with-k-different-integers/) · Hard · O(n) time, O(n) space

**Tags:** Sliding Window

**Hint:** `atMost(k distinct) − atMost(k − 1 distinct)`.

<details>
<summary>Why it works</summary>

"Exactly k" is not monotonic: shrinking a window can drop it from k to k − 1 distinct values, so a single window can't count it directly. "At most k" is monotonic, and every window that ends at `r` and starts at or after `l` is valid, which gives `r − l + 1` new subarrays per step. Subtracting the two counts leaves exactly k.

</details>

<details>
<summary>Approach</summary>

1. Write `atMost(k)` with a frequency map and a `distinct` counter.
2. For each `r`: add `a[r]`; if its count became 1, increment `distinct`.
3. While `distinct > k`: remove `a[l]`; if its count hit 0, decrement `distinct`; move `l`.
4. `count += r − l + 1`. Return `atMost(k) − atMost(k − 1)`.

</details>

### Count Number of Nice Subarrays (LC 1248)

[LeetCode 1248](https://leetcode.com/problems/count-number-of-nice-subarrays/) · Medium · O(n) time, O(1) space

**Tags:** Sliding Window

**Hint:** Map odd → 1 and even → 0; it becomes "binary subarrays with sum k".

### Binary Subarrays With Sum (LC 930)

[LeetCode 930](https://leetcode.com/problems/binary-subarrays-with-sum/) · Medium · O(n) time, O(1) space

**Tags:** Sliding Window

**Hint:** `atMost(goal) − atMost(goal − 1)`.

**Trap:** Guard `atMost(−1)` and return 0.

## Doesn't work when

- **Negative numbers in "sum = k" or "sum ≥ k" questions.** Shrinking no longer reliably lowers the sum, so `l` can't move greedily. Use prefix sums with a hashmap instead, as in Subarray Sum Equals K (LC 560), or a monotonic deque over prefix sums for "shortest with sum ≥ k" (LC 862).

## My mistakes

- Longest: update the answer **after** the while loop. Shortest: update it **inside**.
- Window length is `r - l + 1`, not `r - l`.
- When shrinking, remove `a[l]` from the state **before** incrementing `l`.
- Delete a map key when its count hits 0 if the condition uses `map.size()`.
