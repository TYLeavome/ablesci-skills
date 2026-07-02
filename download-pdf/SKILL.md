---
name: download-pdf
description: 用于根据提供的 DOI 号下载文献, 优先使用本地脚本下载文献 PDF; 若失败, 则自动通过科研通求助并下载 PDF
---

## Skill 目标
接收一个或多个 DOI, 按如下顺序完成 PDF 获取:
1. 先调用 `PDF_downloader.py` 尝试下载 PDF。
2. 若下载失败, 且在输出目录 `PDFs` 下生成了 `Failed.txt`, 并且其中包含目标 DOI, 则改为使用科研通网站的"发布文献求助"功能下载。
3. 下载成功后, 将 PDF 从 `PDFs/` 目录移动到 `~/Documents/PDFs/` 目录。
4. 清理 `Failed.txt` 中对应 DOI 的失败记录, 并将清理后的 `Failed.txt` 也移动到 `~/Documents/PDFs/` 目录。
5. 向用户发送结果通知。

## 严格边界
- 仅执行与 DOI 文献 PDF 下载直接相关的操作。
- 不要使用科研通账号做任何与文献求助下载无关的事情。
- **PDFs 目录仅作为本地脚本的临时中转，下载完成后必须将 PDF 移动到 ~/Documents/PDFs/，不得在 PDFs 目录留存任何 PDF 文件。**
- 不要修改除以下内容之外的任何文件:
  - `PDFs/Failed.txt` 中与当前 DOI 对应的失败记录
- 不要批量清空 `Failed.txt`。
- 仅删除当前成功补救下载的 DOI 对应记录, 包括:
  - 该 DOI 所在行
  - 其下方一行异常信息
  - 与该条记录关联的换行符
- 如果无法确认某条 `Failed.txt` 记录是否属于当前 DOI, 不要删除该记录。

## 输入
- `doi`: 用户提供的 DOI 字符串
- 可选: `download_root` 或项目工作目录
- 可选: `PDF_downloader.py` 脚本路径
- 可选: "启用OpenClaw Chrome Extension" python 脚本路径

## 输出
- 成功时: 告知用户该文献已经下载成功，文件保存在 `~/Documents/PDFs/` 目录
- 超过 24 小时仍未获取到文献时: 告知用户科研通已自动关闭该文献求助
- 失败时: 说明失败发生在哪一步, 并尽可能保留现场信息, 但不要执行无关操作

## 执行原则
- 优先使用本地脚本下载, 只有在本地脚本明确失败时才使用科研通求助。
- 在网页操作中, 尽量等待页面元素真实出现后再点击, 避免固定死等。
- 若页面弹出提示框, 优先读取提示内容并据此判断下一步。
- 若下载动作会打开新的 Chrome 标签页, 仅关闭新标签页, 不要关闭整个浏览器窗口。
- 若已有文献很快获取到, 跳转详情页后可能立即弹窗, 属于正常情况。
- 若 24 小时以上未获取到文献, 视为科研通自动关闭求助, 必须通知用户。

## 关键判定逻辑
### A. 本地脚本下载成功判定
满足以下任一条件, 可视为本地脚本下载成功:
- 输出目录中出现与目标 DOI 对应的新 PDF 文件, 且文件大小大于 0
- 或脚本标准输出/日志明确显示下载成功

### B. 本地脚本下载失败判定
当同时满足以下条件时, 视为本地脚本下载失败, 进入科研通流程:
- `PDFs/Failed.txt` 存在
- `Failed.txt` 中包含目标 DOI

### C. 科研通获取成功判定
满足以下任一情形即可进入下载和采纳流程:
- 跳转详情页后立即出现弹窗: "恭喜您, 已经有人上传了文件, 请在 48 小时内查看并审核。"
- 详情页中出现"待审核"区域, 且其右侧出现可点击的文献下载链接, 通常显示为"文献名称 + 年份"

