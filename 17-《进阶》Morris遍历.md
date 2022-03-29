[TOC]

# 1 Morris遍历

## 1.1 Morris遍历目的

在二叉树的遍历中，有递归方式遍历和非递归遍历两种。不管哪种方式，时间复杂度为O(N)，空间复杂度为O(h),h为树的高度。Morris遍历可以在时间复杂度O(N),空间复杂度O(1)实现二叉树的遍历


### 算法流程

从一个树的头结点cur开始：

1、cur的左树为null，cur = cur.right

2、cur有左树，找到左树的最右的节点mostRight

- mostRight的右孩子指向null，让mostRigth.right = cur;cur = cur.left
- mostRight的右孩子指向当前节点cur，让mostRigth.right = null;cur = cur.rigth
- cur为null的时候，整个流程结束

cur经过的各个节点，称为Morris序，例如如下的树结构：

```
graph TD
1-->2
1-->3
2-->4
2-->5
3-->6
3-->7
```

cur经过的路径也就是Morris序为：1，2，4，2，5，1，3，6，3，7

可以发现，只要当前节点有左孩子，当前节点会来到两次。当左树最右节点的右孩子指向null，可以判定第一次来到cur节点，下一次来到cur就是发现左树的最右节点指向的是自己，说明第二次来到cur节点

在Morris遍历的过程中，第一次到达cur就打印（只会来cur一次的节点也打印）就是树的先序遍历，第二次到达（只会来到cur一次的节点也打印）打印为中序。第二次来到cur节点的时候逆序打印cur左树的右边界，最后逆序打印整颗树的右边界，逆序打印右边界不可以使用额外的结构，因为我们要求空间复杂度为O(1)，可以使用翻转链表

翻转链表：例如a->b->c->d->null。可以让a-null,b->a,c->b,d->c即可。打印完之后把链表再翻转过来

### 时间复杂度估计

cur来到节点的时间复杂度为O(N),每个cur遍历左树最右边界的代价最多为O(2N),所以整个遍历过程为O(N)，整个遍历过程只用到到有限的几个变量，其他使用的都是树本身的指针关系。



```Go
package main

import (
	"fmt"
)

type Node struct {
	Left *Node
	Right *Node
	Val int
}

// morris遍历二叉树，实现时间复杂度O(N), 空间复杂度O(1)。正常的递归遍历，时间复杂度O(N)，空间复杂度O(N)
func morris(head *Node) {
	if head == nil {
		return
	}

	cur := head
	var mostRight *Node
	for cur != nil {
		// cur有没有左树
		mostRight = cur.Left
		if mostRight != nil { // 有左树的情况下
			// 找到cur左树上，真实的最右
			for mostRight.Right != nil && mostRight.Right != cur {
				mostRight = mostRight.Right
			}

			// 从while中出来，mostRight一定是cur左树上的最右节点
			// mostRight如果等于null，说明第一次来到自己
			if mostRight.Right == nil {
				mostRight.Right = cur
				cur = cur.Left
				continue
			} else {
				// 否则第二次来到自己，意味着mostRight.right = cur
				// mostRight.right != null -> mostRight.right == cur
				mostRight.Right = nil
			}
		}
		cur = cur.Right
	}
}

// morris 先序遍历二叉树
func morrisPre(head *Node) {
	if head == nil {
		return
	}

	// cur
	cur1 := head
	// mostRight
	var cur2 *Node

	for cur1 != nil {
		cur2 = cur1.Left
		if cur2 != nil {
			for cur2.Right != nil && cur2.Right != cur1 {
				cur2 = cur2.Right
			}
			if cur2.Right == nil {
				cur2.Right = cur1
				fmt.Print(fmt.Sprintf("%d%s", cur1.Val, " "))
				cur1 = cur1.Left
				continue
			} else {
				cur2.Right = nil
			}
		} else {
			fmt.Print(fmt.Sprintf("%d%s", cur1.Val, " "))
		}
		cur1 = cur1.Right
	}
	fmt.Println()
}

// morris 中序遍历
func morrisIn(head *Node) {
	if head == nil {
		return
	}

	cur := head
	var mostRight *Node
	for cur != nil {
		mostRight = cur.Left
		if mostRight != nil {
			for mostRight.Right != nil && mostRight.Right != cur {
				mostRight = mostRight.Right
			}
			if mostRight.Right == nil {
				mostRight.Right = cur
				cur = cur.Left
				continue
			} else {
				mostRight.Right = nil
			}
		}
		fmt.Print(fmt.Sprintf("%d%s", cur.Val, " "))
		cur = cur.Right
	}
	fmt.Println()
}

// morris 后序遍历
func morrisPos(head *Node) {
	if head == nil {
		return
	}

	cur := head
	var mostRight *Node
	for cur != nil {
		mostRight = cur.Left
		if mostRight != nil {
			for mostRight.Right != nil && mostRight.Right != cur {
				mostRight = mostRight.Right
			}
			if mostRight.Right == nil {
				mostRight.Right = cur
				cur = cur.Left
				continue
			} else { // 回到自己两次，且第二次回到自己的时候是打印时机
				mostRight.Right = nil
				// 翻转右边界链表，打印
				printEdge(cur.Left)
			}
		}
		cur = cur.Right
	}
	// while结束的时候，整颗树的右边界同样的翻转打印一次
	printEdge(head)
	fmt.Println()
}

func printEdge(head *Node) {
	tali := reverseEdge(head)
	cur := tali
	for cur != nil {
		fmt.Print(fmt.Sprintf("%d%s", cur.Val, " "))
		cur = cur.Right
	}
	reverseEdge(tali)
}

func reverseEdge(from *Node) *Node {
	var pre *Node
	var next *Node
	for from != nil {
		next = from.Right
		from.Right = pre
		pre = from
		from = next
	}
	return pre
}

// 在Morris遍历的基础上，判断一颗树是不是一颗搜索二叉树
// 搜索二叉树是左比自己小，右比自己大
// 一颗树中序遍历，值一直在递增，就是搜索二叉树
func isBST(head *Node) bool {
	if head == nil {
		return true
	}
	
	cur := head
	var mostRight *Node
	var pre int
	var ans bool
	for cur != nil {
		mostRight = cur.Left
		if mostRight != nil {
			for mostRight.Right != nil && mostRight.Right != cur {
				mostRight = mostRight.Right
			}
			if mostRight.Right == nil {
				mostRight.Right = cur
				cur = cur.Left
				continue
			} else {
				mostRight.Right = nil
			}
		}
		if  pre >= cur.Val {
			ans = false
		}
		pre = cur.Val
		cur = cur.Right
	}
	return ans
}
```


