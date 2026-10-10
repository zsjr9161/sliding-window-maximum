# sliding window maximum: the O(n) monotonic deque explained step by step, with Python and Java code that survives follow-ups

LeetCode 239 is one of those problems where the naive solution is two lines and the accepted solution is the whole point of the interview. Anyone can write `max(nums[i:i+k])`. The question is whether you can explain why a deque makes it linear, and then write that deque without an off-by-one.

Here's the full picture: the three approaches, the single argument that makes the deque feel inevitable instead of memorized, a complete state trace, working code, and the mistakes that actually cost people the offer.

## What the problem actually asks

You get an integer array and a window size `k`. The window slides from left to right, one position at a time. For every position, you report the maximum inside the window.


nums = [1, 3, -1, -3, 5, 3, 6, 7], k = 3


| Window position | Values in window | Max |
| --- | --- | --- |
| 0 | [1, 3, -1] | 3 |
| 1 | [3, -1, -3] | 3 |
| 2 | [-1, -3, 5] | 5 |
| 3 | [-3, 5, 3] | 5 |
| 4 | [5, 3, 6] | 6 |
| 5 | [3, 6, 7] | 7 |

Output: `[3, 3, 5, 5, 6, 7]`.

Two details matter more than they look. First, the number of windows is exactly `n - k + 1`, so your output array is that length — writing `n` windows is a common silent failure. Second, the problem statement explicitly throws down a follow-up: **can you do it in linear time?** That follow-up is the reason the problem is rated Hard rather than Medium. A heap solution is perfectly correct and usually not what the interviewer is fishing for.

## Three solutions, and what each one buys you

| Approach | Time | Space | What it demonstrates |
| --- | --- | --- | --- |
| Brute force: rescan each window | O(n·k) | O(1) besides output | You can read the problem |
| Max-heap of `(value, index)` | O(n log k) with a bounded heap, O(n log n) if you let it grow | O(k) or O(n) | You know how to handle stale entries |
| Monotonic deque of indices | O(n) | O(k) | The actual answer |

The brute force is not useless in an interview. Say it out loud, state the complexity, and note the waste: consecutive windows share `k - 1` elements, and you're re-scanning all of them. That observation is the doorway to the optimal solution, and interviewers notice when you walk through it instead of teleporting to the "trick".

The heap version is worth knowing because it's the natural first optimization, and it introduces the idea that makes the deque work: **you have to be able to throw away elements that have left the window.** Push `(value, index)`, and before reading the top, pop while the top's index is outside the current window `[i - k + 1, i]`. Values can't be stale-corrected on their own — you need the index to know when an entry expired.

Then come the trade-offs. A heap gives you the max in O(1) and costs O(log n) to insert. Every departing element is lazily discarded rather than actively removed, so the heap grows and shrinks unpredictably. You're paying log factors for a structure that does much more than you need: you never ask for the second-largest value, or for arbitrary retrieval.

## The one argument that makes the deque obvious

This is the part worth memorizing as *reasoning*, not as code.

Take two indices `i < j` inside the same window, with `nums[i] <= nums[j]`. As the window slides right, `j` stays in the window at least as long as `i` does — it's to the right, so it leaves later. And `nums[j]` is at least as large. So there is no future window where `i` can ever be the maximum while `j` is present.

`i` is dominated. Delete it and never think about it again.

That's the whole insight. Applying it repeatedly means the only indices worth keeping are ones whose values are **strictly decreasing** from the front of your structure to the back. The front index is always the current window maximum. When a new element arrives, it evicts every smaller-or-equal value at the back, because those are now dominated. When the window moves past the front index, that index expires.

A useful mental model: a line of people ordered tallest to shortest. A new arrival walks past everyone shorter than them, and those people leave the line permanently — they can never be the tallest person in the room again while the newcomer is standing there. The person at the very front of the line is the tallest, i.e. the window maximum.

Note the word *strictly*. If `nums[i] == nums[j]` with `i < j`, `i` is still dominated (it leaves the window first and offers nothing new), so the back-eviction condition uses `<=`, not `<`.

## Two gates per element

Iterate `i` from 0 to `n - 1`. At each step, exactly two things happen to the deque, and then you read the answer.

1. **Expiry gate.** If the front index is out of the window, drop it. The window covers `[i - k + 1, i]`, so an index is dead when `deque[0] <= i - k`. One element can expire per step at most, so a single `if` is enough.
2. **Obsolescence gate.** While the back index holds a value `<= nums[i]`, pop it. Those indices are dominated by `i` and can never be an answer again.
3. **Insert.** Push `i` onto the back. The deque is now decreasing front to back.
4. **Record.** Once `i >= k - 1`, the window is full and `nums[deque[0]]` is the answer for this position.

Order matters. Doing the expiry check *before* the result is recorded is what keeps stale maxima out of your output. Doing the obsolescence pop *before* the insert is what keeps the deque monotonic.

## Full trace on the example

`nums = [1, 3, -1, -3, 5, 3, 6, 7]`, `k = 3`. The deque holds indices; values shown in parentheses.

