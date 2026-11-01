# CS61A Stream 笔记

## 一句话理解

> Stream 是一个“按需生成”的链表：当前元素已经知道，后半部分只保存生成方法，第一次访问时才计算，之后把结果缓存下来。

```text
当前节点 = first + “以后怎么生成 rest”的函数
```

它不是一次性迭代器。已经访问过的 Stream 节点会保留，可以从头部再次读取。

## 1. 为什么需要 Stream

普通列表会在创建时准备好所有元素：

```python
[1, 2, 3, 4, 5]
```

但这无法直接表示无限序列：

```text
1, 2, 3, 4, 5, ...
```

Stream 不储存所有元素，而是保存生成规则：

```text
现在知道 1
剩下的内容是：把起点加一，再继续生成
```

只有需要下一个元素时，才执行规则。

## 2. Stream 的基本结构

```python
class Stream:
    class empty:
        def __repr__(self):
            return 'Stream.empty'

    empty = empty()

    def __init__(self, first, compute_rest=lambda: empty):
        assert callable(compute_rest)
        self.first = first
        self._compute_rest = compute_rest

    @property
    def rest(self):
        if self._compute_rest is not None:
            self._rest = self._compute_rest()
            self._compute_rest = None
        return self._rest

    def __repr__(self):
        return 'Stream({0}, <...>)'.format(repr(self.first))
```

几个名称的含义：

| 名称 | 含义 |
|---|---|
| `first` | 当前元素，已经保存 |
| `compute_rest` | 不接受参数、返回下一个 Stream 的函数 |
| `_compute_rest` | 内部保存的“计算后半部分”的函数 |
| `_rest` | 第一次计算后缓存的下一节 Stream |
| `rest` | 公开接口，访问时自动处理计算和缓存 |
| `Stream.empty` | 空流的特殊标记 |

## 3. lambda 只是没有名字的函数

下面两种写法表达相同的思路：

```python
def compute_rest():
    return Stream(5)

s = Stream(1, compute_rest)
```

```python
s = Stream(1, lambda: Stream(5))
```

关键在于：创建 `s` 时，函数体不会执行。只有访问：

```python
s.rest
```

时，才真正调用 `compute_rest`。

## 4. 第一次访问 rest 发生什么

假设：

```python
s = Stream(1, lambda: Stream(2, lambda: Stream(3)))
```

刚创建时：

```text
s.first = 1
s._compute_rest = 一个函数
```

第一次访问 `s.rest`：

```python
self._rest = self._compute_rest()
self._compute_rest = None
```

发生两件事：

1. 调用函数，得到下一节 `Stream(2, ...)`。
2. 把它保存到 `_rest`，并把 `_compute_rest` 设为 `None`。

以后再访问 `s.rest`，直接返回缓存的 `_rest`，不会重新计算。

状态变化：

| 时刻 | `_rest` | `_compute_rest` |
|---|---|---|
| 刚创建 | 尚未创建 | 一个函数 |
| 第一次访问 `rest` 后 | 已缓存下一节 Stream | `None` |
| 再次访问 `rest` | 已缓存下一节 Stream | `None` |

因此：

```python
s.rest is s.rest  # True
```

两次得到的是同一个缓存节点。

## 5. 缓存到底是什么

缓存就是：

> 第一次算出来的结果不要丢掉，保存起来；以后需要同一个结果时直接使用。

惰性和缓存是两件事：

```text
惰性：现在先不算，等需要时再算
缓存：算过一次以后，把结果留下来
```

| 访问次数 | 不缓存 | 有缓存 |
|---|---|---|
| 第一次 | 计算 | 计算并保存 |
| 第二次 | 重新计算 | 直接读取保存结果 |
| 第三次 | 重新计算 | 直接读取保存结果 |

不缓存并不是“不需要计算就能取出来”。无论有没有缓存，第一次访问都必须计算。区别在第二次和以后：

- 没有缓存：每次重新执行生成函数。
- 有缓存：第一次执行后保存结果，后续直接读取。

Stream 需要缓存，才能保证同一个位置始终返回同一个节点，并且可以重复读取已经探索过的前缀。

## 6. property

`property` 让方法可以通过普通属性语法访问：

```python
s.rest
```

而不是：

```python
s.rest()
```

访问 `s.rest` 时，Python 会自动执行 `rest` 方法中的代码，完成缓存判断。

`property` 不只是为了简洁，它还：

- 隐藏 `_rest` 和 `_compute_rest` 的内部细节
- 保证每次读取都经过同一套缓存逻辑
- 让 Stream 和 Link 都能用 `node.rest` 这种自然写法
- 防止外部直接覆盖只读的 `rest` 属性

## 7. 访问第三个元素要按顺序吗

可以从第一个节点写：

```python
s.rest.rest.first
```

但运行顺序仍然是：

```text
s.rest       → 生成并缓存第二个节点
第二个.rest  → 生成并缓存第三个节点
.first       → 取出第三个值
```

不能像列表那样写：

```python
s[2]  # Stream 没有随机索引
```