### D. 科研通超时关闭判定
- 自发布求助起超过 24 小时仍未获取到文献
- 或页面明确显示求助已关闭/自动关闭/超时关闭等语义

出现该情况时, 发送消息通知用户: 该 DOI 对应文献超过 24 小时未获取到, 科研通已自动关闭该文献求助。

## 推荐执行流程

### 第 1 步: 预处理
1. 接收 DOI。
2. 标准化 DOI 字符串:
   - 去除首尾空白
   - 保留 DOI 原始大小写展示, 但匹配时允许大小写不敏感
3. 确认以下路径可用:
   - `PDF_downloader.py`
   - `PDFs/`
   - "启用OpenClaw Chrome Extension" python 脚本

### 第 2 步: 调用 `PDF_downloader.py`
1. 按 `PDF_downloader.py` 脚本内部说明调用脚本。
2. 调用后观察:
   - 返回码
   - 控制台输出
   - `PDFs/` 目录变化
   - `PDFs/Failed.txt` 是否出现并包含该 DOI
3. 若脚本成功下载 PDF:
   - 将 PDF 从 `PDFs/` 目录移动到 `~/Documents/PDFs/` 目录
   - 直接向用户发送消息: 文献已经下载成功
   - 结束任务
4. 若脚本失败且 `Failed.txt` 中包含该 DOI:
   - 进入科研通求助流程

### 第 2.5 步: 移动 PDF 到 ~/Documents/PDFs 目录（本地脚本下载成功后执行）
本地脚本 `PDF_downloader.py` 会将 PDF 下载到 `PDFs/` 子目录，需要移动到 `~/Documents/PDFs/` 目录。**`PDFs` 目录仅作为临时中转，下载完成后必须将 PDF 移动到 ~/Documents/PDFs/，不得在 PDFs 目录留存任何 PDF 文件。**

```python
import shutil
import os

pdfs_dir = '/Users/tianye/.openclaw/scripts/PDFs'
doi = '10.xxxx/xxxx'  # 当前 DOI
dest_dir = os.path.expanduser('~/Documents/PDFs')

# 确保目标目录存在
os.makedirs(dest_dir, exist_ok=True)

# 查找对应的 PDF 文件并移动
pattern = doi.replace('/', '_') + '.pdf'
pdf_path = os.path.join(pdfs_dir, pattern)

if os.path.exists(pdf_path):
    shutil.move(pdf_path, os.path.join(dest_dir, pattern))
```

**注意**：
- DOI 中的 `/` 会被替换为 `_` 作为文件名
- 目标路径为 `~/Documents/PDFs/`
- **PDFs 目录仅作中转，任务结束后必须清空，不得留存任何 PDF 文件**

### 第 3 步: 打开科研通并接管浏览器
1. 启动浏览器并进入: `https://www.ablesci.com/`
2. 运行"启用OpenClaw Chrome Extension" python 脚本接管浏览器。
3. 确认浏览器已可由 OpenClaw 控制。
4. 若页面未登录且当前环境具备已保存登录态, 则继续使用现有登录态。
5. 若没有登录态, 则按照环境中预配置的合法方式登录; 不要擅自修改账号设置。

### 第 4 步: 发布文献求助
1. 在网页右上角点击"发布文献求助"按钮。
2. 在"一键求助"右侧的输入框内输入 DOI。
3. 点击"智能提取文献信息"。
4. 等待网页内弹出浮窗。
5. 在浮窗中点击"信息正确, 直接发布"。
6. 弹窗内容更新后, 点击"查看求助详情"。
7. 页面跳转到该求助详情页。

## 第 5 步: 处理详情页弹窗与等待结果
1. 进入详情页后, 立即检查是否出现浮窗:
   - "恭喜您, 已经有人上传了文件, 请在 48 小时内查看并审核。"
2. 若出现该弹窗:
   - 点击"确定"
   - 继续执行下载流程
3. 若未立即出现该弹窗:
   - 继续在详情页检查是否已出现"待审核"区域和下载链接
