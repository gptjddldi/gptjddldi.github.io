---
title: "Leetcode 1052. Grumpy Bookstore Owner"
date: 2024-06-23T13:53:43+09:00
showToc: false
---

### 1052. [Grumpy Bookstore Owner](https://leetcode.com/problems/grumpy-bookstore-owner/description/)

---

#### Description
``` text
There is a bookstore owner that has a store open for n minutes. Every minute, some number of customers enter the store. 
You are given an integer array customers of length n where customers[i] is the number of the customer that enters the store at the start of the ith minute and all those customers leave after the end of that minute.

On some minutes, the bookstore owner is grumpy. You are given a binary array grumpy where grumpy[i] is 1 if the bookstore owner is grumpy during the ith minute, and is 0 otherwise.

When the bookstore owner is grumpy, the customers of that minute are not satisfied, otherwise, they are satisfied.

The bookstore owner knows a secret technique to keep themselves not grumpy for minutes consecutive minutes, but can only use it once.

Return the maximum number of customers that can be satisfied throughout the day.
```

#### Example

``` text
Input: customers = [1,0,1,2,1,1,7,5], grumpy = [0,1,0,1,0,1,0,1], minutes = 3
Output: 16
```

#### Code
``` go
func maxSatisfied(customers []int, grumpy []int, minutes int) int {
	ret := 0
	for i := 0; i < len(customers); i++ {
		if grumpy[i] == 0 {
			ret += customers[i]
		}
	}
	l := 0
	for i := 0; i < minutes; i++ {
		if grumpy[i] == 1 {
			ret += customers[i]
		}
	}
	maxi := ret
	for i := minutes; i < len(customers); i++ {
		if grumpy[i] == 1 {
			ret += customers[i]
		}
		if grumpy[l] == 1 {
			ret -= customers[l]
		}
		l++
		if ret > maxi {
			maxi = ret
		}
	}
	return maxi
}
```

두 개의 포인터 l, i 를 두고 그 간격을 minutes 로 두고 풀었다.


> 이번 달 daily challenge 에선 sliding window 를 활용한 문제가 많이 나오는 것 같다.

> 졸업 작품에서 golang를 처음 사용해 보는데, 아직 낯선 부분이 많아서 당분간은 계속 golang 으로 알고리즘을 해결하면서 익숙해질 생각이다.

> 코틀린이나 파이썬과 다르게 자료구조를 직접 구현해햐 하는 경우가 많아서, 비슷한 문제가 나오면 학습이 잘 될 것 같다.