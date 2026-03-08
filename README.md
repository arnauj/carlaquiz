# Carla Quiz

Aplicación de cuestionarios interactivos al estilo Kahoot, potenciada por IA. Funciona íntegramente en un único archivo `index.html` sin instalaciones, sin servidor y sin dependencias.

## Características

- **Generación de preguntas con IA** a partir de un tema libre o un PDF
- **Soporte para Google Gemini y Anthropic Claude** como proveedores de IA
- **Extracción de texto de PDF** para generar preguntas sobre el contenido
- **Código QR y PIN** para que los alumnos se unan desde cualquier dispositivo del mismo origen
- **Ranking entre preguntas** opcional: muestra la clasificación tras cada pregunta
- **Exportación de resultados** en CSV al finalizar la partida
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
4. Pulsa **Iniciar partida** — aparece un PIN de 6 dígitos y un código QR
5. Espera a que los alumnos se unan y pulsa **Empezar el juego**
6. Controla el ritmo: cada pregunta avanza manualmente tras ver las estadísticas

### Alumno

1. Abre la URL de la app (o escanea el QR) en el mismo dispositivo o en otro en la misma red
2. Pulsa **Unirse como alumno**, introduce el PIN y tu nombre
3. Responde las preguntas antes de que se acabe el tiempo

## Arquitectura

```
index.html          ← Toda la app: HTML + CSS + JavaScript
```

### Comunicación en tiempo real

La comunicación entre profesor y alumno usa la **BroadcastChannel API** del navegador:

- Canal: `carlaquiz-<PIN>`
- No requiere servidor, WebSockets ni conexión a internet
- Solo funciona entre pestañas del mismo origen (mismo dominio y protocolo)

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
| `players` | `Object` | Mapa `nombre → {score, answers[], streak}` |
| `channel` | `BroadcastChannel` | Canal activo de comunicación |
| `gamePin` | `string` | PIN de 6 dígitos de la partida actual |
| `isTeacher` | `boolean` | Distingue la vista de profesor/alumno |
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
- Modelos: `gemini-2.5-flash` (con fallback a `gemini-2.0-flash`)
- Entrada: texto libre o texto extraído del PDF (via pdf.js)
- Configura tu key en [aistudio.google.com/apikey](https://aistudio.google.com/apikey)

### Anthropic Claude
- Modelo: `claude-sonnet-4-20250514`
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
  }
}
```

## Seguridad

- Todo el HTML dinámico pasa por `esc(s)` (escapa usando `textContent`) para prevenir XSS
- Las API keys nunca se envían a ningún servidor propio; se llaman directamente a las APIs de Google/Anthropic desde el navegador
- BroadcastChannel está limitado al mismo origen: no hay exposición de red

## Convenciones del código

- JavaScript ES2020+ plano, sin frameworks ni TypeScript
- CSS con custom properties definidas en `:root`
- UI en español (`lang="es"`); el prompt de IA puede generar preguntas en cualquier idioma
- Sin paso de compilación — editar y recargar es suficiente
