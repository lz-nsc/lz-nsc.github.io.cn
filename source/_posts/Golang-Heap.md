---
title: "Go: 关于heap包"
date: 2022-04-15 17:21:52
tags: Golang 
comments: true
---
## 接口
`heap`包中定义了接口`Interface`。这个接口里有来自`sort`包的`Interface`接口，以及`Push`和`Pop`两个方法。

```go
// go1.18.1/src/container/heap/heap.go
package heap

import "sort"

type Interface interface {
	sort.Interface
	Push(x any) // add x as element Len()
	Pop() any   // remove and return element Len() - 1.
}
```

`sort`包中的`Interface`接口有3个方法，分别是：Len，Less 还有 Swap ：

```go
package sort

// An implementation of Interface can be sorted by the routines in this package.
// The methods refer to elements of the underlying collection by integer index.
type Interface interface {
	// Len is the number of elements in the collection.
	Len() int

	// Less reports whether the element with index i
	// must sort before the element with index j.
	Less(i, j int) bool

	// Swap swaps the elements with indexes i and j.
	Swap(i, j int)
}
```
这意味着如果我们要实现这个`heap`包的接口，那么我们定义的结构体需要实现这5个方法：Len，Less， Swap， Push还有Pop。

比如以下这个例子：

```go
type MinHeap []int
func (m MinHeap) Less(i, j int) bool {
	return m[i] <= m[j]
}

func (m MinHeap) Len() int {
	return len(m)
}

func (m MinHeap) Swap(i, j int) {
	m[i], m[j] = m[j], m[i]
}

func (m *MinHeap) Push(x any) {
	*m = append(*m, x.(int))
}
func (m *MinHeap) Pop() any {
	size := len(*m)
	last := (*m)[size-1]
	*m = (*m)[:size-1]
	return last
}
```
上面定义的这个叫做`MinHeap`的结构体实现了`heap`包接口`Interface`的所有方法。

现在，我们已经定义的一个实现了接口的结构体，接下来要怎么使用这个包，以及这个包是怎么工作的呢？

## 函数
在这个包里面定义了5个函数：

### Init

`Init` 函数将会将提供的实现了接口的结构体构建成堆。

根据这个包里的代码，`Init` 会使用自下而上的方法在O(n)的时间复杂度下来将数组（也有可能是其他数据结构体，本文以常见的数组作为例子）构建成堆。

- 它将从最后一个非叶节点开始，如果这个节点不符合这个我们定义的结构体的要求（例子中我们定义了最小堆，则要求父节点要小于其子节点），将这个节点与其最小的子节点交换。在这里它调用了我们定义的给`Minheap`结构体的`Less`和`Swap`方法来比较以及交换两个节点。

- 然后它将会移向下一个非叶节点并且持续交换直到整个结构体变成了符合要求的最小堆。

```go
func Init(h Interface) {
	// heapify
	n := h.Len()
	for i := n/2 - 1; i >= 0; i-- {
		down(h, i, n)
	}
}
func down(h Interface, i0, n int) bool {
	i := i0
	for {
		j1 := 2*i + 1
		if j1 >= n || j1 < 0 { // j1 < 0 after int overflow
			break
		}
		j := j1 // left child
		if j2 := j1 + 1; j2 < n && h.Less(j2, j1) {
			j = j2 // = 2*i + 2  // right child
		}
		if !h.Less(j, i) {
			break
		}
		h.Swap(i, j)
		i = j
	}
	return i > i0
}
```
使用我们先前定义的`MinHeap`结构体：

```go
func main() {
	fmt.Println("[Heap] Start heap tast, creating original array ...")
	mh := &MinHeap{5, 3, 7, 8, 2, 1}
	fmt.Printf("[Heap] Original array :%v\n", *mh)
	fmt.Println("[Heap] Start to build minHeap...")
	heap.Init(mh)
	fmt.Printf("[Heap] MinHeap:%v\n", *mh)
	...
}
```

`Init`函数将会按照以下图示过程构建堆：

![build_heap.png](/img/build_heap.png)

前面我们给出的实例代码的输出将会是：

```bash
$ go run main.go
[Heap] Start heap tast, creating original array ...
[Heap] Original array :[5 3 7 8 2 1]
[Heap] Start to build minHeap...
[Heap] MinHeap:[1 2 5 8 3 7]
...
```

### Pop

`Pop` 函数会将根节点与数组中的最后一个节点交换，并且缩短数组的长度，以在后续的活动中忽略最后一个节点（也就是堆中的最小值）。

然后它会当前的根结点往下移动直到重新构建出符合要求的结构体（最小堆）。

在这之后， 这个函数将会调用我们给这个结构体定义的`Pop`方法来返回结构体中的最后一个节点，也就是最小值：

```go
// Pop removes and returns the minimum element (according to Less) from the heap.
// The complexity is O(log n) where n = h.Len().
// Pop is equivalent to Remove(h, 0).
func Pop(h Interface) any {
	n := h.Len() - 1
	h.Swap(0, n)
	down(h, 0, n)
	return h.Pop()
}
```
依然是使用先前定义的 `MinHeap`结构体：

```go
func main() {
	...
	fmt.Printf("[Heap] MinHeap:%v\n", *mh)
	first := heap.Pop(mh)
	fmt.Printf("[Heap] Pop:%d MinHeap:%v\n", first, *mh)
	...
}
```
`Pop`函数将会对`Minheap`结构体进行如下操作：

