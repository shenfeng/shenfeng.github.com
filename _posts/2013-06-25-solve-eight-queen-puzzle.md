---
author: feng
date: '2013-6-25 08-21-00'
layout: post
status: publish
desc: 动画演示8皇后的解的搜索过程，python语言实现，回溯算法
title: 图文详解8皇后，python实现
categories: ['algorithm', 'python']
---

*在给女朋友讲8皇后时，用python画了一些图，还做了个gif帮助理解，记录下来，可能对在学习8皇后的朋友有用*

8皇后是一个很有意思的问题，经典的回溯算法教给我们怎么系统的搜索：从一条路往前走，能进则进，不能进则退回来，换条路再试。

### 问题定义
8皇后问题：在8X8格的国际象棋上摆放八个皇后，使其不能互相攻击，即任意两个皇后都不能处于同一行、同一列或同一斜线上

### 结果表示：
由于一个皇后只能在一行，因此可以使用一个数组来记录，数组数字表示皇后在棋牌的列，数组的下标则表示行。python数组下标以0开始，故棋牌横0-7，纵着0-7，例如：

{% highlight python %}
[0, 4, 7, 5, 2, 6, 1, 3]
{% endhighlight %}

表示的棋盘为：

![sample](/imgs/8queen/sample.png)

#### 搜索皇后在n行的位置

前n-1行皇后的位置，存在arr数组里。需要在第n行找一列，使皇后放在那里，不与前面n-1个皇后冲突。
如果`back_tracing` 为`True`，则是在回溯状态，在第n行的result[n_row]列有个皇后，需要看可不可以把她往右边移动

{% highlight python %}
def find_legal_position(arr, n_row, n_queens, trace_back=False):
    start = 0
    if trace_back:
        start = arr[n_row] + 1
    for col in range(start, n_queens):
        ok = True
        for row in range(n_row):  # [0, n_row)
            if arr[row] == col:  # two queens share the same column
                ok = False
                break
            if n_row - row == abs(col - arr[row]): # two queens share the same diagonal
                ok = False
                break
        if ok:
            return col
    return None  # no position we can place a queen on the n_row

{% endhighlight %}

#### 回溯解8皇后

从一条路往前走，能进则进，不能进则退回来，换条路再试

{% highlight python %}
def solve_queen_puzzle(n_queens):
    result = [0] * n_queens
    n_row = 0  # 从0行开始，一行一行安置皇后
    while n_row < n_queens:
        pos = find_legal_position(result, n_row, n_queens)
        if pos is not None: # 往前走
            result[n_row] = pos
            if n_row == n_queens - 1:
                yield result[:] # 找到一个结果，返回
        if pos is None or n_row == n_queens - 1:
            while 1:
                n_row -= 1
                if n_row < 0:
                    return # 所有路已经穷尽
                pos = find_legal_position(result, n_row, n_queens, back_tracing=True)
                if pos:  # 换条路试一下
                    result[n_row] = pos
                    break
        n_row += 1
{% endhighlight %}

### 动画演示搜索过程

红色表示无路可走时进行回溯

![sample](/imgs/8queen/all.gif)