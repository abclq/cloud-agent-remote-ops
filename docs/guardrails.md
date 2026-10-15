# 🛡 护栏设计：拦什么、为什么这么拦

> `exec` 是唯一接受参数的动作 —— 这一页就是它的护栏。

---

## 三道护栏

```
命令进来
   │
   ├─ ① 元字符检查 ───── 有 ; | & < > ` $ ( ) { } 就拒
   │
   ├─ ② 危险模式黑名单 ── 不可逆 / 影响整机 / 改身份 / 改登录
   │
   ├─ ③ 凭据关键字 ───── .ssh / .hermes/shared / id_* / .env / token …
   │
   └─ 通过 → 以「那个用户」身份执行 + 记审计
```

**顺序有意义**：元字符先拦（能拼出任意东西），再拦模式，最后拦凭据。

---

## ① 元字符 —— 全部拒绝

```bash
; | & < > ` $ ( ) { }
```

**为什么这么严**：`exec` 是**单条命令**的语义。一旦允许元字符：

```bash
# 本来只想看日志
exec journalctl -n 5
# 变成
exec journalctl -n 5; rm -rf ~/important
# 或者
exec echo $(cat ~/.ssh/id_ed25519)
```

元字符是"从单条命令升级到任意脚本"的跳板。**全拒是最省心的选择。**

**代价**：`exec "a && b"` 这种多步命令用不了。

**补偿**：让它写进工作区再跑：

```bash
exec echo 'step1; step2; step3' > ~/agent-workspace/job.sh && bash ~/agent-workspace/job.sh
```

⚠️ 注意上面这条**本身**含 `;`（在引号里）—— 所以实践中元字符检查要**先去掉引号内容再检查**，
或者干脆规定「要跑多步就写文件」。**两种做法各有取舍，看你愿意多复杂。**

---

## ② 危险模式黑名单

| 类别 | 拦什么 | 为什么 |
|---|---|---|
| **不可逆** | `rm -rf /`、`rm -rf ~`、`mkfs`、`dd of=/dev/` | 执行完没法回滚 |
| **影响整机** | `systemctl reboot` `poweroff` `halt` `suspend` `hibernate` `kexec` | 会断掉你和它的连接，也可能断掉别人的服务 |
| **停服务** | `systemctl … stop` `disable` `mask` `kill` | **修复只该用 `restart`/`start`** —— 停掉网关就把通信掐断了 |
| **杀进程** | `pkill`、`killall`、`kill <pid>`、`kill -1`、`killall5` | 广谱杀进程；修服务走 `systemctl restart` |
| **改身份** | `sudo`、`su`、`passwd`、`chsh`、`usermod` | 提权链的第一跳 |
| **改登录** | 写 `authorized_keys`、改 `sshd_config` | **它能给自己加钥匙** = 绕过所有护栏 |
| **改护栏自己** | 写 `/usr/local/bin/agent-entry*` | 直接失效 |

### 踩过的坑 ①：`systemctl reboot` 漏网

最初的黑名单只写了 `reboot`（裸命令），忘了 `systemctl reboot`。

**实测发现**：`exec systemctl reboot` 能通过 → **能重启整机** 🔴

补上：

```bash
systemctl\s+(reboot|poweroff|halt|suspend|hibernate|kexec)
kill\s+-1\b
killall5
```

### 踩过的坑 ②：`systemctl stop` 也漏了

补完 `reboot` 之后，又发现 `exec systemctl --user stop <网关>` **是放行的** 🔴

（那次只是碰巧没生效 —— 写的是 `--user`，而网关是系统级单元。换个写法就真停掉了。）

**风险**：对面 AI 只要 `stop` 一下网关，**我们之间的通信就断了** ——
而它自己可能还以为在"修复"。

补上：

```bash
systemctl[^|;&]*\s(stop|disable|mask|kill)\b
(^|[;&|\s])(pkill|killall|kill)\s
```

**口径**：**修复只该用 `restart` / `start`**，不该有"停"这个动作。
`disable` / `mask` 更狠 —— 停了还回不来。

### 两条教训

1. **黑名单要按「语义」列，不是按「字面命令名」列。**
   凡是"能达到这个效果"的写法都要覆盖。
2. **要按「破坏力」想，不是按「名字像不像」想。**
   `restart` 和 `stop` 名字长得像，破坏力天差地别。

---

## ③ 凭据关键字

拦这些路径/关键字：

```
.ssh/          id_rsa  id_ed25519  id_*
.hermes/shared/
.env           .env.*
token  password  secret  passwd
shadow
```

**目标**：防止"护栏全过了，但它顺手 `cat` 走了你的私钥"。

---

