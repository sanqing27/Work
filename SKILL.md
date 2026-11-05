---
name: caa-developer
description: Use when the user wants to generate, write, or review CAA code, CATIA 3DE secondary development, CAA API lookup, or industrial part modeling C++ code. Whenever the user asks to generate or validate 3DE CAA / CAA C++ code, this skill should be used to query the local documentation and must never fabricate APIs.
---

## 前置：config.txt 与缓存

### config.txt
- 位置：skill 目录下 `config.txt`，首行 `visidx_path=<绝对路径>` 指向本机 3DE CAA 文档总索引 `visidx.txt.htm`。
- 每次会话开始先读首行，得到 visidx 绝对路径；后续所有页面路径均相对其所在目录（refman）解析。
- 若 config.txt 缺失、首行格式错误，或指向的 `visidx.txt.htm` 无法定位 → 按「错误处理 · config.txt 缺失 / 格式错误 / 路径不可用 → 由 Agent 获取路径并生成」处理。

### 术语约定（全 skill 统一）
- **接口文档**：`generated\refman\` 下的 C++ API 参考（类/方法签名页），索引源 `visidx.txt.htm` → `cache/caa_index.tsv`。
- **CAA 开发说明文档**：`English\<Module>\` 下的模块/指南说明页（技术文章/用例/速查），索引源各模块 `mkidx_mod_idx_caa.xml` → `cache/caa_desc_index.tsv`。
- 接口文档回答"有哪些接口、签名是什么"；CAA 开发说明文档回答"怎么用、最佳实践、用例"。

### 定位器缓存（两套，均"缓存仅定位器"）
- **接口文档定位器** `cache/caa_index.tsv`：6 列 `symbol|type|qualified_name|page_path|anchor|description`；解析算法见 `references/cache-schema.md`。
- **开发说明文档定位器** `cache/caa_desc_index.tsv`：6 列 `module|topictype|title|page_path|timestamp|sample_src`；`sample_src` 为该页关联的官方示例源码（.edu 工作区）路径，可空；解析算法见 `references/desc-cache-schema.md`。
- 两套缓存头部 3 行均以 `#` 开头，含 `source`/`mtime`/`built`/`rows` 等元数据。

