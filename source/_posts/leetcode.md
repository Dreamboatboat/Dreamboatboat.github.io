---
title: leetcode_test
---

## 1、数组排序

```python
# 快速排序 递归解法
import random
class Solution:
    def sortArray(self, nums: List[int]) -> List[int]:
        if not nums: return []
        pivot = random.choice(nums)
        
        left = [i for i in nums if i < pivot]
        middle = [i for i in nums if i == pivot]
        right = [i for i in nums if i > pivot]

        return self.sortArray(left) + middle + self.sortArray(right)

# 快速排序 第二种写法
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

## 链表相关
### 反转链表
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

### 环形链表Ⅱ
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

## LRU 缓存
```python
class LinkedListNode():
    def __init__(self, key=0, value=0):
        self.key = key
        self.value = value
        self.prev = None
        self.next = None

class LRUCache:

    def __init__(self, capacity: int):
        self.capacity = capacity
        self.size = 0
        self.cache = {}
        self.head = LinkedListNode()
        self.tail = LinkedListNode()
        self.head.next = self.tail
        self.tail.prev = self.head 

    def _add_node_to_head(self, node):
        self.head.next.prev = node
        node.next = self.head.next
        node.prev = self.head
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
            return self.cache[key].value
        else:
            return -1

    def put(self, key: int, value: int) -> None:
        if key in self.cache:
            self.cache[key].value = value
            self._move_node_to_head(self.cache[key])
        else:
            node = LinkedListNode(key, value)
            self._add_node_to_head(node)
            self.cache[key] = node
            self.size += 1
            if self.size > self.capacity:
                tail = self._remove_tail()
                del self.cache[tail.key]
                self.size -= 1
```
## 动态规划
