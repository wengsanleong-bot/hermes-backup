---
name: creative-workflow-dashboard
description: 创作流水线可视化控制面板 - 模块化AI视频创作工作流系统
version: 1.0
created: 2026-04-26
last_updated: 2026-04-26
status: functional
---

# 创作流水线控制面板

> 位置: `~/.hermes/skills/creative/creative-workflow-dashboard/`

## 启动方式

**Web面板**（浏览器）:
```bash
python3 ~/.hermes/skills/creative/creative-workflow-dashboard/scripts/server.py
# 访问 http://localhost:8765
```

**终端面板**（TUI，无需浏览器）:
```bash
python3 ~/.hermes/skills/creative/creative-workflow-dashboard/scripts/tui.py
```

## 架构设计

```
用户浏览器/终端
        │
        ▼
┌───────────────────────────┐
│   Web服务器 (Flask :8765) │  ← server.py
│   TUI面板 (curses)        │  ← tui.py
└───────────┬───────────────┘
            │ 读写状态
            ▼
┌───────────────────────────┐
│   状态存储层               │
│   state/workflow_state.json│
│   state/config.json        │
│   module_outputs/*.json   │
└───────────────────────────┘
```

## 模块定义

| ID | 模块 | 功能 | 状态 |
|----|------|------|------|
| A | 内容选题 | 根据用户定位生成3-5个选题选项 | 待接入真实LLM |
| B | 文案创作 | 根据选题生成3个文案方向（带缓存） | 待接入真实LLM |
| C | 合规审查 | 自动检查文案合规风险 | 待接入真实合规规则 |
| D | 视频生成 | 调用视频工具生成视频（poll_op模式） | 待接入真实API |
| E | 语音合成 | 调用语音工具生成配音（poll_op模式） | 待接入真实API |
| F | 内容组装 | 音画合成、字幕、配乐 | 待开发 |
| G | 多平台发布 | 发布到选定的平台 | 待开发 |

## 架构演进路径（参考ComfyUI）

当前实现是**线性流水线**（A→B→C→D→E→F→G），未来可演进为**图驱动（DAG）流水线**：

```
线性（当前）              图驱动（未来）
─────────────            ──────────────
A→B→C→D→E→F→G           A→B
                          ↓↓
                          C→D→E
                          ↓  ↓
                          F←─┘
```

**图驱动的优势**：节点间通过 `[node_id, output_index]` 引用，可任意跳过/替换/并行节点。

## 设计模式（来自ComfyUI源码研究）

### 1. poll_op 异步任务轮询模式（模块D/E必用）

外部API（SuperGrok视频/Runway视频/ElevenLabs语音）调用分两阶段：

```python
# 阶段1：创建任务（立即返回）
task = sync_op(endpoint, data=request)  # → task_id

# 阶段2：轮询直到完成
result = poll_op(
    poll_endpoint,
    status_extractor=lambda r: r.status,      # 从响应提取状态
    completed_statuses=["succeeded", "done"],  # 完成状态列表
    failed_statuses=["failed", "error"],      # 失败状态列表
    estimated_duration=60,                     # 预估秒数（用于进度显示）
)
```

关键特性：
- `estimated_duration`：用户能看到预估剩余时间
- `status_extractor`：灵活适配不同API的响应格式
- `completed_statuses/failed_statuses`：配置化，非硬编码
- 可取消：用户随时能中断长任务

### 2. 三层缓存机制（模块B/C建议用）

```
输入签名缓存 → 判断是否需要重新执行（节省API调用费用）
     ↓
LRU内存缓存 → 常用结果缓存在内存
     ↓
持久化缓存 → module_outputs/*.json，重启不丢失
```

实现思路：
```python
def compute_cache_key(inputs: dict) -> str:
    """用输入内容的hash作为缓存key"""
    import hashlib, json
    return hashlib.sha256(json.dumps(inputs, sort_keys=True).encode()).hexdigest()

# 检查缓存
cache_key = compute_cache_key({"topic": "SCHD", "style": "analysis"})
if cache_key in cache_store:
    return cache_store[cache_key]
```