## ⚠️ 诚实说明：第 ③ 道是"尽力而为"，不是硬保证

**这是本项目最需要你知道的一条。**

凭据拦截是**关键字匹配** —— 它挡得住：

```bash
exec cat ~/.ssh/id_ed25519        # ⛔ 命中 .ssh/
exec cat /etc/shadow              # ⛔ 命中 shadow
```

它挡不住：

```bash
exec cp ~/.ssh/id_ed25519 /tmp/a && cat /tmp/a    # 元字符会拦 ; 和 &&，但如果它分两次…
exec tar czf /tmp/x.tgz ~/.ssh                    # 不含关键字
exec find / -name "id_*" -exec cat {} \;          # 元字符会拦
exec base64 ~/.ssh/id_ed25519 | head              # 管道会拦，但换写法…
```

**根本问题**：你在用**字符串匹配**去拦**语义**。只要有足够的创造性，总能绕。

### 三条缓解

1. **元字符全拒** —— 挡掉大部分组合技（这是最有价值的一道）
2. **提权链审查** —— 就算它读到了凭据，也升不到 root（见 [提权链审查](privilege-audit.md)）
3. **假设私钥已泄露** —— 从[威胁模型](threat-model.md)出发：那把钥匙本来就不该是"秘密"

**换句话说：凭据护栏是"减少误伤"，不是"安全边界"。安全边界是权限本身。**

---

## 更硬的做法（我们还没上，但值得）

### ① AST 解析代替正则

用 `tree-sitter` 把命令解析成语法树，再按**语义**判断，而不是匹配字符串。

- 能识别"这是在改 authorized_keys"（哪怕写成 `>> ~/.ssh/auth*keys`）
- 能识别变量展开、命令替换

**代价**：要引入解析器、要维护规则。但绕过难度高一个量级。

### ② 凭据占位符注入

不让 AI 看到真值，让它写占位符：

```bash
exec curl -H "Authorization: Bearer {{API_TOKEN}}" https://...
```

**真值在执行那一刻才替换** → AI 从来看不到 token，日志里也不会落明文。

**这比"拦凭据路径"干净得多** —— 因为 AI 就算读到命令全文，也没有秘密。

### ③ 受限 shell

用 `rbash` / `lshell` / `git-shell` 这类受限 shell 做整个会话的边界，
比在脚本里做字符串匹配更结构化。

### ④ 内核级隔离

`bubblewrap` / `firejail` / systemd 的 `DynamicUser` + `ProtectHome` / 容器。

**最强，但最重** —— 适合你完全不信对面、且故障处理不需要碰真实系统的场景。

---

## 实测：拦得准、不误伤

我们做过端到端测试（写脚本自动跑，危险命令**只做正则验证、绝不真执行**）：

| 用例 | 期望 | 结果 |
|---|---|---|
| `rm -rf /` | 拦 | ⛔ ✅ |
| `cat ~/.ssh/id_ed25519` | 拦 | ⛔ ✅ |
| `sudo passwd nobody` | 拦 | ⛔ ✅ |
| `systemctl reboot` | 拦 | ⛔ ✅ |
| `echo a; rm -rf /tmp/x` | 拦（元字符） | ⛔ ✅ |
| `kill -1 1` | 拦 | ⛔ ✅ |
| `journalctl -u x -n 50` | 放行 | ✅ |
| `systemctl --user restart x` | 放行 | ✅ |
| `df -h` | 放行 | ✅ |
| `systemctl --user is-active x` | 放行 | ✅ |
| `bash ~/agent-workspace/x.sh` | 放行 | ✅ |

**9/9 危险拦 + 5/5 正常放行 = 零误伤。**

**为什么"零误伤"重要**：误拦会让 AI 觉得"这工具不好用"，
然后**开始想绕** —— 这才是护栏真正的敌人。

---

## 设计原则

1. **默认拒绝** —— 没在白名单里的都拒，不是"没在黑名单里的都放"
2. **按语义列黑名单** —— `systemctl reboot` 和 `reboot` 是一个东西
3. **失败要说话** —— 被拦返回原因，让 AI 知道换个做法
4. **护栏不归被约束者** —— 脚本必须 root 拥有（见 [架构](architecture.md)）
5. **诚实标注弱点** —— 凭据拦截是"尽力而为"，别当硬保证
6. **零误伤优先于多拦** —— 误拦会诱发绕过

---

## 下一步

- [⚠️ 提权链审查](privilege-audit.md) —— 就算护栏被绕过，最坏到哪
- [架构](architecture.md) —— 护栏为什么不归它所有
- [踩过的坑](pitfalls.md) —— 黑名单漏网的完整记录
