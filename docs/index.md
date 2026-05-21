# Still200 Integration Guide

Everything you need to integrate your API with
[Still200](https://still200.com) —
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
    Follow the [API Set UP Guide](#api-set-up-guide) below to format
    your response. You can then validate that your health check URL
    responds with the expected format by running:

    ```bash
    curl -X POST -H "Content-Type: application/json" \
    -d '{"url": "https://myapi.com/health"}' \
    https://api.still200.com/monitors/validate
    ```

    **Example Success Response**

    ```json
    {
      "service_name": "Healthy API",
      "checks": {
        "database": {
          "status": "healthy",
          "latency_ms": 74.33
        },
        "redis": {
          "status": "healthy",
          "latency_ms": 5.71
        }
      },
      "error": null
    }
    ```

    **Example Validation Failure**
    <!-- markdownlint-disable MD013 -->
    ```json
    {
      "service_name": "",
      "status": "unknown",
      "status_code": 200,
      "checks": {},
      "error": "Invalid response format. Check that your response conforms to the expected response."
    }
    ```
    <!-- markdownlint-enable MD013 -->

2. **Download Still200:** Get the app to start managing your monitors.
3. **Register a Monitor:** Add your newly created health check endpoint
    URL into the app.
4. **Start Monitoring:** Once registered, Still200 will begin checking
    your endpoint on the schedule you configured.
    You'll receive real-time alerts if anything goes down —
    including AI-powered root cause analysis.
    Currently delivered via Apple Push Notifications, with
    more channels coming soon.

## API Set Up Guide

### Health Check Response Spec

Still200 expects your health check endpoint to return a JSON response
in a specific format. This allows Still200 to monitor not just whether
your API is up, but the health of individual dependencies like
databases, caches, and external services.

#### Response Format

```json
{
  "service_name": "My API",
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

#### Python (FastAPI)

```python
import asyncio
from fastapi import FastAPI, Depends
from pydantic import BaseModel
from typing import Dict, TypedDict, Literal

app = FastAPI()

class CheckResult(TypedDict):
    status: Literal["healthy", "degraded", "unhealthy"]
    latency_ms: float
    error: NotRequired[str]

class HealthCheckResponse(BaseModel):
    service_name: str
    checks: Dict[str, CheckResult] = {}

@app.get("/health", response_model=HealthCheckResponse)
async def health(
    db: AsyncSession = Depends(get_db),
    redis_client: aioredis.Redis = Depends(get_redis),
) -> HealthCheckResponse:
    
    # Execute dependency checks concurrently
    async with asyncio.TaskGroup() as tg:
        db_check = tg.create_task(check_db(db=db))
        redis_check = tg.create_task(check_redis(redis_client=redis_client))

    # check methods return a dict with status and latency_ms
    checks = {
        "database": db_check.result(), 
        "redis": redis_check.result()
    }
    
    return HealthCheckResponse(
        service_name="My API",
        checks=checks,
    )
```

## Contributing

Found a bug in the examples or want to add a sample in another language?
Pull Requests are welcome.
