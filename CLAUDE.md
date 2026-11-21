# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 这个仓库是什么

这是一个**可移植的 Agent Skill**，不是可构建的应用。仓库根目录就是技能根目录，`SKILL.md` 的 frontmatter（`name`/`description`）是唯一清单；安装 = 把整个目录复制到某个 Agent 的 skills 目录。仓库没有 package manifest、构建步骤或 CI 打包，Release zip 只是源码快照。

文档本身就是运行逻辑：`SKILL.md` 是 Agent 的总控提示词，`references/` 与各模块的 `GUIDE.md`/DNA 文件是它按需读取的规则。改行为通常意味着改这些提示词文件，而不只是改代码。`SKILL.md` 是运行行为的权威来源，`README.md` 是面向使用者的说明；两者冲突时以 `SKILL.md`、`references/` 和模块规则为准。所有面向 Agent/用户的说明使用中文，并保持既有术语（`write` / `illustrate` / `draft`、`style_id`、`writing_profile`、`track` 等）不变。

复制技能目录不会安装依赖：`install.sh`/`install.ps1` 只做部署和一次 `check`，首次真正调用时才触发依赖检查与授权流程。

## 常用命令

安装（把技能部署到 `~/.codex/skills`、`~/.claude/skills` 或 `~/.agents/skills`）：

```bash
./install.sh
powershell -ExecutionPolicy Bypass -File install.ps1   # Windows；install.bat 只是转调
```

环境引导（`check` 只检查、不下载；`install` 才安装）：

```bash
bash scripts/bootstrap.sh check   --features draft,asr,images,tts
bash scripts/bootstrap.sh install --features draft,asr,images,tts
```

```powershell
powershell -ExecutionPolicy Bypass -File scripts/bootstrap.ps1 -Action check   -Features "draft,asr,images,tts"
powershell -ExecutionPolicy Bypass -File scripts/bootstrap.ps1 -Action install -Features "draft,asr,images,tts"
```

测试（必须用 **Python 3.10+**；脚本在 3.9 上会直接退出。若本机已有托管环境，从用户数据目录的 `runtime.json` 读取 `python` 路径）：

```bash
cd scripts
python3 -m unittest test_bootstrap_environment test_setup_dependencies test_doubao_voice
python3 -m unittest test_setup_dependencies.InstallerTests.test_write_only_never_discovers_or_installs_media_dependencies
```

模块测试需在该脚本目录内运行（测试使用同目录扁平导入）：

```bash
cd modules/render-history-video/scripts && python3 test_alignment.py
cd modules/history-image/scripts       && python3 test_transcription.py
# 需要 pyJianYingDraft + Pillow，否则自动 skip
cd modules/render-history-video/scripts && python3 test_export_jianying.py
```

Windows 原生启动器测试：`powershell -ExecutionPolicy Bypass -File scripts/test_bootstrap_native.ps1`。

流水线工具（入口脚本会在 `__main__` 调用 `activate_runtime()`，自动切换到托管 Python）：

```bash
python3 scripts/check_handoff.py <version>/plan.json --review <version>/semantic_alignment.json
python3 modules/render-history-video/scripts/export_jianying.py <version>/plan.json [--name NAME] [--drafts-root DIR]
python3 modules/render-history-video/scripts/render.py <version>/plan.json --workers 3
python3 modules/history-writing/scripts/history_tool.py search --query 关键词
python3 modules/history-image/scripts/transcribe_with_coli.py --audio narration.mp3 --out-dir <new-dir>
python3 modules/render-history-video/scripts/align_with_coli.py --audio narration.mp3 --storyboard STORYBOARD.json --out-dir <new-dir>
```

仓库没有 lint、format 或 typecheck 配置，也没有统一的测试运行器；不要臆造这些命令。验证以离线单元测试、脚本 JSON 报告和实际产物检查为准。

测试必须保持离线：用 `unittest.mock` 隔离网络、安装和用户配置；不得调用真实 TTS/生图服务，也不得写入真实用户数据目录。

## 架构

### 入口 → 跨阶段规则 → 模块

1. **总入口 `SKILL.md`**：解析用户请求的三个开关（写作/配图/剪映草稿），读取 `references/setup.md` 处理依赖，按赛道选择 `writing_profile`，再路由到模块。它还定义阶段授权、配音缺失时必须给出的两条路径、事实优先级和交付规则。
2. **`references/`**：跨模块规则（依赖初始化、豆包配音、项目记录、事实与风格、人物写作适配、扩展赛道）。
3. **`modules/`**：四个自包含模块，各自以 `GUIDE.md` 为索引，规则、语料、风格文件、脚本都在模块内部：
   - `history-writing/`：历史叙事 DNA、57 篇语料、检索/字数/审计工具。
   - `characterstory-writing/`：玉玲人物故事 DNA 与语料，原稿只作写法样本。
   - `history-image/`：分镜流程、三种 `style_id`、参考图、Coli 转录。
   - `render-history-video/`：时间轴计划、语义核验、推镜/切点运动、剪映导出与可选本地渲染。

模块文档里的相对路径以该 `GUIDE.md` 所在目录为基准；“`<story-studio目录>`”才指技能根目录。新增赛道/画风必须同步注册到 `SKILL.md` 的路由表/模块 `GUIDE.md` 的索引；不要创建空模块或宣称未验证的能力。

### 运行时层（`scripts/`）

