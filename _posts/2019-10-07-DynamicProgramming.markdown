---
layout:     post
title:      "动态规划"
subtitle:   "LeetCode算法"
date:       2019-11-07 12:00:00
author:     "ALID"
header-img: "img/post_bg_dp.jpg"
catalog: true
tags:
    - LeetCode
    - 算法
    - 动态规划
---


## BASE

个人认为是基础算法中最难的部分, 常常被用来求解最优问题.

第一个疑问肯定是**什么时候用**动态规划, 个人认为是当我们会遇到**很多选择**，并且**之前的选择会对之后的选择产生影响**的时候.

无论是背包问题中选择物品, 还是在地下城问题中选择路线, 还是正则表达式匹配中选择当前是否再匹配一次*. 这些问题中都有大量的选择

而且之前的选择会对之后产生影响. 例如之前选择的物品会占用背包的体积影响之后能否放下新的物品; 地下城问题中之前的选择会决定所需的血量; 正则匹配中只有之前可以全部正确的匹配的选择才能继续匹配.

动态规划就是把这些**选择都记录下来**, 再依据之前的选择推出后面的选择, 这就是动态规划的核心: **用小问题推导出最终问题**

第二个疑问是动态规划**怎么做**, 这里当然有标准解答

1.  状态: 记录的到底是什么, 也就是说选择带来的结果
    
2.  转移方程: 怎样可以记录当前的状态
    
3.  特殊情况: 在最初或者是最后常常都有特殊情况出现
    
## CASE

通过这个三点就可以去每次选择最优子结构, 进而得到最后的最优解

### 01背包问题

对于一组不同重量、不可分割的物品，我们需要选择一些装入背包，在满足背包最大重量限制的前提下，背包中物品总重量的最大值是多少呢？

我们假设背包的最大承载重量是 9。我们有 5 个不同的物品，每个物品的重量分别是 2，2，4，6，3。如果我们把这个例子的回溯求解过程，用递归树画出来，就是下面这个样子：

