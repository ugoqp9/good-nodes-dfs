# Count Good Nodes in Binary Tree: One-Pass DFS With a Running Maximum, Explained With Dry Runs and Python, Java, and C++ Code

LeetCode 1448 is one of those problems that looks like a tree traversal and turns out to be a lesson in what you actually need to carry down a recursion. Most people get the shape of the solution right on the first try and still fail the submission, usually because of one comparison operator.

Here's what the problem actually asks, why the naive answer is slower than it needs to be, and the compact version that passes — plus the four mistakes that account for most wrong answers.

## What "good" actually means

The statement is short enough to quote: a node X is good if, on the path from the root to X, there are no nodes with a value **greater than** X.

Two details do a lot of quiet work there.

The first is *on the path from the root to X*. Not the subtree, not the neighbors, not X's own children. Only the ancestors above it matter. That means a node with a tiny value can still be good if every ancestor is even tinier, and a node with a large value can fail if one ancestor beat it.

The second is *greater than*, which is strict. Equal values don't disqualify anything. So a child with the same value as its parent is still good. That single word is why Example 2 returns 3 rather than 2.

And the root, having no ancestors at all, is always good. Every correct solution has to account for that, whether it does so explicitly or by seeding the comparison with something small enough.

## Dry run: `root = [3,1,4,3,null,1,5]`

The array builds this tree:


        3
       / \
      1   4
     /   / \
    3   1   5


Walking it with a running maximum:

| Node | Path to it | Max before it | Good? | Running count |
| --- | --- | --- | --- | --- |
| 3 | [3] | — | yes, root | 1 |
| 1 | [3, 1] | 3 | no, 3 > 1 | 1 |
| 3 | [3, 1, 3] | 3 | yes, 3 ≥ 3 | 2 |
| 4 | [3, 4] | 3 | yes | 3 |
| 1 | [3, 4, 1] | 4 | no | 3 |
| 5 | [3, 4, 5] | 4 | yes | 4 |

Answer: **4**.

Notice that the two nodes marked "no" fail for the same reason — a strictly larger ancestor. And notice the node with value 3 sitting under a node with value 1. Its path max is 3, inherited from the root, not 1 from its parent. That's the whole problem in one cell of the table: the value you compare against is the maximum over the entire path, not the parent's value.

## Why the brute force approach is a trap

The obvious reading of the problem is: for each node, walk back up to the root, collect the path, check whether anything on it is bigger.

That's correct, and it runs in O(n) time per node. On a balanced tree that averages out to something tolerable. On a skewed tree — 10^5 nodes all leaning to one side — you're doing O(n²) work. The constraints say n can be up to 10^5, so that's the difference between passing and a time limit exceeded verdict.

The fix is to stop thinking of the path as data you need. You don't need the path. You need one number: the largest value you've seen on the way down.

So flip the direction. Instead of asking "what's above me?" at each node, carry the answer down as a parameter. When you arrive at a node, you already know the max of everything above it. Compare, update the max, hand it to both children.

That's one traversal, O(n) total. No path reconstruction, no backtracking bookkeeping.

## Recursive DFS in Python

python
class TreeNode:
    def __init__(self, val=0, left=None, right=None):
        self.val = val
        self.left = left
        self.right = right

class Solution:
    def goodNodes(self, root: TreeNode) -> int:
        def dfs(node, max_so_far):
            if node is None:
                return 0
            count = 1 if node.val >= max_so_far else 0
            new_max = max(max_so_far, node.val)
            return count + dfs(node.left, new_max) + dfs(node.right, new_max)

        return dfs(root, root.val)


The seed value matters. Passing `root.val` works because the root is good by definition and no ancestor can beat it. Passing `float("-inf")` also works and reads slightly more honestly. Passing `0` is a bug: node values can be as low as -10^4, so a tree of all-negative values would score a root that should count.

If you'd rather not pass the count around in three pieces, a nonlocal counter is equally fine:

python
class Solution:
    def goodNodes(self, root: TreeNode) -> int:
        count = 0

        def dfs(node, max_so_far):
            nonlocal count
            if node is None:
                return
            if node.val >= max_so_far:
                count += 1
            max_so_far = max(max_so_far, node.val)
            dfs(node.left, max_so_far)
            dfs(node.right, max_so_far)

        dfs(root, root.val)
        return count


Both are O(n). The first is easier to reason about because the counter can't leak between branches; the second is shorter. Whichever you write in an interview, say the complexity out loud before you start typing.

## The same thing in Java and C++

java
class Solution {
    public int goodNodes(TreeNode root) {
        return dfs(root, root.val);
    }

