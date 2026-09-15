---
name: ym-skill-generator
slug: ym-skill-generator
display_name: 技能生成器 · 生成/体检/打包
display_name_en: Skill Generator & Packager
displayName: 技能生成器
description: "按需求生成一份合规、可用的技能骨架并填充内容，或把对话里刚实现的功能体检、脱敏后 打包成可分发、可上架的技能包。当用户说「做个技能」「生成技能」「新建技能」「写个 skill」 「把这个功能打包成技能」「打包技能」「技能打包」「存成技能」「沉淀为技能」「做成技能」 「导出技能」「技能上架」「上传开放平台」「上架开放平台」「发布技能」「技能体检」 「技能合规检查」「package skill」「create skill」「build skill」时使用。 生成时按技能基础结构建骨架（SKILL.md / references / scripts / templates）并补齐市场分发字段； 打包前自动体检 9 类风险：结构、绝对路径、明文凭据、危险操作、外部依赖、数据模板、 文件引用、体积垃圾、市场分发字段，输出 P0/P1/P2 分级报告。 技能上传包就是技能目录本身（{skill-name}/SKILL.md + references/ + scripts/ + templates/）， 平台不需要额外的清单文件。 也负责生成技能发布用的图标（512×512、PNG/JPG、≤500KB）：用 make_icon.py 出图并处理， 图标**不进技能包**，上架时在平台单独提交。当用户说「生成技能图标」「技能图标」「做个图标」 「图标合规检查」「图标超 500KB」时也使用。"
description_zh: 按需求生成技能骨架，体检脱敏后打包成可上架分发的技能包
description_en: Generate, audit, sanitize and package agent skills into publishable bundles
summary: 按需求生成技能骨架，或把已有实现体检、脱敏后打包成可分发、可上架的技能包
category: dev-programming
version: 2.2.2
author: 刘玉明
tags: [技能生成, 技能打包, 技能体检, 技能上架, skill]
trigger:
  - 生成技能
  - 做个技能
  - 新建技能
  - 写个技能
  - 打包技能
  - 技能打包
  - 做成技能
  - 存成技能
  - 沉淀为技能
  - 导出技能
  - 技能上架
  - 上传开放平台
  - 上架开放平台
  - 发布技能
  - 插件包
  - 技能体检
  - 技能合规检查
  - 把功能打包
  - skill 打包
  - 生成技能包
  - 生成技能图标
  - 技能图标
  - 图标合规检查
  - package skill
  - create skill
  - build skill
agent_created: true
---

# 技能生成器 · 生成 / 体检 / 打包 (ym-skill-generator)

**两个入口，一条流水线**：从需求生成技能，或把已有功能打包成技能——终点都是「一份解压即用、换台机器也能跑、可以上架」的技能包。

> **交付标准（先看这条）**
> 终点是**本机 `~/.workbuddy/skills/<name>/` 里有一份能立刻用的技能**，zip 只是给别人的分发副本。
> - `new_skill.py` 默认就生成到技能目录 → **生成即安装**
> - `pack_skill.py` 默认带安装 → **打包即安装**，要纯 zip 才加 `--no-install`
> - 骨架落在工作区、或只出了个 zip，都**不算交付完成**——必须确认技能目录里有它。
> - **要上架开放平台，直接传默认产出的技能目录包**（顶层 `<技能名>/SKILL.md`）——官网要求就是技能目录本身，**不需要**任何清单文件。加 `--platform` 打的插件形态包**不能**用于技能上架（平台会报「压缩包缺少 SKILL.md 文件」）。
> - 上架前用 `make_icon.py` 出一张图标（512×512、PNG/JPG、≤500KB）—— 图标由平台**单独收**，**不在 zip 里**。

核心不在写 zip，而在**生成时的规范**与**打包前的体检**。一个藏着硬编码路径和明文密码的技能，发给别人只会变成事故。

```
① 生成  需求 → 定形态 → 建骨架 → 填内容  ┐
② 打包  已有实现 → 清点产物 → 脱敏      ┴→ 体检 → 打包 → 安装（默认）
```

## 何时使用

| 起点 | 走法 |
|---|---|
| 用户只有一句需求（"我想有个能自动 XX 的技能"） | 阶段一生成 → 阶段二交付 |
| 本次对话已跑通一套脚本/流程，要固化 | 直接进阶段二，先做「清点产物」 |
| 只想给已有技能目录做安全自检 | 跳到 Step 6 体检 |
| 要把技能发给同事 / 上架市场 | 按下面全流程走，打包默认就按市场规范校验 |

---

# 阶段一 · 生成

## Step 1 · 需求澄清

**原则：能从上下文推断的绝不问，一次最多问 3 个。** 只有真正影响设计的才问：

| 问题 | 为什么必须问清 |
|---|---|
| 你希望用户说什么话时触发它？ | 直接决定 `description` 与 `trigger`，写错等于技能不会被调用 |
| 输入从哪来、输出去哪？ | 决定要不要 `scripts/`、要不要外部依赖 |
| 只你自己用，还是要给同事/上架？ | 决定 `--mode market` 还是 `local`，以及脱敏严格程度 |

能推断就按默认走，并在回复里注明假设（"我按 XX 理解，不对的话说一声"）。

## Step 2 · 定形态

技能不是只有一种形状，别给什么都套脚本：

| 形态 | 判据 | 落点 |
|---|---|---|
| **纯指导型** | 以判断、话术、流程约定为主，无固定命令 | 只有 `SKILL.md`（长内容拆 `references/`） |
| **脚本型** | 有固定命令序列、数据处理、外部调用 | `scripts/` + `SKILL.md` 写清调用方式与参数 |
| **模板型** | 反复产出同一形态的东西（报告、卡片、页面） | `templates/` 里放可复制骨架 |
| **混合型**（大多数） | 上面两种以上并存 | 目录齐全，正文分节说明 |

两条判据：

- **能用一句祈使句说清的流程，不要写脚本** —— 脚本是维护负担。
- **需要精确复现的必须写脚本** —— 靠模型每次重写，第三次就会飘。

## Step 3 · 建骨架

一条命令生成，字段齐全、结构合规：

```bash
python scripts/new_skill.py <skill-name> --out <父目录> \
    --display-name <中文名> --desc "<做什么。当用户提到「触发词」时使用。>" \
    --desc-zh "<30 字中文介绍>" --desc-en "<One-line English intro>" \
    --triggers 触发词甲,触发词乙 \
    --dirs scripts,references,config --deps requests --config-fields username,password
```

