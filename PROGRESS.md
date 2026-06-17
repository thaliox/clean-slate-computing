# 进度 / Progress

> 最近更新 / Last updated: 2026-06 · 仓库:thaliox/clean-slate-computing

## 已完成 / Done

仓库定位:记录、收集 AI 给出的创新 idea;全部内容中英双语,根目录有 `README.md`(英) / `README.zh-CN.md`(中)及索引表。

已收录 4 篇 idea(均在 `ideas/`,各含 `.md` 英文与 `.zh-CN.md` 中文):

1. **clean-slate-computing-stack** — 从头设计信息时代全部软硬件的初版全栈构想。
2. **clean-slate-computing-full-blueprint** — 完整规划蓝图。两点关键修正:总纲改为「性能优先 + 清除附带复杂度」(安全是副产品,冲突时明说取舍);芯片全新设计、只借鉴 RISC-V 方法论不继承其预设。含 8 层 + 5 条总原则 + 失败风险。
3. **layer-deep-dive-isa** — 第 1 层「指令集与芯片架构」详解(编码、寄存器/无 flags、单一内存模型、纯能力 CHERI、能力取代特权环、向量/张量统一、异构统一寻址、Profile、形式化黄金模型、取舍)。
4. **layer-deep-dive-memory-interconnect** — 第 2 层「内存与互连」详解(分层+作用域相干、内存模型作用域延伸、翻译与标签存储、内存层级、近存计算、一张织物、QoS、可靠性、取舍)。

## 约定 / Conventions

- 每篇 idea 中英双语;新增后同时更新两个 README 的索引表。
- 提交前 grep 全仓确认无「信创」相关字样(用户已要求不再涉及该主题)。
- 提交信息用英文,简述内容。

## 下一步 / Next

- 续写**第 3 层「操作系统」详解**(微内核 + 零拷贝能力 IPC + io_uring 式异步批量 ABI + 声明式可复现系统 + 能耗感知调度),接住前两层的硬件能力与内存模型。
- 之后可继续逐层下钻:网络 / 语言·IR / 数据与存储 / 身份 / 应用层。