- `bootstrap.sh` / `bootstrap.ps1`：零 Python 依赖的原生启动器。没有 Python 3.10+ 时从 Astral 下载 `uv`，校验 SHA-256、检查归档路径穿越后安装用户私有 Python 3.11，再转交 `bootstrap_environment.py`。
- `bootstrap_environment.py`：按需安装 Node LTS、FFmpeg/ffprobe、Coli、剪映（Windows），发现/创建草稿目录，最后调用 `setup_dependencies.py install`。
- `setup_dependencies.py`：组件检查与安装的唯一来源。`features` 决定依赖；版本是硬编码且被校验的（`pyJianYingDraft==0.3.1`、`websockets==15.0.1`、`@marswave/coli@0.0.20`）。输出 JSON 报告，`ready` 只代表依赖检查通过，不代表生图授权或 TTS 服务已验证。
- `studio_runtime.py`：所有脚本共享的路径/配置层。负责用户数据目录、`runtime.json`、受管 Python 重执行、工具 PATH、草稿目录优先级、外部技能发现，并**拒绝把配置写进技能源码目录**。
- `voice_settings.py` / `doubao_tts.py`：API Key 只从环境变量或技能目录之外的专用凭据存储读取；TTS 不自动重试，失败报告不得泄露 Key、provider 响应或本地路径。

运行时数据一律在技能目录之外：Windows 为 `%LOCALAPPDATA%\story-studio`，macOS/Linux 为 `${XDG_DATA_HOME:-~/.local/share}/story-studio`。`runtime.json` 只记录 `python`、`coli_bin`、`drafts_root`、`tool_dirs`；`STORY_STUDIO_CONFIG`、`JY_PROJECTS_ROOT`、`COLI_BIN` 可覆盖相应位置。草稿目录优先级：显式 `--drafts-root` → `JY_PROJECTS_ROOT` → `runtime.json` → 剪映设置 → 唯一标准目录，否则报错。

模块脚本若需要根目录的共享模块，按现有写法把根 `scripts/` 插入 `sys.path`（如 `parents[3] / "scripts"`），并在 `__main__` 中先调用 `activate_runtime()`；不要自己复制配置/路径逻辑。

### 生产流水线与数据契约

```
写作 → 定稿/事实记录
     → 豆包 TTS（narration.mp3 + 同目录 voice.json）或用户提供的最终音频
     → Coli 本地转录（tokens/timestamps/duration）→ SRT
     → STORYBOARD.md/json（逐镜原文锚点、start/end、图片映射）
     → 实际生图/封面
     → plan.json → check_handoff.py → export_jianying.py（默认）或 render.py（明确要求 MP4）
```

- `STORY_PROJECT.json` 位于**用户项目目录**，记录 `track`、`writing_profile`、`style_id`、本次阶段范围、真实产物的路径与 SHA-256。空的预期文件必须写 `null`，不能冒充已有产物。
- `plan.json` 由 `modules/render-history-video/scripts/render.py` 的 `load_plan()` 统一校验：镜头 ID 唯一且安全、时间连续无重叠/空隙、首镜从 0 开始、末镜覆盖整段音频、尺寸为偶数、默认 1080×1920/30fps。导出和本地渲染都复用这一校验，不要另写一套。
- `semantic_alignment.json` 必须记录实际素材哈希、逐镜覆盖、`workflow`（`audio_first` / `srt_only`）和 `text_audio_review`。`check_handoff.py` 是导出前的强制门禁；它只验证产物身份、时间轴与记录覆盖，不验证语义真伪或播放效果。
- 修改任何上游产物都要新建版本目录；默认不覆盖旧 SRT、旧草稿或旧成片。

### 不可破坏的约束

- **阶段授权**：只执行用户本次明确开启的阶段。`storyboard`（只出分镜/提示词）、`prompt`、`generate`（实际生图）严格区分；提示词或文件名不构成生图授权，未请求配音不发付费请求，未请求 MP4 不自动本地渲染。
- **事实优先**：语料、DNA、对标稿是写法样本，不是史实证据。精确数字、引语、遗言、心理活动必须有依据；证据只支持范围时不得补成精确值。
- **可移植与隐私**：技能源码、文档、示例、提交中不得出现真实姓名、用户名、联系方式、本机绝对路径、密钥或会话记录。配置和用户产物只能写到用户数据目录/工作区。新增文档时也不要写入具体机器的路径。
- **下载与安装安全**：仅 HTTPS，校验 SHA-256，限制归档大小与成员数，解压必须走 `safe_extract`/`Expand-VerifiedZip` 的路径穿越检查；安装外部技能不覆盖已有目录并保留 LICENSE。不要为了让安装“更顺”而放宽这些检查。
- **不换后端**：Coli 只做本地转录，不调用火山字幕 API，也不恢复其凭据；生成失败要报告具体阻塞，不能静默退回字符比例估时或另一套实现。
- **平台验证边界**：完整剪映桌面草稿链以 Windows + 中国剪映为目标；macOS/Linux 的 bootstrap 与文件生成能力不等于完整剪映桌面导出已验证，不能据此宣称跨平台一键成片。

## 修改时的落点

- 改阶段范围、路由、跨阶段规则 → `SKILL.md`。
- 改某赛道写法 → 对应模块的 `GUIDE.md` 及其 DNA/规则文件；保持每条规则只有一个维护来源。
- 改配图画风 → `modules/history-image/styles/<style-id>.md`，并在模块 `GUIDE.md` 和 `SKILL.md` 注册；不要混用不同画风的规则。
- 改剪映/时间轴/运动逻辑 → `modules/render-history-video/`，优先复用 `render.load_plan` 与共享运动曲线。
- 改依赖、路径、配置、安装安全 → `scripts/`，并同步 `references/setup.md` 与测试。
- 新增模块 → 建目录和 `GUIDE.md`，注册路由，并补真实路径测试；不要只声明“支持”而不验证组合使用。
