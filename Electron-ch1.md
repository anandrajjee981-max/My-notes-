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

*End of Chapter 1.*

---

# Chapter 2 — Project Architecture

## 2.1 Why a Monorepo?

Nexus Editor is not one app — it's actually a small ecosystem: a desktop shell, a backend/AI service layer, shared types, build tooling, and documentation. Cramming all of that into a single flat folder gets unmanageable fast (imagine your React components, your Electron main-process code, your AI prompt templates, and your build scripts all fighting for space in one `src/`).

Instead, Nexus Editor uses a **monorepo** — one Git repository containing multiple, clearly separated packages/apps that can depend on each other, be built independently, and be versioned together. This is the same strategy used by VS Code, Slack's desktop client, and most serious Electron products at scale.

```mermaid
graph TD
    Root[Nexus-Editor/] --> Apps[apps/]
    Root --> Packages[packages/]
    Root --> Docs[docs/]
    Root --> Scripts[scripts/]
    Root --> Assets[assets/]

    Apps --> Desktop[desktop/ - Electron + React]
    Apps --> Backend[backend/ - AI service, optional local server]

    Packages --> Shared[shared/ - types, utils, constants]
    Packages --> UI[ui/ - shared React components]
    Packages --> Config[config/ - eslint, tsconfig, build configs]
```

## 2.2 Full Folder Structure

```
Nexus-Editor/
│
├── apps/
│   ├── desktop/                  # The Electron application itself
│   │   ├── src/
│   │   │   ├── main/             # Main Process code
│   │   │   │   ├── index.ts      # App entry point (app.whenReady, etc.)
│   │   │   │   ├── windows/      # BrowserWindow creation & management
│   │   │   │   ├── ipc/          # ipcMain handlers, grouped by domain
│   │   │   │   ├── fs/           # File system logic (read/write/watch)
│   │   │   │   ├── terminal/     # node-pty integration
│   │   │   │   └── git/          # simple-git integration
│   │   │   │
│   │   │   ├── preload/          # Preload scripts
│   │   │   │   └── index.ts      # contextBridge API exposure
│   │   │   │
│   │   │   └── renderer/         # Renderer Process = the React app
│   │   │       ├── src/
│   │   │       │   ├── components/
│   │   │       │   ├── pages/
│   │   │       │   ├── editor/   # Monaco integration
│   │   │       │   ├── ai/       # AI chat panel, context builder UI
│   │   │       │   ├── hooks/
│   │   │       │   └── state/    # Zustand/Redux store
│   │   │       ├── index.html
│   │   │       └── vite.config.ts
│   │   │
│   │   ├── electron-builder.yml  # Packaging config
│   │   └── package.json
│   │
│   └── backend/                  # AI orchestration service (can run locally or remote)
│       ├── src/
│       │   ├── agents/           # RAG pipelines, embeddings, prompt logic
│       │   ├── routes/           # If exposed via local HTTP server
│       │   └── index.ts
│       └── package.json
│
├── packages/
│   ├── shared/                   # Code shared between desktop & backend
│   │   ├── types/                # Shared TypeScript interfaces (IPC contracts!)
│   │   ├── constants/
│   │   └── utils/
│   │
│   ├── ui/                       # Shared, reusable React components (buttons, modals)
│   │
│   └── config/                   # Centralized eslint, prettier, tsconfig base files
│
├── docs/                         # This handbook, ADRs, architecture diagrams
│
├── scripts/                      # Build, release, codegen, CI helper scripts
│
├── assets/                       # App icons, installer graphics, splash screens
│
├── .github/
│   └── workflows/                # CI/CD pipelines
│
├── package.json                  # Root workspace config
├── pnpm-workspace.yaml           # (or turbo.json / nx.json)
└── tsconfig.base.json
```

## 2.3 Why Every Folder Exists

### `apps/desktop/` — Where Electron Lives
This is the actual shipped product: the Electron shell. It contains three clearly separated subfolders that map **directly** onto the architecture from Chapter 1:

- `main/` → Main Process code. Anything that touches the file system, spawns processes, manages windows, or talks to the OS lives here. Notice it's further split by *domain* (`fs/`, `terminal/`, `git/`, `ipc/`) rather than dumped into one file — at production scale, a single `main.js` with 3,000 lines is unmaintainable.
- `preload/` → The bridge scripts. Kept separate because they have a different build target and a different security posture than both main and renderer code.
- `renderer/` → This is where your **frontend lives** — the entire React application, including the Monaco editor integration, the AI chat UI, and all visual components. It's built with Vite and, critically, has *no* Node.js access at runtime (per Chapter 1's security model), even though it physically sits inside the same folder tree.

