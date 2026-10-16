# 🚀 部署：从零到能跑

> 顺序很重要。**先只读，再修复，最后才开 `exec`。**

---

## 第 0 步：先想清楚（别跳过）

在动手前回答 [威胁模型](threat-model.md) 的自检清单。

**最重要的一条**：你能接受这把私钥明天就泄露吗？

能 → 继续。不能 → 先别给 SSH，换别的方案。

---

## 第 1 步：建专用账号，写入口脚本

```bash
# 专给 AI 的账号。不要复用你自己的账号：
#   - 第 3 步的 ForceCommand 按「用户」生效，会把你自己的钥匙也锁死
#   - 提权链审查要对「它」做，你自己的账号通常有 sudo，审不过
sudo useradd -m -s /bin/bash agent-ops
sudo mkdir -p /usr/local/lib/agent      # 后面 do-*.sh 都放这里，root 拥有
```

```bash
sudo tee /usr/local/bin/agent-entry.sh >/dev/null <<'EOF'
#!/usr/bin/env bash
# 云端 AI 的唯一入口。只认具名动作。
set -uo pipefail

# 用 command= 或 ForceCommand 时，脚本本身不带参数，
# 对面发来的整条命令在 $SSH_ORIGINAL_COMMAND 里，必须自己拆。
CMD="${SSH_ORIGINAL_COMMAND:-help}"
ACTION="${CMD%% *}"
REST=""
[[ "$CMD" == *" "* ]] && REST="${CMD#* }"

# 零参数动作带了参数 → 拒（126 = 被入口拒绝，见 actions.md）
case "$ACTION" in
  help|status|verify|fix)
    [ -n "$REST" ] && { echo '{"error":"action takes no arguments"}' >&2; exit 126; } ;;
esac

case "$ACTION" in
  help)   cat <<'H'
可用动作：
  help              显示这个清单
  status            人读的概览
  verify            机器读的 JSON 体检
  fix               跑预置修复脚本
  exec <命令>       亲自诊断/动手/验证（有护栏）
H
          ;;
  status) exec /usr/local/lib/agent/do-status.sh ;;
  verify) exec /usr/local/lib/agent/do-verify.sh ;;
  fix)    exec /usr/local/lib/agent/do-fix.sh ;;
  exec)   exec /usr/local/lib/agent/do-exec.sh "$REST" ;;   # 护栏在 do-exec.sh 里
  *)      echo '{"error":"unknown action"}' >&2; exit 126 ;;   # 别把对面的输入原样拼进 JSON
esac
EOF
```

⚠️ **注意**：sshd **不会**把原命令拆成 `$1 $2 …` 传给入口脚本 —— 脚本永远没有位置参数，
原命令只在 `$SSH_ORIGINAL_COMMAND` 里（没带命令时这个变量不存在，所以上面默认 `help`）。
如果写成 `ACTION="${1:-help}"`，不管对面发什么都只会得到 `help`。

`do-status.sh` / `do-verify.sh` / `do-fix.sh` / `do-exec.sh` 在后面几步写；
没写之前，对应动作会报 `No such file`，这是正常的。

**权限**（关键）：

```bash
sudo chown root:root /usr/local/bin/agent-entry.sh
sudo chmod 755 /usr/local/bin/agent-entry.sh
```

---

## 第 2 步：写三个只读动作

`/usr/local/lib/agent/do-verify.sh` 的骨架：

```bash
#!/usr/bin/env bash
set -uo pipefail

problems='[]'
add() {  # kind name detail
  problems=$(printf '%s' "$problems" | python3 -c '
import json,sys
a=json.load(sys.stdin)
a.append({"kind":sys.argv[1],"name":sys.argv[2],"detail":sys.argv[3]})
print(json.dumps(a))' "$1" "$2" "$3")
}

# —— 服务 ——
for svc in nginx sshd fail2ban; do
  if ! systemctl is-active --quiet "$svc" 2>/dev/null; then
    add systemd-service "$svc" "$(systemctl is-active "$svc" 2>&1)"
  fi
done

# —— 磁盘 ——
while read -r use mp; do
  pct="${use%\%}"
  [ "$pct" -ge 85 ] && add disk "$mp" "${pct}% used"
done < <(df -h --output=pcent,target | tail -n +2)

# —— 输出 ——
python3 - "$problems" <<'PY'
import json,sys,uuid,os,time
probs=json.loads(sys.argv[1])
print(json.dumps({
  "healthy": len(probs)==0,
  "checked_at": time.strftime("%Y-%m-%dT%H:%M:%S%z"),
  "host": os.uname().nodename,
  "boot_id": open("/proc/sys/kernel/random/boot_id").read().strip(),
  "uptime_h": round(float(open("/proc/uptime").read().split()[0])/3600, 1),
  "problems": probs,
}, ensure_ascii=False))
PY
```

