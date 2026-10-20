# Dynamic Programming: Study Notes & Workout Plan

Oct 20, 2026 · @Soham Choudhury

## How to use this guide

DP is not memorising solutions; it is learning to define a state and a recurrence. About 10 patterns cover almost every interview DP question, and this guide drills them in order of difficulty.

1. Read Foundations and the 5-step recipe once, slowly. Work the example by hand.
2. Follow the 6-week plan: about 60–75 minutes a day, 6 days a week, one review day. DP is broader than bit manipulation, so it gets two extra weeks.
3. For every problem, write the state and recurrence in plain English **before** any code. If you cannot say what `dp[i]` means in one sentence, you are not ready to code.
4. Solve top-down (memoised recursion) first, then convert to bottom-up. Interviewers often ask for both, and the conversion is where understanding sticks.
5. Log every miss in the mistake log at the end and re-solve after 3 and 7 days.

All code is Java.

## Foundations

DP is recursion where each distinct subproblem is solved once and its answer reused. It applies when a problem has both properties below.

### The two properties

- **Optimal substructure:** the best answer for the whole is built from best answers to smaller pieces. Shortest path to C through B uses the shortest path to B.
- **Overlapping subproblems:** plain recursion would solve the same piece many times. Naive Fibonacci calls `fib(3)` repeatedly; caching turns O(2^n) into O(n).

If subproblems don't overlap (merge sort), it is divide and conquer, not DP. If a locally best choice is always safe, greedy is simpler; DP is for when you must try options and compare.

### Top-down vs bottom-up

|  | Top-down (memoisation) | Bottom-up (tabulation) |
| --- | --- | --- |
| How | Recursion + cache | Loops filling a table from base cases up |
| Easiest when | You can see the recurrence but not the fill order | The fill order is obvious (i increasing, grid row by row) |
| Computes | Only states actually reached | Every state in the table |
| Risk | Stack overflow for depth over \~10^4 in Java | Wrong loop order reads unfilled cells |
| Space optimisation | Hard | Easy: keep only the previous row(s) |

Interview default: write top-down first because it mirrors your reasoning, then convert if asked or if depth is large.

### Same problem both ways — climbing stairs (1 or 2 steps)

```java
// Top-down
int[] memo;
int ways(int i) {                      // ways to reach step i
    if (i <= 1) return 1;
    if (memo[i] != 0) return memo[i];
    return memo[i] = ways(i - 1) + ways(i - 2);
}

// Bottom-up with O(1) space
int climbStairs(int n) {
    int prev2 = 1, prev1 = 1;          // ways(0), ways(1)
    for (int i = 2; i <= n; i++) {
        int cur = prev1 + prev2;
        prev2 = prev1;
        prev1 = cur;
    }
    return prev1;
}
```

### Java gotchas that cost interviews

- **Memo sentinel:** if 0 is a valid answer, `memo[i] != 0` re-computes forever. Fill with -1 (`Arrays.fill`) or use `Integer[]` / `Long[]` with `null`.
- **Overflow:** counting problems blow past `int` quickly. Use `long` and apply `% 1_000_000_007` after **every** addition.
- **Infinity for min:** `Integer.MAX_VALUE + 1` wraps negative. Use a safe large value like `1_000_000_000` or `n + 1`, or check before adding.
- **Multi-dimensional memo:** `int[][] memo` needs each row filled: `for (int[] row : memo) Arrays.fill(row, -1);`.
- **HashMap keys:** prefer an array index; if you must use a map, build a `long` key (`i * 100_001L + j`), not a String, which is slow.
- **Recursion depth:** Java's default stack handles roughly 5,000–10,000 frames. For n = 10^5, go bottom-up.
- **Off-by-one in sizes:** for string problems, `dp` is usually `(n + 1) × (m + 1)` so that row 0 means "empty prefix".

## The 5-step DP recipe

