# Agent控制机器人物理世界交互方案

> 目标：让电脑上的开发Agent（如Trae IDE AI）通过wo-bot平台控制机器人，实现室内环境感知、家电控制、电脑开机、定时任务等物理世界交互。

---

## 一、项目现状诊断

wo-bot项目目前已完成 **v1.x 通信基础设施层**：

| 已完成 | 说明 |
|--------|------|
| R00002 云台控制 | 摄像头云台角度控制 |
| R00003 ROS 运动控制 | 麦克纳姆轮底盘控制 |
| R00004 基础舞蹈 | DanceController 框架 |
| R00010 WebRTC修复 | 视频流稳定传输 |
| R00011 日志系统优化 | RotatingFileHandler + 增量同步 |
| R00015 音频硬件层 | ALSA音频播放 |
| R00024 音乐投屏 | AirPlay/DLNA/蓝牙 |
| R00031 电量检测 | 电池电压采集+省电策略 |
| R00032 一键喊话 | 客户端到机器人语音喊话 |
| R00033 配置可视化 | binding/power_policy 配置面板 |
| R00035 设备绑定 | 四方式认证+分享绑定 |
| R00038 寻找设备 | 声光提示定位 |
| R00039 软件包管理 | 白名单制+wo-bot-market市场 |
| WebSocket信令 | ws://:8765 协议 |
| WebRTC P2P | 视频流 + DataChannel |
| mDNS发现 | 局域网零配置发现 |

**硬件底座**：Jetson Nano B01 + ROS1 + 麦克纳姆轮底盘 + 2D激光雷达 + 深度云台相机 + 扬声器

**核心问题**：当前机器人只能被动接收遥控指令，缺少自主感知、决策、执行高级任务的能力。但设备绑定（R00035）已完成，为 Agent 安全接入奠定了基础。

---

## 二、总体架构：三层体系

```
┌─────────────────────────────────────────────────────────────┐
│  电脑Agent（Trae IDE / Claude / 自定义Agent）                │
│  通过 WebSocket / MCP 与机器人通信                          │
└──────────────────────────┬──────────────────────────────────┘
                           │ WebSocket (ws://robot:8765)
┌──────────────────────────▼──────────────────────────────────┐
│  第三层：Agent接入层（Agent Integration API）                │
│  扩展WebSocket协议，新增 agent_task / agent_direct_action    │
│  等消息类型，提供能力查询、任务下发、状态反馈                 │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│  第二层：任务编排层（TaskPlanner）                           │
│  接收高层意图 → LLM生成执行计划 → 调度原子动作               │
│  支持失败重规划、进度推送、任务取消                           │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│  第一层：能力原子层（PluginManager + BasePlugin）            │
│  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐    │
│  │导航  │ │视觉  │ │红外  │ │音频  │ │运动  │ │场景  │    │
│  │navig │ │vision│ │ir    │ │voice │ │motion│ │scene │    │
│  └──────┘ └──────┘ └──────┘ └──────┘ └──────┘ └──────┘    │
│  统一生命周期：init → start → execute_action → stop          │
│  故障隔离：单插件崩溃不影响其他插件                           │
└─────────────────────────────────────────────────────────────┘
```

---

## 三、第一层：R00011 插件化架构详细设计

### 3.1 插件接口规范

