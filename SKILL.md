---
name: ym-skill-generator
slug: ym-skill-generator
display_name: 技能生成器 · 生成/体检/打包
display_name_en: Skill Generator & Packager
displayName: 技能生成器
description: "按需求生成一份合规、可用的技能骨架并填充内容，或把对话里刚实现的功能体检、脱敏后 打包成可分发、可上架的技能包。当用户说「做个技能」「生成技能」「新建技能」「写个 skill」 「把这个功能打包成技能」「打包技能」「技能打包」「存成技能」「沉淀为技能」「做成技能」 「导出技能」「技能上架」「上传开放平台」「上架开放平台」「发布技能」「技能体检」 「技能合规检查」「体检器自检」「改版本号」「bump 版本」 「package skill」「create skill」「build skill」时使用。 生成时按技能基础结构建骨架（SKILL.md / references / scripts / templates）并补齐市场分发字段； 打包前自动体检 9 类风险：结构、绝对路径、明文凭据、危险操作、外部依赖、数据模板、 文件引用、体积垃圾、市场分发字段，输出 P0/P1/P2 分级报告，报错附中文「怎么修」； 误报可用 `--ignore-rule` / `.skillignore` 按规则放行，规则含义用 `--explain` 查。 技能上传包就是技能目录本身（{skill-name}/SKILL.md + references/ + scripts/ + templates/）， 平台不需要额外的清单文件。 也负责生成技能发布用的图标（512×512、PNG/JPG、≤500KB）：用 make_icon.py 出图并处理， 图标**不进技能包**，上架时在平台单独提交。当用户说「生成技能图标」「技能图标」「做个图标」 「图标合规检查」「图标超 500KB」时也使用。"
description_zh: 按需求生成技能骨架，体检脱敏后打包成可上架分发的技能包
description_en: Generate, audit, sanitize and package agent skills into publishable bundles
summary: 按需求生成技能骨架，或把已有实现体检、脱敏后打包成可分发、可上架的技能包
category: dev-programming
version: 2.3.0
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
  - 体检报错看不懂
  - 体检误报
  - 体检器自检
  - 改版本号
  - bump 版本
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

### 30 秒上手（只记三条命令）

```bash
python scripts/new_skill.py <英文名> --display-name "<中文名>" --triggers "触发词"   # 生成骨架+自动体检
python scripts/audit_skill.py <技能目录> --market   # 上架前体检
python scripts/pack_skill.py <技能目录>              # 体检 + 打包 + 装本机
```

| 我想… | 用哪条 |
|---|---|
| 从一句需求建技能 | `new_skill.py`（默认生成即安装） |
| 只给已有技能做安全自检 | `audit_skill.py <目录>` |
| 打包发给别人 / 上架 | `pack_skill.py <目录>`（默认装本机 + 出 zip） |
| 发新版前改版本号 | `bump.py <目录> -m "改了什么"` |
| 体检报错 / 怀疑体检器坏了 | `audit_skill.py --explain <规则名>` / `--selftest` |

**退出码**（四个脚本统一）：`0` 成功 · `1` 有 P1 · `2` 有 P0 · `3` 参数/目录错 · `4` 写入打包失败；非 0 时会打印「怎么修」。

```
① 生成  需求 → 定形态 → 建骨架 → 填内容  ┐
② 打包  已有实现 → 清点产物 → 脱敏      ┴→ 体检 → 打包 → 安装（默认）
```

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

**9 类**：结构 · 绝对路径 · 凭据 · 危险操作 · 外部依赖 · 数据模板 · 引用 · 体积与垃圾 · 市场分发字段。
逐项判定标准与放行条件见 `references/checklist.md`。

第 9 项的开关两边不同：**打包时默认检查**（打包往往是分发的第一步），要跳过加 `--no-market`；
**单独体检时默认不查**，上架前才加 `--market` —— 否则体检任何一个老技能都会满屏告警。

### 新增：按规则放行与自解释