### `apps/backend/` — Where AI/Server Logic Lives
Even though Nexus Editor is a desktop app, the AI features (RAG pipeline, embeddings, prompt orchestration — Chapter 10) are complex enough to deserve their own package, separate from Electron's main process. This keeps `main/` focused on OS/window concerns, and keeps AI logic testable and potentially reusable (e.g., if you ever offer a cloud-hosted version of Nexus Editor's AI features).

### `packages/shared/` — Shared Code
The **most important** package for correctness. IPC (Chapter 5) relies on Main and Renderer agreeing on exact function signatures and payload shapes. If you define `readFile(path: string): Promise<string>` in the main process but the renderer expects `Promise<{content: string, encoding: string}>`, you get silent runtime bugs. `packages/shared/types/` holds the single source of truth for these contracts, imported by both `main/` and `renderer/`.

### `packages/ui/` — Shared Components
Purely visual, reusable React pieces (buttons, dialogs, tooltips) that might be used across multiple parts of the renderer, or even a future companion web app (e.g., a marketing site or docs site using the same design system).

### `packages/config/` — Shared Configs
Centralizes `eslint`, `prettier`, and base `tsconfig.json` so `apps/desktop` and `apps/backend` don't duplicate (and drift out of sync on) linting/formatting rules.

### `docs/` — Documentation
This handbook lives here, along with Architecture Decision Records (ADRs) — short markdown files documenting *why* a significant technical choice was made (e.g., "why pnpm over npm," "why Zustand over Redux"). Six months from now, you'll thank yourself.

### `scripts/` — Build & Automation
Custom Node.js/shell scripts for tasks that don't belong in `package.json` one-liners: generating icons for all platforms, bumping versions across all `package.json` files in the monorepo, running release checklists.

### `assets/` — Static/Build Assets
Not runtime assets used by the React app (those live in `renderer/public/`) — this is specifically for **build-time and installer** assets: app icons (`.ico`, `.icns`, `.png` at various sizes), DMG background images, installer splash screens. `electron-builder` (Chapter 12) reads directly from here.

### Root-level files
- `package.json` at the root defines the **workspace** (via `pnpm-workspace.yaml`), not a runnable app itself. It holds dev-tooling dependencies shared across the repo (TypeScript, ESLint, Prettier, Turborepo/Nx if used) and workspace-wide scripts like `pnpm build` that fan out to every app/package.
- `tsconfig.base.json` — the root TypeScript config that every package extends, enforcing consistent compiler options (strict mode, module resolution) project-wide.

## 2.4 Build vs. Production Folders

It's worth explicitly separating **source** from **output**, since this trips up beginners:

| Folder | Purpose | Committed to Git? |
|---|---|---|
| `apps/desktop/src/` | Hand-written source code | ✅ Yes |
| `apps/desktop/dist/` | Compiled output (Vite build of renderer, tsc output of main) | ❌ No (`.gitignore`) |
| `apps/desktop/release/` | Final packaged installers (`.exe`, `.dmg`, `.AppImage`) from `electron-builder` | ❌ No |
| `node_modules/` | Installed dependencies (per-package, hoisted by pnpm) | ❌ No |

`dist/` is intermediate — it's what actually gets loaded by Electron in production mode (Chapter 4 explains dev vs. production loading in detail). `release/` is the final, distributable artifact — what a user downloads and double-clicks to install.

---

## Common Mistakes (Chapter 2)

1. **Mixing main-process and renderer-process code in the same folder/file.** If `src/utils/fileHelpers.ts` is imported by *both* a main-process file and a renderer component, you risk accidentally bundling Node.js-only code (like `fs`) into the renderer bundle, which will break or create security holes.
2. **Putting IPC type definitions in only one side (main or renderer).** This guarantees drift. Always define IPC contracts once, in `packages/shared/types/`, and import from both sides.
3. **Treating `assets/` and `public/` as the same thing.** `public/` (inside the renderer) is for runtime UI assets bundled into the app; `assets/` at the root is for build-time installer assets. Confusing them causes broken icons in production builds.
4. **Committing `dist/`, `release/`, or `node_modules/` to Git.** Bloats the repo and causes merge conflicts on generated files.
5. **Flat, unstructured `main/` folder.** A single giant `main.js` handling windows, IPC, file system, terminal, and Git all in one file becomes unreadable past a few hundred lines. Split by domain early.

---

## Interview Questions (Chapter 2)