```python
# wo-bot-control/src/plugins/base_plugin.py

from abc import ABC, abstractmethod
from enum import Enum
from typing import Any, Dict, List, Optional
from dataclasses import dataclass

class PluginState(Enum):
    UNLOADED = "unloaded"
    LOADING = "loading"
    READY = "ready"
    RUNNING = "running"
    PAUSED = "paused"
    ERROR = "error"
    STOPPING = "stopping"

@dataclass
class ActionSchema:
    """每个插件暴露的能力原子描述"""
    name: str                    # e.g. "navigate_to"
    description: str             # e.g. "导航到指定房间坐标"
    parameters: Dict[str, Any]   # JSON Schema 格式的参数定义
    returns: str                 # 返回值描述
    timeout_ms: int = 30000      # 默认超时

class BasePlugin(ABC):
    """所有功能插件的基类"""

    # --- 元信息 ---
    plugin_id: str = ""          # 唯一标识，如 "navigation", "ir_control"
    plugin_name: str = ""        # 显示名称
    version: str = "1.0.0"
    dependencies: List[str] = [] # 依赖的其他 plugin_id

    # --- 生命周期 ---
    state: PluginState = PluginState.UNLOADED

    @abstractmethod
    async def on_init(self, config: Dict[str, Any]) -> bool:
        """初始化：加载配置、连接硬件、检查依赖"""
        ...

    @abstractmethod
    async def on_start(self) -> bool:
        """启动：开始接收指令、启动后台任务"""
        ...

    @abstractmethod
    async def on_stop(self) -> bool:
        """停止：清理资源、关闭连接"""
        ...

    async def on_pause(self) -> bool:
        """暂停：暂停执行中的任务（可选覆盖）"""
        return True

    async def on_resume(self) -> bool:
        """恢复：恢复暂停的任务（可选覆盖）"""
        return True

    async def health_check(self) -> bool:
        """健康检查：硬件是否在线、进程是否存活"""
        return self.state == PluginState.RUNNING

    # --- 能力暴露 ---
    def get_actions(self) -> List[ActionSchema]:
        """返回本插件提供的所有原子动作"""
        return []

    @abstractmethod
    async def execute_action(self, action_name: str, params: Dict[str, Any]) -> Dict[str, Any]:
        """执行指定的原子动作，返回结果"""
        ...
```

### 3.2 插件管理器

```python
# wo-bot-control/src/core/plugin_manager.py (新增)

import asyncio
import importlib
import traceback
from typing import Dict, Optional
from pathlib import Path

class PluginManager:
    """
    插件管理器：
    - 注册/加载/卸载/热重载插件
    - 监控插件健康状态
    - 转发 execute_action 调用到目标插件
    - 故障隔离：单个插件崩溃不影响其他插件
    """

    def __init__(self, plugins_dir: str = "src/plugins"):
        self._plugins: Dict[str, BasePlugin] = {}
        self._plugins_dir = Path(plugins_dir)
        self._sandbox_tasks: Dict[str, asyncio.Task] = {}

    async def load_plugin(self, plugin_id: str, config: dict) -> bool:
        """加载并初始化插件"""
        # 1. 动态导入插件模块
        module = importlib.import_module(f"plugins.{plugin_id}")
        plugin = module.Plugin()

        # 2. 检查依赖
        for dep_id in plugin.dependencies:
            if dep_id not in self._plugins or self._plugins[dep_id].state != PluginState.RUNNING:
                plugin.state = PluginState.ERROR
                return False

        # 3. 初始化
        plugin.state = PluginState.LOADING
        if not await plugin.on_init(config):
            return False

        plugin.state = PluginState.READY
        if not await plugin.on_start():
            return False

        plugin.state = PluginState.RUNNING
        self._plugins[plugin_id] = plugin

        # 4. 启动健康监控协程
        self._sandbox_tasks[plugin_id] = asyncio.create_task(
            self._monitor_plugin(plugin_id)
        )
        return True

    async def execute_action(self, plugin_id: str, action: str, params: dict) -> dict:
        """转发动作到指定插件（带超时和异常捕获）"""
        plugin = self._plugins.get(plugin_id)
        if not plugin or plugin.state != PluginState.RUNNING:
            return {"success": False, "error": f"Plugin {plugin_id} not available"}

        action_schemas = {a.name: a for a in plugin.get_actions()}
        schema = action_schemas.get(action)
        timeout = schema.timeout_ms / 1000 if schema else 30

        try:
            result = await asyncio.wait_for(
                plugin.execute_action(action, params),
                timeout=timeout
            )
            return {"success": True, "data": result}
        except asyncio.TimeoutError:
            return {"success": False, "error": f"Action {action} timed out after {timeout}s"}
        except Exception as e:
            return {"success": False, "error": str(e), "traceback": traceback.format_exc()}

    def get_all_actions(self) -> Dict[str, list]:
        """获取所有已注册插件的能力清单（供 TaskPlanner 和 Agent 查询）"""
        return {
            pid: [{"name": a.name, "description": a.description, "params": a.parameters}
                  for a in p.get_actions()]
            for pid, p in self._plugins.items()
            if p.state == PluginState.RUNNING
        }

    async def _monitor_plugin(self, plugin_id: str):
        """每5秒检查一次插件健康，异常时尝试重启（最多3次）"""
        fail_count = 0
        while True:
            await asyncio.sleep(5)
            plugin = self._plugins.get(plugin_id)
            if not plugin:
                return
            if not await plugin.health_check():
                fail_count += 1
                if fail_count <= 3:
                    await plugin.on_stop()
                    await plugin.on_start()
                else:
                    plugin.state = PluginState.ERROR
                    return
```

