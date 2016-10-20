+++
image = ""
math = false
tags = [
    "algorithm", 
    "code",
    'leetcode'
]
date = "2016-10-20T21:08:32+08:00"
title = "Battle ships in a Board"

+++

### 问题描述
---
Given an 2D board, count how many different battleships are in it. The battleships are represented with `'X'`s, empty slots are represented with `'.'`s. You may assume the following rules:<!--more-->

* You receive a valid board, made of only battleships or empty slots.
* Battleships can only be placed horizontally or vertically. In other words, they can only be made of the shape `1xN` (1 row, N columns) or `Nx1` (N rows, 1 column), where N can be of any size.
* At least one horizontal or vertical cell separates between two battleships - there are no adjacent battleships.

**Example:**

```
X..X
...X
...X
```
In the above board there are 2 battleships.
```
...X
XXXX
...X
```
This is not a valid board - as battleships will always have a cell separating between them.
Your algorithm should not modify the value of the board.

---

### Solution
```
/**
 * @param {character[][]} board
 * @return {number}
 */
var countBattleships = function(board) {
    let width = board[0].length;
    let height = board.length;
    let tag_num = 0;
    for (let i = 0; i < height; i ++) {
        for (let j = 0; j < width; j ++) {
            let cell = board[i][j];
            if (cell == '.') {
                continue;
            }
            if (cell == 'X' && 
            (i == 0 || board[i-1][j] == '.' )&&
            ( j == 0 || board[i][j-1] == '.' ) ) {
                //找到一个船的部分，并且左方和上方无其余部分的时候累加
                tag_num ++;
            }
        }
    }
    return tag_num;
};

```