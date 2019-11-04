---
layout:     post
title:      "回溯法"
subtitle:   "LeetCode算法"
date:       2019-10-07 12:00:00
author:     "ALID"
header-img: "img/post_bg_ff7.jpg"
catalog: true
tags:
    - LeetCode
    - 算法
    - 回溯法
---

## BSAE

在包含问题的所有解的解空间树中，按照**深度优先搜索的策略**，从根结点出发深度探索解空间树。当探索到某一结点时，要先判断该结点是否包含问题的解，如果包含，就从该结点出发继续探索下去，如果该结点不包含问题的解，则逐层向其祖先结点回溯，即**剪枝**。（其实回溯法就是对隐式图的深度优先搜索算法）

核心：

1. 深度优先遍历

2. 回溯（回到之前经过的每一个节点）

3. 离开条件

## CASE

### 子集问题

回溯法其实就是先进行深度遍历，在走完一条路之后再回到之前的节点，向其他情况进行遍历。例如下图中，先由`空`节点向`1`遍历，当走到`1，2，3`时，再回到`1`节点，选择`1，3`的情况。

![img](/img/in-post/post-backtracking/backtracking1.png)

要怎样实现呢，一般使用**循环+递归**的方式进行处理，先上代码。

```java
public List<List<Integer>> subsets(int[] nums) {
    List<List<Integer>> result = new ArrayList<>();
    find(nums, 0, new ArrayList<>(), result);
    return result;
}

private void find(int[] nums, int index, List<Integer> tmp, List<List<Integer>> result) {
    result.add(new ArrayList<>(tmp));
    for (int i = index; i < nums.length; i++) {
        tmp.add(nums[i]);
        find(nums, i + 1, tmp, result); // 递归
        tmp.remove(tmp.size() - 1);
    }
}
```

可以看到在find方法中，使用了一个循环并且在其中还有递归的处理。循环是为了进行不同情况的遍历（广度遍历），而递归是为了对一种情况进行深度遍历。如下图中

**递归：** [] -> [1] -> [1,2] -> [1,2,3]

**循环：** 1 -> 2 -> 3

![img](/img/in-post/post-backtracking/backtracking2.jpg)

#### 其他解法

对应子集问题我们还可以使用**迭代法**求解。如下图思路就是从空集开始考虑，进而考虑只有一个元素的集合的子集问题，再加到2个元素的集合，这样一步步增加元素迭代求解。

![img](/img/in-post/post-backtracking/backtracking3.png)

可以看到每次就是对之前的结果中每一个情况加上新的情况，并且保留之前所有的情况。

#### 变形题

如果给定的集合是`[1,2,2]`有重复元素的情况，就需要做去重处理。我们知道最简单的去重方法就是使用`set`存储结果，但是这样再输出结果的时候需要转换成`List`，我们也可以通过排序后，加以调节限制的方式进行处理。

```java
public List<List<Integer>> subsetsWithDup(int[] nums) {
    List<List<Integer>> ans = new ArrayList<>();
    Arrays.sort(nums); //排序
    getAns(nums, 0, new ArrayList<>(), ans);
    return ans;
}

private void getAns(int[] nums, int start, ArrayList<Integer> temp, List<List<Integer>> ans) {
    ans.add(new ArrayList<>(temp));
    for (int i = start; i < nums.length; i++) {
        //和上个数字相等就跳过
        if (i > start && nums[i] == nums[i - 1]) {
            continue;
        }
        temp.add(nums[i]);
        getAns(nums, i + 1, temp, ans);
        temp.remove(temp.size() - 1);
    }
}
```

这里所使用的方法是在回溯法中经常会用到的一种去重方式。再回溯后尝试新情况的时候，如果发现这次尝试的情况和前一种情况一样则不再进行尝试。

![img](/img/in-post/post-backtracking/backtracking4.png)

如上图中，到第二次出现`[1,2]`的时候，可以判断得到`nums[1] == nums[2]`，即现在判断的情况和上一次判断的情况完全一样，则不再重复判断。

### 全排列问题

