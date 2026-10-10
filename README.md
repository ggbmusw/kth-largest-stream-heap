# kth largest element in a stream: the size-k min-heap, working Python and Java code, and the edge cases that quietly fail

You're asked to design a class. It takes a `k` and some initial numbers, then `add(val)` gets called over and over — and each call has to hand back the kth largest value seen so far. Not the kth distinct value. Not the kth largest of the original array. The kth largest of everything that has arrived, in sorted order, duplicates included.

That last sentence is where most first attempts go wrong, and it's worth reading twice: `[1, 2, 3, 3]` with `k = 2` gives you `3`, not `2`.

## First, the thing the problem is actually testing

LeetCode 703 is tagged Easy. It's Easy because the trick is small and well known. It's not Easy because you submit it once and move on — this is the canonical "data stream" problem, and interviewers use it as a probe for whether you recognize heap patterns or just pattern-match on the words "kth largest."

Three readings of the problem statement cause most of the tears:

- **"kth largest in sorted order"** means positions, not values. Duplicates occupy multiple positions.
- **`nums` can be shorter than `k`.** The old note said `nums.length ≥ k - 1`; current constraints are `1 <= k <= nums.length + 1`. Either way, your constructor cannot assume it starts with k elements. `add()` has to tolerate a heap that isn't full yet.
- **`add` keeps being called.** Verification calls it up to 10,000 times. Whatever you do per call multiplies by 10,000.

That third point is what kills the obvious solution.

## Why "keep everything and sort" loses

The instinctive version: stash every value in a list, sort it, return the element at index `len(arr) - k`.

Every `add()` costs O(n log n), where n is the total number of elements that have ever arrived. Do that 10,000 times with a few thousand values and you're sorting a growing list thousands of times — and you're doing almost all of it for nothing.

Here's the useful question to ask instead: **which of those values can never be the answer?**

If you only ever need the kth largest, then the k−1 values sitting below it are dead weight. They will never be returned. They don't influence the answer. The only values that matter are the top k, and only the smallest of that group — the kth largest — is the thing you report.

So the target is: maintain exactly k elements, and keep instant access to the smallest of them.

That requirement, "keep a collection whose smallest element is always readable in O(1) and whose size you can cap," describes a min-heap exactly.

## Why a min-heap and not a max-heap

This is the part people fumble under pressure, so get the reasoning straight.

A max-heap over the top k gives you the *largest* element on top. That's the 1st largest, not the kth. And when a new value arrives and the heap overflows, a max-heap evicts the top — the biggest thing you have, which is precisely what you want to keep.

A min-heap over the top k puts the *smallest of the top k* on top. Which is the kth largest by definition. And when the heap overflows, it evicts that same element — the one that just got pushed out of the top k by the newcomer. Both the returned value and the eviction target are the same node, and it's `heap[0]` / `peek()` / `top()`.

Stepping through `k = 3, nums = [4, 5, 8, 2]`:

| Step | Heap after rebalancing | Returned |
| --- | --- | --- |
| init: push 4 | `[4]` | — |
| init: push 5 | `[4, 5]` | — |
| init: push 8 | `[4, 5, 8]` | — |
| init: push 2 → size 4, pop min | `[4, 5, 8]` | — |
| add(3) → size 4, pop 3 | `[4, 5, 8]` | 4 |
| add(5) → size 4, pop 4 | `[5, 5, 8]` | 5 |
| add(10) → size 4, pop 5 | `[5, 8, 10]` | 5 |
| add(9) → size 4, pop 5 | `[8, 9, 10]` | 8 |
| add(4) → size 4, pop 4 | `[8, 9, 10]` | 8 |

Notice the heap holds the three largest values seen so far at every step, and the root is always the current answer. `2`, `3`, and `4` get discarded because they were never going to be in the top 3 — and that's the entire performance argument. Each discarding decision is O(log k) instead of a full re-sort.

## Python

python
import heapq

class KthLargest:
    def __init__(self, k: int, nums: list[int]):
        self.k = k
        self.min_heap = nums[:]
        heapq.heapify(self.min_heap)          # O(n), in-place
        while len(self.min_heap) > k:
            heapq.heappop(self.min_heap)

    def add(self, val: int) -> int:
        if len(self.min_heap) < self.k:
            heapq.heappush(self.min_heap, val)
        elif val > self.min_heap[0]:
            heapq.heapreplace(self.min_heap, val)   # pop min, then push
        return self.min_heap[0]


