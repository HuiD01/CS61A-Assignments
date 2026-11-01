# CS61A Iterator、Generator 与树递归笔记

对应作业：HW05

## 一句话理解

> Generator 是一个可以暂停的普通函数：每执行到 `yield` 就交出一个值，然后停在那里；下一次 `next` 再从暂停处继续。

树递归只抓两个问题：

```text
当前节点要做什么？
每个孩子重复同一个任务后，结果怎样组合？
```

不要一开始就在脑中展开整棵树。

## 1. Iterable、Iterator 与 Generator

### Iterable

可以交给 `iter()` 的对象，例如 list、range、dict。

```python
iter([1, 2, 3])
iter(range(5))
```

### Iterator

保存当前读取位置，可以用 `next()` 一次取一个值。

```python
it = iter([1, 2, 3])

next(it)  # 1
next(it)  # 2
next(it)  # 3
next(it)  # StopIteration
```

Iterator 是单向、一次性的消费过程。

### Generator

包含 `yield` 的函数叫生成器函数。调用生成器函数时不会立刻执行函数体，而是返回一个 generator 对象。

```python
def count_up():
    print("开始")
    yield 1
    print("继续")
    yield 2

g = count_up()   # 此时不打印
```

执行：

```python
next(g)  # 打印“开始”，返回 1
next(g)  # 打印“继续”，返回 2
next(g)  # StopIteration
```

HW05 要求写的是生成器函数，不需要手写带有 `__iter__` 和 `__next__` 的迭代器类。

## 2. 普通函数与生成器函数

普通函数一次返回完整结果：

```python
def make_list():
    result = []
    result.append(1)
    result.append(2)
    return result
```

生成器函数每次只交出一个值：

```python
def make_generator():
    yield 1
    yield 2
```

| 普通函数 | 生成器函数 |
|---|---|
| 调用后立即执行 | 第一次 `next` 才执行 |
| `return` 给出一个值 | `yield` 给出一个值并暂停 |
| 一次返回完整结果 | 每次产生一个结果 |
| 可以重复调用 | 每个生成器对象由头到尾消费一次 |

## 3. 写生成器时该想什么

不要先想“所有结果长什么样”，先问：

```text
1. 我现在应该交出什么？
2. 交出之后，需要保存什么状态？
3. 状态怎样变成下一步？
4. 什么时候停止？
```

例如 Hailstone 序列：

| 状态 | 每次交出 | 下一个状态 |
|---|---|---|
| 当前数字 `n` | `n` | 偶数时 `n // 2`，奇数时 `3 * n + 1` |

## 4. Hailstone

循环写法：

```python
def hailstone(n):
    yield n

    while n != 1:
        if n % 2 == 0:
            n = n // 2
        else:
            n = n * 3 + 1
        yield n

    while True:
        yield 1
```

递归生成器写法：

```python
def hailstone(n):
    yield n

    if n == 1:
        while True:
            yield 1
    elif n % 2 == 0:
        yield from hailstone(n // 2)
    else:
        yield from hailstone(n * 3 + 1)
```

`yield from` 等价于逐个转发：

```python
for value in hailstone(next_n):
    yield value
```

常见错误：

```python
return hailstone(n // 2)
```

`return` 不会把递归生成器中的值逐个交出。如果函数里完全没有 `yield`，它甚至不是生成器函数。

## 5. 合并两个无限生成器

`merge(a, b)` 不能先把两个无限生成器转成列表。

只保存两个当前候选值：

```python
a_val
b_val
```

每次比较：

```text
a_val 更小 → yield a_val，只推进 a
b_val 更小 → yield b_val，只推进 b
两者相等   → 只 yield 一次，同时推进两边
```

```python
def merge(a, b):
    a_val, b_val = next(a), next(b)

    while True:
        if a_val == b_val:
            yield a_val
            a_val, b_val = next(a), next(b)
        elif a_val < b_val:
            yield a_val
            a_val = next(a)
        else:
            yield b_val
            b_val = next(b)
```

重点是只推进交出值的那个生成器，避免漏值或重复值。

## 6. 递归生成器：枚举所有选择

`stair_ways(n)` 的基本情况：

```text
n < 0：走过头，没有走法
n = 0：正好走完，有一种空路线 []
```

每次的选择：

```text
先走 1 阶
先走 2 阶
```

