
### Retries
NGINX retries are failover retries, not replays
Retries are meant to handle transient failures:
* brief network blips
* momentary overload
* pod restarts
* GC pauses
* cold starts

#### nginx setup
proxy_next_upstream
proxy_next_upstream_tries
Ex,
proxy_next_upstream error timeout http_502 http_503 http_504;
proxy_next_upstream_tries 3;

Simulation of failure using,
timeout in frontend -> proxy_send_timeout - 1s

Endpoint - /random-fails
Golang service as slow backend - Delay simulation by 1.5s
Node service as fast backend

Roundrobin routing as default

Added request_id header to the incoming requests in the frontend - to trace retries

Command to get all logs with , (comma). This shows all retries
grep 'ustatus=.*,' /opt/homebrew/var/log/nginx/upstream.log

Logs:
07/Feb/2026:21:29:49 +0530 rid=cf8aa30f66df1ffe6dc021ffd72e6cfd status=200 uaddr=127.0.0.1:8081, 127.0.0.1:3000 ustatus=504, 200 urt=1.003, 0.001 rt=1.003
07/Feb/2026:21:29:49 +0530 rid=002a226c6f62519874cce4a35bd6fdfe status=200 uaddr=127.0.0.1:8081, 127.0.0.1:3000 ustatus=504, 200 urt=1.007, 0.001 rt=1.007
07/Feb/2026:21:29:49 +0530 rid=15ab42708a50449a3fe5ffbea44f3b5d status=200 uaddr=127.0.0.1:8081, 127.0.0.1:3000 ustatus=504, 200 urt=1.007, 0.001 rt=1.008
07/Feb/2026:21:29:49 +0530 rid=cc9a02ad782b77fd3bf1937a11996d80 status=200 uaddr=127.0.0.1:8081, 127.0.0.1:3000 ustatus=504, 200 urt=1.007, 0.001 rt=1.008
07/Feb/2026:21:29:49 +0530 rid=7cdf00a63b0fffcfedce4ca5ebc533d0 status=200 uaddr=127.0.0.1:8081, 127.0.0.1:3000 ustatus=504, 200 urt=1.007, 0.001 rt=1.008
07/Feb/2026:21:29:49 +0530 rid=f77cf9262849cf5b52fa226958edf8a8 status=200 uaddr=127.0.0.1:8081, 127.0.0.1:3000 ustatus=504, 200 urt=1.007, 0.001 rt=1.008
07/Feb/2026:21:29:49 +0530 rid=303c84f6683ac47e2500c071ed588356 status=200 uaddr=127.0.0.1:8081, 127.0.0.1:3000 ustatus=504, 200 urt=1.007, 0.001 rt=1.008
07/Feb/2026:21:29:49 +0530 rid=f290e9cd9442580a5c1661bc2bf67fbd status=200 uaddr=127.0.0.1:8081, 127.0.0.1:3000 ustatus=504, 200 urt=1.007, 0.001 rt=1.008
07/Feb/2026:21:29:49 +0530 rid=be7ebdbc8cfca401b0505439c0268532 status=200 uaddr=127.0.0.1:8081, 127.0.0.1:3000 ustatus=504, 200 urt=1.007, 0.001 rt=1.008
07/Feb/2026:21:29:49 +0530 rid=e4e76d3948bb753ade90c2aec27699d6 status=200 uaddr=127.0.0.1:8081, 127.0.0.1:3000 ustatus=504, 200 urt=1.007, 0.001 rt=1.008
07/Feb/2026:21:29:49 +0530 rid=cbbe586d31b87852d53c77f98110e4a4 status=200 uaddr=127.0.0.1:8081, 127.0.0.1:3000 ustatus=504, 200 urt=1.007, 0.001 rt=1.008
07/Feb/2026:21:29:49 +0530 rid=941c049e031d96e62ccc874246a49ee8 status=200 uaddr=127.0.0.1:8081, 127.0.0.1:3000 ustatus=504, 200 urt=1.007, 0.001 rt=1.008
07/Feb/2026:21:29:49 +0530 rid=04d182b0347b6ef78b08afe289c209ab status=200 uaddr=127.0.0.1:8081, 127.0.0.1:3000 ustatus=504, 200 urt=1.007, 0.001 rt=1.008

#### Retry amplification
retry amplification factor =
(total backend attempts) / (client requests)
1 client request → 2 backend executions
amplification = 2×

* increases load
* hides errors
* inflates latency
* does NOT create new failures by itself

### Retry Storm (self DDOS)

Each request internally,
Go times out (1s)
→ retry to Node
→ Node times out
→ retry again
→ both are now overloaded

Client experience
- some 200s
- many slow responses
- rising 504s

-> Leads to failure amplification
Retries everywhere → storm

#### Circuit breaker
Stopping the storm

upstream random_fails_backend {
    server 127.0.0.1:8081 max_fails=2 fail_timeout=15s;
    server 127.0.0.1:3000 
}

- 2 failures → backend ejected
- stays out for 15s
- no retries to it during that time

NGINX does
* Fail fast
* Evict bad backends
* Half open probing (Implicit)

NGINX doesn't
* Rolling error rates
* Smart budgets
* Adaptive backoff

#### Rate limit (cap traffic) X Retries (increase traffic)

Naive retry -> Retry storm/Self DDOS

##### Smart retry 
- Client gets 429
- Reads Retry-After header
- Applies backoff + jitter
- Retry volume is smaller and spread out
- System stabilizes instead of collapsing

| Signal        | Meaning                 | Retry Action                       |
| ------------- | ----------------------- | ---------------------------------- |
| `429`         | You’re sending too fast | Retry **later**, not immediately   |
| `503`         | Server overloaded       | Retry with **exponential backoff** |
| Timeout       | Maybe transient         | Retry with limit                   |
| `4xx` (other) | Client error            | **Never retry**                    |

In NGINX, rate limit controls ingress pressure and retries controls egress pressure

Upstream Retry considered as a new request. They consume rate limit budget

* Retry budget - Only X% of traffic is allowed to be retries
* Backoff + jitter is mandatory
* Rate-limit aware retries

Retries and rate limits are tightly coupled.
Rate limits protect the system, but retries can bypass that protection unless we enforce retry budgets, backoff, and 429-aware behavior. Otherwise retries turn partial failures into full outages