| 场景 | 怎么做 |
|---|---|
| 某条规则是误报，临时不想看 | `--ignore-rule <规则名>`（可重复） |
| 不确定这条规则判什么 | `--explain <规则名>`：判什么 / 为什么 / 怎么改 / 怎么放行 |
| 永久放行某规则 | 技能根建 `.skillignore`，写一行 `rule:规则名` |
| 排除某些文件不参与体检与打包 | `.skillignore` 里每行一个 glob |
| 示例/产物路径提示太吵 | 默认已折叠成一行计数，`--show-noise` 才展开 |
| 缺的 frontmatter 字段想自动补 | `--fix`（先看建议）→ `--fix --write`（落盘，写前自动备份） |

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

平台是在**技能目录根**找 `SKILL.md`：打成 `skills/<技能名>/SKILL.md` 那种插件形态再传，
会直接报「**压缩包缺少 SKILL.md 文件**」。所以**技能上传包 = 技能目录包 = 默认产出**。

打包开始时先逐项列出市场分发字段的齐备情况（`已填` / `缺失` / `未填（建议补）`），再决定放行还是打回。
产出 zip 顶层目录 = 技能名，空目录也会写进去（刚生成的骨架结构不会丢）。
三类东西自动排除（并在打包时逐条打印，不会悄悄丢）：

| 类别 | 例子 |
|---|---|
| 构建缓存 | `__pycache__`、`node_modules`、`.venv`、`*.pyc`、`*.log` |
| **仓库元数据** | `.git/`、`.gitignore`、`.gitattributes`、`README.md`、`CHANGELOG.md`、`LICENSE` |
| **发布图标** | `icons/`（防御性排除；图标本就不该放进技能目录，见 Step 9） |

**P0 永远阻断**，`--allow-p1` 也不能越过。覆盖已有同名技能需要 `--force`：
旧版会先移到 `~/.workbuddy/skill-backups/` 再让新版就位，不会直接删。

两种情况会**自动跳过安装**（不报错，但结尾用 `!!` 提醒技能还不可用）：源目录已在技能安装目录内
（就是已装技能，也是**防止原地打包把自己删掉**的保险）、或加了 `--no-install`。
安装后技能即刻就绪；若未出现在可用列表，重启会话即可被识别。

### 发布 ≠ 上线

命令返回成功只代表平台**已受理**；线上字段分三档更新，判据是**版本列表里有没有刚发的版本号**。三档表、三个免登录只读接口、插件形态包（`--as-plugin`）的结构与 `--flat-root` 说明，见 `references/publishing.md`。

## Step 9 · 技能图标

平台要求 512×512、PNG/JPG、≤500KB，由「图标」表单**单独收** —— **不进技能包，也不要放进技能目录**（放进目录会让绑定仓库发布被拒）。
完整流程（提示词 → 居中裁切 → 清生成标 → 压缩 → 自查）见 `references/icon-guide.md`。

## 技能目录结构约定

`SKILL.md`（必需）+ `references/`（按需加载的参考资料）+ `scripts/`（可执行脚本）+ `templates/`（一律 `.md` 后缀）+ `assets/`（产出用资源）+ `config/`（只放 `*.example.*` 空模板）。

