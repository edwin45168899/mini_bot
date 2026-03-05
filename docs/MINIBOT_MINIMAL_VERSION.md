# mini_bot 最小可執行版本 — 復刻指南

> 只保留 **CLI 互動聊天 + 基本工具 + LLM 呼叫**，去除所有頻道、Cron、Heartbeat、MCP、Bridge。
> 預估 **~800 行**即可跑通完整的 Agent Loop。
> **預設 LLM Provider：MiniMax M2.5**

---

## 最小版本包含什麼？

```
mini_bot/
├── pyproject.toml          # 精簡依賴
├── minibot/
│   ├── __init__.py
│   ├── __main__.py
│   ├── utils/
│   │   ├── __init__.py
│   │   └── helpers.py       # ~30 行
│   ├── bus/
│   │   ├── __init__.py
│   │   ├── events.py        # ~35 行
│   │   └── queue.py         # ~45 行
│   ├── config/
│   │   ├── __init__.py
│   │   ├── schema.py        # ~60 行（精簡版）
│   │   └── loader.py        # ~40 行
│   ├── session/
│   │   ├── __init__.py
│   │   └── manager.py       # ~100 行
│   ├── providers/
│   │   ├── __init__.py
│   │   ├── base.py          # ~70 行
│   │   └── litellm_provider.py  # ~80 行
│   ├── agent/
│   │   ├── __init__.py
│   │   ├── memory.py        # ~30 行
│   │   ├── context.py       # ~80 行（精簡版）
│   │   ├── loop.py          # ~200 行（精簡版）
│   │   └── tools/
│   │       ├── __init__.py
│   │       ├── base.py      # ~50 行
│   │       ├── registry.py  # ~40 行
│   │       └── filesystem.py # ~80 行
│   └── cli/
│       ├── __init__.py
│       └── commands.py      # ~120 行（精簡版）
└── workspace/
    ├── AGENTS.md
    └── memory/
        └── MEMORY.md
```

**精簡依賴（`pyproject.toml`）**：

```toml
[project]
name = "minibot"
version = "0.1.0"
requires-python = ">=3.11"

dependencies = [
    "typer>=0.20.0,<1.0.0",
    "litellm>=1.81.5,<2.0.0",
    "pydantic>=2.12.0,<3.0.0",
    "pydantic-settings>=2.12.0,<3.0.0",
    "loguru>=0.7.3,<1.0.0",
    "rich>=14.0.0,<15.0.0",
    "prompt-toolkit>=3.0.50,<4.0.0",
    "json-repair>=0.57.0,<1.0.0",
]

[project.scripts]
minibot = "minibot.cli.commands:app"

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["minibot"]
```

---

## 建立順序（6 個步驟）

### 步驟 1：基礎層 — Utils + Bus

**`minibot/utils/helpers.py`**：

```python
"""Utility functions."""
from pathlib import Path
from datetime import datetime

def ensure_dir(path: Path) -> Path:
    path.mkdir(parents=True, exist_ok=True)
    return path

def safe_filename(name: str) -> str:
    for char in '<>:"/\\|?*':
        name = name.replace(char, "_")
    return name.strip()

def get_data_path() -> Path:
    return ensure_dir(Path.home() / ".minibot")

def timestamp() -> str:
    return datetime.now().isoformat()
```

**`minibot/bus/events.py`**：

```python
"""Event types for the message bus."""
from dataclasses import dataclass, field
from datetime import datetime
from typing import Any

@dataclass
class InboundMessage:
    channel: str
    sender_id: str
    chat_id: str
    content: str
    timestamp: datetime = field(default_factory=datetime.now)
    media: list[str] = field(default_factory=list)
    metadata: dict[str, Any] = field(default_factory=dict)

    @property
    def session_key(self) -> str:
        return f"{self.channel}:{self.chat_id}"

@dataclass
class OutboundMessage:
    channel: str
    chat_id: str
    content: str
    reply_to: str | None = None
    media: list[str] = field(default_factory=list)
    metadata: dict[str, Any] = field(default_factory=dict)
```

**`minibot/bus/queue.py`**：

```python
"""Async message queue."""
import asyncio
from minibot.bus.events import InboundMessage, OutboundMessage

class MessageBus:
    def __init__(self):
        self.inbound: asyncio.Queue[InboundMessage] = asyncio.Queue()
        self.outbound: asyncio.Queue[OutboundMessage] = asyncio.Queue()

    async def publish_inbound(self, msg: InboundMessage) -> None:
        await self.inbound.put(msg)

    async def consume_inbound(self) -> InboundMessage:
        return await self.inbound.get()

    async def publish_outbound(self, msg: OutboundMessage) -> None:
        await self.outbound.put(msg)

    async def consume_outbound(self) -> OutboundMessage:
        return await self.outbound.get()
```

