# 🤖 cloud-agent-remote-ops

**中文** | [English](README.en.md) · `MIT` · `锁死入口 + 具名动作 + 带护栏的「亲自修」`

[![stars](https://img.shields.io/github/stars/abclq/cloud-agent-remote-ops?style=flat-square&label=%E2%98%85)](https://github.com/abclq/cloud-agent-remote-ops/stargazers)
[![last-commit](https://img.shields.io/github/last-commit/abclq/cloud-agent-remote-ops?style=flat-square)](https://github.com/abclq/cloud-agent-remote-ops/commits/main)
[![license](https://img.shields.io/github/license/abclq/cloud-agent-remote-ops?style=flat-square)](LICENSE)

**让云端 AI 安全地远程维护你的服务器 —— 它盯得住、修得动，但拿不走你的机器**

---

## 这是什么

你在云端沙盒里跑了个 AI 助手，想让它帮你盯着家里的服务器，出问题了甚至能自己动手修。
但你**不想把整台机器交出去**。

这个项目给出一套实战验证过的做法：

| 层次 | 做法 | 它能干什么 |
|---|---|---|
| 🔒 **锁入口** | SSH `command=` 锁死那把钥匙 | 只能跑一个脚本，**开不了 shell、传不了文件、转不了端口** |
| 🎯 **具名动作** | 分发器只认 `status` / `verify` / `fix` / `exec` / `help` | **不接受参数拼 shell**，想越权都没门 |
| 🩺 **只读诊断** | 命令白名单 + 元字符全拒 + 凭据关键字全拒 | 能查状态、看日志，**看不到任何密钥** |
| 🔧 **亲自修** | `exec` —— 以普通用户身份自己写、自己跑 | **真能修**，但拿不到 root |
| 📋 **审计** | 每次 `exec` 一行一条落盘 | 它干过什么，**全部可追溯** |

---

## 目录

- [为什么要做这个](#-为什么要做这个) —— 威胁模型：沙盒里的私钥不是秘密
- [架构（30 秒版）](#-架构30-秒版) —— 一把锁 + 一个分发器
- [三层能力递进](#-三层能力递进) —— 从只读，到能修
- [护栏是怎么设计的](#-护栏是怎么设计的) —— 拦什么、为什么这么拦
- [⚠️ 开写权限前必做：提权链审查](#️-开写权限前必做提权链审查) —— **这一节最重要**
- [检测的四种状态](#-检测的四种状态) —— 连不上 ≠ 服务挂了
- [告警与回执](#-告警与回执) —— 它凭什么判断"联系不上你"
- [⚠️ 效果边界](#️-效果边界诚实说明) —— **先看这个**
- [🔗 相关项目 & 我们学到了什么](#-相关项目--我们学到了什么) —— 同类项目翻了一遍，抄回来 8 条
- [部署](#-部署) · [目录结构](#-目录结构) · [贡献](#-贡献) · [License](#-license)
- [📚 **docs/ 文档索引**](docs/README.md)

---

## 为什么要做这个

### 威胁模型：沙盒里的私钥不是秘密

先说一个大多数人会忽略的前提：

> **免费沙盒平台侧，通常能看到沙盒里的文件 —— 包括你放进去的私钥。**

也就是说，你给云端 AI 的那把 SSH 私钥，**不能当成"只有它知道"**。
它更像一把**半公开的钥匙**。

结论很直接：

- ❌ **不能给它一台机器的 shell** —— 等于把整台机器交给一个半公开的凭据
- ✅ **只能给它一个"入口"** —— 门口站着安检，它只能在允许的范围内活动

### 那它还能帮我干活吗？

能。而且够用 —— 因为"维护服务器"这件事，**99% 的动作就那么几类**：

- 看服务活没活
- 看日志、看磁盘、看进程
- 重启一个明确挂掉的服务
- 改一个配置、改一个脚本

这几类动作**全都可以被"具名动作"覆盖**，不需要一把万能 shell。

---

## 架构（30 秒版）

```
┌─────────────────┐         ┌──────────────────────────────┐
│  云端 AI 助手    │         │         你的服务器            │
│  （沙盒里）      │         │                              │
│                 │  SSH    │  sshd                        │
│  私钥 ⚠️ 半公开  │ ──────▶ │   │                          │
│                 │ :2222   │   └─ authorized_keys         │
└─────────────────┘         │       command="…/entry.sh"   │
                            │            │                 │
                            │            ▼                 │
                            │      ┌───────────┐           │
                            │      │  分发器    │           │
                            │      └─────┬─────┘           │
                            │   ┌────────┼────────┐        │
                            │   ▼        ▼        ▼        │
                            │ 只读白名单  具名动作   exec   │
                            │ （诊断）   （修复）  （亲自修）│
                            └──────────────────────────────┘
```

**关键点**：`authorized_keys` 里那条 `command="..."` —— 它把整把钥匙**焊死在一个脚本上**。
无论对面发什么命令，sshd 都只会执行那个脚本，并把原始命令塞进环境变量 `$SSH_ORIGINAL_COMMAND` 交给它自己判断。

再配合这几个 sshd 选项，**物理上就开不了 shell**：

```
command="/path/to/entry.sh",restrict
```

> 💡 OpenSSH 7.2+ 有 **`restrict`** 一个词，等价于
> `no-pty,no-port-forwarding,no-agent-forwarding,no-X11-forwarding`，
> 而且以后 OpenSSH 新增的会话特性也会**自动一并关掉** —— 比手写四个更安全。

### ⚠️ 但 `command=` 有个洞：账号自己能改

`authorized_keys` 是**账号自己**拥有的文件。一旦你给了它写权限（比如为了 `exec`），
它就能把自己那行 `command=` 改掉，或者干脆**加一把新钥匙** —— 护栏形同虚设。

**解法**：改用 sshd 的 **`ForceCommand`**（写在 `sshd_config` 的 `Match` 块里，**服务端强制**）：

```sshconfig
Match User your-user
    ForceCommand /usr/local/bin/agent-entry.sh
    PermitTTY no
    X11Forwarding no
    AllowTcpForwarding no
```

| 方式 | 谁设的 | 范围 | 账号能改吗 |
|---|---|---|---|
| `authorized_keys` 的 `command=` | 账号自己 | 单把钥匙 | ✅ **能改**（改自己的文件） |
| `sshd_config` 的 `ForceCommand` | 管理员 root | 整个用户 | ❌ **改不掉** |
| 受限 shell（`chsh -s`） | 管理员 root | 整个账号 | ❌ 改不掉 |

**结论**：要挡住「它改自己的护栏」，只有 `ForceCommand` / 受限 shell 这条路。
`command=` 适合你自己完全掌控的场景；**一旦给了对面写权限，就该上 `ForceCommand`**。

---

## 三层能力递进

### 第 1 层：只读诊断（白名单）

```bash
ssh -p 2222 user@host "df -h"
ssh -p 2222 user@host "systemctl is-active nginx"
ssh -p 2222 user@host "journalctl -u nginx -n 50"
```

三道防线：

1. **元字符全拒** —— `; | & < > \` $ ( ) { }` 一律拒绝（防命令拼接）
2. **凭据关键字全拒** —— `.ssh` `id_*` `.env` `token` `password` `shadow` …
3. **路径守卫** —— 只允许白名单目录下的命令

### 第 2 层：具名动作（不给参数）

```bash
ssh -p 2222 user@host "status"    # 完整巡检 JSON
ssh -p 2222 user@host "verify"    # 精简复验：healthy / problem_count / problems
ssh -p 2222 user@host "fix"       # 跑本机自愈脚本（只重启"明确已死"的服务）
ssh -p 2222 user@host "help"      # 动作清单
```

**为什么"不给参数"很重要**：

```
"fix now"        → ⛔ 拒绝（fix 不接受参数）
"fix; rm -rf /"  → ⛔ 拒绝（含元字符）
"rm -rf /"       → ⛔ 拒绝（未知命令）
```

只要动作是**具名的、参数是固定映射的**，就**不存在"拼接"这个攻击面**。

### 第 3 层：`exec` —— 亲自修

具名动作覆盖不到的边角情况（改配置、改脚本、清磁盘），给它一个 `exec`：

```bash
ssh -p 2222 user@host 'exec systemctl --user restart my-service'
ssh -p 2222 user@host 'exec cat > ~/agent-workspace/fix.sh << "EOF" ... EOF'
```

**代价与收益**：

- ✅ 它**真能自己动手**，而不是只能按你预设的按钮
- ⚠️ 这等于给它**普通用户权限**（拿不到 root —— 前提是你做过[提权链审查](#️-开写权限前必做提权链审查)）
- ⚠️ 凭据类拦截是**关键字尽力而为**，不是硬隔离 —— 见[效果边界](#️-效果边界诚实说明)

---

## 护栏是怎么设计的

`exec` 的护栏分两道 + 一道审计：

### ① 不可逆 / 改身份 / 改登录 —— 一律拦

```
mkfs  fdisk  parted
shutdown  reboot  poweroff  halt
systemctl (reboot|poweroff|halt|suspend|hibernate)
passwd  chpasswd  visudo  useradd  userdel  usermod  chsh
rm -rf /          # 以及 rm -rf ~
dd of=/dev/…
> /etc/sudoers    > /etc/shadow    >> authorized_keys
kill -9 -1        killall5
curl … | bash     wget … | bash
```

### ② 凭据类路径 / 关键字 —— 尽力而为

```
.hermes/shared   ~/.ssh/id_ed25519   id_rsa   id_ecdsa
.env   password   passwd=   secret   api_key   token   credential
```

### ③ 审计 —— 每次调用一行一条

```
2026-01-01T00:00:00+0800 | from=<来源 IP> | systemctl --user restart my-service
```

**它干过什么，全部可追溯。**

### 实测：拦得准、不误伤

```
⛔ 拦掉：rm -rf / · dd of=/dev/sda · shutdown · reboot · visudo
        curl|bash · 改 authorized_keys · systemctl reboot · kill -9 -1
⛔ 拦掉：cat ~/.hermes/shared/x · grep token ~/.hermes/shared · cat ~/.ssh/id_ed25519
✅ 放行：systemctl --user restart x · journalctl -u x -n 50 · df -h
        du -sh ~/* | sort -h | tail -3
```

---

## ⚠️ 开写权限前必做：提权链审查

**这是整个方案里最容易翻车的地方。**

给它 `exec` 之后，它能**写文件**了。那么问题来了：

> **它写的那些文件，有没有哪个会被 `root` 执行？**

如果有 —— 它就能**借 root 的手**把自己提上去，你的所有护栏瞬间归零。

### 审查方法：一张表

把「**它能写的每个位置**」× 「**谁在执行那些位置里的东西**」列出来：

| 它能写的位置 | 谁在执行 | 结论 |
|---|---|---|
| `~/.agent-workspace/**` | 它自己 | ✅ 安全 |
| `~/.local/bin/**` | 用户的 cron | ⚠️ 要看 cron 里那个任务是不是 root 跑 |
| `/etc/cron.d/**` | **root cron** | 🔴 **绝对不能放** |
| `/etc/systemd/system/**` | **root systemd** | 🔴 **绝对不能放** |
| `/etc/sudoers.d/**` | sudo | 🔴 **绝对不能放** |
| root 的 crontab | **root** | 🔴 **绝对不能放** |

### 具体怎么查

```bash
# 1. root 的 crontab 里有没有引用它的入口？
sudo crontab -l | grep -i <它的脚本名>

# 2. 系统 cron 目录里有没有？
sudo grep -rl <它的脚本名> /etc/cron* 2>/dev/null

# 3. system 级 systemd 单元里有没有？
sudo grep -rl <它的脚本名> /etc/systemd/system /usr/lib/systemd/system 2>/dev/null
```

**三处都没有引用 → 提权路是断的 → 可以放心开写权限。**

> 💡 原则：**凡是被 `root` 执行的东西，一律不给它写。**
> 它只能写「以它自己或同一个普通用户身份执行」的文件。

---

## 检测的四种状态

**这一节解决一个很实际的问题：半夜被吵醒。**

从云端 AI 那边看，**服务器关机、服务器断网、隧道挂了、sshd 挂了** ——
表现**一模一样**：连不上。

如果不区分，你**自己关个机**，它就会报"服务全挂了"来轰炸你。

### 四态判定

```
① 连不上（超时 / 拒绝 / 不可达）
   → 判「服务器不可达」（关机 or 断网 or 隧道挂 or sshd 挂）
   → 默认【静默】，不打扰你
   → 只记起始时间；连续不可达超过阈值 → 提示一次（带时长），之后不再重复

② 连上，但认证失败 / 命令被拒
   → 判「配置异常」→ 提示一次

③ 连上，JSON healthy=true
   → 正常，静默
   → 若上次是「不可达」或「有故障」→ 提示一次「已恢复」

④ 连上，JSON 有 problems
   → 真故障 → 提示你 → N 分钟没回执 → 它亲自修
   → 同一问题没变化【不重复提示】，只在状态跳变时提示
```

**一句话**：只有 **「能连上 + 有故障」** 才吵你。

### 为什么"静默"是对的

自己关机 / 出门断网 / 路由器重启 —— 这些都是**你主动做的**，你当然知道。
它不该把"你做的事"当成"故障"来报警。

> 💡 反过来也成立：**如果它一直能连上，就说明你的网络和隧道是通的** ——
> 「连不上」这个信号本身，就已经在告诉你"是服务器那边的问题，不是某个服务的问题"。

---

## 告警与回执

### 为什么要回执

它发现故障 → 给你发告警 → 然后它**怎么知道你有没有收到**？

如果没有回执机制，它只有两个选择：

- 假设你收到了 → 你其实没收到 → **故障没人处理**
- 假设你没收到 → **每次都自己动手** → 那"先通知你"这一步就形同虚设

### 做法：回执 = "联系上了"的唯一凭据

```
它发现故障
   │
   ├─▶ 发告警给你
   │
   ▼
你的服务器收到告警 → 自动处理 → 回一封「回执」给它
   │
   ▼
它收到回执 → 确认「联系上了」→ 静默等待
它没收到回执（超时）→ 判定「联系不上」→ 亲自修
```

**两个坑**（我们都踩过）：

1. 🔴 **发件人白名单漏了它** —— 它的告警被静默过滤 → 它永远"联系不上" → 永远走自修分支
2. 🔴 **回执不能发回本机** —— 否则会自己触发自己，形成死循环

---

## ⚠️ 效果边界（诚实说明）

**别抱不切实际的期待**，这套方案有明确边界：

| 边界 | 说明 |
|---|---|
| 🔴 **凭据拦截不是硬隔离** | 关键字黑名单是"尽力而为"。真要硬隔离得上 `bwrap` / 容器 / 独立用户 |
| 🔴 **它能改自己的护栏** | 如果入口脚本它可写，它就能改掉护栏。**入口必须 root 拥有、只读** |
| 🔴 **拿不到 root 的前提是提权链审查** | 没审就开写权限 = 等于给了 root |
| 🟡 **它看不到"没连上时"的本机状态** | 关机期间的巡检是断档的，恢复后才知道 |
| 🟡 **它自己也会挂** | 云端沙盒通常不持久 → **本机自己的巡检 cron 要留着**，双保险 |
| 🟡 **"不可达"分不清原因** | 关机 / 断网 / 隧道挂 从外面看一样，只能统一判"不可达" |

---

## 🔗 相关项目 & 我们学到了什么

做之前先把同类翻了一遍。**下面这几条是抄回来的**（附出处）：

### 直接改变了我们设计的三条

**① `ForceCommand` + `Match` —— 补掉「账号自己改护栏」的洞**
出自 OpenSSH 官方文档 / [hvr.biz 的 forced command 长文](https://hvr.biz/topic.php?slug=restrict-an-ssh-key-to-one-command-forced-commands)。
我们原本用 `authorized_keys` 的 `command=`，搜到这篇才知道**账号自己改得掉**。

**② 修复前自动备份 + 验证失败自动回滚**
出自 [`SentinelMCP`](https://github.com/AnantharamanKV/SentinalMCP)。
它的做法：改配置前存 checksum 备份，改完验证；**验证不过就自动还原**。
我们原本的 `autofix.py` 只重启服务，没备份。

**③ 无限循环断路器：同一问题失败 2 次就停**
同样出自 `SentinelMCP`。按「诊断指纹」记历史，**同一个指纹失败 ≥2 次 → 降低置信度 / 直接封策略**。
比"无限重试"和"无限告警"都聪明。

### 告警体验上抄的两条

**④ 通知去重** —— 出自 [`psyduckler/self-healing-agents`](https://github.com/psyduckler/self-healing-agents)
它的规则：**同一个 heal 报过就不再报，除非复发**（说明上次的修复没 hold 住）。
这正是我们要的「别一直提示我」。

**⑤ 风险评分分流** —— 同一个项目
`≤0.3` 自动修 · `0.3–0.8` 修完盯紧 · `>0.8` 交人。
比我们「能修就修」更细。

**⑥ 级联失败检测** —— 还是它
3 个 cron 同时挂 → 判定**一个根因**，不是 3 个问题。避免告警风暴。

### 护栏上更高级的思路

**⑦ AST 解析代替正则黑名单** —— 出自 [`sunir/bashguard`](https://github.com/sunir/bashguard)
用 tree-sitter 把命令**解析成语法树**再判，而不是正则匹配字符串 —— 绕过难度高一个量级。
它还拦「试图改 bashguard 自己」，思路和我们一样，但实现更硬。

**⑧ 凭据占位符注入** —— 同一个项目
agent 写 `curl -H "Authorization: Bearer {{GITHUB_TOKEN}}"`，
**真值在执行时才替换** → agent 从来看不到真 token。
比我们「关键字拦凭据路径」干净得多，**下一步想上这个**。

### 同类项目速查

| 项目 | 是什么 | 我们参考了什么 |
|---|---|---|
| [`SentinelMCP`](https://github.com/AnantharamanKV/SentinalMCP) | MCP server，检测→诊断→修复→验证，支持远程 SSH | 备份回滚、循环断路器 |
| [`psyduckler/self-healing-agents`](https://github.com/psyduckler/self-healing-agents) | agent 自愈框架，known-fixes DB | 通知去重、风险评分、级联检测 |
| [`sunir/bashguard`](https://github.com/sunir/bashguard) | LLM 的 bash 拦截器（AST） | AST 解析、凭据占位符 |
| [`bdsh`](https://github.com/RaymiiOrg/boa-diminish-restricted-shell) | 白名单受限 shell，带日志 | 受限 shell 的成熟做法 |
| [`secureshell`](https://github.com/stevewithington/secureshell) / [`cmdguard`](https://github.com/JSLEEKR/cmdguard) / [`agentguard`](https://github.com/alvabillwu/agentguard) | 命令网关，GREEN/YELLOW/RED 分级 | 分级思路 |
| [`awesome-agent-sandboxes`](https://github.com/arjan/awesome-agent-sandboxes) | 沙箱方案合集 | 想上硬隔离时翻这个 |
| `git-shell` | OpenSSH 自带的受限 shell | 官方参考实现 |

### 我们和它们不一样的地方

上面这一堆，**没有一个处理「连不上 ≠ 服务挂了」**。

它们都会在你**自己关机、断网、换 Wi-Fi** 的时候照样告警 ——
而我们的[四态判定](#检测的四种状态)第一句就是：
**连不上默认静默，只有「连上了 + 真有故障」才吵你。**

这一条是我们自己的（也是被真实生活逼出来的）。

---

## 部署

### 1. 服务器侧

```bash
# 入口脚本（root 拥有、只读 —— 关键！）
sudo install -o root -g root -m 0755 entry.sh /usr/local/bin/agent-entry.sh
sudo install -o root -g root -m 0755 guard.sh /usr/local/bin/agent-guard.sh

# 工作区（它可写）
mkdir -p ~/agent-workspace

# 审计目录
sudo mkdir -p /var/log/agent-entry && sudo chown "$USER" /var/log/agent-entry
```

### 2. 锁死那把钥匙

**方案 A（简单）** —— 在 `~/.ssh/authorized_keys` 里加一条（**`command=` 指向 root 拥有的入口**）：

```
command="/usr/local/bin/agent-entry.sh",restrict ssh-ed25519 AAAA…cloud-agent
```

⚠️ 但这个方案**账号自己改得掉**（见上文架构节的洞）。它只适合你自己完全掌控账号的场景。

**方案 B（推荐 —— 给对面写权限时必须用这个）** —— 在 `/etc/ssh/sshd_config` 里用 `Match` 块：

```sshconfig
Match User your-user
    ForceCommand /usr/local/bin/agent-entry.sh
    PermitTTY no
    X11Forwarding no
    AllowTcpForwarding no
    PermitOpen none
```

```bash
# 改完先验证语法，再 reload（别直接 restart，语法错会把自己关在门外）
sudo sshd -t && sudo systemctl reload ssh
```

方案 B 的好处：**服务端强制**，账号改 `authorized_keys`、加新钥匙都逃不掉 ——
这条也是我们给对面开了 `exec` 之后才补上的（先踩坑，后补课）。

### 3. 云端侧

```bash
ssh -i ~/.ssh/agent_key -p 2222 user@your-host.example.com help
ssh -i ~/.ssh/agent_key -p 2222 user@your-host.example.com verify
ssh -i ~/.ssh/agent_key -p 2222 user@your-host.example.com 'exec systemctl --user restart my-service'
```

### 4. 先审提权链，再开 `exec`

**顺序不能反。** 见[上面那一节](#️-开写权限前必做提权链审查)。

---

## 目录结构

```
.
├── README.md              主文（中文）
├── README.en.md           主文（英文）
├── LICENSE                MIT
├── CHANGELOG.md
├── CONTRIBUTORS.md
├── docs/
│   ├── README.md          文档索引（按"你是谁"挑入口）
│   ├── threat-model.md    威胁模型：沙盒里的私钥不是秘密
│   ├── architecture.md    架构：一把锁 + 一个分发器
│   ├── actions.md         动作参考
│   ├── guardrails.md      护栏设计
│   ├── privilege-audit.md 提权链审查（最重要的一节）
│   ├── detection-states.md 四态判定
│   ├── alerts-and-receipts.md 告警与回执
│   ├── deploy.md          部署步骤
│   └── pitfalls.md        踩过的坑
└── scripts/
    ├── entry.sh           入口分发器模板（脱敏通用版）
    └── guard.sh           只读白名单模板（脱敏通用版）
```

---

## 贡献

- 有更狠的护栏思路？开 Issue 聊聊
- 踩过我们没踩过的坑？非常欢迎
- 想加一种新的「具名动作」？直接 PR

**给机器人 / 协作者的作业说明**：见 [`docs/for-agents.md`](docs/for-agents.md)

---

## 致谢

感谢所有把「怎么安全地让 AI 碰我的机器」这个问题认真想过一遍的人。

---

## License

[MIT](LICENSE)
