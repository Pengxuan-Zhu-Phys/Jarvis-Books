---
name: skill-slug
title: 一句话意图（用户会说出口的那句话）
intent: "我想……"
triggers: [关键词1, 关键词2]
level: beginner | intermediate | advanced
verified: YYYY-MM-DD @ <branch/commit>
---

# <title>

## 目标

一两句：做完这个 skill 你会得到什么。

## 前提

- 已安装（`Jarvis2 -v` 有输出）、本地 Redis 已启动
- 其他前提……（没有就删掉这节）

## 复制即用

```yaml
# 最小可运行卡片 — 必须实测验证后才能提交
```

```bash
Jarvis2 run your_card.yaml
```

## 它做了什么

≤5 行，讲这张卡的执行流，不讲架构。

## 常见改动

| 想要…… | 改这里 | 详见 |
|---|---|---|
| …… | `Key: value` | YAML_REFERENCE §x.y |

## 常见坑

- **症状** → 原因 → 一句话修法。
