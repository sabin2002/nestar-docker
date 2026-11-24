# Nestar — Microservices + AI, Dockerized Local Development

Run the Next.js frontend, the NestJS **microservices** backend, MongoDB and a local **AI assistant** (Ollama) using Docker Compose.

## Project structure

```
nestar-docker/
├── 1. Planning/
│   ├── System Request/         why we build it: need, requirements, value, constraints
│   ├── Feasibility Analysis/   technical, economic, organizational feasibility
│   ├── Staffing Plan/          RACI matrix (Nestar RTM and RACI.xlsx, "RACI Matrix" tab)
│   ├── Project Standards/      documentation, coding, testing and git rules
│   └── Risk Assessment/        risks, owners and mitigations
├── 2. Analysis/                SRS (use cases, cloud architecture, sequence diagrams),
│                               before/after analysis, requirements traceability matrix
├── 3. Design/                  architecture and design-overview diagrams (PNG + SVG)
└── 4. Implementation/          all code
    ├── backend/nestar/         NestJS: API gateway + 5 microservices + batch (own git repo)
    ├── frontend/nestar-next/   Next.js website (own git repo)
    ├── compose.yaml            Docker Compose for the whole stack
    └── .env.example            settings for Docker Compose
```

All commands below are run inside `4. Implementation`.

## Architecture

```
Browser ──▶ frontend (Next.js, :3001)
   │
   └──── GraphQL / uploads / WebSocket ──▶ api-gateway (apps/nestar-api, :3000)
                                             │  TCP (@nestjs/microservices)
      ┌───────────────┬──────────────────┬───┴─────────────┬───────────────┐
 member-service  property-service  community-service  social-service   ai-service ──▶ ollama (LLM)
   :4001            :4002             :4003              :4004            :4005
 nestar_member   nestar_property   nestar_community   nestar_social    (no database)
      └───────────────┴──────────────────┴─────────────────┘
                     mongo (one database per service)

 batch (apps/nestar-batch) — cron scheduler, asks property/member services to recompute rankings
```

| App (`backend/nestar/apps/…`) | Responsibility | Owns |
| --- | --- | --- |
| `nestar-api` | **API gateway**: the only public backend. GraphQL schema for the frontend, JWT verification and role guards, image uploads, WebSocket live chat and AI chat. Forwards every request to a service. | — |
| `member-service` | signup / login (password hashing, JWT signing), member profiles, agents, admin member management | `members` |
| `property-service` | property listings, search filters, agent listings, admin moderation, rank calculation | `properties` |
| `community-service` | community board articles and comments | `boardArticles`, `comments` |
| `social-service` | likes, views (recently visited), follows, favorites | `likes`, `views`, `follows` |
| `ai-service` | **Nestar Assistant** chatbot: fetches relevant listings from property-service and asks the local LLM to answer using only those listings | — |
| `nestar-batch` | scheduled ranking jobs (no database access; calls the services) | — |

Shared code lives in `backend/nestar/libs/common` (`@app/common`): GraphQL DTOs, enums, Mongoose schemas, message-pattern names (`rpc/patterns.ts`), the service registry (`rpc/services.ts`) and RPC helpers.

How a request flows: the frontend calls `getProperties` → the gateway's resolver sends `property.list` over TCP → property-service queries its own database, then asks member-service (`member.byIds`) for the agents and social-service (`like.checkMany`) for the user's likes → the combined result goes back to the frontend. The GraphQL API is unchanged from the monolith, plus one new query, `askAssistant(message)`.

### AI assistant

Open the chat bubble on the website and switch to the **AI Assistant** tab. Questions go privately over the WebSocket (`{ event: 'ai', data: '…' }`) to ai-service, which:

1. picks matching listings from property-service (city, type, beds, "under 500k", rent/barter are detected in the question),
2. puts them in the prompt for the local model (Ollama, default `llama3.2`),
3. returns the answer with clickable links to the listings it recommends.

