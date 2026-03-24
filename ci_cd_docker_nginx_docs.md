# ArchionLabs Deployment, CI/CD, and Testing Architecture

This document provides a comprehensive overview of how Docker, Nginx, GitHub Actions (CI/CD), and tests are configured within the ArchionLabs monorepo architecture. 

ArchionLabs utilizes a microservices-based approach with four distinct backend services (built with Python/FastAPI and Node.js) and multiple frontends (built with Next.js).

---

## 1. Docker Setup

The application employs Docker and Docker Compose to orchestrate its backend services and the API Gateway (Nginx). The configuration is primarily defined in the root [docker-compose.yml](file:///d:/University%20of%20Westminster/L5/5COSC021C.Y%20Software%20Development%20Group%20Project/Repo/ArchionLabs/docker-compose.yml) file.

### Backend Services
There are four containerized backend services:
1. **`archion-build-backend`**: Built from `./archion-build/backend`. Exposes port `8000`. Runs Python (FastAPI).
2. **`archion-sim-backend`**: Built from `./archion-sim/backend`. Exposes port `8000`. Runs Python (FastAPI). Uses volumes for `uploads` and `reports`.
3. **`archion-viewer-backend`**: Built from `./archion-viewer/backend`. Exposes port `8000`. Runs Python (FastAPI). Uses a volume for `uploads`.
4. **`archion-community-backend`**: Built from `./archion-community/backend`. Exposes port `5000`. Runs Node.js. Uses a volume for `models`.

**Key Docker Configurations**:
- **Environment Management**: Each service injects environment variables via `.env` files located in their respective directories (e.g., `./archion-build/backend/.env`).
- **Healthchecks**: Every service includes a healthcheck (e.g., `urllib.request.urlopen` for Python APIs or `http.get` for the Node API) to ensure the service is fully operational before traffic is routed.
- **Restart Policy**: `restart: unless-stopped` ensures that the containers automatically restart on failures or system reboots.

---

## 2. Nginx API Gateway Setup

Nginx is used as an API gateway and a reverse proxy, routing incoming traffic from `api.archionlabs.com` to the correct internal backend container.

### `docker-compose` Configuration
- Nginx runs from the `nginx:alpine` image.
- It exposes ports `80` (HTTP) and `443` (HTTPS).
- It relies on (`depends_on: condition: service_started`) all four backend services.
- A dedicated `certbot` container manages SSL certificates (Let's Encrypt), which are shared with Nginx via Docker volumes (`certbot-etc`, `certbot-var`, `certbot-webroot`).

### `nginx.conf` Configuration
The main Nginx configuration file (`./nginx/nginx.conf`) defines how traffic is handled:
- **Client Body Size**: `client_max_body_size 100m;` allows large 3D model uploads.
- **Timeouts**: Generous request timeouts (`proxy_read_timeout 300s`, etc.) to accommodate long-running AI generations and simulations.
- **HTTP/HTTPS Redirection**: Any traffic arriving on port `80` is redirected to HTTPS (`443`), except for `.well-known/acme-challenge/` which is used by Certbot for SSL domain verification.
- **Reverse Proxy Routing based on Prefixes**:
  - `/build/` -> Routes to `http://archion-build-backend:8000/`
  - `/community/` -> Routes to `http://archion-community-backend:5000/`
  - `/sim/` -> Routes to `http://archion-sim-backend:8000/` (includes specialized configuration for Server-Sent Events (SSE) with `proxy_buffering off;` and `chunked_transfer_encoding off;` to stream simulation data in real-time).
  - `/viewer/` -> Routes to `http://archion-viewer-backend:8000/`

---

## 3. CI/CD GitHub Actions

The repository uses GitHub Actions for Continuous Integration (CI) and Continuous Deployment (CD), located in `.github/workflows/`.

### Continuous Integration (`ci.yml`)
Runs on **Push** or **Pull Request** to `main` and `develop` branches.
This workflow ensures that only healthy code is merged. It consists of multiple parallel jobs:
1. **Python Backend Tests** (`test-archion-build`, `test-archion-viewer`, `test-archion-sim`):
   - Scaffolds a Python 3.11 environment.
   - Installs dependencies from `requirements.txt`.
   - Runs **flake8** for linting (failing the build on syntax errors `E9,F63,F7,F82`, but only warning on things like max line length).
   - Runs unit tests using **pytest** against an SQLite in-memory/file test database.
2. **Node.js Backend Lint** (`lint-archion-community`):
   - Sets up Node.js 20, runs `npm ci`, and executes a syntax check with `node --check server.js`.
3. **Frontend Build Validation** (`build-frontends`):
   - Uses a matrix strategy to test multiple Next.js apps (`archion-build/frontend`, `archion-sim/frontend`, `archion-viewer/frontend`, `archion-community/frontend`).
   - Runs `npm ci`, `npm run lint` (non-blocking), and ensures `npm run build` completes successfully.
4. **Docker Build Validation** (`docker-build`):
   - Tests `docker build` commands for all backend services to ensure no container build failures exist in PRs.

### Continuous Deployment (`deploy-backends.yml`)
Runs only on **Push** to `main` when backend files, `docker-compose.yml`, or `nginx` configurations are changed.
- **Dependency**: It requires the `ci` workflow to pass first (`needs: ci`).
- **Deployment Strategy**: 
  - Connects to the DigitalOcean server via SSH using secrets (`DO_DROPLET_IP`, `DO_SSH_USERNAME`, `DO_SSH_PRIVATE_KEY`).
  - Navigates to `/opt/ArchionLabs`.
  - Pulls the latest code using `git fetch` and `git reset --hard origin/main`.
  - Rebuilds and restarts the containers automatically using `docker compose up -d --build`.
  - Prunes dangling Docker images.
  - Finally, runs a mini health-check script (`curl -sf`) to ensure `build/`, `sim/api/health`, `viewer/`, and `community/templates` endpoints are responding correctly post-deployment.

---

## 4. Tests

Tests in this repository are executed during the CI pipeline and are structured specifically per module framework. 

### Backend Testing (Python/FastAPI)
- Uses **pytest** for execution.
- Tests are typically located in a `tests/` directory within the module's backend folder (e.g., `archion-build/backend/tests/`).
- The tests interact with a temporary mock SQLite database configured during CI (e.g., `DATABASE_URL: "sqlite:///./test_archion.db"`).
- They utilize **httpx** (as seen in the `archion-sim` CI requirements) to test FastAPI endpoints cleanly via `TestClient`.
- For `archion-sim`, `pytest` executes end-to-end simulation or unit function testing based on the `core/` logic.

### Static Code Validation
- **Python**: Enforced strict syntax validation (using `flake8`) to catch fatal errors early.
- **JavaScript (Node)**: Checked via `node --check` to catch parsing/compilation issues.
- **Frontend TS/JS (Next.js)**: Validated comprehensively via `npm run build`, which tests Next.js routing, React strict mode rendering, and TypeScript compilation errors (if enforced by tsconfig). Linter explicitly allows warnings (`npm run lint || true`).
