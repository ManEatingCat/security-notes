# 2026-11-04　Linux 进程管理 + 编码与压缩 + Bandit 10–12

> 记录方法，不记录密码。

---

## 一、Linux 进程管理

### ps：看进程

```bash
ps aux
```

三个字母：

| 字母 | 作用 |
|---|---|
| `a` | 所有用户的进程 |
| `u` | 用户友好格式（带 CPU、内存） |
| `x` | 包含没有终端的进程 |

### 输出怎么读

```
USER   PID  %CPU %MEM    VSZ   RSS TTY  STAT START   TIME COMMAND
root     1   1.1  0.3  23180 14504 ?    Ss   17:34   0:01 /sbin/init
```

| 列 | 含义 | 关注点 |
|---|---|---|
| `USER` | 谁启动的 | 普通用户跑 root 进程，可疑 |
| `PID` | 进程 ID | `kill` 用 |
| `%CPU` | CPU 占用 | 长期 90%+ 可能是挖矿 |
| `%MEM` | 内存占用 | 异常高可能是泄漏 |
| `TTY` | 关联终端 | `?` = 后台服务 |
| `STAT` | 状态 | 见下表 |
| `START` | 启动时间 | 凌晨启动的陌生进程要注意 |
| `COMMAND` | 命令 | 看它到底在跑什么 |

**STAT 状态码**：

| 码 | 含义 |
|---|---|
| `R` | 运行中 |
| `S` | 睡眠（最常见） |
| `D` | 不可中断睡眠（等 IO） |
| `T` | 停止（Ctrl+Z） |
| `Z` | 僵尸进程 |

### 常用组合

```bash
ps aux | grep 关键词                  # 找进程
ps aux --sort=-%cpu | head -10        # CPU 占用前十
ps aux | grep 关键词 | grep -v grep   # 排除 grep 自己
pgrep -a 进程名                       # 更干净的写法
```

为什么 `grep` 会匹配到自己：`grep 关键词` 这条命令本身的命令行里就含有关键词。

### 后台运行 &

```bash
sleep 300 &
```

执行后显示：

```
[1] 2974
```

- `[1]` 是**作业号**，shell 内部编号
- `2974` 是 **PID**，内核编号

两者不是一回事：`jobs` 看作业，`ps` 看 PID。

### kill：发信号，不是"杀死"

`kill` 的本质是**给进程发一个信号**。

| 信号 | 编号 | 行为 |
|---|---|---|
| `SIGTERM` | 15（默认） | 请进程自行退出，**可捕获、可忽略** |
| `SIGKILL` | 9 | 内核强制终止，**无法拒绝** |
| `SIGINT` | 2 | 相当于 Ctrl + C |
| `SIGHUP` | 1 | 让服务重读配置 |

**正确顺序**：先 `kill PID`，等几秒没反应，再 `kill -9 PID`。

原因：`SIGTERM` 让程序有机会保存数据、关闭文件；`SIGKILL` 直接斩断，可能留下损坏文件。

```bash
kill 2974
kill -9 2974
pkill 进程名        # 按名字杀
killall 进程名      # 杀所有同名
```

### htop：可视化的进程管理

```bash
sudo apt install -y htop
htop
```

界面信息：

| 项 | 含义 |
|---|---|
| `Load average` | 1/5/15 分钟平均负载，低于核数说明不忙 |
| `Mem` | 物理内存使用 |
| `Swp` | 交换分区使用，长期大于 0 说明内存紧张 |

快捷键：

| 键 | 作用 |
|---|---|
| `F3` | 搜索 |
| `F4` | 过滤 |
| `F5` | 树状视图（看父子进程，排查恶意进程很有用） |
| `F6` | 排序 |
| `F9` | 杀进程 |
| `q` | 退出 |

### apt：包管理

| 命令 | 作用 |
|---|---|
| `apt update` | 刷新"有哪些新版本"的**清单** |
| `apt upgrade` | 按清单**真正安装**新版本 |
| `apt install 包名` | 安装 |
| `apt remove 包名` | 卸载 |
| `apt search 关键词` | 搜索 |

⚠️ **`update` ≠ `upgrade`**。`update` 只同步索引，不动任何已安装软件。

### && 和 ||

```bash
命令A && 命令B    # A 成功才执行 B
命令A || 命令B    # A 失败才执行 B
```

---

## 二、编码、压缩与解压

### 三种完全不同的东西

| 类型 | 例子 | 特点 |
|---|---|---|
| **编码** | base64、URL 编码 | 谁都能还原，不需要密钥 |
| **加密** | AES、RSA | 需要密钥才能还原 |
| **压缩** | gzip、bzip2、tar | 减小体积，打包文件 |

**base64 是编码，不是加密。** 看到它别以为"加密了"，它就是明文换了件外套。

### base64

```bash
base64 -d 文件        # 解码
echo "hello" | base64  # 编码
```

特征：

- 字符集 `A-Z a-z 0-9 + /`
- 结尾常见 `=` 或 `==` 填充
- 长度通常是 4 的倍数

`VGhlIHBhc3N3b3JkIGlz` 解码就是 `The password is`。

### ROT13

字母表移动 13 位：

