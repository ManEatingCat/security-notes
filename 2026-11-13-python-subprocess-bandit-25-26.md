# 2026-11-13　Python 调用系统 + 端口探测 + Bandit 25–26

> 记录方法，不记录密码。

---

## 一、Python：三个系统模块

### os：文件系统

```python
import os

os.getcwd()                    # 当前工作目录
os.listdir(".")                # 列出目录内容
os.path.exists("text.txt")     # 是否存在
os.path.join("a", "b")         # 拼路径，跨平台安全
os.path.getsize("text.txt")    # 文件大小（字节）
```

**为什么用 `os.path.join` 而不是 `"a" + "/" + "b"`**

| 系统 | 分隔符 |
|---|---|
| Windows | `\` |
| Linux | `/` |

`os.path.join("a", "b")` 会自动选对分隔符，代码不用改。

### sys：解释器

```python
import sys

print(sys.argv)      # 命令行参数
# python3 test.py a b  →  ['test.py', 'a', 'b']

sys.exit(1)          # 退出，返回码 1
```

`sys.argv[0]` 永远是脚本名，参数从 `[1]` 开始：

```python
import sys

if len(sys.argv) < 2:
    print("用法: python3 scan.py <host>")
    sys.exit(1)

host = sys.argv[1]
```

**`sys.exit(1)` 表示“出错了”，`sys.exit(0)` 或直接结束表示成功。** 这个返回码会被 shell 的 `$?` 捕获，也能配合 `&&` 使用。

### subprocess：调用外部命令

```python
import subprocess

result = subprocess.run(
    ["ls", "-l"],
    capture_output=True,    # 捕获输出
    text=True               # 按字符串处理
)

print(result.stdout)        # 正常输出
print(result.stderr)        # 错误输出
print(result.returncode)    # 0 = 成功
```

### 列表 vs shell=True

```python
subprocess.run("ls -l", shell=True)      # 能用，但危险
subprocess.run(["ls", "-l"])             # 推荐
```

**为什么 `shell=True` 危险**：

```python
filename = "a.txt; rm -rf ~"        # 用户输入的"文件名"
subprocess.run(f"cat {filename}", shell=True)
```

shell 会把 `;` 后面的命令也执行了。这叫**命令注入**。

用列表形式：

```python
subprocess.run(["cat", filename])    # 整个字符串被当成一个参数，不会执行
```

**安全脚本里永远优先用列表。**

### 封装一个 run 函数

```python
import subprocess

def run(cmd):
    result = subprocess.run(cmd, capture_output=True, text=True)
    if result.returncode != 0:
        print(f"失败: {result.stderr}")
        return None
    return result.stdout
```

**检查 `returncode` 是关键**：命令执行了不等于成功。

### 跨平台的坑

`ls` 是 Linux 命令，Windows 上没有：

```python
subprocess.run(["ls", "-l"])    # Windows 报 FileNotFoundError
```

Windows 的等价命令：

```python
subprocess.run(["cmd", "/c", "dir"])
```

写跨平台工具要判断：

```python
import platform

if platform.system() == "Windows":
    cmd = ["cmd", "/c", "dir"]
else:
    cmd = ["ls", "-l"]
```

**但安全工具基本只在 Linux 跑，优先把 Linux 用熟。**

---

## 二、socket：端口探测

### 完整代码

```python
import socket

def check(host, port, timeout=0.5):
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.settimeout(timeout)
    try:
        s.connect((host, port))
        return True
    except (socket.timeout, ConnectionRefusedError):
        return False
    finally:
        s.close()

targets = [(22, "ssh"), (80, "http"), (443, "https"), (3306, "mysql")]

for port, name in targets:
    status = "开放" if check("127.0.0.1", port) else "关闭"
    print(f"127.0.0.1:{port:<5} {name:<6} {status}")
```

### 三个关键点

**1. 必须设超时**

```python
s.settimeout(0.5)
```

不设超时，连一个不响应的 IP 会卡几十秒（系统的 TCP 超时）。

**2. 连接失败是“预期内的正常情况”**

```python
except (socket.timeout, ConnectionRefusedError):
    return False
```

端口关闭不是“程序出错”，是探测结果。所以这里用 `except` 处理，而不是让它崩溃。

**3. 必须 close**

```python
finally:
    s.close()
