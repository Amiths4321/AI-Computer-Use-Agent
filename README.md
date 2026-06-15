# AI-Computer-Use-Agent
# AI Computer Use Agent

A production-ready AI browser automation agent built as **Project 35** in the AI Solution Architecture learning series. Give any web task in plain English — Qwen2.5-VL sees the browser screen, decides what to click or type, and controls the browser autonomously until the task is complete.

---

## Where this fits in the AI development lifecycle

```
1-34. All previous projects      ✅
35.   AI Computer Use Agent      ← this project (frontier AI pattern)
```

---

## What makes this project unique

Every previous project processed data you gave it. This project goes out and gets data itself — by controlling a real browser like a human would.

| | Web Scraper (Project 18) | Computer Use Agent (this project) |
|---|---|---|
| How it works | BeautifulSoup parses HTML | Qwen2.5-VL sees screenshots visually |
| Handles JavaScript sites | No | Yes — full browser rendering |
| Can click buttons | No | Yes |
| Can fill forms | No | Yes |
| Needs CSS selectors | Yes | No — describes what to click in plain English |
| Behaves like a human | No | Yes |
| Works on any website | No | Yes |

---

## How it works — the ReAct loop

```
Task given: "Find the population of India on Wikipedia"
        ↓
Playwright opens Chromium browser
        ↓
Screenshot taken → sent to Qwen2.5-VL as base64 image
        ↓
Qwen sees the page visually and decides:
  "I see a Google search page. I should navigate to wikipedia.org"
        ↓
Action executed: navigate("https://wikipedia.org")
        ↓
New screenshot taken → sent to Qwen again
        ↓
Qwen sees Wikipedia homepage:
  "I can see a search box. I should search for India"
        ↓
Action: type("India") → press("Enter")
        ↓
New screenshot → Qwen reads India article:
  "I can see the population figure: 1.44 billion. Task complete."
        ↓
FINISH → returns answer
```

---

## Why Qwen2.5-VL is perfect for this

Qwen2.5-VL is a vision-language model — it understands both text and images. When given a screenshot of a webpage, it can:
- Read all visible text on the page
- Identify buttons, links, and form fields
- Understand the page layout and structure
- Decide what action a human would take next

This is exactly the same model you use for all other projects — no new model, no new GPU setup.

---

## Features

- **Visual browser control** — sees screenshots, not HTML source
- **Autonomous task completion** — up to 15 steps per task
- **Full action set** — navigate, click, type, scroll, search Google, read page
- **Live step display** — see what Qwen is thinking and doing in real time
- **Screenshot gallery** — every step captured and displayed
- **Works on any website** — including JavaScript-heavy apps
- **Headless by default** — no browser window opens on screen

---

## Project Structure

```
computer_use_agent/
│
├── computer_app.py      # Streamlit UI — main entry point
├── agent_loop.py        # ReAct loop — runs agent step by step
├── browser_tools.py     # Playwright browser actions
├── vision_agent.py      # Qwen2.5-VL decides next action from screenshot
│
├── screenshots/         # all step screenshots saved here
│
└── requirements.txt
```

---

## Prerequisites

- Python 3.9+
- Virtual environment activated
- Ollama running on remote GPU at `http://10.22.39.192:11434`
- Model `qwen2.5vl:latest` — must be a vision model
- Playwright and Chromium browser installed

---

## Installation

```powershell
cd computer_use_agent

# Activate venv
C:\Dev\venv\Scripts\Activate.ps1

# Install Python dependencies
pip install -r requirements.txt

# Install Playwright browser (one time — downloads ~200MB)
playwright install chromium
```

### `requirements.txt`

```
playwright
streamlit
requests
python-dotenv
pillow
```

---

## Running the App

```powershell
cd "C:\Users\amith\Desktop\Confidential\Misc Projects\P4\computer_use_agent"
C:\Dev\venv\Scripts\Activate.ps1
streamlit run computer_app.py
```

Open `http://localhost:8501`.

---

## Example Tasks

### Simple — single page lookup
```
"Go to wikipedia.org and find the population of India"
"Go to python.org and find the latest Python version"
"Go to news.ycombinator.com and list the top 5 headlines"
```

### Medium — search and read
```
"Search Google for 'what is RAG in AI' and summarise the top result"
"Go to github.com/trending and list the top 5 trending Python repositories"
"Go to en.wikipedia.org/wiki/Mumbai and find its area in square kilometres"
```

### Advanced — multi-step
```
"Search Google for 'FastAPI tutorial', open the first result, and give me the main steps to get started"
"Go to news.ycombinator.com, find the top post, and summarise what it is about"
"Go to wikipedia.org, search for Artificial Intelligence, and list the main subfields"
```

---

## The 7 Available Actions

