+++
math = false
tags = [
    'algorithm',
    'code',
    'leetcode'
]
date = "2016-10-20T13:14:18+08:00"
title = "Reverse String"
image = ""

+++

### 问题描述
---
Write a function that takes a string as input and returns the string reversed.<!--more-->

**Example:**

Given `s = "hello"`, `return "olleh"`.

---

### Solution

最简单的方式就是让指针从末尾一直指向0，将指针位置的char累加至一个变量里

改进的做法是采用两个指针，一个从头开始，一个从尾开始，交换两个指针位置的值
```
/**
 * @param {string} s
 * @return {string}
 */
var reverseString = function(s) {
    let left = '';
    let right = '';
    for (var i = s.length-1,j=0; i > j; i--,j++) {
        right = s[j] + right;
        left += s[i];
    }
    
    return left + (i==j?s[i]:'') + right;
};
```
