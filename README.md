# Firecrawl Self-Hosted

Self-hosted deployment of [Firecrawl](https://github.com/mendableai/firecrawl) with custom crawl-delay protection.

## Prerequisites

- Docker and Docker Compose
- Git

## Setup

### 1. Clone

```bash
git clone https://github.com/mendableai/firecrawl.git
cd firecrawl
git checkout f27c87d9c  # tested commit — newer versions may also work
```

### 2. Apply the crawl-delay patch

Three files are modified on top of upstream to add a `DEFAULT_CRAWL_DELAY` env var that enforces a minimum delay between crawl requests:

- `apps/api/src/config.ts` — adds the config field
- `apps/api/src/controllers/v1/crawl.ts` — applies the delay in v1 crawl
- `apps/api/src/controllers/v2/crawl.ts` — applies the delay in v2 crawl

If you have the patch file:

```bash
git apply crawl-delay.patch
```

### 3. Configure `.env`

Create `firecrawl/.env` with at minimum:

```env
# Required
USE_DB_AUTHENTICATION=false
REDIS_URL=redis://redis:6379
REDIS_RATE_LIMIT_URL=redis://redis:6379
PLAYWRIGHT_MICROSERVICE_URL=http://playwright-service:3000/scrape

# IP protection
CRAWL_CONCURRENT_REQUESTS=3
DEFAULT_CRAWL_DELAY=1
MAX_CONCURRENT_JOBS=5
BROWSER_POOL_SIZE=5
NUM_WORKERS_PER_QUEUE=8

# API key for requests (set to any value you like)
TEST_API_KEY=fc-your-key-here
```

All Supabase, Stripe, and PostHog vars can be left empty — they are for the hosted SaaS version.

### 4. Build and start

```bash
docker compose up -d --build
```

First build takes several minutes. Subsequent starts are fast.

### 5. Verify

```bash
# Check services
docker compose ps

# Test scrape
curl http://localhost:3002/v1/scrape \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer fc-your-key-here' \
  -d '{"url": "https://example.com"}'
```

## Services

| Service | Port | Purpose |
|---------|------|---------|
| api | 3002 | API + harness (spawns all workers internally) |
| redis | 6379 | Cache, rate limiting, crawl state |
| nuq-postgres | 5432 | Job queue database |
| rabbitmq | 5672 | Job notifications |
| playwright-service | 3000 | Browser-based scraping |

## Local modifications

### DEFAULT_CRAWL_DELAY

Forces a delay (in seconds) between crawl requests when the API caller doesn't specify one. When delay > 0, crawls are automatically single-threaded. Set to `0` to disable.

Combined with `CRAWL_CONCURRENT_REQUESTS=3` (default is 10), this keeps the scraper polite to target sites.

## Claude Code integration

Install the Firecrawl CLI and the official Claude Code plugin to give Claude access to scrape, crawl, search, and browser tools — all routed through the self-hosted API.

### 1. Install the CLI

```bash
npm install -g firecrawl-cli
```

### 2. Configure environment

Add to `~/.bashrc` (or `~/.zshrc`):

```bash
export FIRECRAWL_API_URL=http://localhost:3002
export FIRECRAWL_API_KEY=fc-selfhosted
```

Reload your shell or run `source ~/.bashrc`.

### 3. Install the Claude Code plugin

```bash
claude plugins install firecrawl@claude-plugins-official --scope user
```

Restart Claude Code after installing.

### 4. Verify

```bash
firecrawl --status
```

Should show "Authenticated via FIRECRAWL_API_KEY" with 99,999,999 credits (self-hosted unlimited).

## Common pitfalls

- Setting `USE_DB_AUTHENTICATION=true` without Supabase configured crashes workers on blocklist init.
- `nuq-postgres` must be ready before workers start (docker-compose handles this via `depends_on`).
- The `.env` file overrides docker-compose defaults — check both when debugging config issues.
