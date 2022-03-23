[TOC]

# 1 KMP算法(面试高频，劝退)

## 1.1 KMP算法分析

查找字符串问题：例如我们有一个字符串str="abc1234efd"和match="1234"。我们如何查找str字符串中是否包含match字符串的子串？


> 暴力解思路：循环str和match，挨个对比，最差情况为O(N*M)。时间复杂度为O(N*M)

> KMP算法，在N大于M时，可以在时间复杂度为O(N)解决此类问题


我们对str记录字符坐标前的前缀后缀最大匹配长度，例如str="abcabck"

1、对于k位置前的字符，前后缀长度取1时，前缀为"a"后缀为"c"不相等

2、对于k位置前的字符，前后缀长度取2时，前缀为"ab"后缀为"bc"不相等

3、对于k位置前的字符，前后缀长度取3时，前缀为"abc"后缀为"abc"相等

4、对于k位置前的字符，前后缀长度取4时，前缀为"abca"后缀为"cabc"不相等

5、对于k位置前的字符，前后缀长度取5时，前缀为"abcab"后缀为"bcabc"不相等

> 注意前后缀长度不可取k位置前的整体长度6。那么此时k位置前的最大匹配长度为3

所以，例如"aaaaaab","b"的坐标为6，那么"b"坐标前的前后缀最大匹配长度为5


我们对match建立坐标前后缀最大匹配长度数组，概念不存在的设置为-1，例如0位置前没有字符串，就为-1，1位置前只有一个字符，前后缀无法取和坐标前字符串相等，规定为0。例如"aabaabc"，nextArr[]为[-1,0,1,0,1,2,3]


> 暴力方法之所以慢，是因为每次比对，如果match的i位置前都和str匹配上了，但是match的i+1位置没匹配成功。那么str会回退到第一次匹配的下一个位置，match直接回退到0位置再次比对。str和match回退的位置太多，之前的信息全部作废，没有记录


> 而KMP算法而言，如果match的i位置前都和str匹配上了，但是match的i+1位置没匹配成功，那么str位置不回跳，match回跳到当前i+1位置的最大前后缀长度的位置上，去和当前str位置比对。

原理是如果我们当前match位置i+1比对失败了，我们跳到最大前后缀长度的下一个位置去和当前位置比对，如果能匹配上，由于i+1位置之前都匹配的上，那么match的最大后缀长度也比对成功，可以被我们利用起来。替换成match的前缀长度上去继续对比，起到加速的效果


那么为什么str和match最后一个不相等的位置，之前的位置无法配出match，可以反证，如果可以配置出来，那么该串的头信息和match的头信息相等，得出存在比match当前不等位置最大前后缀还要大的前后缀，矛盾

Code:

```Go
package main

import "fmt"

// getIndexOf O(N)
func getIndexOf(s string, m string) int {
	if len(s) == 0 || len(m) == 0 || len(s) < len(m) {
		return -1
	}

	str := []byte(s)
	match := []byte(m)
	x := 0 // str中当前比对到的位置
	y := 0 // match中当前比对到的位置
	// match的长度M，M <= N   O(M)
	next := getNextArray(match) // next[i]  match中i之前的字符串match[0..i-1],最长前后缀相等的长度
	// O(N)
	// x在str中不越界，y在match中不越界
	for x < len(str) && y < len(match) {
		// 如果比对成功，x和y共同往各自的下一个位置移动
		if str[x] == match[y] {
			x++
			y++
		} else if next[y] == -1 { // 表示y已经来到了0位置 y == 0
			// str换下一个位置进行比对
			x++
		} else { // y还可以通过最大前后缀长度往前移动
			y = next[y]
		}
	}
	// 1、 x越界，y没有越界，找不到，返回-1
	// 2、 x没越界，y越界，配出
	// 3、 x越界，y越界 ，配出，str的末尾，等于match
	// 只要y越界，就配出了，配出的位置等于str此时所在的位置x，减去y的长度。就是str存在匹配的字符串的开始位置
	if y == len(match) {
		return x - y
	} else {
		return -1
	}
}

// M   O(M)
func getNextArray(match []byte) []int {
	// 如果match只有一个字符，人为规定-1
	if len(match) == 1 {
		return []int{-1}
	}

	// match不止一个字符，人为规定0位置是-1，1位置是0
	next := make([]int, len(match))

	next[0] = -1
	next[1] = 0

	i := 2
	// cn代表，cn位置的字符，是当前和i-1位置比较的字符
	cn := 0
	for i < len(next) {
		if match[i - 1] == match[cn] { // 跳出来的时候
			// next[i] = cn+1
			// i++
			// cn++
			// 等同于
			cn++
			next[i] = cn
			i++
		} else if cn > 0 { // 跳失败，如果cn>0说明可以继续跳
			cn = next[cn]
		} else { // 跳失败，跳到开头仍然不等
			next[i] = 0
			i++
		}
	}
	return next
}

func main() {
	s := "abc1234efd"
	m := "1234"
	fmt.Println(getIndexOf(s, m))
}
```