| 参数 | 作用 | 默认 |
|---|---|---|
| `--out` | 技能父目录。**默认就是技能目录，所以生成即安装** | `~/.workbuddy/skills` |
| `--prefix` / `--no-prefix` | 技能名前缀。**默认一律加 `ym-`**（个人命名约定，ym = 玉明）；只想给个裸名时加 `--no-prefix` | `ym-` |
| `--install` | 把 `--out` 指到工作区/临时目录时，额外复制一份到技能目录让它可用 | 关 |
| `--desc` / `--desc-zh` / `--desc-en` | 描述三件套 | 生成占位文本，待你替换 |
| `--triggers` | 结构化触发词（逗号分隔） | 空 |
| `--dirs` | 要建的子目录 | `scripts,references` |
| `--deps` | 第三方包；**给了就自动生成 `scripts/setup.py`** | 空 |
| `--config-fields` | 生成 `config/settings.example.json` 的空字段（自动带上 `config` 目录） | 空 |
| `--mode` | `market` 全字段可上架 / `local` 只留本机必需 | `market` |
| `--force` | 覆盖同名目录（**仅限含 `SKILL.md` 的技能目录**，避免误删） | 关 |

> **命名约定：本机自建技能一律 `ym-` 开头。** 所以 `new_skill.py 待办查询` 这类写法直接传裸名即可，
> 脚本会自动补成 `ym-待办查询的英文名`；已经带前缀的不会被叠加；`--no-prefix` 用于确实要发布出去的通用技能。
> 前缀只进 `name` / `slug` / 目录名，**不进展示名** —— 展示名是给人看的，不带技术前缀。

生成结果：

```
<skill-name>/
├── SKILL.md                      # frontmatter 齐全 + 分节正文骨架
├── scripts/                      # 目录条目会写进 zip，结构不丢
│   └── setup.py                  # --deps 有值才生成
├── references/
└── config/settings.example.json  # --dirs config --config-fields a,b
```

**生成即体检 + 生成即就位**：脚本跑完会自动体检并打印 P0/P1，骨架合规才提示"可以开始填内容"；默认就落在技能目录里，省掉"写完了还要搬一次"的环节。若为了边写边改把 `--out` 指到工作区，记得加 `--install`，否则本机不会有这个技能——脚本会在结尾明确警告"本机还没有这个技能"。

技能名会被规整成 kebab-case（`待办查询` → 报错；`OA_Todo` → `oa-todo`），因为市场规范硬要求小写字母+数字+连字符，中文名放 `display_name`。

**要上架的话，起名先去平台搜一遍精确名，有重名就换**——平台允许重名（同名技能会并存），但用户搜到后无法分辨该装哪个，冷启动最吃亏。词根占位实测与命名方法见 `references/naming-and-discovery.md`。

## Step 4 · 填内容

骨架里的 `<...>` 必须**全部替换**，一点不剩。写法要求：

1. **首句说它替用户省掉了什么**，不要复述标题。
2. **执行步骤用祈使句 + 真实命令**（"运行 X 完成 Y"），不要第二人称劝说式表达。
3. **触发词写用户的原话**，别写书面语。用户会说"查待办"，不会说"获取待办事项列表"。
4. **「常见坑」必写** —— 现象 + 原因 + 规避。这是技能最值钱的部分，也是本次实现里真正学到的东西。
5. **只写真实存在的文件** —— 正文里提到的每个路径体检都会去验，写不存在的文件会被判「引用缺失」P1；运行时产物要显式标注为输出文件。
6. **正文超 15000 字符会被警告** —— 把字段表、接口约定、踩坑细节搬进 `references/`，正文只留流程骨架，并写明"什么时候需要读它"（按需加载，别默认全读）。

### frontmatter 字段速查

| 字段 | 何时必填 | 用途 | 缺失后果 |
|---|---|---|---|
| `name` | 必填 | 技能标识，kebab-case，与目录名一致 | 无法定位 |
| `description` | 必填 | 写清用途 + 触发词 | 技能不会被触发 |
| `description_zh` / `description_en` | 要分发 | 中/英文一句话介绍 | 上架失败 |
| `version` | 要分发 | 语义化版本号 | 上架失败 |
| `display_name` / `display_name_en` | 建议 | 市场列表展示名 | 展示名难看 |
| `category` | 建议 | 分类之一（如 `writing`） | 上架需补 |
| `author` | 建议 | 署名；也可写在 `metadata:` 之下 | 归属不明 |
| `allowed-tools` | 可选 | 工具白名单（逗号分隔） | — |
| `disable-model-invocation` | 可选 | `true` 则只能用户手动调用 | — |
| `user-invocable` | 可选 | `false` 则隐藏菜单，仅供 AI 内部调用 | — |
| `agent_created` | 本机 | 允许 Agent 后续用技能管理工具改它 | 无法改 |
| `trigger` | 本机 | 结构化触发词，比散文更稳 | 触发靠猜 |

「何时必填」三档：**必填** = 本机跑不起来；**要分发** = 上架被打回；**建议** = 只影响展示。

更细的设计取舍（形态决策案例、description 写法对照、何时拆 references）见 `references/generation-guide.md`。

---

# 阶段二 · 交付前

## Step 5 · 清点产物（打包已有实现时）

把本次对话真正产出、且**换台机器仍然需要**的东西归类：

| 类别 | 放哪 | 判断标准 |
|---|---|---|
| 主脚本、可执行逻辑 | `scripts/` | 每次都要重写一遍 → 必须固化 |
| 依赖安装、首次配置向导 | `scripts/setup.py` | 有第三方依赖就必须有 |
| 配置模板 | `config/*.example.json` | 字段名留全，值留空 |
| 领域知识、字段表、接口约定、踩坑记录 | `references/` | 塞进 SKILL.md 会把它撑爆 |
| 图片、字体、样板稿、品牌资源 | `assets/` | 用于产出结果，不需要读进上下文 |
| 运行产物（缓存/profile/日志/输出文件） | **不入包** | 使用者本机自己生成 |

**三问筛选法**（拿不准某个文件要不要进包时用）：

