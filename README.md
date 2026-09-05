# Self-Hosted Chatwoot Deployment (Docker Compose & Render Blueprint)

A turnkey, production-ready repository to deploy and operate a self-hosted [Chatwoot](https://www.chatwoot.com/) customer engagement platform:
- **Locally** via **Docker Compose** for testing and development with zero server costs.
- **In Production** via **Render Blueprint (`render.yaml`)** with one-click automated orchestration of Web, Worker, Redis, and PostgreSQL services.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Project Structure](#project-structure)
3. [Local Development with Docker Compose](#local-development-with-docker-compose)
   - [Prerequisites](#prerequisites)
   - [Step 1: Configure Environment Variables](#step-1-configure-environment-variables)
   - [Step 2: Initialize Database Schema](#step-2-initialize-database-schema)
   - [Step 3: Start Services](#step-3-start-services)
   - [Step 4: Verify Installation](#step-4-verify-installation)
4. [First-Time Admin Onboarding](#first-time-admin-onboarding)
   - [Method A: Web UI Initial Setup](#method-a-web-ui-initial-setup)
   - [Method B: Rails Console Superadmin](#method-b-rails-console-superadmin)
5. [Creating an API Channel & Getting Access Tokens](#creating-an-api-channel--getting-access-tokens)
   - [Create an API Channel Inbox](#create-an-api-channel-inbox)
   - [Retrieve Your User API Access Token](#retrieve-your-user-api-access-token)
   - [Sample API Test Request](#sample-api-test-request)
6. [Production Deployment on Render](#production-deployment-on-render)
   - [Render Blueprint Components](#render-blueprint-components)
   - [Step 1: Push Code to Git](#step-1-push-code-to-git)
   - [Step 2: Connect Blueprint on Render](#step-2-connect-blueprint-on-render)
   - [Step 3: Post-Deploy Domain Configuration](#step-3-post-deploy-domain-configuration)
   - [Using External Managed Databases (Neon, Supabase)](#using-external-managed-databases-neon-supabase)
7. [Operational Commands & Troubleshooting](#operational-commands--troubleshooting)

---

## Architecture Overview

Chatwoot consists of four core infrastructure components:

```
                  +-----------------------------------+
                  |           User Browser            |
                  |     (https://yourdomain.com)      |
                  +-----------------+-----------------+
                                    |
                                    v
                  +-----------------------------------+
                  |          Web Service              |
                  |  chatwoot/chatwoot:latest         |
                  |  Rails / Puma Server (:3000)      |
                  +---------+---------------+---------+
                            |               |
               Database     |               |  Queue / Cache
              Connections   |               |  WebSockets
                            v               v
            +---------------+----+     +----+---------------+
            |  PostgreSQL 15     |     |  Redis 7           |
            |  (Persistent Data) |     |  (Cache & Jobs)    |
            +---------------+----+     +----+---------------+
                            ^               ^
                            |               |
                            |               |
                  +---------+---------------+---------+
                  |        Background Worker          |
                  |  chatwoot/chatwoot:latest         |
                  |  Sidekiq Queue Processor          |
                  +-----------------------------------+
```

- **`rails` (Web)**: Serves the HTTP frontend, REST APIs, and WebSocket ActionCable connections.
- **`sidekiq` (Worker)**: Handles asynchronous background tasks (emails, webhooks, incoming message dispatching, scheduled notifications).
- **`postgres` (Database)**: PostgreSQL 15 with UTF-8 encoding storing accounts, conversations, messages, and contacts.
- **`redis` (Cache & Broker)**: Redis 7 providing key-value caching, Pub/Sub for WebSockets, and Sidekiq job queues.

---

## Project Structure

```
├── .env.example          # Environment variables template for local execution
├── .gitignore            # Git exclusions (credentials, volumes, temporary files)
├── docker-compose.yaml   # Local orchestration for Rails, Sidekiq, Postgres, Redis
├── render.yaml           # Infrastructure-as-Code blueprint for Render cloud deployment
└── README.md             # Complete setup and operations manual
```

---

## Local Development with Docker Compose

### Prerequisites

- [Docker](https://docs.docker.com/get-docker/) (v24.0+)
- [Docker Compose](https://docs.docker.com/compose/) (v2.20+)

### Step 1: Configure Environment Variables

Copy the provided `.env.example` template into `.env`:

```bash
cp .env.example .env
```

Generate a secure 64-byte `SECRET_KEY_BASE` and update `.env`:

```bash
openssl rand -hex 64
```

Review `.env` and adjust the database password or ports if needed. The defaults work out-of-the-box for local testing.

### Step 2: Initialize Database Schema

Before booting the web app, initialize the PostgreSQL schema and load default seeds using `db:chatwoot_prepare`:

```bash
# 1. Start the PostgreSQL and Redis containers in the background
docker compose up -d postgres redis

# 2. Run the preparation migration (creates tables, extensions, and initial data)
docker compose run --rm rails bundle exec rails db:chatwoot_prepare
```

> **Note:** `db:chatwoot_prepare` handles creating the database, running all ActiveRecord migrations, and seeding essential system records.

### Step 3: Start Services

Once the database is initialized, start the entire stack:

```bash
docker compose up -d
```

### Step 4: Verify Installation

Check the status and logs of the running containers:

```bash
# View running containers
docker compose ps

# Follow logs from Rails and Sidekiq
docker compose logs -f rails sidekiq
```

Open your browser and navigate to:
```
http://localhost:3000
```

---

## First-Time Admin Onboarding

### Method A: Web UI Initial Setup

1. Open `http://localhost:3000` in your web browser.
2. If this is a fresh database, Chatwoot will redirect you to the initial onboarding screen (`/app/login` or `/app/setup`).
3. Fill in:
   - **Company Name**: Your company or project name.
   - **Full Name**: Your administrator name.
   - **Work Email**: Your administrator email address.
   - **Password**: A strong password (minimum 8 characters).
4. Click **Create Account**. You will be logged in to the Chatwoot dashboard as an Account Administrator.

### Method B: Rails Console Superadmin

To manage platform-level configurations and multi-tenant accounts via the Super Admin portal (`/super_admin`):

1. Open a Rails console in the running container:
   ```bash
   docker compose run --rm rails bundle exec rails console
   ```

2. Create the SuperAdmin user:
   ```ruby
   SuperAdmin.create!(
     name: 'Super Admin',
     email: 'admin@example.com',
     password: 'SecurePassword123!'
   )
   ```

3. Type `exit` to leave the console.
4. Access the superadmin dashboard at `http://localhost:3000/super_admin`.

---

## Creating an API Channel & Getting Access Tokens

Chatwoot's API Channel allows external applications, chatbots, CRM tools, or backend services to send and receive messages within Chatwoot.

### Create an API Channel Inbox

1. Log in to the Chatwoot dashboard.
2. In the left-hand navigation, click **Settings** (gear icon) -> **Inboxes**.
3. Click the **Add Inbox** button at the top right.
4. Choose the **API** channel tile.
5. Configure the channel details:
   - **Channel Name**: e.g., `CRM Backend API` or `Support Bot`.
   - **Webhook URL** (optional): The HTTPS endpoint on your backend where Chatwoot will push real-time events (new messages, status changes).
6. Click **Create API Channel**.
7. Assign agents to the inbox and click **Add Agents**.
8. Copy the generated **Inbox Identifier** (displayed on the screen or in Inbox Settings -> Configuration).

### Retrieve Your User API Access Token

To authenticate REST API calls:

1. Click on your profile avatar / name in the bottom-left corner of the dashboard.
2. Select **Profile Settings**.
3. Scroll down to the **Access Token** section.
4. Click **Copy Token** (or click **Reset Token** to generate a new one).

### Sample API Test Request

Test your connection using `curl`:

```bash
# 1. Fetch current user profile
curl -X GET "http://localhost:3000/api/v1/profile" \
  -H "api_access_token: <YOUR_USER_ACCESS_TOKEN>" \
  -H "Content-Type: application/json"

# 2. Create a contact via API
curl -X POST "http://localhost:3000/api/v1/accounts/1/contacts" \
  -H "api_access_token: <YOUR_USER_ACCESS_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Jane Doe",
    "email": "jane@example.com",
    "phone_number": "+1234567890"
  }'

# 3. Create a conversation in the API channel
curl -X POST "http://localhost:3000/api/v1/accounts/1/conversations" \
  -H "api_access_token: <YOUR_USER_ACCESS_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{
    "inbox_id": "<YOUR_INBOX_ID>",
    "contact_id": 1,
    "message": {
      "content": "Hello from backend service!"
    }
  }'
```

---

## Production Deployment on Render

### Render Blueprint Components

The `render.yaml` file defines a managed infrastructure blueprint:

| Component | Render Type | Details |
| :--- | :--- | :--- |
| `chatwoot-web` | `web` | Web service running `chatwoot/chatwoot:latest`, pre-deploy migration `rails db:chatwoot_prepare`, health check `/health_check` |
| `chatwoot-worker` | `worker` | Background worker running `bundle exec sidekiq -C config/sidekiq.yml` |
| `chatwoot-redis` | `redis` | Managed Redis instance for queue, cache, and WebSocket pub/sub |
| `chatwoot-postgres` | `database` | Managed PostgreSQL 15 database |

### Step 1: Push Code to Git

Initialize Git and push this repository to GitHub or GitLab:

```bash
git init
git add .
git commit -m "feat: initialize Chatwoot self-hosted deployment"
git branch -M main
git remote add origin https://github.com/<your-user>/<your-repo>.git
git push -u origin main
```

### Step 2: Connect Blueprint on Render

1. Log in to [Render Dashboard](https://dashboard.render.com/).
2. Click **New +** in the top navigation and select **Blueprint**.
3. Connect your Git repository (`Srmortal/Chatwoot`).
4. Render will detect `render.yaml` and list the managed resources (`chatwoot-web`, `chatwoot-worker`, and `chatwoot-redis`).
5. Click **Apply**.
6. Render will:
   - Connect to your external Aiven PostgreSQL instance.
   - Provision managed Redis for queues and cache.
   - Run `bundle exec rails db:chatwoot_prepare` via Render's `preDeployCommand`.
   - Deploy `chatwoot-web` and `chatwoot-worker`.

### Step 3: Post-Deploy Domain Configuration

1. After deployment succeeds, open the `chatwoot-web` service on Render and note your assigned URL (e.g., `https://chatwoot-web-xxxx.onrender.com`) or configure a custom domain (e.g., `https://chatwoot.yourcompany.com`).
2. Go to **Environment** on the `chatwoot-web` service.
3. Update `FRONTEND_URL` to match your actual URL (e.g. `https://chatwoot.yourcompany.com`).
4. Click **Save Changes**. The worker service will automatically inherit this value.

### External PostgreSQL Configuration (Aiven / Neon / Supabase)

Chatwoot's Rails database configuration (`config/database.yml`) requires both the connection URI and divided variables:

```yaml
# Full connection string
- key: DATABASE_URL
  sync: false # Paste: postgres://avnadmin:<PASSWORD>@pg-161848d2-youssefmohamedaly750-2283.d.aivencloud.com:17536/defaultdb?sslmode=require

# Divided individual variables required by Chatwoot Rails
- key: POSTGRES_HOST
  value: pg-161848d2-youssefmohamedaly750-2283.d.aivencloud.com
- key: POSTGRES_PORT
  value: "17536"
- key: POSTGRES_USERNAME
  value: avnadmin
- key: POSTGRES_PASSWORD
  sync: false # Paste your database password in Render
- key: POSTGRES_DATABASE
  value: defaultdb
- key: PGSSLMODE
  value: require
- key: POSTGRES_SSLMODE
  value: require
```

---

## Operational Commands & Troubleshooting

### Local Maintenance

```bash
# Restart all containers
docker compose restart

# Stop all containers
docker compose down

# Stop containers and remove volumes (WARNING: wipes all local data)
docker compose down -v

# Run database migrations after pulling a newer Chatwoot image
docker compose run --rm rails bundle exec rails db:migrate

# Access the Rails console in Docker
docker compose exec rails bundle exec rails console

# Access the PostgreSQL CLI
docker compose exec postgres psql -U postgres -d chatwoot_production
```

### Common Issues & Fixes

1. **Redis TLS OpenSSL verification errors in cloud environments:**
   - Set `REDIS_OPENSSL_VERIFY_MODE=none` in your environment variables.
2. **`ActiveRecord::ConnectionNotEstablished` on initial boot:**
   - Make sure `db:chatwoot_prepare` ran successfully before starting the web server.
3. **CORS or redirect loops on login:**
   - Verify `FRONTEND_URL` exactly matches the protocol, domain, and port in your browser address bar (including `https://` in production or `http://localhost:3000` locally).