## 1.2 KMP算法应用

### 题目1：旋转词

例如Str1="123456",对于Str1的旋转词，字符串本身也是其旋转词，Str1="123456"的旋转词为，"123456","234561","345612","456123","561234","612345"。给定Str1和Str2，那么判断这个两个字符串是否互为旋转词？是返回true，不是返回false


暴力解法思路：把str1的所有旋转词都列出来，看str2是否在这些旋转词中。挨个便利str1，循环数组的方式，和str2挨个比对。O(N*N)

KMP解法：str1拼接str1得到str',"123456123456"，我们看str2是否是str'的子串


### 题目2：子树问题

给定两颗二叉树头结点，node1和node2，判断node2为头结点的树，是不是node1的某个子树？



# 2 bfprt算法 (面试常见)

情形：在一个无序数组中，怎么求第k小的数。如果通过排序，那么排序的复杂度为O(n*logn)。问，如何O(N)复杂度解决这个问题？

思路1：我们利用快排的思想，对数组进行荷兰国旗partion过程，每一次partion可以得到随机数m小的区域，等于m的区域，大于m的区域。我们看我们m区域是否包含我们要找的第k小的树，如果没有根据比较，在m左区间或者m右区间继续partion，直到第k小的数在我们的的中间区域。

快排是左右区间都会再进行partion，而该问题只会命中大于区域或小于区域，时间复杂度得到优化。T(n)=T(n/2)+O(n)，时间复杂度为O(N)，由于m随机选，概率收敛为O(N)


思路2：bfprt算法，不使用概率求期望，复杂度仍然严格收敛到O(N)

## 2.1 bfprt算法分析

通过上文，利用荷兰国旗问题的思路为：

1、随机选一个数m

2、进行荷兰国旗，得到小于m区域，等于m区域，大于m区域

3、index命中到等于m区域，返回等于区域的左边界，否则比较，进入小于区域，或者大于区域，只会进入一个区域


bfprt算法，再此基础上唯一的区别是，第一步，如何选择m。快排的思想是随机选择一个

bfprt如何选择m？

- 1、对arr分组，5个一组，所以0到4为一组，5到9为一组，最后不够一组的当成最后一组
- 2、对各个小组进行排序。第一步和第二步进行下来，时间复杂度为O(N)
- 3、把每一小组排序后的中间位置的数拿出来。放入一个数组中m[]。前三步统称为bfprt方法
- 4、对m数组，取中位数，这个数就是我们需要的m

```math
T(N) = T(N/5) + T(?) + O(N)
```

建议画图分析：

T(?)在我们随机选取m的时候，是不确定的，但是在bfprt中，m的左侧范围最多有多少个数，等同于m右侧最少有几个数。

假设我们经过分组拿到的m数组有5个数，中位数是我们的m，在m[]数组中，大于m的有2个，小于m的有2个。对于整的数据规模而言，m[]的规模是n/5。大于m[]中位数的规模为m[]的一半,也就是整体数据规模的n/10。


