# Nexus Editor — Developer Handbook

> A production-grade documentation series for building an AI-powered desktop IDE with Electron, React, and Node.js.

**Audience:** Developers comfortable with React and Node.js, new to Electron.
**Status:** Chapter 1 of 15

---

## Table of Contents (Full Series)

1. Introduction *(this document)*
2. Project Architecture
3. Installation Guide
4. Electron Lifecycle
5. Communication (IPC)
6. File System
7. Monaco Editor
8. Terminal
9. Git Integration
10. AI Integration
11. Security
12. Packaging
13. Deployment
14. Best Practices
15. Learning Roadmap

---

# Chapter 1 — Introduction

## 1.1 What is Electron?

Electron is a framework that lets you build **desktop applications using web technologies** — HTML, CSS, and JavaScript (or TypeScript). Instead of learning a native language like C++, Swift, or C#, you write your app the way you'd write a website, and Electron wraps it into a real, installable desktop program for Windows, macOS, and Linux.

It does this by combining two things under one roof:

- **Chromium** — the same rendering engine that powers Google Chrome. This is what draws your UI (your React app, your buttons, your Monaco editor).
- **Node.js** — a JavaScript runtime that can talk to the operating system: read files, spawn processes, access the network at a low level, control windows, etc.

A normal website running in a browser tab can *only* use Chromium. It has no Node.js access, because letting random websites read your hard drive would be a catastrophic security hole. Electron's entire value proposition is that it lets you **combine both environments in one application**, under your control, with your own security boundaries.

```mermaid
graph LR
    A[Electron App] --> B[Chromium Engine]
    A --> C[Node.js Runtime]
    B --> D[Renders UI: HTML/CSS/React]
    C --> E[System Access: files, processes, OS APIs]
```

For Nexus Editor, this is exactly what we need: a rich, React-based UI (file tree, tabs, Monaco editor, AI chat panel) that must also read and write real files on the user's disk, spawn a real terminal, and run Git commands — things a website can never do.

---

## 1.2 Why Electron Instead of a Web App?

You might ask: "Why not just build Nexus Editor as a web app that runs in the browser?" This is a fair question, and the answer comes down to **capabilities a browser sandbox intentionally forbids**.

| Requirement | Web App (Browser) | Electron Desktop App |
|---|---|---|
| Read/write arbitrary files on disk | ❌ Not without user picking each file manually | ✅ Full file system access |
| Spawn a real terminal (shell process) | ❌ Impossible | ✅ Via `node-pty` |
| Run Git commands directly | ❌ Impossible | ✅ Via `simple-git` / child processes |
| Watch folders for file changes | ❌ Very limited | ✅ Native file watchers |
| Work fully offline | ⚠️ Requires service workers, partial | ✅ Native, no server required |
| Native OS integration (tray, menus, notifications) | ⚠️ Limited | ✅ Full native APIs |
| Distribute as a single installable app | ❌ Needs a browser | ✅ `.exe`, `.dmg`, `.AppImage` |
| Auto-update mechanism | N/A (server-controlled) | ✅ Built-in (`electron-updater`) |

A code editor like VS Code, Cursor, or our Nexus Editor is fundamentally a **file system tool with a UI**. That UI can be a beautiful React app — but the moment you need to open a folder, read hundreds of files, watch them for changes, and spawn a terminal, you have left "web app" territory and entered "desktop app" territory. This is why VS Code, Slack, Discord, and Figma's desktop app are all built on Electron (or the conceptually similar Tauri).

**The tradeoff:** Electron apps are heavier (they ship a Chromium + Node.js runtime, so even a "hello world" app is ~100MB+), and you take on more responsibility for security since you now have real system access. Chapter 11 covers this in depth.

---

## 1.3 Electron Architecture

Electron's architecture is built around a **strict separation of responsibilities** between multiple processes. This is the single most important mental model to internalize before writing any code — almost every bug and security issue in Electron apps comes from misunderstanding this boundary.

