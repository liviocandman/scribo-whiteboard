# Scribo Whiteboard

Scribo is a real-time collaborative whiteboard split into a React/Vite client and an Express/Socket.IO server backed by Redis. It supports room discovery, public and private rooms, live collaborative drawing, grouped undo/redo, shape recognition, bucket fill, export, and multi-user presence.

Verification status for this workspace on March 22, 2026:

- `client`: production build passed, 62 tests passed
- `server`: production build passed, 38 tests passed

## Overview

This repository contains two runnable applications:

- `client/`: the browser UI for room browsing and whiteboard interaction
- `server/`: the HTTP and Socket.IO backend for room management, live sync, and Redis persistence

The user flow is split intentionally:

- REST is used for room browsing, room creation, stats, lookup, and password validation
- Socket.IO is used for live drawing, cursor presence, board state sync, room joins, and undo/redo propagation

## Key Features

- Real-time collaborative drawing with Socket.IO and Redis Pub/Sub fan-out
- Room browser with create, join, delete, search, and stats
- Public and private rooms with password hashing via `bcrypt`
- Auto-created rooms when a user opens a whiteboard URL for a room that does not yet exist
- Drawing tools: pen, magic pen, eraser, bucket fill, and hand-pan
- Grouped undo/redo using `strokeId` so a full gesture is reverted together
- Export current board as PNG
- Presence sync for connected users, cursors, and drawing state
- Touch-friendly zoom and pan interactions
- Canvas performance optimizations including stroke batching, viewport culling, offscreen baking, and a Web Worker flood-fill path

## Architecture

### High-Level Flow

1. The client loads `/` to browse rooms through the HTTP API.
2. The user creates or joins a room from the homepage, then navigates to `/whiteboard?roomId=<id>`.
3. The client collects user identity locally, emits `userSetup`, then joins the room over Socket.IO.
4. The server validates or auto-creates the room, adds the user to Redis-backed room state, and emits `initialState` plus `roomUsers`.
5. Drawing events are persisted to Redis lists, broadcast through Redis Pub/Sub, and replayed to connected clients.

### Backend Layers

- `routes/`: HTTP route registration and Socket.IO route registration
- `controllers/`: request and event handlers
- `services/`: room, drawing, stroke, state, user, and Redis logic
- `models/`: room, stroke, and user validation/serialization helpers
- `config/`: CORS, Redis, Socket.IO, and Pub/Sub setup

### Persistence Model

Redis is used for both storage and fan-out:

- `rooms:all`: set of room ids
- `room:{id}:data`: serialized room metadata
- `room:{id}:users`: active user ids
- `room:{id}:strokes`: list of stroke batches
- `room:{id}:state`: serialized snapshot image
- `room:{id}`: Pub/Sub channel for room events
- `user:{id}:data`: serialized transient user presence

### Client Rendering Pipeline

- Pointer isolation prevents multi-touch interference
- Freehand segments are emitted in batches every ~16 ms
- Bucket fill runs through a Web Worker using transferable buffers
- A world raster is maintained for flood-fill boundaries and redraw recovery
- Viewport culling skips strokes outside the visible area
- Offscreen baking redraws the board into a buffer for smoother zoom/pan

## Tech Stack

| Layer | Technology |
| --- | --- |
| Client | React 19, TypeScript, Vite 7, React Router 7, Socket.IO Client |
| Styling | Plain CSS modules/files in `src/styles` and component CSS files |
| Server | Node.js, Express 5, Socket.IO 4, TypeScript, SWC |
| Persistence | Redis 5, compatible with local Redis or Upstash |
| Security | `bcrypt` for room password hashing, request validation middleware, configurable CORS |
| Testing | Vitest on both client and server |

## Getting Started

### Prerequisites

- Node.js 20+
- npm 9+
- A Redis instance

### Install Dependencies

```bash
cd server
npm install

cd ../client
npm install
```

### Configure Environment

Copy the example files and adjust values:

```bash
cd server
copy .env.example .env

cd ../client
copy .env.example .env
```

Recommended local setup:

- Set `server/.env` `PORT=4000`
- Set `server/.env` `REDIS_URL=...`
- Set `client/.env` `VITE_BACKEND_URL=http://localhost:4000`

Important: the server code falls back to port `3001` if `PORT` is missing, while the client falls back to `http://localhost:4000`. Keep those values aligned.

### Run in Development

Terminal 1:

```bash
cd server
npm run dev
```

Terminal 2:

```bash
cd client
npm run dev
```

Then open `http://localhost:5173`.

### Build and Test

```bash
cd client
npm test -- --run
npm run build

cd ../server
npm test -- --run
npm run build
```

## Product Walkthrough

1. Open the homepage at `/` to browse available rooms.
2. Create a room with a name, description, tags, privacy mode, user limit, and room settings.
3. Join an existing room from the list or by entering a room code.
4. On first entry to the whiteboard, choose a user name and color.
5. Use the left sidebar to switch tools, color, line width, undo, redo, or clear the board.
6. Use the top bar to confirm connection status, see other participants, and export the board.
7. Use mouse wheel or touch gestures to zoom, and the hand tool to pan.

Implementation note: if a user opens `/whiteboard?roomId=<id>` directly and the room does not already exist, the server auto-creates a public room with default settings during the Socket.IO join flow.

## API Documentation

### REST Endpoints

