# Client README

The `client` directory contains the React single-page application for Scribo. It provides the room browser, room creation and join flows, user setup, whiteboard UI, export actions, and the full local canvas rendering pipeline.

Verification status for this workspace on March 22, 2026:

- `npm test -- --run`: 62 tests passed
- `npm run build`: passed

## Overview

The client is a Vite-based React application with two primary routes:

- `/`: room discovery, room creation, and room join flow
- `/whiteboard?roomId=<id>`: collaborative canvas experience

It consumes the backend in two ways:

- REST for room metadata and access workflows
- Socket.IO for live whiteboard synchronization and presence

## Key Features

- Room browser with periodic refresh
- Create-room and join-room modals
- User identity setup with persistent local user id
- Pen, magic pen, eraser, bucket fill, and hand-pan tools
- Optimistic undo/redo grouped by gesture
- PNG export from the current canvas
- Live presence indicators for connected users
- Mouse wheel zoom plus touch pinch/pan gestures
- Offscreen-canvas baking, viewport culling, and worker-based flood fill

## Architecture

### Route and Page Layer

- `App.tsx`: route registration
- `pages/HomePage.tsx`: room browser experience
- `pages/WhiteboardPage.tsx`: room loading plus user setup gate

### State and Interaction Hooks

- `hooks/useRooms.ts`: room list, stats, create/join/delete operations
- `hooks/useUsers.ts`: user setup, presence, remote cursors, drawing state
- `hooks/useWhiteboard.ts`: drawing lifecycle, socket sync, undo/redo, zoom, pan, resize, and event listeners

### Service Layer

- `services/roomService.ts`: fetch wrapper for REST endpoints
- `services/socket.ts`: Socket.IO client bootstrap
- `services/canvasService.ts`: canvas transforms, redraw, world raster, offscreen baking, and bucket fill integration
- `services/drawingService.ts`: local drawing operations and cursor behavior
- `services/fillService.ts`: worker-backed flood fill orchestration

### Utility Layer

- `utils/shapeDetection.ts`: heuristic shape recognition for magic pen
- `utils/shapeGeneration.ts`: circle, square, rectangle, and triangle reconstruction
- `utils/throttle.ts`: throttle, debounce, and rAF throttle helpers
- `workers/fill.worker.ts`: background flood-fill worker

### Room Settings Behavior

The room schema contains several configurable settings. In the current client implementation, the most visible enforced settings are:

- `allowDrawing`
- `allowExport`
- `backgroundColor`
- `enableGrid`

Other settings such as `allowChat`, `requireApproval`, `enableRulers`, `autoSave`, and `historyLimit` are stored in the room model and exposed to the client, but they are not fully represented as distinct UI features yet.

## Tech Stack

| Area | Technology |
| --- | --- |
| Framework | React 19 |
| Language | TypeScript |
| Build Tool | Vite 7 |
| Routing | React Router 7 |
| Realtime | Socket.IO Client 4 |
| Testing | Vitest, Testing Library, JSDOM |
| Styling | CSS files under `src/styles` and component-level CSS |

## Getting Started

### Prerequisites

- Node.js 20+
- npm 9+
- A running backend server

### Install

```bash
npm install
```

### Environment

Copy the example file:

```bash
copy .env.example .env
```

Runtime-critical variable:

| Variable | Required | Purpose |
| --- | --- | --- |
| `VITE_BACKEND_URL` | Yes in most setups | Base URL for REST and Socket.IO |

Important note: only `VITE_BACKEND_URL` is referenced directly by the current source code. The example file includes additional placeholders that are not currently consumed at runtime.

Recommended local value:

```env
VITE_BACKEND_URL=http://localhost:4000
```

### Development

```bash
npm run dev
```

The dev server runs on `http://localhost:5173`.

### Production Build and Test

```bash
npm test -- --run
npm run build
```

## Product Walkthrough

