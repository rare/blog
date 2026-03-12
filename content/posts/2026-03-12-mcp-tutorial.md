---
title: "MCP 实战教程：AI Agent 的通用接口标准"
date: 2026-03-12
tags: [AI, Agent, MCP, 教程]
---

# MCP 实战教程：AI Agent 的通用接口标准

在 AI Agent 快速发展的今天，一个关键问题逐渐浮现：如何让不同的 AI 系统与各种外部工具、数据源实现无缝对接？2024年11月，Anthropic 推出了 Model Context Protocol（MCP），试图为这个问题提供一个标准化的解决方案。不到一年时间，OpenAI、Microsoft 等巨头纷纷采纳这一协议，MCP 正在成为 AI Agent 领域的"USB-C 接口"。本文将深入剖析 MCP 的核心原理，并通过完整的代码示例展示如何在实际项目中应用这一技术。

## 背景：为什么 AI Agent 需要统一接口

在 MCP 出现之前，开发者为 AI Agent 集成外部工具时面临着碎片化的困境。每个 AI 平台、每个工具提供商都定义了自己的接口规范，导致代码难以复用，系统难以扩展。

Anthropic 于 2024年11月发布了 MCP 开放标准，旨在建立 AI 应用与外部数据源之间的通用通信协议。2025年3月，OpenAI 正式采纳 MCP 协议，并将其集成到 ChatGPT 桌面应用中。随后，Microsoft 宣布 MCP 可与 Semantic Kernel、Azure OpenAI 服务无缝集成。2025年12月，MCP 被捐赠给 Linux Foundation 旗下的 Agentic AI Foundation，标志着这一标准进入行业共建阶段。

MCP 的核心价值在于解耦。开发者无需为每个 AI 平台编写特定的适配器，只需要实现一次 MCP 客户端或服务器，就可以在支持 MCP 的任何系统之间自由切换。这种"一次开发，处处运行"的特性，正是其被比喻为"USB-C 接口"的原因。

## 原理：MCP 的架构设计与核心概念

MCP 采用客户端-服务器架构，支持两种基本模式：主机应用作为客户端连接 MCP 服务器，或者 AI Agent 作为客户端调用外部服务。

### 核心组件

MCP 协议定义了三个核心概念。**主机（Host）** 是发起请求的应用程序，如 Claude Desktop、ChatGPT 桌面版或自定义 AI 应用。**客户端（Client）** 负责与服务器建立和维护连接。**服务器（Server）** 提供工具、资源或提示模板，供客户端调用。

### 通信机制

MCP 使用 JSON-RPC 2.0 作为传输协议，支持 stdio（标准输入输出）和 HTTP+SSE（服务器推送事件）两种传输方式。在 stdio 模式下，MCP 服务器作为子进程启动，通过标准输入输出与主机通信；在 HTTP 模式下，服务器暴露 HTTP 端点，支持更灵活的部署架构。

### 能力协商

MCP 引入了"能力（Capabilities）"机制，允许客户端和服务器在建立连接时声明自己支持的功能。服务器可以声明提供工具（tools）、资源（resources）或提示（prompts）；客户端可以声明需要初始化配置或接受服务器发送的通知。

## 实战：构建一个完整的 MCP 服务

下面通过一个具体示例，展示如何从零构建一个 MCP 服务器并与之交互。本示例创建一个天气查询服务，允许 AI Agent 查询指定城市的天气信息。

### 环境准备

首先安装 MCP SDK：

```bash
pip install mcp
```

### 创建 MCP 服务器

新建文件 `weather_server.py`，实现一个提供天气查询功能的 MCP 服务器：

```python
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp.types import Tool, TextContent
from pydantic import AnyUrl
import asyncio

# 模拟天气数据（实际项目中可替换为真实 API）
WEATHER_DATA = {
    "北京": {"temp": 15, "weather": "晴", "wind": "北风3-4级"},
    "上海": {"temp": 22, "weather": "多云", "wind": "东风2-3级"},
    "深圳": {"temp": 28, "weather": "晴", "wind": "南风1-2级"},
}

app = Server("weather-server")

@app.list_tools()
async def list_tools() -> list[Tool]:
    """列出服务器提供的工具"""
    return [
        Tool(
            name="get_weather",
            description="查询指定城市的天气信息",
            inputSchema={
                "type": "object",
                "properties": {
                    "city": {
                        "type": "string",
                        "description": "城市名称，如：北京、上海、深圳"
                    }
                },
                "required": ["city"]
            }
        )
    ]

@app.call_tool()
async def call_tool(name: str, arguments: dict) -> list[TextContent]:
    """执行工具调用"""
    if name == "get_weather":
        city = arguments.get("city")
        weather = WEATHER_DATA.get(city)
        
        if weather:
            result = f"{city}当前天气：{weather['weather']}，温度：{weather['temp']}°C，风力：{weather['wind']}"
        else:
            result = f"暂不支持查询{city}的天气信息"
        
        return [TextContent(type="text", text=result)]
    
    raise ValueError(f"Unknown tool: {name}")

async def main():
    """启动 MCP 服务器"""
    async with stdio_server() as transport:
        await app.run(transport, app.create_initialization_options())

if __name__ == "__main__":
    asyncio.run(main())
```

