---
title: 1647.使数组互补的最小操作次数
date: 2026-05-13 10:14:55
tags: [leetcode, 差分, 数组]
---

## 题目描述
给你一个长度为 **偶数`n`**的整数数组`nums`和一个整数`limit`。每一次操作，你可以将`nums`中的任何整数替换为`1`到`limit`之间的另一个整数。

如果对于所有下标`i`（从0开始）`nums[i] + nums[n - 1 - i]`都等于同一个数，则数组`nums`是互补的。例如，数组`[1,2,3,4]`是互补的，因为所有下标`i`，`nums[i] + nums[n - 1 - i] = 5`。

返回使数组 互补 的最少操作次数

> 示例 1：
> 输入：nums = [1,2,4,3], limit = 4
> 输出：1
> 解释：
>  经过 1 次操作，你可以将数组 nums 变成 [1,2,2,3]（加粗元素是变更的数字）：
> nums[0] + nums[3] = 1 + 3 = 4.
> nums[1] + nums[2] = 2 + 2 = 4.
> nums[2] + nums[1] = 2 + 2 = 4.
> nums[3] + nums[0] = 3 + 1 = 4.
> 对于每个 i ，nums[i] + nums[n-1-i] = 4 ，所以 nums 是互补的。。

## 解法：差分数组
**思路：**
对于每一对对称元素：
(nums[i], nums[n - i - 1])
设：
```
Python
x, y = nums[i], nums[n - i - 1]
a = min(x, y)
b = max(x, y)
```
目标是让所有pair的和都变成同一个值s。
对于一个pair：
- 当s == a + b时，不需要修改，代价为 0 
- 当s 属于 [a + 1, b + limit]时，只需要修改一个数，代价为 1
- 其他情况需要修改两个数，代价为 2
因此每个pair对所有target sum对贡献呈现如下变化：
```
2 ---------------- a+1 -------- a+b -------- b+limit -------- 2*limit
        2次            1次           0次            1次
```
如果暴力枚举所有sum，会是O(n * limit)

这里使用差分数组优化。

差分数组只记录“代价变化的位置”
```
diff[2] += 2                # 默认全部是2次修改
diff[a + 1] -= 1            # 进入1次修改区间
diff[a + b] -= 1            # 到达0次修改点
diff[a + b + 1] += 1        # 离开0次修改点
diff[b + limit + 1] += 1    # 离开1次修改区间
```
最后通过前缀和恢复每个sum的总代价，取最小值即可。
**代码：**
```python
class Solution:
    def minMoves(self, nums: List[int], limit: int) -> int:
        n = len(nums)
        diff = [0] * (2 * limit + 2)

        for i in range(n // 2):
            a = min(nums[i], nums[n - i - 1])
            b = max(nums[i], nums[n - i - 1])

            diff[2] += 2
            diff[a + 1] -= 1
            diff[a + b] -= 1
            diff[a + b + 1] += 1
            diff[b + limit + 1] -= 1
        
        min_ops = n
        cur_ops = 0

        for s in range(2, 2 * limit + 1):
            cur_ops += diff[s]
            if min_ops > cur_ops:
                min_ops = cur_ops
        return min_ops
```
时间复杂度：O(n + limit)
空间复杂度：O(limit)
