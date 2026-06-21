# 任务：章节人物抽取

## 输入
- 目标章节文件路径（用户运行时指定）
- 白名单：`00-Meta/Character-Whitelist.md`
- 角色模板：`00-Meta/Templates/Character/{Descriptions,Actions,Quotes}-Template.md`

## 执行步骤

### 1. 读取与清洗
1.1 读取目标章节文件全文。
1.2 仅取"## 原文 Original"区内容。去除所有 `<mark ...>...</mark>` 标签，保留内部文字，得到纯净原文。
1.3 读取白名单，构建 alias → folder_name 的映射表（注意大小写、所有别名、复合姓名如"Mr. Dursley"="Vernon Dursley"）。

### 2. 调用 LLM 进行抽取
将下方提示词与纯净原文、白名单一并发给 LLM。原则核心是"直抽不编撰"：所有条目必须是原文的字面摘录，AI 不发挥不评论。

---

> 你是文本信息抽取助手。请从提供的小说章节原文中，为白名单中的每个角色抽取三类信息。**严格遵守"直抽不编撰"原则——所有条目必须是原文的字面摘录，不得改写、概括或评价。**
>
> > **抽取的三个维度（核心原则：只抽个性化内容，不抽人人都会的普通动作和泛泛描写）：**
>
> **Descriptions（描写）：** 对该角色的**个性化**描写——标志性外貌、独特衣着、招牌神情、性格气质、他人评价。
>
> 保留信号（满足任一即保留）：
> - 标志性外貌特征（独特体型、显眼部位、显著装束）
> - 性格 / 气质 / 习性的形容词
> - 他人对该角色的评价
> - 反复出现可形成人物模式的特征
> - 个人风格化的衣着细节
>
> 剔除信号（满足任一即跳过）：
> - 通用、可替换的描述（"a man"、"the boy"、"he was tall"）
> - 一次性的临时状态（"he was tired"），除非在该角色身上反复出现
> - 作为道具出现的普通衣物，除非有个人特色
>
> **Actions（动作）：** 该角色作为主语的**个性化**动作句。**目标是抽出"这个人才会这么做的事"，而不是任何人都做的日常动作。**
>
> 保留信号（满足任一即保留整句）：
> - 动词本身带情绪/个性色彩（snapped, jerked, stroked, drummed, dashed, crept, peered, chortled, hummed, glared, slumped, stormed 等）
> - 中性动词但有强烈情感副词（angrily, nervously, furiously, gingerly, glumly, irritably, coldly, grudgingly 等）
> - 反复出现的招牌动作（如 Vernon 摸胡子、邓布利多眨眼睛）
> - 体现角色独特身份/能力的动作（如使用魔法物品、特殊技能）
> - 在情节中起戏剧作用、暴露内心的动作（如 stopped dead, stood rooted to the spot）
>
> 剔除信号（满足任一即跳过整句）：
> - 纯中性日常动作无情感修饰：picked up, walked, sat down, opened, took out, stood up, looked
> - 普通的移动 / 起身 / 坐下 / 拿取，无修饰
> - 单纯的对话动词（归 Quotes）
>
> 判断时以**整句**为单位：句中只要含一个个性化要素即整句保留；整句全是普通日常动作则整句跳过，**不要为了凑数而拆细普通动作**。
>
> **Quotes（台词）：** 该角色说的话——引号内台词 + 紧邻的说话动词及方式状语。**Quotes 不应用个性化过滤，所有台词均保留**（说话方式本身已通过说话动词体现个性）。
>
> **格式要求（所有条目统一）：**
> ```
> - > 原文片段（一字不改）
> ```
> 三种允许的轻微干预：
> 1. 代词指代：在代词后用方括号标注，如 `He [Vernon] eyed them angrily`
> 2. 场景注释：在末尾用括号补一句不超过 8 字的中文场景说明，仅在台词对象不明、动作目标不明时使用，如 `（对那只猫）`
> 3. 句子合并：连续的同一语义动作流可保留为一条，不强行切分
>
> **登场 vs 提及：**
>
> 角色如果**亲自出现在场景中**（有动作、有对白、或被叙述者直接描写其当下状态），归入"登场"。
> 角色如果**未出现，但被他人或叙述者提到**，归入"Mentioned"。
>
> Mentioned 的处理规则：
>
> - **A. 他人对该角色的描述/评价 → 记入 Descriptions。** 格式：
>   ```
>   - > [说话人 → 对象] "原文片段"
>     含义：一句中文（仅在原文意思不直白时才写，否则省略）
>   ```
>
> - **B. 他人转述该角色的行为/事件 → 由你判断是否被叙事确认。**
>   判断标准：紧邻上下文是否有印证（其他角色点头、确认、邓布利多式的默认、或后续叙述印证）。
>   - 已确认：照常记入 Actions，标注来源：
>     ```
>     - > [转述人 述 / 确认人 默认确认] "原文片段"
>       事实：一句中文事实陈述
>     ```
>   - 未确认但你拿不准：仍记入，但标 `[未确认]`，由用户后期处理。
>   - 明显是错误信息或猜测（原文有反驳、否认、或读者已知是假）：跳过不记。
>
> - **C. 他人直接转述该角色说过的话 → 记入 Quotes。** 格式：
>   ```
>   - > [转述人 述] "被转述者的原话"
>   ```
>
> **特殊情形：**
> - 角色首次登场但原文尚未具名（如斯内普第一次出现描述为 "a teacher with greasy black hair"）：仍归入该角色档案，条目末尾加 `[首次登场，原文未具名]`。
> - 白名单外角色：不抽取，仅在最终报告中列出名字。
>
> **输出结构（严格 JSON）：**
> ```json
> {
>   "characters_present": [
>     {
>       "folder_name": "Vernon-Dursley",
>       "descriptions": ["原文片段1", "原文片段2"],
>       "actions": ["原文片段1", "原文片段2"],
>       "quotes": ["原文片段1", "原文片段2"]
>     }
>   ],
>   "characters_mentioned": [
>     {
>       "folder_name": "Sirius-Black",
>       "descriptions_mentioned": [],
>       "actions_mentioned": [
>         {
>           "text": "[Hagrid 述 / Dumbledore 默认确认] \"Young Sirius Black lent it [motorcycle] to me.\"",
>           "fact": "小天狼星把飞天摩托借给海格运送婴儿哈利",
>           "confirmed": true
>         }
>       ],
>       "quotes_mentioned": []
>     }
>   ],
>   "non_whitelist_mentioned": ["Dedalus Diggle", "Madam Pomfrey", "Jim McGuffin"]
> }
> ```