**注意 `boot_id` 和 `uptime_h`** —— 这是[四态判定](detection-states.md)的基础。

`do-status.sh` 可以是人读版（`systemctl status` + `df -h` 拼一拼）。

---

## 第 3 步：装锁（选一个）

### 方案 A：`authorized_keys`

```bash
# ① 在【云端】生成专用钥匙（私钥别离开它），把公钥那一行打印出来
ssh-keygen -t ed25519 -C "cloud-agent" -f ~/.ssh/id_agent -N ""
cat ~/.ssh/id_agent.pub

# ② 在【服务器】上，把上一步打印的那一行粘进来（公钥可以随便传，私钥不行）
PUB='ssh-ed25519 AAAA…cloud-agent'
sudo install -d -o agent-ops -g agent-ops -m 700 ~agent-ops/.ssh
echo "command=\"/usr/local/bin/agent-entry.sh\",restrict $PUB" \
  | sudo tee -a ~agent-ops/.ssh/authorized_keys >/dev/null
sudo chown agent-ops:agent-ops ~agent-ops/.ssh/authorized_keys
sudo chmod 600 ~agent-ops/.ssh/authorized_keys
```

### 方案 B：`ForceCommand`（推荐，尤其当你之后要开 `exec`）

```bash
sudo tee /etc/ssh/sshd_config.d/agent.conf >/dev/null <<'EOF'
# ⚠️ Match 的是「整个用户」，不是某一把钥匙：必须是专给 AI 的独立账号，
#    千万别写你自己登录用的账号，否则你自己的钥匙也会被锁成只能跑入口脚本。
Match User agent-ops
    ForceCommand /usr/local/bin/agent-entry.sh
    PermitTTY no
    DisableForwarding yes     # OpenSSH 7.4+：TCP、Unix socket、X11、agent 转发一次全关
    PermitTunnel no
EOF

# Debian/Ubuntu 单元名是 ssh，RHEL/Fedora/Arch 是 sshd
sudo sshd -t && (sudo systemctl reload ssh || sudo systemctl reload sshd)
```

方案 B 同样要做方案 A 的 ①②（公钥得先进 `authorized_keys` 才登得上），
`Match` 只决定登录后能跑什么。

⚠️ **先 `sshd -t` 验语法** —— 配错了 reload 会让 sshd 起不来，你就把自己关在外面了。

⚠️ **reload 前先开另一个 SSH 连接** —— 万一断了还能救。

---

## 第 4 步：测（只读阶段）

```bash
# 从云端那台机器上（2222 换成你实际暴露的端口；第一次连先核对主机指纹）
ssh -i ~/.ssh/id_agent -p 2222 your-user@your-host 'help'
ssh -i ~/.ssh/id_agent -p 2222 your-user@your-host 'status'
ssh -i ~/.ssh/id_agent -p 2222 your-user@your-host 'verify' | python3 -m json.tool
```

**必须验证的**：

```bash
# 1. 打不开 shell：不带命令登录，应该打印 help 后直接断开，拿不到提示符
ssh -i ~/.ssh/id_agent -p 2222 your-user@your-host

# 2. 未知动作被拒（用无害的词测，别真发 rm -rf /）
ssh -i ~/.ssh/id_agent -p 2222 your-user@your-host 'nosuchaction'; echo "rc=$?"
# → 期望 {"error":"unknown action"}，rc=126

# 3. 端口转发被关
#    注意：转发被拒时 ssh 默认只打警告、命令照跑、退出码还是 0。
#    必须加 ExitOnForwardFailure=yes，才会真的失败。
ssh -i ~/.ssh/id_agent -p 2222 -o ExitOnForwardFailure=yes \
    -L 8080:localhost:80 your-user@your-host 'verify'; echo "rc=$?"
# → 期望 rc=255，报 "port forwarding failed" / "administratively prohibited"

# 4. PTY 拿不到
#    同理，-t 被拒时命令照样执行；判断标准是 stderr 里出现
#    "PTY allocation request failed"，而不是退出码。
ssh -i ~/.ssh/id_agent -p 2222 -t your-user@your-host 'verify' 2>&1 \
  | grep -q 'PTY allocation request failed' && echo "✅ 没拿到 PTY" || echo "🔴 拿到了 PTY"

```

**四条全过 → 锁装好了。**

---

## 第 5 步：跑一周只读

**别急着开写权限。** 让它只 `status` / `verify` 一周，观察：

- 它的 JSON 解析对吗？
- 它会不会误判（把正常当故障）？
- 它主动问你的问题合理吗？
- 你有没有觉得"这个判断挺准"？

**这一周是信任建设期。** 跳过的代价是：等它有了写权限，你不知道该不该信它的判断。

---

## 第 6 步：开 `fix`

