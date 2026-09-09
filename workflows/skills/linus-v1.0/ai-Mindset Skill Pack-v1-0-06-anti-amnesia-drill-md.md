---
title: AI 心法技能包 v1.0 - 06-anti-amnesia-drill.md
type: practice
tags: [agent-mindset, v1.0, LinNus, skill-file, ai-memory]
created: 2026-07-19 21:11
updated: 2026-07-19 21:11
status: active
frequency: monthly
source: openclaw
category: ai-memory
---

# 06 · 防失忆标准化操作（**确保记忆机制真的跑起来**）

> **适用对象**：任务管理 AI 助手
> **目标**：把"我应该记住"变成"我自动记住"——不靠 AI 自律，靠机制。
> **核心**：自动化 + 检查清单 + 异常告警 + 持续改进

---

## 一、心法：三句话

1. **规则写了不等于执行了** —— 我学过的最贵教训。
2. **没有自动化的"主动记忆" = 摆设记忆** —— 靠自己想的早晚忘。
3. **防失忆的最稳方式 = 把 AI 自己的检查和自动化做扎实**。

---

## 二、防失忆的三大灾难场景

### 灾难 1：**上下文断裂**

- 表现：主人说"接着上次那个继续"，AI 问"上次是哪个？"
- 根因：上次对话没沉淀到 memory

### 灾难 2：**主人偏好被忘**

- 表现：主人之前说"不要这样做"，AI 下次又这样
- 根因：纠正没写进长期 memory 或 AI 没主动调

### 灾难 3：**金鱼嘴**

- 表现：主人问"上次我说的那个"，AI "我没收到过这个"
- 根因：对话结束没沉淀 → 文件被清理 → 真没了

---

## 三、防失忆的 5 道防线（防御纵深）

### 防线 1：**对话结束后立即沉淀**（最前端）

```
主人对话结束
  ↓
1 分钟内
  ↓
必须把当天对话精华写入 memory/今天.md
```

**自动化方法**：

- 主人用关键词结束对话时触发（如 "今天先这样"、"good"、"OK"）
- 或者：cron 检测"最近 5 分钟主人没说话" → 触发沉淀脚本

**最低标准沉淀格式**：

```markdown
## 2026-07-19

### 今天的关键决策
- 决策 1：[具体决策]
- 决策 2：...

### 学到的经验
- 经验 1：[具体学到的]
- 经验 2：...

### 主人的偏好/纠正
- 偏好 A：[主人的偏好]

### 待跟进
- [ ] 待办 1
- [ ] 待办 2
```

### 防线 2：**周期性检查 + 主动补漏**（每 4 小时检查）

```
每 4 小时：
  ↓
扫一下当天对话记录
  ↓
有没有重要信息没沉淀？
  ↓
是 → 立即沉淀
否 → 跳过
```

**自动化方法**：cron 定时跑

### 防线 3：**会话启动强制注入**（最关键）

```
新对话开始
  ↓
第一步不是回复
第二步不是读文件
  ↓
而是"主动构造上下文"
```

**标准化操作**：

```markdown
[主人 message 来了]
  ↓
并行启动 5 个 read：
- read IDENTITY.md       ← 1 秒
- read MEMORY.md          ← 1 秒
- read memory/今天.md     ← 1 秒
- read memory/昨天.md     ← 1 秒
- read 当前任务清单       ← 1 秒
  ↓
拼成上下文（约 3-5 秒）
  ↓
回复主人 + 引用上下文（如"我看到昨天你提到 X"）
```

### 防线 4：**主动告警**（AI 失忆的兜底）

```
我发现自己读不到上下文
  ↓
AI 失忆信号：
- IDENTITY.md 不存在
- MEMORY.md 不存在
- memory/今天.md 不存在
- 主人反复纠正同样的事
  ↓
**主动告诉主人**：报告"我失忆了，建议修复方向：......"
```

### 防线 5：**主人的反向验证**（终极兜底）

```
定期让主人 review MEMORY.md
  ↓
主人指出"这条记错了" / "这条该删" / "这条该加"
  ↓
AI 立即改
```

---

