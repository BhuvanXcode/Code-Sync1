# 🔄 Code Sync — Real-Time Collaborative Code Editor

> A powerful, browser-based collaborative coding platform with live editing, AI assistance, whiteboard, group chat, and multi-language code execution — all in one shared workspace.

---

## 📌 Table of Contents

- [Overview](#overview)
- [Key Features](#key-features)
- [Tech Stack](#tech-stack)
- [Algorithms & Models Used](#algorithms--models-used)
- [Project Outcomes & Impact](#project-outcomes--impact)
- [SDG Mapping](#sdg-mapping)
- [Getting Started](#getting-started)
- [Docker Setup](#docker-setup)
- [Project Structure](#project-structure)
- [License](#license)

---

## 🧩 Overview

**Code Sync** is a full-stack real-time collaborative code editor that enables multiple developers to write, run, and review code together — from anywhere. It combines the power of WebSockets, AI code generation, a synchronized whiteboard, and an integrated file system into one seamless development environment.

Inspired by tools like VS Code Live Share and Replit, Code Sync is built for remote teams, coding interviews, and educational use cases.

---

## ✨ Key Features

### 1. 🖊️ Real-Time Collaborative Code Editing
- Multiple users can edit files simultaneously in a shared room
- Live cursor & selection highlighting per user (color-coded)
- Typing indicators and presence awareness via Socket.IO

### 2. 💬 Integrated Group Chat
- Built-in room-based chat for seamless team communication
- Messages broadcast instantly to all room members
- No context switching — chat lives alongside the editor

### 3. 🤖 AI-Powered Code Copilot
- Generate code from natural language prompts using the **Mistral LLM**
- Returns language-specific, formatted code ready to use
- Powered by the Pollinations API

### 4. 🗂️ Collaborative File System
- Create, rename, delete files and directories in real time
- All changes are synced across every connected client instantly
- Download entire project as a `.zip` file

### 5. 🎨 Synchronized Whiteboard
- Integrated drawing board powered by **tldraw**
- Sketch architecture diagrams alongside your code
- Whiteboard state synced across all room members in real time

### 6. ▶️ Live Code Execution
- Run code directly in the browser across multiple languages
- Powered by the **Piston API** with stdin/stdout support
- Language auto-detected from file extension

---

## 🛠️ Tech Stack

| Layer | Technology |
|---|---|
| Frontend | React 18, TypeScript, Vite, Tailwind CSS |
| Code Editor | CodeMirror 6 (`@uiw/react-codemirror`) |
| Real-Time | Socket.IO (client + server) |
| Backend | Node.js, Express, TypeScript |
| Whiteboard | tldraw v2 |
| AI Copilot | Mistral LLM via Pollinations API |
| Code Execution | Piston API |
| Containerization | Docker, Docker Compose |

---

## ⚙️ Algorithms & Models Used

| Algorithm / Model | Purpose |
|---|---|
| **Polynomial Hash Function** | Consistent color assignment per username for cursor highlighting |
| **Debounce Algorithm** | Throttles cursor/typing socket events (100ms–1000ms delay) |
| **Recursive Tree Traversal** | File system CRUD operations on nested directory structures |
| **Event-Driven Broadcast (Pub/Sub)** | Room-scoped socket event distribution via Socket.IO |
| **Mistral LLM** | Transformer-based AI code generation from natural language |
| **CRDT-like Snapshot Merging** | tldraw delta-based whiteboard sync across distributed clients |

---

## 📈 Project Outcomes & Impact

1. Built a production-ready real-time collaborative code editor using React, Socket.IO, and CodeMirror with live cursor and file sync.
2. Mirrors industry tools like VS Code Live Share and Replit, making it directly relevant for remote teams and technical interviews.
3. Integrates an AI Copilot powered by the Mistral model, reflecting the industry shift toward AI-assisted developer workflows.
4. Uniquely combines a live whiteboard, group chat, file system, and code execution in one shared workspace — eliminating tool-switching.
5. Demonstrates end-to-end technical depth from real-time socket architecture to Docker deployment, bridging academic work with real-world engineering standards.

---

## 🌍 SDG Mapping

| SDG | Goal | How Code Sync Contributes |
|---|---|---|
| **SDG 4** | Quality Education | Enables browser-based collaborative coding for online learning, bootcamps & remote classrooms |
| **SDG 8** | Decent Work & Economic Growth | Boosts developer productivity and supports the growing remote work economy |
| **SDG 9** | Industry, Innovation & Infrastructure | Contributes open, scalable digital infrastructure with AI + real-time collaboration |
| **SDG 17** | Partnerships for the Goals | Room-based model enables cross-geography teamwork and collaborative innovation |

---

## 🚀 Getting Started

### Prerequisites

- Node.js v18+
- npm or yarn

### Manual Setup

**1. Clone the repository**
```bash
git clone https://github.com/your-username/code-sync.git
cd code-sync
```

**2. Start the Server**
```bash
cd server
npm install
npm run dev
```
Server runs at `http://localhost:3000`

**3. Start the Client**
```bash
cd client
npm install
npm run dev
```
Client runs at `http://localhost:5173`

**4. Environment Variables**

Create a `.env` file in the `client/` directory:
```env
VITE_BACKEND_URL=http://localhost:3000
```

---

## 🐳 Docker Setup

Run the entire application with a single command:

```bash
docker-compose up --build
```

| Service | Port |
|---|---|
| Server | `http://localhost:3000` |
| Client | `http://localhost:5173` |

---

## 📁 Project Structure

```
code-sync/
├── client/                  # React frontend
│   ├── src/
│   │   ├── components/      # UI components (Editor, Chat, Drawing, etc.)
│   │   ├── context/         # React Contexts (Socket, File, Copilot, etc.)
│   │   ├── hooks/           # Custom React hooks
│   │   ├── pages/           # Page components
│   │   └── types/           # TypeScript types
│   └── Dockerfile
│
├── server/                  # Express + Socket.IO backend
│   ├── src/
│   │   ├── server.ts        # Main server entry point
│   │   └── types/           # Server-side TypeScript types
│   └── Dockerfile
│
└── docker-compose.yml       # Multi-container Docker setup
```

---

## 📄 License

This project is licensed under the **MIT License** — see the [LICENSE](./LICENSE) file for details.

---

<div align="center">
  <p>Built with ❤️ for collaborative development</p>
  <p><strong>Code Sync</strong> — Write Together, Build Together</p>
</div>