## 1.2 Morris遍历的应用

在一颗二叉树中，求该二叉树的最小高度。最小高度指的是，所有叶子节点距离头节点的最小值

> 二叉树递归求解，求左树的最小高度加1和右树的最小高度加1，比较

> Morris遍历求解，每到达一个cur的时候，记录高度。每到达一个cur的时候判断cur是否为叶子节点，更新全局最小值。最后看一下最后一个节点的高度和全局最小高度对比，取最小高度

```Go
package main

import "math"

type Node struct {
	Left *Node
	Right *Node
	Val int
}

// 求二叉树最小高度；解法1 运用二叉树的递归
func minHeight1(head *Node) int {
	if head == nil {
		return 0
	}
	return p(head)
}

func p(x *Node) int {
	if x.Left == nil && x.Right == nil {
		return 1
	}

	// 左右子树起码有一个不为空
	leftH := math.MaxInt
	if x.Left != nil {
		leftH = p(x.Left)
	}

	rightH := math.MaxInt
	if x.Right != nil {
		rightH = p(x.Right)
	}

	return 1 + int(math.Min(float64(leftH), float64(rightH)))
}

// 解法2 根据morris遍历改写
func minHeight2(head *Node) int {
	if head == nil {
		return 0
	}

	cur := head
	var mostRight *Node
	curLevel := 0
	minnHeight := math.MaxInt
	for cur != nil {
		mostRight = cur.Left
		if mostRight != nil {
			rightBoardSize := 1
			for mostRight.Right != nil && mostRight.Right != cur {
				rightBoardSize++
				mostRight = mostRight.Right
			}

			if mostRight.Right == nil { // 第一次到达
				curLevel++
				mostRight.Right = cur
				cur = cur.Left
				continue
			} else { // 第二次到达
				if mostRight.Left == nil {
					minnHeight = int(math.Min(float64(minnHeight), float64(minnHeight)))
				}
				curLevel -= rightBoardSize
				mostRight.Right = nil
			}
		} else { // 只有一次到达
			curLevel++
		}
		cur = cur.Right
	}

	finalRight := 1
	cur = head // 回到头结点
	for cur.Right != nil {
		finalRight++
		cur = cur.Right
	}
	if cur.Left == nil && cur.Right == nil {
		minnHeight = int(math.Min(float64(minnHeight), float64(finalRight)))
	}
	return minnHeight
}
```

## 1.3 Morris遍历为最优解的情景

> 如果我们算法流程设计成，每来到一个节点，需要左右子树的信息进行整合，那么无法使用Morris遍历。该种情况的空间复杂度也一定不是O(1)的

> 如果算法流程是，当前节点使用完左树或者右树的信息后，无需整合，那么可以使用Morris遍历

> Morris遍历属于比较复杂的问题