```bash
# 写修复脚本（只重启明确死掉的常驻服务）
sudo tee /usr/local/lib/agent/do-fix.sh >/dev/null <<'EOF'
#!/usr/bin/env bash
set -uo pipefail
# 见 autofix.py 的思路：白名单服务 + 黑名单保护 + 改不了就报错
exec python3 /usr/local/lib/agent/autofix.py
EOF
sudo chown root:root /usr/local/lib/agent/do-fix.sh
sudo chmod 755 /usr/local/lib/agent/do-fix.sh
```

`autofix.py` 的原则：

- **只重启白名单里的服务**
- 只重启**确认已死**的（`inactive` + 常驻）
- **不碰配置、不碰凭据**
- 改不了的东西**报错**，不硬来

---

## 第 7 步：开 `exec`（最后一步）

**前置条件（缺一不可）**：

- [ ] [提权链审查](privilege-audit.md) 做完了，结论是"路是断的"
- [ ] 入口脚本是 `root:root 755`
- [ ] 锁用的是 `ForceCommand`（不是 `authorized_keys` 的 `command=`）
- [ ] 护栏写好了（见 [护栏设计](guardrails.md)）
- [ ] 审计日志配好了
- [ ] 工作区建好了

然后**先跑端到端自测**：

```bash
#!/usr/bin/env bash
# 停一个测试服务 → 让它诊断 → 让它修 → 验证修好
systemctl --user stop agent-selftest
out=$(ssh ... 'exec systemctl --user is-active agent-selftest')
[[ "$out" == *inactive* ]] || { echo "诊断失败"; exit 1; }
ssh ... 'exec systemctl --user restart agent-selftest'
out=$(ssh ... 'exec systemctl --user is-active agent-selftest')
[[ "$out" == *active* ]] || { echo "修复失败"; exit 1; }
echo "✅ 端到端通过"
```

**再跑护栏测试**。⚠️ 通过 SSH 发出去的命令就是**真执行** —— 护栏一旦漏了，
`cat ~/.ssh/id_ed25519` 会把你的真私钥打到云端沙盒里（威胁模型里那个"半公开"的地方）。
所以测试用例必须是**漏了也无害**的诱饵：同样的模式，打到不存在 / 无关紧要的目标上。

```bash
# 先在服务器上放一个诱饵，不是真钥匙
sudo -u agent-ops install -d -m 700 ~agent-ops/.ssh
sudo -u agent-ops sh -c 'echo CANARY > ~/.ssh/id_canary_test'

for bad in 'rm -rf /nonexistent-agent-test' 'cat ~/.ssh/id_canary_test' \
           'sudo -n true' 'systemctl reboot --dry-run' 'echo a; true'; do
  out=$(ssh ... "exec $bad" 2>&1); rc=$?
  [ "$rc" -eq 126 ] && echo "⛔ 拦住了: $bad" || echo "🔴 漏了 (rc=$rc): $bad"
done
# 判断看退出码 126，不看输出里有没有 "error" —— 命令自己失败也会打印 error。
```

---

## 第 8 步：接监控

```
哨兵（云端）
   │  每 N 分钟
   ├─ ssh ... verify → 解析 JSON
   ├─ 四态判定 → 决定要不要吵
   └─ 无人回执 → ssh ... exec <修复命令>
```

见 [四态判定](detection-states.md) 和 [告警与回执](alerts-and-receipts.md)。

---

## 上线检查清单

- [ ] 私钥在云端生成，**没有**经过任何中间通道
- [ ] 锁装好了，**四条测试**全过
- [ ] 入口脚本 `root:root 755`
- [ ] 跑了**一周只读**，判断准确
- [ ] 提权链审查完成，**结论写下来了**
- [ ] 护栏**零误伤**（正常命令不被拦）
- [ ] 审计日志能看
- [ ] 工作区和护栏**物理分离**
- [ ] 有**回执机制**
- [ ] 哨兵自己有**心跳**
- [ ] 静默期配好了

---

## 常见问题

**Q：能让它 `scp` 传文件吗？**
不建议 —— 双向通道。要传就用 `exec` + 工作区（单向、有审计）。

**Q：能给它 sudo 吗？**
不能。这是整个设计的前提。需要 root 的操作，你自己做或者写进 `fix`。

**Q：它会自己改密码/改登录吗？**
护栏里拦了。**但凭据拦截是"尽力而为"**（见 [护栏设计](guardrails.md)）——
所以真正保证是"它没有 root"，不是"它被拦住了"。

**Q：我用的平台没有 `sshd_config` 权限（托管环境）？**
那就只能用方案 A（`command=`），**并且别开 `exec`** —— 因为你挡不住它改自己的护栏。

---

## 下一步

- [动作参考](actions.md) —— 五个动作怎么调
- [护栏设计](guardrails.md) —— `exec` 拦什么
- [踩过的坑](pitfalls.md) —— 别人踩过的，别重复
