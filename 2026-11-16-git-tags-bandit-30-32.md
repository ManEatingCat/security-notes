# 2026-11-16（实际完成于 11-17）Git 进阶 + Bandit 30–32 + OWASP 概览

> 记录方法，不记录密码。

---

## 一、Git 里藏东西的四个位置

到这一关为止，四个位置全见过了：

| 位置 | 命令 | 对应关卡 |
|---|---|---|
| 文件内容 | `cat 文件` | 27 |
| 提交历史 | `git log -p` | 28 |
| 分支 | `git branch -a` + `git checkout` | 29 |
| **标签** | `git tag` + `git show <标签>` | 30 |

**排查一个陌生仓库时，这四个位置一个都不能漏。**

### 标签（tag）是什么

标签是给某个提交打的**记号**，常用于标记版本（`v1.0.0`）。

```bash
git tag              # 列出所有标签
git show v1.0.0      # 看标签指向的内容
git tag -a v2.0 -m "说明"   # 创建带注释的标签
```

标签可以附在提交上，也可以**单独保存一段内容**（比如 release notes）。所以里面可能藏着没进代码库的东西。

---

## 二、Git 推送与 pre-receive hook

### `git push` 是什么

把本地的提交传到远程仓库。

```bash
git push origin master
```

### 为什么会被拒绝

```
! [remote rejected] master -> master (pre-receive hook declined)
```

**`pre-receive hook`** 是服务端的一个脚本，在接收推送**之前**执行，可以检查内容、拒绝推送。

常见用途：

- 检查提交信息格式
- 禁止提交大文件
- 检查是否包含密钥
- 权限校验

**安全意义**：hook 是**服务端代码**，能执行任意逻辑。如果你能控制它，或者它输出调试信息，就可能泄露数据。

Bandit 31 的密码，就是 hook 在校验内容时直接打印出来的——**它拒绝了推送，但已经把答案说了**。

### 完整流程

```bash
git status              # 先看当前状态
git add -f key.txt      # 暂存（-f 强制，因为 .gitignore 忽略了 *.txt）
git commit -m "..."     # 提交
git push origin master  # 推送
```

**如果 commit 报“无法自动探测邮件地址”**：

```bash
git config --global user.name "你的名字"
git config --global user.email "你的邮箱"
```

---

## 三、Bandit 32：大写 shell 逃逸

### 现象

登录后进入：

```
WELCOME TO THE UPPERCASE SHELL
>>
```

输入的所有字母被转成大写，`ls` 变成 `LS`，无法执行。

### 解法

```
$0
```

`$0` 是 shell 的**内置变量**，表示“当前 shell 程序”。变量名不受大小写转换影响，执行后启动一个新的 bash。

然后：

```bash
ls
cat /etc/bandit_pass/bandit33
```

### 原理：过滤器只作用于它看得见的东西

服务端把输入转大写，以为能阻止小写命令。但它没考虑到：

| 绕过点 | 例子 |
|---|---|
| 变量 | `$0`、`$SHELL`、`${PATH}` |
| 环境变量 | `$HOME` |
| 特殊字符 | `/`、`$`、`*`、`\` |
| 编码 | `$(printf '\x6c\x73')` |

**只要解释器还在，总能找到它“没过滤到”的路径。**

### 正确的防御思路

不是“过滤输入”，而是**不给解释器**：

- 白名单：只允许特定几个命令，其他一律拒绝
- 在受限容器里执行
- 用专门的受限 shell（`rbash`）并锁死 PATH

**“过滤”永远会漏，因为攻击面是解释器的全部能力，而过滤器只能想到有限的几种。**

---

## 四、OWASP Top 10（2021）

### 全表

| 编号 | 名称 | 一句话 |
|---|---|---|
| A01 | 访问控制失效 | 该管的没管住，越权 |
| A02 | 加密机制失效 | 明文传输、弱加密、硬编码密钥 |
| A03 | **注入** | SQL 注入、命令注入、XSS |
| A04 | 不安全设计 | 架构层面就没考虑安全 |
| A05 | 安全配置错误 | 默认密码、调试模式、目录可列 |
| A06 | 过时组件 | 用了有已知漏洞的旧版本 |
| A07 | 认证失效 | 弱密码、无锁定、会话固定 |
| A08 | 完整性失效 | 不验证签名、反序列化 |
| A09 | 日志监控失效 | 被打了不知道 |
| A10 | SSRF | 让服务器替你去访问内网 |

### 三个重点

**A01 访问控制失效**

最常见的漏洞，也最容易上手。

```bash
# 把 URL 里的 id 改一下，看能不能看到别人的数据
/api/user/1001 → /api/user/1002
```

- **水平越权**：同级别用户之间互相访问
- **垂直越权**：普通用户访问管理员功能

**核心问题**：服务端只校验“你登录了吗”，没校验“你有权限看这条数据吗”。

**A03 注入**

你写 `subprocess` 时已经接触过命令注入：

```python
subprocess.run(f"cat {user_input}", shell=True)    # 危险
subprocess.run(["cat", user_input])                # 安全
```

SQL 注入是同一个思路，只是注入的是 SQL 语句：

```python
# 危险
cursor.execute(f"SELECT * FROM users WHERE name = '{name}'")
# 安全
cursor.execute("SELECT * FROM users WHERE name = ?", (name,))
```

**A07 认证失效**

爆破、弱口令、会话固定。

Bandit 24 的 PIN 爆破就是这一类的入门演练。

### 和已经学过的内容对应

| OWASP 项 | 你已经接触过的 |
|---|---|
| A01 访问控制 | setuid 提权（Bandit 19） |
| A03 注入 | `shell=True` 命令注入 |
| A05 配置错误 | cron 脚本写密码到可读路径 |
| A07 认证 | PIN 爆破 |
| A09 日志监控 | 还没接触 |

**你已经在无意识地实践 OWASP 了。** 接下来要做的是把它们系统化。

---

## 五、踩坑记录

| 错误 | 原因 | 正确 |
|---|---|---|
| `git clone ... repo` 报“目录已存在” | 上一关的 `repo` 还在 | 每关单独目录，或指定不同名字 |
| `git commit` 报“无法探测邮件地址” | 没配 `user.email` | `git config --global user.email` |
| `git push` 说 `Everything up-to-date` | commit 失败了，没东西可推 | 先 `git status` 查状态 |
| `git add key.txt` 没反应 | `.gitignore` 里有 `*.txt` | `git add -f` |
| `cd ~/security` 在 Windows 报错 | `~` 是 Linux 的 home | 去 Ubuntu 里跑 |

---

## 六、自测

1. Git 里藏东西的四个位置是哪些？分别用什么命令查？
2. `git tag` 和 `git branch` 有什么区别？
3. `pre-receive hook` 在哪台机器上执行？能做什么？
4. `$0` 为什么能绕过大小写限制？
5. 为什么“过滤输入”不是好的防御思路？应该怎么做？
6. OWASP Top 10 里，你已经实践过哪几项？

---

## 七、Bandit 主线小结

从 0 到 32，覆盖了：

| 类别 | 关卡 |
|---|---|
| 文件操作 | 0–6 |
| 编码与压缩 | 7–12 |
| 认证与网络 | 13–16 |
| 差异与配置 | 17–22 |
| 定时任务与脚本 | 23–24 |
| 受限环境逃逸 | 25–26, 32 |
| Git 信息泄露 | 27–31 |

**接下来该换战场了**：从“做靶场”转向“理解真实漏洞”。