4. 若暂未获取到文献:
   - 在允许的执行机制内持续轮询或等待状态变化
   - 重点观察是否出现以下结果之一:
     - 已出现下载链接
     - 页面提示求助已关闭
     - 超过 24 小时未获取文献
5. 若超过 24 小时仍未获取到文献:
   - 向用户发送消息, 告知科研通已自动关闭该文献求助
   - 结束任务

## 第 6 步: 下载 PDF
1. 定位"待审核"文本框右侧的文献下载链接。
2. 该链接通常显示为文献名称和年份。
3. 点击该下载链接，Chrome 会自动开始下载 PDF 到 `~/Downloads/` 目录。
4. **不要自己构造或访问下载链接 URL**，只需点击链接让 Chrome 处理下载。
5. 下载文件名通常为 `文献标题(科研通-ablesci.com).pdf` 格式。
6. 下载完成后:
   - 关闭新打开的标签页（如果有）
   - 返回求助详情页
7. 将下载的 PDF 移动到 `~/Documents/PDFs/` 并重命名:
   ```python
   import shutil
   import os
   import glob

   doi = '10.xxxx/xxxx'  # 当前 DOI
   downloads = os.path.expanduser('~/Downloads')
   dest_dir = os.path.expanduser('~/Documents/PDFs')

   # 查找刚下载的 PDF（通常是最新修改的）
   pattern = '*(*ablesci.com*).pdf'
   pdfs = glob.glob(os.path.join(downloads, pattern))
   latest_pdf = max(pdfs, key=os.path.getmtime)

   # 目标文件名：DOI 中的 / 替换为 _
   dest_name = doi.replace('/', '_') + '.pdf'
   shutil.move(latest_pdf, os.path.join(dest_dir, dest_name))
   ```
8. **注意**：Chrome 下载目录中的文件名是 `文献标题(科研通-ablesci.com).pdf`，需要重命名为 DOI 格式。

## 第 7 步: 采纳文件并关闭求助
1. 在求助详情页点击"采纳文件"按钮。
2. 在弹出的浮窗中点击"确定"。
3. 若随后又出现页面内弹窗, 内容类似:
   - "感谢使用科研通, 请帮忙推荐该网站"
   则点击"确定"关闭该浮窗。

## 第 8 步: 清理 `Failed.txt` 中对应失败记录
仅在科研通下载成功并完成采纳后执行。

### 清理规则
在 `PDFs/Failed.txt` 中查找当前 DOI 对应记录, 删除以下内容:
1. DOI 所在行
2. 其下方一行异常信息
3. 与这两行关联的多余换行符

### 注意事项
- 只删除当前 DOI 对应记录, 不删除其他 DOI 记录。
- DOI 匹配建议使用大小写不敏感比对。
- 删除后保证文件仍为合法文本格式。
- 若该 DOI 在 `Failed.txt` 中不存在, 则无需报错, 直接继续。

## 第 9 步: 结果通知
### 成功通知
当科研通下载成功, 且已完成采纳及 `Failed.txt` 清理后, 发送消息给用户:
- 文献已经下载成功

### 超时通知
若超过 24 小时仍未获取到文献, 发送消息给用户:
- 该 DOI 对应文献超过 24 小时未获取到, 科研通已自动关闭该文献求助

### 异常通知
若流程异常中断, 发送消息说明:
- DOI
- 当前执行到的步骤
- 失败原因概述
- 是否已保留 `Failed.txt` 原记录

## 建议的鲁棒性要求
- 点击前先检查元素是否可见、可点击。
- 对按钮文本允许轻微变体匹配, 但必须语义一致, 例如:
  - "发布文献求助"
  - "智能提取文献信息"
  - "信息正确, 直接发布"
  - "查看求助详情"
  - "采纳文件"
- 对成功弹窗内容允许相近文本匹配, 但核心语义必须包含:
  - 已有人上传文件
  - 48 小时内查看并审核
