# Nestar — Dockerized Local Development

Run the Next.js frontend, NestJS backend, and MongoDB using Docker Compose.

> These instructions assume the Dockerfiles and `compose.yaml` included in this repository use the folder layout and ports below. This is a local development setup, not a production deployment.

## Requirements

- Docker Desktop installed and running (Windows/macOS), or Docker Engine with the Compose plugin (Linux).
- On Windows, complete Docker Desktop's WSL 2 setup if prompted.
- Git to clone the repository. Alternatively, download and extract the repository ZIP from GitHub.

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

If you downloaded a ZIP, open a terminal in the extracted folder containing `compose.yaml` instead.

### 2. Check configuration

The development Compose configuration supplies the following settings:

| Service | Variable | Local development value |
| --- | --- | --- |
| Backend | `NODE_ENV` | `development` |
| Backend | `PORT_API` | `3000` |
| Backend | `MONGO_DEV` | `mongodb://mongo:27017/nestar` |
| Frontend | `REACT_APP_API_URL` | `http://localhost:3000` |
| Frontend | `REACT_APP_API_GRAPHQL_URL` | `http://localhost:3000/graphql` |
| Frontend | `REACT_APP_API_WS` | `ws://localhost:3007` |

The WebSocket port `3007` is provisional: verify that it matches the backend gateway configuration before relying on chat functionality.

If the application requires additional secrets or service credentials, supply your own through the configuration mechanism used by this repository. Do not commit real secrets. A `.env` file is not automatically passed into a container unless the Compose configuration references it or its values.

The backend uses `mongo`, the database service name, to connect within Docker. Browser requests use `localhost`, because the browser runs on your computer.

### 3. Build and start

From the folder containing `compose.yaml`:

```sh
docker compose up --build
```

The first run downloads images and installs dependencies, so it may take several minutes. Keep the terminal open and wait for the applications to finish starting.

To start in the background instead:

```sh
docker compose up --build -d
```

### 4. Open the application

Open **http://localhost:3001** in your browser.

| Service | Address |
| --- | --- |
| Frontend website | http://localhost:3001 |
| Backend HTTP API | http://localhost:3000 |
| GraphQL endpoint, if configured at this path | http://localhost:3000/graphql |
| WebSocket, pending gateway port verification | ws://localhost:3007 |

Port `3007` is not the frontend website. The backend root URL may show `Cannot GET /` if no root route is defined; that alone does not mean startup failed.

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
docker compose logs --tail=100 backend
docker compose logs --tail=100 frontend
docker compose logs --tail=100 mongo
```

Follow all service logs:

```sh
docker compose logs -f
```

## Database and uploaded files

The Compose configuration uses named volumes for MongoDB data and backend uploads. These persist across ordinary `docker compose down` commands.

A first run creates an **empty database**. This repository does not automatically include the original developer's users, properties, images, or other database records. Follow any repository-specific seed or import instructions if provided.

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

The backend code reads `MONGO_DEV` in development, not `MONGO_URI`. After updating Compose, run `docker compose up --build -d` again.

### Backend cannot connect to MongoDB

Inspect `docker compose logs --tail=100 mongo` and the backend logs. Inside the backend container, the database host must be `mongo`, not `localhost`.

### Frontend cannot reach the API

Check that the backend is running, that the published HTTP port is correct, and that the frontend API URLs point to the backend's address on your computer. The backend must allow the frontend origin if CORS is enforced.

The current frontend Dockerfile uses `yarn dev`; its public environment values are read at startup. If switching to a production `next build` image, supply those values during the build.

### Chat or WebSockets fail

Verify the backend's actual WebSocket listening port, transport, path, and permitted origins. Match the Compose port mapping and `REACT_APP_API_WS` to that configuration. The suggested port 3007 has not been confirmed.

### Build fails or additional configuration is missing

Read the first build/startup error and check the original projects' setup instructions. Additional authentication secrets, external service credentials, or runtime-version requirements may be necessary. Never share secret values in public issues or logs.

## Security

This setup is intended for local development. The example MongoDB service has no authentication and is not published to the host. Application ports are bound to `127.0.0.1` in the suggested Compose configuration.

Do not deploy this configuration publicly without production hardening, authentication, appropriate secrets management, and a production build strategy.

## Credits and licensing

Original projects:

- Backend: https://github.com/sabin2002/nestar
- Frontend: https://github.com/sabin2002/nestar-next

Preserve applicable licenses and copyright notices. Verify permission to redistribute the original code; attribution alone does not grant redistribution rights.
