# CDE 受理品种目录 · 数据采集

抓取 [国家药品监督管理局药品审评中心 (CDE)](https://www.cde.org.cn) 的"受理品种目录"首页数据。

## 项目目录

```
cde_info/
├── README.md          # 本文档
├── requirements.txt   # Python 依赖
├── scrape_cde.py      # 采集入口脚本
└── cde_data.json      # 输出 -> 提取的结构化数据
```

## 快速开始

```bash
# 1. 安装依赖
pip3 install selenium

# 2. 确认 chromedriver 可用（与 Chrome 版本匹配）
chromedriver --version

# 3. 运行采集
python3 scrape_cde.py
```

脚本会自动：
- 查找 chromedriver
- 启动非 headless Chrome
- 打开 CDE 页面并等待 WAF 验证通过
- 提取表格数据，保存到 `cde_data.json`

## 输出示例

```json
[
  {
    "序号": "1",
    "受理号": "JYHZ2600102",
    "药品名称": "塞利尼索片",
    "药品类型": "化药",
    "申请类型": "进口再注册",
    "注册分类": "5.1",
    "企业名称": "Karyopharm Therapeutics Inc.;Catalent CTS, LLC;Antengene Corporation Co. , Ltd.;",
    "承办日期": "2026-04-27"
  }
]
```

## 字段说明

| 列名 | 说明 | 示例值 |
|------|------|--------|
| 序号 | 行号 | 1 |
| 受理号 | CDE 受理编号，前缀代表申报类型 | JYHZ2600102 |
| 药品名称 | 药品通用名 | 塞利尼索片 |
| 药品类型 | 化药 / 中药 / 治疗用生物制品等 | 化药 |
| 申请类型 | 进口再注册 / 仿制 / 补充申请等 | 进口再注册 |
| 注册分类 | 注册分类编号 | 5.1 |
| 企业名称 | 申请人 / 生产企业（分号分隔多家） | Karyopharm Therapeutics Inc.;... |
| 承办日期 | 承办日期 (YYYY-MM-DD) | 2026-04-27 |

### 受理号前缀含义

| 前缀 | 含义 |
|------|------|
| `JYHZ` | 进口药品再注册 |
| `CYHS` | 化学药品上市许可申请（仿制） |
| `CYZB` | 中药补充申请 |
| `CYHB` | 化学药品补充申请 |
| `JYSB` | 进口生物制品补充申请 |
| `CXHS` | 化学药品上市许可申请（创新） |
| `CXSS` | 生物制品上市许可申请（创新） |
| `CXSB` | 生物制品补充申请（创新） |

## 环境要求

| 组件 | 版本/说明 |
|------|-----------|
| Python | 3.9 或更高 |
| Selenium | 4.x（`pip3 install selenium`） |
| Chrome | 建议最新稳定版 |
| ChromeDriver | 必须与 Chrome 主版本号严格一致 |
| 操作系统 | macOS / Linux / Windows |

## 配置 chromedriver

脚本自带自动发现逻辑，按以下优先级查找：

1. 环境变量 `CHROMEDRIVER_PATH`
2. 系统 PATH
3. webdriver-manager 缓存 (`~/.wdm/`)
4. Homebrew 安装路径

### 手动指定

```bash
# 方式 1: 环境变量
CHROMEDRIVER_PATH=/path/to/chromedriver python3 scrape_cde.py

# 方式 2: 放到系统 PATH
cp chromedriver /usr/local/bin/
chmod +x /usr/local/bin/chromedriver
```

### Homebrew 安装 (macOS)

```bash
brew install chromedriver
```

### 手动下载

访问 [Chrome for Testing](https://googlechromelabs.github.io/chrome-for-testing/) 下载与 Chrome 版本匹配的 ChromeDriver。

查看 Chrome 版本：

```bash
# macOS
/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome --version

# Linux
google-chrome --version
```

### 版本不匹配

Chrome 版本更新后，ChromeDriver 可能过旧，报错：

> `session not created: This version of ChromeDriver only supports Chrome version XXX`

解决：重新安装 / 下载匹配版本的 ChromeDriver。

## 技术原理

### 页面加载流程

```
用户访问 URL
    │
    ▼
CloudWAF 拦截（HTTP 202）
    │
    ▼
浏览器执行 JS 挑战（加密计算，~1-3s）
    │
    ▼
WAF 验证通过，返回真实页面
    │
    ▼
LayUI 模板引擎渲染表格（异步 AJAX，~1-2s）
    │
    ▼
tbody#acceptVarietyInfoTbody 出现数据行
```

### CDE 页面 DOM 结构

数据表格位于 `div.tableDataBox > table.layui-table.txt-center` 内：

```html
<tbody id="acceptVarietyInfoTbody">
  <tr>
    <td>1</td>
    <td>JYHZ2600102</td>
    <td>塞利尼索片</td>
    <td>化药</td>          <!-- 药品类型 -->
    <td>进口再注册</td>     <!-- 申请类型 -->
    <td>5.1</td>           <!-- 注册分类 -->
    <td>Karyopharm Therapeutics Inc.;...</td>  <!-- 企业名称 -->
    <td>2026-04-27</td>    <!-- 承办日期 -->
  </tr>
  ...
</tbody>
```

### 反爬机制与绕过

CDE 网站由 **网宿 CloudWAF** 防护，具备以下检测能力：

| 检测维度 | 特征 | 绕过方式 |
|----------|------|----------|
| `navigator.webdriver` | Headless Chrome 会暴露此属性 | 添加 `--disable-blink-features=AutomationControlled` |
| `window.chrome` | 检测 Chrome 对象是否完整 | 非 headless 模式自然通过 |
| `window.outerWidth/outerHeight` | 检测窗口尺寸 | `--window-size=1920,1080` |
| Canvas fingerprinting | JS 挑战 + 加密签名验证 | 非 headless Chrome 自然通过 |

**关键原则：不要使用 headless 模式。** 所有 headless 绕过方案（包括 CDP 注入、undetected-chromedriver）在此 WAF 上均失败。只有非 headless Chrome 能稳定通过。

### 数据提取策略

**不要解析 body text。** 页面中表格字段之间间距不固定，正则/分词会错位。正确做法是 JavaScript 枚举 DOM 元素：

```python
# ✅ 正确：从 <td> 逐格提取
records = driver.execute_script("""
    const tbody = document.getElementById('acceptVarietyInfoTbody');
    const trs = tbody.querySelectorAll('tr');
    return Array.from(trs).map(tr =>
        Array.from(tr.querySelectorAll('td')).map(td => td.textContent.trim())
    );
""")

# ❌ 错误：从 body text 正则解析（字段粘连）
text = driver.find_element(By.TAG_NAME, "body").text
# 输出: "1 JYHZ2600102 塞利尼索片 化药 进口再注册 5.1 Karyopharm..." 
# 字段之间用不定长空格分隔，无法可靠切分
```

### 为什么不使用 browser-use CLI

`browser-use` 需要 Python 3.11+。macOS 系统自带的 Python 是 3.9.x，无法安装。如需使用 browser-use，需要先通过 Homebrew 安装 Python 3.11+ 并创建虚拟环境。

## 故障排查

### 问题 1：提取到 0 条数据

可能原因：

- **WAF 验证未通过** — 查看浏览器窗口中是否显示了正常的 CDE 页面（有表格），还是一直白屏/验证码封面
- **网络慢** — 增大 `WAF_WAIT_SECONDS`（默认 30s）
- **页面结构变动** — 查看 `cde_debug.png` 调试截图，检查 tbody 是否存在

### 问题 2：Chrome 窗口没有出现

确认脚本中没有开启 headless 模式。检查启动参数中没有 `--headless`。

### 问题 3：urllib3 NotOpenSSLWarning

非阻塞警告，忽略即可。原因是 macOS 的 LibreSSL 与 urllib3 v2 不完全兼容。

### 问题 4：SSL 错误 connecting to googlechromelabs.github.io

手动下载 ChromeDriver 放到本地目录，通过 `CHROMEDRIVER_PATH` 指定路径，而不是让 webdriver-manager 自动下载。

### 问题 5：Python 3.9 兼容性

脚本已兼容 Python 3.9+（使用 `Optional` 而非 `X | None` 语法）。如在更低版本（3.8 及以下）出现语法错误，请升级 Python。

## 扩展到多页采集

如需抓取全部 6000+ 条记录，可添加分页循环。页面使用 LayUI 分页组件：

```python
# 伪代码，未完整实现
from selenium.webdriver.common.by import By

while True:
    # 提取当前页数据
    records = extract_table_data(driver)
    all_records.extend(records)

    # 查找 '下一页' 按钮
    next_btn = driver.find_element(By.CSS_SELECTOR, ".layui-laypage-next")
    if "layui-laypage-disabled" in next_btn.get_attribute("class"):
        break  # 最后一页
    next_btn.click()
    time.sleep(3)  # 等待 Ajax 渲染
```
