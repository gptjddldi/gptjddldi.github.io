---
title: "Go interface 와 heap"
date: 2024-06-24T14:50:42+09:00
# showToc: false
---

Go 에서 Maxheap 은 heap 패키지를 이용해 구현할 수 있다.\
아래 interface 의 methods 를 구현하여 Custom Heap 을 구현할 수 있다.
```go
// container/heap
type Interface interface {
	sort.Interface
	Push(x any) // add x as element Len()
	Pop() any   // remove and return element Len() - 1.
}

```

그 전에 go 의 interface 에 대해 먼저 알아보자.

---

#### Interface
interface 는 method 의 집합을 정의하는 타입이다.\
다음처럼 정의할 수 있다.

``` go
type Animal interface {
	Speak() string
}
```

자바와 코틀린에선 `implements` 키워드를 통해 interface 를 구현했는데 go 에선 그렇지 않다. \
특정 타입이 interface 의 method 를 모두 구현했다면 해당 타입은 interface 를 구현했다고 본다.\
([덕타이핑](https://ko.wikipedia.org/wiki/덕_타이핑))

``` go
type Dog struct {
}

func (d Dog) Speak() string {
	return "Woof!"
}

```
여기서 Dog struct 는 Animal interface 를 구현한다. Dog 타입의 값은 Animal 타입으로 사용할 수 있게 된다.

``` go

type Animal interface {
	Speak() string
}

type Dog struct {
}

func (d Dog) Speak() string {
	return "Woof!"
}

func MakeAnimalSpeak(a Animal) string {
	return a.Speak()
}

func main() {
	dog := Dog{}

	println(MakeAnimalSpeak(dog))
}
```
이런 식으로 Doc 타입 객체가 Aniaml 타입의 함수를 쓸 수 있게 된다.

---

#### Heap

MaxHeap 을 구현하기 전에 마지막으로 `heap` 패키지의 주석을 읽어보자.
```
Package heap provides heap operations for any type that implements
heap.Interface. A heap is a tree with the property that each node is the
minimum-valued node in its subtree.

The minimum element in the tree is the root, at index 0.

A heap is a common way to implement a priority queue. To build a priority
queue, implement the Heap interface with the (negative) priority as the
ordering for the Less method, so Push adds items while Pop removes the
highest-priority item from the queue. The Examples include such an
implementation; the file example_pq_test.go has the complete source.
```
heap 패키지는 `heap.Interface` 를 구현한 모든 타입에게 heap operation 을 제공한다고 한다.\
파일 내용을 확인해 보니 `Init`,`Push`,`Pop`,`Remove`,`Fix` 연산을 쓸 수 있는 것 같다.

Custom Heap 을 구현하기 위해 아래 `heap` 패키지의 interface 를 구현해야 한다.

``` go
// heap
type Interface interface {
	sort.Interface
	Push(x any) // add x as element Len()
	Pop() any   // remove and return element Len() - 1.
}

// sort
type Interface interface {
	Len() int
	Less(i, j int) bool
	Swap(i, j int)
}
```

총 5개의 method 를 구현하면 된다.\
아래는 maxHeap 을 구현한 코드이다.
``` go
import "container/heap"

type IntHeap []int

func (h IntHeap) Len() int {
	return len(h)
}

func (h IntHeap) Less(i, j int) bool {
	return h[i] > h[j]
}

func (h IntHeap) Swap(i, j int) {
	h[i], h[j] = h[j], h[i]
}

func (h *IntHeap) Push(x any) {
	*h = append(*h, x.(int))
}

func (h *IntHeap) Pop() any {
	old := *h
	n := len(old)
	x := old[n-1]
	*h = old[0 : n-1]
	return x
}

func main() {
	h := &IntHeap{2, 1, 5, 3, 10}
	heap.Init(h)
	for h.Len() > 0 {
		println(heap.Pop(h).(int))
	}
}
```

``` 
// output
10
5
3
2
1
```
---

#### 결론
다른 오픈소스 코드를 볼 때 interface 를 선언만 하고 실제로 구현하는 코드가 없어서 이상하다고 생각했는데, 이번 학습으로 좀 더 잘 이해할 수 있을 것 같다.


---

#### 2025.02.08 추가

Leetcode에 올라온 오늘의 daily challenge 문제를 heap을 사용해 풀 수 있어서 가져왔다. \
[(design-a-number-container-system)](https://leetcode.com/problems/design-a-number-container-system)

- key: index / value: number
- key: number / value: sorted list of index

두 개의 hashmap을 사용해서 문제를 해결할 수 있다. go 언어는 sorted set 자료구조를 제공하지 않기 때문에 직접 구현해야 한다. \
아래는 heap을 이용해 푼 코드이다.

```go
import "container/heap"

type minHeap []int

func (h minHeap) Len() int           { return len(h) }
func (h minHeap) Less(i, j int) bool { return h[i] < h[j] }
func (h minHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }

func (h *minHeap) Push(x interface{}) {
	*h = append(*h, x.(int))
}

func (h *minHeap) Pop() interface{} {
	old := *h
	n := len(old)
	x := old[n-1]
	*h = old[0 : n-1]
	return x
}

type NumberContainers struct {
	mp1 map[int]int      // index => number
	mp2 map[int]*minHeap // number => index
}

func Constructor() NumberContainers {
	return NumberContainers{
		mp1: make(map[int]int),
		mp2: make(map[int]*minHeap),
	}
}

// mp1[index] 에 number 추가
// mp2[number] 에 index 추가 (정렬된 상태)
func (nc *NumberContainers) Change(index int, number int) {
	nc.mp1[index] = number
	if nc.mp2[number] == nil {
		nc.mp2[number] = &minHeap{}
	}
	heap.Push(nc.mp2[number], index)
}

func (nc *NumberContainers) Find(number int) int {
	if nc.mp2[number] == nil {
		return -1
	}
	for nc.mp2[number].Len() > 0 {
		cur := (*nc.mp2[number])[0]
		if nc.mp1[cur] == number {
			return cur
		}
		heap.Pop(nc.mp2[number])
	}
	return -1
}
```