1. 换台机器 / 换个人，这个文件还有效吗？——无效就别打
2. 它是「输入」还是「输出」？——输出几乎都不该打
3. 少了它，别人能跑起来吗？——不能就必须打，且要在 SKILL.md 里写清怎么用

## Step 6 · 脱敏与参数化

最容易被跳过、后果最严重的一步。三类必须处理：

**① 绝对路径** → 换成动态定位，不要写死盘符和用户名

```python
SKILL_ROOT = Path(__file__).resolve().parent.parent        # 技能自身
USER_DIR   = Path(os.path.expanduser("~")) / ".workbuddy"  # 用户目录
```

文档里一律写 `<技能目录>`、`~/.workbuddy/xxx` 这类占位符。完整替换表见 `references/sanitize-rules.md`。

**② 凭据** → 一律外置，仓库里只留空模板

- 模板：`config/xxx.example.json`，敏感字段留空字符串
- 真值：`~/.workbuddy/<name>_config.json`（首选）或环境变量
- 交互式首次配置：`getpass` 不回显 + 二次确认，写成 `scripts/setup.py`
- **已经外泄过的凭据必须轮换**，改代码不解决问题

**③ 依赖** → 显式声明 + 一键安装

- Python：SKILL.md 列出 pip 包名，并提供 `scripts/setup.py`
- Node：提供 `package.json`
- 外部可执行文件（ffmpeg、edge-tts、浏览器…）：写清安装方式、检测方法、缺失时的行为
- 中文字体：说明回退顺序，否则用户只看到方块

## Step 7 · 体检

```bash
python scripts/audit_skill.py <技能目录>            # 人读报告
python scripts/audit_skill.py <技能目录> --json      # 机器读
python scripts/audit_skill.py <技能目录> --hide-p2   # 只看要命的
python scripts/audit_skill.py <技能目录> --market    # 追加市场分发规范检查
```

同类问题自动聚合（不会逐行刷屏），密钥一律打码回显。退出码：`0` 通过 / `1` 有 P1 / `2` 有 P0。

### 9 类检查

| # | 检查 | 命中后的动作 |
|---|---|---|
| 1 | **结构** SKILL.md、frontmatter、name/description 规范、正文体量 | 补字段、拆 references |
| 2 | **绝对路径** 盘符、UNC、POSIX 家目录；代码记 P1、文档记 P2 | 改成动态定位 |
| 3 | **凭据** 密钥关键字赋值与高置信度字面量（JWT/sk-/AKIA/私钥块） | 外置 + 轮换 |
| 4 | **危险操作** 递归删除、格式化、远程脚本直灌 shell、动态执行外部输入 | 收窄范围或加确认 |
| 5 | **外部依赖** 第三方包、外部可执行文件、中文字体、是否有一键安装入口 | SKILL.md 写「运行前提」 |
| 6 | **数据模板** config 有无 `.example`、模板里有没有真凭据 | 补空模板 |
| 7 | **引用** Markdown/反引号/命令里提到的文件是否真实存在 | 补齐或改路径 |
| 8 | **体积与垃圾** `__pycache__`/`node_modules`/日志、运行档案目录、超大文件 | 排除或确认必要性 |
| 9 | **市场分发** frontmatter 必填字段、kebab-case `name`、中英文介绍语言是否匹配、署名是否还是机器默认值 | 补字段 |

第 9 项的开关两边不同：**打包时默认检查**（打包往往是分发的第一步），要跳过加 `--no-market`；**单独体检时默认不查**，上架前才加 `--market`——否则体检任何一个老技能都会满屏告警。

定级依据是实测（本机 4 个真实市场技能）：`description_zh` / `description_en` 4/4 都有，缺了确实上架失败，记 **P1**；`display_name` 3/4、`category` 2/4、`author` 1/4（且写在 `metadata:` 之下），说明平台没把它们当硬门槛，一律记 **P2**。

逐项判定标准与放行条件见 `references/checklist.md`。

### 已知安全项放行

- 单行：行尾加 `# skill-audit: ignore`
- 整个文件：技能根目录建 `.skillignore`（每行一个 glob）

放行要留理由，别把 `ignore` 当成消音器。

## Step 8 · 打包与安装

```bash
# 打包 + 安装（默认行为：技能立刻可用，同时产出 zip）
# 这个 zip 就是**技能上传包**（顶层 <技能名>/SKILL.md），上架开放平台直接传它
python scripts/pack_skill.py <技能目录> --out <输出目录>

# 另出一份「插件形态包」（插件市场分发用；**技能上架不要用它**）
python scripts/pack_skill.py <技能目录> --as-plugin --out <输出目录>

# 平台若报「不接受顶层目录」，换成根目录即插件根的形态
python scripts/pack_skill.py <技能目录> --as-plugin --flat-root --out <输出目录>

# 只打 zip、不安装（纯粹帮别人打包时才用）
python scripts/pack_skill.py <技能目录> --out <输出目录> --no-install

# 装到自定义目录 / 覆盖同名技能
python scripts/pack_skill.py <技能目录> --install-dir <自定义目录> --force

# 只做本机自用、不在乎上架，跳过市场字段检查
python scripts/pack_skill.py <技能目录> --no-market

# 先看会打进哪些文件，不落盘
python scripts/pack_skill.py <技能目录> --dry-run

# 体检有 P1 但确认安全，强行打包
python scripts/pack_skill.py <技能目录> --allow-p1
```

### 产出形态，别搞混

| | **技能目录包（默认）** | 插件形态包（`--as-plugin`，旧名 `--platform`） |
|---|---|---|
| 产出文件 | `<技能名>.zip` | `<技能名>-plugin.zip` |
| 用途 | 装到本机、发给同事，**以及上传开放平台「技能」类目** | 插件市场分发（把技能挂到插件下、团队 marketplace） |
| 顶层结构 | `<技能名>/SKILL.md` + `references/` + `scripts/` + `templates/` | `<插件名>/.codebuddy-plugin/plugin.json` + `<插件名>/skills/<技能名>/...` |
| 关键差异 | **无清单文件** —— 技能上架要的就是它 | 有 `plugin.json`，**不能**用于技能上架 |