    private int dfs(TreeNode node, int maxSoFar) {
        if (node == null) {
            return 0;
        }
        int count = (node.val >= maxSoFar) ? 1 : 0;
        int newMax = Math.max(maxSoFar, node.val);
        return count + dfs(node.left, newMax) + dfs(node.right, newMax);
    }
}


cpp
class Solution {
public:
    int goodNodes(TreeNode* root) {
        return dfs(root, root->val);
    }

private:
    int dfs(TreeNode* node, int maxSoFar) {
        if (!node) return 0;
        int count = (node->val >= maxSoFar) ? 1 : 0;
        int newMax = std::max(maxSoFar, node->val);
        return count + dfs(node->left, newMax) + dfs(node->right, newMax);
    }
};


The structure is identical in every language: null returns 0, compare, take the new max, recurse twice, add. There's no clever language-specific trick to reach for here.

## Iterative DFS, or why you'd bother

The recursion is elegant and it will stack overflow on Python if the tree is a 10^5-node chain, because CPython's default recursion limit is 1000. Java and C++ will also blow the stack in pathological cases, just at a higher threshold.

So keep the iterative version in your back pocket. Push `(node, max_so_far)` pairs onto an explicit stack and the recursion limit stops being a constraint:

python
class Solution:
    def goodNodes(self, root: TreeNode) -> int:
        if root is None:
            return 0
        count = 0
        stack = [(root, root.val)]
        while stack:
            node, max_so_far = stack.pop()
            if node.val >= max_so_far:
                count += 1
            new_max = max(max_so_far, node.val)
            if node.left:
                stack.append((node.left, new_max))
            if node.right:
                stack.append((node.right, new_max))
        return count


Iterative DFS needs O(h) space for the stack, same as the recursive version needs for the call frames — except the stack lives on the heap, which doesn't have a depth limit imposed by the interpreter.

## BFS works too, but check the space cost

Because the running max travels *with* each node rather than depending on traversal order, you can process the tree level by level just as easily:

python
from collections import deque

class Solution:
    def goodNodes(self, root: TreeNode) -> int:
        if root is None:
            return 0
        count = 0
        q = deque([(root, root.val)])
        while q:
            node, max_so_far = q.popleft()
            if node.val >= max_so_far:
                count += 1
            new_max = max(max_so_far, node.val)
            if node.left:
                q.append((node.left, new_max))
            if node.right:
                q.append((node.right, new_max))
        return count


Same time, different memory profile. The queue holds an entire level, so it peaks around n/2 entries for a perfect tree. The DFS stack peaks at the tree's height, which is log n for a perfect tree. On the inputs LeetCode actually uses, DFS is the cheaper choice.

| Approach | Time | Extra space | Notes |
| --- | --- | --- | --- |
| Per-node path scan (brute force) | O(n²) worst case | O(n) | Correct but too slow for the stated limits |
| Recursive DFS with path max | O(n) | O(h) call frames | Cleanest to write; recursion depth is a real limit |
| Iterative DFS with explicit stack | O(n) | O(h) heap frames | Same logic, no interpreter depth cap |
| BFS with `(node, max)` pairs | O(n) | O(w), up to O(n) | Correct, but usually costs more memory than DFS |

## The four mistakes that cause most failures

**1. Using `>` instead of `>=`.**

The comparison is `node.val >= max_so_far`. People write `>` from muscle memory, then fail Example 2, where `root = [3,3,null,4,2]` returns 3 precisely because a child equal to its parent is still good. If you remember one thing about this problem, remember the equal sign.

**2. Storing the maximum in an instance variable.**

This is the bug that shows up on Stack Overflow most often, and it's worth understanding because it's a general recursion trap. Suppose you do this:

java
int maxL = 0, countL = 0, countR = 0, maxR = 0;


and update those fields while recursing. It seems to work on a simple left-leaning tree. It breaks the moment the traversal moves to the right subtree, because `maxL` still holds the maximum from the *left* branch — it never resets when you leave that branch. A node deep in the right subtree now gets compared against a value it shares no path with.

The fix is exactly what the code above does: make the running max a **parameter**, not a field. Parameters are scoped to the call, so walking back out of a branch automatically restores the previous value. Every value that should be "true for this path only" belongs in a parameter or a local, never in an instance field.

**3. Comparing the wrong two things.**

The same thread has a second, subtler bug: `if (valL > root.val)` where `valL` is the *return value* of the recursive call — a count — being compared against a node's value. Counts and node values have nothing to do with each other; the comparison is meaningless. If your DFS returns a count, then counts get summed. Comparisons happen between node values and maxima. Keeping the two roles separate is half of writing this function correctly.

**4. Seeding the max with zero.**

