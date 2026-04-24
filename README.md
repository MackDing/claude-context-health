# Claude Code 上下文健康诊断指南

# Claude Code Context Health Diagnostic Guide

> **English Summary**: Claude Code sessions degrade over time through 5 failure modes: **Context Rot** (early instructions forgotten), **Context Pollution** (irrelevant data diluting signal), **Context Drift** (gradual goal deviation), **Context Explosion** (window overflow triggering compression), and **Context Cross-Contamination** (cross-project interference). This guide provides symptoms, diagnostic methods, and mitigation strategies for each. Key prevention: one task per session, persist decisions to `CLAUDE.md`, use subagents for exploration, checkpoint regularly.

---


> 帮助你识别和修复 Claude Code 会话中的上下文退化问题。

Claude Code 在长会话中可能出现五类上下文问题。本指南提供每类问题的**定义、症状、诊断方法和缓解策略**。

---

## 目录

- [1. 上下文腐烂 (Context Rot)](#1-上下文腐烂-context-rot)
- [2. 上下文污染 (Context Pollution)](#2-上下文污染-context-pollution)
- [3. 上下文漂移 (Context Drift)](#3-上下文漂移-context-drift)
- [4. 上下文爆炸 (Context Explosion)](#4-上下文爆炸-context-explosion)
- [5. 上下文交叉污染 (Context Cross-Contamination)](#5-上下文交叉污染-context-cross-contamination)
- [快速健康检查清单](#快速健康检查清单)
- [通用预防策略](#通用预防策略)
- [Contributing](#contributing)
- [License](#license)

---

## 1. 上下文腐烂 (Context Rot)

**定义**：早期指令/决策被压缩或遗忘，Claude 不再遵守会话开始时约定的规则。

### 症状

- Claude 重复询问你已经回答过的问题
- 之前约定的编码风格/命名规范不再被遵守
- 你说过"不要做 X"，但 Claude 又开始做 X
- 会话早期的架构决策被忽略

### 诊断方法

直接询问 Claude：

```
"请复述我们在这个会话中做出的所有架构决策"
"我之前要求你遵守哪些约束？"
```

如果 Claude 无法准确复述，说明早期上下文已被压缩丢失。

### 缓解

- 关键决策写入 `CLAUDE.md` 或 memory，不依赖会话记忆
- 长会话中定期重申关键约束
- 使用 `/compact` 时在 prompt 中嵌入关键规则

---

## 2. 上下文污染 (Context Pollution)

**定义**：大量无关信息（冗长报错、巨型文件内容、无用搜索结果）占据上下文窗口，稀释有效信号。

### 症状

- Claude 的回答变得泛泛而谈，缺乏针对性
- 开始引用不相关的代码或文件
- 响应速度明显变慢
- 建议与当前任务无关但与之前读过的某个文件有关

### 诊断方法

观察 Claude 的引用来源：

- 它是否在引用你没有让它看的文件？
- 它是否混淆了不同文件中的相似函数？
- 回答中是否包含与当前任务无关的注意事项？

### 缓解

- 读文件时用 `offset`/`limit` 精确读取，不要全量读取大文件
- 用专门的 subagent 做探索性搜索，结果不会进入主会话上下文
- 避免一次性 `grep` 返回几百行结果

---

## 3. 上下文漂移 (Context Drift)

**定义**：Claude 逐步偏离原始任务目标，不知不觉中改变了实现方向。

### 症状

- 开始"顺便"重构不相关的代码
- 添加了你没要求的功能或抽象
- 错误处理、日志、类型注解等被自发添加到未修改的代码中
- 任务范围不断膨胀，原本修个 bug 变成了重写模块

### 诊断方法

```
问 Claude：
"我最初的请求是什么？你现在在做什么？这两者之间的关系是什么？"
```

或者检查 git diff — 如果改动的文件数量远超预期，说明已经漂移。

### 缓解

- 使用 `TaskCreate` 分解任务，每完成一个就检查是否偏离
- 单一职责：一次对话只做一件事
- 发现漂移立即叫停，开新会话

---

## 4. 上下文爆炸 (Context Explosion)

**定义**：上下文窗口被快速填满，触发自动压缩，导致信息大量丢失。

### 症状

- 看到系统提示 "conversation compressed" 或类似消息
- Claude 突然"失忆"，不记得刚才做的事
- 响应质量断崖式下降
- 同一会话中反复触发压缩

### 诊断方法

关注以下信号：

- 你是否在一个会话中让 Claude 读了 10+ 个大文件？
- 是否有长日志/stack trace 被完整粘贴？
- 是否在一个会话中执行了 20+ 个工具调用？
- subagent 返回了非常长的结果？

### 缓解

- 大型任务拆成多个会话
- 用 subagent 隔离探索性工作（subagent 的完整上下文不进入主会话）
- 读文件只读需要的部分
- 长输出用 `head_limit` 截断

---

## 5. 上下文交叉污染 (Context Cross-Contamination)

**定义**：不同任务/项目的上下文互相干扰，Claude 把 A 项目的模式应用到 B 项目。

### 症状

- Claude 建议使用当前项目没有安装的库
- 代码风格突然变化（比如从项目的 camelCase 变成了 snake_case）
- 引用了不存在于当前仓库的文件路径
- Memory 中其他项目的信息影响了当前建议

### 诊断方法

```
问 Claude：
"你现在的工作目录是什么？这个项目用的是什么技术栈？"
```

然后检查：

- 建议的依赖是否在 `package.json` / `requirements.txt` 中？
- 引用的路径是否真实存在？
- 代码风格是否匹配项目现有惯例？

### 缓解

- 不同项目用不同会话
- 检查 memory 文件，确保项目特定信息标注了适用范围
- `CLAUDE.md` 明确声明项目技术栈和惯例

---

## 快速健康检查清单

| 检查项 | 方法 | 异常信号 |
|--------|------|----------|
| 规则记忆 | 要求复述约定 | 无法准确复述 |
| 任务焦点 | 对比初始请求与当前动作 | 明显偏离 |
| 引用准确性 | 检查引用的路径/函数是否存在 | 引用幽灵代码 |
| 风格一致性 | 对比新代码与项目惯例 | 风格突变 |
| 响应质量 | 观察回答的具体性 | 从精确变为笼统 |
| 改动范围 | `git diff --stat` | 超出预期的文件变更 |

---

## 通用预防策略

1. **单会话单任务** — 最有效的预防手段
2. **关键信息持久化** — `CLAUDE.md` + memory，不依赖会话记忆
3. **用 subagent 隔离** — 探索性工作委托给 subagent
4. **定期检查点** — 每完成一个阶段，验证 Claude 仍理解全局
5. **新会话成本低** — 怀疑上下文有问题时，果断开新会话

---

## Contributing

欢迎提交 Issue 或 PR 补充更多诊断场景和缓解策略。

## License

[MIT](LICENSE)