---

### 步驟 2：Config + Session

**`minibot/config/schema.py`**（精簡版）：

```python
"""Configuration schema (minimal)."""
from pydantic import BaseModel, Field, ConfigDict
from pydantic.alias_generators import to_camel
from pydantic_settings import BaseSettings

class Base(BaseModel):
    model_config = ConfigDict(alias_generator=to_camel, populate_by_name=True)

class AgentDefaults(Base):
    workspace: str = "~/.minibot/workspace"
    model: str = "minimax/MiniMax-M2.5"
    max_tokens: int = 8192
    temperature: float = 0.7
    max_tool_iterations: int = 20
    memory_window: int = 50

class AgentsConfig(Base):
    defaults: AgentDefaults = Field(default_factory=AgentDefaults)

class ProviderConfig(Base):
    api_key: str = ""
    api_base: str | None = None

class ProvidersConfig(Base):
    minimax: ProviderConfig = Field(default_factory=ProviderConfig)
    openrouter: ProviderConfig = Field(default_factory=ProviderConfig)
    anthropic: ProviderConfig = Field(default_factory=ProviderConfig)
    openai: ProviderConfig = Field(default_factory=ProviderConfig)
    deepseek: ProviderConfig = Field(default_factory=ProviderConfig)
    gemini: ProviderConfig = Field(default_factory=ProviderConfig)

class Config(BaseSettings):
    model_config = ConfigDict(alias_generator=to_camel, populate_by_name=True)
    agents: AgentsConfig = Field(default_factory=AgentsConfig)
    providers: ProvidersConfig = Field(default_factory=ProvidersConfig)
```

**`minibot/config/loader.py`**：

```python
"""Config loading."""
import json
from pathlib import Path
from minibot.config.schema import Config

def get_config_path() -> Path:
    return Path.home() / ".minibot" / "config.json"

def load_config(config_path: Path | None = None) -> Config:
    path = config_path or get_config_path()
    if path.exists():
        try:
            with open(path, encoding="utf-8") as f:
                data = json.load(f)
            return Config.model_validate(data)
        except Exception as e:
            print(f"Warning: Config load failed: {e}")
    return Config()

def save_config(config: Config, config_path: Path | None = None) -> None:
    path = config_path or get_config_path()
    path.parent.mkdir(parents=True, exist_ok=True)
    with open(path, "w", encoding="utf-8") as f:
        json.dump(config.model_dump(by_alias=True), f, indent=2, ensure_ascii=False)
```

**`minibot/session/manager.py`**：Session 持久化（JSONL 格式，約 100 行）。

---

### 步驟 3：Provider（LLM 呼叫）

**`minibot/providers/base.py`**：

```python
"""Base LLM provider interface."""
from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from typing import Any

@dataclass
class ToolCallRequest:
    id: str
    name: str
    arguments: dict[str, Any]

@dataclass
class LLMResponse:
    content: str | None
    tool_calls: list[ToolCallRequest] = field(default_factory=list)
    finish_reason: str = "stop"
    usage: dict[str, int] = field(default_factory=dict)

    @property
    def has_tool_calls(self) -> bool:
        return len(self.tool_calls) > 0

class LLMProvider(ABC):
    def __init__(self, api_key: str | None = None, api_base: str | None = None):
        self.api_key = api_key
        self.api_base = api_base

    @abstractmethod
    async def chat(
        self, messages: list[dict], tools: list[dict] | None = None,
        model: str | None = None, max_tokens: int = 4096, temperature: float = 0.7,
    ) -> LLMResponse: ...

    @abstractmethod
    def get_default_model(self) -> str: ...
```

**`minibot/providers/litellm_provider.py`**（MiniMax 版本）：

