---
title: Still200 Integration Guide
---

<!-- markdownlint-disable MD022 -->

## Table of Contents
{: .no_toc .text-delta }

<!-- markdownlint-enable MD022 -->

- TOC
{:toc}

Everything you need to integrate your API with
[Still200](https://still200.com) — an uptime monitoring
platform for developers.
Still200 monitors your API by polling a health endpoint that you define
and expose. Choose the setup that fits your needs:

| | [Simple Setup](#simple-setup) | [Full Setup](#full-setup) |
| --- | --- | --- |
| **Time to integrate** | ~2 minutes | ~15 minutes |
| **What's monitored** | API reachability | API + individual dependencies |
| **Alert detail** | Up/down | Per-dependency root cause analysis |
| **Best for** | Getting started fast | Production services |

## Backend Setup

### Simple Setup

The fastest way to set up monitoring. Add a single route that returns your
service name, register it in the app, and you're done.

```python
from fastapi import FastAPI
 
app = FastAPI()
 
@app.get("/health")
async def health():
    return {"service_name": "my-api"}
```

That's it. No dependencies, no extra packages.
Make sure your endpoint returns a `200` status code — Still200 reads health
from the response body, but won't parse it if the request fails.

### Full Setup

Follow the [Health Check Spec](#health-check-spec) below to
format your response correctly.

## Validate your endpoint

Before registering your health check URL, confirm the format is correct:

A passing response looks like this for the simple setup:

```bash
curl -X POST -H "Content-Type: application/json" \
    -d '{"url": "https://myapi.com/health"}' \
    https://api.still200.com/monitors/validate
```

```json
{
  "service_name": "my-api",
  "status": "healthy",
  "status_code": 200,
  "checks": {},
  "error": null
}
```

And for the full set up:

 ```json
    {
      "service_name": "My API",
      "status": "healthy",
      "status_code": 200,
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

If your format is wrong, the response will include an `error` string describing
what to fix:

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

Once validated, you can register your monitor in the app.

## Register Your Monitor

**1. Download Still200** from the [Apple App Store](https://apps.apple.com/us/app/still200/id6770858177)

**2. Add your monitor.** Paste your health check URL into the app and set
your preferred check interval.

**3. You're live.** Still200 will begin polling your endpoint immediately.
If a check fails, you'll receive a push notification on your phone.
More alert channels coming soon.

---

## Health Check Spec

Still200 monitors the individual dependencies inside your service,
not just whether the HTTP response is 200. To support this,
your health endpoint must return a JSON body in the format described below.

### Endpoint Requirements

- **Method:** `GET`
- **Content-Type:** `application/json`
- **Status Code:** Return `200` regardless of internal health status.
  Still200 reads health from the response body, not the HTTP status code.

### Fields
<!-- markdownlint-disable MD013 -->

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `service_name` | string | Yes | Human-readable name shown in the app and alert notifications. |
| `checks` | object | No | Map of dependency names to their status. An empty object `{}` is valid. |
| `checks[n].status` | string | Yes | One of `healthy`, `degraded`, or `unhealthy`. See [Status Values](#status-values) below. |
| `checks[n].latency_ms` | float | No | Time in milliseconds to connect to or query the dependency. |
| `checks[n].error` | string | No | Error detail surfaced in alerts and RCA when status is not `healthy`. |

### Status Values

| Value | Meaning | Still200 behaviour |
| --- | --- | --- |
| `healthy` | Dependency is reachable and performing normally. | No action. |
| `degraded` | Reachable but slow or returning soft errors. | Triggers an alert after the consecutive failure threshold is met. |
| `unhealthy` | Dependency is down or unreachable. | Triggers an alert after the consecutive failure threshold is met. |

<!-- markdownlint-enable MD013 -->
---

## Code Samples

### Python (FastAPI)

```python
import asyncio
from fastapi import FastAPI, Depends
from pydantic import BaseModel
from typing import Dict, Literal, Optional

app = FastAPI()

class CheckResult(BaseModel):
    status: Literal["healthy", "degraded", "unhealthy"]
    latency_ms: float
    error: Optional[str] = None

class HealthCheckResponse(BaseModel):
    service_name: str
    checks: Dict[str, CheckResult] = {}

@app.get("/health", response_model=HealthCheckResponse)
async def health(
    db = Depends(get_db),
    redis_client = Depends(get_redis)
) -> HealthCheckResponse:
    # Execute dependency checks concurrently
    async with asyncio.TaskGroup() as tg:
        db_check = tg.create_task(check_db(db=db))
        redis_check = tg.create_task(check_redis(redis_client=redis_client))

    checks = {
        "database": db_check.result(), 
        "redis": redis_check.result()
    }
    
    return HealthCheckResponse(
        service_name="My API",
        checks=checks,
    )
```

### Contributing

Found a bug in the examples or want to add a sample in another language?
Pull Requests are welcome.

## Failure Semantics

**Consecutive failure threshold.**
Still200 does not alert on the first failed check. An alert is sent once
your endpoint has failed on a set number of consecutive checks.
This prevents transient blips from waking you up at 3am.

**Timeout.**
If your endpoint does not respond within the check timeout,
Still200 treats it as a failed check. Keep your health handler fast —
aim for under 500ms. Running dependency checks concurrently (as shown
in the samples above) helps significantly.

**HTTP status codes.**
Still200 reads health from the response body. Return `200` from your
endpoint even when internal dependencies are unhealthy — Still200 inspects
the `status` fields inside `checks` to determine alert eligibility.

**Partial failures.**
A single `unhealthy` check in an otherwise healthy response will trigger
the failure path after the threshold is met.

## FAQs

**Does my health endpoint need to be publicly accessible?**

Yes. Still200 polls your endpoint from its own infrastructure, so it must
be reachable over the public internet.

**Can I add custom checks beyond databases and caches?**

Yes. The `checks` map accepts any string key. Common additions include
external API dependencies (`stripe`, `sendgrid`).
Name them whatever is meaningful in your context.
