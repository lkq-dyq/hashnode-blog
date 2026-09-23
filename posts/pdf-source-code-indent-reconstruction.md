---
title: "从软件著作权PDF提取源码并重建Python缩进的技术实践"
slug: pdf-source-code-indent-reconstruction
subtitle: 秦梓恒的技术笔记
tags: python,reverse-engineering,ast
publishedAt: 2026-09-22T09:00:00Z
saveAsDraft: false
enableToc: true
hideFromCommunity: false
canonical: https://lkq-dyq.github.io/python/reverse-engineering/2026/09/22/pdf-source-code-indent-reconstruction.html
---

## 背景

软件著作权登记提交的源代码通常是 PDF 格式（前30页+后30页）。当原始 `.py` 文件丢失、只剩登记 PDF 时，如何从 PDF 中提取出可运行的 Python 代码？

直接用 `pypdf` 提取文本会发现三个致命问题：
1. **全部缩进丢失** — PDF 不保留空白字符，所有行左对齐
2. **行号残留** — 每行前有数字行号（如 `42  def fit(self):`）
3. **多行语句被拆断** — PDF 按页排版，跨行的函数调用、字典定义被强制断行

经过 18 个版本的迭代，我实现了一套基于语法规则的缩进重建器，最终让 543 行代码通过 `ast.parse` 零语法错误检查。本文记录完整技术方案。

## 提取与清理

### 第一步：pypdf 提取原始文本

```python
from pypdf import PdfReader

reader = PdfReader("软著_源代码.pdf")
text = "\n".join(page.extract_text() or "" for page in reader.pages)
```

### 第二步：清理页眉、页码、行号

软著 PDF 每页都有页眉（"计算机软件著作权登记"、软件名称、版本号等）和行号。用正则逐行过滤：

```python
import re

lines_out = []
for line in text.split("\n"):
    s = line.strip()
    if not s:
        continue
    # 过滤页眉
    if any(k in s for k in ["计算机软件著作权登记", "源 代 码", "软件名称：", "版本号："]):
        continue
    # 过滤"第 N 页"
    if re.match(r"^第 \d+ 页$", s):
        continue
    # 过滤纯数字行号
    if re.match(r"^\d{1,4}$", s):
        continue
    # 行号前缀去除："42  def fit(self):" → "def fit(self):"
    m = re.match(r"^(\d+)\s(.+)$", s)
    if m and int(m.group(1)) <= 1200:
        s = m.group(2)
    lines_out.append(s)
```

## 核心难点：续行合并

PDF 按页排版，Python 的多行语句（函数调用跨行、字典跨行、for 循环跨行）会被拆成独立行。如果不合并，缩进重建会产生大量语法错误。

### 策略一：去行内注释

PDF 把多行字典的每个键值对压成独立行，行内有 `#` 注释。如果直接合并，注释会"吃掉"后续键值对。先去掉字符串外的 `#` 注释：

```python
def strip_inline_comment(line):
    """去掉字符串外的 # 注释，保留字符串内的 #"""
    result = []
    in_str = False
    str_ch = None
    i = 0
    while i < len(line):
        ch = line[i]
        if ch == "\\" and in_str:  # 转义字符
            result.append(line[i:i+2])
            i += 2
            continue
        if ch in ('"', "'"):
            if not in_str:
                in_str = True
                str_ch = ch
            elif ch == str_ch:
                in_str = False
            result.append(ch)
            i += 1
            continue
        if ch == "#" and not in_str:
            break  # 注释开始，截断
        result.append(ch)
        i += 1
    return "".join(result).rstrip()
```

### 策略二：括号深度跟踪

**这是最关键的合并策略。** 一行结束时如果括号（圆括号/方括号/花括号）未闭合，说明语句跨行了，需要合并下一行：

```python
def line_needs_merge(line):
    """行结束时是否需要合并下一行"""
    s = line.rstrip()
    if not s:
        return False
    in_str = False
    str_ch = None
    depth = 0  # 括号深度
    i = 0
    while i < len(s):
        ch = s[i]
        if ch == "\\" and in_str:
            i += 2
            continue
        if ch in ('"', "'"):
            if not in_str:
                in_str = True
                str_ch = ch
            elif ch == str_ch:
                in_str = False
        elif not in_str:
            if ch in "([{":
                depth += 1
            elif ch in ")]}":
                depth -= 1
        i += 1
    if in_str:    # 字符串未闭合
        return True
    if depth > 0: # 括号未闭合
        return True
    # for ... in 结尾（无冒号）也是续行
    if re.search(r"\bin$", s) and not s.endswith(":"):
        return True
    return False
```

合并循环：

```python
merged = []
i = 0
while i < len(lines_out):
    line = lines_out[i]
    # 反复合并直到行不需要续行
    while line_needs_merge(line) and i + 1 < len(lines_out):
        i += 1
        line = line + " " + lines_out[i].strip()
    # = 开头续行（元组解包赋值被拆断）
    if merged and re.match(r"^=(?!=)", line.strip()):
        merged[-1] = merged[-1].rstrip() + " " + line.strip()
        i += 1
        continue
    merged.append(line)
    i += 1
```

### 为什么不合并字典/列表的字面量？