1. The homepage loads rooms and stats from the server.
2. Users can search rooms, create a new room, or join by room code.
3. After entering the whiteboard, the app prompts for name and color.
4. The app opens a Socket.IO connection, emits `userSetup`, and joins the requested room.
5. Drawing interactions are applied immediately to the local canvas and synchronized to the room.
6. Undo and redo are optimistic locally, then confirmed in the background through the socket API.
7. Export saves the current canvas to a PNG file in the browser.

## API Documentation

### REST Endpoints Used by the Client

| Method | Path | Used In |
| --- | --- | --- |
| `GET` | `/api/rooms` | Room browser |
| `GET` | `/api/rooms/stats` | Homepage stats |
| `GET` | `/api/rooms/:roomId` | Whiteboard room settings fetch |
| `GET` | `/api/rooms/:roomId/exists` | Room existence checks |
| `POST` | `/api/rooms` | Create room modal |
| `POST` | `/api/rooms/:roomId/join` | Join-room validation |
| `DELETE` | `/api/rooms/:roomId` | Room deletion from browser |

### Socket Events Emitted by the Client

| Event | Payload |
| --- | --- |
| `userSetup` | `{ userId, name, color }` |
| `joinRoom` | `roomId` |
| `leaveRoom` | none |
| `drawing` | `Stroke` |
| `drawing_batch` | `Stroke[]` |
| `saveState` | `snapshot` |
| `resetBoard` | none |
| `undoStroke` | ack callback |
| `redoStroke` | ack callback |
| `cursorMove` | `{ x, y }` |
| `drawingState` | `boolean` |

### Socket Events Consumed by the Client

| Event | Purpose |
| --- | --- |
| `initialState` | Restore room state on join |
| `drawing` | Apply remote single strokes |
| `drawing_batch` | Apply remote batched segments |
| `strokeRemoved` | Reflect undo operations |
| `strokeAdded` | Reflect redo operations |
| `clearBoard` | Reset local board |
| `roomUsers` | Load room participants |
| `userJoined` | Presence update |
| `userLeft` | Presence update |
| `cursorMove` | Presence update |
| `drawingState` | Presence update |
| `error` | Surface socket errors |

## Project Structure

```text
client/
|-- public/
|-- src/
|   |-- components/
|   |   |-- rooms/
|   |   |-- ui/
|   |   `-- whiteboard/
|   |-- hooks/
|   |-- pages/
|   |-- services/
|   |-- styles/
|   |-- types/
|   |-- utils/
|   `-- workers/
|-- index.html
|-- package.json
`-- vite.config.ts
```

## Performance Targets

- Batch outbound drawing segments every `16 ms`
- Avoid redrawing invisible strokes through bounding-box culling
- Reuse an offscreen baked canvas for pan and zoom redraws
- Keep fill operations off the main thread through `fill.worker.ts`
- Debounce snapshot persistence to `1000 ms`
- Keep zoom bounded between `0.25x` and `4x`

Verified build baseline:

- CSS bundle: `45.80 kB`
- vendor chunk: `11.87 kB`
- socket chunk: `41.33 kB`
- main app chunk: `299.44 kB`

## Deployment

The client is configured for static deployment.

- Build command: `npm ci && npm run build`
- Publish directory: `dist`
- SPA fallback is defined in `netlify.toml`
- Cache headers and basic security headers are already included in that config

Typical production setup:

1. Set `VITE_BACKEND_URL` to the backend public URL.
2. Run `npm run build`.
3. Deploy `dist/` to Netlify or another static host.

The helper in `src/utils/config.ts` also supports a same-host fallback in production by deriving the backend host from the browser location when appropriate.

## Contributing

1. Keep UI behavior, socket payloads, and room settings aligned with the server contract.
2. Add or update Vitest coverage when touching utilities or rendering logic.
3. Verify `npm test -- --run` and `npm run build` before merging.
4. Update this README when adding new env vars, routes, or whiteboard tools.

## License

No repository-level `LICENSE` file is currently present. Refer to the root README for the current licensing note before redistributing the client.
