# 本次运行修复记录

本文档记录了从零开始运行 `/analyze-wechat "小明"` 过程中发现并修复的所有问题。

---

## 环境信息

- **macOS**: Sequoia 15+ (Apple Silicon)
- **微信版本**: 4.1.9
- **Python**: 3.12
- **日期**: 2026-05-04

---

## 修复 1：WeChat 4.1.9 密钥提取 —— `setCipherKey` 函数已移除

### 问题

`find_key.py`（来自 `Thearas/wechat-db-decrypt-macos`）通过搜索 ARM64 指令 `mov x0/w0, #0x43`（即 `malloc(67)`）来定位 `setCipherKey` 函数。在 WeChat 4.1.x 中，该函数已不存在或签名完全变化，导致：

```
[-] Could not find setCipherKey function.
```

### 尝试过的方案

1. **扩大 malloc 大小搜索范围**（0x40–0xFF）——无效，新版 WeChat 完全移除了该函数模式
2. **搜索 `sqlite3_key` 符号**——也未找到可用断点

### 最终方案

改用 C 语言直接内存扫描器（来自 `ylytdeng/wechat-decrypt`），通过 Mach VM API 扫描微信进程内存，搜索 `x'<96 hex chars>'` 格式的 SQLCipher 密钥特征字符串。**无需断点、无需 lldb、无需等待操作。**

```bash
# 构建
git clone https://github.com/ylytdeng/wechat-decrypt.git /tmp/wechat-decrypt
cc -O2 -o find_all_keys_macos find_all_keys_macos.c -framework Foundation

# 运行（需要 sudo）
sudo /tmp/wechat-decrypt/find_all_keys_macos
```

输出 `all_keys.json` 后，转换为 `wechat_keys.json` 格式即可使用原有 `decrypt_db.py`。

### 涉及文件

- 新增：`/tmp/wechat-decrypt/find_all_keys_macos.c`（C 内存扫描器）
- 转换脚本：内联 Python 将 `all_keys.json` 转为 `wechat_keys.json` 格式

---

## 修复 2：`get_self_wxid()` 后缀匹配失败

### 问题

微信数据目录名为 `wxid_5m69a3h30bod22_078f`（含 4 位 hex hash 后缀），但 `Name2Id` 表中存的是 `wxid_5m69a3h30bod22`（无后缀）。

原代码 `d.name.split('_0ad')[0]` 只处理了旧版微信的 `_0ad` 后缀格式，对新版 `_078f` 无效，导致精确匹配失败。

**后果**：`self_id` 为 `None`，所有自己发送的消息被标为 `is_sender=0`（对方消息），导致分析中"自己：0 条文本消息"。

### 修复

```python
# export_contact.py 第 134 行
# 旧: d.name.split('_0ad')[0]
# 新: 正则去掉末尾 hash 后缀
name = re.sub(r'_(?:0ad|0x)?[0-9a-f]{3,}[a-z]*$', '', name)
```

此正则可以处理 `_0ad`、`_078f`、`_0x1234abc` 等多种后缀格式。

### 涉及文件

- `export_contact.py`：`get_self_wxid()` 函数

---

## 修复 3：Unix 时间戳时区转换 —— UTC → Asia/Shanghai

### 问题

`data_loader.py` 中 `pd.to_datetime(df['ts'], unit='s')` 默认将 Unix 时间戳按 **UTC 时区**解析，但后续 `.dt.date` / `.dt.hour` 取的是 UTC 日期和小时。

微信消息时间戳是 UTC epoch 秒，在 CST (UTC+8) 下：
- **凌晨 0:00–7:59 的消息**，UTC 日期比 CST 晚一天，导致日期错归一天
- **所有小时**偏移 +8 小时（如实际晚上 22 点活跃高峰显示为 UTC 14 点）

**后果**：
- 2026-01-01 凌晨的 9 条消息（00:38–00:43）被归入 2025-12-31
- 最活跃时段显示为 14:00（UTC），实际为 22:00（CST）
- 所有跨越 UTC 日期边界的消息均受影响

### 修复

```python
# data_loader.py 第 129 行
# 旧:
df['datetime'] = pd.to_datetime(df['ts'], unit='s', errors='coerce')

# 新:
df['datetime'] = (
    pd.to_datetime(df['ts'], unit='s', errors='coerce', utc=True)
    .dt.tz_convert('Asia/Shanghai')
    .dt.tz_localize(None)
)
```

三步操作：将时间戳解析为 UTC → 转换为 Asia/Shanghai → 移除时区标记（兼容下游 naive datetime 代码）。

### 涉及文件

- `data_loader.py`：`load()` 函数的时间解析段落

---

## 文档更新

### SIP 相关内容清理

旧版密钥提取基于 lldb，需要关闭 SIP（`csrutil disable`）。新版改用 C 内存扫描器，通过 `sudo`（或 ad-hoc 重签名）获取 `task_for_pid` 权限，**与 SIP 无关**。