---

## 四、第二层：TaskPlanner 任务编排引擎

```python
# wo-bot-control/src/core/task_planner.py (新增)

import json
import asyncio
from typing import List, Dict, Any
from dataclasses import dataclass
from enum import Enum

class TaskState(Enum):
    PENDING = "pending"
    PLANNING = "planning"
    EXECUTING = "executing"
    COMPLETED = "completed"
    FAILED = "failed"
    CANCELLED = "cancelled"

@dataclass
class TaskStep:
    step_id: int
    plugin_id: str          # 例如 "navigation"
    action: str             # 例如 "navigate_to"
    params: dict            # 例如 {"room": "书房"}
    on_success: int = None  # 成功后跳转到 step_id
    on_failure: int = None  # 失败后跳转到 step_id

class TaskPlanner:
    """
    任务编排引擎：
    1. 接收自然语言/结构化任务描述
    2. 调用 LLM 生成执行计划（JSON 步骤序列）
    3. 按序执行，支持条件跳转和重试
    4. 实时推送进度到 Agent
    """

    def __init__(self, plugin_manager: PluginManager, ws_server):
        self.pm = plugin_manager
        self.ws = ws_server
        self._active_tasks: Dict[str, dict] = {}

    async def execute_task(self, task_id: str, intent: str, context: dict = None):
        """执行一个任务"""
        self._active_tasks[task_id] = {
            "state": TaskState.PLANNING,
            "intent": intent,
            "steps": [],
            "current_step": 0
        }

        # 1. 获取当前可用能力
        capabilities = self.pm.get_all_actions()

        # 2. 让 LLM 生成执行计划
        plan = await self._plan_with_llm(intent, capabilities, context)
        if not plan:
            await self._notify_agent(task_id, TaskState.FAILED, "LLM planning failed")
            return

        steps = [TaskStep(**s) for s in plan["steps"]]
        self._active_tasks[task_id]["steps"] = steps
        self._active_tasks[task_id]["state"] = TaskState.EXECUTING

        # 3. 逐步执行
        step_idx = 0
        while step_idx < len(steps):
            step = steps[step_idx]
            await self._notify_agent(task_id, TaskState.EXECUTING,
                f"执行步骤 {step_idx+1}/{len(steps)}: {step.plugin_id}.{step.action}",
                {"current_step": step_idx, "action": step.action})

            result = await self.pm.execute_action(step.plugin_id, step.action, step.params)

            if result["success"]:
                step_idx = step.on_success if step.on_success is not None else step_idx + 1
            else:
                # 失败：尝试重规划
                retry_plan = await self._replan_with_llm(intent, capabilities, step_idx, result["error"])
                if retry_plan:
                    new_steps = [TaskStep(**s) for s in retry_plan["steps"]]
                    steps = steps[:step_idx] + new_steps
                else:
                    await self._notify_agent(task_id, TaskState.FAILED,
                        f"步骤 {step.action} 失败: {result['error']}")
                    return

        await self._notify_agent(task_id, TaskState.COMPLETED, "任务完成")

    async def _plan_with_llm(self, intent: str, capabilities: dict, context: dict) -> dict:
        """
        调用 LLM 生成执行计划。
        使用与环境感知相关的上下文（当前房间、已识别物体等）
        """
        prompt = f"""你是一个机器人任务规划器。请将用户意图分解为原子动作序列。

可用能力：
{json.dumps(capabilities, ensure_ascii=False, indent=2)}

当前上下文：
{json.dumps(context or {}, ensure_ascii=False)}

用户意图：{intent}

请返回 JSON 格式的执行计划：
{{"steps": [
  {{"step_id": 0, "plugin_id": "xxx", "action": "xxx", "params": {{...}}, "on_success": 1, "on_failure": -1}}
]}}

规则：
- 每个 step 只能调用一个原子动作
- 参数必须符合能力定义的 JSON Schema
- on_success 指向下一个 step_id（默认+1），on_failure 指向失败处理 step（-1 表示中止）
"""
        # 实际实现中调用 LLM API（豆包/OpenAI）
        response = await self._call_llm(prompt)
        try:
            return json.loads(response)
        except json.JSONDecodeError:
            return None

    async def _replan_with_llm(self, intent: str, capabilities: dict,
                                failed_step: int, error: str) -> dict:
        """执行失败后重新规划"""
        prompt = f"""步骤 {failed_step} 执行失败：{error}。请重新规划后续步骤。

原始意图：{intent}
可用能力：{json.dumps(capabilities, ensure_ascii=False)}

请返回可替代的执行步骤 JSON。"""
        response = await self._call_llm(prompt)
        try:
            return json.loads(response)
        except json.JSONDecodeError:
            return None

    async def _notify_agent(self, task_id: str, state: TaskState, message: str, detail: dict = None):
        """通过 WebSocket 推送任务状态给 Agent"""
        await self.ws.broadcast({
            "type": "agent_task_status",
            "data": {
                "task_id": task_id,
                "status": state.value,
                "message": message,
                "detail": detail or {}
            }
        })
```

