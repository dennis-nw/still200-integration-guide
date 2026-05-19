# Still200 Integration Guide

Everything you need to integrate your API with Still200 —
an uptime monitoring platform for indie developers.
This repo contains the API reference, integration docs,
and code samples to help you get your endpoints monitored quickly.

## What's Inside

- **API Reference** — endpoints, authentication, request/response formats,
and error codes.
- **Code Examples** — ready-to-run samples for integrating with Still200.

## Quick Start

1. **Prepare Your API:** Ensure the service you want to monitor
exposes a valid health check endpoint.
Follow the [API Set UP Guide](#api-set-up-guide) below to format your response.
2. **Download Still200:** Get the app to start managing your monitors.
3. **Register a Monitor:** Add your newly created health check endpoint URL into the app.
4. **Start Monitoring:** Still200 will immediately begin checking
your endpoint on your configured schedule.
Once registered, Still200 will begin checking your endpoint on the
schedule you configured.
You'll receive real-time alerts if anything goes down — including AI-powered root
cause analysis. Currently delivered via Apple Push Notifications, with
more channels coming soon.

## API Set Up Guide

### Health Check Response Spec

Still200 expects your health check endpoint to return a JSON response
in a specific format. This allows Still200 to monitor not just whether your API is up,
but the health of individual dependencies like databases, caches, and external services.

#### Response Format

```json
{
  "service_name": "My API",
  "status": "healthy",
  "checks": {
    "database": {
      "status": "healthy",
      "latency_ms": 50.82
    },
    "redis": {
      "status": "healthy",
      "latency_ms": 7.12
    }
  }
}
```

### Code Samples

### Contributing

Found a bug in the examples or want to add a sample in another language?
Pull Requests are welcome.

#### Python

```python
import asyncio
from fastapi import APIRouter, Depends
from pydantic import BaseModel
from typing import Dict, Any

router = APIRouter()

class CheckResult(BaseModel):
    status: str
    latency_ms: float

class HealthCheckResponse(BaseModel):
    service_name: str
    status: str
    checks: Dict[str, CheckResult] = {}

@router.get("/health", response_model=HealthCheckResponse)
async def health(
    db: AsyncSession = Depends(get_db),
    redis_client: aioredis.Redis = Depends(get_redis),
) -> HealthCheckResponse:
    
    # Execute dependency checks concurrently
    async with asyncio.TaskGroup() as tg:
        db_check = tg.create_task(check_db(db=db))
        redis_check = tg.create_task(check_redis(redis_client=redis_client))
        
    # Results are safely available after the context manager exits
    checks = {
        "database": db_check.result(), 
        "redis": redis_check.result()
    }
    
    overall_health = (
        "healthy" if all(c.status == "healthy" for c in checks.values()) 
        else "unhealthy"
    )
    
    return HealthCheckResponse(
        service_name="My API",
        status=overall_health,
        checks=checks,
    )
```