# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Carla Quiz** is a Kahoot-like interactive quiz app powered by AI. It runs entirely in a single `index.html` file with no build system, no dependencies to install, and no server — just open the file in a browser.

Teachers create quizzes (manually or AI-generated from a topic or PDF), share a PIN/QR code, and students join from their phones. Real-time communication between teacher and student views goes through **Firebase Firestore** (`games/{pin}` documents and listeners).

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
- Teacher and students run on any device; they sync through Firestore:
  - `games/{pin}` — game doc with `state` (`lobby` | `question` | `reveal` | `gameover`), `questionData`, `revealData`, final `positions`/`finalRanking`
  - `games/{pin}/players/{name}` — `{name, avatar, clientId}` (clientId lets the same device reconnect and blocks duplicate names)
  - `games/{pin}/answers/{name}` — the student's latest answer `{questionIndex, answerIndex, timeSpent}`
- PIN is a random 6-digit number; students join by entering it on home/join or scanning the QR (`?pin=`)
- Students that reload mid-game are reconnected automatically (`sessionStorage.carlaquiz_session`)

### State (global JS variables)
Key state lives in module-level `let` variables:
- `questions[]` — array of `{question, answers[4], correct, time, image}` objects (autosaved as a draft in `localStorage.carlaquiz_draft`)
- `players{}` — map of `playerName → {score, correct, answers[], streak, avatar}`
- `phase` — teacher state machine (`idle` | `lobby` | `question` | `reveal` | `scoreboard` | `final`); guards double reveals and ignores late answers
- `questionStats[]` — per-question answer counts, used for the final "análisis por pregunta"
- `isTeacher` — boolean, determines which UI flow runs
- `pdfTextContent` / `pdfBase64Data` — extracted PDF content for AI prompts

### Screens (SPA navigation)
`showScreen(id)` swaps `.screen.active`. Screen IDs: `screen-home`, `screen-create`, `screen-join`, `screen-lobby`, `screen-game`, `screen-reveal`, `screen-scoreboard`, `screen-final`, `screen-student-wait`, `screen-student-play`, `screen-student-result`. Game screens are "immersive" (app bar hidden, not pushed to history so the back button can't land on a stale question).

### AI Generation
- Supports two providers selectable via UI: **Google Gemini** and **Anthropic Claude**
- Gemini: tries `gemini-2.5-flash` then falls back to `gemini-2.0-flash`
- Claude: uses `claude-sonnet-5-5` with native PDF document input (base64)
- API keys stored in `sessionStorage` (not `localStorage`) — cleared when tab closes
- `buildPrompt()` constructs a strict JSON-output prompt (level, difficulty, language); `parseQuestionsJson()` strips code fences and falls back to extracting the first `[...]`

### PDF Support
- `pdf.js` (CDN) extracts text for Gemini text prompt
- For Claude, the PDF is sent as base64 via the vision/document API
- Drag-and-drop and file input both supported

### Game Flow (Teacher side)
`startLobby()` → `startGame()` → `showQuestion()` → timer (or `skipQuestion()`) → `revealAnswer()` → `nextQuestion()` → `showScoreboard()` → `continueAfterScoreboard()` … → `endGame()` → `playAgain()` / `backToEditor()`

Keyboard: Enter starts from the lobby, `S` reveals early, Space/Enter/→ advances, `F` fullscreen. Students can answer with 1-4 / A-D.

### Scoring
Correct answer: `500 + 500 × (1 − timeSpent / time)`, +100 streak bonus from the 3rd consecutive hit. Ranking sorts by correct answers, then points. Results export as CSV (`exportResults()`, includes per-question answers).

## Key Conventions
- All HTML output uses `esc(s)` (escapes `& < > " '`, safe in attributes too) to prevent XSS
- CSS: design tokens are custom properties in `:root`; reusable components (`.btn`, `.input`, `.card`, `.chip`, `.answer-btn`, `.timer-ring`, `.modal-overlay`…) live in the `<style>` block; Tailwind (browser CDN) is used for layout utilities. Our CSS is unlayered, so it wins over Tailwind utilities — use `!` utilities to override a component
- Mobile first: modals are bottom sheets on phones, safe-area insets are respected, inputs use 16px text (no iOS zoom), and `.hidden` is `!important` — use `max-sm:hidden` rather than `hidden sm:block`
- Wrap `localStorage`/`sessionStorage` access with `store` / `sessionStore`; use `confirmDialog()` instead of `confirm()`
- No framework, no TypeScript — plain ES2020+ JavaScript
- Spanish UI (`lang="es"`) but the AI prompt supports multiple output languages