由于m[]中的每个数都是从小组中选出来的，那么对于整体数据规模而言，大于m的数整体为3n/10（每个n/10规模的数回到自己的小组，大于等于的每小组有3个）


那么最少有3n/10的规模是大于等于m的，那么对于整体数据规模而言最多有7n/10的小于m的。同理最多有7n/10的数据是大于m的

可得：

```math
T(N) = T(N/5) + T(7n/10) + O(N)
```

数学证明，以上公式无法通过master来算复杂度，但是数学证明复杂度严格O(N)，证明略(算法导论第九章第三节)


> bfprt算法在算法上的地位非常高，它发现只要涉及到我们随便定义的一个常数分组，得到一个表达式，最后收敛到O(N)，那么就可以通过O(N)的复杂度测试


```Go
package main

import (
	"container/heap"
	"fmt"
	"math"
	"math/rand"
	"time"
)

type Heap []int

func (h Heap) Less(i, j int) bool {
	return h[i] > h[j] // 大根堆。小根堆实现为： h[i] <= h[j]
}

func (h Heap) Len() int {
	return len(h)
}

func (h Heap) Swap(i, j int) {
	h[i], h[j] = h[j], h[i]
}

func (h *Heap) Push(v interface{}) {
	*h = append(*h, v.(int))
}

func (h *Heap) Pop() interface{} {
	n := len(*h)
	x := (*h)[n-1]
	*h = (*h)[:n-1]
	return x
}

// minKth1 找一个数组中第k小的数。方法1：利用大根堆，时间复杂度O(N*logK)
func minKth1(arr []int, k int) int {
	maxHeap := &Heap{}
	for i := 0; i < k; i++ { // 加入大根堆
		heap.Push(maxHeap, arr[i])
	}
	heap.Init(maxHeap)

	for i := k; i < len(arr); i++ {
		if arr[i] < (*maxHeap)[0] { // arr[i] 小于堆顶元素。堆顶元素就是0位置元素
			// ！！！ 这里一定要使用系统包中的pop和push，然后把实现当前栈接口的结构传入
			heap.Pop(maxHeap)          // 弹出
			heap.Push(maxHeap, arr[i]) // 入堆
		}
	}

	// return maxHeap.peek()
	return (*maxHeap)[0]
}

// minKth2 找一个数组中第k小的数。方法2：利用快排，时间复杂度O(N)
func minKth2(array []int, k int) int {
	arr := copyArr(array)
	return process2(arr, 0, len(arr)-1, k-1)
}

// copyArr 克隆数组，防止快排影响原数组的元素顺序
func copyArr(arr []int) []int {
	ans := make([]int, len(arr))
	for i := 0; i < len(ans); i++ { // 这里copy数组，不可以使用append。
		ans[i] = arr[i]
	}
	return ans
}

// arr 第k小的数: process2(arr, 0, N-1, k-1)
// arr[L..R]  范围上，如果排序的话(不是真的去排序)，找位于index的数
// index [L..R]
// 通过荷兰国旗的优化，概率期望收敛于O(N)
func process2(arr []int, L, R, index int) int {
	if L == R { // L == R ==INDEX
		return arr[L]
	}

	// 不止一个数  L +  [0, R -L]，随机选一个数.
	pivot := arr[L+rand.Intn(R-L)]

	// 返回以pivot为划分值的中间区域的左右边界
	// range[0] range[1]
	//  L   ..... R     pivot
	//  0         1000     70...800
	rg := partition(arr, L, R, pivot)
	// 如果我们第k小的树正好在这个范围内，返回区域的左边界
	if index >= rg[0] && index <= rg[1] {
		return arr[index]
	} else if index < rg[0] { // index比该区域的左边界小，递归左区间
		return process2(arr, L, rg[0]-1, index)
	} else { // index比该区域的右边界大，递归右区间
		return process2(arr, rg[1]+1, R, index)
	}
}

// partition 荷兰国旗partition问题
func partition(arr []int, L, R, pivot int) []int {
	less := L - 1
	more := R + 1
	cur := L
	for cur < more {
		if arr[cur] < pivot {
			less++
			arr[less], arr[cur] = arr[cur], arr[less]
			cur++
		} else if arr[cur] > pivot {
			more--
			arr[cur], arr[more] = arr[more], arr[cur]
		} else {
			cur++
		}
	}
	return []int{less + 1, more - 1}
}

// minKth3 找一个数组中第k小的数。方法3：利用bfprt算法，时间复杂度O(N)
func minKth3(array []int, k int) int {
	arr := copyArr(array)
	return bfprt(arr, 0, len(arr)-1, k-1)
}

// bfprt arr[L..R]  如果排序的话，位于index位置的数，是什么，返回
func bfprt(arr []int, L, R, index int) int {
	if L == R {
		return arr[L]
	}

	// 通过bfprt分组，最终选出m。不同于随机选择m作为划分值
	pivot := medianOfMedians(arr, L, R)
	rg := partition(arr, L, R, pivot)
	if index >= rg[0] && index <= rg[1] {
		return arr[index]
	} else if index < rg[0] {
		return bfprt(arr, L, rg[0]-1, index)
	} else {
		return bfprt(arr, rg[1]+1, R, index)
	}
}

// arr[L...R]  五个数一组
// 每个小组内部排序
// 每个小组中位数拿出来，组成marr
// marr中的中位数，返回
func medianOfMedians(arr []int, L, R int) int {
	size := R - L - L
	// 是否需要补最后一组，例如13，那么需要补最后一组，最后一组为3个数
	offset := -1
	if size%5 == 0 {
		offset = 0
	} else {
		offset = 1
	}

	// 初始化数组
	mArr := make([]int, size/5+offset)
	for team := 0; team < len(mArr); team++ {
		teamFirst := L + team*5
		// L ... L + 4
		// L +5 ... L +9
		// L +10....L+14
		mArr[team] = getMedian(arr, teamFirst, int(math.Min(float64(R), float64(teamFirst+4))))
	}

	// marr中，找到中位数，原问题是arr拿第k小的数，这里是中位数数组拿到中间位置的数（第mArr.length / 2小的数），相同的问题
	// 返回值就是我们需要的划分值m
	// marr(0, marr.len - 1,  mArr.length / 2 )
	return bfprt(mArr, 0, len(mArr)-1, len(mArr)/2)
}

func getMedian(arr []int, L, R int) int {
	insertionSort(arr, L, R)
	return arr[(L+R)/2]
}

// insertionSort 插入排序
func insertionSort(arr []int, L, R int) {
	for i := L + 1; i <= R; i++ {
		for j := i - 1; j >= L && arr[j] > arr[j+1]; j-- {
			arr[j], arr[j+1] = arr[j+1], arr[j]
		}
	}
}

func main() {
	/*
	   rand.Seed:
	       还函数是用来创建随机数的种子,如果不执行该步骤创建的随机数是一样的，因为默认Go会使用一个固定常量值来作为随机种子。

	   time.Now().UnixNano():
	       当前操作系统时间的毫秒值
	*/
	rand.Seed(time.Now().UnixNano())
	arr := []int{3, 4, 6, 1, 77, 35, 26, 83, 56, 37}
	fmt.Println(minKth1(arr, 3))
	fmt.Println(minKth2(arr, 3))
	fmt.Println(minKth3(arr, 3))
}
```

## 2.2 bfprt算法应用

题目：求一个数组中，拿出所有比第k小的数还小的数

可以通过bfprt拿到第k小的数，再对原数组遍历一遍，小于该数的拿出来，不足k位的，补上第k小的数

> 对于这类问题，笔试的时候最好选择随机m，进行partion。而不是选择bfprt。bfprt的常数项高。面试的时候可以选择bfprt算法


