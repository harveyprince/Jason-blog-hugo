+++
tags = [
    'algorithm',
    'code',
    'leetcode'
]
date = "2016-10-20T11:19:14+08:00"
title = "Number of 1 Bits"
image = ""
math = false

+++

### 问题描述
---
Write a function that takes an unsigned integer and returns the number of `’1'` bits it has (also known as the Hamming weight).<!--more-->

For example, the `32-bit` integer `’11'` has binary representation `00000000000000000000000000001011`, so the function should `return 3`.

---

### Solution

最简单的做法就是将所有位的0或1加起来就是最终结果，这里就不赘述了

方法1:
通过正则进行处理
```
/**
 * @param {number} n - a positive integer
 * @return {number}
 */
var hammingWeight = function(n) {
    return (n).toString(2).replace(/0/g, '').length;
    
};
```

方法2:
通过位操作符进行

通过研究可以知道
```
假设
  n        =>    11000
  n-1      =>    10111

  则
  n & n-1  =>    10000

  就可以做到从右往左逐渐将1抹去      
```
得到解法
```
/**
 * @param {number} n - a positive integer
 * @return {number}
 */
var hammingWeight = function(n) {
    let count = 0;
    while(n!=0){
        count ++;
        n &= (n-1);
    }
    return count;
};
```

实验证明正则的处理更快