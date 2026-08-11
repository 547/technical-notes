# rms-log-decrypt 技能更新学习记录

时间: 2026-07-24 13:55 (GMT+8)
技能路径: ~/Documents/work/codes/skills/rms-log-decrypt

## 本次更新 (v2.4.0, 2026-07-24)

来源核对: CHANGELOG.md + git HEAD (1438ea6) + src/decryptor.py:181 实测

### 核心修复: 非 UTF-8 明文误报解密失败
- 根因: AESCBCDecryptor.decrypt() 原用 data.decode(encoding) 默认 errors='strict'，明文夹非法 UTF-8 字节时抛 UnicodeDecodeError → 被 except 吞掉 → decrypt() 返回 None → 误判"解密失败"并丢弃已解出的合法 JSON。IM 日志混非文本字节易触发。
- 修复: 改为 data.decode(encoding, errors='replace')，坏字节替换为 U+FFFD，其余合法部分照常返回。
- 代码实测: src/decryptor.py:181 已为 errors='replace' ✅
- 回归: 新增 21 用例，真实样本在 references/sample-nonutf8-ciphertext.txt (锚点 https://rms.gree.com/uaa/captcha/verify，即此前解密的 captcha/verify 日志)

### 元数据瑕疵
- SKILL.md frontmatter version 仍为 2.3.0，未 bump 到 2.4.0（文档小瑕疵，行为变更属实）

## 未变更 (v2.3.0 规则继续有效)
- 解密结果一律写临时文件作附件，绝不内联 (避免 IM 显示层截中间)
- 路径走 stderr，agent 读后当附件发、发完 rm
- 完整返回、JSON 缩进 2 空格、禁止截断/省略、响应不废话

## 之前会话已确认的事实
- 聊天显示层会间歇截断超长无空格串 (如长 JWT)，数据源始终完整
- 取数权威方式: 临时文件附件 / 工作区文件，而非内联长结果