1. Why does a production Electron app benefit from a monorepo structure instead of one flat repository?
2. What kind of code should live in `packages/shared/` versus `packages/ui/`? Give an example of each.
3. Why is it dangerous for a utility file to be imported by both `main/` and `renderer/` code without care?
4. What's the difference between `assets/` at the project root and a `public/` folder inside the renderer app?
5. Explain the purpose of `dist/` vs `release/` in the build pipeline.
6. Why should IPC payload types be defined in a shared package rather than duplicated in main and renderer?

---

## Practical Exercises (Chapter 2)

**Exercise 2.1 — Structure From Scratch**
Without copying the example above, sketch your own version of the Nexus Editor folder tree from memory, based only on the responsibilities you learned in Chapter 1 (Main/Renderer/Preload). Then compare it to the structure in this chapter and note any differences.

**Exercise 2.2 — Contract Design**
Design (on paper, in TypeScript syntax, no implementation) the shared type for a single IPC contract: `fs:read-file`. Define the request payload and the response payload as they'd live in `packages/shared/types/`.

**Exercise 2.3 — Audit**
Pick any real open-source Electron app on GitHub (VS Code, Obsidian's community plugins, or similar). Identify: where is their main process code, where is their renderer code, and do they use a monorepo structure? Compare to Nexus Editor's approach.

---

*End of Chapter 2.*

---

# Chapter 3 — Installation Guide

This chapter sets up every tool Nexus Editor needs, explains **why** each one is needed (not just how to install it), and ends with a full breakdown of `package.json` and the ESM vs. CommonJS decision that affects every file you'll write from here on.

## 3.1 Node.js

**What it is:** A JavaScript runtime built on Chrome's V8 engine that lets JavaScript run outside a browser — on your machine, in a terminal, or (in our case) inside Electron's Main Process.

**Why Nexus Editor needs it:** Electron's Main Process *is* a Node.js process. Every file system operation, terminal spawn, and Git command in this handbook runs through Node.js APIs. You also need Node.js locally to run build tools (Vite, TypeScript compiler, pnpm) during development.

**Install:**
- Download the **LTS (Long-Term Support)** version from nodejs.org — never the "Current" version for a production app, since LTS receives stability-focused updates.
- Verify:
```bash
node --version   # should print v20.x.x or later
npm --version
```

**Version note:** Electron bundles its *own* internal Node.js version inside the packaged app. The Node.js you install locally is only used for development tooling — but keeping your local version close to Electron's bundled version avoids subtle API mismatches, especially with native modules (relevant in Chapter 8 for `node-pty`).

## 3.2 npm

**What it is:** Node Package Manager — installed automatically with Node.js. It's the default tool for installing JavaScript/TypeScript packages from the npm registry.

**Why we mention it but don't use it as primary:** npm works fine for small projects, but for a monorepo like Nexus Editor (Chapter 2), npm's workspace support is weaker and its `node_modules` handling is less disk-efficient than pnpm's. We use npm's *registry* (all packages still come from npmjs.com) but not npm as the *installer*.

## 3.3 pnpm

**What it is:** A faster, disk-space-efficient alternative to npm/yarn, with first-class monorepo ("workspace") support.

**Why Nexus Editor uses it specifically:**
- **Disk efficiency:** pnpm uses a single global content-addressable store and hard-links packages into each project, instead of duplicating `node_modules` per package. In a monorepo with `apps/desktop`, `apps/backend`, and multiple `packages/`, this saves gigabytes.
- **Strict dependency resolution:** pnpm doesn't "hoist" dependencies loosely like npm does, which prevents a common class of bugs where a package accidentally works because of a dependency it never explicitly declared (a "phantom dependency").
- **Native workspace protocol:** `workspace:*` lets `apps/desktop` depend on `packages/shared` as a local, always-in-sync package rather than a published npm version.

**Install:**
```bash
npm install -g pnpm
pnpm --version
```

## 3.4 Git

**What it is:** Distributed version control — tracks every change to your codebase, enables branching, and (relevantly for Chapter 9) is the same tool Nexus Editor's built-in Git panel will operate on programmatically via `simple-git`.

**Why it's foundational, not optional:** Beyond your own workflow, Nexus Editor *is itself* a Git client for the projects opened inside it. Understanding Git at the command line is a prerequisite for building and debugging Chapter 9's Git integration.

**Install:** Download from git-scm.com, then configure identity:
```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

## 3.5 VS Code

**What it is:** The editor you'll use to *build* Nexus Editor (a pleasant bit of irony — you're building a code editor, inside a code editor).

**Why VS Code specifically:** It's built on Electron itself, so its debugging tools (especially the integrated terminal and the ability to attach a debugger to a Node.js process) map directly onto the kind of debugging you'll do with Electron's Main Process. Recommended extensions: ESLint, Prettier, and the TypeScript language features that ship built-in.

## 3.6 Electron

**What it is:** The framework itself, as an npm package (`electron`).

**Why it's a devDependency, not a dependency:** This surprises beginners. `electron` is installed as a **devDependency** because it's not code your app imports at runtime in the traditional sense — it's a *binary* (a prebuilt Chromium + Node.js executable) that gets downloaded and used to *run* your app during development, and gets bundled by `electron-builder` (Chapter 12) for production. Your app's actual runtime code (main/preload/renderer) doesn't `import` the electron npm package as a library the way you'd import `react`.

```bash
pnpm add -D electron
```

## 3.7 React

**What it is:** The UI library powering the entire Renderer Process — every component, from the file tree to the AI chat panel, is a React component.

**Why React here specifically:** It's a Renderer-Process-only dependency — it never runs in the Main Process. Given you already know React (per this handbook's assumed audience), the learning curve here is purely about *where* React fits in Electron's architecture (Chapter 1), not React itself.

```bash
pnpm add react react-dom
```

## 3.8 Vite

**What it is:** A modern, fast build tool and dev server for frontend code, replacing older tools like Webpack for most new projects.

**Why Vite for the renderer:** Vite's dev server offers near-instant Hot Module Replacement (HMR), which matters enormously when iterating on UI-heavy features like the Monaco editor integration or the AI chat panel. For Electron specifically, we use `vite-plugin-electron` (or a manual dual-config setup) so Vite can build the renderer *and* watch/rebuild the main and preload TypeScript files during development.

```bash
pnpm add -D vite @vitejs/plugin-react
```

## 3.9 TypeScript

**What it is:** A typed superset of JavaScript that compiles down to plain JS.

**Why it's non-negotiable for Nexus Editor:** The single biggest source of Electron bugs is a **mismatch between what the Main Process sends over IPC and what the Renderer expects to receive** (Chapter 2, 2.3). TypeScript, combined with the shared types in `packages/shared/types/`, catches these mismatches at compile time instead of as a silent runtime `undefined`.

```bash
pnpm add -D typescript
```

---

## 3.10 ESM vs. CommonJS

This is one of the most confusing topics for developers new to the Node.js/Electron ecosystem, so let's be precise.

**CommonJS (CJS)** is Node.js's original module system:
```js
const fs = require('fs');
module.exports = { readFile };
```

**ECMAScript Modules (ESM)** is the modern, standardized JavaScript module system (the same `import`/`export` syntax used in the browser and in React):
```js
import fs from 'fs';
export { readFile };
```

### Why this matters specifically in Electron

- The **Renderer Process** (your React/Vite code) uses **ESM** exclusively — this is standard for any modern frontend tooling, no debate needed.
- The **Main Process and Preload script** run in Node.js, where **both** module systems are valid, but they have different loading behaviors, different `__dirname` availability (ESM doesn't have `__dirname` natively), and — critically — **some native Node.js addons and certain Electron APIs have historically had friction with ESM** in the main process.

### What Nexus Editor uses, and why

**Decision: CommonJS for the Main Process and Preload script, ESM for the Renderer.**

Reasoning:
1. Electron's own internals and many native modules (`node-pty` for Chapter 8's terminal integration is a prime example) have the most battle-tested compatibility with CommonJS in the main process. Using ESM there is *possible* in recent Electron versions but introduces edge cases (like needing `.mjs` extensions or careful `package.json` `"type"` configuration) that aren't worth the complexity for a production app.
2. The Renderer, being bundled by Vite, uses ESM because that's simply how modern frontend tooling works — Vite assumes ESM by default and it enables proper tree-shaking.
3. TypeScript compiles both to their respective targets regardless of which syntax you write in your `.ts` source files, so in practice you can *write* `import`/`export` syntax everywhere in TypeScript, and let the `tsconfig.json` `"module"` setting per-package (`CommonJS` for `main/`, `ESNext` for `renderer/`) determine the actual output format. This gives you consistent authoring syntax with correct compiled output for each environment.

```mermaid
graph LR
    A[main/*.ts - written with import/export] -->|tsc, module: CommonJS| B[dist/main/*.js - require/module.exports]
    C[renderer/*.tsx - written with import/export] -->|Vite, ESM native| D[dist/renderer/*.js - ESM bundles]
```

---

## 3.11 package.json, Fully Explained

At the root of the monorepo:

```json
{
  "name": "nexus-editor",
  "private": true,
  "packageManager": "pnpm@9.0.0",
  "scripts": {
    "dev": "turbo run dev",
    "build": "turbo run build",
    "lint": "turbo run lint"
  },
  "devDependencies": {
    "typescript": "^5.5.0",
    "eslint": "^9.0.0",
    "prettier": "^3.3.0",
    "turbo": "^2.0.0"
  }
}
```

- **`"name"`** — the package identifier. Even for a private, unpublished monorepo root, npm/pnpm requires this field.
- **`"private": true`** — prevents this package (and by extension, accidental `pnpm publish` of your root) from ever being published to the npm registry. Essential for any application (as opposed to a library).
- **`"packageManager"`** — pins the exact pnpm version for the whole team/CI, preventing "works on my machine" issues from pnpm version drift.
- **`"scripts"`** — here we delegate to **Turborepo** (`turbo run dev`), which fans a single command out to every app/package's own `dev` script in the correct dependency order, with caching. `apps/desktop/package.json` and `apps/backend/package.json` each define their own local `dev`/`build` scripts underneath.
- **`"devDependencies"`** — tools needed to *develop* the project but not shipped inside the final app.

Inside `apps/desktop/package.json`, you'd additionally see:

```json
{
  "name": "@nexus-editor/desktop",
  "main": "dist/main/index.js",
  "dependencies": {
    "electron-updater": "^6.0.0",
    "simple-git": "^3.25.0"
  },
  "devDependencies": {
    "electron": "^31.0.0",
    "electron-builder": "^24.13.0",
    "vite": "^5.3.0",
    "react": "^18.3.0"
  }
}
```

- **`"main"`** — this is the field Electron itself reads on launch: it's the entry point to your **compiled** Main Process code (not source). This is what runs when the app starts.
- Note `electron` and `electron-builder` are devDependencies (Section 3.6), while things like `electron-updater` and `simple-git` are real runtime `dependencies`, because that code actually executes inside the shipped app.

---

## Common Mistakes (Chapter 3)

1. **Installing `electron` as a regular dependency instead of a devDependency.** This bloats your production dependency tree unnecessarily and misrepresents what actually ships (the Electron *binary* is bundled separately by electron-builder, not pulled from `node_modules` at runtime the normal way).
2. **Mixing npm and pnpm in the same repo.** Having both a `package-lock.json` and a `pnpm-lock.yaml` causes non-deterministic installs across team members' machines and CI.
3. **Trying to use ESM everywhere "for consistency" without understanding the main-process tradeoffs.** Leads to obscure `ERR_REQUIRE_ESM` errors and native module loading failures with things like `node-pty`.
4. **Forgetting `"private": true`** on an application's root `package.json`, risking an accidental publish.
5. **Not pinning the Electron version.** Using `"electron": "*"` or a loose range means your dev environment and CI could silently pick up a new major Electron version with breaking API changes.

---

## Interview Questions (Chapter 3)

1. Why is `electron` typically listed as a devDependency rather than a dependency, even though the whole app depends on it to run?
2. What specific advantage does pnpm's workspace model give a monorepo like Nexus Editor over plain npm?
3. Explain, in your own words, the practical difference between CommonJS and ESM, and why Electron's Main Process commonly favors CommonJS.
4. What does the `"main"` field in `apps/desktop/package.json` point to, and why does it point to `dist/` rather than `src/`?
5. Why might a native Node.js module like `node-pty` behave differently under ESM vs. CommonJS in the main process?

---

## Practical Exercises (Chapter 3)

**Exercise 3.1 — Environment Setup**
Install Node.js (LTS), pnpm, and Git on your machine. Run `node -v`, `pnpm -v`, and `git --version` and confirm all three succeed.

**Exercise 3.2 — Module System Investigation**
Create two tiny scratch files: one `.cjs` using `require`/`module.exports`, and one `.mjs` using `import`/`export`. Run both with `node file.cjs` and `node file.mjs` and observe any differences (e.g., try accessing `__dirname` in each).

**Exercise 3.3 — package.json Audit**
Open any Electron open-source project's `package.json` on GitHub (e.g., a small Electron boilerplate repo). Identify which dependencies are devDependencies vs. dependencies, and reason about *why* each one is classified that way, using this chapter's logic.

---

*End of Chapter 3. Chapter 4 (Electron Lifecycle) covers `app.whenReady()`, `BrowserWindow` creation and management, window/app events, the differences between dev mode and production mode loading, and the reload/close lifecycle in full detail.*

**Say "next chapter" or "continue" to generate Chapter 4.**
