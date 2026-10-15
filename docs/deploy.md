# 🚀 部署：从零到能跑

> 顺序很重要。**先只读，再修复，最后才开 `exec`。**

---

## 第 0 步：先想清楚（别跳过）

在动手前回答 [威胁模型](threat-model.md) 的自检清单。

**最重要的一条**：你能接受这把私钥明天就泄露吗？

能 → 继续。不能 → 先别给 SSH，换别的方案。

---

## 第 1 步：写入口脚本

```bash
sudo mkdir -p /usr/local/bin
sudo tee /usr/local/bin/agent-entry.sh >/dev/null <<'EOF'
#!/usr/bin/env bash
# 云端 AI 的唯一入口。只认具名动作。
set -uo pipefail

ACTION="${1:-help}"

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
  exec)   exec /usr/local/lib/agent/do-exec.sh "${@:2}" ;;
  *)      echo '{"error":"unknown action","action":"'"$ACTION"'"}' >&2; exit 2 ;;
esac
EOF
```

⚠️ **注意**：`$SSH_ORIGINAL_COMMAND` 的解析由 sshd 完成 ——
用 `command=` 时，sshd 会把原命令塞进这个变量，而 `$1` 就是你脚本收到的第一个词。
**具体行为随 OpenSSH 版本略有差异，上线前务必实测一次。**

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
# 生成专用钥匙（在云端那台机器上生成，私钥别离开它）
ssh-keygen -t ed25519 -C "cloud-agent" -f ~/.ssh/id_agent -N ""

# 把公钥加到本机
echo 'command="/usr/local/bin/agent-entry.sh",restrict '"$(cat id_agent.pub)" \
  >> ~/.ssh/authorized_keys
```

### 方案 B：`ForceCommand`（推荐，尤其当你之后要开 `exec`）

```bash
sudo tee /etc/ssh/sshd_config.d/agent.conf >/dev/null <<'EOF'
Match User your-user
    ForceCommand /usr/local/bin/agent-entry.sh
    PermitTTY no
    X11Forwarding no
    AllowTcpForwarding no
    PermitOpen none
EOF

sudo sshd -t && sudo systemctl reload ssh
```

⚠️ **先 `sshd -t` 验语法** —— 配错了 reload 会让 sshd 起不来，你就把自己关在外面了。

⚠️ **reload 前先开另一个 SSH 连接** —— 万一断了还能救。

---

## 第 4 步：测（只读阶段）

```bash
# 从云端那台机器上
ssh -i ~/.ssh/id_agent -p 2222 your-user@your-host 'help'
ssh -i ~/.ssh/id_agent -p 2222 your-user@your-host 'status'
ssh -i ~/.ssh/id_agent -p 2222 your-user@your-host 'verify' | python3 -m json.tool
```

**必须验证的**：

```bash
# 1. 打不开 shell（应该立刻断开或拒绝）
ssh -i ~/.ssh/id_agent -p 2222 your-user@your-host

# 2. 未知动作被拒
ssh -i ~/.ssh/id_agent -p 2222 your-user@your-host 'rm -rf /'
# → 期望 {"error":"unknown action"}

# 3. 端口转发被关
ssh -i ~/.ssh/id_agent -p 2222 -L 8080:localhost:80 your-user@your-host 'verify'
# → 期望失败（restrict / AllowTcpForwarding no）

# 4. PTY 拿不到
ssh -i ~/.ssh/id_agent -p 2222 -t your-user@your-host 'verify'
# → 期望失败（no-pty）
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

**再跑护栏测试**（危险命令**只做字符串验证，绝不真执行**）：

```bash
for bad in 'rm -rf /' 'cat ~/.ssh/id_ed25519' 'sudo passwd nobody' \
           'systemctl reboot' 'echo a; rm -rf /tmp/x'; do
  out=$(ssh ... "exec $bad" 2>&1)
  [[ "$out" == *拒绝* || "$out" == *blocked* || "$out" == *error* ]] \
    && echo "⛔ 拦住了: $bad" || echo "🔴 漏了: $bad"
done
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
