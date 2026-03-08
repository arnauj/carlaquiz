# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Carla Quiz** is a Kahoot-like interactive quiz app powered by AI. It runs entirely in a single `index.html` file with no build system, no dependencies to install, and no server — just open the file in a browser.

Teachers create quizzes (manually or AI-generated from a topic or PDF), share a PIN/QR code, and students join from the same device or other browser tabs. Real-time communication between teacher and student views uses the browser's **BroadcastChannel API** (same-origin only, no network required).

## Development

No build step. To run:
```
# Open directly in browser
open index.html
# or serve locally
python3 -m http.server 8080
```

The entire app is one file: `index.html` contains all HTML, CSS, and JavaScript.

## Architecture

### Communication
- Teacher and student views run in separate browser tabs on the **same device**
- They communicate via `BroadcastChannel('carlaquiz-' + pin)` — no backend, no WebSockets
- PIN is a random 4-digit number; students join by entering it (or scanning a QR code)

### State (global JS variables)
Key state lives in module-level `let` variables:
- `questions[]` — array of `{q, answers[], correct, time}` objects
- `players{}` — map of `playerName → {score, answers[]}`
- `channel` — the active BroadcastChannel instance
- `isTeacher` — boolean, determines which UI flow runs
- `pdfTextContent` / `pdfBase64Data` — extracted PDF content for AI prompts

### Screens (SPA navigation)
`showScreen(id)` swaps `.screen.active`. Screen IDs: `screen-home`, `screen-create`, `screen-lobby`, `screen-game`, `screen-scoreboard`, `screen-join`, `screen-student-game`, `screen-student-wait`.

### AI Generation
- Supports two providers selectable via UI: **Google Gemini** and **Anthropic Claude**
- Gemini: tries `gemini-2.5-flash` then falls back to `gemini-2.0-flash`
- Claude: uses `claude-sonnet-4-20250514` with vision for PDF (base64)
- API keys stored in `sessionStorage` (not `localStorage`) — cleared when tab closes
- `buildPrompt()` constructs a strict JSON-output prompt; response is parsed with a regex fallback for markdown code blocks

### PDF Support
- `pdf.js` (CDN) extracts text for Gemini text prompt
- For Claude, the PDF is sent as base64 via the vision/document API
- Drag-and-drop and file input both supported

### Game Flow (Teacher side)
`startLobby()` → `startGame()` → `showQuestion()` → timer → `revealAnswer()` → `showScoreboard()` → `nextQuestion()` or `endGame()`

### Scoring
Points = `1000 * (timeLeft / totalTime)` rounded, only awarded for correct answers. Results exportable as CSV via `exportResults()`.

## Key Conventions
- All HTML output uses `esc(s)` (a `textContent`-based escaper) to prevent XSS
- CSS uses CSS custom properties defined in `:root` — edit colors there
- No framework, no TypeScript — plain ES2020+ JavaScript
- Spanish UI (`lang="es"`) but the AI prompt supports multiple output languages