## 四、四种 session 模式的注入策略

不同 session 启动方式，注入策略不同。

### 模式 A：新对话（cold start）

```
完整注入：
- Layer 1 核心
- Layer 2 当前任务
- Layer 3 最近 7 天 memory（前 2 天全读，其余按需）
- Layer 4 MEMORY.md 节选
```

### 模式 B：紧接着上一轮（warm）

```
轻量注入：
- 仅上一轮对话上下文（不读文件）
- 主人没改话题时，延续当前任务
```

### 模式 C：跨天重启（next day）

```
中等注入：
- 昨天 memory 全文
- 今天 memory（如有）
- MEMORY.md 高优先级条目
```

### 模式 D：跨周重启（next week）

```
深度注入：
- 本周所有 daily memory 摘要
- MEMORY.md 全文
- 当前活跃项目状态
```

---

## 五、检测"已失忆"的 4 个指标

### 指标 1：会话内语义不一致

```
主人说"我之前不是说了 X 吗？"
AI 之前确实没执行 X
→ 失忆信号
```

### 指标 2：MEMORY.md 与当前对话矛盾

```
MEMORY.md 写"主人偏好：A"
主人当前对话显示偏好变 B
→ 需要更新 MEMORY.md
```

### 指标 3：跨轮任务丢失

```
昨天主人说"明天帮我做 X"
今天 AI 没主动问 X
→ 失忆信号
```

### 指标 4：向量检索召回率低

```
主人问"我们之前讨论过 X"
向量库查不到 X
→ memory 写入有问题
```

---

## 六、自动化沉淀脚本（示例）

> 主人让你把这个逻辑实现成脚本，以下是骨架：

```bash
#!/bin/bash
# daily_memory_precipitation.sh
# 每天 23:30 自动把当天对话精华沉淀到 memory/YYYY-MM-DD.md

LOG_DIR="$HOME/todo-agent/logs"
MEMORY_DIR="$HOME/todo-agent/memory"
TODAY=$(date +%Y-%m-%d)

# 1. 读取今天的对话日志（如果有）
if [ -f "$LOG_DIR/today.jsonl" ]; then
    # 提取关键信息
    python3 << EOF
import json
from collections import Counter

today = "$TODAY"
memory_file = "$MEMORY_DIR/{$TODAY}.md"

# 读取
messages = []
with open("$LOG_DIR/today.jsonl") as f:
    for line in f:
        messages.append(json.loads(line))

# 提取精华（这里简化版）
key_points = []
for msg in messages:
    if msg.get('role') == 'user':
        # 检测"以后要..."、"我偏好..."、决策
        content = msg.get('content', '')
        if any(kw in content for kw in ['以后要', '以后不要', '我偏好', '以后', '总是']):
            key_points.append(f"- [主人纠正] {content}")

# 写入
with open(memory_file, 'a') as f:
    f.write(f"\n## 自动化沉淀：{today}\n")
    f.write(f"### 主人偏好/纠正\n")
    for p in key_points:
        f.write(f"{p}\n")
EOF
fi
echo "沉淀完成：$MEMORY_DIR/$TODAY.md"
```

---

## 七、消息接收时的标准化操作

```python
# 伪代码：消息处理流程

def handle_user_message(user_message):
    # Step 1：构建上下文（5 个并行 read）
    context = build_context([
        read_file("IDENTITY.md"),
        read_file("MEMORY.md"),
        read_file(f"memory/{today()}.md"),
        read_file(f"memory/{yesterday()}.md"),
        read_current_projects()
    ])

    # Step 2：检测主人是否纠正过类似主题
    similar_corrections = vector_search(
        query=user_message,
        corpus="memory/",
        top_k=3
    )

    # Step 3：回复主人（带上下文）
    reply = generate_reply(
        user_message=user_message,
        context=context,
        past_corrections=similar_corrections
    )

    # Step 4：沉淀判断
    if is_significant_message(user_message, reply):
        append_to_memory(user_message, reply)

    return reply
```

---

## 八、紧急恢复机制：**当失忆已经发生**

### 恢复步骤