> **技能上传包 = 技能目录包 = 默认产出。** 官方要求的技能基础结构就是：
>
> ```
> {skill-name}/
> ├── SKILL.md              # ★ 必须。frontmatter 要齐 description / description_zh /
> │                          #   description_en / version / author
> ├── references/           # 可选，SKILL.md 里用 @references/xxx.md 引用
> ├── scripts/              # 可选，在 SKILL.md 里声明调用命令
> └── templates/            # 可选
> ```
>
> 平台是在**技能目录根**找 `SKILL.md`。打成 `skills/<技能名>/SKILL.md` 那种插件形态再传，
> 会直接报「**压缩包缺少 SKILL.md 文件**」—— 那正是 `--as-plugin` 产物的样子，别拿去传。

### 插件形态包（`--as-plugin`）的结构

只有要把技能**作为插件**分发时才用：

```
<plugin-name>/
├── .codebuddy-plugin/
│   └── plugin.json          # 插件清单
└── skills/                  # 技能默认位置，AI 从这里发现技能
    └── <skill-name>/
        ├── SKILL.md
        ├── scripts/
        ├── references/
        └── templates/
```

平台若报结构类错误、提示不接受顶层目录，加 `--flat-root` 换成「zip 根即插件根」的形态。

`plugin.json` 由脚本从 SKILL.md 的 frontmatter 自动生成：

| 字段 | 来源 | 说明 |
|---|---|---|
| `name` | 技能目录名 | kebab-case |
| `version` | `version` | 缺省 `1.0.0` |
| `description` | `description`（剥掉触发词长尾后截 200 字） | 平台中文搜索匹配的就是它，功能描述要留全 |
| `description_en` | `description_en` | |
| `author.name` | `author` | 字符串自动转成 `{"name": ...}` |
| `category` | `category` | 展示分类（字符串，如 `development`） |
| `keywords` | 名字拆词 + 前 6 个 trigger | 平台检索用 |

> ⚠️ 这是**插件**清单的字段集，与**专家**清单（`expertType` / `displayName` / `categoryId`…）
> 不是一套，别互抄。专家的分类是 `categoryId`（数字枚举），技能是 `category`（字符串）。
| `skills` | **不写** | 技能放 `skills/` 就是平台默认位置，靠默认发现。显式写 `["./skills"]` 有语义风险——平台可能理解为"技能目录列表"，于是去找该目录下层的 SKILL.md，结果一个技能都加载不到 |

> **平铺也合法**：真实市场里 `skill-creator`、`agent-browser` 都是「插件根下直接放 SKILL.md」。但那种写法要靠 `skills` 字段指向插件根，多一处可能出错的地方；统一放进 `skills/`（平台默认位置）最稳，所以脚本默认这么打。

打包开始时先逐项列出市场分发字段的齐备情况（`已填` / `缺失` / `未填（建议补）`），再决定放行还是打回——「按规范打包」这件事是可见的，不用猜。

产出结构固定为规范形态，解压即用（空目录也会写进 zip，刚生成的骨架结构不会丢）：

```
<技能名>.zip
└── <技能名>/
    ├── SKILL.md
    ├── references/     # 有则入包
    ├── scripts/        # 有则入包
    └── templates/      # 有则入包
```

打包时**只装技能本身**，三类东西自动排除：

| 类别 | 例子 |
|---|---|
| 构建缓存 | `__pycache__`、`node_modules`、`.venv`、`*.pyc`、`*.log` |
| **仓库元数据** | `.git/`、`.gitignore`、`.gitattributes`、`README.md`、`CHANGELOG.md`、`LICENSE` |
| **发布图标** | `icons/`（防御性排除；图标本就不该放进技能目录，见 Step 9） |

> 技能目录常常同时是个 git 仓库，最容易踩的坑就是把 `.gitignore` / `README.md` 一起打进包。
> 排除项会在打包时逐条打印出来，不会悄悄丢。

zip 顶层目录 = 技能名。

**P0 永远阻断**，`--allow-p1` 也不能越过。覆盖已有同名技能需要 `--force`，脚本会先打印将被覆盖的路径。

两种情况下会**自动跳过安装**（都不报错，但会在结尾用 `!!` 提醒你技能还不可用）：

| 情况 | 原因 |
|---|---|
| 源目录已在技能安装目录内 | 它就是已装技能，无需复制。这也是**防止"原地打包时把自己删掉"**的保险 |
| 加了 `--no-install` | 你明确只要 zip |

安装后技能即刻就绪；若未出现在可用列表，重启会话即可被识别。

### 发布 ≠ 上线：线上字段分三档更新

上架动作是 `skillhub publish <技能目录> --changelog "..."`（网页发布等价）。**命令返回成功只代表「平台已受理」**，
商店里的字段不是一次性全变，实测分三档：

| 档位 | 字段 | 时机 |
|---|---|---|
| **立即** | `tags` | 秒级生效（skill 级索引，不等审核） |
| **随审核** | `version` / `summary` / `description` / 版本列表 / 下载包 | 只有**审核通过**的版本才写进 `latestVersion` |
| **CLI 改不了** | `category` | **发布 payload 里没有这个字段**，只能去网页 dashboard 改 |

判定「是否真的上线」看**版本列表里有没有刚发的版本号**，不要看命令返回值。`stats.versions` 会把待审版本也计入，
所以「计数涨了但列表没动」= 正在审核。

只读查询接口（无需登录，用来复核发布结果）：

```bash
GET https://api.skillhub.cn/api/v1/skills/<slug>                               # latestVersion / summary / category / stats.versions
GET https://api.skillhub.cn/api/v1/skills/<slug>/versions?page=1&pageSize=20   # 各版本 + 过审后的安全报告
GET https://api.skillhub.cn/api/v1/download?slug=<slug>                        # 看 Content-Disposition 的 filename 是哪个版本
```

反面写法别踩：`/api/v1/skills/@handle/slug` 返回 **405**，`/api/v1/skills/resolve?slug=` 返回 **400**。

## Step 9 · 技能图标（发布时单独用，**必须放在技能目录之外**）

平台创建技能时要求一张图标：**512×512、PNG 或 JPG、≤500KB**。
它是**表单里的一个字段**，不是 zip 里的文件 —— 所以图标单独生成、单独上传。

⚠️ **图标不要放进技能目录。** 技能目录常常同时是 GitHub 仓库，而 SkillHub 绑定仓库
发布时会按「文件类型白名单」逐个校验仓库文件，一张裸 `.png` 会让整次发布被拒
（报「不支持的文件类型: icons/xxx-icon.png」）。默认输出目录在技能目录之外：
`~/.workbuddy/skill-icons/`，可用 `--out` 或环境变量 `SKILLHUB_ICON_DIR` 改。