```mermaid
graph TB
    subgraph "Main Process (Node.js)"
        A[app lifecycle]
        B[BrowserWindow management]
        C[Native OS APIs]
        D[File System Access]
        E[ipcMain]
    end

    subgraph "Preload Script"
        F[contextBridge]
        G[Selective API Exposure]
    end

    subgraph "Renderer Process (Chromium)"
        H[React App / UI]
        I[Monaco Editor]
        J[ipcRenderer - restricted]
    end

    E <-->|IPC Messages| F
    F -->|window.electronAPI| H
    H --> I
    H --> J
    J -->|invoke/send| F
```

Let's break down each piece.

### 1.3.1 Main Process

The **Main Process** is the entry point of every Electron app — a single Node.js process that starts when your app launches. Think of it as the "backend" of your desktop app, even though there's no server involved.

Its responsibilities:

- Creating and managing application windows (`BrowserWindow` instances)
- Controlling the app lifecycle (launch, quit, minimize, activate on macOS dock click, etc.)
- Having **full Node.js access** — it can read/write files, spawn child processes, access native modules
- Owning native OS integrations: application menus, system tray, native dialogs (Open Folder, Save As), notifications
- Listening for messages from renderer processes via `ipcMain` and responding to them

There is only **one Main Process** per app, no matter how many windows you open. In Nexus Editor, this is where file system operations, terminal spawning, and Git commands will actually execute — because only the Main Process has the Node.js APIs needed to do so.

### 1.3.2 Renderer Process

The **Renderer Process** is where your UI lives — one per `BrowserWindow`. This is essentially a Chromium browser tab running your React application. It renders HTML/CSS, runs your React component tree, handles user clicks, and displays the Monaco editor.

