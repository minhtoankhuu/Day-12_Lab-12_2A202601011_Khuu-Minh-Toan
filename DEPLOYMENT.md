# Deployment Information

## Public URL
https://lab12deplou-production.up.railway.app

## Platform
Railway

## Test Commands

### Health Check
```bash
curl https://lab12deplou-production.up.railway.app/health
# Expected: {"status": "ok"}
```

### API Test (with authentication)
```bash
curl -X POST https://lab12deplou-production.up.railway.app/ask \
  -H "X-API-Key: YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"question": "Am I on the cloud?"}'
# Expected: {"answer": "...", "platform": "Railway"}
```

### Auth test (no key → 401)
```bash
curl https://lab12deplou-production.up.railway.app/ask -X POST \
  -H "Content-Type: application/json" -d '{"question":"test"}'
# Expected: HTTP 401
```

## Environment Variables Set on Railway
- PORT (auto-injected by Railway)
- ENVIRONMENT=production
- AGENT_API_KEY=<secret>

## Deploy commands
```bash
cd 03-cloud-deployment/railway
railway login
railway init
railway variables set AGENT_API_KEY=<your-key>
railway variables set ENVIRONMENT=production
railway up . --path-as-root
railway domain
```