```bash
# ① 拿提示词（按本技能 SKILL.md 的展示名与描述拼，保证贴合技能身份）
python scripts/make_icon.py --prompt

# ② 交给 ImageGen（size 用 1024x1024），生成图回来做后处理
#    默认：居中裁切 + 清右下角生成标 + 缩到 512×512 + 压到 ≤500KB
python scripts/make_icon.py <生成图>            # 落盘到 ~/.workbuddy/skill-icons/
python scripts/make_icon.py <生成图> --out <目录> --name <技能名>

# ③ 上传前自查
python scripts/make_icon.py --check ~/.workbuddy/skill-icons/*.png
```

| 开关 | 作用 |
|---|---|
| `--prompt [--lang en]` | 只打印给 ImageGen 的提示词，不处理图片 |
| `--check` | 只做合规检查：512×512、PNG/JPG、≤500KB |
| `--no-center` / `--no-clean` | 关掉居中裁切 / 关掉清生成标（默认都开） |
| `--fmt png\|jpg` | 优先输出格式（默认 png；压不进 500KB 时自动降质或换格式） |
| `--out` / `--name` | 输出目录 / 文件名前缀（默认目录可用环境变量 `SKILLHUB_ICON_DIR` 固定） |

**两个真实踩过的坑，脚本默认已经处理：**

- ImageGen 出的是「1024×1024 圆角方块 + 外圈留白」，主体常偏一侧 —— 整图缩放会留白不对称，
  随手按固定框硬切（如 `(0,0,900,900)`）会**一边内容被截、另一边留白**。`--center` 按圆角方块真实边界取正方形。
- 生成图右下角带「AI生成 / WORKBUDDY」标，落在圆角方块**外侧**背景上。`--clean` 用周围背景重建，不碰主体。

图标 prompt 的要点：**单一主体、居中、不要文字**，用 1~2 个具象物件表达「这个技能做什么」，
缩小到 64px 仍能认出来。（画人物头像是**专家**头像的事，技能图标不讲人。）

**依赖**：生成 / 体检 / 打包全流程只用标准库；`make_icon.py` 需要 Pillow，属可选依赖 ——
`python scripts/setup.py --install-pillow`。不装也能跑到出 zip，只是图标得自己压。

## 技能目录结构约定

```
<skill-name>/
├── SKILL.md              # ★ 必需：frontmatter + 正文
├── references/           # 参考资料（按需加载的 API 规范、示例数据、领域知识）
├── scripts/              # 可执行脚本（数据获取、处理、批量操作）
├── templates/            # 模板文件（报告模板、工作流模板、可复制骨架；一律 .md 后缀）
├── assets/               # 产出用资源（图、字体、样板稿）
└── config/               # 只放 *.example.* 空模板
```

`SKILL.md` 通过反引号路径引用子资源（`` `references/xxx.md` ``）——本机 28 个技能全部采用这种写法，比 `@` 前缀更稳。

## 常见坑

1. **`name` 与目录名不一致** —— 本机自用能吃中文 `name`，但市场校验要求 kebab-case。要分发就统一成小写连字符，中文触发词写进 `description`。
2. **`description` 含尖括号** —— 市场校验直接失败。标题里的 `<技能目录>` 之类占位符不要写进 frontmatter，换成方括号或中文书名号。
3. **骨架文件顶部放 HTML 注释** —— SKILL.md 首行必须是 `---`，前面加注释块会被判 P0「缺 frontmatter」。模板的说明注释一律放文件末尾。
4. **把运行档案打进包** —— 浏览器 `profile/`、`output/`、`*.log`、`*.db` 属于使用者本机数据，打进去既臃肿又可能带隐私。
5. **配置模板里留了真值** —— 最典型的泄漏方式。模板字段留全、值留空。
6. **依赖只写在正文里没脚本** —— 使用者必漏装。有第三方包就给 `scripts/setup.py`（`new_skill.py --deps` 会自动生成）。
7. **SKILL.md 提到文件但没说是输入还是产物** —— 会被判成引用缺失。是运行时产物，就在**同一行**写上「运行时产物 / 输出文件 / 生成」这类字样，体检器会改判 P2（`OUTPUT_HINT_RE`）。
8. **技能目录里写死自己的绝对路径** —— 别人解压到别处就跑不起来。一律用 `Path(__file__)`。
9. **`description` 写成 YAML 折叠块（`>-`）** —— 体检器认，**平台不认**。WorkBuddy 侧折叠块没问题（`fm_text()` 会把缩进块拼起来判），但平台自带的解析器只读「`key: 值`」那一行：`>-` 会被读成**字面量两个字符**，描述在商店里就没了。多行列表（`tags:` 下面写 `- 项`）同理被读空。**要发布就写单行**——长文本折成一行，对完整 YAML 解析器同样合法。`new_skill.py` 已按单行生成。
10. **上一版技能的技能名/目录名不一致导致重复加载** —— 改名时目录名、`name` 字段、zip 名三者要一起改。
11. **只交一个 zip 就收工** —— 用户要的是"有个技能能用"，不是一堆文件。把骨架建在工作区却忘了 `--install`，本机就一直没有这个技能，用户当然会问"怎么只生成了个 zip"。
12. **原地打包已装技能时把自己删掉** —— 安装目标恰好等于源目录，先 `rmtree` 再复制等于自毁。`install_skill` 现在有 `is_inside` 前置判断，改这块逻辑时**不要拿掉**。
13. **把技能打成插件形态去上传** —— 传 `skills/<技能名>/SKILL.md` 那种包，平台在**技能目录根**找不到 SKILL.md，直接报「**压缩包缺少 SKILL.md 文件**」。技能上传包 = 默认产出的技能目录包（顶层 `<技能名>/SKILL.md`）。至于「压缩包缺少 .codebuddy-plugin/plugin.json」—— 那是**专家/插件**那条线的报错，别套到技能上。
14. **把 SKILL.md 的 description 直接搬进 plugin.json**（用 `--as-plugin` 时）—— 里面塞着十几个触发词，是给模型判断触发用的，搬到市场展示位又长又难读。脚本会剥掉「当用户说…」「也适用于…」这类长尾再截断。
15. **把技能图标放进技能目录** —— 图标是平台创建技能时**单独收的一个字段**。放进技能目录，**打包时**会被排除（无害），但**绑定 GitHub 仓库发布时会被平台直接拒收**（报「不支持的文件类型」）。一律输出到技能目录之外（默认 `~/.workbuddy/skill-icons/`）。
16. **把 `.git` / `README.md` 一类打进包** —— 技能目录常常同时是 git 仓库，`.gitignore`、`.gitattributes`、`README.md` 会被顺手打进去，既没用又可能漏出仓库信息。打包器已按「构建缓存 / 仓库元数据 / 发布图标」三类排除，**别改回去**。注意：**打包器排除了 ≠ 发布时排除了** —— SkillHub 扫描 GitHub 仓库时走的是它自己那套白名单，见坑 18。
17. **拿专家头像的规矩做技能图标** —— 技能图标不画人物、不要复杂场景：单一主体、居中、不要文字，缩到 64px 仍能认出来。画人物头像是**专家**的 `avatars/` 那件事。
18. **技能目录里留下非白名单文件，整次发布被拒** —— SkillHub 绑定 GitHub 仓库发布时，会按「文件类型白名单」逐个校验仓库里的文件，命中一个就整单拒收（报「**不支持的文件类型: xxx**」）。除图标外，最容易被忽略的两类：
    - **git 仓库自带的元数据**：`.gitignore`、`.gitattributes` —— 点开头的隐藏文件一律不在白名单里
    - **非白名单扩展名**：`.template`、`.zip`、`.xlsx` 等

    三条对策：
    - `.gitignore` / `.gitattributes` → 迁到 `.git/info/exclude` 与 `.git/info/attributes`（git 官方支持的位置，行为完全一致，但不在工作区、不进仓库，因而扫不到）
    - 模板类文件用 `.md` 后缀，别用 `.template`
    - 图标输出到技能目录之外

    自查一句话：`git ls-files` 列出的每个文件，都该是 `.md` / `.py` / `.json` / `.txt` / `.sh` / `.yaml` 这类纯文本。

