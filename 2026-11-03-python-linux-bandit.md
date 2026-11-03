# 2026-11-03　Python 判断循环 + Linux 管道 + Bandit 7–9

> 记录方法，不记录密码。

## 一、Python：条件与循环

### 今天写的代码

```python
# 猜奇偶
num = int(input("请输入一个整数："))
if num % 2 == 0:
    print("这是偶数")
else:
    print("这是奇数")

# 猜数字
secret = 6
guess = 0
count = 0

while guess != secret:
    guess = int(input("请输入一个整数："))
    count += 1

    if guess > secret:
        print("大了")
    elif guess < secret:
        print("小了")

print(f"猜对了，你猜了{count}次")
```

### 知识点

| 语法 | 作用 |
|---|---|
| `==` `!=` `>` `<` `>=` `<=` | 比较，结果是 `True` / `False` |
| `if / elif / else` | 分支，从上往下判断，命中一个就跳出 |
| `while 条件:` | 条件为真时反复执行 |
| `for i in range(1, 11):` | 循环 1 到 10（**不含 11**） |
| `%` | 取余数，`num % 2 == 0` 判断偶数 |
| `+=` | `count += 1` 等价于 `count = count + 1` |

### 三个必须理解的点

**1. 缩进是语法，不是排版**

```python
if True:
    print("属于 if")    # 4 个空格缩进
print("不属于 if")      # 顶格，在 if 外面
```

缩进错一个空格就报 `IndentationError`。

**2. `elif` 是"否则如果"**

```python
if x > 10:
    ...
elif x > 5:
    ...
else:
    ...
```

从上往下判断，**命中第一个就跳出**，后面的不再检查。

**3. `while` 可能死循环**

```python
while True:      # 条件永远为真
    print("卡住了")
```

如果循环体里没有任何能让条件变假的操作，程序会一直跑，`Ctrl + C` 才能停。

### 一个设计问题

猜数字循环结束时，`guess` 可能等于 `secret`（猜中），也可能因为次数用完而退出。**怎么区分这两种情况？**

答案：循环结束后再判断一次：

```python
if guess == secret:
    print(f"猜对了，用了 {count} 次")
else:
    print(f"没猜出来，答案是 {secret}")
```

这种"循环 + 事后判断"的结构，后面写脚本会反复用到。

---

## 二、Linux：管道与重定向

### 通用结构

```
命令名   选项    目标
ls       -l      /usr/bin
```

### 今天练的七条命令

| 命令 | 含义 |
|---|---|
| `ls /usr/bin \| wc -l` | 数出 `/usr/bin` 有多少个条目 |
| `echo "hello" > test.txt` | 把 hello 写入文件，**覆盖**原有内容 |
| `echo "world" >> test.txt` | 把 world **追加**到文件末尾 |
| `cat test.txt` | 把文件内容原样输出到屏幕 |
| `wc -l test.txt` | 数文件有多少行 |
| `ls /usr/bin \| grep python` | 从列表里挑出**名字含** python 的 |
| `history \| grep ssh` | 从命令历史里挑出含 ssh 的 |

### 管道 vs 重定向

| 符号 | 数据流向 | 类比 |
|---|---|---|
| `\|` | 命令 → **另一个命令** | 水管接水管 |
| `>` | 命令 → **文件**（覆盖） | 水倒进桶里 |
| `>>` | 命令 → **文件末尾**（追加） | 往桶里续水 |

**关键理解**：`ls /usr/bin | wc -l` 执行时，`ls` 的输出**不会显示在屏幕上**，它被管道直接送进了 `wc`。屏幕上只有 `wc` 算出的数字。

### grep 的本质

`grep 关键词` 永远在回答同一个问题：**"哪些行里出现了这个词？"**

它只做**字符串匹配**，不关心那一行是什么内容。所以：

- `ls | grep python` → 筛的是**文件名**
- `grep error app.log` → 筛的是**日志行**

同一条命令，输入不同，意义就不同。

### grep 常用选项

