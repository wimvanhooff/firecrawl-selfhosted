# Firecrawl Self-Hosted Project

This is a self-hosted deployment of [Firecrawl](https://github.com/mendableai/firecrawl).
The cloned repo lives in `firecrawl/`. It has its own upstream `CLAUDE.md` with development workflow instructions.

## Self-hosted setup

- Running via `docker compose` from `firecrawl/docker-compose.yaml`
- No Supabase — `USE_DB_AUTHENTICATION=false` is required in `.env`
- No fire-engine — using Playwright service for browser scraping

## Local modifications (on top of upstream)

### IP protection / rate limiting
We added a `DEFAULT_CRAWL_DELAY` env var (config.ts + v1/v2 crawl controllers) that forces a
delay between crawl requests when the API caller doesn't specify one. When delay > 0, crawls
are automatically single-threaded. Set to 0 to disable.

Relevant `.env` settings:
- `CRAWL_CONCURRENT_REQUESTS=3` (default was 10)
- `DEFAULT_CRAWL_DELAY=1` (1 second between requests)

### Key architecture notes
- **Harness** (`apps/api/src/harness.ts`) — orchestrates all services as child processes
- **NuQ workers** — PostgreSQL-backed job queue (replaced BullMQ for scrape jobs)
  - 5 workers by default (ports 3006-3010), plus prefetch and reconciler workers
  - Requires PostgreSQL (`nuq-postgres` container) and RabbitMQ
- **Config** (`apps/api/src/config.ts`) — Zod-validated env vars with defaults
- **Scrape flow**: API controller → NuQ queue → worker → engine (Playwright/fetch) → result

### Docker services
| Service | Port | Purpose |
|---------|------|---------|
| api | 3002 | API + harness (spawns all workers internally) |
| redis | 6379 | Cache, rate limiting, crawl state |
| nuq-postgres | 5432 | Job queue database |
| rabbitmq | 5672 | Job notifications |
| playwright-service | 3000 | Browser-based scraping |

### Common pitfalls
- `USE_DB_AUTHENTICATION=true` without Supabase configured → workers crash on blocklist init
- `nuq-postgres` must be ready before workers start (harness waits in docker-compose mode)
- The `.env` file overrides docker-compose defaults — check both when debugging config issues

## Replicating on a new machine

See `README.md` for full step-by-step instructions. In short:

1. Clone upstream Firecrawl at commit `f27c87d9c`
2. Apply the crawl-delay patch (3 files in `apps/api/src/`)
3. Create `.env` with `USE_DB_AUTHENTICATION=false` and the IP-protection settings above
4. `docker compose up -d --build`

Generate the patch from this machine with:
```bash
cd firecrawl && git diff HEAD > crawl-delay.patch
```
