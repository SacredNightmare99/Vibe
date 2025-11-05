# Vibe

*Local AI-powered Coding Environment with Automatic Versioning*

---

## Vision

Vibe enables natural “conversation-driven coding” entirely on your own machine.
A Flutter mobile app connects to your development environment over SSH, runs **Gemini CLI** interactively, and automatically checkpoints every code change using **Vibegit**, a lightweight Git-based patch system.

You get the same control and interactability as Gemini CLI — but portable, remote, and version-tracked.

---

## System Overview

```
[ Flutter App (Vibe Mobile) ]
       ⇅ SSH (dartssh2)
 ┌────────────────────────────┐
 │ [Local Machine / Vibe Host]│
 │                            │
 │ ┌─────────────┐             │
 │ │ Gemini CLI  │  ← edits →  │
 │ ├─────────────┤             │
 │ │ Vibe Watch  │ ← detects ─ │
 │ │ (fsnotify)  │             │
 │ ├─────────────┤             │
 │ │ Vibegit CLI │ ← saves ─── │
 │ └─────────────┘             │
 │  (.vibes patches + log)     │
 └────────────────────────────┘
```

---

## Major Components

### 1. **Vibegit (Go CLI)**

Already implemented (`main.go`, `commands.go`), extended with new commands below.

**Purpose:**
Maintain lightweight, human-readable “checkpoints” of project state as Git patches.

**Key Commands:**

| Command                | Function                                        |
| ---------------------- | ----------------------------------------------- |
| `vibe init`            | Initialize `.vibes/` and log.json               |
| `vibe save "<msg>"`    | Save current working diff as a patch            |
| `vibe list`            | Show all recorded vibes                         |
| `vibe checkout <id>`   | Restore project to a saved patch                |
| `vibe diff <id>`       | Show patch contents                             |
| `vibe reset`           | Delete all saved vibes                          |
| **`vibe mark <path>`** | Mark directory for automatic watch              |
| **`vibe watch`**       | Start file-system watcher; auto-save on changes |

---

### 2. **Gemini CLI**

Google’s official command-line tool for Gemini models.
Runs inside the SSH session, providing full interactive REPL (multi-turn chat, diff previews, edits).

**Role in System:**
All AI code editing happens here.
No API integration required.
Any file modifications Gemini makes are visible to `vibe watch`.

---

### 3. **Vibe Watch Daemon**

**Responsibilities:**

* Read `.vibes/tracked.json` (marked directories).
* Use `fsnotify` to monitor file changes recursively.
* When changes occur:

  * Wait for a quiet period (debounce ~5 seconds).
  * Trigger `handleSave()` with message `"Auto-save: Gemini edit"`.

**Behavior Example:**

```
[INFO] Change detected in ~/projects/app1/lib/main.dart
[INFO] Waiting for changes to settle...
[INFO] Auto-saving as new vibe...
Vibe [1730814502] saved: Auto-save: Gemini edit
```

---

### 4. **Flutter App (Vibe Mobile)**

**Libraries:**

* `dartssh2` — persistent SSH connection
* `xterm.dart` — terminal emulator widget
* `get` or `riverpod` — lightweight state management
* Optional: `flutter_ansi` — colorized terminal text rendering

**Responsibilities:**

* Establish SSH connection automatically at startup.
* Launch an interactive PTY session (`bash`).
* Auto-run:

  ```bash
  vibe watch &
  gemini
  ```
* Render Gemini output live with full colors.
* Parse `[VIBE]`-tagged messages from stdout to show:

  * Auto-save notifications
  * Vibe history feed
  * Connection and daemon logs

**UI Concept:**

```
┌──────────────────────────────┐
│ Gemini CLI (live stream)     │
│ > Implement login with Supabase ...│
│ ...                          │
│ [VIBE] Vibe [1730814502] saved │
├──────────────────────────────┤
│  Connected: ~/projects/app1  │
│  Auto-saves: Enabled         │
└──────────────────────────────┘
```

---

## Workflow Example

1. You mark a project:

   ```
   vibe mark ~/projects/myapp
   ```

2. Launch the Flutter app.
   It connects via SSH → runs `vibe watch & gemini`.

3. You start prompting Gemini normally:

   ```
   Add a responsive navbar with search field
   ```

4. Gemini edits project files in `~/projects/myapp`.

5. `vibe watch` detects file changes, waits a few seconds, and executes:

   ```
   vibe save "Auto-save: Gemini edit"
   ```

6. The Flutter app displays:

   ```
   [VIBE] Auto-saved vibe: "Auto-save: Gemini edit"
   ```

7. You continue chatting with Gemini; every file change is versioned locally.

---

## Configuration Files

**`.vibes/tracked.json`**

```json
{
  "projects": [
    "/home/ishaan/projects/myapp",
    "/home/ishaan/projects/portfolio"
  ]
}
```

**`.vibes/log.json`**

```json
[
  {
    "id": 1730814502,
    "message": "Auto-save: Gemini edit",
    "timestamp": "2025-11-05T19:20:00"
  }
]
```

---

## Tech Stack Summary

| Layer      | Technology                           | Purpose                     |
| ---------- | ------------------------------------ | --------------------------- |
| Frontend   | Flutter (`dartssh2`, `xterm.dart`)   | SSH terminal and UI         |
| Backend    | Go (`fsnotify`, `gorilla/websocket`) | Vibegit CLI and watcher     |
| AI Engine  | Gemini CLI                           | Code generation             |
| Versioning | Git + patch diffs                    | Snapshot and restore states |

---

## Future Enhancements

| Feature                   | Description                                                               |
| ------------------------- | ------------------------------------------------------------------------- |
| **Patch summary AI**      | Use Gemini to summarize each auto-saved diff.                             |
| **Mobile diff viewer**    | Show unified diff in the app with syntax highlighting.                    |
| **Auto-tagging**          | Detect “feature/add”, “bugfix/…” based on prompt text.                    |
| **Multi-agent chat**      | Run additional local assistants (Ollama, DeepSeek) for specialized tasks. |
| **Cloud sync (optional)** | Upload `.vibes/log.json` and patches to GitHub repo if desired.           |

---

## Outcome

You get a **local-first AI coding assistant** with:

* True Gemini CLI interactivity
* Continuous local version tracking
* Seamless remote mobile interface

A minimal, elegant “AI-paired terminal” that feels like Copilot + Time Machine for your own codebase.

