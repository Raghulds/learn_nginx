
## Rate limit
NGINX uses a Leaky Bucket–style algorithm. But with burst tolerance, so it looks like token bucket from the outside.
Why: NGINX internally tracks request timestamps and calculates excess
excess = previous_excess + (1 - time_elapsed * rate)
If excess > burst → reject
If excess <= burst:
* nodelay → allow immediately
* no nodelay → delay request
That’s classic leaky bucket behavior.

#### Usage
| Directive        | Context                      |
| ---------------- | ---------------------------- |
| `limit_req_zone` | `http` only                  |
| `limit_req`      | `http`, `server`, `location` |

With,
limit_req zone=api burst=20 nodelay;
| Scenario                    | Result            |
| --------------------------- | ----------------- |
| ≤10 r/s                     | Always allowed    |
| Short spike ≤20             | Allowed instantly |
| Sustained >10 r/s           | Rejected          |
| Sustained without `nodelay` | Delayed           |


Tuning:
| Knob                        | Effect        |
| --------------------------- | ------------- |
| `rate`                      | Leak speed    |
| `burst`                     | Queue depth   |
| `nodelay`                   | Drop vs delay |
| Key (`$binary_remote_addr`) | Fairness      |

#### Observations
wrk -t2 -c10 -d30s http://localhost:8081/fast
Reasons you may see no throttling:
NGINX rate is per second
wrk requests are spread across time



#### Best practice
Client
  ↓
NGINX: coarse IP throttling (DoS protection)
  ↓
App Service: user / token based rate limit
  ↓
Downstream protection (DB, cache, etc.)

nodelay controls latency vs rejection
timeouts control damage radius
retries control amplification