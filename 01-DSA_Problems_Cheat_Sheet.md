# Bloomberg Interview DSA — Problems Solved & Quick Cheat Sheet

## 1. Two Sum

### Problem
Given an integer array `nums` and a target, return the indices of two numbers whose sum equals the target.

Example:
```text
[2,7,11,15], target = 9
→ [0,1]
```

### Approach — HashMap
Use a dictionary:
```text
number → index
```

For every number:
1. Calculate `remain = target - nums[i]`
2. Check whether `remain` already exists in the dictionary.
3. If yes, return its index and current index.
4. Otherwise store `nums[i] → i`.

### C# Pattern
```csharp
var dict = new Dictionary<int, int>();

for (int i = 0; i < nums.Length; i++)
{
    int remain = target - nums[i];

    if (dict.ContainsKey(remain))
        return new int[] { dict[remain], i };

    dict[nums[i]] = i;
}
```

### Complexity
- Time: `O(n)` average
- Space: `O(n)`

### Pitfall
Do not search the dictionary's **values**. The number is stored as the **key**, so use `ContainsKey()` / `TryGetValue()`.

---

# 2. First Unique Character in a String

### Problem
Find the index of the first character that appears exactly once. Return `-1` if none exists.

Examples:
```text
"leetcode"      → 0
"loveleetcode"  → 2
```

### Approach — Frequency Map + Second Pass

**Pass 1:** Count every character.

```text
character → frequency
```

**Pass 2:** Scan the original string from left to right and return the first character whose frequency is `1`.

### Flow
```text
String
  ↓
Count every character
  ↓
Scan string again
  ↓
frequency == 1 ?
  ↓
YES → return index
NO  → continue
```

### Complexity
- Time: `O(n)`
- Space: `O(k)` where `k` is the number of distinct characters

### Pitfall
Do not return the first character you encounter only once during the first pass. You need the complete frequency information first.

---

# 3. Longest Substring Without Repeating Characters

### Problem
Return the length of the longest substring containing no repeated characters.

Examples:
```text
"abcabcbb" → 3
"bbbbb"    → 1
```

### Approach — Sliding Window

Maintain a window:

```text
[slow ........ fast]
```

Use a dictionary:

```text
character → latest index
```

Move `fast` forward.

If the current character is already inside the current window:
```csharp
slow = Math.Max(slow, dict[ch] + 1);
```

Then update its latest index.

### Flow
```text
Expand fast
    ↓
Duplicate?
 ┌──┴──┐
No    Yes
 ↓      ↓
continue  move slow
          to duplicateIndex + 1
             ↓
       update dictionary
             ↓
       calculate max length
```

### Important Example

For `"abba"`:

When the second `b` is found:

```text
old b index = 1
new b index = 2

slow = 2
```

But use `Math.Max()` because `slow` must **never move backwards**.

### Complexity
- Time: `O(n)`
- Space: `O(k)`

### Pitfall
This is WRONG:
```csharp
slow++;
```

The left pointer may need to jump several positions.

Correct:
```csharp
slow = Math.Max(slow, dict[ch] + 1);
```

---

# 4. Valid Palindrome

### Problem
Check whether a string is a palindrome after:
- converting uppercase to lowercase
- ignoring non-alphanumeric characters

Examples:
```text
"A man, a plan, a canal: Panama" → true
"race a car"                      → false
```

### Approach — Two Pointers

Use:

```text
left  → beginning
right → end
```

1. Skip non-alphanumeric characters.
2. Compare the two characters ignoring case.
3. If different → `false`
4. Move both pointers inward.
5. If everything matches → `true`

### Flow
```text
left ---------------- right
  ↓                       ↓
skip invalid          skip invalid
  ↓                       ↓
      compare characters
             ↓
       same? → move inward
       different? → false
```

### C# APIs
```csharp
char.IsLetterOrDigit(ch)
char.ToLower(ch)
```

### Complexity
- Time: `O(n)`
- Space: `O(n)` in the version using `ToCharArray()`
- Can be `O(1)` extra space if indexing the original string directly.

### Pitfall
Do not compare spaces, commas, or other non-alphanumeric characters.