Run every problem through these five steps, in this order, and say each one out loud in interviews. Step 1 is where almost all difficulty lives.

1. **State:** what does `dp[i]` (or `dp[i][j]`) mean? Write it as a sentence: "the max money robbing houses 0..i". Choose the fewest variables that still capture everything the future depends on.
2. **Transition:** how is a state built from smaller states? Ask "what is the last decision?" and list every option for it.
3. **Base cases:** the smallest states you can answer directly (empty prefix, i = 0, a single cell).
4. **Order:** fill states so every dependency is computed first. Top-down handles this for you.
5. **Answer and optimise:** which state is the final answer (`dp[n - 1]`, `max(dp)`, `dp[0][n - 1]`)? Then check whether only the last row or two are needed, and cut space.

### Worked example — House Robber (no two adjacent houses)

Input `[2, 7, 9, 3, 1]`.

1. **State:** `dp[i]` = the most money from houses 0..i.
2. **Transition:** last decision is "rob house i or not". Skip → `dp[i - 1]`. Rob → `nums[i] + dp[i - 2]`. Take the max.
3. **Base:** `dp[0] = 2`, `dp[1] = max(2, 7) = 7`.
4. **Order:** i increasing.
5. **Answer:** `dp[4]`. Only two previous values are used, so space drops to O(1).

| i | nums\[i\] | skip = dp\[i-1\] | rob = nums\[i\] + dp\[i-2\] | dp\[i\] |
| --- | --- | --- | --- | --- |
| 0 | 2 | — | — | 2 |
| 1 | 7 | 2 | 7 | 7 |
| 2 | 9 | 7 | 11 | 11 |
| 3 | 3 | 11 | 10 | 11 |
| 4 | 1 | 11 | 12 | 12 |

```java
int rob(int[] nums) {
    int prev2 = 0, prev1 = 0;      // dp[i-2], dp[i-1]
    for (int x : nums) {
        int cur = Math.max(prev1, prev2 + x);
        prev2 = prev1;
        prev1 = cur;
    }
    return prev1;
}
```

### Finding the state when you're stuck

- Write the brute-force recursion first: `solve(index, ...whatever else changes)`. Its parameters **are** the state.
- If a parameter is a whole list or set, replace it with something smaller: a last-chosen index, a count, a remaining capacity, or a bitmask when n ≤ 20.
- Count states × work per state to get complexity. 10^7 total operations is the comfortable ceiling in Java.

## Problem patterns with Java templates

Ten patterns cover nearly every interview DP. For each: what the state means, the recurrence, and a template to adapt.

### 1. Linear (1D) DP

**State:** `dp[i]` = answer for the prefix ending at or before i. **Recurrence:** combine a few earlier values (`dp[i-1]`, `dp[i-2]`). Examples: climbing stairs, house robber, decode ways. Template: see the recipe's House Robber — almost always O(1) space.

### 2. 0/1 knapsack (each item used at most once)

**State:** `dp[c]` = best value (or reachable?) with capacity c. **Key detail:** loop capacity **downwards** so each item is counted once.

```java
// Subset sum: can some subset reach target?
boolean canReach(int[] nums, int target) {
    boolean[] dp = new boolean[target + 1];
    dp[0] = true;
    for (int x : nums)
        for (int s = target; s >= x; s--)   // downwards!
            dp[s] |= dp[s - x];
    return dp[target];
}
```

### 3. Unbounded knapsack (items reusable)

**State:** `dp[a]` = best answer for amount a. **Key detail:** loop amount **upwards** so an item can be reused. For counting, loop order decides combinations vs permutations.

