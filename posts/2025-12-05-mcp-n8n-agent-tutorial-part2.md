---
title: "MCP Demystified: Build an Autonomous AI Agent with MCP and n8n"
author: "Matthew Dutchess"
date: "2025-12-02"
category: "AI"
excerpt: "Build a fully autonomous morning-planning agent that checks weather, reads your calendar, and sends daily briefings using MCP servers and n8n workflows."
tags: ["AI Agents", "MCP", "n8n", "Automation", "Python", "LLM"]
readTime: "12 min read"
---

# Build an Autonomous AI Agent with MCP and n8n

_This part 2 of a multi-part blog series_

**In this tutorial, you'll build a fully autonomous morning-planning agent** that checks weather, reads your calendar, and sends a daily briefing — using MCP servers and an n8n workflow.

By the end, you'll understand how to create production-ready AI agents that perceive, reason, and act.

---

## What is MCP and Why Use It?

**MCP (Model Context Protocol)** is Anthropic's open standard that lets LLMs discover and use external tools — think "USB standard for AI."

Go checkout out our previous article on [MCP Demystified: The Open Standard Powering the Next Wave of Agentic AI](https://1oceanlabs.com/blog/2025-11-17-mcp-introduction-part1) for more information on why you should use MCP.

## How This Tutorial Works

**The flow you'll build:**

```
┌─────────────────────────────────────────────────────────────┐
│ n8n Workflow (Orchestration Layer)                          │
│                                                             │
│  1. Schedule Trigger (7:00 AM daily)                        │
│         ↓                                                   │
│  2. AI Agent Node ◄── The LLM agent runs HERE inside n8n    │
│         ↓                                                   │
│     "Check weather and calendar, make recommendations"      │
│         ↓                                                   │
│  3. Agent calls MCP tools:                                  │
│         • get_forecast("New York")                          │
│         • check_events("2025-12-02")                        │
│         ↓                                                   │
│  4. MCP Servers fetch data:                                 │
│         • Weather Server → OpenWeather API                  │
│         • Calendar Server → Google Calendar API             │
│         ↓                                                   │
│  5. Agent reasons about data                                │
│         ↓                                                   │
│  6. Sends notification (Email/Slack)                        │
└─────────────────────────────────────────────────────────────┘
```

**⚠️ Important:** The AI agent does **not** run as a standalone server. The agent lives inside n8n's **AI Agent node** and orchestrates MCP tools directly during workflow execution. n8n handles the LLM API calls, tool discovery, and execution flow.

**Key components:**
- **n8n**: Schedules and orchestrates the workflow, hosts the AI Agent node
- **AI Agent node**: Contains the LLM (Claude/GPT) that decides when to use tools
- **MCP Servers**: Expose tools to the LLM (weather, calendar)
- **External APIs**: Real data sources

**What makes this "agentic":**
The LLM autonomously decides which tools to call, in what order, based on the task. You don't hardcode "call weather, then calendar" — the agent figures it out.

---

## What You'll Build

A weather assistant that:
- **Fetches** real-time weather data
- **Analyzes** your calendar for conflicts
- **Decides** what actions to take
- **Notifies** you with personalized recommendations

**Tech stack:**
- **MCP** (Model Context Protocol) — Universal standard for connecting LLMs to tools
- **n8n** — Open-source workflow automation platform (think Zapier, but self-hostable). It provides the visual interface and scheduling for your AI agent.
- **Claude/GPT** — Any LLM that supports tool calling
- **Python** — For building MCP servers

**Architecture:**
```
Schedule Trigger → AI Agent (LLM) → MCP Servers (Weather/Calendar) → APIs
                      ↓
                  Notification (Email/Slack)
```

---

## Prerequisites

```bash
# Required
- Python 3.10+
- Node.js 18+
- Docker (for n8n)
- API keys: OpenWeatherMap, Anthropic/OpenAI

# Install MCP SDK (≥0.3 for WebSocket support)
pip install mcp anthropic httpx python-dotenv

# Start n8n
docker run -it --rm \
  --name n8n \
  -p 5678:5678 \
  -v ~/.n8n:/home/node/.n8n \
  n8nio/n8n
```

---

## Part 1: Build the Weather MCP Server

### Understanding MCP Transports

MCP servers communicate via **transports**:

- **stdio (standard input/output)**: Simple, perfect for local development. The server reads from stdin and writes to stdout.
- **WebSocket**: Production-ready, supports remote servers, multiplexing, and proper sandboxing.

**We'll start with stdio** for easy local testing, then upgrade to WebSocket for production deployment.

### Project Structure

```
mcp-agent/
├── servers/
│   ├── weather_server.py
│   └── calendar_server.py
├── .env
└── requirements.txt
```

### The Weather Server

This MCP server exposes two capabilities:
1. **Tool**: `get_forecast` — Fetch current weather (LLM calls this explicitly)
2. **Subscription**: `weather_alerts` — Stream weather changes (long-lived connection)

**💡 Tools vs Subscriptions:**
- **Tools** = One-time request/response (like calling a function)
- **Subscriptions** = Ongoing stream of events (like WebSocket updates)

```python
# servers/weather_server.py
import os
from mcp import Server
from mcp.server import stdio_server
import httpx
import asyncio
from typing import Dict, Any
from datetime import datetime
from dotenv import load_dotenv

load_dotenv()

server = Server("weather-assistant")

@server.tool()
async def get_forecast(city: str) -> Dict[str, Any]:
    """Fetch current weather forecast for a given city"""
    api_key = os.getenv("OPENWEATHER_API_KEY")
    url = "https://api.openweathermap.org/data/2.5/weather"
    
    params = {"q": city, "appid": api_key, "units": "metric"}
    
    try:
        async with httpx.AsyncClient() as client:
            response = await client.get(url, params=params, timeout=10.0)
            response.raise_for_status()
            data = response.json()
        
        temp = data["main"]["temp"]
        description = data["weather"][0]["description"]
        
        # Check severe conditions first, then temperature
        if "rain" in description.lower() or "storm" in description.lower():
            suggestion = "Bring an umbrella! Consider rescheduling outdoor plans."
        elif temp < 5:
            suggestion = "Bundle up! It's freezing."
        elif temp < 15:
            suggestion = "Grab a jacket."
        elif temp < 25:
            suggestion = "Nice weather for outdoor activities."
        else:
            suggestion = "Beautiful day outside!"
        
        return {
            "city": city,
            "temperature": temp,
            "feels_like": data["main"]["feels_like"],
            "description": description,
            "humidity": data["main"]["humidity"],
            "suggestion": suggestion,
            "status": "success"
        }
        
    except httpx.HTTPError as e:
        return {"error": str(e), "status": "error"}

@server.subscription()
async def weather_alerts(city: str):
    """Stream weather changes every 15 minutes"""
    api_key = os.getenv("OPENWEATHER_API_KEY")
    last_temp = None
    
    while True:
        try:
            async with httpx.AsyncClient() as client:
                response = await client.get(
                    "https://api.openweathermap.org/data/2.5/weather",
                    params={"q": city, "appid": api_key, "units": "metric"},
                    timeout=10.0
                )
                response.raise_for_status()
                data = response.json()
            
            current_temp = data["main"]["temp"]
            description = data["weather"][0]["description"]
            
            # Alert on significant temperature changes
            if last_temp and abs(current_temp - last_temp) > 3:
                yield {
                    "type": "temperature_change",
                    "city": city,
                    "change": current_temp - last_temp,
                    "timestamp": datetime.now().isoformat(),
                    "alert": f"Temperature changed by {abs(current_temp - last_temp):.1f}°C!"
                }
            
            # Alert on severe weather
            if any(kw in description.lower() for kw in ['storm', 'heavy', 'severe']):
                yield {
                    "type": "severe_weather",
                    "city": city,
                    "conditions": description,
                    "timestamp": datetime.now().isoformat()
                }
            
            last_temp = current_temp
            
        except Exception as e:
            yield {"type": "error", "message": str(e)}
        
        await asyncio.sleep(900)  # 15 minutes

if __name__ == "__main__":
    stdio_server(server)
```

**Key concepts:**
- **Tools** = Request/response functions the LLM can call
- **Subscriptions** = Long-lived streams that push updates (use `yield`, not `return`)
- **stdio transport** = Simple for local development (we'll upgrade to WebSocket for production)

### Quick Test Your MCP Server

Before connecting to n8n, test your server locally:

```bash
# Install MCP CLI tool
npm install -g @modelcontextprotocol/inspector

# Test the weather server
npx @modelcontextprotocol/inspector python servers/weather_server.py

# Or use mcp-cli if you have it installed
mcp-cli python servers/weather_server.py
```

This opens an interactive inspector where you can:
- See all available tools and resources
- Call tools with test parameters
- View responses in real-time

**Quick win:** Try calling `get_forecast` with `city: "New York"` to verify everything works before integrating with n8n.

---

## Part 2: Build the Calendar MCP Server

### Understanding Resources

MCP has three types of capabilities:

1. **Tools**: Functions the LLM explicitly calls (`get_forecast("NYC")`)
2. **Subscriptions**: Long-lived streams that push updates
3. **Resources**: Automatic context injection (LLM reads without calling)

**Resources are powerful** because they're automatically included in the LLM's context on **every conversation turn**. The LLM can reference this data without making explicit tool calls.

**Example:**
- **Without resources**: LLM must call `get_calendar_events()` to see today's schedule
- **With resources**: Today's events are already in context — the LLM just reads them

Think of resources as "background knowledge" that's always available to the agent.

### The Calendar Server

**⚠️ Note on Mock Data:** This example uses hardcoded events for `2025-12-02` to keep the tutorial focused. In production, you'd integrate with Google Calendar API using OAuth. The patterns shown here work identically with real APIs.

```python
# servers/calendar_server.py
import os
from mcp import Server
from mcp.server import stdio_server
from datetime import datetime, timedelta
from typing import Dict, List, Any

server = Server("calendar-assistant")

@server.tool()
async def check_events(date: str) -> Dict[str, List[Dict[str, Any]]]:
    """Check calendar events for a specific date (YYYY-MM-DD)"""
    # In production: use Google Calendar API with OAuth
    # For demo: mock data
    mock_events = {
        "2025-12-02": [
            {"time": "09:00", "title": "Team standup", "location": "Office"},
            {"time": "14:00", "title": "Client meeting", "location": "Zoom"},
            {"time": "18:00", "title": "Gym session", "location": "Downtown Fitness"}
        ]
    }
    
    events = mock_events.get(date, [])
    return {
        "date": date,
        "events": events,
        "count": len(events)
    }

@server.tool()
async def suggest_reschedule(event_title: str, reason: str) -> Dict[str, Any]:
    """Suggest rescheduling an event based on weather/conflicts"""
    # In production: integrate with calendar API
    return {
        "event": event_title,
        "reason": reason,
        "suggestion": "Consider rescheduling to tomorrow afternoon",
        "status": "suggestion_generated"
    }

@server.resource("calendar-today")
async def todays_calendar() -> Dict[str, Any]:
    """Automatic context: today's events (no tool call needed)"""
    today = datetime.now().strftime("%Y-%m-%d")
    
    mock_events = {
        "2025-12-02": [
            {"time": "09:00", "title": "Team standup"},
            {"time": "14:00", "title": "Client meeting"},
            {"time": "18:00", "title": "Gym session"}
        ]
    }
    
    return {
        "date": today,
        "events": mock_events.get(today, []),
        "note": "Auto-injected context"
    }

@server.resource("calendar-preferences")
async def user_preferences() -> Dict[str, Any]:
    """User scheduling preferences"""
    return {
        "working_hours": {"start": "09:00", "end": "17:00"},
        "buffer_between_meetings": 15,
        "lunch_time": {"start": "12:00", "end": "13:00"}
    }

if __name__ == "__main__":
    stdio_server(server)
```

**Resources vs Tools:**
- **Tools**: LLM explicitly calls them (`get_forecast(city="NYC")`)
- **Resources**: Auto-injected as context (LLM reads without calling)

---

## Part 3: Build the n8n Workflow

### Step 1: Create Workflow

1. Open n8n at http://localhost:5678
2. Create new workflow: "Weather Assistant Agent"

### Step 2: Schedule Trigger

Add **Schedule Trigger** node:
- Interval: Daily
- Time: 07:00 AM

### Step 3: Configure the AI Agent Node

Add **AI Agent** node and configure:

#### 3.1 Model Configuration

- **Provider**: Claude (Anthropic) or GPT-4 (OpenAI)
- **Model**: `claude-sonnet-4-5-20250929` or `gpt-4-turbo`
- **API Key**: Your key from environment

#### 3.2 Agent Prompt

```
You are a morning planning assistant with access to weather and calendar tools.

Your process:
1. Check weather for the user's city
2. Review today's calendar
3. Identify weather-impacted activities
4. Generate actionable recommendations

Always explain your reasoning clearly.
```

#### 3.3 Define Structured Output Schema

Add JSON schema to ensure predictable output:

```json
{
  "type": "object",
  "properties": {
    "analysis": {
      "type": "string",
      "description": "Your reasoning about weather and calendar"
    },
    "weather_summary": {
      "type": "object",
      "properties": {
        "conditions": {"type": "string"},
        "temperature": {"type": "number"},
        "impact_level": {"type": "string", "enum": ["low", "medium", "high"]}
      }
    },
    "recommendations": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "action": {"type": "string"},
          "priority": {"type": "string", "enum": ["low", "medium", "high"]},
          "reasoning": {"type": "string"}
        }
      }
    }
  },
  "required": ["analysis", "recommendations"]
}
```

#### 3.4 Connect MCP Tools

In AI Agent node, add **MCP Client Tools**:

**Weather Tool (Local Development):**
- Command: `python /path/to/servers/weather_server.py`
- Transport: stdio
- Discovery: Auto

**Calendar Tool (Local Development):**
- Command: `python /path/to/servers/calendar_server.py`
- Transport: stdio
- Discovery: Auto

**Important notes:**

⚠️ **Explicit configuration required:** n8n does not automatically scan the network for MCP servers. You must explicitly configure each MCP server (Weather, Calendar) in the AI Agent node. The "Auto" discovery setting means n8n will discover the tools/resources that server exposes, not the servers themselves.

✅ **LLM compatibility:** Both Claude and GPT-4 support MCP-native tool calling. n8n abstracts away any provider-specific differences, so the same MCP server works with any LLM.

**Note:** For production, use WebSocket transport (covered in deployment section).

#### 3.5 Add Decision Logic

Add **IF** node after AI Agent:

```javascript
{{$json["weather_summary"]["temperature"] < 10 || 
  $json["weather_summary"]["conditions"].toLowerCase().includes("rain")}}
```

- **True**: High-priority notification
- **False**: Standard notification

### Step 4: Send Notification

Add **Email** or **Slack** node:

```
Subject: Your Daily Briefing for {{$now.format("MMM D, YYYY")}}

{{$json["analysis"]}}

Recommendations:
{{$json["recommendations"].map(r => `• ${r.action} (${r.priority})`).join("\n")}}
```

### Step 5: Test

1. Click "Execute Workflow"
2. Check execution log for:
   - MCP server connections
   - Tool calls
   - AI reasoning
   - Final notification

**Debug tip:** Enable "Verbose Logging" in AI Agent node settings.

---

## Part 4: Production Deployment

### Quick Start: Minimal Production Setup

**Goal:** Get your agent running in production with WebSocket transport and Docker.

**Key changes from development:**
- ✅ stdio → WebSocket transport
- ✅ Local file paths → Network URLs
- ✅ Manual execution → Containerized services

### Step 1: Upgrade to WebSocket MCP Server

```python
# servers/weather_server_ws.py
from fastapi import FastAPI, WebSocket
from mcp import Server
from mcp.server.websocket import WebSocketTransport
import uvicorn
import os
from dotenv import load_dotenv

load_dotenv()

app = FastAPI()
server = Server("weather-assistant")

# Copy all your @server.tool() and @server.subscription() definitions here

@app.websocket("/mcp")
async def mcp_endpoint(websocket: WebSocket):
    await websocket.accept()
    transport = WebSocketTransport(websocket)
    await server.run(transport)

@app.get("/health")
async def health_check():
    return {"status": "healthy"}

if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

### Step 2: Dockerize Your MCP Server

**Why Docker?** Ensures consistent environments, easy deployment, and portability across cloud providers.

```dockerfile
# Dockerfile
FROM python:3.11-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Health check dependencies
RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*

COPY servers/ ./servers/
ENV PYTHONUNBUFFERED=1

EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=3s \
  CMD curl -f http://localhost:8000/health || exit 1

CMD ["uvicorn", "servers.weather_server_ws:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Step 3: Update n8n for Production

In n8n MCP tool configuration, change:
- **Command**: `python /path/to/servers/weather_server.py` 
- **Transport**: stdio

To:
- **URL**: `ws://weather-mcp:8000/mcp`
- **Transport**: WebSocket
- **Authentication**: (Add if needed)

**Test it:**
```bash
# Start your Docker container
docker run -p 8000:8000 your-registry/weather-mcp:latest

# Execute workflow in n8n
# Check logs for WebSocket connections
```

---

### Advanced: Kubernetes Deployment

**When to use Kubernetes:**
- You need high availability (multiple replicas)
- You're running multiple MCP servers
- You need auto-scaling based on load
- You want built-in load balancing and health monitoring

**For simpler deployments,** Docker + a single server is often sufficient. Use K8s when scale and reliability are critical.

```yaml
# k8s/mcp-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: weather-mcp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: weather-mcp
  template:
    metadata:
      labels:
        app: weather-mcp
    spec:
      containers:
      - name: weather-mcp
        image: your-registry/weather-mcp:latest
        ports:
        - containerPort: 8000
        env:
        - name: OPENWEATHER_API_KEY
          valueFrom:
            secretKeyRef:
              name: mcp-secrets
              key: openweather-api-key
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 10
          periodSeconds: 30
---
apiVersion: v1
kind: Service
metadata:
  name: weather-mcp
spec:
  selector:
    app: weather-mcp
  ports:
  - port: 8000
    targetPort: 8000
  type: LoadBalancer
```

---

## Part 5: Advanced Patterns

### Security: Tool Permissions

```python
from mcp.server import ToolPermission

@server.tool(permissions=[ToolPermission.REQUIRES_CONFIRMATION])
async def send_email(to: str, subject: str, body: str) -> Dict[str, Any]:
    """Send email - requires user confirmation"""
    return {
        "status": "ok",
        "message": "Email queued (demo mode)",
        "to": to
    }

@server.tool(permissions=[ToolPermission.SIGNED_ONLY])
async def delete_event(event_id: str) -> Dict[str, Any]:
    """Delete event - requires signed request"""
    return {
        "status": "ok",
        "message": "Deletion logged (demo mode)",
        "event_id": event_id
    }
```

**Permission levels:**
- `PUBLIC`: Read-only, anyone can call
- `REQUIRES_CONFIRMATION`: Agent asks user first
- `SIGNED_ONLY`: Cryptographic signature required
- `ADMIN_ONLY`: Privileged agents only

### Standardized Tool Outputs

For better agent decision-making:

```python
@server.tool()
async def get_forecast(city: str) -> Dict[str, Any]:
    start_time = time.time()
    
    try:
        # ... fetch weather ...
        return {
            "ok": True,
            "data": {"city": city, "temperature": temp, "conditions": desc},
            "error": None,
            "observations": [{
                "execution_time_ms": (time.time() - start_time) * 1000,
                "data_freshness": "real-time",
                "confidence": 0.95
            }]
        }
    except Exception as e:
        return {
            "ok": False,
            "data": None,
            "error": str(e),
            "observations": [{"confidence": 0.0}]
        }
```

Agents can now:
- Check `ok` before parsing data
- Understand reliability via `observations`
- Make better decisions

### Multi-Agent Pattern

**When to use:** For complex tasks that require specialized expertise or when a single agent becomes too general-purpose.

**Example use cases:**
- Research tasks (one agent searches, another analyzes, another synthesizes)
- Customer support (triage agent → specialist agents by department)
- Code generation (architect agent → implementation agents → testing agent)

```python
# Supervisor delegates to specialized workers
@supervisor_server.tool()
async def delegate_task(task_type: str, params: dict) -> Dict[str, Any]:
    if task_type == "weather_analysis":
        return await call_weather_agent(params)
    elif task_type == "scheduling":
        return await call_calendar_agent(params)
    return {"error": "Unknown task type"}
```

**Agent hierarchy:**
- **Supervisor**: Plans and delegates
- **Weather Worker**: Meteorological analysis
- **Calendar Worker**: Scheduling logic
- **Memory Worker**: Vector store integration

---

## Common Pitfalls

### 1. Rate Limits

```python
from collections import deque
import time

rate_limiter = deque(maxlen=10)

@server.tool()
async def get_forecast(city: str) -> Dict[str, Any]:
    # Allow 10 calls per minute
    now = time.time()
    rate_limiter.append(now)
    
    if len(rate_limiter) == 10 and now - rate_limiter[0] < 60:
        return {"error": "Rate limit exceeded", "status": "error"}
    
    # ... rest of function ...
```

### 2. Agent Not Using Tools

**Fix:** Make prompt more directive
```
You MUST use the get_forecast tool before responding.
Do not make assumptions about weather.
```

### 3. WebSocket Connection Failures

```bash
# Verify server is running
curl http://localhost:8000/health

# Test WebSocket
wscat -c ws://localhost:8000/mcp

# Check logs
docker logs weather-mcp
```

---

## Conclusion

You've built a production-ready AI agent that:
- ✅ Connects to external data via MCP
- ✅ Makes intelligent decisions with LLMs
- ✅ Takes autonomous actions
- ✅ Runs reliably in production

**This pattern scales** to:
- Customer support agents (CRM integration)
- DevOps agents (infrastructure monitoring)
- Research agents (web search + synthesis)
- Sales agents (lead qualification + scheduling)

The future of AI is autonomous systems that perceive, reason, and act. You just built one. 🚀

---

## Resources

- [MCP Documentation](https://modelcontextprotocol.io/)
- [n8n Documentation](https://docs.n8n.io/)
- [MCP Python SDK](https://github.com/anthropics/mcp-python)
- [Full Code Repository](https://github.com/example/weather-mcp-agent)

---

## Where We're Going Next (Part 3 and beyond)

Multi-Agent Supply Chain Simulator:
- Watch multiple MCP-connected agents collaborate in real-time.

---

## Closing

**Want to extend this?** Try building:
- IoT home automation agent
- Fraud detection agent for transactions
- Content moderation agent for social media

Share what you build — we'd love to see what you create!

---

Crafted with ❤️ by One Ocean Labs