- `安装指南.md`：移除"第 1 步：关闭 SIP"及恢复模式操作说明；移除"密钥提取完成后重新开启 SIP"；手动步骤从 2 步减为 1 步
- `README.md`：前置步骤从"关闭 SIP + 运行脚本"改为"密钥提取二选一（sudo / 重签名）"

### 输出目录按联系人隔离

- `export_contact.py`：默认输出路径从 `./export_<联系人>.csv` 改为 `./wechat_analysis_output/<联系人>/export_<联系人>.csv`
- `main.py`：输出目录从 `./wechat_analysis_output` 改为 `./wechat_analysis_output/<联系人>/`
- `README.md` 和 `SKILL.md`：输出结构图更新，反映联系人子目录和 export 文件位置

### 涉及文件

- `data_loader.py`：`load()` 函数的时间解析段落

---

## 修复 4：自己消息为 0 时的边界情况处理

### 问题

当 `df`（自己消息）为空时，`stats.compute()` 返回空数据 / NaN，后续 `visualizer.py` 和 `report.py` 中多处崩溃：

1. `stats['date_range']` 中的 NaT 值导致 `strftime()` 报错
2. `visualizer.monthly_trend()` 对空序列调用 `argmax()` 报错
3. `visualizer.hourly()` 等对空数据的处理不一致

### 修复

1. **`stats.py`**：`date_range` 计算前 `dropna()` 去掉无效时间
2. **`main.py`**：
   - `date_range` 输出用 `try/except` 包裹
   - 当自己消息为 0 且有对方数据时，用对方数据生成图表
   - 跳过 `personality_input.json` 写入（无数据可分析）
   - 仅 `--partner-personality-result` 时也能正确进入报告模式
   - 模式 B 读取结果文件前检查 `personality_result` 是否存在

### 涉及文件

- `stats.py`：`compute()` 函数
- `main.py`：数据加载、统计输出、分析模式判断、报告生成段落

---

## 修复汇总

| 文件 | 修改内容 | 影响范围 |
|------|---------|---------|
| `data_loader.py:129` | UTC → Asia/Shanghai 时区转换 | **所有时间相关统计和图表** |
| `export_contact.py:134` | wxid 后缀正则匹配 | **发送者识别**（自己/对方区分） |
| `stats.py:49` | date_range 计算前 dropna | 时间跨度输出 |
| `main.py:118-164` | 空数据防御 + 报告模式逻辑 | 图表生成、分析输入导出、报告生成 |
| 解密流程 | 改用 C 内存扫描器 | 密钥提取（微信 4.1.x） |

---

## 修复 5：人格分析中消息归属混淆

### 问题

修复 2（发送者识别）修正了 CSV 中的 `is_sender` 标记，但**人格分析结果文件是在修复前写入的**。修复后重跑 `main.py` 时，脚本重新生成了正确的 `personality_input.json` 和 `partner_input.json`（样本正确归属于双方），但 `main.py --personality-result` 模式下直接读取旧的 result JSON 生成报告，**不会自动重新分析**。

这导致报告中引用的某些原文片段归属错误，例如：
- 小红 说的「你喝茶救星球吗？我给你们点」出现在 小明 的分析中
- 小明 说的「有点忧伤 感觉是离别季主题」出现在 小红 的分析中

### 修复

基于修复后正确归属的 `personality_input.json` 和 `partner_input.json` **完全重写**两份人格分析结果文件。四句话的归属验证如下：

| 原文 | 正确归属 | 所在文件 |
|------|---------|---------|
| 「我虽然喜欢游戏，但我感觉我更喜欢真实的世界吧」 | 小红 | personality_input.json |
| 「你喝茶救星球吗？我给你们点，喝点东西舒服些」 | 小红 | personality_input.json |
| 「我觉得你又不胖，不挺好的嘛」 | 小红 | personality_input.json |
| 「有点忧伤 感觉是离别季主题」 | 小明 | partner_input.json |

### 教训

在任意数据源修复后，**所有下游产物都需要重新生成**，不能跳过中间步骤。具体来说：
1. 修复 `is_sender` → 需重新 `export_contact.py`
2. 重新导出 → 需重新 `main.py`（生成 input JSON）
3. 新的 input JSON → 需重新分析（生成 result JSON）
4. 新的 result JSON → 需重新 `main.py --personality-result`（生成 report）

---

## 对安装指南的更新建议

`安装指南.md` 需要更新以下几点：

1. **密钥提取**：WeChat 4.1.x 不再适用 `find_key.py`（lldb 方式），需要补充 C 内存扫描器的使用说明
2. **SIP**：扫描器需要 `sudo` 运行，或对微信做 ad-hoc 重签名（`sudo codesign --force --deep --sign - /Applications/WeChat.app`）
3. **常见问题**：增加 `setCipherKey function not found` 的说明和解决方案