```

无论成功失败都关闭 socket，否则大量探测时会耗尽文件描述符。

### connect 和 connect_ex

```python
# 方式 1：connect 抛异常
try:
    s.connect((host, port))
    ok = True
except:
    ok = False

# 方式 2：connect_ex 返回码
code = s.connect_ex((host, port))
ok = (code == 0)
```

`connect_ex` 不抛异常，返回 0 表示成功，代码更短。

### 和 nmap 的关系

Nmap 就是把这个 20 行的逻辑做成了工业产品：

| 你的代码 | nmap 的增强 |
|---|---|
| 单线程串行 | 并发数千端口 |
| 固定超时 | 自适应超时策略 |
| 只知道开/关 | 服务识别（`-sV`） |
| 无 | 操作系统指纹（`-O`） |

**你写的这 40 行，就是 nmap 的核心逻辑。**

---

## 三、Bandit 25：分页器逃逸

### 题目

- 主目录有 `bandit26.sshkey`
- bandit26 的登录 shell 是 `/usr/bin/showtext`（一个分页器）
- 登录后立刻退出，不给 bash

### 思路

`showtext` 用 `more` 显示一段文本，然后退出。

如果**文本比窗口高**，`more` 会进入分页模式，等待按键。

### 步骤

1. 下载私钥到本地

```powershell
scp -P 2220 bandit25@bandit.labs.overthewire.org:bandit26.sshkey C:\Users\hester\key26
```

2. **把终端窗口拖到只有 5–6 行高**

3. 登录

```powershell
ssh -i C:\Users\hester\key26 bandit26@bandit.labs.overthewire.org -p 2220
```

4. 看到 `--More--` 时按 **`v`**

`more` 会调用 `vi` 打开当前文件。

5. 在 vim 里

```
:e /etc/bandit_pass/bandit26
```

读到密码。

6. 拿 shell

```
:set shell=/bin/bash
:shell
```

---

## 四、Bandit 26：vim 里拿 shell

拿到 shell 后：

```bash
./bandit27-do cat /etc/bandit_pass/bandit27
```

`bandit27-do` 是 setuid 程序，能以 bandit27 身份执行命令。

### 这关的核心：受限 shell 不是安全边界

| 程序 | 逃逸方式 |
|---|---|
| `more` / `less` | 按 `v` 进 vim |
| `vi` / `vim` | `:!bash` 或 `:set shell=/bin/bash` + `:shell` |
| `man` | `!命令` |
| `python` | `import os; os.system("bash")` |
| `awk` | `system("bash")` |
| `find` | `-exec bash \;` |

**原理**：只要这个“受限程序”能**执行命令、打开文件、调起其他程序**，就一定能绕过。

**现实中的对应**：管理员用 `rbash` 限制用户 shell，但如果 PATH 里还有能执行命令的程序，往往就能逃出来。所以真正的限制得靠 **SELinux / AppArmor / 容器**这类内核级机制。

---

## 五、踩坑记录

| 错误 | 原因 | 正确 |
|---|---|---|
| `subprocess.run(["ls"])` 报错 | Windows 没有 `ls` | 换 Ubuntu，或用 `["cmd","/c","dir"]` |
| nano 里多粘了 `bash`、`python3 run.py` | 复制时带上了代码块外的内容 | 只选代码块内容，粘完检查末尾 |
| `cd ~/security/scripts` 报错 | 目录不存在 | 先 `mkdir -p` |
| 端口探测全“关闭” | Ubuntu 没开这些服务 | 正常；起个 8000 端口验证“开放”分支 |

---

## 六、自测

1. `os.path.join` 比手写字符串拼接好在哪？
2. `sys.argv[0]` 是什么？参数从哪开始？
3. `subprocess.run(["ls","-l"])` 和 `subprocess.run("ls -l", shell=True)` 有什么区别？哪个更安全？
4. 为什么 socket 一定要 `settimeout`？
5. 为什么 socket 一定要 `close`？放在哪个块里？
6. `more` 怎么变成 shell 的？
7. 为什么说“受限 shell 不是安全边界”？

---

## 七、明天

- Git 基础（`clone` / `log` / `diff` / `show`）
- Bandit 27–28（git 历史泄露）
- Python：读写 git 输出，做一个小分析脚本
