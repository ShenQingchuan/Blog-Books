# LeetCode 刷题战记 - No.1237

给出一个函数 `f(x, y)` 和一个目标结果 z，请你计算方程 `f(x,y) == z` 所有可能的正整数数对 `x` 和 `y`。

给定函数是严格单调的，也就是说：

$$
f(x, y) < f(x + 1, y)
f(x, y) < f(x, y + 1)
$$
函数接口定义如下：

```java
interface CustomFunction {
public:
  // Returns positive integer f(x, y) for any given positive integer x and y.
  int f(int x, int y);
};
```

如果你想自定义测试，你可以输入整数 `function_id` 和一个目标结果 `z` 作为输入，其中 `function_id` 表示一个隐藏函数列表中的一个函数编号，题目只会告诉你列表中的 2 个函数。  

你可以将满足条件的 结果数对 按任意顺序返回。



**示例 1：**

```
输入：function_id = 1, z = 5
输出：[[1,4],[2,3],[3,2],[4,1]]
解释：function_id = 1 表示 f(x, y) = x + y
```

**示例 2：**

```
输入：function_id = 2, z = 5
输出：[[1,5],[5,1]]
解释：function_id = 2 表示 f(x, y) = x * y
```

**提示：**

```
1 <= function_id <= 9
1 <= z <= 100
题目保证 f(x, y) == z 的解处于 1 <= x, y <= 1000 的范围内。
在 1 <= x, y <= 1000 的前提下，题目保证 f(x, y) 是一个 32 位有符号整数。
```



## 思路

其实读这个题的题目时，我真的是被它的文字搅和得云里雾里的 ... 其实就按照一门编程语言它给你的解题模板来看最方便，我用 JavaScript，所以是：

1. 给出一个 `customFunction`对象，这里面包含一个函数 `f `，参数是两个整数
2. 给出一个结果值 `z`

叫你给出所有可能的

而且一定要谨记这条非常有用哒性质：**严格单调、且从题目给的范例式子来看是递增**

### 暴力解法

无非就是两层 `for` 循环， `x` 和 `y` 从`0` 到 `1000` ，找到一个解就在`solutions`中添加一个`[x, y]`：

时间复杂度为 $O(1Million)$

### 反向思路

由于题目给的条件 `x` 和 `y` 都小于或等于 `1000` ，如果把 `x` 和 `y` 看作是一个二维矩阵（按平面直角坐标系那么画），那么我们可以得出：

$$
若f(x=1, y=1000) < z  
则(1,1)到(1,1000) 都比z小，这一整「列」将被忽略，x+1  
同理： 
若f(x=1000, y=1) < z 
则(1,1)到(1000,1) 都比z小，这一整「行」将被忽略，y+1 
$$

所以我们 **「 🔪️切！切！切🤪！」**

肯定是要定一条边，让`x`或`y`从`1`到`1000`，我选择 `x`，那么可以写出如下代码：

```js
let y = 1000
const solutions = []

for (let x = 1; x <= 1000; x++) {
  while (y) {
    const d = customfunction.f(x, y) - z
    if (d <= 0) {
      if (!d) {
        solutions.push([x, y])
      }
      break;
    }
    
    y--;
  }
}
```

- 使用 `d` 这个差值是一种「**微优化**」，目的就是为了减少比较

总结一下这样的时间复杂度：已经被降低到了 $O(min(y,1000))$，从 `1M` 到 ` 1K` ，优化幅度还是很明显的。



### 继续优化

上面的做法还不够好，我们还有一个地方可以优化，`for`循环中的`x`界限可以调整为 `z`

```
f(1, *) >= 1 「x最小值为1，* 表示 y 可以为任意，所以结果 z 也应该是 大于等于自己的下界」
f(2, *) >= 2
...
当 x 或 y 大于或等于 z：
f(z, y) >= z+1
```

所以查找的时候`x, y`只用考虑在`[1, ..., z] * [1,...,z]`的范围里就可以了，不管是直接遍历还是二分查找都可以直接用这个范围，遍历`[1, 1000] * [1, 1000]`是完全多余的。



## 题解

```js
var findSolution = function(customfunction, z) {
    const solutions = []
    let y = 1000

    for (let x = 1; x <= z; x++) {
      while (y) {
        const d = customfunction.f(x, y) - z
        if (d <= 0) {
          if (!d) {
            solutions.push([x, y])
          }
          break;
        }
        y--
      }
    }

    return solutions
};
```



## 后续

- 本次刷题战略参考 [Bilibili 求老仙奶我到不了P10](https://www.bilibili.com/video/av85818647)

  其中提到：**其实本题在面试中很常见**，很容易首先涮掉用暴力解的一部分，另外其实这个题虽然表面上题目内数据范围很窄，但是考察的就是在这样的范围内，解题者是否有极致优化的思想，不「做死题」

- 我们的思路优化的目标是朝着 $O(z)$ 去的，而我们翻看一下 LeetCode-CN 上的题解，还有很多同学用「二分查找」的方式在做，就优化到了 $O(nlog_2^n)$

  ```java
  class Solution {
      public List<List<Integer>> findSolution(CustomFunction customfunction, int z) {
          List<List<Integer>> res = new ArrayList<>();
          for (int i = 1; i <= 1000; i++) {
              if (customfunction.f(i,1) > z) {
                  break;
              }
              int left = 1;
              int right = 1000;
              while (left <= right) {
                  int mid = (right + left) / 2;
                  int temp = customfunction.f(i,mid);
                  if (temp == z) {
                      List<Integer> list = new ArrayList<>();
                      list.add(i);
                      list.add(mid);
                      res.add(list);
                      break;
                  } else if (temp > z) {
                      right = mid - 1;
                  } else {
                      left = mid + 1;
                  }
              }
          }
          return res;
      }
  }
  
  本段代码作者：karua
  链接：https://leetcode-cn.com/problems/find-positive-integer-solution-for-a-given-equation/solution/gei-ding-fang-cheng-de-zheng-zheng-shu-jie-bao-li-/
  来源：力扣（LeetCode）
  著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
  ```

  