```
A→N  B→O  ...  M→Z  N→A  ...  Z→M
```

```bash
tr 'A-Za-z' 'N-ZA-Mn-za-m' < 文件
```

**同一个命令既能加密也能解密**，因为移 13 位两次等于没动。

`Gur cnffjbeq vf` → `The password is`。

### tr 的用法

```bash
tr '源字符集' '目标字符集'
```

按位置一一对应替换：

```
tr 'A-Za-z' 'N-ZA-Mn-za-m'
    A-M 对应 N-Z
    N-Z 对应 A-M
    小写同理
```

### hexdump 与 xxd

```bash
xxd data.txt          # 查看 hex dump
xxd -r data.txt > out # 还原成二进制
```

hex dump 的格式：左边是偏移量，中间是十六进制字节，右边是 ASCII 显示。

`-r` 是 reverse，把文本 hex 还原成字节。

### 多层压缩的判断方法

**每一层都先 `file`，再决定用什么命令。**

| `file` 输出 | 解压命令 |
|---|---|
| `gzip compressed data` | `mv x x.gz && gunzip x.gz` |
| `bzip2 compressed data` | `mv x x.bz2 && bunzip2 x.bz2` |
| `POSIX tar archive` | `tar xf x` |
| `ASCII text` | 完成，`cat` |

**关键**：文件类型由**文件头部的魔数**决定，不是扩展名。改后缀名不会改变类型。

| 格式 | 魔数 |
|---|---|
| gzip | `1f 8b` |
| bzip2 | `42 5a 68`（`BZh`） |
| zip | `50 4b`（`PK`） |

---

## 三、Bandit 10–12

### 第 10 关：strings + grep + base64

```bash
strings data.txt | grep "="
```

输出一串以 `==` 结尾的字符串——这是 base64。

```bash
strings data.txt | grep "=" | base64 -d
```

得到 `The password is xxx`。

**为什么用 `strings`**：文件里混着二进制，直接 `cat` 会乱码。

### 第 11 关：ROT13

```bash
cat data.txt
```

输出 `Gur cnffjbeq vf ...`——看着像英语但读不通，这是 ROT13 的典型特征。

```bash
tr 'A-Za-z' 'N-ZA-Mn-za-m' < data.txt
```

**教训**：`base64 -d` 报 `invalid input`，不是命令错了，是**编码类型判断错了**。先 `cat` 看一眼再决定。

### 第 12 关：hex → 多层压缩链

```bash
mkdir -p /tmp/work && cd /tmp/work
xxd -r ~/data.txt > data.bin
file data.bin
```

然后循环：

```
file 判断类型 → 改名 → 解压 → 再 file
```

实际路径：

```
hex → gzip → bzip2 → gzip → tar → tar → bzip2 → tar → ASCII text
```

最终解出一个 49 字节的文本文件，里面是密码。

**判断技巧**：文件大小是线索。49 字节不可能是压缩包，只可能是文本。

### 自动解压脚本（修正版）

第一次写的脚本有个严重 bug：每轮 `for f in *` 会把所有文件重新处理一遍，tar 包被反复解压，一直在原地打转。

修正版：

```bash
for i in $(seq 1 20); do
  echo "--- 第 $i 层 ---"
  for f in *; do
    case "$(file -b "$f")" in
      *gzip*)  mv "$f" "$f.gz"  && gunzip "$f.gz"  && break ;;
      *bzip2*) mv "$f" "$f.bz2" && bunzip2 "$f.bz2" && break ;;
      *tar*)   tar xf "$f" && rm -f "$f" && break ;;
      *text*)  echo "找到：$f"; cat "$f"; exit 0 ;;
    esac
  done
done
```

三个关键点：

1. `&& break`：解完一个就跳出，不再碰其他文件
2. `rm -f "$f"`：tar 解完删掉原包，防止重复
3. 每轮重新遍历：保证处理的是最新文件

---

## 四、踩坑记录

| 错误 | 原因 | 正确 |
|---|---|---|
| `kill <user>` | `< >` 是占位符，不是输入 | `kill 2974` |
| `qpt install` | `apt` 打成了 `qpt` | `apt` |
| 在 shell 里按 `q` | `q` 只在 htop 里有效 | 先 `htop` 再按 `q` |
| `base64 -d` 报 invalid input | 文件其实是 ROT13 | 先 `cat` 判断类型 |
| 解压脚本空转 15 轮 | 没 `break`、没删原文件 | 加 `break` 和 `rm` |
| `kill -9` 报"没有那个进程" | 上一条 `kill` 已经终止了 | 正常现象 |

---

## 五、自测

1. `ps aux` 里 `a`、`u`、`x` 各是什么意思？
2. `kill` 默认发哪个信号？和 `-9` 有什么区别？为什么先试普通 `kill`？
3. 作业号和 PID 有什么区别？
4. `apt update` 和 `apt upgrade` 的区别？
5. base64 是编码还是加密？为什么？
6. 怎么判断一个文件是 gzip 还是 bzip2？
7. `xxd -r` 的 `-r` 是什么意思？

---

## 明天

- Python：函数（`def`、参数、返回值）
- Bandit 13–15
- 把自动解压脚本理解透，能自己写出来
