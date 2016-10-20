+++
math = false
tags = [
    "algorithm", 
    "code",
    'leetcode'
]
image = ""
date = "2016-10-20T21:30:00+08:00"
title = "Queue Reconstruction by Height"

+++

### 问题描述
---
Suppose you have a random list of people standing in a queue. Each person is described by a pair of integers `(h, k)`, where `h` is the height of the person and `k` is the number of people in front of this person who have a height greater than or equal to `h`. Write an algorithm to reconstruct the queue.<!--more-->

**Note:**

The number of people is less than `1,100`.

**Example**

```
Input:
[[7,0], [4,4], [7,1], [5,0], [6,1], [5,2]]

Output:
[[5,0], [7,0], [5,2], [6,1], [4,4], [7,1]]
```

---

### Solution
```
/**
 * @param {number[][]} people
 * @return {number[][]}
 */
var reconstructQueue = function(people) {
    //先根据h从大到小进行排序，h相同的，根据k从小到大排序
    people.sort((a,b)=>a[0]===b[0]?a[1]-b[1]:b[0]-a[0]);
    people.unshift([]);
    //k代表在其前面比他高的个数，由于已经根据高度排过序，所以直接在该位置插入该元素即可
    return people.reduce((list,next)=>{
        list.splice(next[1],0,next);
        return list;
    });
};
```