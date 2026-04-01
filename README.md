# Still200 Integration Guide

Everything you need to integrate your API with Still200 —
an uptime monitoring platform for indie developers.
This repo contains the API reference, integration docs,
and code samples to help you get your endpoints monitored quickly.

## What's Inside

API Reference — endpoints, authentication, request/response formats,
and error codes.
Code Examples — ready-to-run samples for integrating with Still200.

## Quick Start

1. Ensure the service you want to monitor is set up.
Follow this guide to get set up.
2. Download the app to start using Still200.
3. Register a monitor.
4. Start monitoring.
Once registered, Still200 will begin checking your endpoint on the
schedule you configured. You'll receive alerts, Apple Push Notifications
for now but we'll be adding more channels, if anything goes down —
including AI-powered root cause analysis.

## Contributing

Found a bug in the examples or want to add a sample in another language? PRs are welcome.

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
      "latency_ms": 150.82
    },
    "redis": {
      "status": "healthy",
      "latency_ms": 87.12
    }
  }
}
```