| i | nums[i] | Expiry (`<= i-k`) | Back pops | Deque (indices, values) | Result |
| --- | --- | --- | --- | --- | --- |
| 0 | 1 | — | — | [0 (1)] | — |
| 1 | 3 | — | pop 0 (1 ≤ 3) | [1 (3)] | — |
| 2 | -1 | — | — | [1 (3), 2 (-1)] | 3 |
| 3 | -3 | — | — | [1 (3), 2 (-1), 3 (-3)] | 3 |
| 4 | 5 | pop 1 (1 ≤ 1) | pop 3, pop 2 (both ≤ 5) | [4 (5)] | 5 |
| 5 | 3 | — | — | [4 (5), 5 (3)] | 5 |
| 6 | 6 | — | pop 5, pop 4 | [6 (6)] | 6 |
| 7 | 7 | — | pop 6 | [7 (7)] | 7 |

One step deserves attention: at `i = 4`, index 1 expires at the same moment the value 5 evicts everything else. The deque drops from three entries to one. That's normal — the deque length reflects *domination*, not window size, which is why its size is bounded by `k` but usually much smaller.

Total result: `[3, 3, 5, 5, 6, 7]`.

## Python and Java

Python:

python
from collections import deque
from typing import List

def max_sliding_window(nums: List[int], k: int) -> List[int]:
    dq = deque()   # indices; nums at these indices are strictly decreasing
    res = []
    for i, x in enumerate(nums):
        # expiry gate: front index has fallen out of [i-k+1, i]
        if dq and dq[0] <= i - k:
            dq.popleft()
        # obsolescence gate: x dominates every value <= x at the back
        while dq and nums[dq[-1]] <= x:
            dq.pop()
        dq.append(i)
        # window is full, deque front is the maximum
        if i >= k - 1:
            res.append(nums[dq[0]])
    return res


Java:

java
public int[] maxSlidingWindow(int[] nums, int k) {
    int n = nums.length;
    int[] res = new int[n - k + 1];
    Deque<Integer> dq = new ArrayDeque<>(); // indices, values decreasing
    int idx = 0;
    for (int i = 0; i < n; i++) {
        if (!dq.isEmpty() && dq.peekFirst() <= i - k) {
            dq.pollFirst();
        }
        while (!dq.isEmpty() && nums[dq.peekLast()] <= nums[i]) {
            dq.pollLast();
        }
        dq.addLast(i);
        if (i >= k - 1) {
            res[idx++] = nums[dq.peekFirst()];
        }
    }
    return res;
}


Both are the same algorithm. The only structural difference is that Java gives you `ArrayDeque` with explicit `peekFirst`/`peekLast`, while Python's `deque` indexes from both ends.

## The mistakes that actually show up

- **Storing values instead of indices.** You lose the ability to tell when an entry left the window. There is a values-only variant that works by popping the front when it equals `nums[i - k]`, but it's fragile with duplicates: you can't distinguish two equal values that entered the window at different times.
- **Wrong comparison direction.** The invariant is "strictly decreasing front to back", which means the back-popping condition is `nums[back] <= nums[i]`. Flip it and the deque either grows unbounded or returns garbage.
- **Checking expiry after recording the result.** This is the classic silent bug — most test cases still pass, and you leak a stale maximum exactly when the outgoing element was the previous window's max.
- **Writing `n` results instead of `n - k + 1`.** The loop bounds or the result index shifts by one. Guard against it by asserting on the length in your head: six windows for eight elements with `k = 3`.
- **Recording before the window is full.** You need `i >= k - 1` (equivalently `i + 1 >= k`) before the first answer exists.
- **Using a plain list as the container.** `pop(0)` on a Python list is O(n), which quietly destroys the linear runtime you were trying to demonstrate.
- **Ignoring edge cases the interviewer is allowed to change.** The problem guarantees `1 <= k <= n` for a non-empty array, but interview variants ask about `k = 0`, `k > n`, or an empty array. Ask before you assume.
- **Writing the code before stating the domination argument.** In a real screen, the explanation *is* the signal. Handling over the finished function is the fastest way to make an optimal solution look like pattern recall.

## Where the same deque pattern reappears

The reason this problem is worth genuinely understanding rather than memorizing is that the structure generalizes:

- **Sliding window minimum.** Same code, flip the comparison to `>=`. Interviewers use this to see whether you understand the invariant or just copied a template.
- **Longest continuous subarray with absolute difference ≤ limit** (LeetCode 1438). You need both a max deque and a min deque, and a variable-width window.
- **Shortest subarray with sum at least K** (LeetCode 862). Prefix sums plus an *increasing* deque, because you're hunting for the minimum prefix rather than the maximum value.
- **Streaming maximum over the last k values.** The deque is naturally online — elements arrive and are processed once. Nothing about it requires knowing the future.
- **The block prefix/suffix alternative.** Split the array into fixed blocks of size `k`, precompute prefix maxima within each block and suffix maxima within each block, then the window maximum for a window starting at `i` is `max(suffix_max[i], prefix_max[i + k - 1])`. Also O(n), and very cache-friendly. The catch: it needs the entire array up front, so it fails on a stream, and it costs O(n) extra space instead of O(k).

