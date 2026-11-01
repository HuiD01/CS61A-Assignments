# CS61A Lab05 复习笔记

对应代码：`lab05.py`

## 1. Iterable 与 Iterator

- **Iterable（可迭代对象）**：可以传给 `iter()`，例如 list、range。
- **Iterator（迭代器）**：保存当前读取位置，可以用 `next()` 每次取一个值。
- 迭代器只能从头到尾消费一次，耗尽后再调用 `next()` 会抛出 `StopIteration`。
- 对同一个可迭代对象多次调用 `iter()`，可以得到不同的独立迭代器。
- 对迭代器本身调用 `iter()`，返回的通常还是它自己。

```python
r = range(6)
r_iter = iter(r)

next(r_iter)  # 0
list(r)       # [0, 1, 2, 3, 4, 5]，遍历 range 不影响 r_iter
list(r_iter)  # [1, 2, 3, 4, 5]
next(r_iter)  # StopIteration
```

### 常见陷阱

```python
list(t)
```

会立刻消费整个迭代器。如果答案可能较早出现，这种做法会多读取很多元素。

```python
for value in t:
    ...
```

每次循环只读取一个元素，并且找到答案后可以立即 `return`。

## 2. filter、map、zip、all、any

### filter

保留让判断函数返回真值的元素，返回一个懒迭代器。

```python
evens = filter(lambda x: x % 2 == 0, [1, 2, 3, 4])
list(evens)  # [2, 4]
```

### map

对每个元素应用函数，返回一个懒迭代器。

```python
tripled = map(lambda x: x * 3, [1, 2, 3])
list(tripled)  # [3, 6, 9]
```

### zip

按位置把多个可迭代对象组合成元组，读取到最短的输入结束。

```python
list(zip([1, 2, 3], ["a", "b"]))
# [(1, 'a'), (2, 'b')]
```

可用 `dict(zip(keys, values))` 构造字典，也可用 `zip(*pairs)` 拆开配对。

### all 和 any

```python
all(x > 0 for x in [1, 2, 3])   # True
all(x > 0 for x in [1, -2, 3])  # False
any(x > 0 for x in [-1, 0, 2])  # True
```

- `all`：是否所有元素都为真，遇到假值就停止。
- `any`：是否至少有一个元素为真，遇到真值就停止。
- 空可迭代对象：`all([]) == True`，`any([]) == False`。

这里的 `filter`、`map`、`zip` 都返回一次性迭代器，`all` 和 `any` 返回布尔值。

## 3. List 的原地修改

题目要求返回原列表对象时，不能重新绑定：

```python
s = result   # 错：s 指向新列表
```

下面两种写法都会修改原列表内容，并保持对象身份不变：

```python
s[:] = result
```

或者语义更直白地写成：

```python
s.clear()
s.extend(result)
```

验证：

```python
original = [1, 8]
alias = original
original[:] = [1, 4, 4, 8]

original is alias  # True
```

`s[:] = result` 的意思是“用 `result` 的全部元素替换 `s` 的全部内容”，不会让 `s` 指向新对象。

### insert 的代价

```python
s.insert(i, value)
```

会把索引 `i` 后面的元素依次移动，因此单次最坏是 `O(n)`。如果一边遍历一边大量插入，最坏可能达到 `O(n^2)`。

若先构造 `result`，再一次性写回，时间是 `O(n)`，但需要 `O(n)` 额外空间。

## 4. Q2：group_by

先计算 key，再检查该 key 是否已经存在：

```python
def group_by(s, fn):
    grouped = {}
    for item in s:
        key = fn(item)
        if key in grouped:
            grouped[key].append(item)
        else:
            grouped[key] = [item]
    return grouped
```

规律：

- 第一次出现某个 key：创建 `[item]`。
- 再次出现同一个 key：调用 `append(item)`。
- 每个元素只调用一次 `fn`。

## 5. Q6：repeated

题目要求找第一个**连续出现** `k` 次的值


### 固定长度窗口写法（不需要 import）

```python
def repeated(t, k):
    assert k > 1

    window = []
    for value in t:
        window.append(value)

        if len(window) > k:
            window.pop(0)

        if len(window) == k and all(item == value for item in window):
            return value
```

也可以用 `window.count(value) == k` 替代 `all(...)`：

```python
if len(window) == k and window.count(value) == k:
    return value
```

两种检查方式总复杂度都是 `O(nk)`：

### 更省空间的连续计数写法

```python
def repeated(t, k):
    assert k > 1

    previous = next(t)
    streak = 1

    for value in t:
        if value == previous:
            streak += 1
            if streak == k:
                return value
        else:
            previous = value
            streak = 1
```

这里只维护两个状态：

- `previous`：上一个值。
- `streak`：该值当前已经连续出现多少次。

如果当前值与上一个值不同，就把连续次数重置为 `1`。

时间和额外空间复杂度分别是 `O(n)` 和 `O(1)`，并且找到答案后立即停止读取迭代器。

## 6. 代码风格

- 小型函数中，循环结束后直接写 `return`，通常不需要在中间空一行。
- 相邻的顶层函数之间保留两个空行。
- 不要留下只包含空格的空白行。
- 一边遍历列表一边修改它时要格外小心；优先明确区分“正在读取的序列”和“正在构造的结果”。
