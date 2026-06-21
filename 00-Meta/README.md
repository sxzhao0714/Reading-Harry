# HarryPotter-Wiki

一个用于"读哈利波特原文 + 学英文 + 沉淀知识"的 Obsidian 项目。

## 目录说明
- `00-Meta/`：规则、模板、AI 提示词
- `01-Books/`：七卷原文，每章一个 md
- `02-Characters/`：50 位核心角色档案（Descriptions / Actions / Quotes 三维度）
- `03-Vocabulary/`：全局生词总表
- `04-Expressions/`：全局精彩表达总表

## 工作流
1. 在章节 md 中阅读原文，使用 English Made Easy 标亮（四色）
2. 读完后通过 TRAE 触发 AI 整理：高亮总结写回章节文件，人物信息追加到角色档案
3. 定期聚合到全局词表与表达表

## 重要原则
- 原文是唯一事实来源，所有衍生信息必须可溯源回原文（章节双链）
- 角色档案"直抽不编撰"，AI 只摘录原文片段，不编写评价
- AI 写入区域由 `<!-- AI-XXX-START -->` 锚点严格圈定，区外内容 AI 永远不碰
