---
name: browser-agent
description: Linux无图形界面下的浏览器自动化框架 — 支持动态网页抓取、反爬应对、内容提取、任务队列
version: 1.0
created: 2026-04-26
tags: [浏览器自动化, 网页抓取, Playwright, 无头浏览器, 数据采集]
platform: Linux (headless)
dependencies: [playwright, python3]
---

# Browser Agent Framework

> 适用：Linux 无图形界面环境（命令行/服务器）
> 依赖：Playwright + Chromium Headless
> 用途：网页抓取、内容提取、动态页面采集、自动化操作

---

## 架构概览

```
browser_agent/
├── SKILL.md                    # 本文件
├── scripts/
│   └── browser_agent.py        # 单文件，包含所有模块
└── references/
    └── examples.md             # 使用示例
```

> ⚠️ **重要**：所有类（BrowserAgent、AntiDetection、TaskQueue、Storage）都在 `scripts/browser_agent.py` 同一个文件内，导入方式：
> ```python
> from scripts.browser_agent import BrowserAgent, AntiDetection, TaskQueue, Storage, quick_fetch
> ```

---

## 核心能力

| 能力 | 说明 |
|------|------|
| **动态网页抓取** | 支持JS渲染的页面（React/Vue/Angular） |
| **内容提取** | 文本/HTML/截图/表格/JSON |
| **反爬应对** | 多层反检测策略 |
| **任务管理** | 批量URL队列、自动重试 |
| **会话持久化** | Cookie保存、上下文复用 |
| **多浏览器** | Chromium/Firefox/WebKit |

---

## 环境要求

### 1. 安装 Playwright

```bash
# 安装（如果还没装）
uv pip install playwright
python3 -m playwright install chromium
```

### 2. 验证环境

```bash
python3 -c "
from playwright.sync_api import sync_playwright
with sync_playwright() as p:
    browser = p.chromium.launch(args=['--no-sandbox', '--disable-dev-shm-usage'])
    page = browser.new_page()
    page.goto('https://example.com')
    print('Browser OK:', page.title())
    browser.close()
"
```

---

## 快速开始

### 基本用法

```python
from scripts.browser_agent import BrowserAgent

agent = BrowserAgent()
agent.launch()

# 抓取页面
result = agent.get_page('https://www.douyin.com/rule/main')
print(result['title'])
print(result['text'][:500])

agent.close()
```

### 带反爬策略

```python
agent = BrowserAgent(
    headless=True,
    anti_detection=True,  # 启用反爬
    retry=3,              # 重试3次
    timeout=20000         # 超时20秒
)
```

### 批量抓取

```python
urls = [
    'https://www.douyin.com/rule/main',
    'https://www.kuaishou.com',
    'https://www.xiaohongshu.com/rule',
]

results = agent.batch_get(urls)
for url, result in results.items():
    print(f"{url}: {result['status']}")
```

---

## 核心API

### BrowserAgent

| 方法 | 参数 | 返回 | 说明 |
|------|------|------|------|
| `launch()` | `headless=True` | - | 启动浏览器 |
| `close()` | - | - | 关闭浏览器 |
| `get_page()` | `url, timeout, wait_until` | dict | 获取单页 |
| `batch_get()` | `urls, callback` | dict | 批量获取 |
| `screenshot()` | `url, path` | str | 截图 |
| `extract_tables()` | `url` | list | 提取表格 |
| `click_and_wait()` | `selector, url` | dict | 点击跳转 |
| `fill_form()` | `selectors_dict` | dict | 填写表单 |
| `save_cookies()` | `path` | - | 保存Cookie |
| `load_cookies()` | `path` | - | 加载Cookie |

### PageResult

```python
{
    'url': str,           # 最终URL（可能有跳转）
    'status': int,       # HTTP状态码
    'title': str,        # 页面标题
    'text': str,         # 纯文本内容
    'html': str,         # 原始HTML
    'screenshot': str,    # 截图路径（如果请求了）
    'cookies': dict,     # 当前Cookie
}
```

---

## 反爬策略

### AntiDetection 配置类

可以通过 `AntiDetection` 类自定义反爬配置，再传入 BrowserAgent：

```python
from scripts.browser_agent import BrowserAgent, AntiDetection

ad = AntiDetection(
    enabled=True,              # 是否启用反爬
    user_agent_pool=[...],      # UA列表，默认5个常用UA
    random_delay=(0.5, 2.0),   # 延迟范围元组（秒）
    proxy=None                  # 代理地址，如 'http://proxy:8080'
)

# 获取配置信息
ad.get_random_ua()       # 随机返回一个UA
ad.get_random_delay()    # 从random_delay范围返回一个随机秒数
ad.get_launch_args()     # 返回Chromium启动参数列表

agent = BrowserAgent(anti_detection=ad)
```

### 手动启用

