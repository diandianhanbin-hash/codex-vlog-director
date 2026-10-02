# Hiking Highlight Analyzer

一个用于 WorkBuddy 的轻量徒步视频剪辑决策 Skill。

## 它解决什么问题

它不负责“看原始视频”和“不负责真正剪片”，而是位于两者中间：

```text
视频分析 Skill
    ↓
时间轴 / 关键帧 / 场景描述 / 转录
    ↓
Hiking Highlight Analyzer
    ↓
edit-plan.yaml
    ↓
FFmpeg / 视频剪辑 Skill
```

核心目标是把“这段视频发生了什么”转换为“这段视频应该怎么剪”。

## 目录结构

```text
hiking-highlight-analyzer/
├── SKILL.md
├── README.md
├── references/
│   ├── decision_rules.md
│   ├── editing_preference.md
│   ├── output_contract.md
│   └── role_taxonomy.md
└── templates/
    ├── edit-plan.yaml
    └── preference-profile.yaml
```

## WorkBuddy 导入

WorkBuddy 官方支持以 Skill 目录 / ZIP 的方式组织技能，核心文件为 `SKILL.md`，并可包含 `references/` 与 `templates/`。

如果通过开放平台或技能创建入口导入 ZIP，请确保 ZIP 解压后顶层目录为：

```text
hiking-highlight-analyzer/
```

而不是把所有文件散落在 ZIP 根目录。

## 推荐调用方式

### 场景 1：上游已经有结构化分析结果

```text
请使用 hiking-highlight-analyzer 分析这份视频时间轴。
目标成片 8 分钟，保持自然纪实风格，不要过度压缩行走过程。
请输出 edit-plan.yaml。
```

### 场景 2：强调去重

```text
请重点处理大量重复的林间行走和山景，只留下能体现路线推进或环境变化的代表片段。
```

### 场景 3：慢节奏户外片

```text
目标成片 10 分钟左右，允许重要风景停留更久，尽量保留环境声，普通步行镜头仍然要严格去重。
```

## 输入建议

上游分析结果最好包含：

- 时间戳
- 场景/画面描述
- 事件描述
- 关键帧观察
- Whisper 转录
- 技术质量信息

上游结果应包含上面的时间戳、场景/画面描述、事件、关键帧观察、可用转录和技术质量信息。

## 输出

输出 `edit-plan.yaml`，重点字段：

- `roles`
- `scores`
- `similarity_group`
- `decision`
- `recommended_start`
- `recommended_end`
- `reason`
- `story_check`

决策只有四种：

- `must_keep`
- `keep`
- `optional`
- `drop`

## 设计原则

这个 Skill 刻意不写 Python 脚本，因为它解决的是“剪辑决策模型”，不是视频处理执行问题。这样更容易调整个人偏好，也能与不同的视频分析 Skill 和 FFmpeg Skill 解耦。

## 后续最值得扩展的两个方向

1. 增加一个 `personal-preference-calibration` 流程：每次记录你人工改掉哪些 AI 选择，逐渐更新 `editing_preference.md`。
2. 增加多级长视频筛选：先对 1–2 小时素材做粗筛，再对候选 15–30 分钟片段做二次精细分析。
