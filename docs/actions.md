# 🎛 动作参考

> 分发器只认五个动作。**前三个不接受参数**，第五个（`exec`）单独配护栏。

---

## 调用方式

```bash
ssh -i <你的钥匙> -p 2222 user@your-host '动作 [参数]'
```

原命令会进 `$SSH_ORIGINAL_COMMAND`，由分发器解析。

**没带参数**（只登录不执行）→ 视作 `help`，返回动作清单。这对 AI 很友好：
它先 `ssh` 一下就知道自己能干什么。

---

## 动作总览

| 动作 | 参数 | 能力 | 风险 |
|---|---|---|---|
| `help` | 无 | 列出可用动作 | 无 |
| `status` | 无 | 人读的概览 | 无 |
| `verify` | 无 | 机器读的 JSON 体检 | 无 |
| `fix` | 无 | 跑预置修复脚本 | 低 |
| `exec` | `<命令>` | 亲自诊断/动手/验证 | **中** |

---

## `help`

```bash
ssh ... 'help'
```

返回动作清单 + 每个动作的一句话说明 + 工作区路径。**建议 AI 每轮开头先调一次**，
防止你改了动作它不知道。

---

## `status` —— 给人看的

```bash
ssh ... 'status'
```

返回一段人类可读的概览：服务状态、磁盘、内存、最近异常。

**适合**：你自己 ssh 上去看一眼，或者 AI 想给用户讲人话时用。

---

## `verify` —— 给机器看的

```bash
ssh ... 'verify'
```

返回 **JSON**，这是整个系统的核心接口。

**约定字段**：

```json
{
  "healthy": true,
  "checked_at": "2026-10-15T00:12:33+08:00",
  "host": "your-host",
  "boot_id": "3f2a…",
  "uptime_h": 42.5,
  "net_ok": true,
  "frp_ok": true,
  "counts": { "services": 14, "link": 11, "scheduled": 26, "total": 51 },
  "problems": [
    {
      "kind": "systemd-service",
      "name": "example.service",
      "detail": "inactive (dead)"
    }
  ]
}
```

**为什么要有这些字段**：

| 字段 | 给谁用 |
|---|---|
| `healthy` | 一句话判断：要不要吵用户 |
| `checked_at` | 判断数据新鲜度（**缺失 = 不可信，不是"没问题"**） |
| `boot_id` / `uptime_h` | **判断本机是不是刚重启过**（见 [四态判定](detection-states.md)） |
| `net_ok` / `frp_ok` | 区分「本机断网」和「服务挂了」 |
| `problems[]` | 结构化问题列表，方便去重和追踪 |

**`kind` 的取值**建议固定成枚举，方便对面写逻辑：

```
systemd-service | user-service | disk | cron-stale | link | mail-heartbeat | custom
```

---

## `fix` —— 跑预置修复

```bash
ssh ... 'fix'
```

**只跑你预先写好的修复脚本**（比如 `autofix.py`）。它的原则：

- 只重启**明确已经死了**的常驻服务
- 有黑名单（不碰配置、不碰凭据）
- 改不了的东西**报错而不是硬来**

**`fix` 不接受参数** —— 这是刻意的。参数 = 攻击面。

---

## `exec` —— 亲自修（重点）

```bash
ssh ... 'exec <命令>'
```

让它**自己诊断、自己动手、自己验证**，而不是跑你写好的脚本。

### 为什么需要它

`fix` 只能修**你预料到的问题**。真实故障经常是你没预料到的
（配置漂移、权限被改、依赖版本变了……）。

这时候需要它能**看着日志现场判断**，而不是回来告诉你"我修不了，你自己看"。

### 它是什么、不是什么

| 是 | 不是 |
|---|---|
| 以**那个用户**身份执行 | ❌ 不是 root |
| 单条命令，走 shell 解析 | ❌ 不是交互式会话 |
| 有**三道护栏** | ❌ 不是任意命令 |
| **每次都记审计** | ❌ 不是静默执行 |

### 示例（能过的）

```bash
ssh ... 'exec systemctl --user is-active my-service'
ssh ... 'exec systemctl --user restart my-service'
ssh ... 'exec journalctl --user -u my-service -n 50'
ssh ... 'exec df -h'
ssh ... 'exec bash /home/user/agent-workspace/diagnose.sh'
```