---

## 五、第三层：WebSocket Agent 协议扩展

在现有 `websocket协议（用于局域网通讯）.md` 基础上新增以下消息类型：

### 5.1 Agent → 机器人

```json
// 1. 查询可用能力
{ "type": "agent_get_capabilities", "data": {} }
→ 响应: {
    "type": "agent_capabilities",
    "data": {
        "plugins": {
            "navigation": [
                {"name": "navigate_to", "description": "导航到指定房间", "params": {"room": "string"}},
                {"name": "get_position", "description": "获取当前位置", "params": {}}
            ],
            "vision": [
                {"name": "scan_room", "description": "扫描房间，返回物体列表", "params": {}},
                {"name": "identify_object", "description": "识别指定物体位置", "params": {"object": "string"}}
            ],
            "ir_control": [
                {"name": "send_ir", "description": "发送红外指令", "params": {"device": "string", "command": "string"}}
            ],
            "voice": [
                {"name": "speak", "description": "语音播报", "params": {"text": "string"}}
            ]
        }
    }
}

// 2. 下发高层任务（LLM规划模式）
{
    "type": "agent_task",
    "data": {
        "task_id": "uuid-xxxx",
        "intent": "去书房帮我把电脑开机",
        "mode": "auto_plan",
        "context": {
            "priority": "normal",
            "timeout_ms": 120000
        }
    }
}

// 3. 手动指定动作序列（跳过LLM规划）
{
    "type": "agent_task",
    "data": {
        "task_id": "uuid-xxxx",
        "mode": "manual_steps",
        "steps": [
            {"plugin_id": "navigation", "action": "navigate_to", "params": {"room": "书房"}},
            {"plugin_id": "vision", "action": "identify_object", "params": {"object": "电脑"}},
            {"plugin_id": "ir_control", "action": "send_ir", "params": {"device": "电脑", "command": "power_on"}}
        ]
    }
}

// 4. 取消任务
{ "type": "agent_cancel_task", "data": { "task_id": "uuid-xxxx" } }

// 5. 直接调用单个原子动作（Agent已自己规划好）
{
    "type": "agent_direct_action",
    "data": {
        "plugin_id": "navigation",
        "action": "navigate_to",
        "params": {"room": "客厅"}
    }
}
```

### 5.2 机器人 → Agent

```json
// 任务状态更新
{
    "type": "agent_task_status",
    "data": {
        "task_id": "uuid-xxxx",
        "status": "executing",
        "message": "执行步骤 2/4: vision.identify_object",
        "progress": { "current_step": 1, "total_steps": 4 },
        "result": null
    }
}

// 任务完成
{
    "type": "agent_task_status",
    "data": {
        "task_id": "uuid-xxxx",
        "status": "completed",
        "message": "电脑已开机",
        "result": { "room": "书房", "action_taken": "发送红外开机信号", "success": true }
    }
}

// 环境事件推送（主动上报）
{
    "type": "agent_event",
    "data": {
        "event": "person_detected",
        "detail": { "location": "客厅", "is_owner": false, "timestamp": 1719745321 }
    }
}
```