```java
// Fewest coins to make amount
int coinChange(int[] coins, int amount) {
    int[] dp = new int[amount + 1];
    Arrays.fill(dp, amount + 1);            // "infinity" that can't overflow
    dp[0] = 0;
    for (int a = 1; a <= amount; a++)
        for (int c : coins)
            if (c <= a) dp[a] = Math.min(dp[a], dp[a - c] + 1);
    return dp[amount] > amount ? -1 : dp[amount];
}

// Count ways: coins OUTER = combinations (1+2 same as 2+1)
for (int c : coins) for (int a = c; a <= amount; a++) ways[a] += ways[a - c];
// amount OUTER, coins inner = permutations (order matters)
```

### 4. Longest increasing subsequence (LIS)

**State:** `dp[i]` = length of the longest increasing subsequence ending at i; `dp[i] = 1 + max(dp[j])` for j < i with `a[j] < a[i]`. O(n^2). The O(n log n) version keeps the smallest possible tail for each length.

```java
int lengthOfLIS(int[] nums) {
    List<Integer> tails = new ArrayList<>();
    for (int x : nums) {
        int i = Collections.binarySearch(tails, x);
        if (i < 0) i = -(i + 1);             // first tail >= x
        if (i == tails.size()) tails.add(x);
        else tails.set(i, x);
    }
    return tails.size();
}
```

Variants: Russian dolls (sort, then LIS), longest chain, number of LIS.

### 5. Grid DP

**State:** `dp[i][j]` = answer for reaching cell (i, j). **Recurrence:** from the top and the left (or whichever moves are allowed). One row of space is enough.

```java
int minPathSum(int[][] g) {
    int m = g.length, n = g[0].length;
    int[] dp = new int[n];
    for (int i = 0; i < m; i++)
        for (int j = 0; j < n; j++) {
            if (i == 0 && j == 0) dp[j] = g[0][0];
            else if (i == 0) dp[j] = dp[j - 1] + g[i][j];
            else if (j == 0) dp[j] = dp[j] + g[i][j];
            else dp[j] = Math.min(dp[j], dp[j - 1]) + g[i][j];
        }
    return dp[n - 1];
}
```

### 6. Two sequences (LCS, edit distance)

**State:** `dp[i][j]` = answer for the first i chars of a and first j chars of b. **Recurrence:** compare the last characters; if equal, use `dp[i-1][j-1]`; otherwise take the best of dropping one from either side.

```java
int editDistance(String a, String b) {
    int n = a.length(), m = b.length();
    int[][] dp = new int[n + 1][m + 1];
    for (int i = 0; i <= n; i++) dp[i][0] = i;   // delete all
    for (int j = 0; j <= m; j++) dp[0][j] = j;   // insert all
    for (int i = 1; i <= n; i++)
        for (int j = 1; j <= m; j++)
            if (a.charAt(i - 1) == b.charAt(j - 1)) dp[i][j] = dp[i - 1][j - 1];
            else dp[i][j] = 1 + Math.min(dp[i - 1][j - 1],          // replace
                                Math.min(dp[i - 1][j], dp[i][j - 1])); // delete, insert
    return dp[n][m];
}
// LCS: equal → dp[i-1][j-1] + 1; else max(dp[i-1][j], dp[i][j-1])
```

### 7. Interval DP

**State:** `dp[l][r]` = answer for the sub-range l..r. **Recurrence:** try every split point k, or the element handled last. **Order:** by increasing length — the classic mistake is looping l and r directly. Examples: burst balloons, palindromic substrings, matrix-chain, minimum cost to cut a stick.

```java
int maxCoins(int[] nums) {
    int n = nums.length;
    int[] a = new int[n + 2];
    a[0] = a[n + 1] = 1;
    for (int i = 0; i < n; i++) a[i + 1] = nums[i];
    int[][] dp = new int[n + 2][n + 2];         // dp[l][r]: best for balloons strictly between l and r
    for (int len = 2; len <= n + 1; len++)
        for (int l = 0; l + len <= n + 1; l++) {
            int r = l + len;
            for (int k = l + 1; k < r; k++)      // k = last balloon burst in (l, r)
                dp[l][r] = Math.max(dp[l][r], dp[l][k] + dp[k][r] + a[l] * a[k] * a[r]);
        }
    return dp[0][n + 1];
}
```