Two Python details worth knowing.

`heapq` is a min-heap only. There's no max-heap variant; the usual workaround is negating values (`heappush(h, -x)`) and remembering to negate on the way out. Getting that wrong is a classic silent bug, because a negated max-heap still returns plausible-looking numbers.

`heapreplace` and `heappushpop` are not interchangeable. `heapreplace(heap, val)` pops the root and inserts `val` unconditionally — if `val` is smaller than the current minimum, the heap ends up with an even smaller root, which corrupts your invariant. `heappushpop(heap, val)` pushes first, then pops the smallest, so a too-small `val` is the thing that gets popped and the heap is unchanged. That second behavior is the one you want when the heap is already full. The guard `elif val > self.min_heap[0]` above sidesteps the issue entirely, but if you drop the guard, use `heappushpop`.

## Java

java
import java.util.PriorityQueue;

class KthLargest {
    private final PriorityQueue<Integer> minHeap;
    private final int k;

    public KthLargest(int k, int[] nums) {
        this.k = k;
        this.minHeap = new PriorityQueue<>(k);   // default = min-heap
        for (int num : nums) add(num);
    }

    public int add(int val) {
        minHeap.offer(val);
        if (minHeap.size() > k) minHeap.poll();
        return minHeap.peek();
    }
}


`PriorityQueue` is a min-heap out of the box — the opposite convention from C++, where `priority_queue<int>` is a max-heap and you need `greater<int>` to flip it. If you switch languages under interview pressure, that's the detail that bites.

`peek()` is O(1); `poll()` and `offer()` are O(log k).

## C++, for completeness

cpp
class KthLargest {
    priority_queue<int, vector<int>, greater<int>> minHeap;  // greater = min-heap
    int k;
public:
    KthLargest(int k, vector<int>& nums) : k(k) {
        for (int n : nums) {
            minHeap.push(n);
            if ((int)minHeap.size() > k) minHeap.pop();
        }
    }

    int add(int val) {
        minHeap.push(val);
        if ((int)minHeap.size() > k) minHeap.pop();
        return minHeap.top();
    }
};


Adding all initial elements through the same `add` path keeps the invariant identical during construction and streaming. It's slightly less efficient than heapifying, but it's fewer lines and fewer places to be wrong.

## Complexity, side by side

| Approach | Constructor | Each `add` | Space | Passes? |
| --- | --- | --- | --- | --- |
| Sort on every `add` | O(1) | O(n log n) | O(n) | TLE on large streams |
| Sorted list + binary insert | O(n log n) | O(n) shift | O(n) | Correct, but the insert shift dominates |
| Min-heap of size k | O(n log k) | O(log k) | O(k) | Yes — the expected answer |
| Two heaps (smaller/larger split) | O(n log n) | O(log k) | O(n) | Yes, but stores values you never need |

The two-heap version is a legitimate generalization — it's the same machinery as Find Median from a Data Stream, where you genuinely do need values on both sides. Here you don't, and storing them is just extra bookkeeping.

## Edge cases that actually fail submissions

**Constructor with fewer than k elements.** `KthLargest(3, [])` is legal under current constraints. Guard the push path with `len(heap) < k`, or route everything through `add` and let the size check handle it.

**Duplicates.** `[7, 7, 7, 7, 8, 3]` with `k = 4` should give `7`. A min-heap with a strict `>` comparison still produces `7` here, but if you wrote a "distinct values only" set-based solution, you'd return `8`. Distinct was never in the spec.

**Negative values.** `[-10, 1, 3, 1, 4]` with `k = 10` and values down to `-10^4`. The heap handles these fine; index arithmetic like `arr[len(arr) - k]` on a sorted list also works, but any approach that uses `0` as a sentinel or initializes the answer to `0` will silently return wrong output.

**`k = 1`.** The heap never exceeds size 1, so `add` becomes "return the running maximum." Verify your code does that rather than something clever.

**Returning before the heap hits size k.** This shouldn't happen given the guarantee, but if you're writing your own tests, remember that `heap[0]` on an empty heap throws.

## The pattern underneath, and its siblings

"Maintain a bounded heap of the k best candidates, evict the worst of the retained set" is one idea wearing several costumes. Once you see it, the following stop being separate problems:

- **Kth Largest Element in an Array (215)** — same target, but the whole input is available up front, so quickselect gets you O(n) average and a heap gets you O(n log k). Different tradeoff, same "top k" framing.
- **Top K Frequent Elements (347)** — count with a hash map, then keep a size-k min-heap keyed on frequency. Bucket sort beats it to O(n), and that's the follow-up interviewers usually ask.
- **Find Median from a Data Stream (295)** — two heaps, balanced so the split point is the median. The streaming constraint is what forces the heap structure.
- **Kth Smallest Element in a Sorted Matrix (378)** — a k-sized max-heap instead, because you're evicting the largest of the retained set.
- **Sliding Window Maximum (239)** — a monotonic deque does the O(1)-amortized trick that a heap can't, since heaps can't remove arbitrary elements.

The point of the family is recognition speed. Under a 35-minute interview clock, the winner is usually whoever identifies the structure fastest, not whoever writes the tightest loop.

That recognition skill is what structured prep platforms sell. AlgoMonster organizes its curriculum around named patterns rather than a raw problem bank, and this problem sits in its heap/priority-queue material — there's also a free editorial walkthrough of LeetCode 703 with the Python and Java implementations side by side, which is a reasonable way to check your code against theirs before you pay for anything. 👉 [Read the walkthrough and see the pattern index](https://algo.monster?fpr=li69)

## If you're deciding whether to pay for prep

The honest version: this problem does not need a subscription. The idea is three sentences long and you now have it.

What a paid course buys is the map — knowing that "kth largest in a stream" and "median from a data stream" and "top k frequent" are the same structural family, so you stop treating 231 problems as 231 separate things to memorize. That's a real advantage if you're preparing across a few weeks and starting from scattered knowledge. It's close to worthless if you already recognize the patterns and just need reps.

AlgoMonster's own subscribe page lists it plainly. Current plans as displayed:

| Plan | What's included | Price | Billing | Get it |
| --- | --- | --- | --- | --- |
| Free | Limited access to sample lessons and problems | $0 | — | [Start with the free tier](https://algo.monster?fpr=li69) |
| Pro — Yearly | All patterns, lessons and problems; unlimited AI Assistant; unlimited Speedrun; company questions updated monthly; 500+ lessons across 6 courses, 1,000+ illustrations, 3,000+ editorials; member discount on live bootcamps | $99/year ($8.25/mo billed annually) | Annual, cancel or upgrade anytime | [Compare the Yearly plan](https://algo.monster?fpr=li69) |
| Pro — Lifetime | Everything in Yearly, plus lifetime access and all future content updates | $189 one-time | Once | [Compare the Lifetime plan](https://algo.monster?fpr=li69) |

Three practical notes.

There's no monthly plan. Annual or lifetime, that's the whole menu — so "try it for one month" isn't an option, and the free tier is your only trial. Separately, AlgoMonster runs frequent sales; recent snapshots of the subscribe page have shown the yearly plan discounted as far as $69.99 and lifetime as low as $89. Prices move, so check the page rather than trusting a number from any review, including reviews that quote $459 lifetime or $119–299 yearly. Those figures don't match what the official page shows and appear to be either stale or conflated with list prices.

On format: AlgoMonster is text and diagrams, not video. If you learn by watching someone solve it on a whiteboard, this isn't the product for you, and no discount fixes that. Third-party reviews also note the problem bank is small by LeetCode standards and that system design coverage is introductory — the value is concentrated in the pattern curriculum.

## Quick answers

**Can I just use a sorted list?** Yes, and it will pass on LeetCode if the input is small. It's not what's being tested.

**Is `nums` always at least k long?** No. Current constraints allow `nums.length` of 0 with `k` up to `nums.length + 1`. Handle a partially filled heap.

**Do I need to dedupe?** No. The problem uses sorted-order positions, so duplicates count separately.

**What's the answer if a tie crosses the boundary?** With `k = 2` and values `[1, 2, 3, 3]`, the answer is `3` — the top two are both threes, and the smaller of them is `3`.

**What's the target complexity?** O(n log k) to build, O(log k) per `add`, O(k) space. If your solution stores every value that ever arrived, you've missed the point even if the tests pass.

Write the invariant down before you write the code: *the heap holds exactly the k largest values seen so far, or fewer if fewer have arrived.* Every line you write should preserve it.
