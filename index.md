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
| **Time to integrate** | ~2 minutes | ~5 minutes |
| **What's monitored** | API reachability | API + individual dependencies |
| **Status classification** | N/A | Automatic — report `latency_ms` and/or `error`, Still200 does the rest |
| **Alert detail** | Up/down | Per-dependency root cause analysis |
| **Best for** | Getting started fast | Production services |

## Backend Setup

### Simple Setup

The fastest way to set up monitoring. Add a single route that returns your
service name, register it in the app, and you're done.

```python
from fastapi import FastAPI, status

app = FastAPI()

@app.get("/health", status_code=status.HTTP_200_OK)
async def health():
    return {"service_name": "my-api"}
```

That's it. No dependencies, no extra packages.
Make sure your endpoint returns a `200` status code — Still200 reads health
from the response body, but won't parse it if the request fails.

### Full Setup

Report each dependency's latency and/or error.
Still200 classifies the status for you.

```python
import time
from fastapi import FastAPI

app = FastAPI()

@app.get("/health")
async def health():
    checks = {}

    start = time.perf_counter()
    try:
        await db.execute("SELECT 1")
        checks["database"] = {"latency_ms": (time.perf_counter() - start) * 1000}
    except Exception as e:
        checks["database"] = {"error": str(e)}

    return {"service_name": "my-api", "checks": checks}
```

That's the entire contract: report what happened, Still200 decides what it
means.

