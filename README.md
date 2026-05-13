# HookForge

> *A generic service to send, retry, and manage webhooks.*

## Description

### What?

HookForge is a self-contained service that lets you send, retry, and manage event-triggered POST requests (webhooks). It provides a fully dockerized setup that is easy to orchestrate, manage, and scale.

### Why?

A webhook is essentially a POST request triggered by an event. However, managing the full lifecycle of webhooks is tricky:

- Dealing with server failures on both the sending and receiving ends
- Managing HTTP timeouts
- Retrying requests gracefully without overloading recipients
- Avoiding retry loops on the sending side
- Monitoring and providing scope for manual interventions
- Scaling quickly — vertically or horizontally
- Decoupling webhook logic from your primary application logic

HookForge takes care of all of this, so you don't have to.

### How?

HookForge exposes a single endpoint where you post your webhook payload, destination URL, and auth details — it handles the POST request asynchronously in the background using:

- **[FastAPI](https://fastapi.tiangolo.com/)** + **[Uvicorn](https://www.uvicorn.org/)** — ASGI server
- **[Redis](https://redis.io/)** + **[RQ](https://python-rq.org/docs/jobs/)** — async message queue with failure handling
- **[Rqmonitor](https://github.com/pranavgupta1234/rqmonitor)** — dashboard for monitoring and retrying failed jobs
- **[Rich](https://github.com/willmcgugan/rich)** — colorized container logs

**Architecture:**

```
[Client] → POST payload → [App] → enqueues job → [Worker] → POST to destination
                                                [Monitor] — GUI dashboard
```

Multiple worker instances can be spawned to achieve horizontal scale-up.

## Installation

* Make sure you have [Docker](https://www.docker.com/) and [Docker Compose V2](https://docs.docker.com/compose/cli-command/) installed.
* Clone the repository and head to the root directory.
* Start all services:

```bash
make start-servers
```

This starts:
- `app` server on port `5000`
- Alpine-based Redis on port `6380`
- A single `worker` instance
- `rqmonitor` on port `8899`

To shut everything down:

```bash
make stop-servers
```

## Usage

### Send a webhook via cURL

```sh
curl -X 'POST' \
  'http://localhost:5000/hook_slinger/' \
  -H 'accept: application/json' \
  -H 'Authorization: Token $5$1O/inyTZhNvFt.GW$Zfckz9OL.lm2wh3IewTm8YJ914wjz5txFnXG5XW.wb4' \
  -H 'Content-Type: application/json' \
  -d '{
  "to_url": "https://webhook.site/b30da7ce-c3cc-47e2-b2ae-68747b3d7789",
  "to_auth": "",
  "tag": "Dhaka",
  "group": "Bangladesh",
  "payload": {
    "greetings": "Hello, world!"
  }
}' | python -m json.tool
```

Expected response:

```json
{
    "status": "queued",
    "ok": true,
    "message": "Webhook registration successful.",
    "job_id": "Bangladesh_Dhaka_a07ca786-0b7a-4029-bac0-9a7c6eb68a98",
    "queued_at": "2021-11-06T16:54:54.728999"
}
```

### Send a webhook via Python

```python
import asyncio
from http import HTTPStatus
from pprint import pprint

import httpx


async def send_webhook() -> None:
    wh_payload = {
        "to_url": "https://webhook.site/b30da7ce-c3cc-47e2-b2ae-68747b3d7789",
        "to_auth": "",
        "tag": "Dhaka",
        "group": "Bangladesh",
        "payload": {"greetings": "Hello, world!"},
    }

    async with httpx.AsyncClient(http2=True) as session:
        headers = {
            "Content-Type": "application/json",
            "Authorization": (
                "Token $5$1O/inyTZhNvFt.GW$Zfckz9OL.lm2wh3IewTm8YJ914wjz5txFnXG5XW.wb4"
            ),
        }
        response = await session.post(
            "http://localhost:5000/hook_slinger",
            headers=headers,
            json=wh_payload,
            follow_redirects=True,
        )
        assert response.status_code == HTTPStatus.ACCEPTED
        pprint(response.json())


if __name__ == "__main__":
    asyncio.run(send_webhook())
```

### Container logs

```bash
make app-logs      # App server logs
make worker-logs   # Worker instance logs
```

### Horizontal scale-up

Spawn 3 worker containers:

```bash
make stop-servers
make worker-scale n=3
```

## Troubleshooting

Ensure `to_url` in your payload can accept POST requests and returns HTTP 201. Example payload structure:

```json
{
    "to_url": "https://your-destination.com/webhook",
    "to_auth": "",
    "tag": "my-tag",
    "group": "my-group",
    "payload": {"key": "value"}
}
```

By default, HookForge retries a failed job **3 times** with **5 second linear backoff**. Configure via `.env`:

```
MAX_RETRIES=3
INTERVAL=5
```

## License

[MIT](LICENSE)