---

# 5. Valid Parentheses

### Problem
Given a string containing:
```text
()  {}  []
```
determine whether the brackets are valid.

Examples:
```text
"()"      → true
"([{}])"  → true
"([)]"    → false
```

### Approach — Stack

Opening brackets are pushed onto a stack.

When a closing bracket appears:
1. Stack must not be empty.
2. Pop the most recent opening bracket.
3. Check whether it matches the closing bracket.

At the end:
```text
Stack empty → valid
Stack not empty → invalid
```

### Why Stack?
Because brackets follow **LIFO**:

```text
Last opened
     ↓
First closed
```

### Flow
```text
Character
   ↓
Opening?
 ┌─┴─┐
Yes No
 ↓   ↓
Push  Stack empty?
      ↓
    invalid
      ↓
    Pop
      ↓
   Matching?
   /       \
 Yes       No
 ↓          ↓
continue   false
```

### Complexity
- Time: `O(n)`
- Space: `O(n)`

### Pitfall
Always check:
```csharp
if (stack.Count == 0)
    return false;
```
before popping.

---

# 6. Reverse Linked List

### Problem
Reverse a singly linked list.

Example:
```text
1 → 2 → 3 → 4 → 5 → null

becomes

5 → 4 → 3 → 2 → 1 → null
```

### Approach — Three References

Use:
```text
prev
curr
temp
```

For every node:

```text
temp = curr.next
curr.next = prev
prev = curr
curr = temp
```

### Flow
```text
Before:
prev   curr
 ↓      ↓
null   1 → 2 → 3

Save next
    ↓
temp = 2

Reverse link
    ↓
1 → null

Move prev
    ↓
prev = 1

Move curr
    ↓
curr = 2
```

Repeat until `curr == null`.

Return `prev`.

### Complexity
- Time: `O(n)`
- Space: `O(1)`

### Pitfall
Save `curr.next` BEFORE changing `curr.next`.

Otherwise the remaining list can be lost.

---

# 7. Binary Search

### Problem
Given a sorted array, find the target index. Return `-1` if not found.

Examples:
```text
[1,3,5,7,9,11], target = 7 → 3
target = 4 → -1
```

### Approach — Two Pointers

Maintain:
```text
left
right
```

Calculate:
```csharp
mid = left + (right - left) / 2;
```

Compare `target` with `nums[mid]`.

```text
target == nums[mid]
    → return mid

target > nums[mid]
    → search right
    → left = mid + 1

target < nums[mid]
    → search left
    → right = mid - 1
```

### Flow
```text
left -------- mid -------- right
             ↓
         compare target
        /       |       \
      <         =         >
      ↓         ↓         ↓
   right=mid-1 return   left=mid+1
```

### Complexity
- Time: `O(log n)`
- Space: `O(1)`

### Pitfall
Do not write:
```csharp
right--;
```
when the target is greater than `mid`.

Correct:
```csharp
left = mid + 1;
```

---

# 8. Merge Intervals

### Problem
Merge overlapping intervals.

Example:
```text
[[1,3],[2,6],[8,10],[15,18]]

→

[[1,6],[8,10],[15,18]]
```

Also:
```text
[[1,4],[4,5]]
→
[[1,5]]
```

### Approach — Sort + Merge

First sort intervals by starting point.

Then maintain:
```text
currentStart
currentEnd
```

For every next interval:

```text
nextStart <= currentEnd
        ↓
    overlapping
        ↓
currentEnd = max(currentEnd, nextEnd)
```

Otherwise:
```text
save current interval
start a new interval
```

### Flow
```text
SORT
 ↓
Take current interval
 ↓
Compare next.start with current.end
      /              \
 overlap             no overlap
    ↓                    ↓
merge              save current
                       ↓
                  start next
```

### Complexity
- Time: `O(n log n)` due to sorting
- Space: `O(n)` for result

### Pitfall
Use:
```csharp
nextStart <= currentEnd
```

not only `<`.

So `[1,4]` and `[4,5]` are considered overlapping.

---

# 9. Maximum Depth of Binary Tree

