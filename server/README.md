# Server README

The `server` directory contains the Express and Socket.IO backend for Scribo. It manages room metadata, room lifecycle, live whiteboard synchronization, Redis persistence, user presence, and health/status endpoints.

Verification status for this workspace on March 22, 2026:

- `npm test -- --run`: 38 tests passed
- `npm run build`: passed

## Overview

The server exposes:

- an HTTP API for room discovery and room management
- a Socket.IO API for realtime whiteboard collaboration
- Redis-backed storage for room metadata, active users, snapshots, and stroke history

The entrypoint is `src/index.ts`, which loads environment variables, creates `WhiteboardServer`, connects Redis, registers routes, and starts the HTTP plus Socket.IO server.

## Key Features

- Express 5 API for rooms and health checks
- Socket.IO room join, leave, drawing, undo, redo, clear, and presence events
- Redis Pub/Sub fan-out for realtime room synchronization
- Redis-backed storage for strokes, snapshots, and room/user metadata
- Auto-create room behavior on socket join when a room does not exist
- Password-protected rooms using `bcrypt`
- Targeted undo/redo events through `strokeRemoved` and `strokeAdded`
- Validation middleware for room creation and join flows
- Configurable CORS and health endpoint for deployment environments

## Architecture

### Startup Sequence

1. `src/index.ts` loads `.env` and creates `WhiteboardServer`.
2. `WhiteboardServer.initialize()` connects Redis.
3. HTTP routes are mounted through `createApp()`.
4. Socket.IO routes are registered in `routes/socket/index.ts`.
5. Redis Pub/Sub subscriptions are started through `config/pubsub.config.ts`.
6. Graceful shutdown handlers are attached for process signals and unhandled failures.

### Main Backend Modules

- `app.ts`: Express app, JSON parsing, CORS, request logging, error handling
- `server.ts`: HTTP server, Socket.IO server, Redis bootstrap, graceful shutdown
- `routes/http/*`: REST endpoints
- `routes/socket/*`: socket event wiring
- `controllers/http/*`: room and health controllers
- `controllers/socket/*`: room, user, and drawing socket controllers
- `services/*`: domain logic for rooms, strokes, snapshots, users, drawing, and Redis
- `models/*`: validation, serialization, room-id generation, password helpers

### Redis Data Model

| Key | Description |
| --- | --- |
| `rooms:all` | Set of room ids |
| `room:{id}:data` | Serialized room metadata |
| `room:{id}:users` | Active user ids |
| `room:{id}:state` | Serialized snapshot string |
| `room:{id}:strokes` | Stored stroke batches |
| `room:{id}` | Pub/Sub channel for room broadcast |
| `user:{id}:data` | User presence data |

### Drawing Flow

1. Client emits `drawing` or `drawing_batch`.
2. `DrawingSocketController` validates incoming stroke payloads.
3. `DrawingService` stores the stroke or batch in Redis through `StrokeService`.
4. `StrokeService` publishes the room event through Redis Pub/Sub.
5. `config/pubsub.config.ts` receives the message and broadcasts it to the Socket.IO room.
6. Undo and redo operate on grouped `strokeId` batches instead of isolated line segments.

## Tech Stack

| Area | Technology |
| --- | --- |
| Runtime | Node.js |
| Framework | Express 5 |
| Realtime | Socket.IO 4 |
| Language | TypeScript |
| Build | SWC |
| Persistence | Redis 5 |
| Security | `bcrypt`, request validation, configurable CORS |
| Testing | Vitest |

## Getting Started

### Prerequisites

- Node.js 20+
- npm 9+
- Redis, local or managed

### Install

```bash
npm install
```

### Environment

Copy the example file:

```bash
copy .env.example .env
```

The most important variables are:

| Variable | Required | Purpose |
| --- | --- | --- |
| `PORT` | Recommended | HTTP and Socket.IO port |
| `REDIS_URL` | Yes in most deployments | Managed Redis connection string |
| `REDIS_HOST` / `REDIS_PORT` | Optional fallback | Local Redis connection |
| `CORS_ORIGINS` | Yes for production | Allowed frontend origins |
| `NODE_ENV` | Recommended | Error handling and deployment mode |

Recommended local values:

```env
PORT=4000
REDIS_URL=redis://localhost:6379
CORS_ORIGINS=http://localhost:5173
```

Important: the server code defaults to port `3001` if `PORT` is missing, but the client defaults to backend port `4000`. Set `PORT=4000` locally unless you also change `VITE_BACKEND_URL` on the client.

### Development

```bash
npm run dev
```

### Test and Build

```bash
npm test -- --run
npm run build
```

## Product Walkthrough

