---
name: shot-scraper
description: "Visual verification and DOM inspection for frontend development. Auto-screenshots after UI changes, extracts rendered HTML, dumps accessibility trees, executes JavaScript for state inspection. Uses shot-scraper via uvx."
globs: ["**/*.tsx", "**/*.jsx", "**/*.vue", "**/*.svelte", "**/*.css", "**/*.html"]
alwaysApply: false
metadata:
  author: Bill Welense
  version: "1.0.0"
  triggers:
    - Screenshot a page
    - Check what the UI looks like
    - Verify frontend changes
    - Inspect the DOM
    - Check accessibility
    - Debug rendering issue
  categories:
    - visual-verification
    - dom-inspection
    - accessibility
    - javascript-execution
---

# shot-scraper: Visual Verification & DOM Inspection

Use [shot-scraper](https://github.com/simonw/shot-scraper) to verify frontend changes visually and inspect DOM/accessibility state during development and debugging.

**Prerequisites:** `uv` must be installed. shot-scraper runs via `uvx shot-scraper` — no pre-install needed.

## When to Use This Skill

| Trigger | Action |
|---------|--------|
| You just edited `.tsx`, `.jsx`, `.vue`, `.svelte`, `.css`, or `.html` files | **Auto-verify:** screenshot the relevant page |
| User says "screenshot", "check what it looks like", "inspect the DOM" | **Manual:** run the requested inspection |
| You're debugging a rendering issue | **Debug:** screenshot + HTML/a11y/JS inspection |

## Decision Tree

```
Did you just modify frontend files?
├─ YES → Go to: Auto-Verification Workflow
└─ NO
   ├─ User asked to inspect a page? → Go to: Manual Inspection
   └─ Debugging a rendering issue? → Go to: Debugging Workflow
```

## Output Conventions

All output goes to `.agents/screenshots/`. Create it if it doesn't exist:

```bash
mkdir -p .agents/screenshots
```

**File naming:** `YYYYMMDD-HHMMSS-<description>.{png,html,json}`
- Timestamp: 24-hour format, no separators in date/time groups
- Description: kebab-case, describes what was captured
- Example: `20260415-143052-homepage-after-nav-update.png`

Remind the user to add `.agents/` to `.gitignore` if it isn't already.

## Core Commands

### Screenshot

```bash
uvx shot-scraper shot <url> -o .agents/screenshots/<timestamp>-<description>.png
```

Common options:
- `--selector "css"` — capture a specific element
- `-w 1280 -h 800` — set viewport size (defaults: 1280 wide, full page height)
- `--wait 2000` — wait N milliseconds before capture (for async rendering)
- `--wait-for "document.querySelector('.loaded')"` — wait until JS condition is true
- `--log-console` — capture console.log output to stderr

### HTML Extraction

```bash
uvx shot-scraper html <url> -o .agents/screenshots/<timestamp>-<description>.html
```

Common options:
- `--selector "css"` — extract outerHTML of a specific element
- `-j "javascript"` — execute JS before extracting HTML

### Accessibility Tree

```bash
uvx shot-scraper accessibility <url> -o .agents/screenshots/<timestamp>-<description>.json
```

Returns Chromium's accessibility tree as JSON. Use for verifying semantic HTML, ARIA attributes, and screen reader compatibility.

### JavaScript Execution

```bash
uvx shot-scraper javascript <url> "js expression" -o .agents/screenshots/<timestamp>-<description>.json
```

The JS expression's return value is saved as JSON. Use for:
- Extracting component state: `"document.querySelector('#app').__vue__.$data"`
- Checking computed styles: `"getComputedStyle(document.querySelector('.hero')).display"`
- Counting elements: `"document.querySelectorAll('.error').length"`
- Checking for console errors: `"window.__errors || []"`

## Auto-Verification Workflow

Run this after making changes to frontend files:

### Step 1: Detect Dev Server

Check common ports to find a running dev server:

```bash
curl -s -o /dev/null -w "%{http_code}" http://localhost:3000
curl -s -o /dev/null -w "%{http_code}" http://localhost:3001
curl -s -o /dev/null -w "%{http_code}" http://localhost:5173
curl -s -o /dev/null -w "%{http_code}" http://localhost:4321
curl -s -o /dev/null -w "%{http_code}" http://localhost:8080
```

A `200` (or `304`) means the server is running on that port.

**If no server is responding:**
1. Read `package.json` and look for scripts named `dev`, `start`, or `serve` (check in that order)
2. Start the matching script in the background: `npm run dev &` (or `pnpm`, `yarn`, `bun` based on lockfile)
3. Poll the expected port every 2 seconds for up to 15 seconds:
   ```bash
   for i in 1 2 3 4 5 6 7; do
     curl -s -o /dev/null -w "%{http_code}" http://localhost:3000 | grep -q "200" && break
     sleep 2
   done
   ```
4. If still not responding after 15 seconds, tell the user: "Dev server isn't responding. Please start it and I'll take the screenshot."

### Step 2: Take Screenshot

```bash
mkdir -p .agents/screenshots
uvx shot-scraper shot http://localhost:<port> -o .agents/screenshots/<timestamp>-<description>.png --log-console
```

### Step 3: Read and Assess

Use the Read tool to view the PNG file. Assess:
- Does the UI match the intent of the changes you just made?
- Is the layout intact? Any broken elements?
- Is content rendering correctly?

### Step 4: Report

- **Looks correct:** Brief confirmation — "Verified: the nav update renders correctly with the new dropdown menu visible."
- **Something looks wrong:** Report what you see, then automatically follow up with HTML/a11y/JS inspection to diagnose.

## Manual Inspection

When the user asks to inspect a page:

1. Determine intent from their request:
   - "screenshot" / "what does it look like" → **Screenshot**
   - "inspect the DOM" / "show me the HTML" → **HTML extraction**
   - "check accessibility" / "a11y" → **Accessibility tree**
   - "check state" / "run JS" / "what's the value of" → **JavaScript execution**
2. Run the appropriate command
3. Read the output file
4. Report findings with specifics

## Debugging Workflow

When investigating a rendering issue:

1. **Screenshot** the current state to see what's actually rendering
2. If the issue isn't clear from the screenshot:
   - **HTML extraction** — is the expected markup in the DOM?
   - **Accessibility tree** — are semantic elements and ARIA attributes correct?
   - **JavaScript execution** — what's the component state? Any JS errors? What are the computed styles?
3. Correlate findings with the source code to identify the root cause

## Error Recovery

### Playwright Not Installed

If shot-scraper fails with a browser-related error on first run:

```bash
uvx shot-scraper install
```

Then retry the original command.

### Blank or Error Page

1. Retry with a wait: add `--wait 2000` to give the page time to render
2. Check console output: add `--log-console` and look at stderr
3. Extract HTML to see what the DOM actually contains:
   ```bash
   uvx shot-scraper html <url> -o .agents/screenshots/<timestamp>-blank-page-debug.html
   ```

### Selector Not Found

If `--selector` fails, fall back to a full-page screenshot and note in your report that the selector didn't match.

### Large or Long Pages

Default viewport is 1280x800. For full-page captures, you can extend the height:
```bash
uvx shot-scraper shot <url> -o output.png -w 1280 -h 2000
```

Note if you only captured above-the-fold content.