---

## 六、各能力模块设计

### 6.1 导航模块 (navigation plugin)

> 依赖：R00013 避障安全系统 + R00014 人体跟随系统（SLAM建图）

```python
class NavigationPlugin(BasePlugin):
    plugin_id = "navigation"

    def get_actions(self):
        return [
            ActionSchema("navigate_to", "导航到指定房间或坐标",
                {"room": {"type": "string", "description": "房间名称，如 书房/客厅/卧室"}}),
            ActionSchema("get_position", "获取当前位置", {}),
            ActionSchema("scan_and_map", "扫描当前空间并更新地图", {}),
            ActionSchema("return_to_dock", "返回充电坞", {}),
        ]
```

**电脑开机**实现方案：
- **WoL (Wake-on-LAN)**：机器人通过局域网发送魔术包唤醒电脑（推荐）
- **GPIO 模拟按键**：若机器人与电脑有物理连线，用 GPIO 输出短接开机跳线
- 在 `agent_task` 的 context 中由 Agent 指定开机方式

### 6.2 红外控制模块 (ir_control plugin)

> 依赖：R00020 分体式红外收发遥控家居模块

```python
class IRControlPlugin(BasePlugin):
    plugin_id = "ir_control"

    def get_actions(self):
        return [
            ActionSchema("send_ir", "发送红外指令", {
                "device_type": {"type": "string", "enum": ["tv", "ac", "fan", "light", "projector"]},
                "brand": {"type": "string"},
                "command": {"type": "string", "description": "如 power_on, volume_up, temp_26"}
            }),
            ActionSchema("learn_ir", "学习新红外码", {
                "device_name": {"type": "string"}
            }),
            ActionSchema("list_devices", "列出已配对设备", {}),
        ]
```

### 6.3 视觉模块 (vision plugin)

> 依赖：R00017 摄像头视觉拓展功能

```python
class VisionPlugin(BasePlugin):
    plugin_id = "vision"
    dependencies = ["gimbal"]  # 依赖云台控制

    def get_actions(self):
        return [
            ActionSchema("scan_room", "扫描当前房间，返回物体和人列表",
                {"duration_ms": {"type": "int", "default": 5000}}),
            ActionSchema("identify_object", "识别并定位指定物体",
                {"object": {"type": "string"}, "timeout_ms": {"type": "int", "default": 10000}}),
            ActionSchema("detect_person", "检测房间内的人",
                {"check_owner": {"type": "bool", "default": true}}),
            ActionSchema("take_photo", "拍照并返回图片", {}),
        ]
```

### 6.4 音频模块 (voice plugin)

> 依赖：R00016 TTS/ASR 语音助手 + R00032 一键喊话

```python
class VoicePlugin(BasePlugin):
    plugin_id = "voice"

    def get_actions(self):
        return [
            ActionSchema("speak", "语音播报", {
                "text": {"type": "string", "description": "要播报的文本内容"}
            }),
            ActionSchema("listen", "语音识别（监听指令）", {
                "timeout_ms": {"type": "int", "default": 5000}
            }),
        ]
```

---

## 七、完整调用链路示例

```
电脑Agent (Trae IDE)
    │
    │  WebSocket 连接 ws://192.168.1.47:8765
    │  发送: {"type": "agent_task", "data": {"task_id": "t001",
    │          "intent": "去书房帮我把电脑开机", "mode": "auto_plan"}}
    ▼
wo-bot-control (Jetson Nano)
    │
    ├─ TaskPlanner 收到任务
    │   ├─ 查询 PluginManager → 获取当前可用能力清单
    │   ├─ 调用 LLM → 生成步骤序列:
    │   │   [{step:0, plugin:"navigation", action:"navigate_to", params:{room:"书房"}},
    │   │    {step:1, plugin:"vision", action:"identify_object", params:{object:"电脑"}},
    │   │    {step:2, plugin:"ir_control", action:"send_ir", params:{device:"电脑",command:"power_on"}},
    │   │    {step:3, plugin:"voice", action:"speak", params:{text:"电脑已开机"}}]
    │   │
    │   ├─ 执行 step 0: navigation.navigate_to("书房")
    │   │   └─ SLAM路径规划 → 麦克纳姆轮移动 → 到达书房
    │   │
    │   ├─ 执行 step 1: vision.identify_object("电脑")
    │   │   └─ 云台旋转扫描 → YOLO识别 → 返回电脑坐标
    │   │
    │   ├─ 执行 step 2: ir_control.send_ir("电脑", "power_on")
    │   │   └─ 云台对准电脑方向 → 发射红外/WoL → 等待确认
    │   │
    │   └─ 执行 step 3: voice.speak("电脑已开机")
    │       └─ TTS播报
    │
    └─ 每步实时推送 agent_task_status 给 Agent
```

