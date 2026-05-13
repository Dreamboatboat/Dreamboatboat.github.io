---
title: 面试常见考算法题
date: 2026-05-02 22:43:55
tags: code
---

## 1、手撕快速排序
```python
# 解法一 递归实现
class Solution:
    def sortArray(self, nums: List[int]) -> List[int]:
        if not nums: return []
        pivot = random.choice(nums)

        left = [i for i in nums if i < pivot]
        middle = [i for i in nums if i == pivot]
        right = [i for i in nums if i > pivot]

        return self.sortArray(left) + middle + self.sortArray(right)

# 解法二 遍历实现
class Solution:
    def sortArray(self, nums: List[int]) -> List[int]:
        def _partition(lo, hi):
            pivot_index = random.randint(lo, hi)
            nums[lo], nums[pivot_index] = nums[pivot_index], nums[lo]

            pivot = nums[lo]

            i, j  = lo - 1, hi + 1
            while True:
                i += 1
                while nums[i] < pivot:
                    i += 1
                j -= 1
                while nums[j] > pivot:
                    j -= 1
                if i >= j:
                    return j
            
                nums[i], nums[j] = nums[j], nums[i]
        
        def _quick(lo, hi):
            if lo < hi:
                p = _partition(lo, hi)
                _quick(lo, p)
                _quick(p + 1, hi)
        
        if len(nums) > 1:
            _quick(0, len(nums) - 1)
        
        return nums
```
---
## 2、反转链表
```python
class Solution:
    def reverseList(self, head: Optional[ListNode]) -> Optional[ListNode]:
        prev, cur = None, head

        while cur:
            next_node = cur.next
            cur.next = prev
            prev = cur
            cur = next_node
        
        return prev
```
---
## 3、LRU缓存
```python
class LinkedListnode():
    def __init__(self, key=0, val=0):
        self.key = key
        self.val = val
        self.prev = None
        self.next = None

class LRUCache:

    def __init__(self, capacity: int):
        self.capacity = capacity
        self.size = 0
        self.cache = {}
        self.head = LinkedListnode()
        self.tail = LinkedListnode()
        self.head.next = self.tail
        self.tail.prev = self.head
    
    def _add_node_to_head(self, node):
        node.next = self.head.next
        node.prev = self.head
        self.head.next.prev = node
        self.head.next = node
    
    def _remove_node(self, node):
        node.prev.next = node.next
        node.next.prev = node.prev
    
    def _move_node_to_head(self, node):
        self._remove_node(node)
        self._add_node_to_head(node)

    def _remove_tail(self):
        tail = self.tail.prev
        self._remove_node(tail)
        return tail

    def get(self, key: int) -> int:
        if key in self.cache:
            self._move_node_to_head(self.cache[key])
            return self.cache[key].val
        return -1        

    def put(self, key: int, value: int) -> None:
        if key in self.cache:
            node = self.cache[key]
            node.val = value
            self._move_node_to_head(node)
        else:
            node = LinkedListnode(key, value)
            self._add_node_to_head(node)
            self.cache[key] = node
            self.size += 1
            if self.size > self.capacity:
                tail = self._remove_tail()
                del self.cache[tail.key]
                self.size -= 1
```
---

## 4、环形链表
```python
class Solution:
    def detectCycle(self, head: Optional[ListNode]) -> Optional[ListNode]:
        slow = fast = head
        while fast and fast.next:
            slow, fast = slow.next, fast.next.next
            if slow == fast:
                break

        if not fast or not fast.next:
            return None
        
        slow = head
        while slow != fast:
            slow, fast = slow.next, fast.next
        return slow
```

## 5、无重复最长子串
```python
class Solution:
    def lengthOfLongestSubstring(self, s: str) -> int:
        n = len(s)
        if n <= 1: return n

        cache = set()
        max_len = 0
        left = 0

        for right in range(len(s)):
            while s[right] in cache:
                cache.remove(s[left])
                left += 1
            
            cache.add(s[right])
            max_len = max(max_len, right - left + 1)
        return max_len
```

## 6、有效的括号
```python
class Solution:
    def isValid(self, s: str) -> bool:
        if len(s) % 2 == 1:
            return False
        
        pairs = {
            ")": "(",
            "]": "[",
            "}": "{",
        }
        stack = list()
        for ch in s:
            if ch in pairs:
                if not stack or stack[-1] != pairs[ch]:
                    return False
                stack.pop()
            else:
                stack.append(ch)
        
        return not stack
```