19. **中文展示名写成下划线 `display_name`（平台不认）** —— **平台只认驼峰 `displayName`**；`display_name` 是 WorkBuddy 本机的字段，平台完全不读。缺了不报错、发布照样成功，但**商店列表里直接显示英文 slug**（例如 `ym-skill-generator`）—— 静默失败，最容易漏。要发布的技能**两个都写**：`display_name` 给本机，`displayName` 给平台。另外 `display_name` 里的「 · 功能列举」后缀不要带进 `displayName`，展示名越干净越好。
    同一个字段族还有两处：平台发布 CLI 要求 `slug`（与 `name` 一致），缺了硬报错 `SKILL.md 缺少 slug`；`summary` 是商店列表摘要，一般直接复用 `description_zh`。

20. **`category` 用了平台枚举外的值** —— 平台只认 13 个 key：`office-efficiency` / `content-creation` / `dev-programming` / `data-analysis` / `design-media` / `ai-agent` / `knowledge-management` / `business-ops` / `education` / `professional` / `it-ops-security` / `life-service` / `pay-skill`。传别的值不报错，但上架后显示「**未分类**」。注意 `development` **不是**平台值，对应的是 `dev-programming` —— `new_skill.py` 的 `--category` 默认值已改对，体检器也会校验（`SKILLHUB_CATEGORIES`）。

21. **frontmatter 里写行尾 `#` 注释** —— 平台的简易解析器**不剥行尾注释**，注释会被拼进字段值（`category: development  # 分类` 读出来就是整个字符串，于是「未分类」）。注释一律**单独成行**写在字段上方，发布前也记得把整行注释删干净。

22. **以为 CLI 发布能把分类一起改掉** —— SkillHub CLI 的发布 payload 固定只有 `slug` / `version` / `displayName` / `summary` / `description` / `tags` / `license` / `homepage` / `changelog` 九个键，**没有 `category`**。于是出现最迷惑的一幕：SKILL.md 里分类写对了、CLI 也返回 `✓ Published`，线上分类纹丝不动。改分类只能走**网页 dashboard**；发布后用 `GET /api/v1/skills/<slug>` 复核 `skill.category`。

23. **拿 `✓ Published` 当「已经上线」** —— 返回成功只表示平台**已受理**。实测线上字段分三档：`tags` 秒级生效；`version` / `summary` / 描述 / 版本列表 / 下载包要等**三线安全审核**（内容合规 + 科恩漏洞扫描 + 云鼎 AI 安全评估）通过才写进 `latestVersion`；`category` 走 CLI 永远不变。所以发完立刻去商店看「没变化」是正常的，别急着重发——先查版本列表确认新版本号在不在。

24. **忘了 `ym-` 前缀，或反过来给通用技能硬加前缀** —— 本机自建技能统一 `ym-` 开头，
    `new_skill.py` 已默认补上。改名时记住 **`name` / 目录名 / 仓库名三处同步**，漏一处就会
    「本地目录叫 A、商店里叫 B」，`skillhub install <slug>` 也对不上。真要发布给外人用的通用技能
    加 `--no-prefix`，别把个人前缀带进别人的安装清单。

## 版本历史

### v2.2.2 (2026-09-15)

- **`new_skill.py` 默认给技能名加 `ym-` 前缀**（个人命名约定，ym = 玉明）：新增 `--prefix` /
  `--no-prefix`；已带前缀不叠加；**展示名自动剥掉前缀**（`ym-demo-test` → `Demo Test`，
  前缀只进 `name` / `slug` / 目录名）。
- 新技能 `ym-skillhub-publisher` 同步按此约定命名。

### v2.2.1 (2026-09-15)

- **补「发布后核查」实战结论**（发布 v2.2.0 时实测）：
  - CLI 的发布 payload 固定只有 9 个键，**不含 `category`** —— SKILL.md 里分类改对了、
    CLI 也返回 `✓ Published`，线上分类依旧不变，**只能走网页 dashboard 改**。
  - 线上字段分三档更新：`tags` 秒级生效；`version` / `summary` / 描述 / 版本列表 / 下载包
    要等三线安全审核通过；`category` 走 CLI 永远不动。
- Step 8 新增「发布 ≠ 上线」小节：三档更新表 + 三个免登录只读状态接口
  （`/api/v1/skills/<slug>`、`/versions`、`/download`），并标出两个会踩的路径
  （`/skills/@handle/slug` → 405，`/skills/resolve?slug=` → 400）。
