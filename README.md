# 🍪 微信聊天记录分析工具

> 一个运行在 Claude Code 里的 Skill：输入联系人的名字，自动生成**双人聊天行为可视化 + AI 人格对比分析报告**。无需 API Key，分析由 Claude Code 本地完成。

---

## 效果预览

<p align="center"><img src="pics/preview-header.png" width="70%" alt="报告总览"></p>

<table>
<tr>
<td width="50%"><img src="pics/preview-wordcloud-big5.png" alt="词云对比 + Big Five 蝴蝶图"></td>
<td width="50%"><img src="pics/preview-heatmap.png" alt="聊天频率热力图"></td>
</tr>
</table>

<p align="center"><img src="pics/preview-mbti.png" width="70%" alt="MBTI 双人推断"></p>

<p align="center"><img src="pics/preview-style.png" width="70%" alt="AI 风格总结"></p>

所有内容整合为一份精美 HTML 报告，图表可直接截图发布。

---

## 使用要求

| 项目 | 要求 |
|------|------|
| 操作系统 | macOS 12 及以上 |
| 微信版本 | Mac 客户端 4.x |
| Python | 3.10 及以上（`python3 --version` 检查） |
| Claude Code | 已安装（`claude --version` 检查） |

---

## 安装部署

**克隆仓库，在目录内打开 Claude Code，即可使用。**

```bash
git clone git@github.com:HxWGuang/ginger_wechat_portrait.git ~/wechat-analyzer
cd ~/wechat-analyzer
claude
```

仓库内置了 `.claude/commands/analyze-wechat.md`，Claude Code 在该目录下会**自动识别**，无需任何注册步骤。

> **想在任意目录都能用？** 在 Claude Code 里说：
> `"帮我把 analyze-wechat 全局安装"`
> Claude 会自动把命令文件复制到 `~/.claude/commands/`。

---

## 快速开始

在仓库目录打开 Claude Code，直接运行：

```
/analyze-wechat
```

或者自然语言：

```
帮我分析和小明的聊天记录
```

**Skill 会自动完成从安装依赖到生成报告的全部流程。**

### 高级选项

`main.py` 支持以下命令行参数，在 Step 9a 环节生效：

```bash
# 调整采样量（默认 100 条，增大可获得更精准的分析）
python main.py export_xxx.csv --sample-size 500

# 全量分析：使用所有过滤后消息（上限 3000 条），不进行采样
python main.py export_xxx.csv --full

# 自定义自己和对方的名字（默认从 CSV 文件名推断）
python main.py export_xxx.csv --self-name "我" --partner-name "小明"
```

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `--sample-size N` | 100 | 采样消息数上限 |
| `--full` | 关闭 | 全量分析模式，跳过采样 |
| `--output DIR` | `./wechat_analysis_output` | 输出根目录（每个联系人自动创建子目录） |
| `--self-name NAME` | 从 meta 读取 | 自己的显示名 |
| `--partner-name NAME` | 从 meta 读取 | 对方的显示名 |

> 量化语言特征（观点词率、情绪词率等）**始终基于全部过滤后消息计算**，不受采样量影响，确保统计指标尽可能精准。

---

## 需要手动完成的前置步骤

以下是一次性操作，完成后后续所有分析均全自动。

### 密钥提取（仅首次，二选一）

微信聊天数据库使用 SQLCipher 加密，首次使用需要从微信进程内存中提取密钥。

**方案 A（推荐）：sudo 运行内存扫描器**

Skill 会自动从 [ylytdeng/wechat-decrypt](https://github.com/ylytdeng/wechat-decrypt) 克隆并编译 C 扫描器，你只需在 Terminal.app 中执行一次 `sudo` 命令，约 10 秒完成。

**方案 B：给微信做 ad-hoc 重签名**

```bash
sudo codesign --force --deep --sign - /Applications/WeChat.app
```
重签名后重启微信，之后无需 sudo 也可运行扫描器。

> 两种方案都**不需要关闭 SIP**（与旧版不同）。详细说明见 [安装指南.md](./安装指南.md)

---

## Skill 会问你哪些问题？

整个流程**最多只会被问 3 次**：

| 时机 | 问题 | 频率 |
|------|------|------|
| 启动时 | 你想分析和谁的聊天记录？ | 每次（除非命令里已指定） |
| 首次运行（仅 Level 3） | 确认微信已登录并给出手动命令 | 仅首次提取密钥时 |
| 按需 | 找到多个同名联系人，选择哪个？ | 有重名联系人时 |

其余所有步骤均**全自动完成**，无需干预。

---

## 输出文件

分析完成后，所有文件保存在工具目录的 `wechat_analysis_output/` 下（默认 `~/.claude/wechat-analyzer/wechat_analysis_output/`），**按联系人自动创建独立子目录**，分析多人时互不干扰：

```
wechat_analysis_output/
├── 小明/
│   ├── export_小明.csv            ← 导出的原始消息（中间文件）
│   ├── export_小明.json
│   ├── export_小明.meta.json      ← 昵称、头像等元数据
│   ├── report.html                ← 完整 HTML 报告（浏览器打开）
│   ├── report.css
│   ├── personality_result.json    ← 自己的 AI 分析结果
│   ├── partner_result.json        ← 对方的 AI 分析结果（双人模式）
│   ├── personality_input.json     ← 分析输入（消息样本 + 统计）
│   ├── partner_input.json         ← 对方的分析输入
│   ├── personality_raw.json       ← 分析原始数据备份
│   └── charts/
│       ├── hourly.png             ← 24 小时发消息分布
│       ├── monthly_trend.png      ← 月度消息趋势
│       ├── weekday_bar.png        ← 星期分布
│       ├── word_cloud_pair.png    ← 双人高频词词云
│       ├── length_dist.png        ← 消息长度分布
│       └── radar.png              ← Big Five 雷达图
├── 小红/                          ← 分析其他人时自动创建独立子目录
└── ...
```

---

## 隐私说明

- 所有数据处理在**本地**完成
- AI 人格分析由 **Claude Code 本身**完成，不需要 Anthropic API Key，不向外部发送消息内容
- 不会收集或上传任何数据到其他地方
- 请勿用于分析他人设备上的数据

---

## 技术流程

见 [SKILL.md](./SKILL.md)

---

*macOS 12+ · WeChat 4.x · Python 3.10+ · Claude Code*
