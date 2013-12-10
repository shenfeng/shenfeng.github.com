---
author: feng
date: '2012-11-30 08-21-00'
layout: post
status: publish
desc: golang，go语言学习笔记，btree简单实现：插入，查找
title: go语言学习笔记：B-tree
categories: ['golang']
---

这段时间对google出的go语言比较感兴趣。比较看中的原因：
1. Robert Griesemer, [Rob Pike](http://en.wikipedia.org/wiki/Rob_Pike), [Ken Thompson](http://en.wikipedia.org/wiki/Ken_Thompson)。 Unix，UTF8，正则表达式等等有他诸多贡献。 `Rob Pike`：Unix，UTF8，Plan 9等，并且几十年的并发开发。`Robert Griesemer`： hotspot jvm。 他们都是计算机行业的牛人， 牛人出品，值得一试。
2. go简单明了
3. 通过`go` `goroutine` `select` `channel`来对解决并发问题。

用它写程序是一种学习方法，就试着写了一下[B-tree](http://en.wikipedia.org/wiki/B-tree)，回忆一下大学的课程

{% highlight go %}

package btree
import (
	"bytes"
	"fmt"
)
type Key int
type Node struct {
	Leaf     bool
	N        int
	Keys     []Key
	Children []*Node
}
func (x *Node) Search(k Key) (n *Node, idx int) {
	i := 0
	for i < x.N && x.Keys[i] < k {
		i += 1
	}
	if i < x.N && k == x.Keys[i] {
		n, idx = x, i
	} else if x.Leaf == false {
		n, idx = x.Children[i].Search(k)
	}
	return
}
func newNode(n, branch int, leaf bool) *Node {
	return &Node{
		Leaf:     leaf,
		N:        n,
		Keys:     make([]Key, branch*2-1),
		Children: make([]*Node, branch*2),
	}
}
func (parent *Node) Split(branch, idx int) { //  idx is Children's index
	full := parent.Children[idx]
	// make a new node, copy full's right most to it
	n := newNode(branch-1, branch, full.Leaf)
	for i := 0; i < branch-1; i++ {
		n.Keys[i] = full.Keys[i+branch]
		n.Children[i] = full.Children[i+branch]
	}
	n.Children[branch-1] = full.Children[2*branch-1] // copy last child
	full.N = branch - 1 // is half full now, copied to n(new one)
	// shift parent, add new key and children
	for i := parent.N; i > idx; i-- {
		parent.Children[i] = parent.Children[i-1]
		parent.Keys[i+1] = parent.Keys[i]
	}
	parent.Keys[idx] = full.Keys[branch-1]
	parent.Children[idx+1] = n
	parent.N += 1
}
func (tree *Btree) Insert(k Key) {
	root := tree.Root
	if root.N == 2*tree.branch-1 {
		s := newNode(0, tree.branch, false)
		tree.Root = s
		s.Children[0] = root
		s.Split(tree.branch, 0)
		s.InsertNonFull(tree.branch, k)
	} else {
		root.InsertNonFull(tree.branch, k)
	}
}
func (x *Node) InsertNonFull(branch int, k Key) {
	i := x.N
	if x.Leaf {
		for i > 0 && k < x.Keys[i-1] {
			x.Keys[i] = x.Keys[i-1]
			i -= 1
		}
		x.Keys[i] = k
		x.N += 1
	} else {
		for i > 0 && k < x.Keys[i-1] {
			i -= 1
		}
		c := x.Children[i]
		if c.N == 2*branch-1 {
			x.Split(branch, i)
			if k > x.Keys[i] {
				i += 1
			}
		}
		x.Children[i].InsertNonFull(branch, k)
	}
}
func space(n int) string {
	s := ""
	for i := 0; i < n; i++ {
		s += " "
	}
	return s
}
func (x *Node) String() string {
	return fmt.Sprintf("{n=%d, Leaf=%v, Keys=%v, Children=%v}\n",
		x.N, x.Leaf, x.Keys, x.Children)
}
func (tree *Btree) String() string {
	return tree.Root.String()
}
type Btree struct {
	Root   *Node
	branch int
}
func New(branch int) *Btree {
	return &Btree{Root: newNode(0, branch, true), branch: branch}
}
func (tree *Btree) Search(k Key) (n *Node, idx int) {
	return tree.Root.Search(k)
}

{% endhighlight %}
