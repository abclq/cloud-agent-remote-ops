# 🏗 架构：一把锁 + 一个分发器

> 整台机器只暴露**一个入口脚本**，脚本只认**五个具名动作**。

---

## 全景图

```
        ┌──────────────────────────────┐
        │  云端 AI（沙盒，不归你所有）  │
        │  私钥住在这儿 = 半公开       │
        └──────────────┬───────────────┘
                       │ ssh -p 2222 user@host 'verify'
                       │
        ┌──────────────▼───────────────┐
        │  sshd                        │
        │  ├ ForceCommand / command=   │  ← ① 锁：打不开 shell
        │  └ restrict                  │     连 PTY/转发都没有
        └──────────────┬───────────────┘
                       │ 只能调这一个脚本，动作作为参数
                       │
        ┌──────────────▼───────────────┐
        │  /usr/local/bin/agent-entry  │  ← ② root 拥有、只读
        │  （它读得到、改不了）         │
        └──────────────┬───────────────┘
                       │
        ┌──────────────▼───────────────┐
        │  dispatcher（分发器）         │  ← ③ 只认具名动作
        │  status | verify | fix |      │     不接受任意字符串
        │  exec   | help                │
        └──────────────┬───────────────┘
                       │
        ┌──────────────▼───────────────┐
        │  你机器上的真实东西           │
        │  服务 / 配置 / 日志           │
        │  ── 以「那个用户」身份 ──     │
        │  ── 拿不到 root ──            │
        └──────────────────────────────┘
```

**三个数字标记就是三条约束，缺一不可**（见[威胁模型](threat-model.md)）。

---

## ① 锁：怎么让它打不开 shell

### 方案 A：`authorized_keys` 的 `command=`

```
command="/usr/local/bin/agent-entry.sh",restrict ssh-ed25519 AAAA…cloud-agent
```

- `command=` → 无论它请求什么命令，sshd 只执行这一个脚本，原命令塞进 `$SSH_ORIGINAL_COMMAND`
- `restrict` → OpenSSH 7.2+ 的**一个词**，等价于 `no-pty,no-port-forwarding,no-agent-forwarding,no-X11-forwarding`，而且以后 OpenSSH 加的新特性会自动一并关掉

### ⚠️ 方案 A 的洞

`authorized_keys` 是**那个用户自己**拥有的文件。

一旦你为了让它能修东西而给了写权限 → **它能改掉自己那一行 `command=`**，
或者干脆追加一把自己的新钥匙 → **锁没了**。

### 方案 B：`sshd_config` 的 `ForceCommand`（推荐）

```sshconfig
Match User your-user
    ForceCommand /usr/local/bin/agent-entry.sh
    PermitTTY no
    X11Forwarding no
    AllowTcpForwarding no
    PermitOpen none
```

```bash
sudo sshd -t && sudo systemctl reload ssh   # 先验语法，再 reload
```

**服务端强制** —— 它改 `authorized_keys`、加新钥匙，都逃不掉。

### 对比

| | 谁设的 | 范围 | 它能改吗 |
|---|---|---|---|
| `authorized_keys` `command=` | 那个用户自己 | 单把钥匙 | ✅ **能改** |
| `sshd_config` `ForceCommand` | root | 整个用户 | ❌ 改不掉 |
| 受限 shell（`chsh -s`） | root | 整个账号 | ❌ 改不掉 |

**给写权限之前，请务必上方案 B。** 我们先用了 A，踩了坑才补的 B。

---

## ② root 拥有、只读的入口

```bash
sudo install -o root -g root -m 755 agent-entry.sh /usr/local/bin/agent-entry.sh
```

为什么必须 root：

- 入口脚本是**护栏的唯一实现**。它归那个用户所有 → 它能改掉护栏 → 护栏不存在
- root 拥有 + 755 → 它**读得到**（能调），**改不了**（改护栏要 sudo）

工作区分开：

```
/usr/local/bin/agent-entry.sh   root:root 755   ← 护栏，改不了
~/agent-workspace/              user:user 700   ← 它能随便写
~/.hermes/logs/*.log            user:user       ← 审计
```

**护栏和作业区物理分离** —— 这是整个设计的核心。

---

## ③ 分发器：只认具名动作

```bash
case "$1" in
  status)  do_status  ;;
  verify)  do_verify  ;;
  fix)     do_fix     ;;
  exec)    do_exec "$@" ;;   # ← 唯一"像 shell"的，配护栏
  help)    do_help    ;;
  *)       echo '{"error":"unknown action"}'; exit 2 ;;
esac
```

关键点：

- **不接受任意字符串** —— 没有 `eval`、没有 `sh -c "$*"`
- 动作名是**白名单**，其他一律拒绝
- 前三个动作**不接受参数** → 攻击面为零
- 只有 `exec` 接受参数 → 所以它单独配三道护栏（见 [护栏设计](guardrails.md)）

---

## 三层能力递进

| 层 | 动作 | 能力 | 风险 |
|---|---|---|---|
| 1 | `status` `verify` | 只读诊断 | 极低 |
| 2 | `fix` | 跑预置修复脚本 | 低（脚本是你写的） |
| 3 | `exec` | 自己诊断、自己动手 | **中**（要护栏 + 审计 + 提权链审查） |

**建议顺序**：先只开第 1 层跑一周 → 确认它靠谱 → 再开第 2 层 → 最后才开第 3 层。

---

## 一个实用技巧：转发壳

**场景**：你想换入口脚本，但 `authorized_keys` 是受保护文件（有些环境不让改），
或者 `command=` 指向的老脚本已经在用了，不想动 sshd。

**做法**：把 `command=` 指向的那个普通文件，改成**转发壳**：

```bash
#!/usr/bin/env bash
# 不带参数 → 保持老行为（向后兼容）
if [ -z "${SSH_ORIGINAL_COMMAND:-}" ]; then
    exec /path/to/old-behavior.sh
fi
# 带参数 → 转交给新入口
exec /usr/local/bin/agent-entry.sh "$SSH_ORIGINAL_COMMAND"
```

**好处**：不用碰 `authorized_keys`、不用 reload sshd、老调用方不受影响。

**代价**：多一层间接。而且这个壳如果归那个用户所有，它**也能改** —— 所以壳里**不该有护栏**，
护栏必须在 root 拥有的那个脚本里。

---

## 为什么不做的事

| 想法 | 为什么不做 |
|---|---|
| 给完整 shell + 靠 sudo 白名单 | sudo 白名单的洞比想象多；而且 shell 本身能读凭据 |
| 用 Web 终端代替 SSH | 对 AI 没意义 —— AI 要的是可编程接口，不是给人看的终端 |
| 让它 `scp` 传文件 | 传文件 = 双向通道，能传进来就能传出去（`restrict` 里本来也关了） |
| 给它 root 但"加审计" | 审计只能事后发现，不能阻止；提权链审查是事前阻止 |
| 靠 AI 自己守规矩（提示词） | 提示注入一句话就能推翻。**约束必须在 AI 碰不到的地方** |

最后一条是整个项目的灵魂：**护栏不能由被约束者持有。**

---

## 下一步

- [动作参考](actions.md) —— 五个动作具体怎么调
- [护栏设计](guardrails.md) —— `exec` 的三道护栏
- [⚠️ 提权链审查](privilege-audit.md) —— 开写权限前的必做项
