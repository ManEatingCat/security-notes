# 2026-11-19　第一个真实漏洞：SQL 注入

> 自己搭靶场，自己打穿，自己修复。

---

## 一、HTTP 请求长什么样

浏览器点“登录”时，发出去的是一段**纯文本**：

```http
POST /login HTTP/1.1
Host: 127.0.0.1:5000
Content-Type: application/x-www-form-urlencoded

username=admin&password=S3cr3tP@ss
```

| 部分 | 含义 |
|---|---|
| `POST /login` | 方法 + 路径 |
| `Host` | 目标主机 |
| 空行之前 | 请求头 |
| 空行之后 | **请求体**（表单数据） |

服务端返回：

```http
HTTP/1.1 200 OK
Content-Type: text/html

<h1>欢迎 admin</h1>
```

**所有 Web 漏洞，本质都是“这段文本被服务端怎么处理”。**

---

## 二、搭靶场

### 环境

- Ubuntu 虚拟机
- Flask（`sudo apt install -y python3-flask`）
- SQLite（Python 内置）

### 建数据库

```python
import sqlite3

conn = sqlite3.connect("users.db")
cur = conn.cursor()
cur.execute("DROP TABLE IF EXISTS users")
cur.execute("CREATE TABLE users (id INTEGER, username TEXT, password TEXT)")
cur.execute("INSERT INTO users VALUES (1, 'admin', 'S3cr3tP@ss')")
cur.execute("INSERT INTO users VALUES (2, 'guest', 'guest123')")
conn.commit()
conn.close()
```

### 有漏洞的登录（漏洞版）

```python
from flask import Flask, request
import sqlite3

app = Flask(__name__)

@app.route("/login", methods=["GET", "POST"])
def login():
    if request.method == "POST":
        username = request.form["username"]
        password = request.form["password"]

        conn = sqlite3.connect("users.db")
        cur = conn.cursor()

        # ⚠️ 漏洞：字符串拼接
        query = f"SELECT * FROM users WHERE username='{username}' AND password='{password}'"
        print("执行的 SQL:", query)

        cur.execute(query)
        row = cur.fetchone()
        conn.close()

        if row:
            return f"<h1>欢迎 {row[1]}</h1>"
        return "<h1>登录失败</h1>"

    return """
    <form method="post">
      用户名: <input name="username"><br>
      密码: <input name="password"><br>
      <button>登录</button>
    </form>
    """

app.run(host="127.0.0.1", port=5000)
```

---

## 三、攻击

### 正常登录

用户名 `admin`，密码 `S3cr3tP@ss`。

终端打印：

```sql
SELECT * FROM users WHERE username='admin' AND password='S3cr3tP@ss'
```

### 攻击 1：绕过密码

用户名填：

```
' OR '1'='1' --
```

密码随便。

终端打印：

```sql
SELECT * FROM users WHERE username='' OR '1'='1' --' AND password='x'
```

**登录成功。**

### 拆解

| 你输入的部分 | 作用 |
|---|---|
| `'` | 闭合 `username='` 的引号 |
| `OR '1'='1'` | 加一个永远为真的条件 |
| `--` | 注释掉后面所有内容 |

WHERE 条件变成：

```sql
username='' OR '1'='1'
```

永远成立，数据库返回第一行（admin）。

### 攻击 2：直接登录管理员

用户名：

```
admin' --
```

SQL：

```sql
SELECT * FROM users WHERE username='admin' --' AND password='x'
```

密码条件被注释掉，直接匹配 admin。

### 核心结论

> **你的输入从“数据”变成了“代码”。**

数据库看到的是两段字符串拼在一起，它没法分辨哪部分是开发者写的、哪部分是你写的。

---

## 四、修复：参数化查询

### 改动

```python
# 漏洞版
query = f"SELECT * FROM users WHERE username='{username}' AND password='{password}'"
cur.execute(query)

# 安全版
query = "SELECT * FROM users WHERE username=? AND password=?"
cur.execute(query, (username, password))
```

### 为什么能防住

**参数化查询的执行过程**：

1. 数据库先收到 SQL **结构**：`SELECT * FROM users WHERE username=? AND password=?`
2. 数据库解析结构，确定“`?` 是一个值的位置”
3. 你的输入作为**纯数据**填进那个位置

**输入永远只能是“值”，不可能变成“语法”。**

### 对比

| | 漏洞版 | 安全版 |
|---|---|---|
| SQL 字符串里 | 有用户输入 | 只有 `?` |
| 输入什么时候进来 | 拼字符串时 | 执行时单独传 |
| 数据库怎么理解 | 分不清语法和数据 | 先定结构，再填值 |
| 能否注入 | 能 | 不能 |

### 一个直观的判断方法

修复后，终端打印：

```
执行的 SQL: SELECT * FROM users WHERE username=? AND password=?
```

**看不到你的输入了。**

而漏洞版能看到完整输入：

```
执行的 SQL: SELECT * FROM users WHERE username='admin' AND password='S3cr3tP@ss'
```

**“用户输入是否出现在 SQL 字符串里”，就是判断注入点的最直观方式。**

---

## 五、这不只是 SQL 的问题

同一个模式，出现在所有地方：

| 场景 | 漏洞写法 | 安全写法 |
|---|---|---|
| SQL | `f"... '{name}'"` | `execute(sql, (name,))` |
| Shell | `f"cat {file}"` + `shell=True` | `["cat", file]` |
| HTML | `f"<h1>{comment}</h1>"` | 模板引擎自动转义 |
| 日志 | `f"login: {user}"` | 结构化日志 |

**共同点：用户输入进入了“会被解释执行”的位置。**

### 防御的三层

| 层 | 做法 | 可靠性 |
|---|---|---|
| 1 | **参数化 / 转义**（让输入只能是数据） | 高 |
| 2 | 白名单校验（只允许特定字符/值） | 中 |
| 3 | 过滤黑名单（禁止 `'`、`--`） | **低，别用** |

**黑名单为什么不可靠**：攻击者总能用编码、大小写、注释变体绕过。参数化是从机制上不可能，不是“挡住”。

---

## 六、踩坑记录

| 问题 | 原因 | 解决 |
|---|---|---|
| Flask 服务总是停 | 按了 `Ctrl + C` | 启动后手离开键盘 |
| `python3 app.py` 找不到文件 | 在 home 目录，不在 lab | `cd ~/security/lab` 或看提示符 |
| 浏览器“无法连接” | 服务被停了 | 看终端有没有 `$` 提示符 |
| `nano app.py` 显示 `[ 新文件 ]` | 目录错了 | 先 `ls` 确认文件存在 |
| 改代码后没生效 | 没重启 Flask | `Ctrl+C` → 改 → 重新启动 |

---

## 七、自测

1. 一个 HTTP 请求由哪几部分组成？请求体在哪？
2. `' OR '1'='1' --` 的三个部分分别起什么作用？
3. 为什么参数化查询能防住注入？它“过滤”了输入吗？
4. 怎么从终端输出判断一个登录接口有没有注入点？
5. `shell=True` 和 SQL 拼接，为什么是同一类问题？
6. 为什么黑名单过滤不可靠？

---

## 八、明天

- 把靶场扩展：加一个“搜索”功能，再注入一次
- 学 XSS（跨站脚本），对比“注入到 SQL”和“注入到 HTML”的区别
- PortSwigger 的第一个实验
