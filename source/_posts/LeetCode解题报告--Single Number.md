title: "LeetCode解题报告--Single Number"
date: 2015-04-24 13:02:02
category: 编程
tags: LeetCode
---

刷LeetCode被很多人誉为秒杀求职编程考核的最为快捷和高效的方式，管他有用没用，刷刷更健康。
<!--more-->

# 题目
Given an array of integers, every element appears twice except for one. Find that single one.
Note:
Your algorithm should have a linear runtime complexity. Could you implement it without using extra memory?
大意就是一个int数组里面有除了一个数出现一次，其余都出现两次，设计一个线性时间复杂度的算法找出这个数字。
<!--more-->

# 分析
实话说，看着就想暴力一下，但是要线性算法，O(n)时间复杂度，这就有难度了。想了很久，其实可以看到出现两次，既然出现两次，如果一起做异或位运算，那剩下的不就是那个只出现一次的数字了？

# Talk is cheap
```cpp
class Solution {
public:
    int singleNumber(int A[], int n) {
        int i;
        int res=A[0];
        for(i=1;i<n;i++)
            res=res^A[i];
        return res;
    }
};
```

# Another idea
Python中有一个字典的数据结构，类似map，如果一次遍历将所有整数放在一个字典中计数，再次遍历直接找到value=1的索引就可以了。一次遍历完成。
```python
def SingleNumber(A):
    dic={}
    for i in A:
        if i not in dic:
            dic[i]=1
        else:
            #dic[i]+=1
            del dic[i] #重复出现就删掉
    return dic.keys()[0]
```

---
**以后刷题报告不再更新，直接在GitHub上代码**