```python
def stair_ways(n):
    if n < 0:
        return

    if n == 0:
        yield []
        return

    for step in (1, 2):
        for rest in stair_ways(n - step):
            yield [step] + rest
```

两层循环的含义：

```text
外层：枚举第一步
内层：遍历剩余楼梯的所有走法
组合：把第一步加到每条剩余路线前面
```

例如 `stair_ways(3)`：

```text
先走 1，剩下 2 阶的所有走法是 [1, 1]、[2]
组合后得到 [1, 1, 1]、[1, 2]

先走 2，剩下 1 阶的走法是 [1]
组合后得到 [2, 1]
```

最终：

```python
[[1, 1, 1], [1, 2], [2, 1]]
```

## 7. 树的基本表示

树的结构是：

```text
label + 若干 branches，每个 branch 又是一棵树
```

```python
t = tree(1, [tree(2), tree(3)])
```

实际嵌套列表：

```python
[1, [2], [3]]
```

| 表达式 | 结果 |
|---|---|
| `label(t)` | `1` |
| `branches(t)` | `[[2], [3]]` |
| `is_leaf(t)` | `False` |

## 8. 树递归的抽象方法

不要在脑中同时展开所有分支。只处理当前节点，并假设递归函数已经在孩子上正确工作。

固定问三个问题：

```text
1. 基本情况是什么？
2. 当前节点本身要贡献什么？
3. 每个孩子返回结果后，父节点怎样组合？
```

树递归的常见结构：

```python
if 当前节点满足条件:
    yield 当前节点形成的结果

for child in branches(t):
    for result in 递归处理child:
        yield 把当前节点加进result
```

## 9. 枚举从根到目标节点的路径

函数约定：

> `yield_paths(t, value)` 产生从当前子树根 `t` 到目标 `value` 的所有路径。

```python
def yield_paths(t, value):
    if label(t) == value:
        yield [label(t)]

    for b in branches(t):
        for path in yield_paths(b, value):
            yield [label(t)] + path
```

### 当前节点就是目标

```python
if label(t) == value:
    yield [label(t)]
```

从当前节点到自身的路径只有自己。

### 从孩子中找到路径

孩子产生的路径从孩子自己开始，例如：

```python
[5]
```

父节点把自己的 label 加到前面：

```python
[1] + [5]  # [1, 5]
```

如果孩子的路径是：

```python
[2, 5]
```

父节点就会产生：

```python
[1, 2, 5]
```

递归函数一开始不需要知道所有祖先。它只产生相对于当前子树的路径，父节点在返回过程中逐层补上自己：

```text
孩子返回 [5]
父节点 2 扩展为 [2, 5]
父节点 1 再扩展为 [1, 2, 5]
```

这种方式不容易出现共享路径被修改、回溯时忘记删除元素等问题。

## 10. `yield from` 与嵌套 for

下面两种写法等价：

```python
for result in yield_paths(b, value):
    yield result
```

```python
yield from yield_paths(b, value)
```

如果还要加工每个结果，例如给路径前面加节点，就使用：

```python
for path in yield_paths(b, value):
    yield [label(t)] + path
```

## 11. 常见错误清单

- 函数需要生成多个值时忘记写 `yield`
- 把递归生成器 `return` 掉，没有继续转发其中的值
- 只 `yield` 一个完整列表，而不是逐个 `yield`
- `yield (1, 2)` 产生的是元组 `(1, 2)`，不是两个独立值
- 合并无限生成器时试图先 `list(...)`
- `n == 0` 时忘记 `yield []`
- 树递归中重复处理节点或漏掉分支
- 树递归中使用共享路径列表后忘记恢复状态
- 函数返回新树，但调用处忽略返回值

## 12. 从题解中重新学习

看过题解后，不要只确认“每一行我看懂了”。应该关掉答案，按顺序重新写：

1. 写出最小输入和预期输出。
2. 标出必须记住的状态。
3. 写出基本情况。
4. 写出一次状态转移或一次递归选择。
5. 最后补充循环或递归组合。

如果某一步说不清楚，说明抽象还没有建立，而不是单纯“代码没背熟”。

## 13. 三句话总结

```text
Generator：每到一个 yield 交出一个值，然后暂停并保留状态。
递归生成器：子问题产生多个结果，父问题逐个转发或加工。
树递归：只处理当前节点，假设孩子已经能正确完成同一个任务。
```
