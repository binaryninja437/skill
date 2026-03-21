# Viewer Setup Reference

This file defines how Agent 3 should open the saved `.md` note in a browser,
covering the markdown viewer server, cross-platform launch, port handling, and fallbacks.

---

## Overview

Agent 3 receives the absolute path to `consultation-note.md` from Agent 2 and opens it
for the doctor to read in a rendered (formatted) view.

Two approaches, in priority order:

1. **Markdown viewer server** — if Node.js and the viewer script are available
2. **Direct browser fallback** — open the `.md` file directly in the browser

---

## Approach 1 — Markdown Viewer Server

The markdown viewer script renders `.md` files as HTML in a local server.

### Locating the Script

Do NOT hardcode a path to the viewer script. Resolve it at runtime:

```python
import os, shutil

# Option A: Check if a viewer script path was provided by the user
viewer_script = user_provided_viewer_path  # may be None

# Option B: Search in common locations
if not viewer_script:
    candidates = [
        os.path.join(os.path.expanduser("~"), ".claude", "skills",
                     "markdown-novel-viewer", "scripts", "server.cjs"),
        os.path.join(os.getcwd(), "scripts", "server.cjs"),
    ]
    for c in candidates:
        if os.path.exists(c):
            viewer_script = c
            break

# Option C: Check if 'npx' or a global viewer is available
if not viewer_script:
    viewer_script = None  # fall back to Approach 2
```

### Launch Command (Cross-Platform)

```python
import subprocess, sys, json

def launch_viewer(viewer_script, note_path):
    cmd = ["node", viewer_script, "--file", note_path, "--open", "--background"]

    result = subprocess.run(cmd, capture_output=True, text=True, timeout=10)

    # Parse JSON output to get the URL
    try:
        data = json.loads(result.stdout)
        url = data.get("url")
    except (json.JSONDecodeError, AttributeError):
        # Some viewer versions print the URL as plain text
        url = result.stdout.strip()

    return url
```

### Port Handling

The viewer defaults to port 3456. If that port is in use, it auto-increments to the next
available port. Always use the URL returned by the command — never hardcode a port.

If the URL is not returned (server failed to start):
```python
# Force-open whatever URL was printed, or fall back
url = result.stdout.strip() or "http://localhost:3456"
```

### Opening the Browser

After getting the URL, open it non-blocking:

```python
def open_browser(url):
    if sys.platform == "win32":
        subprocess.Popen(["cmd", "/c", "start", "", url])
    elif sys.platform == "darwin":
        subprocess.Popen(["open", url])
    else:
        subprocess.Popen(["xdg-open", url])
```

Report the URL to the doctor even if the browser opened automatically.

---

## Approach 2 — Direct Browser Fallback

If the viewer server is unavailable (Node.js not installed, script not found), open the
`.md` file directly in the default browser. Most modern browsers render Markdown files
adequately, though without full styling.

```python
def open_md_directly(note_path):
    # Convert to file:// URI
    from pathlib import Path
    file_uri = Path(note_path).as_uri()

    if sys.platform == "win32":
        subprocess.Popen(["cmd", "/c", "start", "", file_uri])
    elif sys.platform == "darwin":
        subprocess.Popen(["open", file_uri])
    else:
        subprocess.Popen(["xdg-open", file_uri])

    return file_uri
```

Tell the doctor:
> "The markdown viewer server was not found. The note has been opened directly in your
> browser at: `[file URI]`. For a better-formatted view, install the markdown viewer
> (see skill README)."

---

## Checking Node.js Availability

```python
import shutil
node_available = shutil.which("node") is not None
```

If `node_available` is False, skip Approach 1 entirely and go straight to Approach 2.

---

## Full Agent 3 Decision Flow

```
1. Receive note_path from Agent 2
2. Check: does Node.js exist?
   NO  → Approach 2 (direct browser open)
   YES → Search for viewer_script
         NOT FOUND → Approach 2
         FOUND     → Approach 1 (launch viewer server)
3. Open browser with URL or file:// URI
4. Report to doctor: file path + URL/URI + any fallback note
```

---

## Troubleshooting

| Problem | Fix |
|---|---|
| `node: command not found` | Node.js not installed — use Approach 2 fallback |
| `viewer_script` not found at any candidate path | Use Approach 2; ask user to provide script path if they want the styled viewer |
| Port already in use (EADDRINUSE) | Viewer auto-increments — use the URL it prints |
| Browser doesn't open automatically | Manually print the URL so the doctor can paste it |
| Note file not found | Verify Agent 2 saved successfully before Agent 3 runs |
