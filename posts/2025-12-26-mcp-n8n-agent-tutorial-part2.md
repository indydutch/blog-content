---
title: "MCP Demystified: Build an Autonomous AI Agent with MCP and n8n"
author: "Matthew Dutchess"
date: "2025-12-26"
category: "AI"
excerpt: "Build a fully autonomous morning-planning agent that checks weather, reads your calendar, and sends daily briefings using MCP servers and n8n workflows."
tags: ["AI Agents", "MCP", "n8n", "Automation", "Python", "LLM"]
readTime: "20 min read"
---

# Build an Autonomous AI Agent with MCP and n8n

_This is Part 2 of a multi-part blog series_

**In this tutorial, you'll build a fully autonomous morning-planning agent** that checks weather, reads your calendar, and sends a daily briefing — using MCP servers and an n8n workflow.

By the end, you'll understand how to create production-ready AI agents that perceive, reason, and act.

**What's included:**
- ✅ Complete, tested MCP server code (Weather + Calendar)
- ✅ Importable n8n workflow JSON
- ✅ Docker Compose for one-command deployment
- ✅ Step-by-step setup instructions

---

## What is MCP and Why Use It?

**MCP (Model Context Protocol)** is Anthropic's open standard that lets LLMs discover and use external tools — think "USB-C port for AI."

### Breaking Down the "USB-C for AI" Analogy

Just like USB-C provides a universal way to connect any device (phone, laptop, monitor) to any accessory (charger, keyboard, external drive), MCP provides a universal way to connect any AI model to any external capability. Before USB-C, you needed different cables for everything. Before MCP, developers had to write custom integrations for every tool an AI needed to use. MCP standardizes this, meaning:

- **Any MCP-compatible AI** can use **any MCP server** without custom code
- Tool creators build once, and every AI can use it
- You can mix and match tools from different sources

