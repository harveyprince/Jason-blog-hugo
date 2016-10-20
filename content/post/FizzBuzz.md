+++
title = "Fizz Buzz"
image = ""
math = false
tags = [
    'algorithm',
    'code',
    'leetcode'
]
date = "2016-10-20T12:37:17+08:00"

+++

### 问题描述
---
Write a program that outputs the string representation of numbers from 1 to n.

But for multiples of three it should output “Fizz” instead of the number and for the multiples of five output “Buzz”. For numbers which are multiples of both three and five output “FizzBuzz”.<!--more-->

**Example:**
```
n = 15,

Return:
[
    "1",
    "2",
    "Fizz",
    "4",
    "Buzz",
    "Fizz",
    "7",
    "8",
    "Fizz",
    "Buzz",
    "11",
    "Fizz",
    "13",
    "14",
    "FizzBuzz"
]
```

---
### Solution
```
/**
 * @param {number} n
 * @return {string[]}
 */
var fizzBuzz = function(n) {
    var res = [];
    for (let i = 1; i <=n ; i++) {
        let name = '';
        i%3 == 0?name+='Fizz':'';
        i%5 == 0?name+='Buzz':'';
        res.push(name==''?i .toString():name);
    }
    return res;
};
```