### Problem
Return the maximum depth of a binary tree.

Example:
```text
        3
       / \
      9   20
         /  \
        15   7

depth = 3
```

### Approach — DFS + Recursion

The key formula:

```text
depth(node)
=
1 + max(
    depth(node.left),
    depth(node.right)
)
```

Base case:

```csharp
if (root == null)
    return 0;
```

### Flow
```text
             node
            /    \
         left    right
          ↓        ↓
       depth     depth
          \       /
           max()
             ↓
            +1
```

### C# Pattern
```csharp
if (root == null)
    return 0;

return 1 + Math.Max(
    MaxDepth(root.left),
    MaxDepth(root.right)
);
```

### Complexity
- Time: `O(n)`
- Space: `O(h)` recursion stack
- `h` = tree height

### Pitfall
No global counter is required.

Each recursive call returns the depth of **its own subtree**.

---

# 10. Binary Tree Level Order Traversal

### Problem
Return tree nodes level by level from left to right.

Example:
```text
        3
       / \
      9   20
         /  \
        15   7

→
[
  [3],
  [9,20],
  [15,7]
]
```

### Approach — BFS + Queue

Use:
```csharp
Queue<TreeNode>
```

The important trick:

```csharp
int levelSize = queue.Count;
```

This tells us exactly how many nodes belong to the current level.

### Flow
```text
Queue = [3]
   ↓
levelSize = 1
   ↓
process 3
   ↓
Queue = [9,20]

levelSize = 2
   ↓
process 9 and 20
   ↓
Queue = [15,7]

levelSize = 2
   ↓
process 15 and 7
```

### Core Pattern
```csharp
while (queue.Count > 0)
{
    int levelSize = queue.Count;

    for (int i = 0; i < levelSize; i++)
    {
        TreeNode node = queue.Dequeue();

        // process node

        if (node.left != null)
            queue.Enqueue(node.left);

        if (node.right != null)
            queue.Enqueue(node.right);
    }
}
```

### Complexity
- Time: `O(n)`
- Space: `O(n)`

### Pitfall
Do not simply process the queue until empty and expect to know the levels.

Capture:
```csharp
levelSize = queue.Count
```
before processing the current level.

---

# QUICK PATTERN CHEAT SHEET

| Problem | Pattern | Main Data Structure | Key Idea | Time | Space |
|---|---|---|---|---|---|
| Two Sum | HashMap | `Dictionary<int,int>` | Store number → index | O(n) | O(n) |
| First Unique Character | Frequency Map | `Dictionary<char,int>` | Count first, scan second | O(n) | O(k) |
| Longest Substring | Sliding Window | Dictionary | `slow` + `fast`, latest index | O(n) | O(k) |
| Valid Palindrome | Two Pointers | None | Compare from both ends | O(n) | O(1)* |
| Valid Parentheses | Stack | `Stack<char>` | LIFO matching | O(n) | O(n) |
| Reverse Linked List | Pointer Manipulation | `ListNode` | Reverse links with 3 pointers | O(n) | O(1) |
| Binary Search | Binary Search | None | Eliminate half each time | O(log n) | O(1) |
| Merge Intervals | Sort + Merge | `List<int[]>` | Merge if `next.start <= end` | O(n log n) | O(n) |
| Max Depth | DFS | Recursion | `1 + max(left,right)` | O(n) | O(h) |
| Level Order | BFS | `Queue<TreeNode>` | Process `queue.Count` per level | O(n) | O(n) |

\* `O(1)` if operating directly on the string; the earlier `ToCharArray()` implementation uses `O(n)` extra space.

---

# C# COLLECTIONS TO REMEMBER

## Dictionary

```csharp
var dict = new Dictionary<int, int>();

dict[key] = value;

dict.ContainsKey(key);

dict[key];

dict.TryGetValue(key, out int value);
```

Use when you need:
```text
key → value
```

Common interview uses:
- Frequency counting
- Fast lookup
- Number → index
- Character → latest index

---

## Stack

```csharp
var stack = new Stack<char>();

stack.Push(ch);

char x = stack.Pop();

stack.Peek();

stack.Count;
```

