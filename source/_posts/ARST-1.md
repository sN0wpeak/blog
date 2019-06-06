---
title: ARST-1
date: 2019-03-17 11:23:41
tags: ARST
---
## A
``` go

func twoSum(nums []int, target int) []int {
	var m = make(map[int]int)
	m[nums[0]] = 0
	for i := 1; i < len(nums); i++ {
		var c = target - nums[i]
		if n, ok := m[c]; ok {
			return []int{n, i}
		} else {
			m[nums[i]] = i
		}
	}
	return nil
}
```

## R


## T

## S
