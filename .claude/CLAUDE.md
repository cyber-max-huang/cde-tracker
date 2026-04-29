## 项目: CDE 受理品种目录数据采集

### 入口
- `scrape_cde.py` — 一键运行，自动发现 chromedriver + 提取+保存
- `cde_data.json` — 结构化输出（10条/页，8列）
- `README.md` — 完整文档（含原理、踩坑、故障排查）

### 关键技术事实
- CDE 有 **网宿 CloudWAF** 防护——headless 一律返回空页面
- 非 headless Chrome 是唯一可行方案（undetected-chromedriver 也失败）
- 数据在 `tbody#acceptVarietyInfoTbody > tr > td`，LayUI 模板引擎渲染
- **必须用 JS 枚举 <td> 提取，不能解析 body text**（字段间距不固定）
- 脚本兼容 Python 3.9+（不能使用 Python 3.10+ 的 `X | None` 语法）
- chromedriver 自动发现：环境变量 → PATH → ~/.wdm/ → Homebrew
- 当前环境: Python 3.9.6, Chrome 147, Selenium 4.36.0
