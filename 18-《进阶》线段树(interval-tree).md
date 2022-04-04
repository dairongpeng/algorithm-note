[TOC]

# 1 线段树（又名为线段修改树）

线段树所要解决的问题是，区间的修改，查询和更新，如何更新查询的更快？

线段树结构提供三个主要的方法, 假设大小为N的数组，以下三个方法，均要达到O(logN) ：

```Go
type SegmentTreeInterface interface {
	// Add L到R范围的数，每个数加上V
	// Add(L, R, V int, arr []int)
	Add(L, R, C, l, r, rt int)

	// Update L到R范围的数，每个数都更新成V
	// Update(L, R, V int, arr []int)
	Update(L, R, C, l, r, rt int)

	// GetSum L到R范围的数，累加和返回
	// GetSum(L, R int, arr []int)
	GetSum(L, R, l, r, rt int) int
}
```


## 1.1 线段树概念建立

### 1.1.1 累加和数组建立

1、对于大小为n的数组，我们二分它，每次二分我们都记录一个信息

2、对于每次二分，成立树结构，我们想拿任何区间的信息，可以由我们的二分结构组合得到。例如我们1到8的数组，可以二分得到的信息为：

```
graph TD
'1-8'-->'1-4'
'1-8'-->'5-8'
'1-4'-->'1-2'
'1-4'-->'3-4'
'5-8'-->'5-6'
'5-8'-->'7-8'
'1-2'-->'1'
'1-2'-->'2'
'3-4'-->'3'
'3-4'-->'4'
'5-6'-->'5'
'5-6'-->'6'
'7-8'-->'7'
'7-8'-->'8'
```

每一个节点的信息，可以由该节点左右孩子信息得到，最下层信息就是自己的信息。由以上的规则，对于N个数，我们需要申请2N-1个空间用来保存节点信息。如果N并非等于2的某次方，我们把N补成2的某次方的长度，用来保证我们构建出来的信息数是满二叉树。例如我们的长度是6，我们补到8个，后两个位置值为0。


对于任意的N,我们需要准备多少空间，可以把N补成2的某次方，得到的二分信息都装下？答案是4N。4N虽然有可能多分空间，但是多余的空间都是0，并无影响，而且兼容N为任意值的情况

例如四个数长度的数组arr[4]{3,2,5,7}，我们得到累加和的二分信息为如下的树：

```
graph TD
'1到4=17'-->'1到2=5'
'1到4=17'-->'3到4=12'
'1到2=5'-->'3'
'1到2=5'-->'2'
'3到4=12'-->'5'
'3到4=12'-->'7'
```

我们申请4N的空间，即16，arr[16]。0位置不用。arr[1]=17，arr[2]=5,arr[3]=12,arr[4]=3,arr[5]=2,arr[6]=5,arr[7]=7。剩下位置都为0。任何一个节点左孩子下标为2i，右孩子下标为2i+1


得到累加和信息的分布树的大小，和值的情况，那么update更新树，和add累加树，同样的大小和同样的坐标关系构建。


### 1.1.2更新结构数组建立

懒更新概念，例如有8个数，我们要把1到6的数都减小2。那么先看1到6是否完全囊括8个数，如果囊括直接更新。很显然这里没有囊括，记录要更新1到6，下发该任务给1到4和5到8。1到6完全囊括1到4，记录到lazy中，不再下发；5到8没有囊括1到6，继续下发给5到6和7到8，5到6被囊括，记录到lazy不再继续下发，7到8不接受该任务

这种懒更新机制的时间复杂度为O(logN)，由于一个区间经过左右子树下发，只会经过一个绝对路径到叶子节点，其他节点都会被懒住。如果某个节点有新的任务进来，会把之前懒住的信息下发给左右孩子


对于update操作，如果update操作经过的信息节点上存在懒任务，那么该次update操作会取消该节点的lazy，无需下发，因为下发了也会给update覆盖掉；