- 对推荐网站类弹窗允许模糊匹配后关闭。
- 若页面结构变化, 优先按文本语义寻找元素, 其次再考虑 CSS/XPath 定位。

## 建议的消息模板
### 模板 1: 本地脚本直接成功
`DOI: {doi}\n已通过本地脚本成功下载 PDF。`

### 模板 2: 科研通补救成功
`DOI: {doi}\n文献已经下载成功, 并已完成科研通求助关闭及 Failed.txt 清理。`

### 模板 3: 超时关闭
`DOI: {doi}\n超过 24 小时仍未获取到文献, 科研通已自动关闭该文献求助。`

### 模板 4: 异常中断
`DOI: {doi}\n下载流程在步骤 {step_name} 处中断。错误概述: {error_summary}。当前未删除 Failed.txt 中原始失败记录。`

## OpenClaw 可直接参考的执行指令
你是一个只负责根据 DOI 下载文献 PDF 的自动化代理。请严格按以下顺序执行:

1. 优先调用 `PDF_downloader.py` 下载目标 DOI 的 PDF, 调用方法以该脚本内说明为准。
2. 若下载成功, 将 PDF 从 `PDFs/` 移动到 `~/Documents/PDFs/` 目录。
3. 若下载失败, 且 `PDFs/Failed.txt` 存在并包含该 DOI, 则打开 `https://www.ablesci.com/`。
4. 使用"启用OpenClaw Chrome Extension" python 脚本接管浏览器。
5. 点击右上角"发布文献求助"。
6. 输入 DOI 后点击"智能提取文献信息"。
7. 在弹窗中点击"信息正确, 直接发布"。
8. 弹窗更新后点击"查看求助详情"。
9. 若跳转后立即弹出"恭喜您, 已经有人上传了文件, 请在 48 小时内查看并审核。"则点击"确定"。
10. 在"待审核"右侧找到文献下载链接并点击, 等待 PDF 下载完成。
11. 关闭下载产生的新标签页, 回到详情页。
12. 点击"采纳文件", 并在确认弹窗中点击"确定"。
13. 如果随后出现"感谢使用科研通, 请帮忙推荐该网站"等类似弹窗, 点击"确定"关闭。
14. 从 `PDFs/Failed.txt` 中删除当前 DOI 对应的整条失败记录: DOI 行 + 下方异常信息行 + 关联换行符, 并将清理后的 Failed.txt 也移动到 `~/Documents/PDFs/` 目录。
15. 向用户发送成功消息。
16. 若超过 24 小时仍未获取到文献, 向用户发送"科研通已自动关闭该文献求助"的消息。
17. 除以上操作外, 不要执行任何无关操作。

## 可选附录: `Failed.txt` 清理伪代码
```python
from pathlib import Path
import re

failed_path = Path('PDFs/Failed.txt')
doi = target_doi.strip()

if failed_path.exists():
    lines = failed_path.read_text(encoding='utf-8', errors='ignore').splitlines(keepends=True)
    i = 0
    new_lines = []
    while i < len(lines):
        current = lines[i]
        if doi.lower() in current.lower():
            i += 1
            if i < len(lines):
                i += 1
            while i < len(lines) and lines[i].strip() == '':
                i += 1
            continue
        new_lines.append(current)
        i += 1
    failed_path.write_text(''.join(new_lines), encoding='utf-8')
```

## 浏览器自动化踩坑经验总结（2026-03-21）

### 1. Chrome Relay Tab 不稳定问题

**现象**：执行 `click` 等操作后，tab 进入 "tab not found" 状态，后续所有操作都失败。

**原因**：点击链接触发 SPA 路由切换后，OpenClaw Chrome Relay 的 targetId 虽然保留，但底层 CDP 连接丢失。

**对策**：
- 执行关键操作（点击按钮/输入）后，立即用 `tabs` 命令确认 tab 是否还活着
- 若返回 "tab not found"，立即用 `browser open` 创建新 tab，再重新操作
- 不要在单一 tab 上连续做多步操作，每步之间加 `tabs` 检查