### 新鲜度检查（重建触发，两套缓存）
- 接口文档缓存：比较头部 `# size=... mtime=...` 与实时 `visidx.txt.htm` 的 size/mtime → 不符用 `scripts/build_cache.py` 重建。
- 开发说明文档缓存：比较头部 `# english_mtime=...` 与实时 `English\` 根目录 mtime → 不符用 `scripts/build_desc_cache.py` 重建。
- 也可手动强制：`python scripts/build_cache.py --rebuild` / `python scripts/build_desc_cache.py`。

### 铁律：缓存仅定位器
- 两套缓存都只存定位信息（接口：`symbol -> page_path + anchor`；说明：`标题/类型 -> page_path`），**绝不缓存签名/正文**。
- 签名/正文永远在运行时从对应 .htm 页读取，保证与文档一致、无过期。

## 实现方法论：目标型请求先枚举候选（仅当请求是"行为/目标"而非"已知符号"）

> 触发条件：用户给出的是"做什么"（如"提取最后一个 '-' 之后的子串"），而非具体 API 符号。
> 已知符号的查询（如"CreatePad 的签名"）跳过本节，直接走「运行时查询流程」，避免拖慢每次查找。

### ① 判定请求类型
- 请求含具体符号名 → 跳过本节，走「运行时查询流程」。
- 请求是行为/目标（字符串、集合、转换、日期等常见域）→ 进入枚举。

### ② 候选枚举（有界、机械执行）
- 优先按类限定名前缀 Grep 缓存（如 `CATUnicodeString::`），一次取得该类全部方法面。
- 类名未知时，才用 ≤3 个语义关键词 Grep symbol+description 列，取前 ~10 条即停。
- 枚举只用于"生成候选"。description 列截断 100 字符且可能误导（如 SearchSubString 描述写
  "first occurence" 却支持反向搜索），**绝不据 description 直接选型**。

### ③ 候选比对（必须读页）
- 对每个候选，按「运行时查询流程」读其文档页签名，比对语义是否匹配目标。
- 仅在"已读页、已核实签名"的候选中择优；未读页的候选一律不得选用。
- 优先"已读页核实的 CAA 接口"，而非盲目偏好 CAA 接口。

### ④ 选型 / 回退 / 阻塞（三种结局，见「文档引用表」三态）
- 有已核实匹配的 CAA 接口 → 使用之（表实现方式 = CAA接口）。
- 无对应接口但通用实现可行 → 手写，须附「枚举记录」（表实现方式 = 通用实现）。
- 无对应接口且通用实现也不可行 → **阻塞**：停止、不产出代码、报缺口并等用户决策。

### ⑤ 铁律不变
- 本节偏好仅限"已读页核实"的候选；绝不因偏好而臆造接口或跳过文档回溯。
- 缓存/visidx 未收录 ≠ CAA 不存在 → 一律表述为"未在缓存中找到"，不得断言"不存在"。
- 无法实现时同样禁止臆造或半成品交付；阻塞并等用户确认后继续。

## 运行时查询流程

> 默认只用 Grep + Read 两个工具。目标：给定 API 符号 → 定位文档页 → 读取签名。
> 需要**一次性列出整类方法/签名、或符号歧义较重**时，可先用 `scripts/signature_lookup.py` 快速批量抽取（见下 ④），再按需 Read 页面核实关键签名。

### ① Grep 缓存定位
- 在 `cache/caa_index.tsv` 中按 symbol 精确匹配（如 `CreatePad`），得到 1+ 行。
- 每行含 type / qualified_name / page_path / anchor / description，据此定位页面。

### ② 定位页面行号并读取窗口
- anchor 非空：Grep 目标页 `id="<anchor>"` 得到行号 → Read 该行附近约 100 行窗口。
- anchor 为空（define/enum/typedef/data/class/struct）：直接 Read 页面 `dt class='dt1'` 区块。
- 页面可能超过 Read 单次上限（2000 行），必须先 Grep 锚点定位行号，再按窗口读取。

### ③ 抽取签名
- 从窗口内按「从页面抽取方法签名」章节的规则抽取类型、方法名、签名、返回、参数、描述。

### ④（可选）签名批量查询脚本 `scripts/signature_lookup.py`
- 作用：一次性列出某类全部方法签名，或查单个方法；用 bs4 稳健解析，无需手写临时脚本。
- 用法：`python scripts\signature_lookup.py CATAdpOpener`（列全部） / `... CATAdpOpener CompleteAndOpen`（单方法）。
- 定位用 `cache/caa_index.tsv`，解析用 BeautifulSoup；**不缓存签名**，页面仍是唯一真源。
- 适用：目标型请求需要枚举整类候选方法时（配合「实现方法论」② 候选枚举）；多行命中需快速确认限定名时。
- 仍属"运行时快速预览"，最终采用的 API 签名必须回到 Read 页面核实（铁律不变）。

### 多行命中消歧
- 同一 symbol 命中多行（如 `GetValue` 77 行）时，用 qualified_name 消歧。
- 选与用户需求匹配的限定名（类名 + 方法名），再按该行 page_path/anchor 定位。

## 从页面抽取方法签名

页面为 Windows-1252 编码的 HTML，签名块结构（真实页实测）：

```
<dt class='dtmethod' >
  <a id="CreatePad"></a><a id="CreatePad(CATBaseUnknown_var&amp;)"></a>
<span class = 'method'>public virtual <script ...> activateLink('CATIMmiMechanicalFeature','CATIMmiMechanicalFeature_var')</script>  <span class='objname'>CreatePad</span><span class = 'signature'>( const <script ...> activateLink('CATBaseUnknown_var','CATBaseUnknown_var&amp;')</script> <span class='param'>ihSketch_input</span><span class='param'>= NULL_var</span>)</span>...</span></dt>
  <dd class='ddmethod'> Creates a new pad. <dl class='desclist'> ... <dt class='return'> ... </dl></dd>