Think:
```text
LIFO
Last In → First Out
```

Common use:
- Parentheses
- DFS
- Undo operations
- Expression parsing

---

## Queue

```csharp
var queue = new Queue<TreeNode>();

queue.Enqueue(node);

TreeNode x = queue.Dequeue();

queue.Peek();

queue.Count;
```

Think:
```text
FIFO
First In → First Out
```

Common use:
- BFS
- Level order traversal
- Scheduling

---

# THE BIG INTERVIEW PATTERN MAP

When you see...

### "Find pair / fast lookup"
Think:
```text
HashMap / Dictionary
```

### "Longest/shortest substring"
Think:
```text
Sliding Window
```

### "Palindrome / compare from both ends"
Think:
```text
Two Pointers
```

### "Matching brackets / last opened first"
Think:
```text
Stack
```

### "Linked list manipulation"
Think:
```text
Pointers
prev / curr / next
```

### "Sorted array + search"
Think:
```text
Binary Search
```

### "Overlapping ranges / schedules"
Think:
```text
Sort + Intervals
```

### "Tree: calculate something down branches"
Think:
```text
DFS / Recursion
```

### "Tree: level by level"
Think:
```text
BFS / Queue
```

### "Grid / islands / connected components"
Think:
```text
DFS or BFS
```

### "Top K / kth largest"
Think:
```text
Heap / Priority Queue
```

### "Graph conversion / routes / relationships"
Think:
```text
Graph + DFS/BFS
```

### "Cache with fast lookup + ordering"
Think:
```text
HashMap + Linked List
```

---

# MOST IMPORTANT RECURSION TEMPLATE

For tree problems:

```csharp
int Solve(TreeNode node)
{
    if (node == null)
        return 0;

    int left = Solve(node.left);
    int right = Solve(node.right);

    return 1 + Math.Max(left, right);
}
```

The exact operation changes depending on the problem.

---

# MOST IMPORTANT SLIDING WINDOW TEMPLATE

```csharp
int slow = 0;

for (int fast = 0; fast < n; fast++)
{
    // add fast element

    while (window is invalid)
    {
        // remove slow element
        slow++;
    }

    // update answer
}
```

---

# MOST IMPORTANT TWO-POINTER TEMPLATE

```csharp
int left = 0;
int right = n - 1;

while (left < right)
{
    // compare/use left and right

    left++;
    right--;
}
```

---

# MOST IMPORTANT BFS TEMPLATE

```csharp
var queue = new Queue<TreeNode>();
queue.Enqueue(root);

while (queue.Count > 0)
{
    int levelSize = queue.Count;

    for (int i = 0; i < levelSize; i++)
    {
        var node = queue.Dequeue();

        // process node

        if (node.left != null)
            queue.Enqueue(node.left);

        if (node.right != null)
            queue.Enqueue(node.right);
    }
}
```

---

# MOST IMPORTANT LINKED-LIST REVERSAL TEMPLATE

```csharp
ListNode prev = null;
ListNode curr = head;

while (curr != null)
{
    ListNode next = curr.next;

    curr.next = prev;

    prev = curr;
    curr = next;
}

return prev;
```

---

# INTERVIEW HABIT

For each problem, quickly identify:

```text
1. What is the input?
2. What is being asked?
3. Which pattern fits?
4. Which data structure?
5. Edge cases?
6. Time complexity?
7. Space complexity?
```

Then explain the approach before coding.

## Problems completed so far

```text
1. Two Sum
2. First Unique Character
3. Longest Substring Without Repeating Characters
4. Valid Palindrome
5. Valid Parentheses
6. Reverse Linked List
7. Binary Search
8. Merge Intervals
9. Maximum Depth of Binary Tree
10. Binary Tree Level Order Traversal
```

## Next planned problems

```text
11. Number of Islands
12. Clone Graph
13. Currency Conversion
14. Top K Frequent Elements
15. Kth Largest Element
16. LRU Cache
17. Sliding Window variant
18. Subarray Sum Equals K
19. Meeting Rooms / Scheduling
20. Bloomberg-style mixed problem
```
