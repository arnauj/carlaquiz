# GEMINI.md - Carla Quiz Context

This file provides foundational context and mandates for Gemini CLI when working on the **Carla Quiz** project.

## Project Overview
**Carla Quiz** is a Kahoot-inspired interactive quiz application powered by AI (Google Gemini and Anthropic Claude). It is a "Serverless Single-File App" designed for maximum portability and ease of use.

- **Architecture:** Single-page application (SPA) contained entirely within `index.html`.
- **Core Technologies:** Vanilla HTML5, CSS3 (Custom Properties), and JavaScript (ES2020+).
- **Communication:** Uses the **BroadcastChannel API** for real-time synchronization between "Teacher" and "Student" tabs on the same origin (no WebSockets or backend required).
- **AI Integration:** Generates questions from text or PDFs using Google Gemini (`gemini-3-flash-preview` with `gemini-3.1-flash-lite-preview` fallback) and Anthropic Claude (`claude-3-5-sonnet-20241022`).
- **Persistence (Optional):** Firebase Firestore for cloud saving of quizzes and API keys.
- **Dependencies:** None required for core functionality; uses CDNs for `pdf.js` and `qrcode.js`.

## Building and Running
No build step or installation is required.

- **Direct Execution:** Open `index.html` in any modern web browser.
- **Local Server (Recommended):** To facilitate QR code scanning and multi-device testing (on the same network/origin), run a local static server:
  ```bash
  # Using Python
  python3 -m http.server 8080
  
  # Using Node (if installed)
  npx serve .
  ```
- **Testing:** Open two tabs of the same URL—one as "Teacher" and one as "Student"—to test the game loop locally.

## Development Conventions

### 1. Single-File Integrity
- **Mandate:** All new features, styles, and logic **must** be integrated into `index.html` unless a modular refactor is explicitly requested.
- **Structure:** Maintain the existing order: `<style>` blocks in `<head>`, HTML structure in `<body>`, and `<script type="module">` at the end of `<body>`.

### 2. JavaScript & State Management
- **ES Modules:** The script is defined as `type="module"`.
- **Global State:** Key application state is managed via top-level variables (e.g., `questions`, `players`, `currentQuestionIndex`).
- **Navigation:** UI transitions are handled by the `showScreen(screenId)` function, which toggles the `.active` class on `.screen` elements.
- **Security:** Always use the `esc(str)` helper function when rendering dynamic content to the DOM to prevent XSS.

### 3. Styling
- **CSS Variables:** Use the custom properties defined in `:root` for colors and spacing to ensure visual consistency.
- **Layout:** Primarily uses Flexbox and Grid.

### 4. AI Prompting
- The application uses a strict JSON-output prompt strategy. When modifying AI logic, ensure the `buildPrompt()` function remains compatible with the expected JSON schema:
  ```json
  [
    {
      "question": "Text",
      "answers": ["A", "B", "C", "D"],
      "correct": 0,
      "time": 20
    }
  ]
  ```

### 5. Localization
- The UI is currently in **Spanish (`lang="es"`)**. Maintain Spanish for all user-facing labels and messages unless instructed otherwise.

## Key Files
- `index.html`: The entire application (HTML, CSS, JS).
- `README.md`: High-level project documentation and usage guide.
- `CLAUDE.md`: Implementation details and specific guidance for AI assistants.