### 8. State machine DP

**State:** a few named variables for "which mode am I in" (holding a stock, sold today, resting). **Recurrence:** each day, every mode is computed from the previous day's modes. Covers all the stock problems.

```java
int maxProfitWithCooldown(int[] prices) {
    int hold = Integer.MIN_VALUE / 2, sold = 0, rest = 0;
    for (int p : prices) {
        int prevHold = hold, prevSold = sold;
        hold = Math.max(hold, rest - p);   // keep holding, or buy after resting
        sold = prevHold + p;               // sell today
        rest = Math.max(rest, prevSold);   // cooldown after a sale
    }
    return Math.max(sold, rest);
}
```

### 9. Tree DP

**State:** each node returns a small array describing its subtree, such as {best if I take this node, best if I skip it}. **Order:** post-order DFS, children before parent.

```java
int[] dfs(TreeNode node) {                 // {rob this node, skip this node}
    if (node == null) return new int[]{0, 0};
    int[] L = dfs(node.left), R = dfs(node.right);
    int rob = node.val + L[1] + R[1];
    int skip = Math.max(L[0], L[1]) + Math.max(R[0], R[1]);
    return new int[]{rob, skip};
}
```

### 10. Bitmask DP

**State:** `dp[mask]` where the mask records which of n ≤ 20 items are used. Covered in full, with a template, in pattern 4 of Bit Manipulation: Study Notes & Workout Plan.

### Beyond the ten (only if time remains)

Digit DP (count numbers ≤ N with a property), DP with a monotonic deque (sliding-window max inside a recurrence), and probability/expected-value DP. Rare in most interviews; skip until the plan is done.

## Recognising a DP problem and its pattern

The question words "number of ways", "minimum/maximum", "is it possible" plus choices that affect later choices almost always mean DP. Then the input shape picks the pattern.

| Signal in the problem | Likely pattern |
| --- | --- |
| One array or string, each step depends on the last 1–2 steps | 1. Linear DP |
| Pick a subset of items, each once, to hit or stay under a total | 2. 0/1 knapsack |
| Unlimited copies of each item (coins, pieces) | 3. Unbounded knapsack |
| "Longest increasing / chain / nested" subsequence | 4. LIS |
| 2D grid, moves only right/down, "paths" or "min cost path" | 5. Grid DP |
| Two strings or arrays, "common", "convert", "match", "interleave" | 6. Two sequences |
| Merge, burst, cut or partition a range; answer depends on a split point | 7. Interval DP |
| Buy/sell, cooldown, "at most k transactions", on/off modes | 8. State machine DP |
| Answer for a tree combining children's answers | 9. Tree DP |
| n ≤ 20 and "use all items in some order" | 10. Bitmask DP |

**DP or something else?** If a local best choice is always safe (intervals by end time, jump game reachability), try greedy first. If the input is a general graph with costs, think shortest paths. If n ≤ 15 and the problem asks to list every solution, it is backtracking — DP counts or optimises, it does not enumerate.

## 6-week workout plan

Six weeks, 60–75 minutes a day, one pattern family per week, ending in mixed mocks. Each day: 10 minutes of drills (next section), then the listed problems. Tick a day off only when you can state each problem's `dp` meaning and recurrence from memory.

Rules for each problem: write the 5-step recipe as comments before coding; 25 minutes on Medium before a hint, 40 on Hard; solve top-down, then convert to bottom-up; log the state definition in the mistake log, even if you solved it.

### Week 1 — Linear DP and the recipe

Goal: define `dp[i]` in one sentence without hesitation.