| Method | Path | Purpose |
| --- | --- | --- |
| `GET` | `/health` | Health check |
| `GET` | `/api/rooms` | List rooms, optionally filtered |
| `GET` | `/api/rooms/stats` | Room aggregate stats |
| `GET` | `/api/rooms/:roomId` | Get one room |
| `GET` | `/api/rooms/:roomId/exists` | Check existence |
| `POST` | `/api/rooms` | Create a room |
| `POST` | `/api/rooms/:roomId/join` | Validate room access and password |
| `DELETE` | `/api/rooms/:roomId` | Delete a room |

### Create Room Payload

```json
{
  "name": "Design Review",
  "description": "Sketches and comments",
  "isPublic": true,
  "password": "",
  "maxUsers": 10,
  "tags": ["design", "review"],
  "settings": {
    "allowDrawing": true,
    "allowChat": true,
    "allowExport": true,
    "requireApproval": false,
    "backgroundColor": "#ffffff",
    "canvasSize": "large",
    "enableGrid": false,
    "enableRulers": false,
    "autoSave": true,
    "historyLimit": 20
  }
}
```

Validation enforced by the server:

- room name: 3 to 50 chars
- description: up to 200 chars
- private rooms require a password of at least 4 chars
- `maxUsers`: 1 to 50
- tags: up to 5
- `historyLimit`: 5 to 100
- custom canvas sizes must be at least `400x300`

### Socket.IO Events

Client to server:

| Event | Payload | Notes |
| --- | --- | --- |
| `userSetup` | `{ userId, name, color }` | Required before room join |
| `joinRoom` | `roomId` | Joins or auto-creates room |
| `leaveRoom` | none | Leaves current room |
| `drawing` | `Stroke` | Single stroke, also used for bucket fill and magic pen result |
| `drawing_batch` | `Stroke[]` | Batched freehand segments |
| `saveState` | `snapshot` | Debounced canvas image snapshot |
| `resetBoard` | none | Clears room board |
| `undoStroke` | ack callback | Undo latest local gesture |
| `redoStroke` | ack callback | Redo latest undone gesture |
| `cursorMove` | `{ x, y }` | Presence sync |
| `drawingState` | `boolean` | Presence sync |

Server to client:

| Event | Payload | Notes |
| --- | --- | --- |
| `initialState` | `{ snapshot, strokes }` | Initial replay after join |
| `drawing` | `Stroke` | Single-stroke broadcast |
| `drawing_batch` | `Stroke[]` | Batch broadcast from Redis Pub/Sub |
| `strokeRemoved` | `{ strokeId, userId }` | Targeted undo propagation |
| `strokeAdded` | `{ strokes, strokeId, userId }` | Targeted redo propagation |
| `clearBoard` | none | Clear broadcast |
| `roomUsers` | `User[]` | Current users in room |
| `userJoined` | `User` | Presence notification |
| `userLeft` | `userId` | Presence notification |
| `cursorMove` | `{ userId, position }` | Presence notification |
| `drawingState` | `{ userId, isDrawing }` | Presence notification |
| `error` | `{ event, message, timestamp }` | Error channel |

## Project Structure

```text
whiteboard/
|-- client/
|   |-- public/
|   |-- src/
|   |   |-- components/
|   |   |-- hooks/
|   |   |-- pages/
|   |   |-- services/
|   |   |-- styles/
|   |   |-- types/
|   |   |-- utils/
|   |   `-- workers/
|   `-- README.md
|-- server/
|   |-- src/
|   |   |-- api/
|   |   |-- config/
|   |   |-- controllers/
|   |   |-- managers/
|   |   |-- middleware/
|   |   |-- models/
|   |   |-- routes/
|   |   |-- services/
|   |   |-- socket/
|   |   |-- types/
|   |   `-- utils/
|   `-- README.md
`-- render.yaml
```

## Performance Targets

- Keep freehand collaboration responsive by batching outbound segments every `16 ms`
- Keep zoom/pan bounded between `0.25x` and `4x`
- Avoid full-history work on every frame with grouped strokes and bounding-box culling
- Preserve UI responsiveness during fill operations through a Web Worker implementation
- Reuse pre-rendered board state through offscreen baking for pan/zoom redraws
- Persist board snapshots with a `1000 ms` debounce instead of every stroke

Verified build baseline from this repository state:

- client CSS bundle: `45.80 kB`
- client app bundle: `299.44 kB`
- client socket chunk: `41.33 kB`
- server SWC compile: `46` files

## Deployment

The repository already contains deployment-oriented config:

- `client/netlify.toml`: static hosting configuration with SPA redirects and cache/security headers
- `render.yaml`: Render blueprint for the Node.js backend

Recommended production split:

1. Deploy `client/` to Netlify or another static host.
2. Deploy `server/` to Render or another Node host that supports WebSockets.
3. Provide a Redis endpoint through Upstash or a managed Redis provider.
4. Set `client` `VITE_BACKEND_URL` to the backend public URL.
5. Set `server` `CORS_ORIGINS` to the client origin.

Render blueprint values already model the expected backend environment, including:

- `PORT`
- `REDIS_URL`
- `CORS_ORIGINS`
- `NODE_ENV`
- room, user, and logging-related operational variables

## Contributing

1. Install dependencies in both `client` and `server`.
2. Keep client and server event contracts aligned whenever socket payloads change.
3. Run tests and production builds before submitting changes.
4. Update the relevant README when routes, events, env vars, or deployment settings change.
5. Prefer focused PRs that change either client UI behavior, backend behavior, or shared contracts in one logical unit.

## License

This repository does not currently include a top-level `LICENSE` file. The server package metadata declares `ISC`, but distribution terms are not fully documented until a repository-level license file is added.