最初版本用括号深度跟踪会合并整个字典定义，但行内注释会破坏合并。去掉行内注释后，字典的每个键值对变成独立行——括号深度跟踪不再触发（因为每个键值对行的括号是平衡的）。这反而正确保留了字典的多行结构，由后续缩进重建处理。

## 缩进重建

### 基本逻辑

维护一个栈，每个元素记录 `(缩进量, 类型)`。遇到 `class`/`def`/`:` 结尾的行时入栈，遇到同级别或更高级别的关键字时退栈：

```python
IND = 4
result = []
stack = [(0, None)]  # (indent, type)

def cur():
    return stack[-1][0]
```

### 退栈规则

**`class` 到来** — 退到顶层：

```python
elif re.match(r"^class ", s):
    while len(stack) > 1:
        stack.pop()
```

**`def` / `@decorator` 到来** — 退掉上层的 `def` 和 `block`（同级方法）：

```python
if s.startswith("@") or re.match(r"^def ", s):
    while len(stack) > 1 and stack[-1][1] in ("def", "block"):
        stack.pop()
```

**`return` / `raise` / `break` / `continue` / `pass`** — 这些终止语句之后的代码通常需要退栈（退出当前 block）。只退 1 层，避免过度退栈：

```python
TERMINATORS = ("return ", "raise ", "break", "continue", "pass")
prev_was_terminator = False

# ... 在循环中
elif prev_was_terminator:
    if len(stack) > 1 and stack[-1][1] == "block":
        stack.pop()
    prev_was_terminator = False
```

### block 子类型精确退栈（关键突破）

`else`/`elif` 需要退到对应的 `if`，`except`/`finally` 需要退到对应的 `try`。给 block 标记子类型：

```python
# stack 元素改为 (indent, type, subtype)
# subtype: if / for / while / try / with

def block_subtype(line):
    s = line.strip()
    if s.startswith("try"):    return "try"
    if s.startswith("for "):   return "for"
    if s.startswith("while "): return "while"
    if s.startswith("if "):    return "if"
    if s.startswith("with "):  return "with"
    return "block"
```

`else`/`elif` 退到最近的 `if`/`for`/`while`：

```python
elif s.startswith("elif ") or s.startswith("else"):
    # 如果前面是 terminator，先退1层
    if prev_was_terminator:
        pop_block()
    # 退到最近的 if/for/while block
    while len(stack) > 1 and stack[-1][1] == "block" \
            and stack[-1][2] not in ("if", "for", "while"):
        stack.pop()
    pop_block()  # 退掉 if/for/while block
```

`except`/`finally` 退到最近的 `try`：

```python
elif re.match(r"^except", s) or s.startswith("finally"):
    if prev_was_terminator:
        pop_block()
    while len(stack) > 1 and stack[-1][1] == "block" \
            and stack[-1][2] != "try":
        stack.pop()
    pop_block()  # 退掉 try block
```

### `:` 结尾判定

判断一行是否以 `:` 结尾需要排除字符串内的冒号和括号内的冒号（如字典 `{key: value}`）：

```python
def ends_colon(line):
    s = line.rstrip()
    if not s.endswith(":"):
        return False
    in_str = False
    str_ch = None
    depth = 0
    for ch in s:
        if in_str:
            if ch == str_ch:
                in_str = False
        else:
            if ch in ('"', "'"):
                in_str = True
                str_ch = ch
            elif ch in "([{":
                depth += 1
            elif ch in ")]}":
                depth -= 1
    return depth <= 0
```

## 最终效果

```python
import ast

code = open("creditguard_v1.0.py", encoding="utf-8").read()
ast.parse(code)  # 通过！543 行，零语法错误
```

## 迭代历程

| 版本 | 行数 | 结果 | 关键改进 |
|------|------|------|----------|
| v1-v4 | 1149→399 | 失败 | 续行合并太激进 |
| v5-v7 | ~900 | 失败 | def 退栈、@decorator 退栈 |
| v8 | 937 | 前629行正确 | 保守合并（字符串+=合并） |
| v11 | 656 | 失败 | 括号深度跟踪合并 |
| v12 | 546 | 失败(行301) | 去行内注释 |
| v13-v14 | 545 | 失败(行353) | = 续行、for-in 续行 |
| v15-v17 | 543 | 失败(行386-392) | return/raise 退栈 |
| **v18** | **543** | **通过** | block 子类型精确退栈 |

## 局限性

1. **行内注释丢失** — 为避免合并冲突去掉了字符串外 `#` 注释
2. **换行位置可能与原始代码略有差异** — 多行语句合并后换行点不固定
3. **缩进层级依赖启发式退栈** — `return`/`raise` 后退 1 层是经验规则，复杂嵌套场景可能不完美

## 总结

从 PDF 重建 Python 缩进的核心挑战是：没有原始缩进信息，必须从语法规则反推。关键技术点：

1. **括号深度跟踪**是最可靠的续行合并策略
2. **去行内注释**避免字典合并时注释吃掉键值对
3. **block 子类型标记**让 `else`/`except` 精确退栈到对应关键字
4. **`return`/`raise` 退栈**处理终止语句后的代码块切换

最终 543 行代码通过 `ast.parse` 零错误验证，代码已开源：[github.com/lkq-dyq/CreditGuard-V1.0](https://github.com/lkq-dyq/CreditGuard-V1.0)