### 示例（会被拦的）

```bash
ssh ... 'exec rm -rf /'                    # ⛔ 不可逆
ssh ... 'exec systemctl reboot'            # ⛔ 影响整机
ssh ... 'exec cat ~/.ssh/id_ed25519'       # ⛔ 凭据
ssh ... 'exec cat /etc/shadow'             # ⛔ 凭据
ssh ... 'exec sudo passwd nobody'          # ⛔ 改身份
ssh ... 'exec echo a; rm -rf /tmp/x'       # ⛔ 元字符（`;`）
```

被拦会返回明确的 JSON 错误 + 原因，**不是静默失败** —— 这样 AI 知道换个做法，
而不是以为命令执行了。

### 工作区

```bash
ssh ... 'exec echo "echo hi" > ~/agent-workspace/hello.sh && bash ~/agent-workspace/hello.sh'
```

`~/agent-workspace/` 是**给它的可写区**。它在这儿写脚本、跑脚本、迭代。

**你的护栏和它的工作区是分开的** —— 见 [架构](architecture.md) 第 ② 节。

### 审计

每次 `exec` 都记一行：

```
2026-10-15T00:12:33+08:00  from=<来源>  rc=0  cmd=<命令>
```

路径建议 `~/.hermes/logs/remote-agent-exec.log`。

**审计是事后手段，不能替代护栏** —— 但它是唯一能回答"昨晚到底谁改了什么"的东西。

---

## 未知动作

```bash
ssh ... 'rm -rf /'
```

→ 动作名不匹配任何白名单 → 被拒绝，退出码 **126**。

**注意**：这里 `rm -rf /` 是**动作名**，不是命令 —— 它根本不会被执行。
这正是"只认具名动作"的价值。

---

## 退出码约定

**这一节很重要** —— 对面的 AI 靠退出码判断"修没修好"，
所以必须能区分「**被拒绝**」和「**命令自己失败**」。

| 退出码 | 含义 | 对方该怎么做 |
|---|---|---|
| `0` | 成功 | 继续 |
| `126` | **被护栏/终端拒绝** | **别重试、别换写法绕** —— 换个合法做法，或报告需要人工 |
| 其他 | 命令自己的退出码，**原样透传** | 正常判断（如 `systemctl is-active` 返回 3 = 未运行） |

### 为什么不用 `1`

最初拒绝也返回 `1` —— 结果和"命令执行失败"**分不开**。

这是对面 AI 实测发现的：

> 护栏拒绝也返回 1，跟命令本身失败分不开。建议拒绝时改用一个专用的码，比如 126。

`126` 在 shell 约定里本来就表示 *"找到了但不可执行"* —— 语义刚好对上"**入口不让这么干**"。

### 实测

```bash
# 拒绝类 → 126
ssh ... 'rm -rf /'                  # 126
ssh ... 'cat ~/.ssh/id_ed25519'     # 126
ssh ... 'systemctl reboot'          # 126
ssh ... 'df -h | grep x'            # 126（元字符）

# 命令类 → 原样
ssh ... 'exec true'                 # 0
ssh ... 'exec false'                # 1
ssh ... 'exec exit 7'               # 7

# 正常动作 → 0
ssh ... 'help'                      # 0
ssh ... 'verify'                    # 0
```

**约定也写进了 `help` 输出** —— 对面 AI 跑一次 `help` 就知道。

---

## 设计原则小结

1. **动作名白名单**，未知一律拒
2. **能不给参数就不给** —— `status`/`verify`/`fix` 零参数
3. **给参数的那个**（`exec`）必须配护栏 + 审计
4. **返回结构化 JSON** —— AI 好解析，你也好 grep
5. **失败要说话** —— 被拦、超时、脚本崩，都要有明确返回

---

## 下一步

- [护栏设计](guardrails.md) —— `exec` 到底拦什么
- [⚠️ 提权链审查](privilege-audit.md) —— 开 `exec` 之前必做
- [给协作者的作业说明](for-agents.md) —— 把这页发给对面的 AI