That contrast is a good thing to have ready: two different O(n) solutions with different constraints on the input.

## Why it's O(n), and the follow-ups you should expect

Each index is pushed exactly once and popped at most once — either by the obsolescence gate or the expiry gate, never both. So the total number of deque operations across the whole run is bounded by `2n`, and the inner `while` loop is amortized O(1) per element even though a single iteration can pop several entries. That's the amortized argument, and saying the word "amortized" while explaining it is worth real points.

Space is O(k): the deque holds only indices from the current window, and dominated ones are removed, so at most `k` entries.

Follow-ups that come up regularly: what happens if the array is a stream (the deque still works, the block method doesn't); what if `k` equals `n` (one window, one answer); what if you need the max *and* the position of the max (store indices, which you already do); what if values repeat (the `<=` in the back-popping condition is why the invariant stays strict).

If you want a longer walkthrough with the same example traced in multiple languages, AlgoMonster keeps a public page for this exact problem: 👉 [read the AlgoMonster walkthrough of LeetCode 239](https://algo.monster/liteproblems/239?fpr=li69).

## Practicing the pattern instead of re-deriving it under pressure

The awkward truth about sliding window maximum is that it's rarely memorized and rarely re-derived cleanly from scratch 40 minutes into an interview loop. What separates people who pass from people who don't is whether "this is a fixed-window extremum problem" fires early — before they've burned 15 minutes on a heap.

That's the gap AlgoMonster is built around. It's a pattern-based prep platform from ex-Google engineers and competitive programmers: instead of a bank of thousands of problems, you get a structured set of coding patterns (the sliding window family among them, alongside two pointers, BFS/DFS, dynamic programming, and so on), each with a conceptual explanation, reusable template code, and diagrams. The format is text and interactive diagrams — there's no video — plus decision flowcharts for the "which pattern does this problem want?" question, a Speedrun mode for timed pre-interview review, company-tagged question sets updated monthly, and an AI assistant you can ask for hints while working a problem.

Worth being straight about the limits, because they show up in third-party reviews: it's DSA-focused, so senior-level system design and behavioral rounds need separate material; there are no live mock interviews with real engineers; and the problem count is a curated few hundred rather than LeetCode's thousands. If you learn better from video walkthroughs, this isn't that.

The pricing is simpler than most prep tools: there's no monthly plan, so you're choosing between annual and lifetime.

| Plan | What you get | Price | Billing | Get it |
| --- | --- | --- | --- | --- |
| Free | Limited access to a subset of patterns and problems — enough to judge the teaching style | $0 | No card | [Start on the free tier](https://algo.monster?fpr=li69) |
| Pro — Yearly | All patterns, lessons and problems; unlimited AI assistant; unlimited Speedrun; company-tagged questions updated monthly; multiple-language templates; 12 months of full access; member discount on live bootcamps | $99/year (listed at $198) | Billed annually, currently with a short trial before the first charge | [Get Pro Yearly](https://algo.monster/subscribe?fpr=li69) |
| Pro — Lifetime | Everything in the annual plan, plus lifetime content updates and permanent access with no renewals | $189 one-time (listed at $378) | Billed once | [Get Pro Lifetime](https://algo.monster/subscribe?fpr=li69) |

Two caveats worth knowing before you buy. First, AlgoMonster's prices move a lot — independent write-ups in circulation quote annual prices anywhere from $89 to $299 and lifetime prices from $99 to $459, which tells you more about how frequently the promotional pricing changes than about the product. Treat the number on the checkout page as the real one. Second, the annual plan is the better fit if you have an interview in the next two months; lifetime only pays off if you expect to revisit this material across more than one job search.

If your actual problem is that you can write the deque but keep losing the thread on *why* it works under a follow-up question, the pattern-level explanation is the thing to buy, not more problems. 👉 [See AlgoMonster's current plans and pricing](https://algo.monster/subscribe?fpr=li69)

## Short answers to the questions people actually search

**Is O(n log k) good enough?** It passes technically. If the interviewer asks the explicit linear-time follow-up from the problem statement, it isn't the answer they want.

**Why a deque and not a queue?** You need to remove from both ends: expired indices from the front, dominated indices from the back. A single-ended queue gives you one of the two.

**Why store indices rather than values?** Because you need to know *when* an element expires, and a value alone doesn't carry that information. Indices also let you recover the value in O(1) with a lookup.

**What's the size of the deque at most?** `k`. Never more, because entries outside the window are removed before they can accumulate.

**Does the array need to be sorted or non-negative?** No. Negatives work fine — the algorithm compares values, and the expiry logic is purely positional.

**Can I solve it with sliding window in the usual sense?** Not really. A standard sliding window maintains a running aggregate that can be updated incrementally on add and remove; a maximum can't be decremented away, which is exactly why you need the monotonic structure on top of the window.