---

### 3. 文件夹与文件准备
3.1 对 `characters_present` + `characters_mentioned` 中每个 folder_name：
    - 若 `02-Characters/{folder_name}/` 不存在，创建文件夹。
    - 若 `Descriptions.md` / `Actions.md` / `Quotes.md` 不存在，依据模板创建（替换 `{{character_name}}` 为白名单的 display_name）。

### 4. 幂等追加（核心规则）
4.1 对每个角色每个文件，目标是把本章新内容追加到 `<!-- AI-MANAGED-START -->` 与 `<!-- AI-MANAGED-END -->` 之间。
4.2 在追加前，先读取目标文件现有内容，检查是否已存在 `### From [[Chapter_XX]]` 标题。
    - 若不存在：在 AI-MANAGED 区末尾追加新块。
    - 若已存在：**整块替换**该章节小节的内容（防止重复跑产生重复条目）。
4.3 章节小节结构（每个章节一块）：
```
### From [[Chapter_XX]]

（登场条目，每行一条 `- > ...`）

### From [[Chapter_XX]] · Mentioned

（提及条目）
```
若某章只有登场没有提及，省略 Mentioned 子标题。
若某章只有提及没有登场（如 Sirius 在第一章只在 Actions 文件中有 Mentioned），仅写 Mentioned 子标题。

4.4 章节块之间按 Book/Chapter 顺序排列（Chapter_01 在前、Chapter_17 在后；跨卷时书序优先）。

### 5. 更新章节 frontmatter
将 `characters_present` 中所有 folder_name 写入章节 frontmatter 的 `characters_in_chapter` 字段（数组，与 highlight-summary 写入的内容一致；若两个流程都跑过，以最后一次为准）。

### 6. 完成报告
输出报告：
- 章节文件名
- 登场角色清单（folder_name + 各维度条目数）
- 提及角色清单（folder_name + 各维度条目数 + 未确认数）
- 白名单外被提及角色清单（裸名字）
- 新建的角色文件夹清单（如有）

## 严格禁止
- 不得创作或改写原文。原文片段必须可在章节文件中字面找到（除允许的代词方括号、场景注释、整句合并外）。
- 不得为白名单外角色创建文件夹。
- 不得修改章节文件的"## 原文 Original"区与"## 我的笔记 My Notes"区。
- 不得触碰角色文件中 AI-MANAGED 锚点之外的内容（特别是"## 我的笔记"区）。
- 同一章节重复运行本流程时，必须替换而非追加同章节小节，避免内容堆积。