```

### 抽取规则
- 类型：`activateLink('Type',...)` 第一参（如 `CATIMmiMechanicalFeature`）。
- 方法名：`<span class='objname'>` 内文本。
- 签名：`<span class='signature'>` 内文本（含参数列表）。
- 返回类型：`<dt class='return'>` 后的 `<dd>` 文本。
- 参数：`<span class='param'>` 内文本（参数名、默认值）。
- 描述：`<dd class='ddmethod'>` 文本；参数/返回详情在 `<dl class='desclist'>` 内。

### HTML 实体解码
- 抽取后解码：`&amp;`→`&`、`&lt;`→`<`、`&gt;`→`>`、`&quot;`→`"`、`&nbsp;`→空格 等。

### 属性间距容错
- `class='x'` 与 `class = 'x'` 都匹配（真实页两种写法并存，如 `class='objname'` 与 `class = 'signature'`）。

## 生成代码联动查开发说明文档（补充上下文）

> 生成/校验 CAA 代码时，除查「接口文档」（签名）外，还应检索「CAA 开发说明文档」获取工程上下文。

### 触发
- 目标型/实现型请求，且已确定涉及的关键类名或模块名（如 `CATIPdgUsePrtFactory`、`CAAPriPartDesign`）。

### 检索
- 用关键类名/模块名 Grep `cache/caa_desc_index.tsv` 的 `module` / `title` 列，命中的 `Ta`（技术文章）/ `Uc`（用例）页作为补充上下文。
- 按 `page_path`（相对 doc 根 `win_b64.doc`）定位 `.htm` 页并 Read 正文。

### 读官方示例源码（sample_src 命中时）
- 若命中行第 6 列 `sample_src` 非空，表明该页有发布在 `CAADoc\<FW>.edu\` 下的**官方真实源码**。
- 拼绝对路径：`.edu` 根 = doc 根（`win_b64.doc`）的上一级 `CAADoc\`，即 `CAADoc\` + `sample_src`；Read 该 `.cpp/.h`。
- 官方源码是实现**最高优先级参考**（胜过文字说明与接口摘要），据其核对类名、方法名、DLC 生命周期、工厂/会话获取套路。
- 例：`sample_src=CAAApplicationFrame.edu/CAAAfrProgressTask.m/src/CAAAfrProgressTaskSampleCmd.cpp`
  → `D:\DS\B424\CAADoc\CAAApplicationFrame.edu\CAAAfrProgressTask.m\src\CAAAfrProgressTaskSampleCmd.cpp`

### 用途
- 优先看 `Uc`（含代码实例、DLC 生命周期、工厂/容器获取套路）与 `Ta`（原理/最佳实践）。
- 接口文档未写明的"如何获取实例 / 生命周期 / 错误码"常在用例页可查（如 `CATIPdgUsePrtFactory` 的 To-get）。
- 用例页 `sample_src` 命中的官方 `.edu` 源码，直接作为可参考实现。

### 边界
- 说明文档仅作**补充参考**，不替代接口文档；要用的 API 依旧必须在「接口文档」回溯签名（铁律不变）。
- 命中多个说明页时取 1-3 个最相关，避免上下文膨胀；无命中则跳过，不阻塞。

## 硬性停用规则

### 0 命中处理链（核心）
1. Grep 缓存 symbol 0 命中。
2. 先验缓存新鲜度：比较缓存头部 `mtime` 与实时 visidx 的 mtime；不一致 → 重建缓存后再查一次。
3. 重建后仍 0 命中 = 该符号无文档。
4. **不使用**该 API。向用户报告缺口：符号名、查询过程、可能原因。
5. 建议等价文档化替代：用另一 symbol 查询试探（如 `CreatePad` 无文档则试 `CreatePocket`、`AddPad` 等），给出候选。
6. **等待用户确认后才继续**。未经确认不得用替代 API 或臆造用法。

### 绝对禁止
- 绝不用记忆臆造 API 用法。
- 绝不用无文档 API。
- 绝不用缓存中的 description 冒充签名（缓存仅定位器）。
- 任何 API 用法必须能回溯到文档页面签名。

## 文档引用表格式

每次任务结束必须输出 markdown 表格，列出本次用到的全部 API，并标记每个 API 的实现方式。

### 表头

| API 符号 | 完整限定名 | 页面路径 | 锚点 | 状态(文档化/未收录/无法实现) | 实现方式(CAA接口/通用实现/-) |
|----------|-----------|----------|------|---------------------------|---------------------------|
| CreatePad | CATIPdgUsePrtFactory::CreatePad | CATPartUseItf/interface_CATIPdgUsePrtFactory_31082.htm | CreatePad | 文档化 | CAA接口 |
| GetValue | 按实际命中行填写 | 按实际 page_path 填写 | 按实际 anchor 填写 | 文档化 | CAA接口 |
| SomeAPI | - | - | - | 未收录 | 通用实现 |

- 页面路径为相对 refman 目录的路径（与缓存 page_path 一致）。
- 锚点为空时填 `-`。
- **状态** 表示该符号的文档收录情况：`文档化` / `未收录` / `无法实现`。
- **实现方式** 表示交付的实现路径：`CAA接口` / `通用实现`；无法实现时填 `-`。
- 未收录条目必须说明缺口与建议替代（见硬性停用规则）。

### 三态：状态 × 实现方式

| 结局 | 状态 | 实现方式 | 附带输出 |
|------|------|----------|----------|
| 命中已核实 CAA 接口 | `文档化` | `CAA接口` | 无（行内可回溯 page/anchor） |
| 无接口但通用实现可行 | `未收录` | `通用实现` | 「枚举记录」块 |
| 无接口且通用实现不可行 | `无法实现` | `-` | 「无法实现（阻塞）」报告块 |

### 形态二：通用实现回退（附枚举记录，审计用）

| API 符号 | 完整限定名 | 页面路径 | 锚点 | 状态 | 实现方式 |
|----------|-----------|----------|------|------|----------|
| - | - | - | - | 未收录 | 通用实现 |

```
枚举记录:
- Grep 缓存：类前缀 `CATUnicodeString::` → 候选 SubString/SearchSubString/Reverse...，
  逐个读页核实均不匹配目标 → 未采用。
