# skill-generator · 技能生成器

按一句需求生成合规可用的技能骨架，或把对话里刚实现的功能**体检、脱敏、打包**成可分发、可上架的技能包。

覆盖 WorkBuddy 技能（Skill）从**从零创建**到**交付上架**的完整链路：需求澄清 → 建骨架 → 填内容 → 脱敏 → 体检 → 打包安装。

## 解决什么问题

- **不知道骨架该长什么样** —— 按技能基础结构一次建全：`SKILL.md` / `references/` / `scripts/` / `templates/`
- **上架字段缺斤少两** —— 自动补齐市场分发字段（`description_zh` / `description_en` / `version` / `author` 等）
- **带病打包** —— 打包前自动体检 9 类风险，输出 P0 / P1 / P2 分级报告
- **把不该发的东西发出去了** —— 硬编码路径、明文凭据、内网地址、个人数据自动识别并提示脱敏

## 安装

克隆到 WorkBuddy 技能目录即可被识别：

```bash
git clone https://github.com/liuyuming0823/skill-generator.git ~/.workbuddy/skills/skill-generator
```

Windows 路径：`%USERPROFILE%\.workbuddy\skills\skill-generator`

## 快速开始

```bash
cd ~/.workbuddy/skills/skill-generator

# 从一句需求生成骨架（生成后自动体检，并直接安装到本机技能目录）
python scripts/new_skill.py my-skill \
  --desc "一句话说清做什么/何时触发" \
  --triggers "触发词1,触发词2" \
  --dirs scripts,references \
  --deps requests

# 只体检
python scripts/audit_skill.py <技能目录>

# 体检 + 打包 + 安装（默认就是这条链路）
python scripts/pack_skill.py <技能目录>
```

## 两个脚本

### `new_skill.py` —— 生成骨架

| 参数 | 说明 |
|------|------|
| `<skill_name>` | 技能名，kebab-case |
| `--desc` / `--desc-zh` / `--desc-en` | 描述：做什么 / 何时触发 / 触发词 |
| `--display-name` / `--display-name-en` | 展示名 |
| `--triggers` | 逗号分隔的触发词 |
| `--dirs` | 要创建的目录，默认 `scripts,references` |
| `--deps` | 第三方依赖，会自动生成 `scripts/setup.py` |
| `--config-fields` | 配置字段，会生成 `config/settings.example.json` |
| `--author` / `--category` | 署名与分类，署名默认读 `~/.workbuddy/USER.md` |
| `--mode market\|local` | 按市场规范生成 / 本机自用 |
| `--out` / `--install` / `--force` | 输出目录 / 直接安装 / 覆盖同名 |
| `--no-audit` / `--json` | 跳过体检 / 输出 JSON |

### `pack_skill.py` —— 体检 + 打包 + 安装

```bash
python scripts/pack_skill.py <dir>               # 默认：技能目录包 + 安装到本机
python scripts/pack_skill.py <dir> --as-plugin   # 插件形态包（插件市场分发用）
python scripts/pack_skill.py <dir> --no-install  # 只出 zip，不安装
python scripts/pack_skill.py <dir> --no-market   # 跳过市场字段检查（本机自用）
python scripts/pack_skill.py <dir> --dry-run     # 只看会打进哪些文件
```

## 体检的 9 类风险

结构完整性、绝对路径、明文凭据、危险操作、外部依赖、数据模板、文件引用完整性、体积垃圾、市场分发字段。

分级：**P0 阻断**（真实凭据、破坏性操作）/ **P1 警告**（硬编码路径、外部依赖、引用缺失、缺市场必填字段）/ **P2 知情**（命名规范、外部副作用、运行时数据、垃圾文件）。

放行方式：对应行尾加 `# skill-audit: ignore`，或在技能根目录放 `.skillignore`。

## 产出形态，别搞混

| 形态 | 结构 | 用途 |
|------|------|------|
| 技能目录包 | 顶层 `{skill-name}/SKILL.md` + `references/` / `scripts/` / `templates/` | 本机包，**也是开放平台技能类目上传包**（不需要任何清单文件） |
| 插件形态包 | `.codebuddy-plugin/plugin.json` + `skills/{技能名}/SKILL.md` | 插件市场分发用，**不能**用于技能上架 |

## 目录结构

```
skill-generator/
├── SKILL.md                      # 技能主文档（完整工作流）
├── references/
│   ├── checklist.md              # 交付前检查清单
│   ├── generation-guide.md       # 骨架生成指引
│   ├── naming-and-discovery.md   # 命名与可发现性
│   └── sanitize-rules.md         # 脱敏规则
├── scripts/
│   ├── new_skill.py              # 生成骨架
│   ├── audit_skill.py            # 体检
│   └── pack_skill.py             # 打包 + 安装
└── templates/
    └── SKILL.md.template         # SKILL.md 骨架模板
```

## 依赖

纯 Python 标准库，无需安装任何第三方包。

## 相关项目

- [expert-packager](https://github.com/liuyuming0823/expert-packager) —— 配套的**专家**生成器：生成、校验、安装、打包 WorkBuddy 专家包

这对工具的分工是：**skill-generator 管「技能」，expert-packager 管「专家」**，两条上传规范不复用，别混。