- 新增坑 22（以为 CLI 能改分类）、坑 23（拿 `✓ Published` 当已上线）。
- `references/checklist.md` 同步补「`category` 有第二条坑：CLI 传不了它」。

### v2.2.0 (2026-09-15)

- **修「商店里没有中文名」**：本技能发布后商店里显示英文 slug，根因是 frontmatter 写的是
  下划线 `display_name`，而**平台只认驼峰 `displayName`**。补 `slug` / `displayName` /
  `summary` / `tags` 四个平台字段；`display_name` 保留给 WorkBuddy 本机。
- **`category: development` → `dev-programming`**：`development` 不在平台 13 个枚举内，
  这是商店里显示「未分类」的原因。
- **description 三件套由 YAML 折叠块改为单行**：平台解析器只读「`key: 值`」那一行，
  `>-` 会被读成字面量 `>-`。`tags` 改内联列表写法（多行列表同样会被读空）。
- **`new_skill.py` 同步修**：生成的 frontmatter 直接就是平台可读形态（单行 + 平台字段），
  `--category` 默认值改为 `dev-programming`。
- **`audit_skill.py` 新增三项检查**：缺 `displayName` 判 P1；`description`/`tags` 等用块标量、
  `category` 不在枚举内各判 P2。
- `templates/skill-skeleton.md` 同步：字段注释改为独立成行（行尾注释会被平台读进值里）。

### v2.1.0 (2026-09-15)

- **改名 `skill-generator` → `ym-skill-generator`**：原 slug 已被他人占用（发布时被平台
  拒收「slug 'skill-generator' 已被其他用户占用」）。`name` / 目录名 / GitHub 仓库名
  **三处同步改**，保持一致，否则 `skillhub install <slug>` 会对不上。
- **更正 v2.0.1 的旧结论**：当时实测「`skill-generator` 精确名零重名」已失效。
  换用 `ym-` 前缀后，`ym-skill-generator` 等 9 个候选实测公开层面均无同名技能。
- `references/naming-and-discovery.md` 补「slug 占用怎么查」：判据是下载接口
  `GET /api/v1/download?slug=<slug>`（200 = 已被别人发布 / 404 = 公开层面无此技能）；
  两个反直觉点 —— **404 ≠ 可用**（被占用但未公开发布的同样 404），
  且**搜索接口不能判重**（`q=` 是模糊匹配，对短前缀直接返回兜底热门榜）。

### v2.0.9 (2026-09-15)

- **修发布阻断**：绑定 GitHub 仓库发布时，平台按「文件类型白名单」扫描仓库里的文件，
  技能目录里的 `.gitignore`、`.gitattributes`、`icons/*.png`、`templates/*.template`
  会让整次发布被拒（报「不支持的文件类型」）。本技能自身做了三处清理，并写进坑 18：
  - `.gitignore` / `.gitattributes` → 迁到 `.git/info/exclude` 与 `.git/info/attributes`
  - 图标不再输出到技能目录内：`make_icon.py` 默认改到 `~/.workbuddy/skill-icons/`
    （可用 `--out` 或环境变量 `SKILLHUB_ICON_DIR` 覆盖）
  - `templates/SKILL.md.template` → `templates/skill-skeleton.md`（`.template` 不在白名单）
- 「打包时排除图标」的规则**保留**（防御性），但职责已从「排除」改为「根本不该放进去」。
- 新增坑 18（非白名单文件导致发布被拒）。

### v2.0.8 (2026-09-15)

- **新增 `make_icon.py`：技能图标能力**（提示词构建 → 居中裁切 + 清生成标 → 512×512 / ≤500KB → 上传前自查）。
  - `--prompt` 按本技能 SKILL.md 的展示名与描述拼 ImageGen 提示词；`--check` 只做合规检查；
    直接传生成图则输出到技能根下的 `icons/`。
  - **图标不进技能包**：平台在创建技能时把图标当**表单字段**单独收，所以 `icons/` 打包时一律排除，
    上架时单独上传。人物头像那套（专家 `avatars/`）不适用于技能图标。
- **打包排除规则补齐**：新增「仓库元数据」（`.git/`、`.gitignore`、`.gitattributes`、`README.md`、
  `CHANGELOG.md`、`LICENSE`）与「发布图标」（`icons/`）两类排除；技能目录同时是 git 仓库时
  不再把仓库文件打进包。排除项会分类打印，不再与缓存混在一起。
- 新增坑 15（图标打进包）、坑 16（把 `.git` / `README.md` 打进包）、坑 17（拿专家头像规矩做技能图标）。
- 补 `scripts/setup.py`（环境自检 + `--install-pillow`）：图标能力带来唯一的可选依赖 Pillow，
  按本技能自己的约定（坑 6「有第三方包就给 setup.py」）补上一键安装入口。

### v2.0.7 (2026-09-15)

- 明确并验证技能上传包的最大目录深度为“技能根/二级目录/文件”；打包器继续输出扁平技能目录包，不生成插件嵌套结构。

### v2.0.6 (2026-09-15)

**修正一处根本性认知错误**。此前认为「技能上架也要 `.codebuddy-plugin/plugin.json`」，并据此把技能打成插件形态（`skills/<技能名>/SKILL.md`）——**这是错的**，照着传必被拒。官方要求的技能基础结构就是**技能目录本身**：

```
{skill-name}/
├── SKILL.md              # ★ 必须
├── references/  scripts/  templates/     # 均可选
```

**不需要**任何清单文件。平台是在技能目录根找 `SKILL.md`，按插件形态传会报「**压缩包缺少 SKILL.md 文件**」。

- **默认产出的技能目录包就是技能上传包**，原样传即可。
- `--platform` 更名 **`--as-plugin`**（旧名保留兼容）：它产出的是**插件形态**包，用于插件市场分发；上传开放平台技能类目**不要**用它。输出里已加显式告警。
- 纠正「压缩包缺少 .codebuddy-plugin/plugin.json」的归属：那是**专家 / 插件**那条上传线的报错，与技能上传无关。这条认知此前被错套在技能上。
- 文档补上官方要求的技能基础结构、frontmatter **必填字段表**（`description` / `description_zh` / `description_en` / `version` / `author`），以及技能 / 专家两条上传线的结构差异对照。

### v2.0.5 (2026-09-15)

