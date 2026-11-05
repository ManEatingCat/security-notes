# 2026-11-05　Python 函数 + vim + Bandit 13–15

> 记录方法，不记录密码。

---

## 一、Python：函数

### 为什么需要函数

同一段逻辑用三次，复制三遍很蠢。函数就是**把一段代码打包、起个名字、随时调用**。

### 基本语法

```python
def greet(name):
    return f"你好，{name}"

print(greet("小明"))
```

| 部分 | 含义 |
|---|---|
| `def` | 定义函数的关键字 |
| `greet` | 函数名 |
| `name` | 参数（调用时传进来） |
| `return` | 把结果交回去，函数到此结束 |

### 三个要点

**1. 有返回值 vs 没返回值**

```python
def add(a, b):
    return a + b        # 有返回值，可以 x = add(1, 2)

def show(msg):
    print(msg)          # 只打印，返回 None

x = add(1, 2)           # x = 3
y = show("hi")          # y = None
```

**2. 默认参数**

```python
def power(base, exp=2):
    return base ** exp

power(3)      # 9
power(3, 3)   # 27
```

**3. 作用域**

```python
def f():
    x = 10      # 局部变量，函数外访问不到

f()
print(x)        # NameError
```

### 设计原则：函数只做一件事

不推荐（既算又打印）：

```python
def total(nums):
    print(f"总和：{sum(nums)}")
    return sum(nums)
```

推荐（只算，打印交给调用处）：

```python
def total(nums):
    return sum(nums)

print(f"总和：{total(nums)}")
```

**为什么**：以后要把结果写文件、存数据库、发给别人，函数一行都不用改。函数里写死 `print`，换个输出方式就得改函数。

### 命名注意

- 别用内置函数名当变量名：`sum`、`max`、`min`、`list`、`dict`
- 名字要说明“做什么”，不是“是什么”：`total` 比 `sum_num` 清楚

### 今日练习代码

```python
nums = []

def total(nums):
    return sum(nums)

def average(nums):
    return sum(nums) / len(nums)

def max_min(nums):
    return max(nums), min(nums)

for i in range(5):
    nums.append(int(input(f"这是第{i+1}个数：")))

print(f"总和：{total(nums)}")
print(f"平均：{average(nums):.2f}")

hi, lo = max_min(nums)
print(f"最大：{hi} 最小：{lo}")
```

`return a, b` 返回的是**元组**，可以用 `x, y = 函数()` 一次接住。

---

## 二、vim 基础

### 三种模式

| 模式 | 进入方式 | 能干什么 |
|---|---|---|
| 普通模式 | 打开就是 | 移动、删除、复制 |
| 插入模式 | 按 `i` | 打字 |
| 命令模式 | 按 `:` | 保存、退出、替换 |

### 生存命令

| 操作 | 按键 |
|---|---|
| 进入插入 | `i` |
| 回到普通 | `Esc` |
| 保存退出 | `:wq` 回车 |
| 不保存退出 | `:q!` 回车 |
| 删除一行 | `dd` |
| 复制一行 | `yy` |
| 粘贴 | `p` |
| 撤销 | `u` |
| 搜索 | `/关键词` 回车 |
| 跳到文件首/尾 | `gg` / `G` |

### 最容易卡的地方

打完字直接敲 `:wq` 没反应——因为还在插入模式，`:wq` 被当成普通字符打进去了。

**先 `Esc`，再 `:wq`。**

---

## 三、Bandit 13–15

### 第 13 关：SSH 私钥登录

```bash
ls
# HINT  sshkey.private

cat HINT
```

私钥文件属主是 bandit14，所以 `chmod` 报 `Operation not permitted`。复制到可写目录：

```bash
cp sshkey.private /tmp/key
chmod 600 /tmp/key
```

然后原本应该：

```bash
ssh -i /tmp/key bandit14@localhost -p 2220
```

**但 2026 年版本禁止从 localhost 登录**：

```
!!! Connecting from/to localhost is blocked to conserve resources.
```

所以改成：**把私钥下载到本机，从本机直连**。

```powershell
# Windows PowerShell 里执行
scp -P 2220 bandit13@bandit.labs.overthewire.org:sshkey.private C:\Users\hester\bandit14.key
ssh -i C:\Users\hester\bandit14.key bandit14@bandit.labs.overthewire.org -p 2220
```

### 公私钥认证原理

| 文件 | 在哪 | 作用 |
|---|---|---|
| 公钥（`id_ed25519.pub`） | 服务器 | 用来验证 |
| 私钥（`id_ed25519`） | 你自己手里 | 用来证明身份 |

流程：

1. 你连服务器，说“我要用这把公钥登录”
2. 服务器发一段随机数据
3. 你用私钥签名，发回去
4. 服务器用公钥验证签名
5. 验证通过，放行

**整个过程私钥从不离开你的机器**，所以比密码安全：密码要传给服务器，公钥认证不用。

### 第 14 关：向端口发送数据

```bash
cat /etc/bandit_pass/bandit14
```

Bandit 每关密码都存在 `/etc/bandit_pass/` 下，权限设计成只有对应用户能读。

```bash
cat /etc/bandit_pass/bandit14 | nc localhost 30000
```

- `nc`（netcat）向指定 IP 和端口发数据
- 用管道把密码直接喂给 `nc`，避免手抄

返回 bandit15 的密码。

### 第 15 关：TLS 加密连接

```bash
openssl s_client -connect localhost:30001
```

连上后输入密码，找返回结果里的 `Correct!`。

**为什么要用 openssl 而不是 nc**：30001 端口上的服务是 **TLS 加密**的，明文 `nc` 连上去只会看到乱码或直接断开。

`s_client` 是命令行版 TLS 客户端，负责完成握手、加密、解密。

### 这三关串起来的知识

| 关卡 | 在学什么 |
|---|---|
| 13 | 身份认证（公私钥） |
| 14 | 明文协议通信（nc） |
| 15 | 加密协议通信（TLS） |

**同一个端口，可能有完全不同的协议栈。** 判断一个服务在说什么，是渗透测试里最基本的一步（后面会用 `nmap -sV` 自动化这件事）。

---

## 四、踩坑记录

| 错误 | 原因 | 正确 |
|---|---|---|
| `chmod 600 sshkey.private` 失败 | 文件属主是 bandit14 | 复制到 /tmp 再改 |
| `ssh -i key bandit14@localhost` 被拒 | 新版禁止 localhost 登录 | 从本机直连 |
| 在 Bandit 里跑 `scp ... C:\...` | scp 的“本地”是运行它的机器 | 在 Windows PowerShell 里跑 |
| 把密码当命令敲 | 密码只该出现在提示符后 | 别当命令输入 |
| 在插入模式敲 `:wq` | 没先按 Esc | 先 Esc 再 `:wq` |

---

## 五、自测

1. `def` 后面的括号里放什么？`return` 和 `print` 有什么区别？
2. 为什么建议函数只返回值、不在里面 `print`？
3. 为什么不能用 `sum` 当变量名？
4. vim 里怎么从插入模式回到普通模式？怎么保存退出？
5. 公钥和私钥分别放在哪？为什么私钥不能给别人？
6. `scp` 命令在哪台机器上运行，文件就存到哪台机器——这句话怎么理解？
7. `nc` 和 `openssl s_client` 的区别是什么？

---

## 明天

- Python：列表推导式、字典
- Bandit 16–18
- 开始每天 15 分钟的英文文档阅读