### 2. 页面 URL 与实际状态不同步

**现象**：执行 `evaluate(() => { link.click() })` 后 URL 变了，但 tab title 仍为空，snapshot 取到的 DOM 是旧页面的。

**原因**：SPA 页面点击后 URL 变化是异步的，CDP 快照可能拿到的是路由切换前的 DOM。

**对策**：
- 已知路由（如发布页 `https://www.ablesci.com/assist/create`）直接用 `navigate` 打开，不要通过点击链接跳转
- JavaScript `click()` 触发导航后，用 `navigate` 重新加载目标 URL 确保状态同步

### 3. ARIA 快照 ref 失效

**现象**：拿到 snapshot 后，用 `axXX` ref 做 `act` 操作报 `TimeoutError`。

**原因**：SPA 页面任何操作（甚至只是等待）都可能导致 DOM 结构变化，ref 指向的节点已不存在。

**对策**：
- 每次需要操作前，都重新做一次 `snapshot` 获取最新 ref
- 不要跨操作使用同一个 ref

### 4. evaluate + setTimeout 的无效性

**现象**：`evaluate(() => { setTimeout(() => { /* 操作 */ }, 2000); return 'done'; })` 返回 'done' 后，setTimeout 里的操作已无法获取返回值。

**原因**：evaluate 函数立即返回，后续的 setTimeout 回调拿不到 CDP 执行上下文。

**对策**：不要在 evaluate 里用 setTimeout 做等待，应该在外部用 `timeoutMs` 参数控制等待节奏。

### 5. 表单输入的正确方法

**现象**：`act kind=fill` / `act kind=type` 报 `fields are required` 或 `TimeoutError`。

**原因**：OpenClaw browser 的 `fill`/`type`/`click` 依赖 aria-ref 或 selector，但 ref 总是失效。

**正确方法**：使用 `act kind=evaluate` 执行 JavaScript：

```javascript
// 通过已知 ID 填写表单（最可靠）
const input = document.getElementById('onekey');
input.focus();
input.value = '10.1111/1541-4337.12869';
input.dispatchEvent(new Event('input', { bubbles: true }));
input.dispatchEvent(new Event('change', { bubbles: true }));
return 'success: ' + input.value;
```

**找到表单元素的方法**：
1. `evaluate(() => { const inputs = document.querySelectorAll('input'); ... })` 列出所有 input
2. 识别 `id` 属性（如 `onekey`、`Assist-doi`）
3. 后续用 `document.getElementById(id)` 操作

### 6. 科研通网站关键元素 ID 速查

| 元素 | ID |
|------|-----|
| 一键求助输入框 | `onekey` |
| DOI 输入框 | `Assist-doi` |
| 标题输入框 | `Assist-title` |
| 文献链接输入框 | `Assist-url` |
| 悬赏积分输入框 | `Assist-point` |
| 备注输入框 | `Assist-remark` |
| 自动关闭时间 | `Assist-close_at` |

### 7. 多 tab 管理策略

**现象**：调试过程中累积大量无用 tab，目标 tab 越来越难找。

**策略**：
- 操作完成后随手关闭不用的 tab（`browser close`）
- 每次重新开始时只保留一个干净的 tab
- 重大操作前用 `browser stop` + `browser start` 重置 browser 状态

### 8. `browser open` vs `browser navigate`

- `browser open`：创建**新** tab，返回新 targetId
- `browser navigate`：在**当前** tab 中加载 URL（若 tab 已死则报错）
- 路由已知时用 `navigate`，不确定时用 `open`

### 9. 点击链接的推荐写法

```javascript
// 通过文本内容查找并点击链接（比 aria-ref 稳定）
const links = document.querySelectorAll('a');
for (const l of links) {
  if (l.textContent.trim() === '发布文献求助') {
    l.click();
    return 'clicked';
  }
}
```

### 10. 执行节奏建议