![](https://static001.geekbang.org/resource/image/42/ea/42ca6cec4ad034fc3e5c0605fbacecea.jpg)

我们把整个求解过程分为 n 个阶段，每个阶段会决策一个物品是否放到背包中。每个物品决策（放入或者不放入背包）完之后，背包中的物品的重量会有多种情况，也就是说，会达到多种不同的**状态**，对应到递归树中，就是有很多不同的节点。

我们把每一层重复的状态（节点）合并，只记录不同的状态，然后**基于上一层的状态**集合，来**推导下一层的状态**集合。我们可以通过合并每一层重复的状态，这样就保证每一层不同状态的个数都不会超过 w 个（w 表示背包的承载重量）

我们用一个二维数组 states[n][w+1]，来记录每层可以达到的不同状态。这就是第一个核心点 `记录状态`

### 一维问题

一维问题是最基础的动态规划问题, 只需要一层循环就可以解决问题. 例如最爬楼梯问题. 但也有稍微复杂的情况, 比如下面的最长有效括号问题.

#### 最长有效括号问题

当遇到`)`时构建状态转移方程进行求解`f(i)=max[之前最长，这次的长度]`

**状态**: dp[i] 当前位置为结算的最长有效括号长度. 这个有点绕, 其实就是每次能组成有效括号的时候记录一下当前的有效括号长度, 而如果没记录则为0.

**转移方程**: 这里有2中情况, 核心是依据状态在当前是`)`的时候进行计算

1.  `···()`的情况, 这种情况`i-1`就是`(`也就是说直接在`i-2`位置记录长度的基础上`+2`, 当然`i-2`的位置根本不是有效括号
    
    ```java
    dp[i] = dp[i - 2] + 2
    ```
    
2.  `···(···)`的情况, 这里要求括号中的`···`都是有效括号. 这里要计入两种有效括号长度. 一个是和刚刚一样的, 这次有效括号前面的记录长度`dp[i - dp[i - 1] - 2]`. 另一部分则是该次有效括号总计的匹配长度`dp[i - 1]`. 这两个记录再加上2就是本次的长度了
    
    ```java
    dp[i] = dp[i - 1] + dp[i - dp[i - 1] - 2] + 2
    ```
    

是不是很复杂, 这道题就是需要考虑的情况非常多. 而且还没有结束, 还需要定义特殊情况

**特殊情况**: 这里的特殊情况其实很简单, 只要考虑数组越界就好了, 也就是如果之前的`···`根本不存在肯定要特殊处理一下.

```java
public int longestValidParentheses(String s) {
    int maxans = 0;
    int dp[] = new int[s.length()];
    for (int i = 1; i < s.length(); i++) {
        if (s.charAt(i) == ')') {
            if (s.charAt(i - 1) == '(') {
                dp[i] = (i >= 2 ? dp[i - 2] : 0) + 2;
            } else if (i - dp[i - 1] > 0 && s.charAt(i - dp[i - 1] - 1) == '(') {
                dp[i] = dp[i - 1] + 
                  ((i - dp[i - 1]) >= 2 ? dp[i - dp[i - 1] - 2] : 0) + 2;
            }
            maxans = Math.max(maxans, dp[i]);
        }
    }
    return maxans;
}
```

### 二维问题

二维动态规划问题简单来说就是需要建立二维数组记录状态, 进而求解. 这也是大多数动态规划问题的解法. 最初举例的背包问题也是二维问题, 仅仅是因为它太典型了才拿来放在最前面

#### 地下城问题

一些恶魔抓住了公主（P）并将她关在了地下城的右下角。地下城是由 M x N 个房间组成的二维网格。我们英勇的骑士（K）最初被安置在左上角的房间里，他必须穿过地下城并通过对抗恶魔来拯救公主。

骑士的初始健康点数为一个正整数。如果他的健康点数在某一时刻降至 0 或以下，他会立即死亡。

有些房间由恶魔守卫，因此骑士在进入这些房间时会失去健康点数（若房间里的值为负整数，则表示骑士将损失健康点数）；其他房间要么是空的（房间里的值为 0），要么包含增加骑士健康点数的魔法球（若房间里的值为正整数，则表示骑士将增加健康点数）

为了尽快到达公主，骑士决定每次只向右或向下移动一步。

```
-2, -3,  3
-5, -10, 1
10, 30, -5
```

关键是要逆推求解

```
-5(6)  30(1)    10(1)  
1(5)   -10(11)  -5(6)  
3(2)   -3(5)    -2(7) 
```

上图是逆向的地下城图, 和上图正好是对称的. 括号中就是勇士从这里出发最少需要的血量. 以本应该是最后一步的`-5`来说, 如果是直接走到最后一步则需要的血量是6点. 进而在`1`位置的时候, 因为可以加一点血, 所以5点血就够了.

如果是`-3`的位置, 我们有两个选择. 可以走到-10也可以走到3的位置, 这里就可以比较下如果从那个位置开始走, 哪个需要的血量少. 显而易见3的位置只需要2点血就可以通关, 在加上这里所需的3点血, 就得到了5点血的结果.

以此推算, 直到真正的起点`-2`处, 就应该需要7点血

在上面的推算过程中, 其实我们已经得到了**状态转移方程**

```
blood[i][j] = Math.min(blood[i + 1][j], blood[i][j + 1]) - dungeon[i][j];
```

当前所需最少血量为, 下面能走的两部中需要血量最少的再减去这里消耗的血量就是当前位置所需的最少血量了. 当然我们还需要加上边界条件, 最终就可以得到实现代码了.

```java
private int calculateMinimumHP(int[][] dungeon) {
    int rowLen = dungeon.length;
    int colLen = dungeon[0].length;
    int[][] blood = new int[rowLen][colLen];
    for (int i = colLen - 1; i >= 0; i--) {
        for (int j = rowLen - 1; j >= 0; j--) {
            if (i == rowLen - 1 && j == colLen - 1) {
                blood[i][j] = dungeon[i][j];
            } else if (i == rowLen - 1) {
                blood[i][j] = blood[i][j + 1] + dungeon[i][j];
            } else if (j == colLen - 1) {
                blood[i][j] = blood[i + 1][j] + dungeon[i][j];
            } else {
                blood[i][j] = Math.max(blood[i + 1][j], blood[i][j + 1]) + dungeon[i][j];
            }
            blood[i][j] = Math.min(0, blood[i][j]);
            int i1 = 1 - blood[i][j];
            System.out.print(dungeon[i][j] + "(" + i1 + ")  ");
        }
        System.out.println();
    }
    return 1 - blood[0][0];
}
```

#### 正则表达式匹配

给出一个字符串 s 和一个字符规律 p，请你来实现一个支持 '.' 和 '*' 的正则表达式匹配。

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
解释: ".*" 表示可匹配零个或多个（'*'）任意字符（'.'）
```

##### 首先还是要建立状态转移方程

1.  **状态**: dp[i][j] 表示 text 从 i 开始到最后，pattern 从 j 开始到最后，此时 text 和 pattern 是否匹配
    
2.  **转移方程**: 这里就有两种情况了,也就是这道题的难点在于`*`字符, 因为其可以匹配零到多个字符. 也就是说匹配几个字符都有可能. 这样就需要**尝试匹配0到多次**. 每次都**面临多种选择**不正是动态规划的标准题型吗? 动态规划循环的意义在于尝试每一种选择的情况,进而依据之前的选择进行推出之后的选择.
    
    我们就可以依据我们要做出的选择建立状态转移方程.
    
3.  当前后一位不是`*`: 当前的text和pattern匹配 **且**  `dp[i + 1][j + 1]`为true (按照我们设定的状态这里代表了ext 从 `i+1` 开始到最后，pattern 从 `j+1` 开始到最后，可以匹配)
    
4.  当前后一位是`*`: `dp[i][j + 2]`为true (代表了跳过*和前一位字符) **或者** 是当前的text和pattern匹配(保证当前可匹配上) **且**  `dp[i + 1][j]`为true (text后面都已经匹配上) **即, 尝试当前不匹配 或者 尝试再匹配一位**
    

##### 之后是考虑特殊情况

我们的状态是`i,j`之后都是匹配成功的, 那势必要考虑第一次循环的时候怎么知道状态是什么. 这里也是动态规划法常用的解决思路.

```java
boolean[][] dp = new boolean[text.length() + 1][pattern.length() + 1]
```

dp数组的长度增加一位来记录初始特殊情况, 在这道题也就是两边为空的情况.

1.  两边都为空 -> true **[初始化特殊情况]**
    
2.  text为空, 如果pattern为`x*`(x代表任意字符), 则可以匹配0次 -> true **[在循环中判断]**
    
3.  pattern为空, text不为空 -> false **[直接跳过全为false]**
    

这两步清楚后代码就很简单了

```java
private boolean isMatchR(String text, String pattern) {
    // 多一维的空间，因为要考虑两边为空的情况进行匹配(这就是动态规划中的特殊情况处理, 在判断的时候会用到这些特殊情况)
    // 当然如果text不为空, p为空则肯定匹配不上, 所以可以忽略, 因为定义的数组默认都是False
    // dp[i][j]dp[i][j]表示 text 从 i 开始到最后，pattern 从 j 开始到最后，此时 text 和 pattern 是否匹配。
    boolean[][] dp = new boolean[text.length() + 1][pattern.length() + 1];
    // dp[len][len] 代表两个空串是否匹配了，"" 和 "" ，当然是 true 了。
    dp[text.length()][pattern.length()] = true;

    // 为什么text要从len开始减少, 是因为要匹配一圈text为空的情况.
    for (int i = text.length(); i >= 0; i--) {
        for (int j = pattern.length() - 1; j >= 0; j--) {
            boolean first_match = i < text.length() && (pattern.charAt(j) == text.charAt(i) || pattern.charAt(j) == '.');
            if (j + 1 < pattern.length() && pattern.charAt(j + 1) == '*') {
                dp[i][j] = dp[i][j + 2] || first_match && dp[i + 1][j];
            } else {
                dp[i][j] = first_match && dp[i + 1][j + 1];
            }
        }
        System.out.println();
    }
    return dp[0][0];
}
```

以上是逆推的做法, 当然也可以正推出来.

```java
private boolean isMatchS(String s, String p) {
        boolean[][] dp = new boolean[s.length() + 1][p.length() + 1];
        dp[0][0] = true;
        // sl/pl 代表的是长度, 因为要考虑长度为0的情况
        // i,j代表的是字符匹配到的位置
        // dp[sl][pl] 代表长度为sl,pl之前都可以匹配 -> 状态
        // sl,pl 长度都为0可以匹配 -> 特殊1
        // p为空 s不为空肯定都匹配不上 所以要考虑s为空p不为空的所有情况 s从长度为0开始循环 -> 特殊2
        for (int sl = 0; sl <= s.length(); sl++) {
            for (int pl = 1; pl <= p.length(); pl++) {
                int i = sl - 1, j = pl - 1;
                if (sl == 0) {
                    dp[sl][pl] = p.charAt(j) == '*' && dp[sl][pl - 2];
                } else if (p.charAt(j) == '*') {
                    boolean singleMatch = s.charAt(i) == p.charAt(j - 1) || p.charAt(j - 1) == '.';
                    dp[sl][pl] = singleMatch && dp[sl - 1][pl] || dp[sl][pl - 2];
                } else {
                    boolean singleMatch = s.charAt(i) == p.charAt(j) || p.charAt(j) == '.';
                    dp[sl][pl] = singleMatch && dp[sl - 1][pl - 1];
                }
            }
        }
        return dp[s.length()][p.length()];
    }
```

#### 最长回文字串

该提的关键是由长度为0开始循环, 核心也是找到3要素

> 状态: 开始位置为`start`结算位置为`end`的字符串是回文字串
> 
> 转移方程: P[start + 1][end - 1] && s.charAt(start) == s.charAt(end)
> 
> 特殊情况: len=1 一定为true **而** len=2 只需要判断 s.charAt(start) == s.charAt(end)

```java
public String longestPalindrome(String s) {
    int length = s.length();
    boolean[][] P = new boolean[length][length];
    int maxLen = 0;
    String maxPal = "";
    for (int len = 1; len <= length; len++) //遍历所有的长度
        for (int start = 0; start < length; start++) {
            int end = start + len - 1;
            //下标已经越界，结束本次循环
            if (end >= length) break;
            //长度为 1 和 2 的单独判断下
            P[start][end] = (len == 1 || len == 2 || P[start + 1][end - 1])
                    && s.charAt(start) == s.charAt(end); 
            if (P[start][end] && len > maxLen) {
                maxPal = s.substring(start, end + 1);
            }
        }
    return maxPal;
}
```