```Go
package main

import "fmt"

type SegmentTreeInterface interface {
	// Add L到R范围的数，每个数加上V
	// Add(L, R, V int, arr []int)
	Add(L, R, C, l, r, rt int)

	// Update L到R范围的数，每个数都更新成V
	// Update(L, R, V int, arr []int)
	Update(L, R, C, l, r, rt int)

	// GetSum L到R范围的数，累加和返回
	// GetSum(L, R int, arr []int)
	GetSum(L, R, l, r, rt int) int
}

// SegmentTree 线段树
type SegmentTree struct {
	// arr[]为原序列的信息从0开始，但在arr里是从1开始的
	// sum[]模拟线段树维护区间和
	// lazy[]为累加懒惰标记
	// change[]为更新的值
	// update[]为更新慵懒标记
	maxN int
	arr []int
	// 4*len(arr)
	sum []int
	// 4*len(arr)
	lazy []int
	// 4*len(arr)
	change []int
	// 4*len(arr)
	update []bool
}

// InitSegmentTree 初始化一个线段树。根据int[] origin来初始化线段树结构
func InitSegmentTree(origin []int) *SegmentTree {
	sgTree := SegmentTree{}
	MaxN := len(origin) + 1
	arr := make([]int, MaxN) // arr[0] 不用  从1开始使用
	for i := 1; i < MaxN; i++ {
		arr[i] = origin[i - 1]
	}

	// sum数组开辟的大小是原始数组的4倍
	sum := make([]int, MaxN << 2) // 用来支持脑补概念中，某一个范围的累加和信息
	lazy := make([]int, MaxN << 2) // 用来支持脑补概念中，某一个范围沒有往下傳遞的纍加任務
	change := make([]int, MaxN << 2) // 用来支持脑补概念中，某一个范围有没有更新操作的任务
	update := make([]bool, MaxN << 2) // 用来支持脑补概念中，某一个范围更新任务，更新成了什么

	sgTree.maxN = MaxN
	sgTree.arr = arr
	sgTree.sum = sum
	sgTree.lazy = lazy
	sgTree.change = change
	sgTree.update = update
	return &sgTree
}

// PushUp 汇总线段树当前位置rt的信息，为左孩子信息加上右孩子信息
func (sgTree *SegmentTree) PushUp(rt int) {
	sgTree.sum[rt] = sgTree.sum[rt << 1] + sgTree.sum[rt << 1 | 1]
}

// PushDown 线段树之前的，所有懒增加，和懒更新，从父范围，发给左右两个子范围
// 分发策略是什么
// ln表示左子树元素结点个数，rn表示右子树结点个数
func  (sgTree *SegmentTree) PushDown(rt, ln, rn int) {
	// 首先检查父亲范围上有没有懒更新操作
	if sgTree.update[rt] {
		// 父范围有懒更新操作，左右子范围就有懒更新操作
		sgTree.update[rt << 1] = true
		sgTree.update[rt << 1 | 1] = true
		// 左右子范围的change以父亲分发的为准
		sgTree.change[rt << 1] = sgTree.change[rt]
		sgTree.change[rt << 1 | 1] = sgTree.change[rt]
		// 左右子范围的懒任务全部清空
		sgTree.lazy[rt << 1] = 0
		sgTree.lazy[rt << 1 | 1] = 0
		// 左右子范围的累加和全部变为当前父节点下发的change乘以左右孩子的范围个数
		sgTree.sum[rt << 1] = sgTree.change[rt] * ln
		sgTree.sum[rt << 1 | 1] = sgTree.change[rt] * rn
		// 父范围的更新任务被分发到左右子范围，当前父范围的更新任务改为false
		sgTree.update[rt] = false
	}

	// 如果上面的if也进入，该if也进入，表示之前的最晚懒住的更新到现在还没有发生过新的更新使之下发，却来了个add任务
	// 所以该节点即懒住了更新任务，又懒住一个add任务，接着又来了一个update任务，所以更新要先下发到子范围，接着要把当前的add任务下发下去
	// 如果当前节点的懒信息不为空。
	if sgTree.lazy[rt] != 0 {
		// 下发给左孩子
		sgTree.lazy[rt << 1] += sgTree.lazy[rt]
		sgTree.sum[rt << 1] += sgTree.lazy[rt] * ln
		// 下发给右孩子
		sgTree.lazy[rt << 1 | 1] += sgTree.lazy[rt]
		sgTree.sum[rt << 1 | 1] += sgTree.lazy[rt] * rn
		// 清空当前节点的懒任务信息
		sgTree.lazy[rt] = 0
	}
}

// Build 在初始化阶段，先把sum数组，填好
// 在arr[l~r]范围上，去build，1~N，
// rt :  这个范围在sum中的下标
func  (sgTree *SegmentTree) Build(l, r, rt int) {
	if l == r {
		sgTree.sum[rt] = sgTree.arr[l]
		return
	}
	// 得到l到r的中间位置
	mid := (l + r) >> 1
	// l到r左侧，填充到sum数组rt下标的2倍的位置，因为在数组中当前节点和左孩子的关系得到
	// 递归rt左区间
	sgTree.Build(l, mid, rt << 1)
	// 右侧，填充到2*rt+1的位置
	// 递归rt右区间
	sgTree.Build(mid + 1, r, rt << 1 | 1)
	sgTree.PushUp(rt)
}

// Update 线段树更新操作
func  (sgTree *SegmentTree) Update(L, R, C, l, r, rt int) {
	// 如果更新任务彻底覆盖当前边界
	if L <= l && r <= R {
		// 当前位置的update标记为true
		sgTree.update[rt] = true
		// 当前位置需要改变为C, update和change搭配使用
		sgTree.change[rt] = C
		// 当前节点的累加和信息，被C * (r - l + 1)覆盖掉
		sgTree.sum[rt] = C * (r -l + 1)
		// 清空之前存在该节点的懒任务
		sgTree.lazy[rt] = 0
		return
	}
	// 当前任务躲不掉，无法懒更新，要往下发
	mid := (l + r) >> 1
	// 之前的，所有懒更新，从父范围，发给左右两个子范围
	sgTree.PushDown(rt, mid - l + 1, r - mid)
	// 更新任务发给左孩子
	if L <= mid {
		sgTree.Update(L, R, C, l, mid, rt << 1)
	}
	// 更新任务发给右孩子
	if R > mid {
		sgTree.Update(L, R, C, mid + 1, r, rt << 1 | 1)
	}

	sgTree.PushUp(rt)
}

// Add 线段树加值操作
// L..R -> 任务范围 ,所有的值累加上C
// l,r -> 表达的范围
// rt  去哪找l，r范围上的信息
func  (sgTree *SegmentTree) Add(L, R, C, l, r, rt int) {
	// 任务的范围彻底覆盖了，当前表达的范围，懒住
	if L <= l && r <= R {
		// 当前位置的累加和加上C * (r - l + 1),等同于下边节点都加上C,由于被懒住，下面节点并没有真正意思上add一个C
		sgTree.sum[rt] += C * (r - l + 1)
		// 之前懒住的信息，例如之前该节点加上3，又来一个加上7的任务，那么此时lazt[rt]==10
		sgTree.lazy[rt] += C
		return
	}

	// 任务并没有把l...r全包住
	// 要把当前任务往下发
	// 任务  L, R  没有把本身表达范围 l,r 彻底包住
	mid := (l + r) >> 1 // l..mid  (rt << 1)   mid+1...r(rt << 1 | 1)
	// 下发之前该节点所有攒的懒任务到孩子节点
	sgTree.PushDown(rt, mid - l + 1, r - mid)
	// 左孩子是否需要接到任务
	if L <= mid {
		sgTree.Add(L, R, C, l, mid, rt << 1)
	}

	// 右孩子是否需要接到任务
	if R > mid {
		sgTree.Add(L, R, C, mid + 1, r, rt << 1 | 1)
	}
	// 左右孩子做完任务后，我更新我的sum信息
	sgTree.PushUp(rt)
}

// GetSum 1~6 累加和是多少？ 1~8   rt
func  (sgTree *SegmentTree) GetSum(L, R, l, r, rt int) int {
	// 累加任务覆盖当前节点范围，返回当前节点范围的累加和
	if L <= l && r <= R {
		return sgTree.sum[rt]
	}

	// 没覆盖当前节点的范围，汇总左右子范围的累加和，汇总给到当前节点
	mid := (l + r) >> 1
	sgTree.PushDown(rt, mid - l + 1, r - mid)
	ans := 0
	if L <= mid {
		ans += sgTree.GetSum(L, R, l, mid, rt << 1)
	}
	if R > mid {
		ans += sgTree.GetSum(L, R, mid + 1, r, rt << 1 | 1)
	}
	return ans
}


// ---------- //
// 线段树暴力解，用来做对数器
// sgTree 模拟线段树结构
type sgTree []int

// BuildTestTree  构建测试线段树
func BuildTestTree(origin []int) *sgTree {
	arr := make([]int, len(origin) + 1)
	// 做一层拷贝，arr[0]位置废弃不用，下标从1开始
	for i := 0; i < len(origin); i++ {
		arr[i + 1] = origin[i]
	}
	sg := sgTree{}
	sg = arr
	return &sg
}

func (sgt *sgTree) Update(L, R, C int) {
	for i := L; i <= R; i++ {
		(*sgt)[i] = C
	}
}

func (sgt *sgTree) Add(L, R, C int) {
	for i := L; i <= R; i++ {
		(*sgt)[i] += C
	}
}

func (sgt *sgTree) GetSum(L, R int) int {
	ans := 0
	for i := L; i <= R; i++ {
		ans += (*sgt)[i]
	}
	return ans
}

func main() {
	origin := []int{2, 1, 1, 2, 3, 4, 5}
	// 构建一个线段树
	sg := InitSegmentTree(origin)
	sgTest := BuildTestTree(origin)


	S := 1 // 整个区间的开始位置，规定从1开始，不从0开始 -> 固定
	N := len(origin) // 整个区间的结束位置，规定能到N，不是N-1 -> 固定
	root := 1 // 整棵树的头节点位置，规定是1，不是0 -> 固定
	L := 2 // 操作区间的开始位置 -> 可变
	R := 5 // 操作区间的结束位置 -> 可变
	C := 4 // 要加的数字或者要更新的数字 -> 可变
	// 区间生成，必须在[S,N]整个范围上build
	sg.Build(S, N , root)
	// 区间修改，可以改变L、R和C的值，其他值不可改变
	sg.Add(L, R, C, S, N, root)
	// 区间更新，可以改变L、R和C的值，其他值不可改变
	sg.Update(L, R, C, S, N ,root)
	// 区间查询，可以改变L和R的值，其他值不可改变
	sum := sg.GetSum(L, R, S, N, root)
	fmt.Println(fmt.Sprintf("segmentTree: %d", sum))

	sgTest.Add(L, R, C)
	sgTest.Update(L, R, C)
	testSum := sgTest.GetSum(L, R)
	fmt.Println(fmt.Sprintf("segmentTreeTest: %d", testSum))
}
```