| Action | What it does | Example parameter |
|---|---|---|
| navigate | Go to a URL | `{"url": "https://wikipedia.org"}` |
| click | Click on visible text | `{"text": "Search"}` |
| type | Type into a field | `{"text": "India population"}` |
| press | Press a keyboard key | `{"key": "Enter"}` |
| scroll | Scroll the page | `{"direction": "down", "amount": 3}` |
| search_google | Search Google | `{"query": "Python RAG tutorial"}` |
| read_page | Extract all page text | `{}` |
| FINISH | Task complete | `{"answer": "Population is 1.44 billion"}` |

---

## What Qwen Sees Per Step

Each step, Qwen receives:
1. The current task description
2. The current page URL
3. Last 5 steps taken (history)
4. A screenshot of the current browser state

From this it decides what single action to take next. This is identical to how a human would browse — see the screen, decide what to do, do it, see what changed.

---

## Known Fix — Windows asyncio Error

### Error
```
NotImplementedError at agent_loop.py line 146
asyncio.run() → _make_subprocess_transport → raise NotImplementedError
```

### Cause
Streamlit on Windows already has a running asyncio event loop. Calling `asyncio.run()` inside Streamlit tries to create a second event loop inside the first, which Windows does not support.

### Fix applied in `agent_loop.py`
```python
# BROKEN — conflicts with Streamlit's event loop on Windows
def run_task(task, on_step, headless):
    return asyncio.run(run_agent(task, on_step, headless))

# FIXED — separate thread with its own fresh event loop
def run_task(task, on_step, headless):
    result_container = {}

    def thread_target():
        loop = asyncio.new_event_loop()
        asyncio.set_event_loop(loop)
        result_container["result"] = loop.run_until_complete(
            run_agent(task, on_step, headless)
        )
        loop.close()

    t = threading.Thread(target=thread_target)
    t.start()
    t.join()
    return result_container["result"]
```

---

## Common Errors

| Error | Cause | Fix |
|---|---|---|
| `NotImplementedError` | asyncio conflict on Windows | Use fixed `agent_loop.py` with threading |
| `playwright install` not run | Chromium not downloaded | Run `playwright install chromium` |
| `TimeoutError on navigation` | Slow website | Increase timeout in `browser_tools.py` navigate() |
| `Ollama connection error` | Remote GPU not reachable | Check `curl http://10.22.39.192:11434/api/tags` |
| Agent loops without finishing | Qwen misreads screenshot | Try a simpler task first, increase MAX_STEPS |
| Screenshot is blank | Page not loaded yet | Increase `wait_for_timeout` in browser_tools.py |

---

## Limitations

- Works best on simple, text-rich websites
- May struggle with CAPTCHAs or login walls
- Free ngrok tunnels and dynamic content may cause issues
- Maximum 15 steps per task (configurable in `agent_loop.py`)
- Agent cannot download files or handle file upload dialogs
- Do not use for tasks requiring passwords or sensitive data

---

## Architecture — Decision Flow

```
computer_app.py
    ↓ run_task(task)
agent_loop.py
    ↓ Thread → new event loop
    ↓ run_agent() [async loop]
        ↓ browser_tools.screenshot()
        ↓ vision_agent.decide_action(screenshot, task, history)
            ↓ Qwen2.5-VL sees screenshot + reads prompt
            ↓ Returns JSON: { thought, action, parameters }
        ↓ browser_tools.{navigate|click|type|scroll|...}()
        ↓ repeat until FINISH or MAX_STEPS
    ↓ Returns { answer, steps, screenshots }
computer_app.py
    ↓ Displays answer + step breakdown + screenshot gallery
```

---

## Comparison to Industry

| Product | How it works | Model |
|---|---|---|
| OpenAI Operator | Screenshot → GPT-4o decides action | GPT-4o Vision |
| Claude Computer Use | Screenshot → Claude decides action | Claude 3.5 Sonnet |
| **This project** | Screenshot → Qwen2.5-VL decides action | Qwen2.5-VL (local) |

The architecture is identical. The only difference is the underlying vision model. You have built the same pattern as OpenAI Operator — locally, for free, with no API costs.

---

## Part of a Larger Project Series

| # | Project | Core skill learned |
|---|---|---|
| 6 | Multi-Agent Research | Web search + agent tools |
| 8 | Multimodal AI | Image + text with Qwen2.5-VL |
| 18 | Web Scraper | HTML parsing with BeautifulSoup |
| 35 | **Computer Use Agent** | **Visual browser control, Playwright, frontier AI** |

---

## Author

Built as part of an AI Solution Architecture learning project.
Model: `qwen2.5vl:latest` via Ollama on remote GPU `10.22.39.192:11434`
Browser automation: Playwright (Chromium)
No OpenAI · No Anthropic · Fully open source