全排列问题要比子集问题复杂一些，对应子集问题的限制条件(离开递归的条件)非常简单，在循环中设置`i < nums.length`就可以了。而对应全排列问题，就需要单独设置离开条件。

```java
public List<List<Integer>> permute(int[] nums) {
    List<List<Integer>> result = new ArrayList<>();
    backTrackingWithRemove(nums, result, new ArrayList<>());
    return result;
}


private void backTrackingWithRemove(int[] nums, List<List<Integer>> result, List<Integer> tmp) {
    if (tmp.size() == nums.length) {
        result.add(new ArrayList<>(tmp));
        return;
    }
    for (int i = 0; i < nums.length; i++) {
        if (tmp.contains(nums[i])) {
            continue;
        }
        tmp.add(nums[i]);
        backTrackingWithRemove(nums, result, tmp);
        tmp.remove(tmp.size() - 1);
    }
}
```

上述代码，大致一看其实和子集问题的代码好像差别不大，也是**循环+递归**的方式。但是可以看到，有一处`return`和一处`continue`，2处限制条件。第一处很好理解，当我们拿到一个结果后就可以直接结束。而第二处限制条件我们可以看一下寻找全排列的过程。以下为`tmp`中储存的所有中间结果。

```
中间结果: []
add: 1  中间结果: [1]
跳过: 1  add: 2  中间结果: [1, 2]
跳过: 1  跳过: 2  add: 3  中间结果: [1, 2, 3]
remove: 3  remove: 2  add: 3  中间结果: [1, 3]
跳过: 1  add: 2  中间结果: [1, 3, 2] // case
remove: 2  跳过: 3  remove: 3  remove: 1  add: 2  中间结果: [2]
add: 1  中间结果: [2, 1]
跳过: 1  跳过: 2  add: 3  中间结果: [2, 1, 3]
remove: 3  remove: 1  跳过: 2  add: 3  中间结果: [2, 3]
add: 1  中间结果: [2, 3, 1]
remove: 1  跳过: 2  跳过: 3  remove: 3  remove: 2  add: 3  中间结果: [3]
add: 1  中间结果: [3, 1]
跳过: 1  add: 2  中间结果: [3, 1, 2]
remove: 2  跳过: 3  remove: 1  add: 2  中间结果: [3, 2]
add: 1  中间结果: [3, 2, 1]
```

在代码中为了找到所有的情况，注意循环是从0开始的，也就是每次循环都会去尝试所有的可能。例如当tmp = `[1, 3]`的时候，开始从0循环，因为已经有第0个元素也就是1了，我们不能再向数组中放一次1，所以当判断数组中有1的时候就结束这次循环。提现在上图中case那一行的跳过 1，add 2 进而得到[1, 3, 2]

#### 其他解法

对应全排列问题还可以换个思路求解，虽然也是回溯法，但可以通过**交换元素进行回溯**。

> **思路：**
> 
> 遍历:1-2互换    [1, 3, 2]
>
> 回溯:1-2换回来 [1, 2, 3]
>
> 遍历:0-1互换    [2, 1, 3]
>
> 遍历:1-2互换    [2, 3, 1]
>
> 回溯:1-2换回来 [2, 1, 3]
>
> 回溯:0-1换回来 [1, 2, 3]
>
> 遍历:0-2互换   [3, 2, 1]
>
> 遍历:1-2互换    [3, 1, 2]
>
> 回溯:1-2换回来 [3, 2, 1]
>
> 回溯:0-2换回来 [1, 2, 3]

所以我需要一条遍历的语句，以及一条互换的语句。因为是要换回来所以语句是一样的。

```Java
Collections.swap(nums, index, i);
```

加上我们之前总结的，循环是为了广度遍历，而递归是为了深度遍历。所以在递归前交换，得到新结果，而在递归后交换回来（回溯），进而可以进行广度遍历。

