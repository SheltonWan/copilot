---
description: "项目级 Agent 指令 — 文档创建后自动转换为 PDF"
---

# 项目指令

## KnowFlow 公开发布

当在 `docs/` 目录下创建或修改 Markdown（`.md`）文件时，在完成“文档自动转 PDF”规则并确认 PDF 已同步到 iCloud 后，必须调用 KnowFlow CLI 将该文档公开发布。

### 规则

1. **执行顺序**：必须先完成 Markdown 转 PDF 和 PDF iCloud 同步，确认成功后再发布；任一步失败都不得继续发布。
2. **推荐命令**：使用已安装的全局 `knowflow` CLI：
      ```bash
      knowflow publish docs/<subdir>/<filename>.md --title "<文档标题>" --tags "<标签1>,<标签2>"
      ```
3. **CLI 配置**：确保 `KNOWFLOW_SITE` 和 `KNOWFLOW_API_KEY` 已通过环境变量提供；如项目存在 `knowflow/.env.myknowflow`，发布前使用 `set -a && source knowflow/.env.myknowflow && set +a` 加载。禁止在提示词、命令输出、文档或提交文件中暴露 API Key。
4. **可选参数**：可根据文档内容使用 `--space <slug>`、`--tags <list>`、`--title <title>`、`--json`、`--quiet` 或 `--verbose`；API Key 不得作为命令行参数传入。
5. **回退方式**：CLI 不可用时，停止发布并报告缺少 CLI 或配置，不要自行改用未验证的 HTTP 请求。
      ```bash
      set -a && source knowflow/.env.myknowflow && set +a
      knowflow publish docs/<subdir>/<filename>.md --title "<文档标题>" --tags "<标签1>,<标签2>"
      ```
6. **发布结果**：发布成功后返回 KnowFlow 公开文章的绝对 URL；发布失败时保留错误信息并停止，不绕过安全扫描或强行重试。

## 文档自动转 PDF

当在 `docs/` 目录下创建或修改任何 Markdown（`.md`）文件后，**必须** 自动调用 `~/aitools/md_to_pdf.sh` 将其转换为 PDF；PDF 创建或更新后，还必须自动复制到 `~/Library/Mobile Documents/com~apple~CloudDocs/copilot/pdf`。

### 规则

1. **触发条件**：在 `docs/` 目录下新建或编辑 `.md` 文件。
2. **执行命令**：
   ```bash
   bash ~/aitools/md_to_pdf.sh docs/<subdir>/<filename>.md
   ```
3. **输出位置**：PDF 会自动生成到 `pdfdocs/` 目录下，保持与 `docs/` 相同的子目录结构。
4. **行为**：每次创建/修改文档后，应在同一轮对话中立即执行转换，无需用户额外提醒。
5. **PDF 同步（中文标题命名）**：PDF 转换成功后，确保目标目录存在，并将对应的 PDF 复制到 iCloud 目录；**目标文件名取源 Markdown 文档的第一个 `# ` 标题（中文标题）**，允许覆盖同名文件。
   ```bash
   # 从源 md 提取第一个 H1 标题作为目标文件名
   TITLE="$(grep -m1 '^# ' docs/<subdir>/<filename>.md | sed 's/^# *//')"
   TITLE="${TITLE//\//-}"   # 文件名不能含 /，替换为 -
   mkdir -p "$HOME/Library/Mobile Documents/com~apple~CloudDocs/copilot/pdf"
   cp -f pdfdocs/<subdir>/<filename>.pdf \
         "$HOME/Library/Mobile Documents/com~apple~CloudDocs/copilot/pdf/${TITLE}.pdf"
   ```

### 示例

```bash
# 创建 docs/backend/report.md 后：
bash ~/aitools/md_to_pdf.sh docs/backend/report.md
# → 生成 pdfdocs/backend/report.pdf
# 假设 report.md 第一个标题为 "# 中文报告标题"
TITLE="$(grep -m1 '^# ' docs/backend/report.md | sed 's/^# *//')"
mkdir -p "$HOME/Library/Mobile Documents/com~apple~CloudDocs/copilot/pdf"
cp -f pdfdocs/backend/report.pdf \
      "$HOME/Library/Mobile Documents/com~apple~CloudDocs/copilot/pdf/${TITLE}.pdf"
# → 同步为 iCloud 目录下的「中文报告标题.pdf」，并覆盖同名文件
```
