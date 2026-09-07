# Building Resilient HTTP Clients in Python with httpx

Production services talk to other services constantly: payment APIs, internal microservices, object storage, webhooks. Those calls fail in predictable ways—slow responses, transient 503s, connection resets—and a naive `requests.get()` leaves your app brittle. This guide shows how to build a practical, resilient HTTP client with [httpx](https://www.python-http.org/), focusing on timeouts, retries, and idempotency so you retry the right calls the right way.

## Why httpx for resilience work

httpx is a modern HTTP client for Python with sync and async APIs, connection pooling, and first-class timeout configuration. Unlike older patterns that bolt retries onto a global session with unclear semantics, httpx makes it straightforward to set per-request timeouts and compose transport-level behavior. You still need to design retry policy yourself (or via a small helper)—and that design is where most reliability bugs hide.

We will use the synchronous client here for clarity. The same ideas apply to `httpx.AsyncClient`.

## Start with timeouts, not retries

Retries without timeouts amplify outages: every slow upstream becomes a pile of waiting workers. Configure timeouts before you add retries.

httpx exposes a `Timeout` object with connect, read, write, and pool components:

```python
import httpx

timeout = httpx.Timeout(
    connect=5.0,   # seconds to establish TCP/TLS
    read=10.0,     # seconds waiting for response body
    write=10.0,    # seconds sending the request body
    pool=5.0,      # seconds waiting for a connection from the pool
)

client = httpx.Client(timeout=timeout, base_url="https://httpbin.org")
```

A single float (`timeout=10.0`) applies to all phases, which is fine for demos and dangerous for production. Prefer explicit components: a payment authorize might need a longer read timeout than a health check.

Always close the client (or use a context manager) so pooled connections are released:

```python
with httpx.Client(timeout=timeout, base_url="https://httpbin.org") as client:
    response = client.get("/get")
    response.raise_for_status()
    print(response.json()["url"])
```

## Classify failures before you retry

Not every error deserves a retry. A good rule of thumb:

| Situation | Retry? | Why |
|-----------|--------|-----|
| Connection error / DNS blip | Yes | Often transient |
| Timeout (read/connect) | Maybe | Upstream may still be processing |
| HTTP 408, 429, 500, 502, 503, 504 | Yes (with backoff) | Classic transient statuses |
| HTTP 400, 401, 403, 404, 422 | No | Client/auth/business errors |
| HTTP 409 Conflict | Usually no | Needs application logic |

Retrying a `POST /charges` after a read timeout can double-bill a customer if the first request succeeded and you never saw the response. That is an **idempotency** problem, not a transport problem.

## Implement retries with exponential backoff and jitter

Here is a small, dependency-free retry helper that only retries safe conditions. It uses exponential backoff with full jitter (a pattern popularized by AWS architecture blogs and widely used in client libraries):

```python
import random
import time
from typing import Callable

import httpx

RETRYABLE_STATUS = {408, 429, 500, 502, 503, 504}
IDEMPOTENT_METHODS = {"GET", "HEAD", "OPTIONS", "PUT", "DELETE"}


def _should_retry(method: str, exc: Exception | None, response: httpx.Response | None) -> bool:
    if method.upper() not in IDEMPOTENT_METHODS:
        return False
    if isinstance(exc, (httpx.ConnectError, httpx.ReadTimeout, httpx.WriteTimeout, httpx.PoolTimeout)):
        return True
    if response is not None and response.status_code in RETRYABLE_STATUS:
        return True
    return False


def request_with_retries(
    client: httpx.Client,
    method: str,
    url: str,
    *,
    max_attempts: int = 4,
    base_delay: float = 0.25,
    max_delay: float = 8.0,
    **kwargs,
) -> httpx.Response:
    last_exc: Exception | None = None
    response: httpx.Response | None = None

    for attempt in range(1, max_attempts + 1):
        response = None
        last_exc = None
        try:
            response = client.request(method, url, **kwargs)
            if not _should_retry(method, None, response):
                return response
        except httpx.HTTPError as exc:
            last_exc = exc
            if not _should_retry(method, exc, None):
                raise

        if attempt == max_attempts:
            break

        # Full jitter: sleep in [0, min(max_delay, base * 2^(attempt-1))]
        ceiling = min(max_delay, base_delay * (2 ** (attempt - 1)))
        sleep_for = random.uniform(0, ceiling)
        time.sleep(sleep_for)

    if last_exc is not None:
        raise last_exc
    assert response is not None
    return response
```

Usage:

```python
with httpx.Client(timeout=httpx.Timeout(5.0, read=10.0)) as client:
    resp = request_with_retries(client, "GET", "https://httpbin.org/status/503")
    print(resp.status_code)
```

Against `httpbin.org/status/503`, you should see multiple attempts and eventually a 503 response (or a raised error if you choose to raise on final failure). Against a healthy endpoint, you get a single round trip.

### Respect `Retry-After`

For 429 and some 503 responses, servers send `Retry-After` as seconds or an HTTP date. Prefer that delay over your backoff ceiling when present:

```python
def retry_after_seconds(response: httpx.Response) -> float | None:
    value = response.headers.get("Retry-After")
    if value is None:
        return None
    try:
        return float(value)
    except ValueError:
        return None  # HTTP-date parsing omitted for brevity
```

Wire that into the sleep calculation when `_should_retry` is true because of status code.

## Idempotency keys for unsafe methods

GET is naturally idempotent. POST often is not. Many APIs (Stripe-style payment APIs being the well-known public example) accept an `Idempotency-Key` header so the server can deduplicate retries of the same logical operation.

Client pattern:

```python
import uuid

def post_once(client: httpx.Client, path: str, json: dict) -> httpx.Response:
    key = str(uuid.uuid4())
    headers = {"Idempotency-Key": key}
    # Only retry if your partner API documents safe replay with this header
    return request_with_retries(
        client,
        "POST",
        path,
        headers=headers,
        json=json,
        max_attempts=3,
    )
```

Important: extend `_should_retry` for POST **only** when you know the server honors idempotency keys (or when the operation is safely replayable by design). Blindly adding POST to `IDEMPOTENT_METHODS` is how duplicate side effects happen.

A safer variant is an explicit allowlist:

```python
def request_with_retries(..., *, allow_non_idempotent: bool = False):
    ...
```

and gate non-idempotent retries on that flag plus a required idempotency header.

## Put it together in a small client class

For application code, wrap policy in one place:

```python
class ApiClient:
    def __init__(self, base_url: str):
        self._client = httpx.Client(
            base_url=base_url,
            timeout=httpx.Timeout(connect=5.0, read=15.0, write=15.0, pool=5.0),
            headers={"User-Agent": "myapp-httpx/1.0"},
        )

    def get_json(self, path: str) -> dict:
        resp = request_with_retries(self._client, "GET", path)
        resp.raise_for_status()
        return resp.json()

    def close(self) -> None:
        self._client.close()

    def __enter__(self) -> "ApiClient":
        return self

    def __exit__(self, *args) -> None:
        self.close()
```

Callers get retries and timeouts without re-implementing them in every module. Log attempt number, status, and sleep duration at WARNING level so on-call engineers can see retry storms during an incident.

## Testing resilience without flaky sleeps

Unit-test the decision function and mock the transport:

```python
import httpx


def test_retries_on_503():
    transport = httpx.MockTransport(
        lambda request: httpx.Response(503 if request.url.path == "/flaky" else 200)
    )
    with httpx.Client(transport=transport, base_url="https://example.test") as client:
        # Replace sleep in tests, or inject a clock; assert attempt count via a counter wrapper
        resp = client.get("/flaky")
        assert resp.status_code == 503
```

For integration tests, prefer recording attempt counts with a custom transport rather than asserting wall-clock delays.


## Connection limits and blast radius

Timeouts and retries protect one request. Connection limits protect the process. httpx uses httpcore pools; set explicit limits so a slow dependency cannot exhaust file descriptors or threads:

```python
limits = httpx.Limits(
    max_connections=100,
    max_keepalive_connections=20,
    keepalive_expiry=30.0,
)
client = httpx.Client(timeout=timeout, limits=limits)
```

Pair this with a modest `max_attempts`. During a total upstream outage, four attempts with backoff across one hundred workers is already a lot of synchronized load—jitter helps, but so does capping concurrency at the caller (semaphores in async code, queue depth in sync workers).

## When to stop and fail

Resilience is not infinite patience. After retries are exhausted, fail fast and surface a clear error to the caller: include the method, path (not secrets), attempt count, and last status or exception type. Let the next layer decide—queue a job, show a user message, or trip a circuit breaker. Libraries such as circuit breakers are optional; the required part is a single place that records exhaustion metrics so you notice elevated error budgets.

## Operational takeaways

1. **Timeouts first.** Retries without timeouts multiply load during brownouts.
2. **Retry only what is safe.** Idempotent methods and documented idempotency keys; never invent safety.
3. **Backoff with jitter.** Synchronized retries create thundering herds.
4. **Honor `Retry-After`.** Servers often know better than your defaults.
5. **Centralize policy.** One client wrapper beats twenty slightly different retry loops.
6. **Observe.** Metrics on retry rate, final failure rate, and latency percentiles tell you when upstream health is degrading.

Resilient HTTP clients are less about clever libraries and more about clear policy: what you wait for, what you replay, and what you refuse to guess about. httpx gives you the primitives—timeouts, transports, and clean request APIs—so that policy can live in a few dozen lines you actually understand and test.