```java
private List<List<Integer>> permute(int[] nums) {
    List<List<Integer>> result = new ArrayList<>();
    List<Integer> numList = new ArrayList<>();
    for (int num : nums) {
        numList.add(num);
    }
    int len = numList.size();
    backTracking(0, len, result, numList);
    return result;
}

private void backTracking(int index, int len, List<List<Integer>> result, List<Integer> numList) {
    if (index == len) {
        result.add(new ArrayList<>(numList));
    }
    for (int i = index; i < len; i++) {
        if (index != i) {
            Collections.swap(numList, index, i);
        }
        backTracking(index + 1, len, result, numList);
        if (index != i) {
            Collections.swap(numList, index, i); // 回溯
        }
    }
}
```

### 组合总和问题

给定一个数组 `candidates` 和一个目标数 `target` ，找出 `candidates` 中所有可以使数字和为 `target` 的组合。

```
输入: candidates = [10,1,2,7,6,1,5], target = 8,
所求解集为:
[
 [1, 7],
 [1, 2, 5],
 [2, 6],
 [1, 1, 6]
]
```

做到这里就很简单了，最简单的思路就是得到全排列(有重复元素的情况)，之后拿到和为给定值的集合就可以了。唯一不一样的点就是增加限制条件在一次深度遍历中拿到一个解就可以结束此次深度遍历。

```java
public List<List<Integer>> combinationSum2(int[] candidates, int target) {
    List<List<Integer>> result = new ArrayList<>();
    Arrays.sort(candidates);
    backTracking(0, candidates, target, result, new ArrayList<>());
    return result;
}

private void backTracking(int index, int[] candidates, int target, List<List<Integer>> result, List<Integer> tmp) {
    if (target == 0) {
        result.add(new ArrayList<>(tmp));
        return;
    }
    for (int i = index; i < candidates.length; i++) {
        if (i > index && candidates[i] == candidates[i - 1]) {
            continue;
        }
        tmp.add(candidates[i]);
        backTracking(i + 1, candidates, target - candidates[i], result, tmp);
        tmp.remove(tmp.size() - 1);
    }
}
```

> **以上就是回溯法的基础问题, 准备好解决一些更难的问题了吗?**

### 正则表达式匹配

给出一个字符串`s`和一个字符规律`p`，请你来实现一个支持 '.' 和 '*' 的正则表达式匹配。

'.' 匹配任意单个字符
'*' 匹配零个或多个前面的那一个元素
所谓匹配，是要涵盖 整个 字符串 s的，而不是部分字符串。

说明:

s 可能为空，且只包含从 a-z 的小写字母。
p 可能为空，且只包含从 a-z 的小写字母，以及字符 . 和 *。

```
输入:
s = "ab"
p = ".*"
输出: true
解释: ".*" 表示可匹配零个或多个（'*'）任意字符（'.'）。
```

前面几道题因为是同一类型的题目,可以看出代码结构都很类似. 而这道题和前面几道题的方法就有一些差别了.

这道题难点在于`*`字符, 因为其可以匹配零到多个字符. 也就是说匹配几个字符都有可能. 这样就需要**尝试匹配0到多次**, 很明显出现了需要**深度搜索**的情况, 而如果深度搜索没有拿到解, 就需要**回溯**继续寻找其他解法. 而这正好是回溯算法的核心思想.

```java
public boolean isMatch(String s, String p) {
    if (p.isEmpty()) return s.isEmpty(); // 空集处理
    boolean singleMatch = !s.isEmpty() && (s.charAt(0) == p.charAt(0) || p.charAt(0) == '.');

    if (p.length() >= 2 && p.charAt(1) == '*') {
        // 如果下一位是*则 需要考虑2种情况 这一位是否进行匹配
        boolean notMatch = isMatch(s, p.substring(2)); // 当前不匹配
        boolean oneMatch = singleMatch && isMatch(s.substring(1), p);// 匹配一次
        return notMatch || oneMatch;
    } else {
        // 如果不是* 且当前字符可以匹配 则同时向后移动一位
        return singleMatch && isMatch(s.substring(1), p.substring(1));
    }
}
```

通过上面的代码可以看到, 在出现*的时候尝试2种情况. 本次匹配一个字符, 或不匹配. 并进行递归调用, 以拿到深度遍历的最终解.