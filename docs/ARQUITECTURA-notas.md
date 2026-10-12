# Arquitectura de VoiceStudio (notas)

Mapa de alto nivel armado leyendo la estructura de carpetas y los puntos de entrada
(2026-10-12, versión 0.5.7). No es una lectura módulo por módulo.

App local de tres capas: Electron hace de cáscara de escritorio, un backend FastAPI
en Python hace el trabajo pesado, y los modelos de voz corren como motores
intercambiables detrás de ese backend.

```
┌─ Electron (electron/) ──────────────────────────────────────┐
│  main/      proceso Node: ventana, IPC, supervisor del      │
│             backend, updater, dictado, permisos             │
│  preload/   puente seguro main ↔ renderer                   │
│  renderer/  UI React + Vite (routes/, features/, i18n/)     │
└───────────────┬─────────────────────────────────────────────┘
                │ HTTP/WebSocket localhost:3900
┌─ Backend FastAPI (backend/) ────────────────────────────────┐
│  api/routers/   ~50 routers (generation, dub_*, audiobook,  │
│                 dictation, profiles, openai_compat…)        │
│  services/      lógica de negocio (TTS en chunks, ASR, DSP) │
│  core/          infraestructura (DB, cola de jobs, eventos, │
│                 seguridad de paths, analytics opt-in)       │
│  mcp_server.py  expone la app como herramientas para agentes│
│  engines/       adaptadores de motores TTS/ASR              │
└───────────────┬─────────────────────────────────────────────┘
                │
┌─ Motores / modelos ─────────────────────────────────────────┐
│  omnivoice/ (modelo base), IndexTTS, CosyVoice, Supertonic3,│
│  PocketTTS, VoxCPM2, MOSS… + native/ (helper en Rust)       │
└─────────────────────────────────────────────────────────────┘
```

## Las piezas

### 1. Electron (`electron/`)
- `src/main/` arranca la ventana y lanza o supervisa el backend (en el log de dev:
  `[supervisor] stage=attaching port=3900`).
- `src/renderer/` es una SPA con React y Vite. En desarrollo la sirve el puerto 3902.
- `native/desktop-bridge/` es un helper en Rust para cosas que Node no hace bien
  (captura de audio, atajos globales).

### 2. Backend (`backend/`)
- `main.py` crea la app FastAPI: monta los routers, el servidor MCP en `/mcp`, los
  archivos de audio en `/audio` y, al final, la SPA en `/`.
- Persistencia: SQLite (`core/db.py`) con migraciones Alembic
  (`migrations/versions/`). Los datos del usuario viven en `omnivoice_data/`.
- Trabajo largo: `core/job_queue.py` y `core/event_bus.py` manejan jobs asíncronos
  con progreso en tiempo real.
- Compatibilidad: `openai_compat.py` imita la API de OpenAI para usarlo como TTS
  desde otras herramientas.

### 3. Motores (`backend/engines/`)
- Cada motor tiene su carpeta y se carga como adaptador. Varios corren en
  subprocesos (`*_subprocess/`) con su propio entorno Python.
- Se aíslan porque traen dependencias que chocan entre sí: instalar uno no rompe a
  los otros.
- `_echo/` es el ejemplo mínimo para copiar al agregar un motor nuevo.

### 4. Configuración compartida
- `config/app.json` define los puertos (3900 backend, 3901 web, 3902 Electron dev).
  Es la única fuente: el código lee de ahí en vez de hardcodear.

## Decisiones de diseño a recordar
- **Backend separado de la UI:** es un servidor HTTP normal, así que el mismo código
  sirve a Electron, al modo web (`dev:web`), a Docker y a agentes vía MCP.
- **Motores en subprocesos:** sostiene la regla de que los usuarios con motores ya
  instalados no tengan que reinstalar.
- **Una sola fuente de verdad por dato:** la versión sale del `package.json` raíz y
  los puertos de `config/app.json`. Hay tests (`tests/test_app_version.py`) que
  fallan si los espejos se desincronizan.

## Reglas del proyecto (de `CLAUDE.md`) que condicionan la customización
- **Local primero:** nada sale de la máquina sin consentimiento explícito. Toda
  llamada de red nueva debe respetarlo.
- **Paridad multiplataforma:** una función por defecto debe andar igual en macOS,
  Windows y Linux.
- **Textos de UI solo vía i18n:** prohibido hardcodear texto CJK fuera de
  `electron/src/renderer/src/i18n/`.
- **Esquema de DB solo por Alembic.** No tocar `omnivoice_data/` a mano.
- **Commits sin trailers de co-autoría de IA** (check `commit-identity` en PRs al
  repo original).

## Dónde tocar según el objetivo
| Objetivo | Dónde tocar |
|---|---|
| Cambiar la UI | `electron/src/renderer/src/features/` y `routes/` |
| Agregar un endpoint | un router nuevo en `backend/api/routers/` |
| Agregar un motor de voz | carpeta nueva en `backend/engines/` (copiar `_echo/`) |
| Integrar con agentes | `backend/mcp_server.py` |

## Comandos de desarrollo
```bash
bun install --frozen-lockfile   # dependencias JS
bun run setup:api               # dependencias Python (uv + torch)
bun run dev                     # Electron en modo desarrollo
bun run dev:web                 # renderer en navegador + backend
```
Nota: `bun run dev` corre `bun install` y puede modificar `bun.lock`; no
commitearlo (`git checkout -- bun.lock`).