1. A client can browse room metadata through the HTTP API.
2. After local user setup in the client, the browser emits `userSetup`.
3. The browser emits `joinRoom` with the target room id.
4. If the room does not exist, `RoomSocketController` auto-creates a public room with default settings.
5. The server adds the user to the room, emits `roomUsers`, and sends `initialState`.
6. As drawing events arrive, the server validates, persists, and broadcasts stroke data.
7. Undo removes all strokes sharing the last local `strokeId`; redo restores the grouped strokes.
8. On disconnect, the user is removed from the room and a `userLeft` event is broadcast.

## API Documentation

### HTTP Endpoints

| Method | Path | Description |
| --- | --- | --- |
| `GET` | `/health` | Returns `{ status, timestamp, service }` |
| `GET` | `/api/rooms` | Returns room list with optional filters |
| `GET` | `/api/rooms/stats` | Returns aggregate room stats |
| `GET` | `/api/rooms/:roomId` | Returns a single room or `404` |
| `GET` | `/api/rooms/:roomId/exists` | Returns `{ exists: boolean }` |
| `POST` | `/api/rooms` | Creates a room |
| `POST` | `/api/rooms/:roomId/join` | Validates access to a room |
| `DELETE` | `/api/rooms/:roomId` | Deletes a room and its Redis keys |

### Room Filter Query Parameters

`GET /api/rooms` supports:

- `search`
- `tags` as comma-separated values
- `isPublicOnly=true|false`
- `sortBy=name|created|updated|users`
- `sortOrder=asc|desc`

### Create Room Request Shape

```json
{
  "name": "Workshop Board",
  "description": "Realtime planning session",
  "isPublic": false,
  "password": "abcd1234",
  "maxUsers": 12,
  "tags": ["workshop", "planning"],
  "settings": {
    "allowDrawing": true,
    "allowChat": false,
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

Validation rules enforced by middleware:

- name length: `3-50`
- description length: `<= 200`
- private rooms require password length `>= 4`
- max users: `1-50`
- tags: up to `5`
- `backgroundColor` must be a hex color
- `canvasSize` must be `small`, `medium`, `large`, or `custom`
- custom sizes must be at least `400x300`
- `historyLimit` must be `5-100`

### Socket.IO Events

Client to server:

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

Server to client:

| Event | Payload |
| --- | --- |
| `initialState` | `{ snapshot, strokes }` |
| `drawing` | `Stroke` |
| `drawing_batch` | `Stroke[]` |
| `strokeRemoved` | `{ strokeId, userId }` |
| `strokeAdded` | `{ strokes, strokeId, userId }` |
| `clearBoard` | none |
| `roomUsers` | `User[]` |
| `userJoined` | `User` |
| `userLeft` | `userId` |
| `cursorMove` | `{ userId, position }` |
| `drawingState` | `{ userId, isDrawing }` |
| `error` | `{ event, message, timestamp }` |

## Project Structure

```text
server/
|-- src/
|   |-- api/
|   |-- config/
|   |-- controllers/
|   |   |-- http/
|   |   `-- socket/
|   |-- managers/
|   |-- middleware/
|   |-- models/
|   |-- routes/
|   |   |-- http/
|   |   `-- socket/
|   |-- services/
|   |-- socket/
|   |-- types/
|   `-- utils/
|-- package.json
|-- tsconfig.json
`-- README.md
```

## Performance Targets

- Broadcast freehand input as batches instead of one event per segment whenever possible
- Keep undo and redo targeted through `strokeRemoved` and `strokeAdded` instead of sending a full board reload
- Maintain Redis-backed replay state for quick `initialState` reconstruction
- Keep Socket.IO alive with:
  - `PING_TIMEOUT = 60000`
  - `PING_INTERVAL = 25000`
- Use TTL-backed storage for rooms, users, and snapshots

Verified backend baseline:

- SWC build compiles `46` files successfully
- Vitest suite passes `38` tests

## Deployment

The repository root includes `render.yaml`, which models the intended backend deployment on Render.

That blueprint already defines:

- `rootDir: server`
- build command: `npm ci --include=dev && npm run build`
- start command: `npm run start`
- health check path: `/health`

Production requirements:

1. Provide a working Redis endpoint through `REDIS_URL` or host/port credentials.
2. Set `CORS_ORIGINS` to the client origin.
3. Ensure the hosting platform supports WebSockets.
4. Run the compiled server with `npm run start`.

## Contributing

1. Keep HTTP and Socket.IO contracts backward compatible when possible.
2. Add tests when changing validation, room lifecycle, or drawing persistence behavior.
3. Run `npm test -- --run` and `npm run build` before submitting changes.
4. Update this README when env vars, routes, or socket payloads change.

## License

`package.json` declares `ISC`, but the repository root does not currently include a top-level `LICENSE` file. Add one before relying on repository-wide redistribution terms.
