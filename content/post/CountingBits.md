+++
date = "2016-10-20T09:56:13+08:00"
title = "Counting Bits"
tags = [
    "algorithm", 
    "code",
    'leetcode'
]
draft = false
+++
### 问题描述
---
Given a non negative integer number num. For every numbers i in the range `0 ≤ i ≤ num` calculate the number of 1's in their binary representation and return them as an array.<!--more-->

**Example:**
For `num = 5` you should `return [0,1,1,2,1,2]`.

Follow up:

It is very easy to come up with a solution with run time `O(n*sizeof(integer))`. But can you do it in linear time `O(n)` /possibly in a single pass?
Space complexity should be `O(n)`.
Can you do it like a boss? Do it without using any builtin function like `__builtin_popcount` in c++ or in any other language.

Hint:

* You should make use of what you have produced already.
* Divide the numbers in ranges like `[2-3]`, `[4-7]`, `[8-15]` and so on. And try to generate new range from previous.
* Or does the odd/even status of the number help you in calculating the number of 1s?

---

### Solution

根据提示可以知道`[2-3]`为`[10,11]`，`[4-7]`为`[100,111]`，可以看出范围的划分为`2^n ~ 2^(n+1)-1`，要利用已有的结果，即`1**`实际上是在`**`的基础上加了`1`，`1*`又是在`*`的基础上加`1`

>`num = 5`的结果为`[0,1,1,2,1,2]`

可以看作

```
    0  1
    0  1  (0  1)+1  =  0  1  1  2
    0  1  1  2  (0  1  1  2)+1 = 0  1  1  2  1  2  2  3
```
得到如下解法
```
/**
 * @param {number} num
 * @return {number[]}
 */
var countBits = function(num) {
    var border = 1;
    var arr = [0];
    for (let i = 1;i<=num ;i++) {
        
        if (i >= border) {
            let idx = i - border;
            arr.push(arr[idx]+1);
            if (i == 2*border-1) {
                border *= 2;
            }
        }
        
    }
    
    return arr;
};
```
还可以考虑成

```
 7 => 111     >>>1      右移一位  成了 11（即3）  ，3所包含的1的数量再加上最末位的数值（0或1）就是结果
```
得到如下
```
/**
 * @param {number} num
 * @return {number[]}
 */
var countBits = function(num) {
    var arr = [0];
    for (let i = 1;i<=num ;i++) {
        
        arr.push(arr[i>>>1]+(i&1));
        
    }
    
    return arr;
};
```