Covered above, but it's underrated because LeetCode's own examples are all positive. Node values range from -10^4 to 10^4. Seed with `root.val` or negative infinity and you never have to think about it again.

## Edge cases worth naming in an interview

The recursion handles all of these without special-casing, which is a good thing to point out:

- **Empty tree** — the constraints say at least one node, but a null check costs nothing and interviewers ask.
- **Single node** — returns 1. The root is always good.
- **Strictly decreasing chain** — every node is good, because each one is smaller than the ones above it... wait, no: each node is *smaller* than its ancestors, so each node is *not* good. Only the root is. Worth walking through, because the intuition flips depending on which way the chain leans. An increasing chain gives every node a good verdict; a decreasing chain gives exactly one.
- **All values equal** — `>=` means every node is good, count equals n.
- **Negative values** — the seed choice matters, as above.

That decreasing-chain case is the one to rehearse. It's the fastest way to show you understand that "good" is about ancestors, not about the node's own magnitude.

## How this problem fits a pattern-based prep routine

1448 is a textbook example of a pattern rather than a one-off. The reusable idea is: **when a problem asks about properties of the root-to-node path, pass the property down as a recursion parameter instead of rebuilding it at each node.** The same move shows up in path sum problems, maximum path value problems, and anything that asks "does every node on this path satisfy X". Once you've internalized the parameter-carrying trick, you solve the class, not the problem.

That's the pitch behind AlgoMonster, which organizes interview prep around patterns instead of a raw problem count. Their material on this problem and its neighbors is written as text with interactive diagrams — no video — with editorial explanations attached to each mapped problem, and the site's own subscription page states Pro access covers 500+ lessons across 6 courses, 1,000+ illustrations, and 3,000+ editorials, plus an AI assistant and a timed "Speedrun" mode. The content is authored by engineers who worked at Google and by competitive programmers.

The honest limits, which reviewers consistently flag: the problem bank is far smaller than LeetCode's (roughly 230 mapped problems, by their count, versus thousands), there are no video walkthroughs at all, and system design coverage is introductory — not enough for senior-level loops. It's built for people who can already write working code and need to get faster at recognizing which approach applies.

If that's your gap, 👉 [you can look at AlgoMonster's plans here](https://algo.monster?fpr=li69). There's a free tier with limited content, which is genuinely enough to judge whether the text-and-diagram format clicks for you before paying anything.

## Current plans and pricing

AlgoMonster runs promotions constantly, so treat the numbers below as a snapshot rather than a permanent menu. The subscription page currently lists three paid options:

| Plan | What you get | Price | Billing | Get it |
| --- | --- | --- | --- | --- |
| Free | Limited access to a subset of patterns and problems | $0 | — | [Start with the free tier](https://algo.monster?fpr=li69) |
| Pro — Monthly | All patterns, lessons and problems; unlimited AI assistant; unlimited Speedrun; company question sets; 12 months not included — one month of full access | $18.99 | Billed monthly, cancel or upgrade anytime | [Check the monthly plan](https://algo.monster?fpr=li69) |
| Pro — Yearly | Same full access, 12 months, member discount on live bootcamps | $99/year (shown as $8.25/mo billed annually) | Billed annually | [View the yearly plan](https://algo.monster?fpr=li69) |
| Pro — Lifetime | Everything in the annual plan plus lifetime updates to included Pro content | $189 | Billed once | [See the lifetime option](https://algo.monster?fpr=li69) |

A few things worth knowing before you pick:

The yearly price moves. During sales it has been listed as low as $69.99/year, and third-party deal trackers list the pre-discount figures for the annual and lifetime plans noticeably higher than what's shown on the page. If you're not in a hurry, timing matters more than it usually does with subscription products.

There's no team or enterprise tier. This is a single-candidate product.

The break-even math is simple: if you expect to interview more than once over the next few years — which is the normal case for engineers who switch jobs — the lifetime plan costs less than two years of the annual one. If this is a three-week sprint and you won't touch the platform again, the monthly or annual option is the lower-risk spend.

For a single problem like 1448 specifically, none of this is necessary. The write-up above is the whole solution. The subscription only makes sense if you want the surrounding twenty-odd tree and DFS problems in the same structure, with the pattern spelled out before the code.

## Try it yourself before you read the editorial

Close this tab and write the recursive version from memory. Seed the max with `root.val`, use `>=`, and keep the maximum in a parameter rather than a field. Then test it against `[3,3,null,4,2]` and confirm you get 3 — if you get 2, you know which of the two mistakes you made.

Then run it on a tree that's a 10^5-node chain in Python. If it throws a recursion error, you've just found the reason the iterative version exists, and you'll remember it better than you'd remember a paragraph telling you so.