```python
"""LiteLLM-based provider (MiniMax optimized)."""
import os
import json_repair
from typing import Any
from loguru import logger
import litellm
from minibot.providers.base import LLMProvider, LLMResponse, ToolCallRequest

class LiteLLMProvider(LLMProvider):
    def __init__(self, api_key: str, model: str, api_base: str | None = None,
                 env_key: str = "MINIMAX_API_KEY", litellm_prefix: str = ""):
        super().__init__(api_key, api_base)
        self.model = model
        self.litellm_prefix = litellm_prefix
        os.environ[env_key] = api_key
        if api_base:
            os.environ["MINIMAX_API_BASE"] = api_base
        litellm.drop_params = True

    def _prefixed_model(self, model: str | None) -> str:
        m = model or self.model
        if self.litellm_prefix and not m.startswith(self.litellm_prefix + "/"):
            return f"{self.litellm_prefix}/{m}"
        return m

    async def chat(self, messages, tools=None, model=None,
                   max_tokens=4096, temperature=0.7) -> LLMResponse:
        kwargs: dict[str, Any] = {
            "model": self._prefixed_model(model),
            "messages": messages,
            "max_tokens": max_tokens,
            "temperature": temperature,
        }
        if self.api_base:
            kwargs["api_base"] = self.api_base
        if self.api_key:
            kwargs["api_key"] = self.api_key
        if tools:
            kwargs["tools"] = tools
            kwargs["tool_choice"] = "auto"

        try:
            resp = await litellm.acompletion(**kwargs)
            choice = resp.choices[0]
            msg = choice.message

            tool_calls = []
            if msg.tool_calls:
                for tc in msg.tool_calls:
                    args = tc.function.arguments
                    if isinstance(args, str):
                        args = json_repair.loads(args)
                    tool_calls.append(ToolCallRequest(
                        id=tc.id, name=tc.function.name, arguments=args
                    ))

            return LLMResponse(
                content=msg.content,
                tool_calls=tool_calls,
                finish_reason=choice.finish_reason or "stop",
                usage=dict(resp.usage) if resp.usage else {},
            )
        except Exception as e:
            logger.error("LLM call failed: {}", e)
            return LLMResponse(content=f"Error: {e}")

    def get_default_model(self) -> str:
        return self.model
```

---

### 步驟 4：Tool 系統（只保留 filesystem）

**`minibot/agent/tools/base.py`**：

```python
"""Base tool class."""
from abc import ABC, abstractmethod
from typing import Any

class Tool(ABC):
    @property
    @abstractmethod
    def name(self) -> str: ...
    @property
    @abstractmethod
    def description(self) -> str: ...
    @property
    @abstractmethod
    def parameters(self) -> dict[str, Any]: ...
    @abstractmethod
    async def execute(self, **kwargs) -> str: ...

    def to_schema(self) -> dict[str, Any]:
        return {
            "type": "function",
            "function": {
                "name": self.name,
                "description": self.description,
                "parameters": self.parameters,
            }
        }
```

**`minibot/agent/tools/registry.py`**：

```python
"""Tool registry."""
from typing import Any
from minibot.agent.tools.base import Tool

class ToolRegistry:
    def __init__(self):
        self._tools: dict[str, Tool] = {}

    def register(self, tool: Tool) -> None:
        self._tools[tool.name] = tool

    def get(self, name: str) -> Tool | None:
        return self._tools.get(name)

    def get_definitions(self) -> list[dict[str, Any]]:
        return [t.to_schema() for t in self._tools.values()]

    async def execute(self, name: str, params: dict[str, Any]) -> str:
        tool = self._tools.get(name)
        if not tool:
            return f"Error: Tool '{name}' not found"
        try:
            return await tool.execute(**params)
        except Exception as e:
            return f"Error executing {name}: {e}"
```

**`minibot/agent/tools/filesystem.py`**（3 個基本工具）：

```python
"""File system tools."""
from pathlib import Path
from minibot.agent.tools.base import Tool

class ReadFileTool(Tool):
    name = "read_file"
    description = "Read the contents of a file."
    parameters = {
        "type": "object",
        "properties": {"path": {"type": "string", "description": "File path"}},
        "required": ["path"]
    }
    async def execute(self, path: str) -> str:
        p = Path(path).expanduser()
        if not p.exists():
            return f"Error: File not found: {path}"
        return p.read_text(encoding="utf-8")[:50000]

class WriteFileTool(Tool):
    name = "write_file"
    description = "Write content to a file."
    parameters = {
        "type": "object",
        "properties": {
            "path": {"type": "string"}, "content": {"type": "string"}
        },
        "required": ["path", "content"]
    }
    async def execute(self, path: str, content: str) -> str:
        p = Path(path).expanduser()
        p.parent.mkdir(parents=True, exist_ok=True)
        p.write_text(content, encoding="utf-8")
        return f"Written {len(content)} chars to {path}"

class ListDirTool(Tool):
    name = "list_dir"
    description = "List contents of a directory."
    parameters = {
        "type": "object",
        "properties": {"path": {"type": "string", "description": "Directory path"}},
        "required": ["path"]
    }
    async def execute(self, path: str) -> str:
        p = Path(path).expanduser()
        if not p.is_dir():
            return f"Error: Not a directory: {path}"
        items = []
        for item in sorted(p.iterdir()):
            prefix = "[DIR] " if item.is_dir() else "[FILE]"
            items.append(f"{prefix} {item.name}")
        return "\n".join(items) or "(empty directory)"
```