1. **获取页面**：`navigate` 或 `open`（直接给 URL，不要通过链接跳转）
2. **操作表单**：用 `evaluate` + `document.getElementById`
3. **点击按钮**：优先 `evaluate` 里的 `click()`，aria-ref act 作为备选
4. **检查结果**：用 `screenshot` 确认，不要只靠返回值判断
5. **状态不对**：立即 `browser stop` + `browser start` 重置

## 科研通已有文献时的下载经验

### 场景描述
当科研通已有用户上传了目标文献（状态为"待审核"）时，下载流程如下：

### 已有文献下载链接的获取方法

当详情页出现"待审核"区域后，可通过 JavaScript 快速定位下载链接：

```javascript
// 方法1：通过 href 中包含 .pdf 的链接
document.querySelector('a[href*=".pdf"]')?.href

// 方法2：通过链接描述"点击下载"
const links = document.querySelectorAll('a');
for (const l of links) {
  if (l.href && (l.href.includes('.pdf') || l.textContent.includes('.pdf'))) {
    return l.href;
  }
}

// 方法3：通过 aria 描述查找
document.querySelector('a[description="点击下载"]')?.href
```

典型返回格式：`https://www.ablesci.com/assist/download?id=ZwWXBN`

### curl 直接下载的局限性

**重要发现**：即使获取到了下载链接 `https://www.ablesci.com/assist/download?id=XXX`，直接用 curl 或 wget 下载会返回 HTML 页面而非 PDF 文件。

**原因**：科研通的文件下载需要有效的登录会话（Cookie/Session），直接 HTTP 请求缺少认证信息。

**解决方案**：
1. **推荐**：在已登录的 Chrome 浏览器中点击下载链接，让浏览器自动处理下载
2. 若需要脚本化处理，需要在请求中携带有效的 Cookie：
   ```bash
   curl -L -o output.pdf "https://www.ablesci.com/assist/download?id=XXX" \
     -H "Cookie: _csrf=xxx; PHPSESSID=xxx" \
     -A "Mozilla/5.0 ..."
   ```
   但这需要先从浏览器中提取 Cookie，实现较复杂。

### 页面状态判断

详情页出现以下状态时，表示已有文件可下载：
- 区域标题为"待审核"
- 存在 `button[name="采纳文件"]` 和 `button[name="驳回文件"]`
- 下载链接文本通常为 `文献标题.pdf`

### 操作流程建议

1. 详情页出现"待审核"后，用 evaluate JavaScript 获取下载链接
2. 在 Chrome 浏览器中点击该链接，浏览器会自动下载
3. 下载完成后切换回求助详情页，执行"采纳文件"操作
4. 注意：不要在详情页通过 aria-ref 直接点击下载链接（容易超时），用 JavaScript 拿到 URL 后让用户手动点击或在新 tab 中打开

### 采纳文件的具体操作步骤

当文件已上传（状态为"待审核"）时的采纳流程：

#### 第1步：点击"采纳文件"按钮

使用 JavaScript 查找并点击：

```javascript
const btns = document.querySelectorAll('button');
for (const b of btns) {
  if (b.textContent.includes('采纳文件')) {
    b.click();
    return 'clicked: ' + b.textContent.trim();
  }
}
```

#### 第2步：处理"确认接受应助吗？"对话框

点击"采纳文件"后会弹出确认对话框，内容为"确认接受应助吗？"，底部有"确定"按钮。

**对话框 HTML 结构：**
```html
<div class="layui-layer layui-layer-dialog">
  <div class="layui-layer-content">确认接受应助吗？</div>
  <div class="layui-layer-btn">
    <a class="layui-layer-btn0">确定</a>
  </div>
</div>
```

**点击方法：**
```javascript
const btn = document.querySelector('.layui-layer-btn0');
if (btn) {
  btn.click();
  return 'clicked btn0';
}
```

#### 第3步：处理推荐网站对话框（如果出现）

采纳确认后，可能还会弹出一个推荐网站的对话框，内容类似"感谢使用科研通，请帮忙推荐该网站"，点击"确定"关闭即可。