## 1.2 线段树案例实战

想象一下标准的俄罗斯方块游戏，X轴是积木最终下落到底的轴线
下面是这个游戏的简化版：

1）只会下落正方形积木

2）[a,b] -> 代表一个边长为b的正方形积木，积木左边缘沿着X = a这条线从上方掉落

3）认为整个X轴都可能接住积木，也就是说简化版游戏是没有整体的左右边界的

4）没有整体的左右边界，所以简化版游戏不会消除积木，因为不会有哪一层被填满。

给定一个N*2的二维数组matrix，可以代表N个积木依次掉落，
返回每一次掉落之后的最大高度

> 线段树原结构，是收集范围累加和，本题是范围上收集最大高度当成收集的信息

```Go
package main

import "math"

// fallingSquares
func fallingSquares(positions [][]int) []int {
	m := index(positions)
	// 100   -> 1    306 ->   2   403 -> 3
	// [100,403]   1~3
	N := len(m) // 1 ~ 	N
	var res []int
	origin := make([]int, N)
	max := 0
	sgTree := InitSegmentTree(origin)
	// 每落一个正方形，收集一下，所有东西组成的图像，最高高度是什么
	for _, arr := range positions {
		L := m[arr[0]]
		R := m[arr[0] + arr[1] - 1]
		height := sgTree.GetSum(L, R, 1, N, 1) + arr[1]
		max = int(math.Max(float64(max), float64(height)))
		res = append(res, max)
		sgTree.Update(L, R, height, 1, N, 1)
	}
	return res
}

// positions
// [2,7] -> 表示位置从2开始，边长为7的方块，落下的x轴范围为2到8，不包括9是因为下一个位置为9可以落得下来； 2 , 8
// [3, 10] -> 3, 12
//
// 用treeSet做离散化，避免多申请空间
func index(positions [][]int) map[int]int {
	pos := make(map[int]string, 0)
	for _, arr := range positions {
		pos[arr[0]] = ""
		pos[arr[0] + arr[1] - 1] = ""
	}

	m := make(map[int]int, 0)
	count := 0
	for key := range pos {
		count++
		m[key] = count
	}
	return m
}
```

本题为leetCode原题：https://leetcode.com/problems/falling-squares/


## 1.3 什么样的题目可以用线段树来解决？

区间范围上，统一增加，或者统一更新一个值。大范围信息可以只由左、右两侧信息加工出，
而不必遍历左右两个子范围的具体状况