### 配置 Claude Desktop 使用 MCP

在 macOS 上，编辑 `~/Library/Application Support/Claude/claude_desktop_config.json`：

```json
{
  "mcpServers": {
    "weather": {
      "command": "python",
      "args": ["/path/to/weather_server.py"]
    }
  }
}
```

配置完成后重启 Claude Desktop，用户就可以通过自然语言让 Claude 查询天气，Claude 会自动调用 MCP 服务器提供的工具。

### 创建 MCP 客户端

如果需要在自定义应用中调用 MCP 服务，可以创建客户端进行连接：

```python
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client
import asyncio

async def query_weather(city: str):
    """通过 MCP 客户端查询天气"""
    server_params = StdioServerParameters(
        command="python",
        args=["/path/to/weather_server.py"]
    )
    
    async with stdio_client(server_params) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()
            
            result = await session.call_tool("get_weather", {"city": city})
            return result[0].text

async def main():
    weather = await query_weather("北京")
    print(weather)

asyncio.run(main())
```

这段代码展示了如何通过 MCP 客户端连接到服务器并调用工具。通过更换服务器地址，相同的客户端代码可以连接不同的 MCP 服务。

## 对比：MCP 与传统方案

在 MCP 出现之前，开发者通常采用自定义 API 或 Function Calling 机制来实现 AI 与外部工具的集成。对比这些方案，可以更清晰地理解 MCP 的优势与局限。

| 特性 | MCP | 自定义 API | OpenAI Function Calling |
|------|-----|------------|------------------------|
| 标准化程度 | 开放标准 | 私有协议 | 厂商限定 |
| 跨平台支持 | 跨多个 AI 供应商 | 需要适配器 | 仅 OpenAI |
| 工具发现 | 自动发现 | 手动配置 | 手动定义 |
| 传输方式 | stdio/HTTP | HTTP/WebSocket | API 调用 |

MCP 的核心优势在于标准化带来的生态互联。一旦工具实现了 MCP 接口，任何支持 MCP 的 AI 应用都可以直接使用，无需重复开发。但这也意味着迁移成本——如果现有系统已经深度定制了自定义接口，转向 MCP 需要一定的改造投入。

## 风险提示

在拥抱 MCP 之前，以下几点值得注意：

**生态尚不成熟**。MCP 虽然获得了 OpenAI 和 Microsoft 的支持，但整个生态仍处于早期阶段。服务器实现数量有限，部分工具的 MCP 支持可能不够稳定。在生产环境中使用前，务必进行充分测试。

**学习曲线不可忽视**。MCP 的概念本身并不复杂，但从"会写 Python"到"能搭建生产级 MCP 服务"仍有距离。异步编程、JSON-RPC 协议、服务器部署等环节都可能成为瓶颈。

**供应商锁定风险**。尽管 MCP 是开放标准，但不同 AI 供应商对 MCP 的支持程度和实现方式存在差异。选择 MCP 意味着要与特定版本、特定实现绑定，需要关注兼容性。

**安全性考量**。MCP 服务器可以访问本地资源（文件系统、环境变量等），这带来了安全隐患。在生产环境中，需要严格控制服务器权限，避免敏感数据泄露。

## 总结

MCP 作为 AI Agent 领域的开放标准，正在快速获得行业认可。从 Anthropic 的原创到 OpenAI、Microsoft 的跟进，再到 Linux Foundation 的托管，MCP 已经具备了成为行业基础设施的潜力。

通过本文的实战示例可以看到，MCP 的设计理念简洁清晰：定义标准化的通信协议，让 AI 应用与外部工具之间建立松耦合的连接。对于开发者而言，这意味着可以更专注于业务逻辑，而非重复造轮子。

## 实践建议

如果你准备在项目中引入 MCP，以下是几点实践经验：

第一，从简单场景开始。天气查询、文件读写等轻量级工具是理想的试点项目，可以快速验证 MCP 的工作机制。

第二，关注官方生态。Anthropic 维护着丰富的 MCP 服务器实现，包括 GitHub、Slack、Postgres 等常用工具，这些开源实现是学习的绝佳资源。

第三，渐进式迁移。如果现有系统已经实现了自定义工具接口，不必急于全面重构。可以先在新功能中使用 MCP，逐步积累经验后再考虑整体迁移。

## 延伸阅读

- [MCP 官方文档](https://modelcontextprotocol.io)
- [Anthropic MCP GitHub 仓库](https://github.com/anthropics/mcp)
- [Linux Foundation Agentic AI Foundation](https://foundation.agenticai.ai)
