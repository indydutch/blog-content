---
title: "MCP Demystified: Build an Autonomous AI Agent with MCP and n8n"
author: "Matthew Dutchess"
date: "2025-12-15"
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

Go checkout out our previous article on [MCP Demystified: The Open Standard Powering the Next Wave of Agentic AI](https://1oceanlabs.com/blog/2025-11-17-mcp-introduction-part1) for more information on why you should use MCP.

---

## Architecture Overview

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

**What makes this "agentic":** The LLM autonomously decides which tools to call and in what order based on the task—you don't hardcode the logic.

---

## Prerequisites

You'll need:

- **Docker & Docker Compose** — [Install Docker](https://docs.docker.com/get-docker/)
- **OpenWeatherMap API Key** (free) — [Get API Key](https://openweathermap.org/api)
- **OpenAI API Key** — [Get API Key](https://platform.openai.com/api-keys)
- **~15 minutes** for setup

> **Note:** The calendar server uses mock data by default. For real Google Calendar integration, you'll need to set up Google Cloud OAuth credentials (instructions in `.env.example`).

---

## Quick Start (5 Minutes)

### Step 1: Get the Project Files

Download or create the project structure:

```bash
mkdir mcp-agent-project && cd mcp-agent-project
mkdir servers
```

Create these files (code provided in sections below or refer to the *Resources* section at the end of the article):
- `servers/weather_server.py`
- `servers/calendar_server.py`
- `docker-compose.yml`
- `Dockerfile.mcp`
- `requirements.txt`
- `.env`
- `n8n-workflow.json`

### Step 2: Configure Environment

Create `.env` file:

```bash
# Required API Keys
OPENWEATHER_API_KEY=your_openweathermap_api_key_here
OPENAI_API_KEY=your_openai_api_key_here

# n8n Configuration
N8N_USER=admin
N8N_PASSWORD=your_secure_password
WEBHOOK_URL=http://localhost:5678
TIMEZONE=America/New_York

# Optional: Google Calendar (see .env.example for details)
# GOOGLE_CLIENT_ID=...
# GOOGLE_CLIENT_SECRET=...
```

### Step 3: Start Everything

```bash
docker-compose up -d
```

### Step 4: Import Workflow

1. Open http://localhost:5678
2. Login with your credentials
3. Create a new workflow
4. Import `n8n-workflow.json`
5. Configure OpenAI credentials
6. Click "Execute Workflow"

---

## Part 1: Weather MCP Server

### Understanding MCP Primitives

MCP has **three core types** of capabilities:

| Type | Description | How It Works |
|------|-------------|--------------|
| **Tools** | Functions the LLM can call | LLM explicitly invokes with parameters |
| **Resources** | Context data | Auto-injected into LLM's context |
| **Prompts** | Reusable templates | User-selectable interaction patterns |

### The Weather Server Code

```python
# servers/weather_server.py
import os
import sys
from typing import Any
from datetime import datetime

import httpx
from dotenv import load_dotenv
from fastmcp import FastMCP

# Load environment variables
load_dotenv()

# Create the MCP server instance
mcp = FastMCP("weather-assistant")


# Health check endpoint for Docker/Kubernetes
@mcp.custom_route("/health", methods=["GET"])
async def health_check(request):
    """Health check endpoint for container orchestration."""
    from starlette.responses import JSONResponse
    return JSONResponse({"status": "healthy", "service": "weather-mcp"})


@mcp.tool()
async def get_forecast(city: str) -> dict[str, Any]:
    """
    Fetch current weather forecast for a given city.
    
    Args:
        city: The name of the city to get weather for (e.g., "New York", "London", "Tokyo")
    
    Returns:
        Weather data including temperature, conditions, humidity, and smart recommendations
    """
    api_key = os.getenv("OPENWEATHER_API_KEY")
    if not api_key:
        return {
            "error": "OPENWEATHER_API_KEY environment variable not configured",
            "status": "error",
            "suggestion": "Please set your OpenWeatherMap API key"
        }
    
    url = "https://api.openweathermap.org/data/2.5/weather"
    params = {"q": city, "appid": api_key, "units": "metric"}
    
    try:
        async with httpx.AsyncClient() as client:
            response = await client.get(url, params=params, timeout=10.0)
            response.raise_for_status()
            data = response.json()
        
        temp = data["main"]["temp"]
        feels_like = data["main"]["feels_like"]
        description = data["weather"][0]["description"]
        humidity = data["main"]["humidity"]
        wind_speed = data.get("wind", {}).get("speed", 0)
        
        # Generate smart suggestions based on conditions
        suggestions = []
        
        # Weather condition suggestions
        if "rain" in description.lower():
            suggestions.append("Bring an umbrella - rain expected")
        if "storm" in description.lower() or "thunder" in description.lower():
            suggestions.append("⚠️ Storm warning - consider staying indoors")
        if "snow" in description.lower():
            suggestions.append("Snow expected - dress warmly and drive carefully")
        
        # Temperature suggestions
        if temp < 0:
            suggestions.append("Freezing conditions - bundle up with layers")
        elif temp < 10:
            suggestions.append("Cold weather - wear a warm coat")
        elif temp < 18:
            suggestions.append("Cool weather - a jacket is recommended")
        elif temp > 30:
            suggestions.append("Hot weather - stay hydrated and seek shade")
        elif temp > 25:
            suggestions.append("Warm weather - great for outdoor activities")
        else:
            suggestions.append("Pleasant temperature for most activities")
        
        # Wind suggestions
        if wind_speed > 10:
            suggestions.append(f"Windy conditions ({wind_speed} m/s) - secure loose items")
        
        # Humidity suggestions  
        if humidity > 80:
            suggestions.append("High humidity - may feel warmer than actual temperature")
        
        return {
            "city": city,
            "country": data.get("sys", {}).get("country", "Unknown"),
            "temperature": round(temp, 1),
            "feels_like": round(feels_like, 1),
            "description": description.title(),
            "humidity": humidity,
            "wind_speed": wind_speed,
            "suggestions": suggestions,
            "timestamp": datetime.utcnow().isoformat(),
            "status": "success"
        }
        
    except httpx.HTTPStatusError as e:
        if e.response.status_code == 404:
            return {
                "error": f"City '{city}' not found",
                "suggestion": "Check the spelling or try a different city name",
                "status": "error"
            }
        return {
            "error": f"Weather API error: {e.response.status_code}",
            "status": "error"
        }
    except httpx.RequestError as e:
        return {
            "error": f"Network request failed: {str(e)}",
            "suggestion": "Check your internet connection",
            "status": "error"
        }
    except Exception as e:
        return {
            "error": f"Unexpected error: {str(e)}",
            "status": "error"
        }


@mcp.tool()
async def get_forecast_multi(cities: list[str]) -> dict[str, Any]:
    """
    Fetch weather forecasts for multiple cities at once.
    
    Args:
        cities: List of city names to get weather for (e.g., ["New York", "London", "Tokyo"])
    
    Returns:
        Weather data for all requested cities with a summary
    """
    if not cities:
        return {"error": "No cities provided", "status": "error"}
    
    if len(cities) > 10:
        return {"error": "Maximum 10 cities allowed per request", "status": "error"}
    
    results = {}
    successful = 0
    failed = 0
    
    for city in cities:
        result = await get_forecast(city)
        results[city] = result
        if result.get("status") == "success":
            successful += 1
        else:
            failed += 1
    
    return {
        "forecasts": results,
        "summary": {
            "total_cities": len(cities),
            "successful": successful,
            "failed": failed
        },
        "timestamp": datetime.utcnow().isoformat(),
        "status": "success" if successful > 0 else "error"
    }


@mcp.tool()
async def check_outdoor_conditions(city: str) -> dict[str, Any]:
    """
    Check if weather conditions are suitable for outdoor activities.
    
    Args:
        city: The city to check conditions for
    
    Returns:
        Assessment of outdoor suitability with specific activity recommendations
    """
    weather = await get_forecast(city)
    
    if weather.get("status") == "error":
        return weather
    
    temp = weather["temperature"]
    description = weather["description"].lower()
    humidity = weather["humidity"]
    wind_speed = weather["wind_speed"]
    
    # Calculate outdoor score (0-100)
    score = 100
    concerns = []
    recommended_activities = []
    avoid_activities = []
    
    # Temperature scoring
    if temp < 0:
        score -= 40
        concerns.append("Freezing temperatures")
        avoid_activities.extend(["picnics", "outdoor dining"])
    elif temp < 10:
        score -= 20
        concerns.append("Cold weather")
        recommended_activities.append("hiking with warm clothes")
    elif temp > 35:
        score -= 35
        concerns.append("Extreme heat")
        avoid_activities.extend(["intense exercise", "prolonged sun exposure"])
    elif temp > 30:
        score -= 15
        concerns.append("Hot weather")
        recommended_activities.append("water activities")
    elif 18 <= temp <= 25:
        recommended_activities.extend(["walking", "cycling", "outdoor sports"])
    
    # Precipitation scoring
    if "rain" in description:
        score -= 30
        concerns.append("Rain expected")
        avoid_activities.extend(["outdoor dining", "hiking"])
    elif "storm" in description or "thunder" in description:
        score -= 50
        concerns.append("Storm conditions")
        avoid_activities.extend(["all outdoor activities"])
    elif "snow" in description:
        score -= 25
        concerns.append("Snow expected")
        recommended_activities.append("winter sports")
        avoid_activities.append("driving")
    
    # Wind scoring
    if wind_speed > 15:
        score -= 20
        concerns.append("High winds")
        avoid_activities.extend(["cycling", "umbrella use"])
    elif wind_speed > 10:
        score -= 10
        concerns.append("Windy conditions")
    
    # Humidity scoring
    if humidity > 85:
        score -= 10
        concerns.append("Very humid")
    
    # Ensure score is in valid range
    score = max(0, min(100, score))
    
    # Determine overall rating
    if score >= 80:
        rating = "Excellent"
        recommendation = "Perfect conditions for outdoor activities!"
    elif score >= 60:
        rating = "Good"
        recommendation = "Generally good for outdoor activities with minor considerations."
    elif score >= 40:
        rating = "Fair"
        recommendation = "Outdoor activities possible but prepare accordingly."
    elif score >= 20:
        rating = "Poor"
        recommendation = "Consider indoor alternatives or limit outdoor exposure."
    else:
        rating = "Not Recommended"
        recommendation = "Best to stay indoors today."
    
    return {
        "city": city,
        "outdoor_score": score,
        "rating": rating,
        "recommendation": recommendation,
        "concerns": concerns if concerns else ["None - great conditions!"],
        "recommended_activities": recommended_activities if recommended_activities else ["Most outdoor activities"],
        "avoid_activities": avoid_activities if avoid_activities else ["None specifically"],
        "current_conditions": {
            "temperature": temp,
            "description": weather["description"],
            "humidity": humidity,
            "wind_speed": wind_speed
        },
        "timestamp": datetime.utcnow().isoformat(),
        "status": "success"
    }


# Resource for automatic context injection
@mcp.resource("weather://help")
def weather_help() -> str:
    """Help information about available weather tools."""
    return """Weather Assistant - Available Tools:

1. get_forecast(city)
   - Get current weather for any city
   - Returns: temperature, conditions, humidity, wind, and suggestions
   - Example: get_forecast("New York")

2. get_forecast_multi(cities)
   - Get weather for multiple cities at once
   - Maximum 10 cities per request
   - Example: get_forecast_multi(["London", "Paris", "Berlin"])

3. check_outdoor_conditions(city)
   - Check if conditions are good for outdoor activities
   - Returns: score (0-100), rating, and activity recommendations
   - Example: check_outdoor_conditions("Seattle")

Tips:
- Use full city names for best results
- Add country code for ambiguous cities: "Paris, FR" vs "Paris, US"
- Check outdoor_conditions before planning outdoor events
"""


# Run the server
if __name__ == "__main__":
    import argparse
    
    parser = argparse.ArgumentParser(description="Weather MCP Server")
    parser.add_argument(
        "--transport",
        choices=["stdio", "http", "sse"],
        default="stdio",
        help="Transport protocol to use"
    )
    parser.add_argument("--host", default="0.0.0.0", help="Host to bind to")
    parser.add_argument("--port", type=int, default=8000, help="Port to bind to")
    
    args = parser.parse_args()
    
    if args.transport == "stdio":
        mcp.run()
    elif args.transport == "http":
        mcp.run(transport="http", host=args.host, port=args.port)
    elif args.transport == "sse":
        mcp.run(transport="sse", host=args.host, port=args.port)
```

---

## Part 2: Calendar MCP Server

```python
# servers/calendar_server.py
"""
Calendar MCP Server - Exposes calendar tools to LLMs via MCP.

Usage:
    Development:  fastmcp dev servers/calendar_server.py
    Production:   python servers/calendar_server.py --transport http --port 8001
"""

import os
from typing import Any
from datetime import datetime, timedelta
from fastmcp import FastMCP

# Create the MCP server
mcp = FastMCP("calendar-assistant")


# Health check endpoint for Docker/Kubernetes
@mcp.custom_route("/health", methods=["GET"])
async def health_check(request):
    """Health check endpoint for container orchestration."""
    from starlette.responses import JSONResponse
    return JSONResponse({"status": "healthy", "service": "calendar-mcp"})

# Mock calendar data - In production, replace with real API integration
# Keys are dates in YYYY-MM-DD format
MOCK_EVENTS: dict[str, list[dict[str, Any]]] = {}

def _generate_mock_data() -> dict[str, list[dict[str, Any]]]:
    """Generate mock calendar data for the next 7 days."""
    today = datetime.now()
    events = {}
    
    # Today's events
    today_str = today.strftime("%Y-%m-%d")
    events[today_str] = [
        {
            "id": "evt_001",
            "time": "09:00",
            "end_time": "09:30",
            "title": "Team standup",
            "location": "Conference Room A",
            "type": "meeting",
            "outdoor": False,
            "attendees": ["team@company.com"]
        },
        {
            "id": "evt_002",
            "time": "11:00",
            "end_time": "12:00",
            "title": "Project review",
            "location": "Zoom",
            "type": "meeting",
            "outdoor": False,
            "attendees": ["manager@company.com", "team@company.com"]
        },
        {
            "id": "evt_003",
            "time": "14:00",
            "end_time": "15:00",
            "title": "Client presentation",
            "location": "Client Office",
            "type": "meeting",
            "outdoor": False,
            "attendees": ["client@external.com"]
        },
        {
            "id": "evt_004",
            "time": "18:00",
            "end_time": "19:00",
            "title": "Evening jog",
            "location": "Central Park",
            "type": "personal",
            "outdoor": True,
            "attendees": []
        }
    ]
    
    # Tomorrow's events
    tomorrow = today + timedelta(days=1)
    tomorrow_str = tomorrow.strftime("%Y-%m-%d")
    events[tomorrow_str] = [
        {
            "id": "evt_005",
            "time": "10:00",
            "end_time": "11:30",
            "title": "Strategy meeting",
            "location": "Boardroom",
            "type": "meeting",
            "outdoor": False,
            "attendees": ["leadership@company.com"]
        },
        {
            "id": "evt_006",
            "time": "13:00",
            "end_time": "14:00",
            "title": "Lunch with Alex",
            "location": "Outdoor Café",
            "type": "personal",
            "outdoor": True,
            "attendees": ["alex@friend.com"]
        },
        {
            "id": "evt_007",
            "time": "16:00",
            "end_time": "17:00",
            "title": "Code review",
            "location": "Remote",
            "type": "work",
            "outdoor": False,
            "attendees": ["dev-team@company.com"]
        }
    ]
    
    # Day after tomorrow
    day_after = today + timedelta(days=2)
    day_after_str = day_after.strftime("%Y-%m-%d")
    events[day_after_str] = [
        {
            "id": "evt_008",
            "time": "09:30",
            "end_time": "10:30",
            "title": "Morning yoga",
            "location": "Rooftop Garden",
            "type": "personal",
            "outdoor": True,
            "attendees": []
        },
        {
            "id": "evt_009",
            "time": "14:00",
            "end_time": "16:00",
            "title": "Workshop presentation",
            "location": "Training Center",
            "type": "work",
            "outdoor": False,
            "attendees": ["workshop-attendees@company.com"]
        }
    ]
    
    return events

# Initialize mock data
MOCK_EVENTS = _generate_mock_data()


async def _get_events_for_date(date: str) -> dict[str, Any]:
    """
    Internal helper to get events for a date.
    Used by multiple tools to avoid calling tool from tool.
    """
    # Validate date format
    try:
        parsed_date = datetime.strptime(date, "%Y-%m-%d")
    except ValueError:
        return {
            "error": "Invalid date format. Please use YYYY-MM-DD (e.g., '2025-12-08')",
            "status": "error"
        }
    
    events = MOCK_EVENTS.get(date, [])
    
    # Calculate some useful metadata
    outdoor_events = [e for e in events if e.get("outdoor", False)]
    meetings = [e for e in events if e.get("type") == "meeting"]
    
    return {
        "date": date,
        "day_of_week": parsed_date.strftime("%A"),
        "events": events,
        "summary": {
            "total_events": len(events),
            "meetings": len(meetings),
            "outdoor_events": len(outdoor_events),
            "has_outdoor_events": len(outdoor_events) > 0
        },
        "status": "success"
    }


@mcp.tool()
async def check_events(date: str) -> dict[str, Any]:
    """
    Check calendar events for a specific date.
    
    Args:
        date: Date in YYYY-MM-DD format (e.g., "2025-12-08")
    
    Returns:
        List of events for the specified date with details
    """
    return await _get_events_for_date(date)


@mcp.tool()
async def check_events_range(start_date: str, end_date: str) -> dict[str, Any]:
    """
    Check calendar events for a date range.
    
    Args:
        start_date: Start date in YYYY-MM-DD format
        end_date: End date in YYYY-MM-DD format
    
    Returns:
        Events for all dates in the range
    """
    try:
        start = datetime.strptime(start_date, "%Y-%m-%d")
        end = datetime.strptime(end_date, "%Y-%m-%d")
    except ValueError:
        return {
            "error": "Invalid date format. Please use YYYY-MM-DD",
            "status": "error"
        }
    
    if end < start:
        return {
            "error": "End date must be after start date",
            "status": "error"
        }
    
    if (end - start).days > 30:
        return {
            "error": "Date range cannot exceed 30 days",
            "status": "error"
        }
    
    all_events = {}
    total_events = 0
    total_outdoor = 0
    current = start
    
    while current <= end:
        date_str = current.strftime("%Y-%m-%d")
        events = MOCK_EVENTS.get(date_str, [])
        if events:
            all_events[date_str] = events
            total_events += len(events)
            total_outdoor += len([e for e in events if e.get("outdoor", False)])
        current += timedelta(days=1)
    
    return {
        "start_date": start_date,
        "end_date": end_date,
        "events_by_date": all_events,
        "summary": {
            "total_events": total_events,
            "days_with_events": len(all_events),
            "total_outdoor_events": total_outdoor
        },
        "status": "success"
    }


@mcp.tool()
async def get_outdoor_events(date: str) -> dict[str, Any]:
    """
    Get only outdoor events for a specific date.
    
    Useful for checking which events might be affected by weather conditions.
    
    Args:
        date: Date in YYYY-MM-DD format
    
    Returns:
        List of outdoor events that may be weather-dependent
    """
    result = await _get_events_for_date(date)
    
    if result.get("status") == "error":
        return result
    
    events = result.get("events", [])
    outdoor_events = [e for e in events if e.get("outdoor", False)]
    
    return {
        "date": date,
        "outdoor_events": outdoor_events,
        "count": len(outdoor_events),
        "weather_sensitive": len(outdoor_events) > 0,
        "recommendation": (
            "Consider checking weather before these outdoor events"
            if outdoor_events
            else "No outdoor events scheduled - no weather concerns"
        ),
        "status": "success"
    }


@mcp.tool()
async def suggest_reschedule(
    event_title: str,
    reason: str,
    preferred_dates: list[str] | None = None
) -> dict[str, Any]:
    """
    Suggest rescheduling an event with alternative times.
    
    Args:
        event_title: Title of the event to reschedule
        reason: Reason for rescheduling (e.g., "bad weather", "conflict", "illness")
        preferred_dates: Optional list of preferred alternative dates (YYYY-MM-DD)
    
    Returns:
        Rescheduling suggestions with available time slots
    """
    # Find the event
    found_event = None
    found_date = None
    
    for date, events in MOCK_EVENTS.items():
        for event in events:
            if event["title"].lower() == event_title.lower():
                found_event = event
                found_date = date
                break
        if found_event:
            break
    
    if not found_event:
        return {
            "error": f"Event '{event_title}' not found in calendar",
            "suggestion": "Check the exact event title and try again",
            "status": "error"
        }
    
    # Generate alternative times
    alternatives = []
    base_date = datetime.strptime(found_date, "%Y-%m-%d")
    
    # Suggest times over the next 3 days
    for day_offset in range(1, 4):
        alt_date = base_date + timedelta(days=day_offset)
        alt_date_str = alt_date.strftime("%Y-%m-%d")
        
        # Check if this is a preferred date
        is_preferred = (
            preferred_dates and alt_date_str in preferred_dates
        )
        
        # Find available slots (simplified logic)
        available_times = ["09:00", "10:00", "14:00", "15:00", "16:00"]
        existing_times = [
            e["time"] for e in MOCK_EVENTS.get(alt_date_str, [])
        ]
        
        free_times = [t for t in available_times if t not in existing_times]
        
        if free_times:
            alternatives.append({
                "date": alt_date_str,
                "day": alt_date.strftime("%A"),
                "available_times": free_times[:3],  # Limit to 3 suggestions
                "is_preferred": is_preferred
            })
    
    return {
        "event": {
            "title": found_event["title"],
            "original_date": found_date,
            "original_time": found_event["time"],
            "location": found_event["location"]
        },
        "reason": reason,
        "alternatives": alternatives,
        "recommendation": (
            f"Based on '{reason}', we suggest moving '{event_title}' to one of the available slots above."
        ),
        "note": "In production, this would integrate with your calendar to actually reschedule the event.",
        "status": "success"
    }


@mcp.tool()
async def get_free_slots(date: str, duration_minutes: int = 60) -> dict[str, Any]:
    """
    Find available time slots on a given date.
    
    Args:
        date: Date to check in YYYY-MM-DD format
        duration_minutes: Required duration for the slot (default: 60)
    
    Returns:
        List of available time slots
    """
    try:
        parsed_date = datetime.strptime(date, "%Y-%m-%d")
    except ValueError:
        return {
            "error": "Invalid date format. Please use YYYY-MM-DD",
            "status": "error"
        }
    
    events = MOCK_EVENTS.get(date, [])
    
    # Define working hours (9 AM to 6 PM)
    working_start = 9 * 60  # 9:00 in minutes
    working_end = 18 * 60   # 18:00 in minutes
    
    # Convert events to blocked time ranges
    blocked_ranges = []
    for event in events:
        start_parts = event["time"].split(":")
        start_minutes = int(start_parts[0]) * 60 + int(start_parts[1])
        
        end_parts = event.get("end_time", event["time"]).split(":")
        end_minutes = int(end_parts[0]) * 60 + int(end_parts[1])
        
        # Add buffer time
        blocked_ranges.append((start_minutes - 15, end_minutes + 15))
    
    # Sort blocked ranges
    blocked_ranges.sort()
    
    # Find free slots
    free_slots = []
    current_time = working_start
    
    for block_start, block_end in blocked_ranges:
        if current_time + duration_minutes <= block_start:
            slot_start = f"{current_time // 60:02d}:{current_time % 60:02d}"
            slot_end = f"{block_start // 60:02d}:{(block_start) % 60:02d}"
            free_slots.append({
                "start": slot_start,
                "end": slot_end,
                "duration_available": block_start - current_time
            })
        current_time = max(current_time, block_end)
    
    # Check remaining time until end of day
    if current_time + duration_minutes <= working_end:
        slot_start = f"{current_time // 60:02d}:{current_time % 60:02d}"
        slot_end = f"{working_end // 60:02d}:{working_end % 60:02d}"
        free_slots.append({
            "start": slot_start,
            "end": slot_end,
            "duration_available": working_end - current_time
        })
    
    # Filter slots that meet minimum duration
    suitable_slots = [
        slot for slot in free_slots
        if slot["duration_available"] >= duration_minutes
    ]
    
    return {
        "date": date,
        "day": parsed_date.strftime("%A"),
        "requested_duration": duration_minutes,
        "available_slots": suitable_slots,
        "total_slots": len(suitable_slots),
        "status": "success"
    }


# Resources for automatic context injection
@mcp.resource("calendar://today")
def todays_schedule() -> str:
    """Today's calendar schedule (auto-injected into LLM context)."""
    today = datetime.now().strftime("%Y-%m-%d")
    events = MOCK_EVENTS.get(today, [])
    
    if not events:
        return f"No events scheduled for today ({today})"
    
    lines = [f"📅 Today's Schedule ({today}):", ""]
    for event in events:
        outdoor_tag = " 🌳" if event.get("outdoor") else ""
        lines.append(
            f"• {event['time']} - {event['title']} @ {event['location']}{outdoor_tag}"
        )
    
    outdoor_count = len([e for e in events if e.get("outdoor")])
    if outdoor_count:
        lines.append("")
        lines.append(f"⚠️ {outdoor_count} outdoor event(s) - check weather!")
    
    return "\n".join(lines)


@mcp.resource("calendar://preferences")
def user_preferences() -> str:
    """User scheduling preferences (auto-injected into LLM context)."""
    return """📋 User Scheduling Preferences:

Working Hours: 9:00 AM - 6:00 PM (Monday-Friday)
Time Zone: Local time

Preferences:
• 15-minute buffer between meetings
• Lunch break: 12:00 PM - 1:00 PM (keep free when possible)
• Morning hours preferred for focused work
• Outdoor activities preferred in good weather
• Maximum 3 consecutive hours of meetings

Notification Settings:
• Remind 15 minutes before meetings
• Weather alerts for outdoor events
• Daily briefing at 7:00 AM
"""


@mcp.resource("calendar://help")
def calendar_help() -> str:
    """Help information about available calendar tools."""
    return """Calendar Assistant - Available Tools:

1. check_events(date)
   - Get all events for a specific date
   - Example: check_events("2025-12-08")

2. check_events_range(start_date, end_date)
   - Get events for a date range (max 30 days)
   - Example: check_events_range("2025-12-08", "2025-12-15")

3. get_outdoor_events(date)
   - Find outdoor events that may be affected by weather
   - Example: get_outdoor_events("2025-12-08")

4. suggest_reschedule(event_title, reason, preferred_dates)
   - Get rescheduling suggestions for an event
   - Example: suggest_reschedule("Evening jog", "rain expected", ["2025-12-09"])

5. get_free_slots(date, duration_minutes)
   - Find available time slots on a date
   - Example: get_free_slots("2025-12-08", 60)

Tips:
- Always use YYYY-MM-DD format for dates
- Check outdoor events before making weather-dependent plans
- Use suggest_reschedule when weather or conflicts arise
"""


# Run the server
if __name__ == "__main__":
    import argparse
    
    parser = argparse.ArgumentParser(description="Calendar MCP Server")
    parser.add_argument(
        "--transport",
        choices=["stdio", "http", "sse"],
        default="stdio",
        help="Transport protocol to use"
    )
    parser.add_argument("--host", default="0.0.0.0", help="Host to bind to")
    parser.add_argument("--port", type=int, default=8001, help="Port to bind to")
    
    args = parser.parse_args()
    
    if args.transport == "stdio":
        mcp.run()
    elif args.transport == "http":
        mcp.run(transport="http", host=args.host, port=args.port)
    elif args.transport == "sse":
        mcp.run(transport="sse", host=args.host, port=args.port)
```

---

## Part 3: Docker Configuration

### Dockerfile.mcp

```dockerfile
FROM python:3.11-slim

WORKDIR /app

RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY servers/ ./servers/

ENV PYTHONUNBUFFERED=1
EXPOSE 8000

HEALTHCHECK --interval=10s --timeout=5s --start-period=15s --retries=3 \
    CMD curl -sf http://localhost:8000/health || exit 1

CMD ["python", "servers/weather_server.py", "--transport", "http"]
```

### requirements.txt

```
fastmcp>=2.0.0
httpx>=0.25.0
python-dotenv>=1.0.0
uvicorn>=0.24.0
```

### docker-compose.yml

```yaml
services:
  weather-mcp:
    build:
      context: .
      dockerfile: Dockerfile.mcp
    container_name: weather-mcp
    ports:
      - "8000:8000"
    environment:
      - OPENWEATHER_API_KEY=${OPENWEATHER_API_KEY}
      - PYTHONUNBUFFERED=1
    command: ["python", "servers/weather_server.py", "--transport", "http", "--port", "8000"]
    healthcheck:
      test: ["CMD", "curl", "-sf", "http://localhost:8000/health"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 15s
    networks:
      - mcp-network
    restart: unless-stopped

  calendar-mcp:
    build:
      context: .
      dockerfile: Dockerfile.mcp
    container_name: calendar-mcp
    ports:
      - "8001:8001"
    environment:
      - PYTHONUNBUFFERED=1
    command: ["python", "servers/calendar_server.py", "--transport", "http", "--port", "8001"]
    healthcheck:
      test: ["CMD", "curl", "-sf", "http://localhost:8001/health"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 15s
    networks:
      - mcp-network
    restart: unless-stopped

  n8n:
    image: n8nio/n8n:latest
    container_name: n8n
    ports:
      - "5678:5678"
    environment:
      - N8N_COMMUNITY_PACKAGES_ALLOW_TOOL_USAGE=true
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=${N8N_USER:-admin}
      - N8N_BASIC_AUTH_PASSWORD=${N8N_PASSWORD:-admin}
      - WEBHOOK_URL=${WEBHOOK_URL:-http://localhost:5678}
      - GENERIC_TIMEZONE=${TIMEZONE:-America/New_York}
      - TZ=${TIMEZONE:-America/New_York}
      - MCP_OPENWEATHER_API_KEY=${OPENWEATHER_API_KEY}
    volumes:
      - n8n_data:/home/node/.n8n
      - ./n8n-workflow.json:/home/node/workflow.json:ro
    depends_on:
      weather-mcp:
        condition: service_healthy
      calendar-mcp:
        condition: service_healthy
    networks:
      - mcp-network
    restart: unless-stopped

networks:
  mcp-network:
    driver: bridge

volumes:
  n8n_data:
    driver: local
```

---

## Part 4: n8n Workflow Setup

### MCP Client Tool Configuration

n8n includes built-in MCP support with the **MCP Client Tool** node:

1. **Add AI Agent node** to your workflow
2. **Add MCP Client Tool** nodes (connect to Agent's Tools input)
3. **Configure endpoints:**
   - Weather: `http://weather-mcp:8000/mcp`
   - Calendar: `http://calendar-mcp:8001/mcp`
4. **Authentication:** None (internal Docker network)
5. **Tools to Include:** All

### Importing the Workflow

1. Open n8n at http://localhost:5678
2. Create a new workflow.
3. Click the 3 dots on the upper task bar → **Import from file...**
4. Select `n8n-workflow.json`
5. Configure credentials:
   - Click **OpenAI Chat Model** → Add OpenAI API key
   - Click **Send Email** nodes → Configure SMTP/Gmail
6. Click **Execute Workflow** to test

### Workflow Structure

```
[Daily Trigger] → [Set Context] → [AI Agent] → [Check Alerts] → [Send Notification]
                                       ↑
                                 [MCP Tools]
                            - Weather MCP  (port 8000)
                            - Calendar MCP (port 8001)
```

### AI Agent Configuration

**System Prompt:**
```
You are a morning planning assistant with access to weather and calendar tools.

You MUST use the available tools to get real data - never make assumptions.

Workflow:
1. Call get_forecast for the user's city
2. Call check_events for today's date
3. Call get_outdoor_events to find weather-sensitive activities
4. Analyze conflicts and provide recommendations

Be concise and actionable.
```

**User Prompt:**
```
Create a morning briefing for today ({{ $json.date }}) in {{ $json.city }}.

Include:
- Weather summary and what to wear
- Today's schedule overview
- Any weather concerns for outdoor events
- Top 3 recommendations
```

---

## Part 5: Testing Your Setup

### Test MCP Servers Directly

```bash
# Check if servers are running
docker-compose ps

# Test weather server health
curl http://localhost:8000/health

# Test calendar server health
curl http://localhost:8001/health
```

### Test with MCP Inspector

```bash
# Install FastMCP
pip install fastmcp

# Interactive testing
cd servers
fastmcp dev weather_server.py

# In inspector, call:
# Tool: get_forecast
# Parameters: {"city": "New York"}
```

### Test n8n Workflow

1. Click **Execute Workflow**
2. Watch execution flow through nodes
3. Check the output in the final node

### View Logs

```bash
docker-compose logs -f weather-mcp
docker-compose logs -f n8n
```

---

## Common Issues & Solutions

### "Agent not calling tools"

Add explicit instructions to the system prompt:
```
You MUST call the tools to get data before responding.
Do not make assumptions about weather or calendar events.
```

### "MCP connection failed"

```bash
# Verify servers are healthy
docker-compose ps

# Test from n8n container
docker-compose exec n8n curl http://weather-mcp:8000/health
docker-compose exec n8n curl http://calendar-mcp:8001/health
```

### Gmail API "Forbidden" Error

If you see "Gmail API has not been used in project..." error:
1. Click the link in the error message
2. Click **Enable API**
3. Wait 1-2 minutes for propagation
4. Retry in n8n

### "Insufficient quota" (OpenAI)

Your OpenAI account is out of credits. Either:
- Add billing credits at [platform.openai.com/account/billing](https://platform.openai.com/account/billing)
- Switch to Anthropic Claude or another LLM provider

---

## Production Deployment

### Security Checklist

- [ ] Change default passwords
- [ ] Use HTTPS with reverse proxy
- [ ] Restrict network access to MCP servers
- [ ] Use secrets management for API keys

### Scaling

For high availability, use:
- Multiple MCP server replicas
- Load balancer in front of servers
- Kubernetes for orchestration

---

## Conclusion

You've built a production-ready AI agent that:
- ✅ Connects to external data via MCP
- ✅ Makes intelligent decisions with LLMs
- ✅ Takes autonomous actions
- ✅ Runs reliably in Docker

**This pattern scales** to customer support agents, DevOps automation, research assistants, and much more.

The future of AI is autonomous systems that perceive, reason, and act. You just built one. 🚀

---

## Resources

- [MCP n8n Tutorial GitHub Project]()
- [MCP Protocol Specification](https://modelcontextprotocol.io/specification)
- [FastMCP Python SDK](https://github.com/jlowin/fastmcp)
- [n8n Documentation](https://docs.n8n.io/)
- [n8n MCP Integration](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/)

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