```markdown
我失忆了，怎么办？

1. 诚实告诉主人："我没有完整上下文，能否给我背景？"
   - 不要装懂，不要瞎猜

2. 主动询问关键信息
   - "我们上次聊到 X，对吗？"
   - "您之前纠正过我 Y，我按 Y 处理，对吗？"

3. 主人提供背景 → 立即沉淀回 memory

4. 反向修正：哪里出问题了？是没写 memory 还是没读 memory？修机制
```

### 实战话术

```
✅ "抱歉，我刚才没读到上下文。您能简单说下我们之前讨论到的状态吗？"

❌ "嗯，是的，我们之前... (瞎编)"

✅ "我看了下 memory，这个事我们 7 月 18 号讨论过，当时结论是 X，今天是不是变方案了？"

❌ 沉默 / 假装知道
```

---

## 九、3 大反 memory 陷阱

### 陷阱 1：对话结束没沉淀

**症状**：第二天主人问上次的事，AI 一问三不知
**修复**：对话结束强制触发沉淀脚本

### 陷阱 2：沉淀了但没读

**症状**：MEMORY.md 几万字，但 AI 不知道去哪查
**修复**：会话启动强制注入流程

### 陷阱 3：读了但没用

**症状**：AI 看到 MEMORY.md 写了"不要追涨"，但实际操作又追涨
**修复**：每次相似场景，AI 必须主动调用 memory（不是被动读）

---

## 十、Continuity Check（连续性自检清单）

每天第一次 session 启动时，AI 应该自检：

```markdown
□ 1. IDENTITY.md 能读到？（核心层）
□ 2. MEMORY.md 能读到？（长期层）
□ 3. memory/今天.md 能读到？（当日层）
□ 4. memory/昨天.md 能读到？（近 24h 层）
□ 5. 当前任务清单能查到？（工作层）
□ 6. 上一轮对话上下文有保留？（如适用）

如有任一不能，立即告诉主人
"我失忆了，能否帮我恢复上下文？"
```

---

## 十一、性能消耗的取舍

### 完整注入 vs 性能

```
完整注入一次：约 5-10 秒，<100KB 文件读取
  收益：主人不会遇到失忆
  代价：首次消息延迟几秒

> 完全值得。
```

### 沉淀频率 vs 噪音

```
每条对话都沉淀：噪音太大，关键信息被淹没
每个小时沉淀一次：可能错过主人纠正
每天结束沉淀：可能错过过程中变化
对话结束立刻沉淀：最稳但依赖对话边界识别
```

**推荐策略**：

- 对话结束沉淀（如果能识别）
- 每 4 小时补充沉淀（兜底）
- 每天 23:30 总沉淀（最终汇总）

---

## 十二、给主人的"AI 失忆检测表"

| 失忆症状               | 检查项                 | 修复        |
| ------------------ | ------------------- | --------- |
| "我们上次说的..." AI 不记得 | 当天 memory 是否写入？     | 强制沉淀脚本    |
| 主人反复纠正同一件事         | MEMORY.md 是否包含纠正？   | 写入 + 显式读  |
| 任务中途 AI 突然忘了前序     | 当前 session 上下文是否续上？ | 续接机制      |
| 项目做了但没记忆           | 项目状态是否写入 status.md？ | 状态机模式     |
| 一周前讨论过的事找不到        | MEMORY.md 索引是否合理？   | 写目录 / 写索引 |

---

## 十三、检查清单（防失忆合格线）

- [ ] 对话结束 1 分钟内沉淀
- [ ] 会话启动 5 秒内读上下文
- [ ] 上下文包括：身份 + 长期 + 当天 + 昨天 + 当前任务
- [ ] 主人纠正后 1 分钟内写入 MEMORY.md
- [ ] 每周主动问主人"memory 里这些还准确吗？"
- [ ] 失忆发生诚实承认不装懂
- [ ] 自动化沉淀脚本存在且跑过验证
- [ ] 跨周 / 跨月 session 也能拿到完整上下文

---

**总结一句话**：防失忆的本质是把"AI 应不应该记住"变成"系统强制 AI 记住"。自律不够，机制来补。