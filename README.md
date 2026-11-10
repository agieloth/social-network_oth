# Social Network

A Facebook-like social network: profiles (public or private), followers, posts with fine-grained privacy, groups with posts, events and a group chat, real-time private messages and notifications.

| Layer     | Stack                                                                 |
|-----------|-----------------------------------------------------------------------|
| Backend   | Go 1.24, `net/http`, SQLite (`mattn/go-sqlite3`), `golang-migrate`, Gorilla WebSocket, bcrypt |
| Frontend  | Next.js 15 (App Router), React 19, CSS Modules                        |
| Real time | WebSocket handled in a Web Worker on the client, hub on the server    |
| Deploy    | Docker / docker-compose                                               |

## Features

- **Accounts**: registration with optional avatar, session cookie (HttpOnly, SameSite=Strict), logout closes the user's WebSockets.
- **Profiles**: public or private. A private profile only shows its posts and connections to accepted followers.
- **Followers**: instant follow on public profiles, follow requests (accept / decline) on private ones.
- **Posts**: text + image, visibility `public`, `followers` or `custom` (hand-picked followers), comments.
- **Groups**: creation, invitations, join requests approved by the creator, group posts and comments, events with going / not going answers.
- **Chat**: private messages between users when at least one follows the other, group chat for members.
- **Notifications**: follow requests, group invitations, join requests, new events — stored and pushed in real time.

## Architecture

```
backend/
├── main.go              # wiring, routes, HTTP server, graceful shutdown
├── db/
│   ├── sqlite/          # connection (foreign keys ON, WAL) + migrations runner
│   └── migrations/      # versioned SQL migrations
├── handlers/            # HTTP layer: parse request, call service, map errors to status codes
│   └── group/           # /api/groups/{id}/... router and handlers
├── services/            # business rules and authorization checks
├── repositories/        # SQL only
├── hub/                 # WebSocket hub (thread-safe, multi-tab)
├── media/               # public URL of uploaded files
├── models/
└── utils/               # middlewares (CORS, auth, rate limit), JSON helpers, uploads

frontend/src/
├── app/                 # pages (App Router)
├── contexts/            # auth, worker, chat, navbar state
├── lib/                 # API client, media URL helper
└── config/              # backend URLs
frontend/public/worker.js  # owns the WebSocket connection
```

Requests flow `handler → service → repository`. Authorization (who can see or write what) lives in the services, so the HTTP and WebSocket entry points share the same rules.

## Run locally

Requirements: Go 1.24+ with a C compiler (cgo is needed by SQLite), Node.js 20+.

```bash
# Backend — http://localhost:8080
cd backend
go run .

# Frontend — http://localhost:3000
cd frontend
cp .env.example .env.local
npm install
npm run dev
```

Migrations run automatically at startup. The database is created in `backend/data/` and uploads are stored in `backend/uploads/` (both git-ignored). See `backend/.env.example` for the available settings.

### With Docker

```bash
docker compose up --build
```

## Configuration

| Variable (backend)      | Default                     | Purpose                                        |
|-------------------------|-----------------------------|------------------------------------------------|
| `PORT`                  | `8080`                      | HTTP port                                      |
| `DB_PATH`               | `./data/social.db`          | SQLite file                                    |
| `API_BASE_URL`          | `http://localhost:8080`     | Used to build absolute image URLs              |
| `CORS_ALLOWED_ORIGINS`  | `http://localhost:3000,...` | Allowed frontend origins                       |
| `WS_ALLOWED_ORIGINS`    | `http://localhost:3000,...` | Allowed WebSocket origins                      |
| `COOKIE_SECURE`         | `true`                      | `false` only for plain-HTTP development        |
| `TRUST_PROXY_HEADERS`   | `false`                     | `true` behind a proxy that sets `X-Forwarded-For` |

| Variable (frontend, build time) | Default                   |
|---------------------------------|---------------------------|
| `NEXT_PUBLIC_API_URL`           | `http://localhost:8080`   |
| `NEXT_PUBLIC_WS_URL`            | `ws://localhost:8080/ws`  |

## Quality checks

```bash
cd backend && gofmt -l . && go vet ./... && go build -race ./...
cd frontend && npm run lint && npm run build
```