#### 第4步：验证采纳结果

成功标志：
- 页面标题从"待确认"变为"已完结"
- 时间线中的状态变为"已采纳"
- "采纳文件"和"驳回文件"按钮消失

**验证方法：**
```javascript
// 检查页面标题
document.title.includes('已完结')

// 或检查时间线中的状态
const headings = document.querySelectorAll('h1, h2, h3, heading');
for (const h of headings) {
  if (h.textContent.includes('已采纳') || h.textContent.includes('已完结')) {
    return 'success';
  }
}
```

### 求助状态变化对照表

| 状态 | 标题显示 | 含义 |
|------|---------|------|
| 待确认 | 【待确认】 | 有人上传了文件，等待求助人审核 |
| 已完结 | 【已完结】 | 求助人已采纳文件，求助成功关闭 |
| 已关闭 | 【已关闭】 | 超过48小时未确认，系统自动关闭 |

### 多弹窗连续处理经验

有时弹窗是连续出现的，需要逐个关闭：

1. **第一个弹窗**："确认接受应助吗？" → 点击"确定"
2. **第二个弹窗**（如有）："感谢使用科研通..." → 点击"确定"

如果第一个弹窗关闭后页面状态已变为"已完结"，说明整个流程已完成，无需继续处理后续弹窗。

### 求助发布后"暂无链接"状态的处理

**场景描述**：发布求助后，页面显示"暂无链接，等待应助者上传"或"现在，等待您上传文献"，表示暂时没有人上传文件。

**状态判断**：
页面表格中"下载"行显示"暂无链接，等待应助者上传"，且无"待审核"区域。

**处理策略**：
1. **立即通知用户**：告诉用户求助已成功发布，正在等待应助，无需持续监控
2. **结束本次任务**：向用户说明当前状态和预计等待时间
3. **不轮询**：科研通通常需要数小时甚至1-2天才能获得文献，不应在此时持续轮询
4. **等待通知**：用户会在有人上传文件时收到通知，届时再处理审核和采纳

**用户通知模板**：
```
DOI: {doi}
科研通求助已成功发布，当前状态：暂无链接，正在等待应助者上传文献。
系统显示还剩约 {N} 天关闭。请耐心等待，有文件上传后系统会通知您。
```

### 重复发布拦截处理

**场景描述**：发布求助时提示"对不起，无法提交！系统检测到您在1小时内发布了相同DOI的文献求助"。

**处理方法**：
1. 弹窗中通常会有"查看"链接，点击获取已有求助的详情页 URL
2. 从弹窗 HTML 中提取链接：
   ```javascript
   const layer = document.querySelector('.layui-layer');
   const links = layer ? layer.querySelectorAll('a') : [];
   for (const l of links) {
     if (l.textContent.includes('查看')) {
       return l.href; // e.g. https://www.ablesci.com/assist/detail?id=79GaGA
     }
   }
   ```
3. 跳转到已有求助详情页检查状态
4. 如果已有求助状态为"求助中"且无文件，继续等待即可

### 长时间等待的监控策略

科研通文献求助的典型时间范围：
- **数分钟内**：热门文献/常见文献可能很快有人上传
- **1-24小时**：普通文献的正常等待时间
- **超过24小时**：可视为等待时间较长，系统可能会自动关闭

**推荐做法**：
- 发布求助后立即结束任务，不要原地等待
- 让用户决定是否继续等待或采取其他方式
- 如果用户后续询问，可再次打开详情页检查状态

## 结束条件
满足以下任一条件即可结束:
- 本地脚本成功下载 PDF 并已通知用户
- 科研通补救下载成功, 已采纳文件, 已清理 `Failed.txt`, 并已通知用户
- 科研通超过 24 小时未获取到文献, 已通知用户
- 流程发生不可恢复错误, 已向用户说明失败位置和原因
- 求助已发布但暂时无文件可下载，已通知用户等待，无需继续监控
