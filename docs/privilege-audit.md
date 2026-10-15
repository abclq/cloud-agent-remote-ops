# ⚠️ 提权链审查：开写权限之前的必做项

> **护栏会被绕过。所以真正的问题是：绕过之后，它能到哪？**

---

## 一句话

**开写权限之前，先审「它能写的文件 × 谁在执行它」。**

凡是 root 会执行的文件，一律不放写。

---

## 为什么这一步不能省

我们的三道护栏（见 [护栏设计](guardrails.md)）是**字符串匹配**。
字符串匹配**一定**能被绕过 —— 只是难易问题。

所以不能只靠护栏。要再加一层：**就算护栏被绕过，也升不到 root。**

这一层不是代码，是**架构审查**。

---

## 审查方法：四步

### 第 1 步：列出它能写的地方

```bash
# 它拥有的目录
find / -user "$USER" -type d -writable 2>/dev/null | head -50

# 它拥有的可执行文件
find / -user "$USER" -type f -perm -u+x 2>/dev/null | head -50
```

典型答案：家目录、`~/.config`、`~/agent-workspace/`、`~/.hermes/`。

### 第 2 步：找出「谁会在更高权限下执行这些文件」

要查**四类**执行者：

```bash
# ① root 的 crontab
sudo crontab -l

# ② 系统级 cron
sudo ls -la /etc/cron.d/ /etc/cron.daily/ /etc/cron.hourly/ 2>/dev/null
sudo grep -rn "$USER\|/home/" /etc/cron.d/ /etc/crontab 2>/dev/null

# ③ 系统级 systemd（注意：不是 --user！）
sudo grep -rln "$USER\|/home/" /etc/systemd/system/ /lib/systemd/system/ 2>/dev/null

# ④ 别的提权入口
sudo grep -rn "$USER" /etc/sudoers /etc/sudoers.d/ 2>/dev/null
```

### 第 3 步：判断

| 它写的文件 | root 会执行吗 | 结论 |
|---|---|---|
| `~/agent-workspace/x.sh` | ❌ 没人执行 | 安全 |
| `~/.hermes/scripts/fix.py` | ❌ 只有它自己跑 | 安全 |
| `~/.config/systemd/user/*.service` | ❌ `--user` 服务 = 用户级 | 安全 |
| `/etc/cron.d/job` | ✅ root 执行 | 🔴 **危险** |
| root crontab 里引用的脚本 | ✅ root 执行 | 🔴 **危险** |
| `/usr/local/bin/agent-entry.sh` | ✅ root 的 sshd 执行 | 🔴 **危险** |

**只要出现一行 ✅ → 这条路是通的 → 别开写权限（或者先把那个引用去掉）。**

### 第 4 步：写下来

把结论落成文档（就是你现在读的这页）。**半年后你会忘了自己审过什么。**

---

## 我们的实际结论

审的时候是这么个情况：

```
✅ root crontab              → 没有引用任何它拥有的路径
✅ /etc/cron.d/ /etc/crontab → 同上
✅ /etc/systemd/system/      → 同上（注意：--user 服务不算）
✅ sudoers                   → 没有它的条目
```

→ **提权路是断的，可以开写权限。**

最坏情况：它拿到**那个用户**的全部权限（能读写家目录、能重启自己的用户级服务），
但**碰不到 root**。

### 但审出一个真问题 🔴

**入口脚本本身归那个用户所有。**

```
-rwxr-xr-x 1 user user /home/user/.hermes/scripts/agent-entry.sh
              ↑ 它自己的
```

→ 它能 `exec` 改掉入口脚本 → **改掉自己的护栏** 🔴

**这就是审查的价值** —— 不是理论问题，是当场发现的实际问题。

**修法**（见 [架构](architecture.md) 方案 B）：

```bash
sudo install -o root -g root -m 755 agent-entry.sh /usr/local/bin/agent-entry.sh
# 再把 command= / ForceCommand 指到 /usr/local/bin/ 那个
```

`root:root 755` → 它读得到、调得动、**改不了**。

---

## 常见坑

### 坑 1：以为 `--user` 服务是 root 跑的

```bash
systemctl --user status x    # 用户级，不是 root
systemctl status x           # 系统级，可能是 root
```

**审查时要查系统级**。用户级服务它自己能改，无所谓 —— 因为那本来就是它的权限范围。

### 坑 2：只查 crontab，漏了 `systemd timer`

```bash
sudo systemctl list-timers --all    # 别忘了这个
```

### 坑 3：忘了「root 拥有的文件被它写」

如果 `/etc/cron.d/` 里有个文件是 `root:root 664`，而它在 `root` 组里 → **它能改**。

```bash
# 找 root 拥有但组可写的
sudo find /etc -user root -perm -g+w 2>/dev/null
```

### 坑 4：没查 sudo 的 NOPASSWD 条目

```bash
sudo grep -rn 'NOPASSWD' /etc/sudoers /etc/sudoers.d/ 2>/dev/null
```

一条 `NOPASSWD: ALL` 就等于 root。

---

## 审查完之后

**可以开写权限了吗？** 取决于你要开哪一层：

| 要开的层 | 审查要求 |
|---|---|
| `status` / `verify`（只读） | 不用审（本来就没写） |
| `fix`（跑你写的脚本） | 审一次（脚本可能写文件） |
| `exec`（它自己动手） | **必须审 + 入口必须 root 拥有** |

**`exec` 是唯一需要"入口 root 化"的动作。** 因为只有它能让对面自由写。

---

## 一条更省心的路

如果你不想审 —— **那就别开 `exec`**。

`status` / `verify` / `fix` 三层已经能覆盖"日常巡检 + 已知故障自动修"。
`exec` 的价值是"处理你预料不到的故障"，代价是"你得做架构审查"。

**这是明码标价的取舍，不是必须走的路。**

---

## 自检清单

- [ ] 列出它拥有的所有可写目录和可执行文件
- [ ] 查 root crontab / `/etc/cron*` / system systemd / sudoers
- [ ] 确认没有一处 root 执行它拥有的文件
- [ ] **入口脚本是 root:root 755 吗？**
- [ ] 查过 `NOPASSWD` 条目吗？
- [ ] 查过 `systemd timer` 吗？
- [ ] 结论**写进文档**了吗？

---

## 下一步

- [护栏设计](guardrails.md) —— 第一道防线
- [架构](architecture.md) —— 入口为什么要 root 拥有
- [部署](deploy.md) —— 具体怎么装