![pop_heap.png](/img/pop_heap.png)

示例代码的输出将会是：

```bash
...
[Heap] MinHeap:[1 2 5 8 3 7]
[Heap] Pop:1 MinHeap:[2 3 5 8 7]
...
```

### Push

`Push` 函数会首先调用传入的接口（`MinHeap`）的`Push`方法来将新节点放到数组的最后一个。

然后他将会持续调用接口的`Less`和`Swap`方法将新的节点向上移动，直到新的数组依旧符合`MinHeap`的要求（父节点要小于其子节点）。

```go
// Push pushes the element x onto the heap.
// The complexity is O(log n) where n = h.Len().
func Push(h Interface, x any) {
	h.Push(x)
	up(h, h.Len()-1)
}

func up(h Interface, j int) {
	for {
		i := (j - 1) / 2 // parent
		if i == j || !h.Less(j, i) {
			break
		}
		h.Swap(i, j)
		j = i
	}
}
```

使用我们先前定义的`MinHeap`结构体：

```go
func main() {
	...
	fmt.Printf("[Heap] MinHeap:%v\n", *mh)
	first := heap.Pop(mh)
	fmt.Printf("[Heap] Pop:%d MinHeap:%v\n", first, *mh)
	heap.Push(mh, 4)
	fmt.Printf("[Heap] Push:%d MinHeap:%v\n", 4, *mh)
	...
}
```
`Push`函数将会按照以下图示过程添加新节点：

![push_heap.png](/img/push_heap.png)

以上代码的输出将会是：

```bash
...
[Heap] MinHeap:[1 2 5 8 3 7]
[Heap] Pop:1 MinHeap:[2 3 5 8 7]
[Heap] Push:4 MinHeap:[2 3 4 8 7 5]
...
```

### Fix

`heap`包的`Fix` 函数可在结构中某个节点的值被改变的时候使用。在这种情况下我们无需使用`Init`函数来重新构建整个结构，我们只需要将值改变了的节点向上/下移动到合适的位置，使整个结构再次符合要求。

```go
// Fix re-establishes the heap ordering after the element at index i has changed its value.
// Changing the value of the element at index i and then calling Fix is equivalent to,
// but less expensive than, calling Remove(h, i) followed by a Push of the new value.
// The complexity is O(log n) where n = h.Len().
func Fix(h Interface, i int) {
	if !down(h, i, h.Len()) {
		up(h, i)
	}
}
```
`down`函数会返回一个boolean值来表示在刚刚的操作中目标节点的位置是否发生了移动，如果没有，那么这个节点也许需要向上移动。

例如:

```go
func main() {
	...
	fmt.Printf("[Heap] Changing one node of minHeap %v..\n", *mh)
	(*mh)[2] = 6
	fmt.Printf("[Heap] MinHeap will be fixed:%v\n", *mh)
	heap.Fix(mh, 2)
	fmt.Printf("[Heap] MinHeap after fix:%v\n", *mh)

	fmt.Printf("[Heap] Changing one node of minHeap %v..\n", *mh)
	(*mh)[1] = 1
	heap.Fix(mh, 1)
	fmt.Printf("[Heap] MinHeap after fix:%v\n", *mh)
	...
}
```
`Fix`函数将会按照以下图示过程移动改变了的节点：

![Fix_heap.png](/img/Fix_heap.png)

示例代码的输出将会是：

```bash
[Heap] Changing one node of minHeap [2 3 4 8 7 5]..
[Heap] MinHeap will be fixed:[2 3 6 8 7 5]
[Heap] MinHeap after fix:[2 3 5 8 7 6]
[Heap] Changing one node of minHeap [2 3 5 8 7 6]..
[Heap] MinHeap after fix:[1 2 5 8 7 6]
```

### Remove

`Remove` 函数有一点像是`Pop`和`Fix`函数的结合。
- 它会首先缩短数组的长度来将最后的一个节点排除在未来的所有操作中。
- 然后将目标节点与最后一个节点交换位置。
- 由于目标节点所在的位置的值改变了，这个函数将会像`Fix`函数一样运行从而将整个结构调整到符合条件为止。
- 最后它将会取出最后一个节点，也就是目标节点。

```go
// Remove removes and returns the element at index i from the heap.
// The complexity is O(log n) where n = h.Len().
func Remove(h Interface, i int) any {
	n := h.Len() - 1
	if n != i {
		h.Swap(i, n)
		if !down(h, i, n) {
			up(h, i)
		}
	}
	return h.Pop()
}
```

示例代码：

```go
func main(){
	...
	//Remove
	fmt.Printf("[Heap] Removing one node from minHeap %v..\n", *mh)
	elem := heap.Remove(mh, 1)
	fmt.Printf("[Heap] MinHeap after removing element %d :%v\n", elem, *mh)
	...
}
```

`Remove`函数将会如下图所示来移除节点：

![Remove_heap.png](/img/Remove_heap.png)

示例代码的输出：

```bash
[Heap] Removing one node from minHeap [1 2 5 8 7 6]..
[Heap] MinHeap after removing element 2 :[1 6 5 8 7]
```

以上就是所有的我想要谈论的关于`Heap`包的内容，完整的示例代码储存在这个仓库中：[lz-nsz/go-example/heap](https://github.com/lz-nsc/go-example/tree/main/heap)