- `--platform` 合成的 plugin.json 补 `category` 独立字段（技能上架的展示分类）。原先只把它塞进 `keywords`，平台展示位就没分类。⚠️ 技能用 `category`（字符串），专家用 `categoryId`（数字枚举），是两套规范，别串。
- 描述里触发词长尾的剥离分界词补全：新增「也适用于 / 亦适用于 / 触发词：/ 使用场景：」。此前只认「当用户说」，像「也适用于『生成专家包』这类说法」整句会残留进市场文案。
- 新增 `keywords` 取词口径说明：名字拆词 + 触发词前 6 个（category 已单独成字段，不再重复塞进去）。

### v2.0.4 (2026-09-15)
- 修引用检查误报：正文**同一行**标注了「运行时产物 / 输出文件 / 生成」等字样的路径，改判 P2 而非 P1「引用缺失」——此前写 `avatars/expert.png` 这类**运行后才生成**的文件会被判成断链，逼得用户改写文档才躲得过去
- 危险操作（P1）的提示文案补上放行办法：作用范围已收窄时在该行行尾加 `# skill-audit: ignore`——此前只说要"在 SKILL.md 说明"，但判定根本不看 SKILL.md，照着做也消不掉告警
- 回归复测 8 技能：P0/P1 无新增（P2 计数变化仅因「约定/示例路径」分桶合并）

### v2.0.3 (2026-09-15)
- 新增 `--platform`：产出**开放平台插件包**（`<技能名>-plugin.zip`），结构为 `<插件名>/.codebuddy-plugin/plugin.json` + `<插件名>/skills/<技能名>/...`
- 修「上传开放平台报 **压缩包缺少 .codebuddy-plugin/plugin.json 文件**」：平台分发单元是**插件**，技能包与本机技能包是两种形态，以前只能产出后者
- `plugin.json` 从 SKILL.md frontmatter 自动生成（`name`/`version`/`description`/`description_en`/`author`/`keywords`/`skills`），描述会剔除触发词堆砌并截断到 200 字
- 新增 `read_frontmatter()`：自解析 YAML，支持折叠块（`>-`）与列表；此前只有体检器内部有类似逻辑
- SKILL.md 新增「两种产出形态」对照表与开放平台插件结构说明
- 新增 `--flat-root`：平台包可不带顶层目录（zip 根即插件根），应对平台拒绝顶层目录的情况——两种形态无法从报错文案区分，故都保留

### v2.0.2 (2026-09-15)
- **交付标准写进流程**：终点是「本机技能目录里有一份能用的技能」，zip 只是分发副本
- `pack_skill.py` **安装改为默认行为**（原先要手动加 `--install`，导致打包完只剩一个 zip、本机没有技能），新增 `--no-install` 给"只帮别人打包"的场景；结尾固定打印「技能目录 / 分发 zip」，未安装时用 `!!` 明确告警
- `new_skill.py` 新增 `--install`：`--out` 指向技能目录之外时补装一份；结尾区分"技能已就绪"与"本机还没有这个技能"
- 防自毁保险：新增 `is_inside`，安装目标落在源目录之内时拒绝覆盖，避免原地打包自我删除
- 安装时同步创建空目录，本机副本结构与 zip 一致

### v2.0.1 (2026-09-15)
- **为上架做可发现性优化**：`display_name` 改为「技能生成器 · 生成/体检/打包」（一次覆盖三个搜索词），`display_name_en` 加 `Packager`
- `description` 扩充长尾触发说法（技能打包 / 导出技能 / 技能上架 / 技能体检 / 技能合规检查 / build skill），`description_zh` / `description_en` 补入上架、分发、publishable 等词
- `trigger` 从 13 条扩到 19 条
- 新增 `references/naming-and-discovery.md`：三层可发现性模型、词根占位实测数据（creator/maker/studio/kit/forge/foundry 均已被占，`skill-generator` 精确名零重名）、关键词穷举法、起名检查清单
- **`name` 保持不变**——实测确认它零重名，而所有候选替代词根都已被占用

### v2.0.0 (2026-09-15)
- **技能从「技能打包器」升级为「技能生成器」**：新增从一句需求生成技能的能力，名字、目录、zip 名同步改为 `skill-generator`
- 新增 `scripts/new_skill.py`：一条命令生成合规骨架（frontmatter 齐全 + 分节正文 + 按需子目录 + 自动 `scripts/setup.py` / `config/*.example.json`），**生成后自动体检**，闭环到 0 告警
- 新增 `references/generation-guide.md`：形态决策、description 写法对照、资源分层判据
- `pack_skill.py` 把**空目录**也写进 zip —— 否则刚生成的骨架解压后 `scripts/`、`references/` 会丢
- 两个脚本加 `sys.dont_write_bytecode`，运行后不再在技能目录里留 `__pycache__`
- 修体检器误报：`description` 写成 YAML 折叠块（`>-`）时被判「过短」；`description_zh` / `description_en` 同理漏检。新增 `fm_text()` 统一处理块标量
- `templates/SKILL.md.template` 的说明注释从**文件开头移到末尾**——原来照抄模板会直接触发 P0「缺 frontmatter」

### v1.0.1 (2026-09-15)
- 对齐技能市场分发规范：frontmatter 补齐 `display_name` / `display_name_en` / `description_zh` / `description_en` / `category`，`author` 改为真实署名
- 新增 `templates/SKILL.md.template`：含全套字段的技能骨架
- `audit_skill.py` 新增 `--market` 模式：第 9 类检查「市场分发」，字段定级依据实测而非规范文本
- `pack_skill.py` **默认按市场规范打包**（`--no-market` 可关），并在打包前逐项列出字段齐备情况
- 修误报：`const sid = resp.result.sessionId` 这类**属性访问**被当成明文凭据（`PROPERTY_CHAIN` 判定）
- `fm_value` 支持 `metadata:` 下的**缩进嵌套**字段

### v1.0.0 (2026-09-15)
- 首版（原名 skill-packager）：产物清点 → 边界判定 → 骨架 → 脱敏 → 体检 → 打包 → 安装 全流程
- `scripts/audit_skill.py`：8 类检查，P0/P1/P2 分级，同类问题聚合，密钥打码
- `scripts/pack_skill.py`：体检门禁 + 自动排除垃圾 + zip + 安装
- `references/checklist.md`：逐项判定标准与放行条件
- `references/sanitize-rules.md`：绝对路径与凭据的替换映射表
