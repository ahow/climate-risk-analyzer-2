# Heroku Deployment Guide

This guide explains how to deploy the Climate Risk Analyzer to Heroku with parallel processing capabilities.

## Architecture Overview

The Heroku deployment uses two types of dynos:
- **Web dyno**: Serves the React frontend and Express API
- **Worker dyno**: Processes company analyses in parallel using a database-backed job queue

This architecture allows you to:
- Scale workers independently (add more worker dynos for faster processing)
- Survive dyno restarts without losing progress
- Process multiple companies simultaneously

## Prerequisites

1. Heroku CLI installed and logged in
2. PostgreSQL database (Heroku Postgres add-on)
3. API keys for:
   - Anthropic Claude (`ANTHROPIC_API_KEY`)
   - Google Custom Search (`GOOGLE_SEARCH_API_KEY`, `GOOGLE_SEARCH_ENGINE_ID`)

## Quick Deploy

### Option 1: Deploy Button
Click the "Deploy to Heroku" button if available, which uses `app.json` for configuration.

### Option 2: Manual Deployment

```bash
# Create Heroku app
heroku create your-app-name

# Add PostgreSQL
heroku addons:create heroku-postgresql:essential-0

# Set environment variables
heroku config:set NODE_ENV=production
heroku config:set ANTHROPIC_API_KEY=your_key
heroku config:set GOOGLE_SEARCH_API_KEY=your_key
heroku config:set GOOGLE_SEARCH_ENGINE_ID=your_engine_id
heroku config:set SESSION_SECRET=$(openssl rand -base64 32)
heroku config:set PARALLEL_JOBS=3

# Deploy
git push heroku main

# Run database migrations
heroku run npm run db:push

# Scale dynos
heroku ps:scale web=1 worker=1
```

## Configuration Options

### Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `PARALLEL_JOBS` | Companies processed simultaneously per worker | 3 |
| `POLL_INTERVAL` | How often workers check for new jobs (ms) | 5000 |
| `JOB_TIMEOUT` | Max time for a single job before retry (ms) | 300000 |

### Scaling Workers

For faster processing, scale up worker dynos:

```bash
# Single worker (3 parallel jobs = 3 companies at once)
heroku ps:scale worker=1

# Two workers (6 companies at once)
heroku ps:scale worker=2

# Three workers (9 companies at once)
heroku ps:scale worker=3
```

Each worker processes `PARALLEL_JOBS` companies simultaneously.

## API Endpoints (Heroku-specific)

### Start Batch Analysis
```
POST /api/heroku/batch/start
Body: { "parallelWorkers": 3, "companyIds": [1, 2, 3] } // optional filters
```

### Check Batch Status
```
GET /api/heroku/batch/status
Response: {
  "batchId": 1,
  "status": "running",
  "totalJobs": 100,
  "completedJobs": 45,
  "failedJobs": 2,
  "progressPercent": 45
}
```

### Cancel Batch
```
POST /api/heroku/batch/cancel
```

### Retry Failed Jobs
```
POST /api/heroku/batch/retry-failed
```

### View Jobs
```
GET /api/heroku/jobs?status=failed&limit=50
```

## Cost Estimation

### Heroku Costs (Monthly)
- Basic Web Dyno: ~$7/month
- Basic Worker Dyno: ~$7/month per worker
- Heroku Postgres Essential-0: ~$5/month
- **Total minimum**: ~$19/month

### API Costs (Variable)
- Anthropic Claude: ~$0.50-2.00 per company
- Google Custom Search: Free tier includes 100 queries/day, then $5/1000 queries

For 1000 companies:
- Claude API: ~$500-2000
- Google Search (if exceeding free tier): ~$50

## Monitoring

### View Logs
```bash
heroku logs --tail -a your-app-name
```

### Filter by Process
```bash
# Web dyno logs only
heroku logs --tail --dyno web

# Worker dyno logs only
heroku logs --tail --dyno worker
```

## Troubleshooting

### Jobs Stuck in "Processing"
Jobs may get stuck if a worker crashes. The worker automatically reclaims stale jobs (older than `JOB_TIMEOUT`), but you can also manually reset them:

```sql
UPDATE analysis_jobs 
SET status = 'pending', worker_id = NULL, attempts = 0 
WHERE status = 'processing' AND started_at < NOW() - INTERVAL '10 minutes';
```

### Worker Not Picking Up Jobs
1. Check worker is running: `heroku ps`
2. Check logs for errors: `heroku logs --tail --dyno worker`
3. Verify database connection: `heroku pg:info`

### High API Costs
- Reduce `PARALLEL_JOBS` to process fewer companies simultaneously
- Monitor Claude API usage in Anthropic dashboard
- Consider caching document content to avoid re-processing

## Migrating from Replit

1. Export your database from Replit (use the SQL export feature)
2. Import into Heroku Postgres: `heroku pg:psql < export.sql`
3. Verify data: `heroku pg:psql` then `SELECT COUNT(*) FROM companies;`
4. Deploy code and start workers

## Pre-Deployment Setup

Before deploying, add the worker script to your `package.json` scripts:

```json
"scripts": {
  ...
  "worker": "NODE_ENV=production node --loader tsx server/heroku-worker.ts",
  "worker:dev": "NODE_ENV=development tsx server/heroku-worker.ts"
}
```

## File Structure

```
Procfile              # Defines web and worker processes
app.json              # Heroku app configuration
server/heroku-worker.ts    # Worker process code
server/heroku-routes.ts    # Queue-based API endpoints
shared/schema.ts      # Includes job queue tables
```