---

## 八、关于"外出采购"的特殊考虑

这是最复杂的能力，需要：

1. **室外 SLAM 导航**：依赖 R00014 的 SLAM 基础 + 室外环境适配（GPS辅助定位）
2. **电梯/门禁交互**：需要机械臂或与物业IoT系统对接（当前硬件不支持）
3. **4G/5G 远程通信**：依赖 R00029，确保离开WiFi覆盖后仍可控
4. **更高算力**：Jetson Nano 可能不足以支撑室外实时导航和避障

**建议**：将"外出采购"作为 **v4.0+ 远期目标**，先聚焦室内场景。若确实需要，可考虑升级硬件为 Jetson Orin 并增加 GPS + 室外深度相机。

---

## 九、依赖的需求列表

| 需求编号 | 需求名称 | 当前状态 | 在方案中的作用 |
|----------|----------|----------|---------------|
| R00011 | 系统底层架构与工程化 | ⚪ 未开始 | **核心基础**：插件化框架 |
| R00013 | 全屋多层级避障安全系统 | ⚪ 未开始 | 导航模块的避障能力 |
| R00014 | 全屋人体全天候跟随系统 | ❌ 已移除 | SLAM建图导航能力 |
| R00016 | 语音交互TTS/ASR | ⚪ 未开始 | 语音播报和语音指令 |
| R00017 | 摄像头视觉拓展功能 | ⚪ 未开始 | 物体识别、环境感知 |
| R00020 | 分体式红外收发遥控 | ⚪ 未开始 | 家电红外控制 |
| R00025 | 招牌核心场景联动 | ⚪ 未开始 | 场景模式编排 |
| R00026 | 定时自动化任务系统 | ⚪ 未开始 | 定时任务调度 |
| R00029 | 4G/5G全网通远控 | ⚪ 未开始 | 室外远程通信 |
| R00012 | 自动充电续航管理系统 | ⚪ 未开始 | 自主回充续航 |

---

## 十、推荐实施路线

### Phase 1（当前→2周）：Agent通信基础
- [ ] R00011 插件化架构：实现 PluginManager + BasePlugin 框架
- [ ] WebSocket Agent 协议扩展：新增 agent_task/agent_direct_action 等消息类型
- [ ] 简单 TaskPlanner 原型：硬编码规则 + LLM fallback

### Phase 2（2-4周）：室内感知与控制
- [ ] R00017 视觉拓展：YOLO物体识别 → vision plugin
- [ ] R00020 红外家电控制完成 → ir_control plugin
- [ ] R00013 基础避障 + R00014 SLAM建图导航 → navigation plugin
- [ ] TaskPlanner 对接上述能力插件

### Phase 3（4-8周）：端到端闭环
- [ ] R00016 TTS/ASR 语音交互 → voice plugin
- [ ] R00026 定时自动化任务
- [ ] R00025 场景联动
- [ ] 完整"感知→决策→执行→反馈"闭环

### Phase 4（远期）：室外扩展
- [ ] R00029 4G/5G远控
- [ ] R00012 自动回充
- [ ] 室外导航（需硬件升级：GPS + Jetson Orin）
- [ ] 外出采购场景验证

---

## 十一、最快验证路径

完成 Phase 1 + 将已完成模块改造为插件后，即可跑通第一个端到端 Agent 任务：

```
Agent指令: "去客厅并对我说一句你好"
  → navigation.navigate_to("客厅")   [复用已完成运动控制]
  → voice.speak("你好")              [复用已完成音频层]
```

验证通过后，逐步叠加视觉、红外等插件。