---

### 步驟 5：Agent 核心迴圈（精簡版）

**`minibot/agent/memory.py`**：31 行，雙層記憶（MEMORY.md + HISTORY.md）。

**`minibot/agent/context.py`**（精簡版）：

```python
"""Context builder (minimal)."""
import platform
from datetime import datetime
from pathlib import Path
from minibot.agent.memory import MemoryStore

class ContextBuilder:
    def __init__(self, workspace: Path):
        self.workspace = workspace
        self.memory = MemoryStore(workspace)

    def build_system_prompt(self) -> str:
        parts = [
            "You are mini_bot 🤖, a personal AI assistant.",
            f"Platform: {platform.system()} {platform.release()}",
            f"Working directory: {self.workspace}",
            f"Current time: {datetime.now().isoformat()}",
        ]
        for name in ("AGENTS.md", "SOUL.md"):
            f = self.workspace / name
            if f.exists():
                parts.append(f"\n## {name}\n{f.read_text(encoding='utf-8')}")
        mem = self.memory.get_memory_context()
        if mem:
            parts.append(f"\n{mem}")
        return "\n\n".join(parts)

    def build_messages(self, history: list[dict], current_message: str) -> list[dict]:
        messages = [{"role": "system", "content": self.build_system_prompt()}]
        messages.extend(history)
        messages.append({"role": "user", "content": current_message})
        return messages
```

**`minibot/agent/loop.py`**（精簡版，~150 行）：AgentLoop 負責 LLM ↔ Tool 迭代執行迴圈。

---

### 步驟 6：CLI 指令（精簡版）

**`minibot/cli/commands.py`**：

三個指令：
- `minibot onboard` — 初始化設定與 workspace
- `minibot agent` — 互動聊天（支援 `--message` 單次模式）
- `minibot status` — 顯示設定狀態

---

## 執行方式

```powershell
# 1. 進入專案目錄
cd c:\GitHub\chiisen\mini_bot

# 2. 安裝
pip install -e .

# 3. 初始化
minibot onboard

# 4. 設定 API Key（編輯 ~/.minibot/config.json）
# {
#   "providers": {
#     "minimax": {
#       "apiKey": "your-minimax-api-key",
#       "apiBase": "https://api.minimax.io/v1"
#     }
#   },
#   "agents": { "defaults": { "model": "minimax/MiniMax-M2.5" } }
# }

# 5. 聊天！
minibot agent
minibot agent -m "Hello!"

# 6. 查看狀態
minibot status
```

---

## 最小版本 vs 完整版本差異

| 功能 | 最小版本 | 完整版本 |
|------|---------|---------|
| CLI 互動聊天 | ✅ | ✅ |
| Session 持久化 | ✅ | ✅ |
| 記憶系統 | ✅ (MEMORY.md) | ✅ + 自動整合 |
| 檔案工具 | ✅ (read/write/list) | ✅ + edit |
| Shell 執行 | ❌ | ✅ |
| Web 搜尋/擷取 | ❌ | ✅ |
| Telegram 頻道 | ❌ | ✅ |
| Discord 頻道 | ❌ | ✅ |
| 其他 8 個頻道 | ❌ | ✅ |
| MCP 支援 | ❌ | ✅ |
| Cron 排程 | ❌ | ✅ |
| Heartbeat | ❌ | ✅ |
| SubAgent | ❌ | ✅ |
| 技能系統 | ❌ | ✅ |
| Docker | ❌ | ✅ |
| Provider 數量 | 6 | 15+ |
| 預估行數 | **~800 行** | **~4,000 行** |

---

## 擴展路線

一旦最小版本跑通，可按此順序逐步加入更多功能：

1. **Shell 工具** (`exec`) → 讓 Agent 能執行指令
2. **Web 工具** (`web_search`, `web_fetch`) → 讓 Agent 能上網
3. **記憶自動整合** → `_consolidate_memory()` 方法
4. **Telegram 頻道** → 第一個外部頻道
5. **Cron 排程** → 定時任務
6. **MCP 支援** → 外部工具伺服器
7. **更多頻道** → Discord, Slack, ...

---

*本指南基於 mini_bot v0.1.0，提取最小可執行子集，預設使用 MiniMax M2.5 作為 LLM Provider。*