- 结论：未在缓存/visidx 中找到对应 CAA 接口，采用通用实现。
```

### 形态三：无法实现（阻塞，等用户决策）

| API 符号 | 完整限定名 | 页面路径 | 锚点 | 状态 | 实现方式 |
|----------|-----------|----------|------|------|----------|
| - | - | - | - | 无法实现 | - |

```
无法实现说明（阻塞，未产出代码）:
- 枚举记录：已枚举并读页核实的候选及不匹配原因。
- 通用实现不可行原因：具体说明为何无法安全手写（如依赖运行时未知数据、需未文档化能力）。
- 结论：当前信息下无法实现该目标，已停止。
- 待用户决策：更换/简化目标？补充缺失输入？或接受近似实现？
```

## 错误处理

### config.txt 缺失 / 格式错误 / 路径不可用 → 由 Agent 获取路径并生成
- 场景：config.txt 不存在；或首行不以 `visidx_path=` 开头；或所指向的 `visidx.txt.htm` 不存在/无法定位。
- 处理（闭环）：
  1. **明确提醒用户提供 `visidx.txt.htm` 的绝对路径**（说明这是本 skill 查询本地 3DE CAA 文档的先决条件）。
  2. 取得有效绝对路径后，**由 Agent 生成/重写 config.txt**：首行写 `visidx_path=<绝对路径>`。
  3. 生成立即校验字形：首行以 `visidx_path=` 开头、文件存在；仍不可用则回到 1 再次索取。
- 示例：
  ```
  visidx_path=D:\DS\B424\CAADoc\win_b64.doc\generated\refman\visidx.txt.htm
  ```
- 生成/修复成功后，按「前置」章节继续（必要时先做缓存新鲜度检查/重建）。

### 页面缺失（路径校验警告）
- 缓存行 page_path 对应文件不存在 → 报告该页面缺失，标记该 API 为未收录，不臆造签名。

### 缓存与源不一致
- 头部 size/mtime 与实时 visidx 不符 → 按「前置」章节重建缓存后重查。

### 目标无法实现（阻塞）
- 场景：目标既无已核实匹配的 CAA 接口，通用实现也不可行（如依赖运行时未知数据、需未文档化能力且无法安全手写）。
- 处理：**停止，不产出代码，不臆造/半成品交付**；按「文档引用表」形态三输出「无法实现」报告块。
- 等用户决策后才继续（更换/简化目标、补充输入、或接受近似实现）。
- 表述措辞统一为"未在缓存/visidx 中找到对应 CAA 接口"，不得断言"CAA 中不存在"。
