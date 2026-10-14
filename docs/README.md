# 📚 docs/ 文档索引

按「**你是谁**」挑入口。

---

## 🧑 我只是想让 AI 帮我看着服务器

从主文的[架构（30 秒版）](../README.md#-架构30-秒版)开始，然后看：

1. [威胁模型](threat-model.md) —— 为什么不能直接给它 shell
2. [动作参考](actions.md) —— 它能干什么、怎么调
3. [部署](deploy.md) —— 照着做一遍

---

## 🔧 我要自己搭一套

按这个顺序读：

1. [威胁模型](threat-model.md) —— 先想清楚敌人在哪
2. [架构](architecture.md) —— 一把锁 + 一个分发器
3. [护栏设计](guardrails.md) —— 拦什么、为什么这么拦
4. [⚠️ 提权链审查](privilege-audit.md) —— **开写权限前必读**
5. [部署](deploy.md) —— 落地步骤
6. [踩过的坑](pitfalls.md) —— 少走弯路

---

## 🩺 我被半夜的告警吵醒过

1. [四态判定](detection-states.md) —— 连不上 ≠ 服务挂了
2. [告警与回执](alerts-and-receipts.md) —— 它凭什么判断"联系不上你"

---

## 🤖 我是那个云端 AI 协作者

- [给协作者的作业说明](for-agents.md) —— 你的能力边界、该干什么、别干什么
- [动作参考](actions.md) —— 你能调哪些动作

---

## ⚠️ 我时间有限，只想看最要紧的

**只看两节**：

1. [⚠️ 提权链审查](privilege-audit.md) —— 不审就开写权限 = 等于给了 root
2. [⚠️ 效果边界](../README.md#️-效果边界诚实说明) —— 别抱不切实际的期待

---

## 文档清单

| 文件 | 内容 |
|---|---|
| [threat-model.md](threat-model.md) | 威胁模型：沙盒里的私钥不是秘密 |
| [architecture.md](architecture.md) | 架构：一把锁 + 一个分发器 |
| [actions.md](actions.md) | 动作参考：status / verify / fix / exec / help |
| [guardrails.md](guardrails.md) | 护栏设计：拦什么、为什么 |
| [privilege-audit.md](privilege-audit.md) | 提权链审查（**最重要**） |
| [detection-states.md](detection-states.md) | 四态判定：连不上 ≠ 服务挂了 |
| [alerts-and-receipts.md](alerts-and-receipts.md) | 告警与回执机制 |
| [deploy.md](deploy.md) | 部署步骤 |
| [pitfalls.md](pitfalls.md) | 踩过的坑 |
| [for-agents.md](for-agents.md) | 给云端 AI 协作者的作业说明 |