Critically, by default (and for good security reasons we'll explain in 1.3.6), the Renderer Process **does not have direct access to Node.js or the file system**. It behaves like a regular, sandboxed web page. If Nexus Editor's React UI wants to "open a folder," it cannot call `fs.readdir()` directly — it must *ask* the Main Process to do it on its behalf, through IPC.

You can have multiple renderer processes (multiple windows), and each one is isolated from the others, just like browser tabs are isolated from each other.

### 1.3.3 Preload Script

The **Preload Script** is a special script that runs in a unique, privileged context: it has access to a limited set of Node.js APIs (because Electron trusts it), *and* it runs before your web page's JavaScript loads, in the same window as the renderer.

Its job is to act as a **secure bridge** — it selectively exposes a small, controlled set of functions to the Renderer Process via `contextBridge`, without ever exposing raw Node.js or `ipcRenderer` wholesale. This is the "gatekeeper" pattern: the preload script decides exactly what the untrusted UI code is allowed to ask for.

For Nexus Editor, our preload script will expose something like `window.electronAPI.openFolder()`, `window.electronAPI.readFile(path)`, etc. — never the raw `fs` module.

### 1.3.4 IPC Communication

**IPC** stands for **Inter-Process Communication**. Since the Main Process and Renderer Process are two entirely separate OS processes (they don't share memory), they cannot call each other's functions directly. Instead, they pass messages back and forth — this is IPC.

Electron gives us two matching modules:

- `ipcMain` — used in the Main Process to *listen* for messages and *reply*
- `ipcRenderer` — used in the Preload Script to *send* messages and *receive* replies

```mermaid
sequenceDiagram
    participant R as Renderer (React UI)
    participant P as Preload (contextBridge)
    participant M as Main Process

    R->>P: window.electronAPI.readFile('/path/file.ts')
    P->>M: ipcRenderer.invoke('fs:read-file', path)
    M->>M: fs.readFile(path)
    M-->>P: return file contents
    P-->>R: Promise resolves with contents
```

This request/response pattern (`invoke` / `handle`) is the most common form of IPC and is covered in full depth, with every API explained, in **Chapter 5: Communication**.

### 1.3.5 Context Isolation

**Context Isolation** is a security feature (enabled by default since Electron 12, and mandatory for Nexus Editor) that ensures the JavaScript running in your preload script and the JavaScript running in your web page (React app) execute in **separate JavaScript contexts**, even though they share the same window.

Without context isolation, a malicious script running in your renderer (say, injected through a compromised npm package or an unsanitized AI-generated code preview) could potentially reach into and tamper with your preload script's privileged objects, and from there escalate to Node.js access. With context isolation **on**, the only communication channel between the two worlds is the explicit, narrow API you expose through `contextBridge` — nothing more.

**Rule for Nexus Editor:** Context Isolation is always `true`. There is no valid reason to disable it in a modern Electron app.

### 1.3.6 Security Best Practices

Given everything above, here are the foundational security rules that shape every architectural decision in this handbook (expanded fully in Chapter 11):

1. **`contextIsolation: true`** — always. Isolates preload/main-world JS contexts.
2. **`nodeIntegration: false`** — the Renderer Process must never have direct Node.js access.
3. **`sandbox: true`** where possible — further restricts what the renderer can do at the OS level.
4. **Never use `remote` module** — it's deprecated and dangerous; it blurred the Main/Renderer boundary.
5. **Validate all IPC input** — the Main Process must treat every message from a renderer as untrusted input, especially since Nexus Editor will render AI-generated content.
6. **Use a strict Content Security Policy (CSP)** — restricts what scripts/resources the renderer can load.
7. **Only expose the minimum API surface via `contextBridge`** — never expose `ipcRenderer` itself, only specific, purpose-built functions.

These aren't optional extras — they're the reason the Main/Renderer/Preload split exists at all. Every chapter from here on builds *on top of* this boundary, never around it.

---

## Common Mistakes (Chapter 1)

1. **Enabling `nodeIntegration: true` "to make things easier."** This defeats the entire security model and exposes the full Node.js API — including `require('fs')`, `require('child_process')` — to any script running in your UI, including third-party npm packages and (in our case) AI-generated content rendered in the UI.
2. **Thinking Electron is "just a browser wrapper."** It is a *two-process architecture with a security boundary*, not a single environment. Beginners often write code assuming the renderer can do anything Node.js can.
3. **Putting file system logic in the Renderer Process.** Even with `nodeIntegration` off, some developers try to route around the design by disabling isolation "temporarily." All file/OS access belongs in the Main Process, full stop.
4. **Confusing "Main Process" with "backend server."** There's no HTTP server involved (unless you add one for a specific feature); Main Process communication happens via IPC, not `fetch`.
5. **Assuming one Main Process per window.** There is exactly **one** Main Process for the entire app, regardless of how many `BrowserWindow`s exist.

---

## Interview Questions (Chapter 1)

1. What are the three core building blocks of an Electron app, and what does each one own?
2. Why can't a Renderer Process directly call `fs.readFile()`, even though Electron bundles Node.js?
3. Explain the role of the preload script and why it runs in a "privileged but isolated" context.
4. What does `contextIsolation` actually protect against? Give a concrete attack scenario it prevents.
5. Why would a code editor application (like VS Code or Nexus Editor) choose Electron over a pure web app? Name at least three capabilities that require it.
6. What is the difference between `nodeIntegration` and `contextIsolation` — are they the same setting?
7. How many Main Processes exist in an Electron app with 3 open windows? How many Renderer Processes?

---

## Practical Exercises (Chapter 1)

**Exercise 1.1 — Process Mapping**
Draw (on paper or in Mermaid) the Nexus Editor architecture showing: Main Process, one Renderer Process, and the Preload script. Label which process would be responsible for: (a) opening a folder picker dialog, (b) rendering the file tree, (c) reading a `.ts` file's contents, (d) running `git status`.

**Exercise 1.2 — Security Reasoning**
Without writing code yet, explain in your own words: if Nexus Editor's AI assistant generates a code snippet and displays it in the Renderer Process, why is it critical that the Renderer has no direct Node.js access, even if the AI-generated code looks harmless?

**Exercise 1.3 — Research**
Look up how VS Code structures its Main/Renderer split (it's open source). Identify one design decision that matches what you learned in this chapter.

---

*End of Chapter 1. Chapter 2 (Project Architecture) will cover the full Nexus Editor folder structure, explaining why every directory — `apps/desktop`, `apps/backend`, `packages/`, `docs/`, `scripts/`, `assets/` — exists and what belongs where.*

**Say "next chapter" or "continue" to generate Chapter 2.**