```python
agent = BrowserAgent(
    anti_detection=True,  # 使用默认反爬配置
    # 或传入 AntiDetection 实例自定义
    proxy='http://proxy:8080'  # 可选代理
)
```

---

## 内容提取

### 提取文本

```python
result = agent.get_page(url)
text = result['text']

# 或者精确提取
text = agent.extract_text(url, selector='.article-content')
```

### 提取表格

```python
tables = agent.extract_tables(url)
for table in tables:
    print(table['headers'])
    for row in table['rows']:
        print(row)
```

### 提取JSON

```python
data = agent.extract_json(url, selector='#__NEXT_DATA__')
```

---

## 任务队列

### 批量处理

```python
from scripts.browser_agent import TaskQueue

queue = TaskQueue(concurrency=3)  # 最多3个并发

def handler(result):
    print(f"完成: {result['url']}")

queue.process(urls, callback=handler)
```

### 队列管理

```python
queue.add(url)
queue.add(urls)
queue.status()  # 返回 {'total', 'completed', 'pending', 'paused'}
queue.pause()
queue.resume()
```

---

## 存储模块

### 保存结果

```python
from scripts.browser_agent import Storage

storage = Storage('./data')
storage.save('douyin_rules', result)
```

### 加载/查询

```python
data = storage.load('douyin_rules')
exists = storage.exists('douyin_rules')
```

---

## 使用示例

### 抓取抖音规则中心

```python
from scripts.browser_agent import BrowserAgent

agent = BrowserAgent(anti_detection=True)
agent.launch()

result = agent.get_page(
    'https://www.douyin.com/rule/main',
    wait_time=3000  # 等待JS渲染3秒
)

print(f"标题: {result['title']}")
print(f"状态: {result['status']}")
print(f"内容: {result['text'][:1000]}")

agent.close()
```

### 批量抓取多平台规则

```python
from scripts.browser_agent import BrowserAgent
from scripts.task_queue import TaskQueue

agent = BrowserAgent(anti_detection=True)
agent.launch()

urls = {
    '抖音': 'https://www.douyin.com/rule/main',
    '快手': 'https://www.kuaishou.com',
    '小红书': 'https://www.xiaohongshu.com/rule',
}

results = agent.batch_get(list(urls.values()), concurrency=2)

for name, url in urls.items():
    r = results[url]
    print(f"{name}: {r['status']} - {r['title']}")

agent.close()
```

---

## 故障排除

### 浏览器启动失败

```bash
# 检查依赖
python3 -m playwright install-deps chromium

# 或者用系统Chromium
export PLAYWRIGHT_CHROMIUM_EXECUTABLE_PATH=/usr/bin/chromium
```

### 页面加载超时

```python
agent = BrowserAgent()
agent.get_page(url, timeout=30000)  # 增加到30秒
```

### 内容为空

```python
# 增加等待时间
result = agent.get_page(url, wait_time=5000)  # 5秒
```

### 被反爬拦截

```python
agent = BrowserAgent(
    anti_detection=True,
    proxy='http://proxy:8080'  # 使用代理
)
```

---

## 定时任务

### 自动更新规则（每月）

```bash
# 写入crontab
0 0 1 * * /path/to/scripts/update_rules.py
```

---

## 平台抓取经验

### 已验证的平台规则页面

| 平台 | 规则URL | 可访问性 | 说明 |
|------|---------|----------|------|
| 抖音 | https://www.douyin.com/rule/main | ✅ | JS渲染页面，正常抓取 |
| 快手 | https://www.kuaishou.com/norm | ✅ | tab参数：community/live/userInfo/comment/liveCover |
| 小红书 | 多个路径尝试 | ❌ | **IP被标记为风险**(300012)，需Cookie或扫码 |
| B站 | https://www.bilibili.com/protocal/licence.html | ✅ | 用户协议完整；/blackboard/rules.html **返回404已下线** |
| B站 | https://member.bilibili.com/studio/convention | ⚠️ | 社区公约仅目录，内容需登录 |
| 视频号 | https://weixin.qq.com/cgi-bin/readtemplate?t=weixin_agreement&s=video | ✅ | 视频号运营规范完整 |

### 平台反爬经验

| 平台 | 反爬机制 | 应对策略 |
|------|----------|----------|
| 小红书 | IP风险判定(300012)、强制App跳转 | 使用有效Cookie；移动端UA可能绕过 |
| B站 | 规则页面下线(404)；部分超时 | 使用替代页面（用户协议/帮助页） |
| 视频号 | 部分端点返回405 | 使用GET而非POST；通过/platform/rules找链接 |
| 快手 | /rules路径302重定向 | 使用/norm路径 |
| 抖音 | JS渲染 | Playwright正常支持，等待networkidle |

---

## 下一步

详见 `references/examples.md` 中的完整示例。

---

*Framework v1.0 — 2026-04-26*