| 选项 | 作用 |
|---|---|
| `-i` | 忽略大小写 |
| `-n` | 显示行号 |
| `-v` | 反向，输出**不含**关键词的行 |
| `-r` | 递归搜索整个目录 |
| `-c` | 只输出匹配的行数 |

---

## 三、Bandit 7–9

### 第 7 关：grep 搜索

```bash
grep millionth data.txt
```

逐行扫描 `data.txt`，输出包含 `millionth` 的行。

**实际用途**：

```bash
grep -rn "password" /var/www/html      # 源码里找写死的密码
grep -i "union select" access.log      # 日志里找 SQL 注入特征
grep -v "^#" /etc/ssh/sshd_config      # 去掉注释看配置
```

### 第 8 关：sort + uniq -u

```bash
sort data.txt | uniq -u
```

| 命令 | 作用 |
|---|---|
| `sort` | 按字典序排序 |
| `uniq -u` | 只保留**没有重复**的行 |

**为什么必须 sort**：`uniq` 只比较**相邻**的行。

文件内容：

```
apple
banana
apple
```

直接 `uniq -u`：两个 apple 不相邻，都被当成"唯一"，全部输出，错的。

先 `sort`：

```
apple
apple
banana
```

重复的挨在一起，`uniq -u` 才能识别并丢掉，只输出 `banana`。

**记住：`uniq` 永远和 `sort` 一起用。**

`uniq` 的三个常用选项：

| 选项 | 作用 |
|---|---|
| `-u` | 只显示**不重复**的行 |
| `-d` | 只显示**重复**的行 |
| `-c` | 统计每行出现次数 |

经典组合：

```bash
cat access.log | awk '{print $1}' | sort | uniq -c | sort -rn | head
```

统计访问次数最多的 IP。拆开看：取第一列（IP）→ 排序 → 计数 → 按次数倒序 → 取前 10。

### 第 9 关：strings 提取二进制字符串

```bash
strings data.txt | grep '==='
```

`strings` 扫描文件字节，把**连续的可打印字符**抽出来（默认长度 ≥ 4），遇到不可打印字节就断开。

**为什么需要它**：`data.txt` 是二进制，`cat` 会输出乱码甚至搞乱终端。`strings` 把里面藏着的可读文本"捞"出来。

**实际用途**：

```bash
strings ./program | grep -i "password"       # 程序里有没有硬编码密码
strings sample.exe | grep -E "https?://"     # 恶意样本里的 C2 地址
```

这是逆向工程和恶意代码分析的**第一个动作**：先看看二进制里有什么人类能读的东西。

---

## 四、三关对应的三种能力

| 关卡 | 动作 | 底层问题 |
|---|---|---|
| 7 | 在文本里搜关键词 | **搜索** |
| 8 | 找出独特的那个 | **去重 / 异常识别** |
| 9 | 从二进制提取可读信息 | **类型转换** |

以后拿到任何未知数据，都是这三步：

1. 先 `file` 看类型
2. 文本就 `grep`
3. 二进制就 `strings`
4. 数据量大就 `sort | uniq`

---

## 五、踩坑记录

| 错误 | 原因 | 正确 |
|---|---|---|
| `/use/bin` | `usr` 少写一个 `r` | `/usr/bin` |
| `wc -1` | 数字 1 和小写 l 混淆 | `wc -l`（line） |
| `uniq -u` 不加 sort | uniq 只比较相邻行 | 先 `sort` |
| 循环结束后不判断 | 分不清"猜中"和"次数用完" | 循环外再 `if` 一次 |

---

## 六、自测

1. `while` 和 `for` 分别适合什么场景？
2. 为什么 `if` 判断后要缩进？缩进错了会怎样？
3. `|` 和 `>` 的区别是什么？
4. 为什么 `uniq -u` 前面必须加 `sort`？
5. `grep -v` 是干什么的？
6. `strings` 命令解决了什么问题？

能答上 5 个，今天就算过关。

---

## 明天

- Python：列表 list（增删改查、切片）
- Bandit 10–12
- 把命令练习迁移到虚拟机里做一遍