The model runs on your computer — no API key and no data sent to a cloud AI. Try it in the GraphQL playground (http://localhost:3000/graphql) too:

```graphql
query { askAssistant(message: "Any villa in Busan with 3 beds?") { reply model propertyIds } }
```

> These instructions assume the Dockerfiles and `compose.yaml` included in this repository use the folder layout and ports below. This is a local development setup, not a production deployment.

## Requirements

- Docker Desktop installed and running (Windows/macOS), or Docker Engine with the Compose plugin (Linux).
- On Windows, complete Docker Desktop's WSL 2 setup if prompted.
- Git to clone the repository. Alternatively, download and extract the repository ZIP from GitHub.
- About 8 GB of RAM available to Docker (the AI model alone needs roughly 3–4 GB) and about 5 GB of free disk space.

You do **not** need to install Node.js, npm, Yarn, or MongoDB on your computer. Docker installs application dependencies inside the images.

Check Docker is ready:

```sh
docker --version
docker compose version
```

## Quick start

### 1. Clone the repository

Replace the placeholder URL with this repository's GitHub URL:

```sh
git clone https://github.com/YOUR_USERNAME/nestar-docker.git
cd nestar-docker
```

If you downloaded a ZIP, open a terminal in the extracted folder instead.

Then go to the code folder:

```sh
cd "4. Implementation"
```

### 2. Check configuration

The development Compose configuration supplies the following settings:

| Service | Variable | Local development value |
| --- | --- | --- |
| All backend apps | `NODE_ENV` | `development` |
| All backend apps | `MONGO_DEV` | `mongodb://mongo:27017/nestar` (each service uses its own database name on this server) |
| All backend apps | `SECRET_TOKEN` | from `.env`, default `nestar-dev-secret-change-me` |
| All backend apps | `*_SERVICE_HOST` | Docker service names, e.g. `member-service` |
| api-gateway | `PORT_API` | `3000` |
| ai-service | `OLLAMA_URL` / `OLLAMA_MODEL` | `http://ollama:11434` / `llama3.2` |
| Frontend | `REACT_APP_API_URL` | `http://localhost:3000` |
| Frontend | `REACT_APP_API_GRAPHQL_URL` | `http://localhost:3000/graphql` |
| Frontend | `REACT_APP_API_WS` | `ws://localhost:3000` (the WebSocket shares the gateway's port) |

To change the JWT secret or the AI model, copy `4. Implementation/.env.example` to `.env` in the same folder and edit it.

If the application requires additional secrets or service credentials, supply your own through the configuration mechanism used by this repository. Do not commit real secrets. A `.env` file is not automatically passed into a container unless the Compose configuration references it or its values.

The backend uses `mongo`, the database service name, to connect within Docker. Browser requests use `localhost`, because the browser runs on your computer.

### 3. Build and start

From the `4. Implementation` folder (it contains `compose.yaml`):

```sh
docker compose up --build
```

The first run downloads images, installs dependencies, builds all backend apps and downloads the AI model (about 2 GB, done once by the `ollama-pull` container), so it may take several minutes. Keep the terminal open and wait for the applications to finish starting.

To start in the background instead:

```sh
docker compose up --build -d
```

### 4. Open the application

Open **http://localhost:3001** in your browser.

| Service | Address |
| --- | --- |
| Frontend website | http://localhost:3001 |
| API gateway (HTTP) | http://localhost:3000 |
| GraphQL endpoint + playground | http://localhost:3000/graphql |
| WebSocket (live chat + AI) | ws://localhost:3000 |
| Ollama (AI model server) | http://localhost:11434 |

The microservices (ports 4001–4005) are only reachable inside the Docker network; the browser always goes through the gateway.

## Stop and restart

Stop foreground logs with `Ctrl+C`. To stop and remove the stack's containers:

```sh
docker compose down
```

Start it again:

```sh
docker compose up -d
```

After changing source code, Dockerfiles, or dependencies, rebuild:

```sh
docker compose up --build -d
```

In this setup, source files are copied into the images rather than bind-mounted. Local code changes therefore require a rebuild.

## Logs and status

```sh
docker compose ps
docker compose logs --tail=100 api-gateway
docker compose logs --tail=100 member-service property-service community-service social-service
docker compose logs --tail=100 ai-service ollama ollama-pull
docker compose logs --tail=100 frontend mongo
```

Follow all service logs:

```sh
docker compose logs -f
```

## Database and uploaded files

The Compose configuration uses named volumes for MongoDB data and backend uploads. These persist across ordinary `docker compose down` commands.

A first run creates an **empty database**. This repository does not automatically include the original developer's users, properties, images, or other database records. Follow any repository-specific seed or import instructions if provided.

### Moving data from the old single database

Before the split, everything was stored in one database called `nestar`. Each service now has its own database (`nestar_member`, `nestar_property`, `nestar_community`, `nestar_social`). If you already have data in `nestar`, copy it once:

```sh
docker compose run --rm member-service node scripts/split-database.js
```

The old `nestar` database is only read, never changed. Collections that already contain data are skipped; add `--force` to copy again. Without Docker, run `npm run db:split` in `4. Implementation/backend/nestar`.

### Reset local data — destructive

The following command deletes the Compose stack's named volumes, including database records and uploads. Only run it if you intend to erase that data:

```sh
docker compose down -v
```

## Troubleshooting

### Docker daemon is unavailable

Start Docker Desktop and wait until it is ready, then retry the command.

### A host port is already in use

Stop the conflicting application, or change the host side of the relevant port mapping in `compose.yaml`. For example, to expose the frontend on port 3002 instead of 3001:

```yaml
ports:
  - "127.0.0.1:3002:3001"
```

Then open http://localhost:3002. If you change backend host ports, also update the browser-facing API URLs. Changes to the frontend origin may require corresponding backend CORS settings.

### MongoDB URI is undefined

Confirm that the backend service receives:

```yaml
environment:
  NODE_ENV: development
  MONGO_DEV: mongodb://mongo:27017/nestar
```

The backend code reads `MONGO_DEV` in development, not `MONGO_URI`. Each service then picks its own database name (override with `MONGO_DB_NAME`). After updating Compose, run `docker compose up --build -d` again.

### Backend cannot connect to MongoDB

Inspect `docker compose logs --tail=100 mongo` and the backend logs. Inside the backend container, the database host must be `mongo`, not `localhost`.

### Frontend cannot reach the API

Check that the backend is running, that the published HTTP port is correct, and that the frontend API URLs point to the backend's address on your computer. The backend must allow the frontend origin if CORS is enforced.

The current frontend Dockerfile uses `yarn dev`; its public environment values are read at startup. If switching to a production `next build` image, supply those values during the build.

### Chat or WebSockets fail

The WebSocket server runs inside the api-gateway on the same port as GraphQL (`PORT_API`). `REACT_APP_API_WS` must point to that port: `ws://localhost:3000` in Docker, `ws://localhost:3007` when running locally with the default `backend/nestar/.env`.

### "Service unavailable (…)" errors

The gateway could not reach a microservice. Check `docker compose ps` and the logs of the service named in the pattern (for example `member.login` → `member-service`). Services can start a few seconds after the gateway; retry once they are up.

### AI assistant does not answer

- "AI model … is not installed": the model is still downloading. Watch `docker compose logs -f ollama-pull`, or run `docker compose exec ollama ollama pull llama3.2`.
- "AI assistant is offline": the `ollama` container is not running (`docker compose up -d ollama`).
- Slow answers: without a GPU the model runs on the CPU. Use a smaller model with `OLLAMA_MODEL=llama3.2:1b` in `.env`, or enable the GPU block in `compose.yaml`. You can also install Ollama directly on Windows (https://ollama.com) and set `OLLAMA_URL=http://host.docker.internal:11434`.

### Build fails or additional configuration is missing

Read the first build/startup error and check the original projects' setup instructions. Additional authentication secrets, external service credentials, or runtime-version requirements may be necessary. Never share secret values in public issues or logs.

## Running the backend without Docker

In `4. Implementation/backend/nestar` (needs Node.js 22, a local MongoDB and, for the assistant, Ollama with `ollama pull llama3.2`):

```sh
npm install
cp .env.example .env          # then edit SECRET_TOKEN, MONGO_DEV
npm run start:dev:all         # gateway + 5 microservices with hot reload
npm run start:dev:batch       # optional: ranking scheduler
```

Each app can also be started on its own, e.g. `npm run start:dev:property`. `npm run build:all` builds every app into `dist/apps/`.

## Security

This setup is intended for local development. The example MongoDB service has no authentication and is not published to the host. Application ports are bound to `127.0.0.1` in the suggested Compose configuration.

Do not deploy this configuration publicly without production hardening, authentication, appropriate secrets management, and a production build strategy.

## Credits and licensing

Original projects:

- Backend: https://github.com/sabin2002/nestar
- Frontend: https://github.com/sabin2002/nestar-next

Preserve applicable licenses and copyright notices. Verify permission to redistribute the original code; attribution alone does not grant redistribution rights.