`SKILL.md` 用反引号路径引用子资源（`` `references/xxx.md` ``）——本机 28 个技能全部采用这种写法，比 `@` 前缀更稳。

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
10. **只交一个 zip 就收工** —— 用户要的是"有个技能能用"，不是一堆文件。把骨架建在工作区却忘了 `--install`，本机就一直没有这个技能，用户当然会问"怎么只生成了个 zip"。
11. **把技能打成插件形态去上传** —— 传 `skills/<技能名>/SKILL.md` 那种包，平台在**技能目录根**找不到 SKILL.md，直接报「**压缩包缺少 SKILL.md 文件**」。技能上传包 = 默认产出的技能目录包（顶层 `<技能名>/SKILL.md`）。至于「压缩包缺少 .codebuddy-plugin/plugin.json」—— 那是**专家/插件**那条线的报错，别套到技能上。
12. **把技能图标放进技能目录** —— 图标是平台创建技能时**单独收的一个字段**。放进技能目录，**打包时**会被排除（无害），但**绑定 GitHub 仓库发布时会被平台直接拒收**（报「不支持的文件类型」）。一律输出到技能目录之外（默认 `~/.workbuddy/skill-icons/`）。
13. **中文展示名写成下划线 `display_name`（平台不认）** —— **平台只认驼峰 `displayName`**；`display_name` 是 WorkBuddy 本机的字段，平台完全不读。缺了不报错、发布照样成功，但**商店列表里直接显示英文 slug**（例如 `ym-skill-generator`）—— 静默失败，最容易漏。要发布的技能**两个都写**：`display_name` 给本机，`displayName` 给平台。另外 `display_name` 里的「 · 功能列举」后缀不要带进 `displayName`，展示名越干净越好。
    同一个字段族还有两处：平台发布 CLI 要求 `slug`（与 `name` 一致），缺了硬报错 `SKILL.md 缺少 slug`；`summary` 是商店列表摘要，一般直接复用 `description_zh`。

14. **`category` 用了平台枚举外的值** —— 平台只认 13 个 key：`office-efficiency` / `content-creation` / `dev-programming` / `data-analysis` / `design-media` / `ai-agent` / `knowledge-management` / `business-ops` / `education` / `professional` / `it-ops-security` / `life-service` / `pay-skill`。传别的值不报错，但上架后显示「**未分类**」。注意 `development` **不是**平台值，对应的是 `dev-programming` —— `new_skill.py` 的 `--category` 默认值已改对，体检器也会校验（`SKILLHUB_CATEGORIES`）。

15. **以为 CLI 发布能把分类一起改掉** —— SkillHub CLI 的发布 payload 固定只有 `slug` / `version` / `displayName` / `summary` / `description` / `tags` / `license` / `homepage` / `changelog` 九个键，**没有 `category`**。于是出现最迷惑的一幕：SKILL.md 里分类写对了、CLI 也返回 `✓ Published`，线上分类纹丝不动。改分类只能走**网页 dashboard**；发布后用 `GET /api/v1/skills/<slug>` 复核 `skill.category`。

上架阶段还有 2 条坑（非白名单文件导致整次发布被拒、`✓ Published` ≠ 已上线），见 `references/publishing.md`；低频坑见 `references/pitfalls.md`。

## 版本历史

（完整历史见 `references/CHANGELOG.md`，这里只留最近两版）

### v2.3.0 (2026-09-16)

- **可靠性**：新增 `scripts/_friendly.py`（四个脚本共用的中文报错 + 统一退出码）；
  `--force` 覆盖改为「临时目录 + 备份到 `~/.workbuddy/skill-backups/` + 原子替换」，
  不再 `rmtree` 后重建；pip 安装加重试与清华源兜底；新增 `audit_skill.py --selftest`。
- **降误报**：`.skillignore` 真正生效（此前文档写了、代码没读）；新增 `--ignore-rule` /
  `--explain <规则名>` / `--show-noise`；内置三类噪音豁免；新增 `--fix [--write]`。
- **新命令**：`scripts/bump.py`（递增版本 + 写版本历史，一条命令）。
- **文档瘦身**：正文 27255 → 约 15000 字符，细节迁 `references/`（CHANGELOG / icon-guide /
  publishing / pitfalls）；顶部加 30 秒 TL;DR 与退出码表。

### v2.2.3 (2026-09-15)

- `sanitize-rules.md` 补一条硬要求：**`# skill-audit: ignore` 必须加在被判定的那一行行尾**。
- 相关技能更名：`ym-skillhub-publisher` 已与 `skillhub-store` 合并为 **`ym-skillhub`**。