- [ ] Day 1 — [70 Climbing Stairs](https://leetcode.com/problems/climbing-stairs/) · [746 Min Cost Climbing Stairs](https://leetcode.com/problems/min-cost-climbing-stairs/) · [509 Fibonacci Number](https://leetcode.com/problems/fibonacci-number/) (both top-down and bottom-up)
- [ ] Day 2 — [198 House Robber](https://leetcode.com/problems/house-robber/) · [213 House Robber II](https://leetcode.com/problems/house-robber-ii/)
- [ ] Day 3 — [91 Decode Ways](https://leetcode.com/problems/decode-ways/) · [139 Word Break](https://leetcode.com/problems/word-break/)
- [ ] Day 4 — [53 Maximum Subarray](https://leetcode.com/problems/maximum-subarray/) (as DP) · [152 Maximum Product Subarray](https://leetcode.com/problems/maximum-product-subarray/)
- [ ] Day 5 — [740 Delete and Earn](https://leetcode.com/problems/delete-and-earn/) · [983 Minimum Cost For Tickets](https://leetcode.com/problems/minimum-cost-for-tickets/)
- [ ] Day 6 — Timed test, 50 minutes: [2140 Solving Questions With Brainpower](https://leetcode.com/problems/solving-questions-with-brainpower/) · [1043 Partition Array for Maximum Sum](https://leetcode.com/problems/partition-array-for-maximum-sum/)
- [ ] Day 7 — Review: re-solve logged misses; explain House Robber with the 5 steps out loud

### Week 2 — Knapsack (0/1 and unbounded)

Goal: know instantly which way to loop capacity, and why.

- [ ] Day 1 — [416 Partition Equal Subset Sum](https://leetcode.com/problems/partition-equal-subset-sum/) · [494 Target Sum](https://leetcode.com/problems/target-sum/)
- [ ] Day 2 — [322 Coin Change](https://leetcode.com/problems/coin-change/) · [518 Coin Change II](https://leetcode.com/problems/coin-change-ii/)
- [ ] Day 3 — [377 Combination Sum IV](https://leetcode.com/problems/combination-sum-iv/) (permutations — compare with 518) · [279 Perfect Squares](https://leetcode.com/problems/perfect-squares/)
- [ ] Day 4 — [1049 Last Stone Weight II](https://leetcode.com/problems/last-stone-weight-ii/) · [474 Ones and Zeroes](https://leetcode.com/problems/ones-and-zeroes/) (2D capacity)
- [ ] Day 5 — [1155 Number of Dice Rolls With Target Sum](https://leetcode.com/problems/number-of-dice-rolls-with-target-sum/) · [879 Profitable Schemes](https://leetcode.com/problems/profitable-schemes/) (Hard)
- [ ] Day 6 — Timed test, 50 minutes: [343 Integer Break](https://leetcode.com/problems/integer-break/) · [2787 Ways to Express an Integer as Sum of Powers](https://leetcode.com/problems/ways-to-express-an-integer-as-sum-of-powers/)
- [ ] Day 7 — Review: write 0/1 and unbounded templates from memory side by side

### Week 3 — Grid DP and LIS

Goal: handle 2D state and reduce it to one row of space.

- [ ] Day 1 — [62 Unique Paths](https://leetcode.com/problems/unique-paths/) · [63 Unique Paths II](https://leetcode.com/problems/unique-paths-ii/) · [64 Minimum Path Sum](https://leetcode.com/problems/minimum-path-sum/)
- [ ] Day 2 — [120 Triangle](https://leetcode.com/problems/triangle/) · [931 Minimum Falling Path Sum](https://leetcode.com/problems/minimum-falling-path-sum/) · [221 Maximal Square](https://leetcode.com/problems/maximal-square/)
- [ ] Day 3 — [300 Longest Increasing Subsequence](https://leetcode.com/problems/longest-increasing-subsequence/) (O(n^2) then O(n log n)) · [646 Maximum Length of Pair Chain](https://leetcode.com/problems/maximum-length-of-pair-chain/)
- [ ] Day 4 — [354 Russian Doll Envelopes](https://leetcode.com/problems/russian-doll-envelopes/) (Hard) · [673 Number of LIS](https://leetcode.com/problems/number-of-longest-increasing-subsequence/)
- [ ] Day 5 — [368 Largest Divisible Subset](https://leetcode.com/problems/largest-divisible-subset/) · [1048 Longest String Chain](https://leetcode.com/problems/longest-string-chain/)
- [ ] Day 6 — Timed test, 60 minutes: [174 Dungeon Game](https://leetcode.com/problems/dungeon-game/) · [1463 Cherry Pickup II](https://leetcode.com/problems/cherry-pickup-ii/)
- [ ] Day 7 — Review: re-solve logged misses; reconstruct an actual LIS, not just its length

### Week 4 — Two sequences and palindromes

Goal: fill a `(n + 1) × (m + 1)` table correctly the first time.

- [ ] Day 1 — [1143 Longest Common Subsequence](https://leetcode.com/problems/longest-common-subsequence/) · [583 Delete Operation for Two Strings](https://leetcode.com/problems/delete-operation-for-two-strings/)
- [ ] Day 2 — [72 Edit Distance](https://leetcode.com/problems/edit-distance/) · [712 Minimum ASCII Delete Sum](https://leetcode.com/problems/minimum-ascii-delete-sum-for-two-strings/)
- [ ] Day 3 — [5 Longest Palindromic Substring](https://leetcode.com/problems/longest-palindromic-substring/) · [647 Palindromic Substrings](https://leetcode.com/problems/palindromic-substrings/) · [516 Longest Palindromic Subsequence](https://leetcode.com/problems/longest-palindromic-subsequence/)
- [ ] Day 4 — [97 Interleaving String](https://leetcode.com/problems/interleaving-string/) · [115 Distinct Subsequences](https://leetcode.com/problems/distinct-subsequences/)
- [ ] Day 5 — [44 Wildcard Matching](https://leetcode.com/problems/wildcard-matching/) · [10 Regular Expression Matching](https://leetcode.com/problems/regular-expression-matching/) (both Hard)
- [ ] Day 6 — Timed test, 60 minutes: [1035 Uncrossed Lines](https://leetcode.com/problems/uncrossed-lines/) · [1092 Shortest Common Supersequence](https://leetcode.com/problems/shortest-common-supersequence/) (Hard)
- [ ] Day 7 — Review: draw the edit-distance table for "horse" → "ros" by hand

### Week 5 — State machines and interval DP

Goal: model "modes" as variables, and fill interval tables by length.

- [ ] Day 1 — [309 Stock with Cooldown](https://leetcode.com/problems/best-time-to-buy-and-sell-stock-with-cooldown/) · [714 Stock with Transaction Fee](https://leetcode.com/problems/best-time-to-buy-and-sell-stock-with-transaction-fee/)
- [ ] Day 2 — [123 Stock III](https://leetcode.com/problems/best-time-to-buy-and-sell-stock-iii/) · [188 Stock IV](https://leetcode.com/problems/best-time-to-buy-and-sell-stock-iv/) (both Hard)
- [ ] Day 3 — [312 Burst Balloons](https://leetcode.com/problems/burst-balloons/) · [1547 Minimum Cost to Cut a Stick](https://leetcode.com/problems/minimum-cost-to-cut-a-stick/) (both Hard)
- [ ] Day 4 — [132 Palindrome Partitioning II](https://leetcode.com/problems/palindrome-partitioning-ii/) · [1039 Minimum Score Triangulation of Polygon](https://leetcode.com/problems/minimum-score-triangulation-of-polygon/)
- [ ] Day 5 — [375 Guess Number Higher or Lower II](https://leetcode.com/problems/guess-number-higher-or-lower-ii/) · [664 Strange Printer](https://leetcode.com/problems/strange-printer/) (Hard)
- [ ] Day 6 — Timed test, 50 minutes: [877 Stone Game](https://leetcode.com/problems/stone-game/) · [1140 Stone Game II](https://leetcode.com/problems/stone-game-ii/)
- [ ] Day 7 — Review: re-solve logged misses; explain why interval DP loops by length

### Week 6 — Tree DP, bitmask DP, mixed mocks

Goal: solve unseen DP problems under interview time pressure.

- [ ] Day 1 — [337 House Robber III](https://leetcode.com/problems/house-robber-iii/) · [124 Binary Tree Maximum Path Sum](https://leetcode.com/problems/binary-tree-maximum-path-sum/)
- [ ] Day 2 — [96 Unique Binary Search Trees](https://leetcode.com/problems/unique-binary-search-trees/) · [968 Binary Tree Cameras](https://leetcode.com/problems/binary-tree-cameras/) (Hard)
- [ ] Day 3 — Bitmask DP: [526 Beautiful Arrangement](https://leetcode.com/problems/beautiful-arrangement/) · [1434 Number of Ways to Wear Different Hats](https://leetcode.com/problems/number-of-ways-to-wear-different-hats-to-each-other/) (Hard)
- [ ] Day 4 — Mock, 60 minutes, talk out loud: [1235 Maximum Profit in Job Scheduling](https://leetcode.com/problems/maximum-profit-in-job-scheduling/) · [403 Frog Jump](https://leetcode.com/problems/frog-jump/)
- [ ] Day 5 — Mock, 60 minutes: [1312 Minimum Insertions to Make a Palindrome](https://leetcode.com/problems/minimum-insertion-steps-to-make-a-string-palindrome/) · [1458 Max Dot Product of Two Subsequences](https://leetcode.com/problems/max-dot-product-of-two-subsequences/) · [2218 Maximum Value of K Coins From Piles](https://leetcode.com/problems/maximum-value-of-k-coins-from-piles/) (stretch)
- [ ] Day 6 — Re-solve the five hardest entries in your mistake log, cold
- [ ] Day 7 — Rest, or teach one problem from each pattern to a friend (or a rubber duck)

After week 6, keep it alive with 3 random DP problems a week from LeetCode's Dynamic Programming tag, at least one Hard.

## Drills, review routine and mistake log

Ten minutes of state-definition drills a day trains the one skill DP really tests. Do them before the day's problems, on paper, without coding.

### Daily 10-minute drills

1. **State only (4 min):** take 2 problems you have not solved (any DP-tagged problem title) and write only the state sentence and the recurrence. Do not solve them. Check against the editorial later in the week.
2. **Table by hand (3 min):** fill a tiny table: LCS of "abcde" and "ace", coin change for amount 6 with coins {1, 3, 4}, or House Robber on a 5-element array.
3. **Template from memory (3 min):** rotate one per day — 0/1 knapsack, unbounded knapsack, LIS (n log n), edit distance, interval DP by length, stock state machine, tree DP returning a pair.

Answer key for 2: LCS("abcde", "ace") = 3. Coin change for 6 with {1, 3, 4} = 2 coins (3 + 3); greedy picks 4 + 1 + 1 = 3 coins, which is why this needs DP.

### Review routine

- Re-solve every logged problem 3 days later, then 7 days later, from a blank editor.
- A problem leaves the log only after two clean re-solves.
- Each Sunday, read the "State" column top to bottom. If you can't recall the recurrence from the state alone, re-solve that problem.

### Mistake log

One row per problem missed or solved slowly. The "State" column is the single most useful thing to review.

| Date | Problem | Pattern | State (what dp means) | What went wrong | Re-solve day 3 | Re-solve day 7 |
| --- | --- | --- | --- | --- | --- | --- |
| 2026-10-20 | Example: 518 Coin Change II | 3. Unbounded knapsack | ways\[a\] = combinations summing to a | Looped amount outer, counted permutations |  |  |
|  |  |  |  |  |  |  |
|  |  |  |  |  |  |  |
