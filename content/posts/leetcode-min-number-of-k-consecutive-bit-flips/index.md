---
title: "Leetcode 995. Minimum Number of K Consecutive Bit Flips"
date: 2024-06-24T11:17:56+09:00
showToc: false
---

### 995. [Minimum Number of K Consecutive Bit Flips](https://leetcode.com/problems/minimum-number-of-k-consecutive-bit-flips/description/)

---

#### Description

```text
You are given a binary array nums and an integer k.

A k-bit flip is choosing a subarray of length k from nums and simultaneously changing every 0 in the subarray to 1, and every 1 in the subarray to 0.

Return the minimum number of k-bit flips required so that there is no 0 in the array. If it is not possible, return -1.

A subarray is a contiguous part of an array.
```

#### Example
```text
Example 1:
Input: nums = [0,1,0], k = 1
Output: 2
Explanation: Flip nums[0], then flip nums[2].

Example 2:
Input: nums = [1,1,0], k = 2
Output: -1
Explanation: No matter how we flip subarrays of size 2, we cannot make the array become [1,1,1].

Example 3:
Input: nums = [0,0,0,1,0,1,1,0], k = 3
Output: 3
Explanation: 
Flip nums[0],nums[1],nums[2]: nums becomes [1,1,1,1,0,1,1,0]
Flip nums[4],nums[5],nums[6]: nums becomes [1,1,1,1,1,0,0,0]
Flip nums[5],nums[6],nums[7]: nums becomes [1,1,1,1,1,1,1,1]

```

#### Code

``` go
func minKBitFlips(nums []int, k int) int {
    ret := 0
    for i:=0; i <len(nums)-k+1; i++ {
        if nums[i] == 0 {
            ret++
            for j:=0; j<k && i+j < len(nums); j++ {
                nums[i+j] = flip(nums[i+j])
            }

        }
    }

    for i:=0; i<len(nums); i++ {
        if nums[i] == 0 {
            return -1
        }
    }
    return ret
}

func flip(num int) int {
    if num == 0 {
        return 1
    }
    return 0
}
```
=> Time Limit Exceeded


``` go
func minKBitFlips(nums []int, k int) int {
    ret := 0
    curFlipCount := 0
    for i:=0; i <len(nums); i++ {
        if i >= k && nums[i-k]==2 { // i-k 에서 플립했으면 줄여줌
            curFlipCount-- 
        }
        if curFlipCount % 2  == nums[i] { // i-k ~ i-1 까지 플립 몇번했는지와, 지금 value 비교해서 플립이 필요한지 확인
            if i + k > len(nums) {
                return -1
            }
            nums[i] = 2
            curFlipCount++
            ret++
        }
    }

    return ret
}
```
=> Time: O(N), Space: O(1)

처음 제출에서 시간 복잡도 O(N*K) 로 O(N^2) 과 동일하여 시간이 초과됐다.\
실제로 flip 을 수행하지 않고 그것을 세는 방식을 이용하여 해결했다.

curFilpCount 를 nums[i] 에 Filp 이 발생한 횟수라고 하자.\
```if curFlipCount % 2  == nums[i] ``` 에서 i-k ~ i-1 까지 플립한 횟수와 i 번째 값을 비교한다. 플립한 횟수가 홀수이고, nums[i]=1 이면 플립해야 하고, 플립한 횟수가 짝수이고 nums[i]=0 이면 플립해야 한다.