### 3. GraphBuilder 编程式工作流构建

```python
from .graph_utils import GraphBuilder

gb = GraphBuilder()
a = gb.node("LLMNode", topic="股市分析")
b = gb.node("VideoNode", prompt=a.out(0))  # 引用上一节点输出
c = gb.node("AudioNode", text=b.out(0))
workflow = gb.finalize()  # → 标准的dict格式
```

比手动拼JSON更优雅，支持动态子图注入。

### 4. Node输入输出类型系统

每个模块定义清晰的三元组：

| 部分 | 内容 |
|------|------|
| **Schema** | 模块ID、分类、描述 |
| **Inputs** | 每个输入的类型（String/Image/Video/Combo）+ 验证规则 |
| **Outputs** | 输出类型 + 缓存策略 |

参考：`comfy_api_nodes/nodes_runway.py` 的 `define_schema()` + `execute()` 模式。

## 修改可行性设计

每个模块支持完整的修改循环：

```
任意模块输出 ←→ 可编辑 ←→ 可重新运行

具体操作：
- 点击任意模块可查看其输出内容
- 修改后点击"重置"可清空该模块及后续所有模块
- 从该模块重新开始运行
```

修改场景矩阵：
```
从任意模块都可以退回到前面任意模块重新开始
  A(选题)✗ → 重置A → 重新选B → C → D → E → F → G
  A → B(文案)✗ → 重置B → C → D → E → F → G
  A → B → C(审查)✗ → 重置C → D → E → F → G
  ...以此类推
```

## API 端点

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/state` | 获取完整工作流状态 |
| GET | `/api/config` | 获取工具配置 |
| PUT | `/api/config` | 更新工具配置 |
| POST | `/api/module/<id>/run` | 运行指定模块 |
| POST | `/api/module/<id>/reset` | 重置指定模块 |
| GET | `/api/module/<id>/output` | 获取模块输出 |
| PUT | `/api/module/<id>/input` | 更新模块输入 |
| POST | `/api/workflow/start` | 从指定模块开始 |
| POST | `/api/workflow/reset` | 重置整个工作流 |

## 状态定义

| 状态 | 说明 |
|------|------|
| `idle` | 待启动 |
| `running` | 进行中（2秒轮询更新） |
| `done` | 已完成 |
| `error` | 错误 |
| `paused` | 暂停 |

## 文件结构

```
creative-workflow-dashboard/
├── SKILL.md
├── scripts/
│   ├── server.py      ← Web服务器 (Flask)
│   └── tui.py        ← 终端面板 (curses)
├── templates/
│   └── dashboard.html ← Web前端
├── state/
│   ├── workflow_state.json
│   ├── config.json
│   └── module_outputs/
│       ├── A_output.json  ← 选题
│       ├── B_output.json  ← 文案
│       ├── C_output.json  ← 审查
│       ├── D_output.json  ← 视频
│       ├── E_output.json  ← 语音
│       ├── F_output.json  ← 组装
│       └── G_output.json  ← 发布
└── static/               ← 预留静态资源
```

## TUI 快捷键

| 按键 | 功能 |
|------|------|
| `↑/↓` 或 `j/k` | 选择模块 |
| `Enter` | 运行当前模块 |
| `r` | 重置当前模块 |
| `c` | 重置整个工作流 |
| `l` | 从A开始完整流程 |
| `q` | 退出 |

## Web面板功能

- 7个模块状态可视化
- 每个模块详情面板（选项列表/文件输出/审查报告）
- 工具配置实时切换（LLM/视频/语音）
- 发布平台选择
- 实时运行日志
- 自动轮询（2秒）更新状态

## 下一步

- [ ] 接入真实LLM（MiniMax/Grok）到模块A、B
- [ ] 接入合规规则到模块C
- [ ] 接入视频生成API到模块D
- [ ] 接入语音合成API到模块E
- [ ] 开发模块F（内容组装）
- [ ] 开发模块G（多平台发布）