每个节点只知道当前值和怎样生成下一节，不知道更后面的节点在哪里，所以必须沿着 rest 链前进。

如果前面已经访问过，部分步骤会直接读取缓存。

## 8. repr 是什么

`repr` 用于生成对象的表示形式，常用于调试和交互式解释器显示。

```python
>>> str("hello")
hello

>>> repr("hello")
"'hello'"
```

Stream 定义了：

```python
def __repr__(self):
    return 'Stream({0}, <...>)'.format(repr(self.first))
```

所以显示：

```python
Stream(1, <...>)
```

这里的 `<...>` 不是空流，而是表示“后面的内容暂时没有展开”。

如果 `__repr__` 尝试显示所有元素，无限流就永远打印不完，而且会强迫 Stream 提前计算所有 `rest`，破坏惰性。因此 `repr` 有意不展开后半部分。

## 9. 下划线不是强制私有

Python 中，单下划线开头的名称只是一种约定：

```python
_rest
_compute_rest
```

表示“这是内部实现，外部通常不要直接依赖”。Python 不会阻止访问：

```python
s._rest
```

但在 Stream 中应该使用公开接口：

```python
s.rest
```

因为直接访问 `_rest` 可能遇到：

- `_rest` 还没有被计算和创建
- 绕开缓存流程
- 依赖内部实现，代码容易失效

## 10. 无限整数流

```python
def integer_stream(first):
    def compute_rest():
        return integer_stream(first + 1)
    return Stream(first, compute_rest)
```

创建：

```python
positives = integer_stream(1)
```

时只会生成第一个节点：

```text
1 + “以后生成从 2 开始的整数流”
```

访问 `positives.rest` 后，才生成：

```text
2 + “以后生成从 3 开始的整数流”
```

每次需求只展开一个节点，所以 Stream 可以表示无限序列。

不能直接对无限流调用 `list()`，因为那会要求计算出无限多个元素。

## 11. map_stream 与 filter_stream

`map_stream`：

```python
def map_stream(fn, s):
    if s is Stream.empty:
        return s

    def compute_rest():
        return map_stream(fn, s.rest)

    return Stream(fn(s.first), compute_rest)
```

- 新流的第一项 `fn(s.first)` 会立即计算。
- 剩余部分通过 `compute_rest` 延迟计算。

`filter_stream`：

```python
def filter_stream(fn, s):
    if s is Stream.empty:
        return s

    def compute_rest():
        return filter_stream(fn, s.rest)

    if fn(s.first):
        return Stream(s.first, compute_rest)
    else:
        return compute_rest()
```

如果第一个元素不符合条件，就继续向后找，直到找到新的第一项。

惰性不等于完全不计算。`filter_stream` 会计算到足够产生当前第一项为止。

## 12. 质数流的大致思路

```text
2
保留剩余数字中不能被 2 整除的
取下一个 3
保留剩余数字中不能被 3 整除的
取下一个 5
……
```

每一层过滤都通过递归的 `compute_rest` 延迟执行。请求前七个质数时，只计算到能够产生 `17` 为止。

## 13. Stream 与 Iterator 的区别

| 特点 | Iterator | Stream |
|---|---|---|
| 是否保存读取位置 | 是 | 每个节点保存自己的 rest |
| 已访问内容能否重读 | 通常不能 | 能，节点会缓存 |
| 是否适合无限数据 | 可以 | 可以 |
| 从头部再次读取 | 通常不行 | 可以 |

```python
first_k_as_list(prime_numbers, 7)
```

只会让函数内部的局部变量向后移动，不会销毁 `prime_numbers` 的头部。之后：

```python
prime_numbers.first
```

仍然是 `2`。

## 14. 最容易混淆的点

- `s.first` 是普通属性，不加括号。
- `s.rest` 是 property，也不加括号。
- `s.rest()` 会尝试调用返回的 Stream，通常报错。
- `_compute_rest` 必须是无参数函数，返回值必须是 Stream 或 `Stream.empty`。
- 判断空流用 `is Stream.empty`，不要用 `==`。
- `repr` 不展开 Stream，避免无限打印。
- 访问第 `n` 个元素仍然要沿着 rest 链逐层前进。
- 第一次读取仍要计算，缓存只在后续读取和重复遍历时节省工作。
- 惰性表示“需要时才计算”，缓存表示“算完以后保存”。

## 15. 自测

能回答下面这些问题，才算真正理解：

1. 为什么 `Stream(1, lambda: Stream(2))` 创建时不会计算后半部分？
2. 第一次访问 `s.rest` 后，`_rest` 和 `_compute_rest` 分别是什么？
3. 为什么 `s.rest is s.rest` 是 `True`？
4. 为什么 `first_k_as_list` 消费流之后，仍然可以从头部访问？
5. 没有缓存时，第二次访问 `rest` 会发生什么？
6. 为什么 `s.rest.rest.first` 仍然按第二、第三的顺序计算？
7. 为什么 Stream 的 `repr` 只显示第一个元素？
