---
title: "二分查找的变化"
date: 2022-04-14T23:55:27+08:00
draft: false

summary: 梳理一下二分查找的几种写法
description: 简单梳理一下二分查找的几种写法

tags: ["cpp", "algorithm"]
categories: ["LeetCode"]
---

## 判断一个数是否存在

```C++
// program 1.1
int binarySerarch(vector<int>& nums, int target)
{
    int left;
    int mid;
    int right;

    left = 0;
    right = nums.size();                    // 只能访问 [left, right) 区间中的元素
    
    while(left < right)                     // left < right 意味着 [left, right) 仍然是一个有效的区间, 可以继续搜索
    {
        mid = (right - left) / 2 + left;    // 划分出两个区间, 一个点 [left, mid), mid, [mid + 1, right)

        if (target < nums[mid])             // target 位于左区间
        {
            right = mid;                    // 不应当 -1, 网络上一些写法是有问题的
        }
        else if (target > nums[mid])        // target 位于右区间
        {
            left = mid + 1;
        }
        else // if (target == nums[mid])    // target 恰为mid
        {
            return mid;
        }
    }
    
    return -1;
}
```

```C++
// program 1.2
int binarySerarch(vector<int>& nums, int target)
{
    int left;
    int mid;
    int right;

    left = 0;
    right = nums.size() - 1;                // 只能访问 [left, right] 区间中的元素
    
    while(left <= right)                    // left <= right 意味着 [left, right] 仍然是一个有效的区间, 可以继续搜索
    {
        mid = (right - left) / 2 + left;    // 划分出两个区间, 一个点 [left, mid - 1], mid, [mid + 1, right]

        if (target < nums[mid])             // target 位于左区间
        {
            right = mid - 1;
        }
        else if (target > nums[mid])        // target 位于右区间
        {
            left = mid + 1;
        }
        else // if (target == nums[mid])    // target 恰为 mid
        {
            return mid;
        }
    }

    return -1;
}
```

## 寻找一个左侧边界

```C++
// program 2.1
int lower_bound(vector<int>& nums, int target)
{
    int left;
    int mid;
    int right;

    left = 0;
    right = nums.size();

    while(left < right)
    {
        mid = (right - left) / 2 + left;    // [left, mid), mid, [mid + 1, right)

        if (target < nums[mid])
        {
            right = mid;
        }
        else if (target > nums[mid])
        {
            left = mid + 1;
        }
        else // if (target == nums[mid])
        {
            right = mid;                    // 与 program 1.1 的唯一区别就在这里
        }
    }

    if(left > nums.size() || nums[left] != target) // 对这里的 if 的意义详见后文
    {
        return -1;
    }

    return left;
}
```

简化后即可得到:

```C++
// program 2.2
int lower_bound(vector<int>& nums, int target)
{
    int left;
    int mid;
    int right;

    left = 0;
    right = nums.size();

    while(left < right)
    {
        mid = (right - left) / 2 + left;

        if (target > nums[mid])
        {
            left = mid + 1;
        }
        else
        {
            right = mid;
        }
    }

    if(left > nums.size() || nums[left] != target)    // 可能依然会越界, 后期验证一下
    {
        return -1;
    }

    return left;
}
```

这里需要注意一下最后的 if 判断, 实际上，无论是 Python 标准库：

```Python
def bisect_left(a, x, lo=0, hi=None):
    """Return the index where to insert item x in list a, assuming a is sorted.
    The return value i is such that all e in a[:i] have e < x, and all e in
    a[i:] have e >= x.  So if x already appears in the list, a.insert(x) will
    insert just before the leftmost x already there.
    Optional args lo (default 0) and hi (default len(a)) bound the
    slice of a to be searched.
    """

    if lo < 0:
        raise ValueError('lo must be non-negative')
    if hi is None:
        hi = len(a)
    while lo < hi:
        mid = (lo+hi)//2
        # Use __lt__ to match the logic in list.sort() and in heapq
        if a[mid] < x: lo = mid+1
        else: hi = mid
    return lo
```

还是 C++ 给出的 possible implementation, 都没有这里的 if 判断:

```C++
template<class ForwardIt, class T, class Compare>
ForwardIt lower_bound(ForwardIt first, ForwardIt last, const T& value, Compare comp)
{
    ForwardIt it;
    typename std::iterator_traits<ForwardIt>::difference_type count, step;
    count = std::distance(first, last);
 
    while (count > 0) {
        it = first;
        step = count / 2;
        std::advance(it, step);
        if (comp(*it, value)) {
            first = ++it;
            count -= step + 1;
        }
        else
            count = step;
    }
    return first;
}
```

这是因为 Python 和 C++ 中的 `lower_bound()` 函数被设计为: 返回第一个**不小于**指定值的元素. 因此如果希望严格找到第一个等于指定值的元素，还需要使用 if 进行进一步判断.

## 寻找一个右侧边界

在 program 2.2的基础上, 很容易得到下面的代码:

```C++
// program 3.1
int upper_bound(vector<int>& nums, int target)
{
    int left;
    int mid;
    int right;
    
    left = 0;
    right = nums.size();

    while (left < right)
    {
        mid = left + (right - left) / 2;

        if (target < nums[mid])             // 恰与 program 2.2 相反
        {
            right = mid;
        }
        else
        {
            left = mid + 1;
        }
    }

    if (left >= nums.size() || nums[left - 1] != target)
    {
        return -1;
    }

    return left - 1;                        // 这里与 program 2.2 不同
}
```