Go checkout out our previous article on [MCP Demystified: The Open Standard Powering the Next Wave of Agentic AI](https://1oceanlabs.com/blog/2025-11-17-mcp-introduction-part1) for more information on why you should use MCP.

---

## Architecture Overview

Before diving into the diagram, let's understand what we're building: a system where an AI agent wakes up every morning, checks the weather and your calendar, figures out if there are any conflicts (like an outdoor meeting on a rainy day), and sends you a personalized briefing. The magic is that *you don't program the logic*—the AI decides what to do based on the situation.

```
┌─────────────────────────────────────────────────────────────────┐
│                     n8n Workflow Engine                         │
│                                                                 │
│  ┌──────────────┐     ┌─────────────────┐     ┌───────────────┐ │
│  │    Daily     │───▶│    AI Agent      │───▶│  Send Email/  │ │
│  │   Trigger    │     │   (GPT-4/Claude)│     │    Slack      │ │
│  └──────────────┘     └────────┬────────┘     └───────────────┘ │
│                                │                                │
│                                │ MCP Tool Calls                 │
│                                ▼                                │
│              ┌─────────────────┴────────────────┐               │
│              │                                  │               │
│              ▼                                  ▼               │
│     ┌─────────────────┐               ┌─────────────────┐       │
│     │  Weather MCP    │               │  Calendar MCP   │       │
│     │    Server       │               │    Server       │       │
│     │  (Port 8000)    │               │  (Port 8001)    │       │
│     └────────┬────────┘               └────────┬────────┘       │
└──────────────┼─────────────────────────────────┼────────────────┘
               │                                 │
               ▼                                 ▼
      ┌─────────────────┐               ┌─────────────────┐
      │  OpenWeatherMap │               │  Mock Calendar  │
      │      API        │               │  (or Google)    │
      └─────────────────┘               └─────────────────┘
```

![n8n Workflow Engine Diagram](https://github.com/indydutch/blog-content/blob/main/images/n8n-workflow-diagram.png)

### Understanding Each Component

Here's what each piece does:

| Component | What It Is | What It Does |
|-----------|-----------|--------------|
| **n8n Workflow Engine** | An automation platform (like Zapier, but open-source and self-hosted) | Orchestrates the entire flow—triggers the agent, passes data between steps, sends the final email |
| **Daily Trigger** | A scheduled timer | Kicks off the workflow at the same time every day (like a cron job) |
| **AI Agent** | GPT-4 or Claude with tool-calling abilities | The "brain" that decides what data to fetch and how to synthesize it into a useful briefing |
| **MCP Servers** | Small web services that expose tools | Translators between the AI and external services (weather APIs, calendars, etc.) |
| **Weather MCP Server** | Python service running on port 8000 | Wraps the OpenWeatherMap API so the AI can ask "What's the weather in Tampa?" |
| **Calendar MCP Server** | Python service running on port 8001 | Provides access to your schedule so the AI can see today's meetings |

**What makes this "agentic":** The LLM autonomously decides which tools to call and in what order based on the task—you don't hardcode the logic.

Think of it this way: you don't write code that says "first get weather, then get calendar, then compare them." Instead, you give the AI a goal ("create a morning briefing") and access to tools. The AI figures out the steps itself. This is what separates an "agent" from a simple chatbot.

---

## Prerequisites

You'll need:

- **Docker & Docker Compose** — [Install Docker](https://docs.docker.com/get-docker/)
- **OpenWeatherMap API Key** (free) — [Get API Key](https://openweathermap.org/api)
- **OpenAI API Key** — [Get API Key](https://platform.openai.com/api-keys)
- **~15 minutes** for setup

### What is Docker and Why Do We Need It?

If you're new to Docker, here's a quick primer: Docker is a tool that packages applications into "containers"—self-contained environments that include everything the app needs to run (code, libraries, system tools). Think of it like shipping containers for software:

- **Without Docker**: "It works on my machine!" → teammate can't run it because they have different Python versions, missing libraries, etc.
- **With Docker**: Your app runs identically everywhere because it brings its own environment

**Docker Compose** takes this further by letting you define *multiple* containers that work together (in our case: n8n + weather server + calendar server) and start them all with one command.

> **Note:** The calendar server uses mock data by default. For real Google Calendar integration, you'll need to set up Google Cloud OAuth credentials (instructions in `.env.example`).

---

## Quick Start (5 Minutes)

### Step 1: Clone the Repository

```bash
git clone https://github.com/oneoceanlabs/mcp-agent-project
cd mcp-agent-project
```

**What these commands do:**
- `git clone` downloads a copy of the project from GitHub to your computer
- `cd` ("change directory") moves you into the project folder

**Repository structure:**
```
mcp-agent-project/
├── servers/
│   ├── weather_server.py      # Weather MCP server
│   └── calendar_server.py     # Calendar MCP server
├── docker-compose.yml          # One-command deployment
├── Dockerfile.mcp             # MCP server container
├── requirements.txt           # Python dependencies
├── .env.example              # Environment template
├── n8n-workflow.json         # Importable n8n workflow
└── README.md
```

**Understanding the file structure:**
- `servers/` contains the Python code for our MCP servers
- `docker-compose.yml` defines how all our services connect together
- `Dockerfile.mcp` tells Docker how to build a container for our Python servers
- `requirements.txt` lists Python packages we need (like a shopping list for pip)
- `.env.example` is a template for your secret keys (API keys, passwords)
- `n8n-workflow.json` is the pre-built workflow you'll import into n8n

[📂 View complete repository →](https://github.com/oneoceanlabs/mcp-agent-project)

### Step 2: Configure Environment Variables

```bash
cp .env.example .env
```

This command copies the example environment file to create your actual `.env` file. Environment variables are how we safely store secrets like API keys—they stay on your machine and never get committed to Git.

Add your API keys to `.env`:

```bash
OPENWEATHER_API_KEY=your_openweather_api_key_here
OPENAI_API_KEY=your_openai_api_key_here
N8N_ENCRYPTION_KEY=random_32_char_string
```

**Where to get these keys:**
- **OPENWEATHER_API_KEY**: Sign up at [openweathermap.org](https://openweathermap.org/api), go to "API Keys" in your account
- **OPENAI_API_KEY**: From [platform.openai.com/api-keys](https://platform.openai.com/api-keys)
- **N8N_ENCRYPTION_KEY**: Generate a random string (you can use `openssl rand -hex 16` in your terminal)

### Step 3: Start Everything

```bash
# Build and start all services
docker-compose up -d

# Check status
docker-compose ps
```

**What's happening here:**
- `docker-compose up` reads the `docker-compose.yml` file and starts all defined services
- The `-d` flag runs everything in "detached" mode (in the background, so you get your terminal back)
- `docker-compose ps` shows you the status of all running containers

This starts:
- **n8n** (http://localhost:5678) — The workflow automation interface
- **Weather MCP Server** (http://localhost:8000) — Our weather tool server
- **Calendar MCP Server** (http://localhost:8001) — Our calendar tool server

### Step 4: Import the Workflow

1. Open n8n at http://localhost:5678
2. Login with `admin` / `your_password` (from .env)
3. Create a new workflow
4. Click the 3 dots on the upper task bar → **Import from file...**
5. Select `n8n-workflow.json`
6. Configure credentials
7. Click **Execute Workflow**

That's it! Your agent will run daily.

---

## How It Works: MCP Servers

### Understanding MCP Server Capabilities

MCP servers expose three types of capabilities to LLMs. Think of these as different ways the AI can interact with your server:

**1. Tools** — Functions the LLM explicitly calls

Tools are like functions in a restaurant menu—the AI looks at what's available, picks what it needs, and explicitly orders it. The AI *chooses* when to call a tool.

```python
@mcp.tool()
async def get_forecast(city: str) -> dict[str, Any]:
    """Fetch current weather for a city"""
    # Implementation fetches from OpenWeatherMap API
```

**2. Resources** — Context automatically injected into every LLM conversation

Resources are like the bread basket that arrives at your table automatically—you didn't order it, but it's there if you want it. Resources provide background information the AI can reference without making an explicit call.

```python
@mcp.resource("weather://conditions")
async def current_conditions():
    """Background weather data the LLM can reference without calling"""
    return {"temperature": 72, "condition": "sunny"}
```

**3. Prompts** — Reusable templates with placeholders

Prompts are like form letters with blanks to fill in—pre-written templates that can be customized with specific data.

```python
@mcp.prompt()
def daily_briefing_template():
    """Template: 'Based on {weather} and {calendar}, suggest...'"""
```

---

## The Weather MCP Server

Our weather server uses **FastMCP** (a simplified MCP framework) and provides three tools:

1. **`get_forecast(city)`** — Basic weather lookup
2. **`get_forecast_multi(cities)`** — Batch city lookups
3. **`check_outdoor_conditions(city)`** — Activity recommendations

Plus a **resource** for automatic help documentation injection.

### Core Tool Implementation

Here's how a basic MCP tool is structured. Don't worry if some of this syntax is new—we'll break it down:

```python
from fastmcp import FastMCP
import httpx
import os

mcp = FastMCP("Weather Assistant")

@mcp.tool()
async def get_forecast(city: str) -> dict[str, Any]:
    """
    Fetch current weather forecast for a given city.
    
    Args:
        city: Name of the city (e.g., "New York", "London")
        
    Returns:
        Weather data with temperature, conditions, and smart recommendations
    """
    api_key = os.getenv("OPENWEATHER_API_KEY")
    
    async with httpx.AsyncClient() as client:
        response = await client.get(
            "https://api.openweathermap.org/data/2.5/weather",
            params={"q": city, "appid": api_key, "units": "metric"}
        )
        data = response.json()
    
    # Generate context-aware suggestions
    suggestions = []
    if "rain" in data["weather"][0]["description"].lower():
        suggestions.append("Bring an umbrella - rain expected")
    if data["main"]["temp"] < 10:
        suggestions.append("Cold weather - wear a warm coat")
    
    return {
        "city": city,
        "temperature": round(data["main"]["temp"], 1),
        "description": data["weather"][0]["description"].title(),
        "suggestions": suggestions,
        "status": "success"
    }
```

**Pro Tip: Why FastMCP?** We're using **FastMCP** here, a high-level framework that simplifies server creation. It abstracts away about 50 lines of boilerplate code you'd normally need to set up a standard MCP server from scratch. While the official Anthropic docs show the "standard" way, FastMCP is the go-to for rapid prototyping and clean production code.

### Breaking Down the Code

Let's demystify some of the Python concepts used here:

**The `@mcp.tool()` decorator:**
The `@` symbol in Python is a "decorator"—a way to modify or enhance a function. Think of it like putting a sticky note on a function that says "Hey MCP framework, register this function as a tool!" When the AI asks "what tools do you have?", MCP knows to include this function in the list.

**`async def` and `await`:**
These keywords enable "asynchronous" programming. Here's the simple version: when your code needs to wait for something slow (like a network request), `async`/`await` lets Python do other work instead of just sitting there. Imagine ordering coffee—instead of standing at the counter staring until it's ready, you can check your phone. That's async. The `await` keyword is where you say "pause here until this thing finishes."

**`httpx.AsyncClient()`:**
This is a modern HTTP library for Python—it's what makes the actual web request to OpenWeatherMap. The `async with` part ensures the connection is properly cleaned up when we're done.

**Type hints (`city: str`, `-> dict[str, Any]`):**
These tell Python (and other developers) what types of data the function expects and returns. `city: str` means "city should be a string." `-> dict[str, Any]` means "this function returns a dictionary." They're not strictly required, but they make code easier to understand and help catch bugs.

**Key features:**
- FastMCP framework for simplified server creation
- Async operations with httpx
- Smart suggestions based on temperature, precipitation, wind, and humidity
- Comprehensive error handling (404 for city not found, network errors)

[📄 View complete weather_server.py →](https://github.com/oneoceanlabs/mcp-agent-project/blob/main/servers/weather_server.py)

### Advanced Tool: Outdoor Activity Scoring

The `check_outdoor_conditions` tool evaluates weather suitability for outdoor activities:

```python
@mcp.tool()
async def check_outdoor_conditions(city: str) -> dict[str, Any]:
    """
    Check if weather conditions are suitable for outdoor activities.
    Returns a score (0-100) and specific recommendations.
    """
    weather = await get_forecast(city)
    
    score = 100
    concerns = []
    recommended_activities = []
    
    # Temperature scoring
    if weather["temperature"] < 0:
        score -= 40
        concerns.append("Freezing temperatures")
    elif weather["temperature"] > 35:
        score -= 35
        concerns.append("Extreme heat")
    
    # Precipitation scoring
    if "rain" in weather["description"].lower():
        score -= 30
        concerns.append("Rain expected")
    
    return {
        "outdoor_score": max(0, min(100, score)),
        "rating": "Excellent" if score >= 80 else "Good",
        "recommendation": "Perfect for outdoor activities!",
        "concerns": concerns,
        "recommended_activities": ["walking", "cycling"]
    }
```

**What's happening here:** This function calls our existing `get_forecast` function (notice the `await` since it's async), then applies scoring rules to determine how suitable the weather is for outdoor activities. The `max(0, min(100, score))` is a clever way to ensure the score stays between 0 and 100—it can't go negative or above 100 no matter how many deductions we make.

**Scoring factors:**
- Temperature (freezing: -40, extreme heat: -35, cold: -20)
- Precipitation (storms: -50, rain: -30, snow: -25)
- Wind speed (high winds: -20, windy: -10)
- Humidity (very humid: -10)

The tool returns activity recommendations based on the final score (0-100).

#### The Agentic Edge: Why hardcode a score?
You might notice we’ve hardcoded specific logic for the "Outdoor Score" (e.g., -30 for rain). In a traditional app, this is where the flexibility ends.

**The MCP Advantage:** Because this data is fed to an LLM, the agent can reason beyond these numbers. If a user says, "I'm a professional storm chaser," the agent can see our "Not Recommended" rating and correctly pivot to: _"The conditions are dangerous for most, but perfect for your chase. Here is the local wind shear data."_ > We provide the **structured baseline**; the Agent provides the **unstructured reasoning**.

### Resource: Built-in Documentation

The server includes a help resource that's automatically available to the LLM:

```python
@mcp.resource("weather://help")
def weather_help() -> str:
    """Help information about available weather tools."""
    return """Weather Assistant - Available Tools:

1. get_forecast(city) - Current weather for any city
2. get_forecast_multi(cities) - Batch weather lookup (max 10)
3. check_outdoor_conditions(city) - Outdoor suitability score

Tips:
- Use full city names for best results
- Add country code for ambiguous cities: "Paris, FR"
"""
```

This documentation is injected into the LLM's context automatically, helping it understand tool capabilities without explicit prompting. It's like giving the AI a quick reference card about what your server can do.

---

## The Calendar MCP Server

The calendar server demonstrates MCP **resources** — automatic context injection.

### Resources vs Tools

Here's the key difference illustrated with code:

**Without resources (requires explicit tool call):**
```python
@mcp.tool()
async def get_todays_events() -> dict:
    """LLM must explicitly call this"""
    return {"events": [...]}
```

With this approach, the AI has to *decide* to call the function. If the AI doesn't think to ask about today's events, it won't have that information.

**With resources (automatic context):**
```python
@mcp.resource("calendar://today")
def todays_events() -> str:
    """Automatically available in LLM context on every turn"""
    return json.dumps({"events": [...]})
```

With resources, the data is already there—the AI sees it without asking. It's the difference between having to Google something vs. already knowing it.

The LLM can reference today's calendar without making a tool call — it's already in context.

### Calendar Tool Implementation

```python
@mcp.tool()
async def check_events(date: str) -> dict[str, Any]:
    """
    Check calendar events for a specific date.
    
    Args:
        date: Date in YYYY-MM-DD format
    
    Returns:
        List of events with times and locations
    """
    # In production: Google Calendar API with OAuth
    # For demo: mock data
    mock_events = {
        "2025-12-15": [
            {"time": "09:00", "title": "Team standup"},
            {"time": "14:00", "title": "Client meeting"},
        ]
    }
    
    return {
        "date": date,
        "events": mock_events.get(date, []),
        "count": len(mock_events.get(date, []))
    }
```

**About the mock data:** In a production system, this would connect to Google Calendar via OAuth. For learning purposes, we're using hardcoded sample events. The `.get(date, [])` pattern is a safe way to look up a dictionary key—if the date doesn't exist, it returns an empty list instead of crashing.

[📄 View complete calendar_server.py →](https://github.com/oneoceanlabs/mcp-agent-project/blob/main/servers/calendar_server.py)

### Resource: User Preferences

Preferences are automatically injected as context:

```python
@mcp.resource("calendar://preferences")
async def user_preferences() -> str:
    """User scheduling preferences (auto-injected context)"""
    return json.dumps({
        "working_hours": {"start": "09:00", "end": "17:00"},
        "buffer_between_meetings": 15,
        "lunch_time": {"start": "12:00", "end": "13:00"}
    })
```

The LLM sees these preferences on every turn and can factor them into scheduling decisions. For example, if the AI is suggesting a reschedule time, it already knows not to suggest noon (lunch time) or 6 PM (after working hours).

---

## The n8n Workflow

### What is n8n?

If you're new to n8n, it's a workflow automation tool—think of it as a visual programming environment where you connect boxes (called "nodes") to create automations. Each node does one thing: fetch data, transform it, send an email, call an API, etc. You connect them together like LEGO blocks to build complex automations without writing much code.

### Workflow Components

The workflow consists of these nodes:

1. **Daily Trigger** — Runs every 24 hours (configurable timing)
2. **Set User Context** — Defines city ("Tampa") and current date
3. **Morning Briefing Agent** — GPT-4.1 with MCP tool access
4. **Weather MCP Tools** — Connected to http://weather-mcp:8000/mcp
5. **Calendar MCP Tools** — Connected to http://calendar-mcp:8001/mcp
6. **Weather Alert Check** — IF node detecting keywords (rain, storm, reschedule)
7. **High Priority Format** / **Normal Priority Format** — Format email based on alert
8. **Send High Priority Email** / **Send Normal Email** — Deliver the briefing

**Execution flow:**
```
Daily Trigger → Set User Context → AI Agent (calls MCP tools) 
→ Weather Alert Check → [High Priority / Normal Priority] → Send Email
```

**Reading the flow:** Data flows left to right. The Daily Trigger starts things at 7 AM (or whenever you configure). It passes control to "Set User Context" which adds variables like city and date. The AI Agent receives this context, calls the MCP tools as needed, and produces a briefing. The IF node checks if there are any weather concerns, and routes to either a high-priority or normal-priority email formatter. Finally, the email is sent.

**Built-in documentation:** The workflow includes a sticky note with setup instructions and MCP server URL configuration. This appears as a yellow note in the n8n canvas when you import the workflow.

### AI Agent Configuration

The AI Agent node is configured with:

**Model settings:**
```json
{
  "model": "gpt-4.1",
  "temperature": 0.3  // Lower temp for more consistent, factual responses
}
```

**What is "temperature"?** In AI models, temperature controls randomness. At 0, the model always picks the most likely response (very deterministic). At 1, it's more creative and varied. For factual tasks like weather reports, we want consistency, so we use 0.3.

**System message:**
```
You are a helpful morning planning assistant. You have access to weather 
and calendar tools via MCP (Model Context Protocol).

Your job is to:
1. ALWAYS use the available tools to get real data - never make assumptions
2. Analyze the weather conditions and calendar events together
3. Identify any conflicts between weather and outdoor activities
4. Provide clear, actionable recommendations

Be concise but thorough. If weather is bad and there are outdoor events, 
suggest rescheduling.

IMPORTANT: You MUST call the tools to get data before responding. 
Do not guess or assume weather conditions.
```

**Why this matters:** The system message shapes the AI's behavior. Without explicit instructions to use tools, the AI might try to guess the weather based on its training data (which is outdated). The "MUST call the tools" instruction prevents this.

**Prompt template:**
```
Create a morning briefing for today ({{ $json.date }}) in {{ $json.city }}.

Your tasks:
1. First, check the weather forecast for {{ $json.city }}
2. Then, check today's calendar events
3. Identify any outdoor events that might be affected by weather
4. Provide personalized recommendations

Format your response as a friendly morning briefing with:
- Weather summary and what to wear
- Today's schedule overview
- Any weather-related concerns for outdoor events
- Top 3 actionable recommendations for the day
```

The prompt uses n8n expressions (the `{{ }}` syntax) to inject the city and date from the previous node. This is n8n's templating language—similar to how you might use `${variable}` in JavaScript.

### Connecting MCP Servers

In the AI Agent node, add **MCP Tool** connections:

**Weather MCP:**
- **Endpoint URL**: `http://weather-mcp:8000/mcp`

**Calendar MCP:**
- **Endpoint URL**: `http://calendar-mcp:8001/mcp`

**Important:** These URLs use Docker Compose service names (`weather-mcp`, `calendar-mcp`). When containers run on the same Docker network, they can reach each other by service name—Docker handles the networking. If running servers locally or on different hosts, adjust the URLs accordingly (e.g., `http://localhost:8000/mcp`).

The agent automatically discovers all available tools from these servers and can call them autonomously based on the prompt.

### User Context Setup

Before the AI agent runs, we set the context variables:

**Set User Context node:**
```json
{
  "city": "Tampa",
  "date": "{{ $now.format('yyyy-MM-dd') }}"
}
```

This allows you to easily change the target city or use dynamic dates. The values are then available to the agent via `{{ $json.city }}` and `{{ $json.date }}` in the prompt. The `$now.format()` function gives us today's date in the format the calendar server expects.

### Weather Alert Detection

Instead of requiring structured output, the workflow uses simple keyword detection to identify severe weather situations:

**IF Node - Weather Alert Check:**
```javascript
// Checks if agent output contains any of these keywords
$json.output.toLowerCase().includes("rain") OR
$json.output.toLowerCase().includes("storm") OR
$json.output.toLowerCase().includes("reschedule")
```

**Why keyword detection instead of structured output?** Asking an LLM to output perfect JSON every time adds complexity and can fail. Simple keyword detection is more robust—if the AI mentions "rain" anywhere in its response, we flag it as a weather alert. This is a pragmatic engineering choice.

**Branch logic:**
- **True** (alert detected) → High Priority Email with ⚠️ prefix
- **False** (normal conditions) → Normal Email with ☀️ prefix

This approach is simpler than structured schemas and works well for binary decisions. The agent's text response is checked for specific keywords that indicate weather concerns.

**High Priority Format:**
```json
{
  "priority": "high",
  "subject": "⚠️ Weather Alert: Morning Briefing - Dec 15, 2025",
  "briefing": "{{ agent_output }}"
}
```

**Normal Priority Format:**
```json
{
  "priority": "normal",
  "subject": "☀️ Morning Briefing - Dec 15, 2025",
  "briefing": "{{ agent_output }}"
}
```

This ensures the agent always returns data we can process in downstream nodes.

[📄 View complete workflow JSON →](https://github.com/oneoceanlabs/mcp-agent-project/blob/main/n8n-workflow.json)

---

## Docker Deployment

### Docker Compose Overview

Our `docker-compose.yml` orchestrates all services:

```yaml
services:
  n8n:
    image: n8nio/n8n:latest
    ports:
      - "5678:5678"
    environment:
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
    volumes:
      - n8n_data:/home/node/.n8n
    networks:
      - mcp-network

  weather-mcp:
    build:
      context: .
      dockerfile: Dockerfile.mcp
    command: python servers/weather_server.py
    ports:
      - "8000:8000"
    environment:
      - OPENWEATHER_API_KEY=${OPENWEATHER_API_KEY}
    networks:
      - mcp-network

  calendar-mcp:
    build:
      context: .
      dockerfile: Dockerfile.mcp
    command: python servers/calendar_server.py
    ports:
      - "8001:8001"
    networks:
      - mcp-network

networks:
  mcp-network:
    driver: bridge

volumes:
  n8n_data:
```

### Understanding Docker Compose YAML

Let's break down what each section means:

| Section | Purpose | Example |
|---------|---------|---------|
| `services` | Defines each container to run | n8n, weather-mcp, calendar-mcp |
| `image` | Uses a pre-built image from Docker Hub | `n8nio/n8n:latest` |
| `build` | Builds an image from a Dockerfile | Our custom MCP servers |
| `ports` | Maps container ports to your machine | `"8000:8000"` means localhost:8000 → container:8000 |
| `environment` | Sets environment variables inside the container | API keys, config |
| `volumes` | Persists data outside the container | n8n saves workflows here |
| `networks` | Defines how containers talk to each other | All on `mcp-network` |

**Key features:**
- All services on same Docker network for easy communication
- Volume persistence for n8n data (your workflows survive restarts)
- Health checks on MCP servers
- Automatic restarts

[📄 View complete docker-compose.yml →](https://github.com/oneoceanlabs/mcp-agent-project/blob/main/docker-compose.yml)

### MCP Server Dockerfile

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY servers/ ./servers/

# Health check endpoint
HEALTHCHECK --interval=30s --timeout=3s \
  CMD python -c "import httpx; httpx.get('http://localhost:8000/health')"

EXPOSE 8000

CMD ["python", "servers/weather_server.py"]
```

**Reading the Dockerfile:**
1. `FROM python:3.11-slim` — Start with a lightweight Python 3.11 image
2. `WORKDIR /app` — All following commands run from /app
3. `COPY requirements.txt .` — Copy our dependencies list
4. `RUN pip install...` — Install Python packages
5. `COPY servers/ ./servers/` — Copy our server code
6. `HEALTHCHECK` — Docker will ping this endpoint to verify the server is healthy
7. `EXPOSE 8000` — Document which port the container uses
8. `CMD` — The command to run when the container starts

[📄 View complete Dockerfile.mcp →](https://github.com/oneoceanlabs/mcp-agent-project/blob/main/Dockerfile.mcp)

---

## Testing Your Agent

### Test MCP Servers Independently

Before connecting to n8n, verify your servers work:

```bash
# Test weather server
curl http://localhost:8000/health

# Expected output: Health check returns 'healthy' status
{
  "status": "healthy",
  "service": "weather-mcp"
}

```

**What's `curl`?** It's a command-line tool for making HTTP requests. Think of it as a browser without the visual interface: you tell it a URL, and it shows you what the server returns.

### Test the Full Workflow

1. Open n8n at http://localhost:5678
2. Create a new workflow
3. Add any desired nodes or import a workflow JSON file
4. Click "Execute Workflow" manually
5. Check the execution log for:
   - ✅ MCP server connections
   - ✅ Tool calls (weather, calendar)
   - ✅ AI reasoning steps
   - ✅ Final notification

**Debug tips:**
- Enable "Verbose Logging" in AI Agent node
- Check Docker logs: `docker-compose logs weather-mcp`
- Use n8n's built-in debugger to inspect data between nodes

---

## Common Pitfalls & Solutions

### Issue 1: MCP Server Connection Failed

**Symptoms:** 
- n8n can't connect to MCP servers
- Error: "Could not connect to MCP server at http://weather-mcp:8000/mcp"

**Solutions:**
```bash
# 1. Verify servers are running
docker-compose ps

# Expected output:
# weather-mcp    running    0.0.0.0:8000->8000/tcp
# calendar-mcp   running    0.0.0.0:8001->8001/tcp
# n8n            running    0.0.0.0:5678->5678/tcp

# 2. Check server health
curl http://localhost:8000/health
curl http://localhost:8001/health

# 3. Check Docker network connectivity
docker-compose exec n8n ping weather-mcp
docker-compose exec n8n ping calendar-mcp

# 4. View server logs for errors
docker-compose logs weather-mcp
docker-compose logs calendar-mcp
```

**If running locally (not Docker Compose):**
Update MCP Tool node URLs in n8n:
- Weather: `http://localhost:8000/mcp` (not weather-mcp)
- Calendar: `http://localhost:8001/mcp` (not calendar-mcp)

### Issue 2: Workflow Import Failed

**Symptoms:**
- n8n shows "Invalid workflow format"
- Missing nodes or connections

**Solutions:**
1. **Ensure n8n version compatibility:**
   ```bash
   docker pull n8nio/n8n:latest
   docker-compose up -d n8n
   ```

2. **Check for required nodes:**
   - `@n8n/n8n-nodes-langchain.agent`
   - `@n8n/n8n-nodes-langchain.lmChatOpenAi`
   - `@n8n/n8n-nodes-langchain.mcpClientTool`

   These are included in n8n v1.19.0+

3. **Manual import steps:**
   - Copy `n8n-workflow.json` content
   - In n8n: New Workflow → Import → Paste JSON

### Issue 3: Email Not Sending

**Symptoms:**
- Workflow executes successfully
- No email received

**Solutions:**
1. **Check email node configuration:**
   - Both "Send High Priority Email" and "Send Normal Email" need credentials
   - Verify SMTP settings (host, port, username, password)
   - Test credentials: Save → Test Connection

2. **Common SMTP configurations:**
   ```
   Gmail:
   - Host: smtp.gmail.com
   - Port: 587 (TLS) or 465 (SSL)
   - Use App Password (not regular password)
   
   Outlook:
   - Host: smtp-mail.outlook.com
   - Port: 587
   
   SendGrid:
   - Host: smtp.sendgrid.net
   - Port: 587
   - Username: apikey
   - Password: Your SendGrid API key
   ```

3. **Check spam folder** for test emails

### Issue 4: Agent Not Using Tools

**Symptoms:** AI responds without calling MCP tools

**Solutions:**
1. **Make prompt more directive:**
   ```
   You MUST use the get_forecast tool before responding.
   Do not make assumptions about weather.
   ```

2. **Verify tool discovery:**
   - Check n8n logs for MCP tool registration
   - Confirm server exposes tools at `/mcp/tools` endpoint

3. **Check tool descriptions:**
   - Make them action-oriented: "Fetch weather data" (good) vs "Weather" (bad)

### Issue 5: Rate Limiting

**Symptoms:** OpenWeatherMap API errors after multiple calls

**Solution:** Add rate limiting to your MCP server:

```python
from collections import deque
import time

rate_limiter = deque(maxlen=60)  # 60 calls per minute

@mcp.tool()
async def get_forecast(city: str):
    now = time.time()
    rate_limiter.append(now)
    
    # Check if we exceeded rate limit
    if len(rate_limiter) == 60 and now - rate_limiter[0] < 60:
        return {"error": "Rate limit exceeded", "retry_after": 60}
    
    # ... rest of implementation
```

**How this works:** The `deque` (double-ended queue) keeps track of the last 60 API call timestamps. If we've made 60 calls in under a minute, we return an error instead of hitting the API. This is a simple "sliding window" rate limiter.

---

## Advanced: Production Deployment

### Environment-Specific Configurations

**Development (.env):**
```bash
ENVIRONMENT=development
LOG_LEVEL=DEBUG
OPENWEATHER_API_KEY=your_key
```

**Production (.env.production):**
```bash
ENVIRONMENT=production
LOG_LEVEL=INFO
OPENWEATHER_API_KEY=your_key

# Additional production settings
SENTRY_DSN=your_sentry_dsn
REDIS_URL=redis://redis:6379
ENABLE_RATE_LIMITING=true
MAX_REQUESTS_PER_MINUTE=30
```

**Why different environments?** In development, you want verbose logging to debug issues. In production, you want less noise, error tracking (Sentry), caching (Redis), and stricter rate limits.

### Kubernetes Deployment

For production scale, deploy to Kubernetes:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: weather-mcp
spec:
  replicas: 3
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
              key: openweather-key
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
```

**What's Kubernetes?** If Docker is like shipping containers, Kubernetes is like the port authority that manages where containers go, ensures they're healthy, and spins up replacements when they fail. The `replicas: 3` means "always keep 3 copies of this server running"—if one crashes, Kubernetes automatically starts a new one.

### Monitoring & Observability

Add Prometheus metrics to your MCP server:

```python
from prometheus_client import Counter, Histogram, start_http_server

# Metrics
tool_calls = Counter('mcp_tool_calls_total', 'Total MCP tool calls', ['tool_name'])
tool_duration = Histogram('mcp_tool_duration_seconds', 'Tool execution time', ['tool_name'])

@mcp.tool()
async def get_forecast(city: str):
    with tool_duration.labels(tool_name='get_forecast').time():
        tool_calls.labels(tool_name='get_forecast').inc()
        # ... implementation
```

**What are these metrics?**
- **Counter**: Counts how many times something happened (e.g., "get_forecast called 1,247 times today")
- **Histogram**: Measures how long things take (e.g., "95% of weather calls complete in under 200ms")

Start metrics server:
```python
start_http_server(9090)  # Metrics at http://localhost:9090/metrics
```

Prometheus (a monitoring tool) scrapes this endpoint and stores the data for dashboards and alerts.

---

## Security Best Practices

### 1. API Key Management

❌ **Don't:** Hardcode API keys
```python
api_key = "sk-1234567890abcdef"  # BAD!
```

This is dangerous because: (1) Keys end up in version control, (2) Anyone with code access has your keys, (3) Rotating keys requires code changes.

✅ **Do:** Use environment variables
```python
api_key = os.getenv("OPENWEATHER_API_KEY")
if not api_key:
    raise ValueError("OPENWEATHER_API_KEY not set")
```

Environment variables are external to your code—they can be changed without redeploying, and they're not committed to Git.

### 2. Input Validation

```python
@mcp.tool()
async def get_forecast(city: str):
    # Validate input
    if not city or len(city) > 100:
        return {"error": "Invalid city name"}
    
    # Sanitize input (prevent injection)
    city = city.strip().replace(";", "")
```

**Why sanitize?** Users (or AI models) might pass malicious input. While this simple example just removes semicolons, production code should use proper sanitization libraries. Never trust user input.

### 3. Rate Limiting

See "Common Pitfalls" section for implementation.

### 4. HTTPS in Production

Update Docker Compose for SSL:
```yaml
n8n:
  environment:
    - N8N_PROTOCOL=https
    - N8N_SSL_KEY=/certs/privkey.pem
    - N8N_SSL_CERT=/certs/fullchain.pem
  volumes:
    - ./certs:/certs:ro
```

**Why HTTPS?** Without encryption, anyone on the network can see your API keys and data in transit. HTTPS encrypts everything between the client and server.

---

## Extending Your Agent

### Add New MCP Tools

Want to add more capabilities? Create new tools:

**Email MCP Server:**
```python
@mcp.tool()
async def send_email(to: str, subject: str, body: str):
    """Send emails via SendGrid"""
    # Implementation
```

**Database MCP Server:**
```python
@mcp.tool()
async def query_database(sql: str):
    """Execute read-only SQL queries"""
    # Implementation with safety checks
```

**Slack MCP Server:**
```python
@mcp.tool()
async def post_message(channel: str, text: str):
    """Post messages to Slack"""
    # Implementation using Slack API
```

Add them to `docker-compose.yml`:
```yaml
services:
  email-mcp:
    build: .
    command: python servers/email_server.py
    ports:
      - "8002:8002"
```

### Multi-Agent Patterns

For complex tasks, use a supervisor agent that delegates to specialized workers:

```python
@mcp.tool()
async def delegate_task(task_type: str, params: dict):
    """Supervisor delegates to specialized agents"""
    if task_type == "weather_analysis":
        return await call_weather_agent(params)
    elif task_type == "scheduling":
        return await call_calendar_agent(params)
    elif task_type == "communication":
        return await call_comms_agent(params)
```

**Agent hierarchy:**
- **Supervisor**: Plans and delegates tasks
- **Weather Worker**: Meteorological analysis
- **Calendar Worker**: Scheduling logic
- **Communication Worker**: Notifications

This pattern scales to complex problems where a single AI would be overwhelmed. The supervisor breaks down the problem and assigns pieces to specialists.

---

## What's Next?

You've built a production-ready AI agent! Here are ways to expand:

**Immediate improvements:**
- [ ] Add Google Calendar OAuth integration
- [ ] Implement Slack notifications
- [ ] Add more weather tools (5-day forecast, historical data)
- [ ] Create scheduling suggestions based on weather

**Advanced features:**
- [ ] Vector database for agent memory (remember past interactions)
- [ ] Multi-city support with timezone awareness
- [ ] Machine learning for personalized suggestions
- [ ] Voice interface with speech-to-text

**Production hardening:**
- [ ] Add comprehensive logging with Sentry
- [ ] Implement circuit breakers for API calls
- [ ] Set up automated testing (pytest + CI/CD)
- [ ] Create disaster recovery procedures

---

### Scaling

For high availability, use:
- Multiple MCP server replicas
- Load balancer in front of servers
- Kubernetes for orchestration

---

## Conclusion

You've built an autonomous AI agent that:
- ✅ Fetches real-time weather data via MCP
- ✅ Accesses calendar information
- ✅ Makes intelligent decisions about daily planning
- ✅ Sends actionable notifications
- ✅ Runs reliably in Docker containers

**The same architecture scales to:**
- Customer support agents (CRM + knowledge base)
- DevOps agents (infrastructure monitoring + incident response)
- Research agents (web search + document analysis)
- Sales agents (lead qualification + meeting scheduling)

The future of AI is autonomous systems that perceive, reason, and act. You just built one. 🚀

---

## Resources

### Code & Documentation
- [📂 Complete GitHub Repository](https://github.com/oneoceanlabs/mcp-agent-project)
- [📖 MCP Official Documentation](https://modelcontextprotocol.io/)
- [📖 n8n Documentation](https://docs.n8n.io/)
- [📖 FastMCP Python Library](https://github.com/jlowin/fastmcp)

### API Documentation
- [OpenWeatherMap API](https://openweathermap.org/api)
- [Google Calendar API](https://developers.google.com/calendar)
- [OpenAI API](https://platform.openai.com/docs)

### Community
- [n8n Community Forum](https://community.n8n.io/)
- [Blog Series Part 1: MCP Introduction](https://1oceanlabs.com/blog/2025-11-17-mcp-introduction-part1)

### Further Learning

If some concepts in this tutorial were new to you, here are some resources to deepen your understanding:

**Docker & Containers:**
- [Docker's Official Getting Started Guide](https://docs.docker.com/get-started/)
- [Docker Compose Tutorial](https://docs.docker.com/compose/gettingstarted/)

**Python Async Programming:**
- [Python asyncio documentation](https://docs.python.org/3/library/asyncio.html)
- [Real Python: Async IO in Python](https://realpython.com/async-io-python/)

**n8n Workflow Automation:**
- [n8n Quickstart](https://docs.n8n.io/try-it-out/quickstart/)
- [n8n Courses](https://docs.n8n.io/courses/)

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

**Crafted with ❤️ by One Ocean Labs**