## 7、三数之和
```python
class Solution:
    def threeSum(self, nums: List[int]) -> List[List[int]]:
        n = len(nums)
        nums.sort()
        ans = []

        for first in range(n):
            if first > 0 and nums[first] == nums[first - 1]:
                continue
            
            third = n - 1
            target = -nums[first]
            for second in range(first + 1, n):
                if second > first + 1 and nums[second] == nums[second - 1]:
                    continue
                
                while second < third and nums[second] + nums[third] > target:
                    third -= 1
                
                if second == third:
                    break

                if nums[second] + nums[third] == target:
                    ans.append([nums[first], nums[second], nums[third]])
            
        return ans
```

## 8、全排列
```python
class Solution:
    def permute(self, nums: List[int]) -> List[List[int]]:
        res = []
        used = [False] * len(nums)

        def backtrack(path):
            if len(path) == len(nums):
                res.append(path[:])
                return 
            
            for i in range(len(nums)):
                if used[i]: continue
                used[i] = True
                path.append(nums[i])
                backtrack(path)
                path.pop()
                used[i] = False
        
        backtrack([])
        return res
```

## 9、合并两个有序链表
```python
class Solution:
    def mergeTwoLists(self, list1: Optional[ListNode], list2: Optional[ListNode]) -> Optional[ListNode]:
        dummy = cur = ListNode()
        while list1 and list2:
            if list1.val <= list2.val:
                cur.next = list1
                list1 = list1.next
            else:
                cur.next = list2
                list2 = list2.next
            cur = cur.next
        
        cur.next = list1 if list1 else list2

        return dummy.next
```

## 10、前k个高频单词
```python
class Solution:
    def topKFrequent(self, words: List[str], k: int) -> List[str]:
        counter = collections.Counter(words)
        res = sorted(counter, key=lambda word: (-counter[word], word))
        return res[:k]
```

## 11、三角形最小路径和
```python
class Solution:
    def minimumTotal(self, triangle: List[List[int]]) -> int:
        dp = triangle[-1][:]

        for i in range(len(triangle) - 2, -1, -1):
            for j in range(len(triangle[i])):
                dp[j] = triangle[i][j] + min(dp[j], dp[j + 1])
        return dp[0]
```

## 12、划分字母区间
```python
class Solution:
    def partitionLabels(self, s: str) -> List[int]:
        last_occurence = {}
        for i, char in enumerate(s):
            last_occurence[char] = i
        
        start = 0
        end = 0
        res = []
        for i, char in enumerate(s):
            end = max(end, last_occurence[char])

            if i == end:
                res.append(end - start + 1)
                start = i + 1
        
        return res
```

## 13、搜索旋转排序数组
```python
class Solution:
    def search(self, nums: List[int], target: int) -> int:
        if not nums: return -1
        l, r = 0, len(nums) - 1
        while l <= r:
            mid = (l + r) // 2
            if nums[mid] == target:
                return mid
            if nums[0] <= nums[mid]:
                if nums[0] <= target < nums[mid]:
                    r = mid - 1
                else:
                    l = mid + 1
            else:
                if nums[mid] < target <= nums[len(nums) - 1]:
                    l = mid + 1
                else:
                    r = mid - 1
        return -1
```

## 14、回文数
```python
class Solution:
    def isPalindrome(self, x: int) -> bool:
        if x  < 0 or (x % 10 == 0 and x != 0):
            return False
        
        revertedNumber = 0
        while x > revertedNumber:
            revertedNumber = revertedNumber * 10 + x % 10
            x //= 10
        
        return x == revertedNumber or x == revertedNumber // 10

# 解法2
class Solution:
    def isPalindrome(self, x: int) -> bool:
        return str(x) == str(x)[::-1]
```

## 15、螺旋矩阵
```python
class Solution:
    def spiralOrder(self, matrix: List[List[int]]) -> List[int]:
        if not matrix or not matrix[0]: return []
        rows, columns = len(matrix), len(matrix[0])
        total = rows * columns
        visted = [[False] * columns for _ in range(rows)]
        order = [0] * total

        directions = [[0, 1], [1, 0], [0, -1], [-1, 0]]
        directionIndex = 0
        row, column = 0, 0
        for i in range(total):
            order[i] = matrix[row][column]
            visted[row][column] = True
            nextRow, nextColumn = row + directions[directionIndex][0], column + directions[directionIndex][1]
            if not (0 <= nextRow < rows and 0 <= nextColumn < columns and not visted[nextRow][nextColumn]):
                directionIndex = (directionIndex + 1) % 4
            row += directions[directionIndex][0]
            column += directions[directionIndex][1]
        return order 
```