Follow the [Health Check Spec](#health-check-spec) below for the complete
field reference.

## Validate your endpoint

Before registering your health check URL, confirm the format is correct:

```bash
curl -X POST -H "Content-Type: application/json" \
    -d '{"url": "https://myapi.com/health"}' \
    https://api.still200.com/monitors/validate
```

A passing response looks like this for the simple setup:

```json
{
  "service_name": "my-api",
  "status": "healthy",
  "status_code": 200,
  "checks": {},
  "error": null
}
```

For the full setup, here's what **your endpoint** would return:

```json
{
  "service_name": "My API",
  "status": "healthy",
  "checks": {
    "database": {"latency_ms": 74.33},
    "redis": {"latency_ms": 4.11}
  }
}
```

And here's what `/monitors/validate` reports back, with each check's status
filled in:

```json
{
  "service_name": "My API",
  "status": "healthy",
  "status_code": 200,
  "checks": {
    "database": {
      "status": "healthy",
      "latency_ms": 74.33,
      "error": null
    },
    "redis": {
      "status": "healthy",
      "latency_ms": 4.11,
      "error": null
    }
  },
  "error": null
}
```

[Status Derivation](#status-derivation).

If your format is wrong, the response will include an `error` string
describing what to fix, and `checks` will be `null`, meaning Still200
couldn't determine your checks at all (as opposed to `{}`, which means it
reached you and you explicitly reported zero checks):

<!-- markdownlint-disable MD013 -->
```json
{
  "service_name": "",
  "status": "unknown",
  "status_code": 200,
  "checks": null,
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
your health endpoint must adhere to the spec below and
return a JSON body in the format described below.

### Endpoint Requirements

- **Method:** `GET`
- **Content-Type:** `application/json`
- **Status Code:** Return `200` regardless of internal health status.
  Still200 reads health from the response body, not the HTTP status code.
- **Timeout:** Still200 waits up to 5 seconds for a response. Anything
  slower is treated as a failed check. See [Failure Semantics](#failure-semantics).

### Fields
<!-- markdownlint-disable MD013 -->

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `service_name` | string | Yes | Human-readable name shown in the app and alert notifications. |
| `checks` | object | No | Map of dependency names to their status. An empty object `{}` is valid. |
| `checks[n].latency_ms` | float | No | Time in milliseconds to connect to or query the dependency. Used to auto-detect `degraded` when `status` is omitted. |
| `checks[n].error` | string | No | Error detail. Surfaced in alerts and root-cause summaries. If present, the check is automatically marked `unhealthy`. |

<!-- markdownlint-enable MD013 -->

### Status Derivation

Still200 derives the health status based on the response your endpoint returns:

| Condition | Derived status |
| --- | --- |
| `error` is present | `unhealthy` |
| No `error`, and `latency_ms` is over 1,000ms | `degraded` |
| Anything else, including an empty `{}` | `healthy` |

### Status Values
<!-- markdownlint-disable MD013 -->

| Value | Meaning | Still200 behaviour |
| --- | --- | --- |
| `healthy` | Dependency is reachable and performing normally. | No action. |
| `degraded` | Reachable but slow | Shown in the app and in root-cause summaries. **Does not** trigger an alert, no matter how long it persists — see [Failure Semantics](#failure-semantics). |
| `unhealthy` | Dependency is down or unreachable. | Counts toward the alert threshold. |

<!-- markdownlint-enable MD013 -->

---

## Code Samples

### Python (FastAPI)

```python
import asyncio
import time
from typing import Dict, Optional

from fastapi import Depends, FastAPI
from pydantic import BaseModel

app = FastAPI()


class CheckResult(BaseModel):
    latency_ms: Optional[float] = None
    error: Optional[str] = None


class HealthCheckResponse(BaseModel):
    service_name: str
    checks: Dict[str, CheckResult] = {}


async def check_db(db) -> CheckResult:
    start = time.perf_counter()
    try:
        await db.execute("SELECT 1")
        return CheckResult(latency_ms=(time.perf_counter() - start) * 1000)
    except Exception as e:
        return CheckResult(error=str(e))


async def check_redis(redis_client) -> CheckResult:
    start = time.perf_counter()
    try:
        await redis_client.ping()
        return CheckResult(latency_ms=(time.perf_counter() - start) * 1000)
    except Exception as e:
        return CheckResult(error=str(e))


@app.get("/health", response_model=HealthCheckResponse)
async def health(
    db=Depends(get_db),
    redis_client=Depends(get_redis),
) -> HealthCheckResponse:
    # Run dependency checks concurrently to keep the endpoint fast
    async with asyncio.TaskGroup() as tg:
        db_check = tg.create_task(check_db(db))
        redis_check = tg.create_task(check_redis(redis_client))

    return HealthCheckResponse(
        service_name="My API",
        checks={"database": db_check.result(), "redis": redis_check.result()},
    )
```

### Contributing

Found a bug in the examples or want to add a sample in another language?
Pull Requests are welcome.

## Failure Semantics

**Consecutive failure threshold.**
Still200 does not alert on the first failed check. An alert fires once your
endpoint fails 3 consecutive polls. This prevents a single transient blip
from waking you up at 3am. `degraded` results never count toward this
threshold, no matter how many occur in a row.

**Timeout.**
If your endpoint doesn't respond within 5 seconds, Still200 treats the poll
as failed — the same as if you'd returned an error. Keep your health
handler fast: aim for well under that, and run dependency checks
concurrently (as shown in the sample above) rather than sequentially.

**Malformed responses.**
A non-200 status, unparseable JSON, or a `checks` entry that doesn't match
the expected shape is treated as a failed poll rather than silently
ignored — the same 3-consecutive-failure threshold applies. Use
[Validate your endpoint](#validate-your-endpoint) to confirm your format
ahead of time.

**HTTP status codes.**
Still200 reads health from the response body, not the HTTP status. Return
`200` even when internal dependencies are unhealthy — Still200 inspects
each entry in `checks` (explicit or derived) to determine alert eligibility.

**Partial failures.**
The worst status among your checks wins. A single `unhealthy` check in an
otherwise healthy response triggers the failure path once the threshold is
met. A response that's entirely `degraded` — no matter how long — will not.

## FAQs

**Do I have to classify each dependency as healthy/degraded/unhealthy myself?**

No — that's the point of the Full Setup. Report `latency_ms` and/or `error`
per check and Still200 classifies it for you.

**Why didn't I get an alert when a dependency was degraded?**

By design. `degraded` means "still working, just slow or flaky" — Still200
tracks and surfaces it, but only `unhealthy` (and unreachable or malformed
responses) count toward the alert threshold. This keeps you from getting
paged for something that isn't actually down.

**What's the difference between `checks: {}` and `checks: null` in the
validate response?**

`{}` means Still200 reached your endpoint and it explicitly reported zero
checks — a valid, simple integration. `null` means Still200 couldn't
determine your checks at all (the request failed, timed out, or the
response wasn't valid JSON) — check the `error` field for why.

**Does my health endpoint need to be publicly accessible?**

Yes. Still200 polls your endpoint from its own infrastructure, so it must
be reachable over the public internet.

**Can I add custom checks beyond databases and caches?**

Yes. The `checks` map accepts any string key. Common additions include
external API dependencies (`stripe`, `sendgrid`). Name them whatever is
meaningful in your context.
