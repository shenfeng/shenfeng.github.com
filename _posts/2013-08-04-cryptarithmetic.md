---
author: feng
date: '2013-08-04 08-21-00'
layout: post
status: publish
title: 算式迷 cryptarithmetic 的python解法
---

这周末两天都在听 Peter Norvig 在udacity的公开课《Design of Computer Programs》，[https://www.udacity.com/course/cs212](https://www.udacity.com/course/cs212)，很有意思，很受启发。Peter Norvig一把年纪了（1956年生，今年56岁了），很有激情，非常佩服

庆幸大学时期便坚持完整读厚厚的英文原版图书，坚持听英文的公开课，使得听Peter Norvig视频，没有障碍。如此才能受教于Peter Norvig，虽仅视频观摩，仍觉三生有幸，受益匪浅。

在第二讲“Back of the Envelope”里面，他一步一步的设计程序，来解算式迷（cryptarithmetic），并优化性能。算式迷也是我小学时期做的题目，长大了，发现居然还能用计算机程序轻松的解决，觉得很有意思，记录下来，权当备忘：

### 什么是算式迷（cryptarithmetic）

例如:

![sample](/imgs/cryptarithmetic1.png)

*图片来自维基百科 [Verbal arithmetic](http://en.wikipedia.org/wiki/Verbal_arithmetic)*

1. 字母代表0-9之间的数字.
2. 不同的字母代表不同的数字.
3. 相同的字母代表相同的数字.

那么各个字母分别代表什么数字，才能使等式成立？

### 程序解法

程序列举出所有可能，挑出满足条件的，便是答案。问题变为：

1. 如何列举出所有可能

2. 如何判断满足条件

#### 用排列列举出所有可能

`itertools.permutations`可以列出排列，比如
{% highlight python %}
# 1， 2， 3的全排列
list(itertools.permutations((1, 2, 3)))
=>
[(1, 2, 3), (1, 3, 2), (2, 1, 3), (2, 3, 1), (3, 1, 2), (3, 2, 1)]

# 1， 2， 3的全排列，仅取前面2位
list(itertools.permutations((1, 2, 3), 2))
=>
[(1, 2), (1, 3), (2, 1), (2, 3), (3, 1), (3, 2)]

def fill_in(formula):
    # 找出所有的大写字母（替换为数字）
    letters = ''.join(set([s for s in formula if s in string.uppercase]))
    # 列出所有的可能
    for i in itertools.permutations('0123456789', len(letters)):
        trans = string.maketrans(letters, ''.join(i))
        # 变换。 比如"A+A==B" => "2+2==3"
        yield string.translate(formula, trans)

{% endhighlight %}


#### 如何判断满足条件

`eval` 可以用来帮忙：
{% highlight python %}
eval('123 + 345 == 567') => False
eval('123 + 345 == 468') => True

def valid(f):
    "Return True iff the statement eval to True, eg: 1+1==2"
    try:
        return not re.search(r'\b0[0-9]', f) and eval(f) is True
    except ArithmeticError:
        return False
{% endhighlight %}

#### 解决问题

{% highlight python %}
def solve(formula):
    return (f for f in fill_in(formula) if valid(f))

for s in solve('SEND+MORE==MONEY'):
    print s

=> 9567+1085==10652

{% endhighlight %}

但解决的过程比较漫长，我的电脑需要需要花掉17.5s，才能找出解。profile发现eval花了大部分时间(14.3s)：

{% highlight sh %}
  python -m cProfile code.py

  1814400    0.920    0.000    4.042    0.000 re.py:139(search)
  1814400    1.136    0.000    1.472    0.000 re.py:226(_compile)
  1451520   13.955    0.000   14.334    0.000 {eval}
{% endhighlight %}

### 优化性能

Peter Norvig说，优化它有两种做法：使eval变得更快，或者减少调用次数。他选择了第二种方式：
eval一个string，需要经历parse，compile为bytecode，执行的三个阶段，前两个阶段是可以复用的：
比如可以把表达式变成下面的函数：
{% highlight python %}

# SEND+MORE==MONEY
lambda E, D, M, O, N, S, R, Y: (1*D+10*N+100*E+1000*S)+(1*E+10*R+100*O+1000*M)==(1*Y+10*E+100*N+1000*O+10000*M)

# E, D, M, O, N, S, R, Y 是参数

{% endhighlight %}

变成函数的过程，可以用eval，并且仅仅调用一次，其它的仅是调用生成的函数

{% highlight python %}
def compile_word(word):
    if word.isupper():
        terms = ['%d*%s' % (10**i, d)
                for i, d in enumerate(word[::-1])]
        return '(' + '+'.join(terms) + ')'
    else:
        return word

def compile_formula(formula, verbose=False):
    letters = set(re.findall('[A-Z]', formula))
    terms = re.split('([A-Z]+)', formula)
    params = ', '.join(letters)
    expr = ''.join(map(compile_word, terms))
    f = 'lambda %s: %s' % (params, expr)
    if verbose: print f
    return eval(f), ''.join(letters)

def faster_solve(formula):
    f, letters = compile_formula(formula, True)

    for digits in itertools.permutations(range(10), len(letters)):
        try:
            if f(*digits) is True:
                table = string.maketrans(letters, ''.join(map(str, digits)))
                r = string.translate(formula, table)
                # 数字的首位不能是0
                if not re.search(r'\b0[0-9]', r):
                    yield r
        except ArithmeticError:
            pass
{% endhighlight %}

在我的机器上，`faster_solve` 仅需要1.2s，比未优化的`solve`快10几倍

Google后发现，还有用prolog解决此问题的，速度比这个快上千倍（花的CPU时间在ms一下）[Cryptarithmetic Puzzle Solver](http://bach.istc.kobe-u.ac.jp/llp/crypt.html)，以后有时间再研究一下，应该比较有意思。
