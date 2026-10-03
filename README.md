# Carla Quiz

Aplicación de cuestionarios interactivos al estilo Kahoot, potenciada por IA. Funciona íntegramente en un único archivo `index.html` sin instalaciones, sin servidor y sin dependencias.

## Características

- **Generación de preguntas con IA** a partir de un tema libre o un PDF
- **Soporte para Google Gemini y Anthropic Claude** como proveedores de IA
- **Extracción de texto de PDF** para generar preguntas sobre el contenido
- **Código QR y PIN** para que los alumnos se unan desde su móvil; enlace copiable y botón Compartir
- **Diseño mobile-first**: botones grandes con formas ▲◆●■ (accesibles para daltónicos), bottom sheets, safe areas, pantalla siempre encendida durante la partida (Wake Lock)
- **Avatares** y reconexión automática si el alumno recarga la página; nombres duplicados bloqueados
- **Feedback en vivo para el alumno**: cuenta atrás, puntos ganados, posición en el ranking, vibración
- **Control del profesor**: revelar antes de tiempo, pantalla completa, sonidos opcionales y atajos de teclado
- **Ranking entre preguntas** opcional, con flechas de subida/bajada de puesto
- **Análisis final por pregunta** (% de aciertos) y **exportación CSV** con las respuestas de cada alumno
- **Editor mejorado**: marcar la correcta con un toque, duplicar, deshacer borrado, barajar orden, tiempo para todas, borrador autoguardado
- **IA con dificultad y más idiomas** (alemán, italiano, portugués, gallego, euskera…)
- **Guardado de cuestionarios** en la nube con cuenta de Google (Firebase Firestore)
- **Configuración de API keys** guardada por usuario en Firebase
- Sin frameworks, sin TypeScript, sin paso de compilación

## Cómo usar

### Opción 1: Abrir directamente en el navegador

```bash
open index.html
```

### Opción 2: Servidor local (recomendado para el QR)

```bash
python3 -m http.server 8080
# Abre http://localhost:8080
```

## Flujo de uso

### Profesor (creador del cuestionario)

1. Abre la app y pulsa **Crear Cuestionario**
2. Genera preguntas con IA (por tema o PDF) o añádelas manualmente
3. (Opcional) Activa/desactiva **Mostrar ranking entre preguntas**
4. Pulsa **Iniciar partida** — aparece un PIN de 6 dígitos y un código QR (puedes copiar o compartir el enlace)
5. Espera a que los alumnos se unan y pulsa **Empezar el juego**
6. Controla el ritmo: cada pregunta avanza manualmente tras ver las estadísticas

### Alumno

1. Escanea el QR o abre la app e introduce el PIN en **¿Tienes un PIN?**
2. Escribe tu nombre y elige un avatar
3. Responde las preguntas antes de que se acabe el tiempo

## Arquitectura

```
index.html          ← Toda la app: HTML + CSS + JavaScript
```

### Comunicación en tiempo real

La comunicación entre profesor y alumnos usa **Firebase Firestore**:

- `games/<PIN>`: estado de la partida (`lobby`, `question`, `reveal`, `gameover`) y datos de la pregunta/revelado
- `games/<PIN>/players/<nombre>`: jugadores (`name`, `avatar`, `clientId`)
- `games/<PIN>/answers/<nombre>`: última respuesta de cada alumno

### Pantallas (SPA)

La navegación es de una sola página. `showScreen(id)` activa/desactiva clases `.screen.active`.

| ID | Descripción |
|---|---|
| `screen-home` | Inicio |
| `screen-create` | Crear/editar cuestionario |
| `screen-lobby` | Sala de espera (profesor) |
| `screen-game` | Proyección de pregunta (profesor) |
| `screen-reveal` | Respuesta revelada con estadísticas |
| `screen-scoreboard` | Ranking entre preguntas |
| `screen-final` | Resultados finales |
| `screen-join` | Unirse como alumno |
| `screen-student-wait` | Espera del alumno |
| `screen-student-play` | Pantalla de juego del alumno |
| `screen-student-result` | Resultado final del alumno |

### Estado global (variables JS)

| Variable | Tipo | Descripción |
|---|---|---|
| `questions` | `Array` | Lista de preguntas `{question, answers[], correct, time}` |
| `players` | `Object` | Mapa `nombre → {score, correct, answers[], streak, avatar}` |
| `channel` | `BroadcastChannel` | Canal activo de comunicación |
| `gamePin` | `string` | PIN de 6 dígitos de la partida actual |
| `isTeacher` | `boolean` | Distingue la vista de profesor/alumno |
| `phase` | `string` | Fase del profesor: `idle`, `lobby`, `question`, `reveal`, `scoreboard`, `final` |
| `showRankingBetweenQuestions` | `boolean` | Muestra ranking tras cada pregunta |

### Flujo de juego (profesor)

```
startLobby() → startGame() → showQuestion() → timer → revealAnswer()
      ↑                                                      ↓
      └─────────────── continueAfterScoreboard() ←── nextQuestion()
                                                         ↓ (última)
                                                       endGame()
```

### Puntuación

- Base: `500 + 500 × (1 - tiempo_empleado / tiempo_total)` puntos si acierta
- Bonus de racha: +100 puntos a partir de 3 respuestas correctas consecutivas
- 0 puntos si falla o no responde

## Proveedores de IA

### Google Gemini
- Modelos: `gemini-2.5-flash` (con fallback a `gemini-3.1-flash-lite-preview`)
- Entrada: texto libre o texto extraído del PDF (via pdf.js)
- Configura tu key en [aistudio.google.com/apikey](https://aistudio.google.com/apikey)

### Anthropic Claude
- Modelo: `claude-sonnet-5-5`
- Entrada: texto libre o PDF completo en base64 (visión de documentos)
- Configura tu key en [console.anthropic.com](https://console.anthropic.com)

Las API keys se guardan en `sessionStorage` (se borran al cerrar la pestaña) o en Firebase si el usuario inicia sesión.

## Firebase (opcional)

Permite guardar y cargar cuestionarios y API keys entre sesiones con cuenta de Google.

Para configurar tu propio proyecto Firebase:

1. Ve a [console.firebase.google.com](https://console.firebase.google.com) y crea un proyecto
2. En **Authentication → Sign-in method**, activa Google
3. En **Firestore Database**, crea una base de datos en modo producción
4. Copia la configuración de **Project Settings → General → Tu aplicación web** en la constante `FIREBASE_CONFIG` de `index.html`

### Reglas de Firestore recomendadas

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /users/{userId}/{document=**} {
      allow read, write: if request.auth != null && request.auth.uid == userId;
    }
    match /quizzes/{userId}/{document=**} {
      allow read, write: if request.auth != null && request.auth.uid == userId;
    }
    // Partidas en directo: los alumnos no inician sesión
    match /games/{pin}/{document=**} {
      allow read, write: if true;
    }
  }
}
```

## Seguridad

- Todo el HTML dinámico pasa por `esc(s)` (escapa `& < > " '`) para prevenir XSS
- Las API keys nunca se envían a ningún servidor propio; se llaman directamente a las APIs de Google/Anthropic desde el navegador

## Convenciones del código

- JavaScript ES2020+ plano, sin frameworks ni TypeScript
- CSS con custom properties definidas en `:root`
- UI en español (`lang="es"`); el prompt de IA puede generar preguntas en cualquier idioma
- Sin paso de compilación — editar y recargar es suficiente
