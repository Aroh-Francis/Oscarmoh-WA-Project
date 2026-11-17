# Local Docker & Multi-Container Deployment Guide

This guide provides end-to-end instructions for building, pushing, testing, and running the **Retail Store Sample Application** locally with Docker and Docker Compose before setting up a CI/CD pipeline.

---

## Table of Contents
1. [Architecture Overview & Prerequisites](#1-architecture-overview--prerequisites)
2. [Step 1: Build Containers with Dockerfiles](#step-1-build-containers-with-dockerfiles)
3. [Step 2: Tag and Push Images to a Container Registry](#step-2-tag-and-push-images-to-a-container-registry)
4. [Step 3: Run Multi-Containers for Individual Services](#step-3-run-multi-containers-for-individual-services)
   - [3.1 Cart Service & DynamoDB Local](#31-cart-service--dynamodb-local)
   - [3.2 Catalog Service & MariaDB](#32-catalog-service--mariadb)
   - [3.3 Checkout Service & Redis](#33-checkout-service--redis)
   - [3.4 Orders Service, PostgreSQL & RabbitMQ](#34-orders-service-postgresql--rabbitmq)
   - [3.5 UI Service](#35-ui-service)
5. [Step 4: Run the Full Application Stack (`src/app`)](#step-4-run-the-full-application-stack-srcapp)
6. [Testing the End-to-End Application Flow](#testing-the-end-to-end-application-flow)
7. [Troubleshooting & Common Pitfalls](#troubleshooting--common-pitfalls)
8. [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)

---

## 1. Architecture Overview & Prerequisites

### Services Breakdown

The application is structured into 5 microservices, each with its own source code, Dockerfile, and Compose file:

| Service | Language / Framework | Supporting Containers | Internal Port | Standalone Host Port | Full Stack Host Port |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **UI** | Java 21 / Spring Boot | *None (connects to APIs)* | `8080` | `8888` | **`8888`** (exposed) |
| **Catalog** | Go / Gin | MariaDB 10.9 (`catalog-db`) | `8080` | `8081` | *Internal only* |
| **Cart** | Java 21 / Spring Boot | DynamoDB Local (`carts-db`) | `8080` | `8082` | *Internal only* |
| **Orders** | Java 21 / Spring Boot | PostgreSQL 16 (`orders-db`), RabbitMQ 3 | `8080` | `8083` | *Internal only* |
| **Checkout** | Node.js 20 / NestJS | Redis 6 (`checkout-redis`) | `8080` | `8085` | *Internal only* |

### Prerequisites
- **Docker Desktop** installed and running on your Mac/Linux/Windows machine.
- **Docker Compose v2.20+** (`docker compose version`). Docker Compose v2 is required because `src/app/docker-compose.yml` uses the top-level `include:` syntax.
- **Terminal** opened at the root directory of this repository:
  ```bash
  cd "/Users/francisaroh/Documents/Udemy/DevOps Project/v 1.3.0/retail-store-sample-app-1.3.0 3"
  ```

> [!NOTE]
> **Do I need a GitHub repository to run this?**
> **No!** You can build, run, and test every container and Compose file **100% locally** without a GitHub repository or remote registry. Docker builds and caches images directly on your local machine. You only need a GitHub repo / registry later when setting up CI/CD pipelines (e.g., GitHub Actions).

---

## Step 1: Build Containers with Dockerfiles

### 1.0 Choose Your Workflow (Local Only vs GHCR vs Docker Hub)

You can choose whichever mode suits your current goal:

#### Option A: 100% Local / Offline (No Registry, No GitHub, No Push Needed)
Use this if you just want to test Dockerfiles and Compose locally:
```bash
export IMAGE_PREFIX="retail-store"
export TAG="local"
```

#### Option B: GitHub Container Registry (GHCR - Recommended for CI/CD)
> **Crucial Rule for GHCR:** GHCR requires your GitHub username/organization to be in **lowercase**!
```bash
export REGISTRY="ghcr.io"
export REGISTRY_USER="<your-github-username-in-lowercase>"
export IMAGE_PREFIX="${REGISTRY}/${REGISTRY_USER}"
export TAG="latest"
```

#### Option C: Docker Hub
```bash
export REGISTRY_USER="<your-dockerhub-username>"
export IMAGE_PREFIX="${REGISTRY_USER}"
export TAG="latest"
```

> **Note for Apple Silicon (M1/M2/M3/M4 Macs):**
> If you plan to deploy these images later to x86/amd64 cloud servers (e.g. AWS EKS, ECS, or EC2), pass `--platform linux/amd64` to the `docker build` command. For purely local testing on your Mac, building native ARM64 without `--platform` is faster.

### 1.1 Build Cart
- **Dockerfile:** `src/cart/Dockerfile`
- **Context:** `src/cart`
```bash
docker build -t ${IMAGE_PREFIX}/retail-store-cart:${TAG} -f src/cart/Dockerfile src/cart
```

### 1.2 Build Catalog
- **Dockerfile:** `src/catalog/Dockerfile`
- **Context:** `src/catalog`
```bash
docker build -t ${IMAGE_PREFIX}/retail-store-catalog:${TAG} -f src/catalog/Dockerfile src/catalog
```

### 1.3 Build Checkout
- **Dockerfile:** `src/checkout/Dockerfile`
- **Context:** `src/checkout`
```bash
docker build -t ${IMAGE_PREFIX}/retail-store-checkout:${TAG} -f src/checkout/Dockerfile src/checkout
```

### 1.4 Build Orders
- **Dockerfile:** `src/orders/Dockerfile`
- **Context:** `src/orders`
```bash
docker build -t ${IMAGE_PREFIX}/retail-store-orders:${TAG} -f src/orders/Dockerfile src/orders
```

### 1.5 Build UI
- **Dockerfile:** `src/ui/Dockerfile`
- **Context:** `src/ui`
```bash
docker build -t ${IMAGE_PREFIX}/retail-store-ui:${TAG} -f src/ui/Dockerfile src/ui
```

```bash
docker run -d -p 8888:8080 --name my-ui retail-store-ui:local
```

### 1.6 One-Liner to Build All 5 Images
You can build all services sequentially with this script loop:

```bash
for service in cart catalog checkout orders ui; do
  echo "===> Building retail-store-${service}..."
  docker build -t ${IMAGE_PREFIX}/retail-store-${service}:${TAG} -f src/${service}/Dockerfile src/${service}
done
```

Verify your built images:
```bash
docker images | grep retail-store
```

---

## Step 2: Tag and Push Images to a Container Registry

### 2.1 Log in to Container Registry

#### If using GitHub Container Registry (GHCR):
1. Create a GitHub Personal Access Token (Classic):
   - In GitHub: **Settings** > **Developer settings** > **Personal access tokens** > **Tokens (classic)**.
   - Click **Generate new token (classic)**.
   - Select scopes: `write:packages`, `read:packages`, and `delete:packages` (optional).
   - Copy the generated token.
2. In your terminal, authenticate Docker with GHCR:
   ```bash
   export CR_PAT="<your-github-personal-access-token>"
   echo $CR_PAT | docker login ghcr.io -u <your-github-username> --password-stdin
   ```
   *(Or if you have the GitHub CLI installed: `gh auth token | docker login ghcr.io -u <your-github-username> --password-stdin`)*



#### Tag Your Local Image

Rename your local image so Docker knows to route it to your GHCR account (all lowercase is mandatory):
```bash
docker tag retail-store-ui:100 ghcr.io/aroh-francis/retail-store-ui:100
```


Push the Image

Upload the tagged image to GHCR:
```bash
docker push ghcr.io/aroh-francis/retail-store-ui:100
```

Once pushed, you can go to your GitHub profile → Packages → click retail-store-ui → Connect repository to link it directly to Oscarmoh-WA-Project and adjust its visibility (Public or Private).


#### If using Docker Hub:
```bash
docker login
```
*(Enter your Docker Hub username and password/Personal Access Token).*

#### (Optional) If using AWS ECR:
```bash
aws ecr get-login-password --region <region> | docker login --username AWS --password-stdin <aws_account_id>.dkr.ecr.<region>.amazonaws.com
```

### 2.2 Optional: Tag with Version Release
It is best practice for CI/CD to tag with both a version (e.g. `1.3.0`) and `latest`:
```bash
for service in cart catalog checkout orders ui; do
  docker tag ${IMAGE_PREFIX}/retail-store-${service}:${TAG} ${IMAGE_PREFIX}/retail-store-${service}:1.3.0
done
```

### 2.3 Push Images to Registry
Push individually:
```bash
docker push ${IMAGE_PREFIX}/retail-store-cart:${TAG}
docker push ${IMAGE_PREFIX}/retail-store-catalog:${TAG}
docker push ${IMAGE_PREFIX}/retail-store-checkout:${TAG}
docker push ${IMAGE_PREFIX}/retail-store-orders:${TAG}
docker push ${IMAGE_PREFIX}/retail-store-ui:${TAG}
```

Or push all with a single loop:
```bash
for service in cart catalog checkout orders ui; do
  echo "===> Pushing retail-store-${service}..."
  docker push ${IMAGE_PREFIX}/retail-store-${service}:${TAG}
done
```

> [!NOTE]
> **GHCR Package Visibility:** By default, new images pushed to GHCR might be private. To make them publicly accessible (so CI/CD or other machines can pull without authenticating), visit:
> `https://github.com/users/<your-username>/packages` > Click the package > **Package settings** > Scroll down to **Danger Zone** > **Change visibility** > Select **Public**.

---

## Step 3: Run Multi-Containers for Individual Services

Each service directory contains its own `Dockerfile` and `docker-compose.yml` that starts the service alongside its dedicated database, in-memory cache, or message broker.

> [!TIP]
> **One-by-One Local Verification Workflow:**
> Testing each service individually is the best way to verify Dockerfiles and databases before running everything together.
> For each service:
> 1. Build the local image with `docker build` (or let Compose build it).
> 2. Start the service with `docker compose up -d`.
> 3. Verify health and APIs with `curl`.
> 4. Stop the service with `docker compose down -v` before starting the next to avoid port collisions.

---

### 3.1 Cart Service & DynamoDB Local
- **Components:** `cart` (Java app) + `carts-db` (Amazon DynamoDB Local on port 8000)
- **Host Port Exposed:** `8082`

**1. (Optional) Build image independently:**
```bash
docker build -t retail-store-cart:local src/cart
```

**2. Start Cart and DynamoDB Local:**
```bash
docker compose -f src/cart/docker-compose.yml up -d --build
```

**3. Check Status & Logs:**
```bash
docker compose -f src/cart/docker-compose.yml ps
docker compose -f src/cart/docker-compose.yml logs -f cart
```

**4. Verify Health Endpoint:**
```bash
curl http://localhost:8082/actuator/health
# Expected JSON output: {"status":"UP",...}
```

**5. Tear Down before testing next service:**
```bash
docker compose -f src/cart/docker-compose.yml down -v
```

---

### 3.2 Catalog Service & MariaDB
- **Components:** `catalog` (Go app) + `catalog-db` (MariaDB 10.9 on port 3306)
- **Host Port Exposed:** `8081`
- **Required Variable:** `DB_PASSWORD` is needed for MariaDB and Catalog authentication.

**1. (Optional) Build image independently:**
```bash
docker build -t retail-store-catalog:local src/catalog
```

**2. Start Catalog and MariaDB:**
```bash
DB_PASSWORD=testing docker compose -f src/catalog/docker-compose.yml up -d --build
```

**3. Check Status & Logs:**
```bash
docker compose -f src/catalog/docker-compose.yml ps
docker compose -f src/catalog/docker-compose.yml logs -f catalog
```

**4. Verify Health & Query Products:**
```bash
# Health check:
curl http://localhost:8081/health
# Expected: {"status":"ok"}

# Query products:
curl http://localhost:8081/products
```

**5. Tear Down before testing next service:**
```bash
docker compose -f src/catalog/docker-compose.yml down -v
```

---

### 3.3 Checkout Service & Redis
- **Components:** `checkout` (Node.js NestJS app) + `checkout-redis` (Redis 6.0 on port 6379)
- **Host Port Exposed:** `8085`

**1. (Optional) Build image independently:**
```bash
docker build -t retail-store-checkout:local src/checkout
```

**2. Start Checkout and Redis:**
```bash
docker compose -f src/checkout/docker-compose.yml up -d --build
```

**3. Check Status & Logs:**
```bash
docker compose -f src/checkout/docker-compose.yml ps
docker compose -f src/checkout/docker-compose.yml logs -f checkout
```

**4. Verify Health Endpoint:**
```bash
curl http://localhost:8085/health
# Expected: {"status":"ok"}
```

**5. Tear Down before testing next service:**
```bash
docker compose -f src/checkout/docker-compose.yml down -v
```

---

### 3.4 Orders Service, PostgreSQL & RabbitMQ
- **Components:** `orders` (Java app) + `orders-db` (Postgres 16.1 on port 5432) + `rabbitmq` (RabbitMQ on ports 61616, 8161)
- **Host Port Exposed:** `8083`
- **Required Variable:** `DB_PASSWORD` for PostgreSQL and RabbitMQ default credentials.

**1. (Optional) Build image independently:**
```bash
docker build -t retail-store-orders:local src/orders
```

**2. Start Orders, PostgreSQL, and RabbitMQ:**
```bash
DB_PASSWORD=testing docker compose -f src/orders/docker-compose.yml up -d --build
```

**3. Check Status & Logs:**
```bash
docker compose -f src/orders/docker-compose.yml ps
docker compose -f src/orders/docker-compose.yml logs -f orders
```

**4. Verify Health Endpoint:**
```bash
curl http://localhost:8083/actuator/health
# Expected: {"status":"UP",...}
```

**5. Tear Down before testing next service:**
```bash
docker compose -f src/orders/docker-compose.yml down -v
```

---

### 3.5 UI Service
- **Components:** `ui` (Java Spring Boot storefront)
- **Host Port Exposed:** `8888`

**1. (Optional) Build image independently:**
```bash
docker build -t retail-store-ui:local src/ui
```

**2. Start UI:**
```bash
docker compose -f src/ui/docker-compose.yml up -d --build
```

**3. Verify Health Endpoint & Browser:**
```bash
curl http://localhost:8888/actuator/health
```
*(You can also visit `http://localhost:8888` in your browser)*

**4. Tear Down before running full application:**
```bash
docker compose -f src/ui/docker-compose.yml down -v
```

---

## Step 4: Run the Full Application Stack (`src/app`)

In production or local full-stack development, you run all 5 microservices and their supporting infrastructure together.

### How `src/app` Works:
1. `src/app/docker-compose.yml` includes the compose files of UI, Catalog, Cart, Checkout, and Orders using Compose `include:`.
2. `src/app/compose.override.yaml` connects the microservices together:
   - Configures the UI environment variables with the internal container DNS addresses:
     - `RETAIL_UI_ENDPOINTS_CATALOG=http://catalog:8080`
     - `RETAIL_UI_ENDPOINTS_CARTS=http://carts:8080`
     - `RETAIL_UI_ENDPOINTS_ORDERS=http://orders:8080`
     - `RETAIL_UI_ENDPOINTS_CHECKOUT=http://checkout:8080`
   - Configures Checkout to reach Orders: `RETAIL_CHECKOUT_ENDPOINTS_ORDERS=http://orders:8080`.
   - Overrides port mappings for internal databases and services so **only UI port `8888`** is exposed to your host machine.
   - Enforces service health check dependencies (`service_healthy`) so UI boots only when backend microservices are ready.

---

### 4.1 Launch Option A: Direct Docker Compose Command (Recommended)

From the project root:

```bash
DB_PASSWORD=testing docker compose \
  -f src/app/docker-compose.yml \
  -f src/app/compose.override.yaml \
  up --build
```

> **Tip:** Add `-d` to run in the background (detached mode):
> ```bash
> DB_PASSWORD=testing docker compose \
>   -f src/app/docker-compose.yml \
>   -f src/app/compose.override.yaml \
>   up -d --build
> ```

---

### 4.2 Launch Option B: Running from the `src/app` Directory

You can also `cd` directly into `src/app`. Docker Compose automatically discovers `compose.override.yaml` in the current working directory:

```bash
cd src/app
DB_PASSWORD=testing docker compose up -d --build
```
*(To return to the project root: `cd ../..`)*

---

### 4.3 Launch Option C: Using Project NPM / Yarn Scripts

The root `package.json` includes convenience scripts configured for `src/app`:

```bash
# 1. Export the database password in your terminal session
export DB_PASSWORD=testing

# 2. Run the compose script
npm run compose:up
# Or if using yarn:
# yarn compose:up
```

---

### 4.4 Verify Running Full-Stack Containers

Check running containers:
```bash
docker compose -f src/app/docker-compose.yml ps
```

You should see 9 healthy running containers:
- `app-ui-1`
- `app-catalog-1`
- `app-catalog-db-1`
- `app-cart-1`
- `app-carts-db-1`
- `app-checkout-1`
- `app-checkout-redis-1`
- `app-orders-1`
- `app-orders-db-1`
- `app-rabbitmq-1`

---

## Testing the End-to-End Application Flow

1. **Access the Storefront UI:**
   Open your browser and navigate to:
   [http://localhost:8888](http://localhost:8888)

2. **Catalog Verification:**
   The homepage should display sample products loaded from MariaDB through the Catalog service.

3. **Cart Verification:**
   Click on any item and select **"Add to Cart"**. Navigate to the shopping cart to verify Cart service persistence with DynamoDB.

4. **Checkout & Orders Verification:**
   Proceed to checkout, enter mock shipping details, and submit the order. Checkout service communicates with Redis and the Orders service (which stores order details into PostgreSQL and publishes events to RabbitMQ).

5. **Tail Logs in Real Time:**
   To follow logs for all containers:
   ```bash
   docker compose -f src/app/docker-compose.yml logs -f
   ```
   Or tail a specific service (e.g. `ui`, `orders`, `catalog`):
   ```bash
   docker compose -f src/app/docker-compose.yml logs -f ui
   ```

---

## Graceful Teardown of Full Stack

To stop all containers and remove networks:
```bash
docker compose \
  -f src/app/docker-compose.yml \
  -f src/app/compose.override.yaml \
  down
```

To also remove named volumes (clearing local database test data):
```bash
docker compose \
  -f src/app/docker-compose.yml \
  -f src/app/compose.override.yaml \
  down -v
```

If you used the `package.json` scripts:
```bash
npm run compose:down
# or: yarn compose:down
```

---

## Troubleshooting & Common Pitfalls

### 1. `variable "DB_PASSWORD" is not set`
- **Symptom:** Warnings or database authentication failures in Catalog or Orders.
- **Fix:** Always prefix commands with `DB_PASSWORD=testing` or run `export DB_PASSWORD=testing` in your shell session.

### 2. `address already in use` (Port Conflicts)
- **Symptom:** Error starting container because port `8888`, `8080`, `3306`, `5432`, or `6379` is already bound.
- **Fix:**
  - Make sure previous standalone service containers are stopped (`docker stop $(docker ps -q)`).
  - Check which process is holding the port (e.g. on macOS: `lsof -i :8888` or `lsof -i :5432`).

### 3. Docker Compose `unknown flag: --project-directory` or `unknown attribute: include`
- **Symptom:** Docker Compose reports syntax errors with `include:`.
- **Fix:** Ensure you are using Docker Compose v2 (`docker compose` without a hyphen, not legacy `docker-compose`). Update Docker Desktop to the latest version.

### 4. Apple Silicon Architecture Mismatch (ARM64 vs AMD64)
- When building on an M1/M2/M3/M4 Mac for remote environments (like AWS or CI/CD runners), add `--platform linux/amd64`:
  ```bash
  docker build --platform linux/amd64 -t ${IMAGE_PREFIX}/retail-store-cart:latest -f src/cart/Dockerfile src/cart
  ```

---

## Quick Reference Cheat Sheet

```bash
# ---------------------------------------------
# 1. Environment Setup (Choose GHCR or Docker Hub)
# ---------------------------------------------
# For GHCR (GitHub Container Registry):
export IMAGE_PREFIX="ghcr.io/<your-github-username-in-lowercase>"

# OR For Docker Hub:
# export IMAGE_PREFIX="<your-dockerhub-username>"

export TAG="latest"
export DB_PASSWORD="testing"

# ---------------------------------------------
# 2. Build All Images from Dockerfiles
# ---------------------------------------------
docker build -t ${IMAGE_PREFIX}/retail-store-cart:${TAG} -f src/cart/Dockerfile src/cart
docker build -t ${IMAGE_PREFIX}/retail-store-catalog:${TAG} -f src/catalog/Dockerfile src/catalog
docker build -t ${IMAGE_PREFIX}/retail-store-checkout:${TAG} -f src/checkout/Dockerfile src/checkout
docker build -t ${IMAGE_PREFIX}/retail-store-orders:${TAG} -f src/orders/Dockerfile src/orders
docker build -t ${IMAGE_PREFIX}/retail-store-ui:${TAG} -f src/ui/Dockerfile src/ui

# ---------------------------------------------
# 3. Log In & Push All Images to Registry
# ---------------------------------------------
# For GHCR: echo $CR_PAT | docker login ghcr.io -u <username> --password-stdin
# For Docker Hub: docker login
for s in cart catalog checkout orders ui; do docker push ${IMAGE_PREFIX}/retail-store-$s:${TAG}; done

# ---------------------------------------------
# 4. Run Single Service (Example: Catalog)
# ---------------------------------------------
DB_PASSWORD=testing docker compose -f src/catalog/docker-compose.yml up -d --build
curl http://localhost:8081/health
docker compose -f src/catalog/docker-compose.yml down -v

# ---------------------------------------------
# 5. Run Full Application (src/app)
# ---------------------------------------------
DB_PASSWORD=testing docker compose -f src/app/docker-compose.yml -f src/app/compose.override.yaml up -d --build

# Open Storefront: http://localhost:8888

# Check Status & Logs
docker compose -f src/app/docker-compose.yml ps
docker compose -f src/app/docker-compose.yml logs -f ui

# Teardown Full App
docker compose -f src/app/docker-compose.yml -f src/app/compose.override.yaml down -v
```
