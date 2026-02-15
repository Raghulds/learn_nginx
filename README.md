# traffic experimentation with nginx 
NGINX, LB

## About NGINX
Request flow:
* Connection acceptance - A worker process accepts the conn. One event loop can handle thousands of connections
* Request parsing - NGINX parses HTTP method, path, headers, body
* Decision layer - Brain of NGINX
    request
    ├─ server block (host)
    ├─ location block (path match)
    ├─ rewrite / return / deny ?
    ├─ proxy_pass ?
* Upstream selection - NGINX chooses which backend. Load balancing by policies. NGINX doesn't know, 
    how expensive the request is, 
    how slow the DB queries are, 
    how close a backend is to collapse. 
It only knows connection count, failures, timeouts
* Proxying
* Response handling

Then, connections may be reused or closed based on the policy 

## Internal architecture
Master process - Reads config, Spawns workers, Handles reloads (Zero downtime)

Worker processes - Handles all traffic, event driven, single threaded, non-blocking IO
Worker saturation happens when slow requests occupy shared event-loop time

This is how it scales.
If one request blocks, it hurts everything on that worker
Thats why: slow upstreams, no timeouts, big buffers destroy tail latency

Where does request wait in NGINX?
- Connection queue
- Waiting inside worker (event loop)
- Waiting for upstream slot
- Waiting on buffers

What resource is shared in NGINX?
- Worker event loop
- Upstream connections
- Buffers & memory
- Retries budget

What happens when one part slows?
    Upstream responses slow
    Workers hold sockets longer 
    Fewer free upstream connections
    New requests wait longer
    Clients retry
    Traffic multiplies
    Collapse
In the mean time, connections still succeed, health checks pass, no errors

Who notices first?
Slow backend: Client sees latency first, NGINX believes system is healthy
Timeouts: Client sees errors faster, backend protected
No timeouts: Client hangs as worker stays occupied with slower APIs, system degrades slowly

Isolation patterns
* Separate upstreams
* Separate listen ports
* Separate NGINX instances

Tail latency: 
Tail-latency-optimized systems are intentionally unfair

## Traffic handling
AFTER WARMUP:
wrk talked directly to the app
async behavior hid contention

Node - DIRECT
wrk -L -t4 -c200 -d60s http://localhost:3000/ping
Running 1m test @ http://localhost:3000/ping
  4 threads and 200 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    12.81ms    4.39ms  42.70ms   58.59%
    Req/Sec     3.92k   118.81     4.22k    85.96%
  Latency Distribution
     50%   12.82ms
     75%   16.55ms
     90%   18.82ms
     99%   20.61ms
  936639 requests in 1.00m, 217.80MB read
  Non-2xx or 3xx responses: 1877
Requests/sec:  15608.07
Transfer/sec:      3.63MB

Go - DIRECT
wrk -L -t4 -c200 -d60s http://localhost:8081/ping
Running 1m test @ http://localhost:8081/ping
  4 threads and 200 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    26.40ms   45.22ms 318.72ms   90.96%
    Req/Sec     3.52k   200.20     4.17k    70.58%
  Latency Distribution
     50%   13.22ms
     75%   17.41ms
     90%   55.98ms
     99%  237.99ms
  841708 requests in 1.00m, 205.19MB read
  Non-2xx or 3xx responses: 1696
Requests/sec:  14025.81
Transfer/sec:      3.42MB

### NGINX without tuning
NGINX introduces:
* finite worker processes
* finite worker connections
* finite upstream sockets
* kernel accept queues
* TCP backpressure

Node - dumb NGINX
wrk -L -t4 -c200 -d60s http://localhost:8082/ping
Running 1m test @ http://localhost:8082/ping
  4 threads and 200 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    62.53ms   25.39ms 363.69ms   79.04%
    Req/Sec   808.97    538.57     3.84k    95.19%
  Latency Distribution
     50%   67.04ms
     75%   73.32ms
     90%   80.56ms
     99%  129.91ms
  193350 requests in 1.00m, 55.71MB read
  Socket errors: connect 6, read 0, write 0, timeout 0
  Non-2xx or 3xx responses: 161100
Requests/sec:   3217.31
Transfer/sec:      0.93MB

Go - dumb NGINX
wrk -L -t4 -c200 -d60s http://localhost:8082/ping
Running 1m test @ http://localhost:8082/ping
  4 threads and 200 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    57.82ms   24.60ms 426.83ms   78.51%
    Req/Sec     0.85k   530.26     3.80k    95.10%
  Latency Distribution
     50%   64.49ms
     75%   66.11ms
     90%   73.20ms
     99%  121.11ms
  203159 requests in 1.00m, 60.45MB read
  Socket errors: connect 182, read 0, write 0, timeout 0
  Non-2xx or 3xx responses: 170762
Requests/sec:   3380.42
Transfer/sec:      1.01MB

Latency went to 5X
RPS dropped by 5X
Non 2xxs/3xxs:
Not app errors. They are:
* upstream connection failures
* accept queue saturation
* backpressure surfacing
* requests never reaching the app

Throughput ≈ concurrency / latency
Now:
* NGINX becomes the choke point
* async advantages disappear
* everyone waits in line

### Enable upstream keepalive
NGINX reuses TCP connections to your app instead of opening a new one for every request. Opens 64 connections once and reuses
This reduces connection churn
 Node - NGINX
wrk -L -t4 -c200 -d60s http://localhost:8082/ping
Running 1m test @ http://localhost:8082/ping
  4 threads and 200 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    13.29ms    5.52ms 140.57ms   73.00%
    Req/Sec     3.81k   242.04     4.20k    92.29%
  Latency Distribution
     50%   13.13ms
     75%   16.93ms
     90%   19.16ms
     99%   21.97ms
  909538 requests in 1.00m, 210.62MB read
  Non-2xx or 3xx responses: 1877
Requests/sec:  15146.06
Transfer/sec:      3.51MB

Go - NGINX
wrk -L -t4 -c200 -d60s http://localhost:8082/ping
Running 1m test @ http://localhost:8082/ping
  4 threads and 200 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    26.49ms   44.70ms 327.17ms   91.18%
    Req/Sec     3.42k   331.06     4.14k    84.79%
  Latency Distribution
     50%   13.62ms
     75%   18.18ms
     90%   52.29ms
     99%  237.77ms
  815845 requests in 1.00m, 234.65MB read
  Non-2xx or 3xx responses: 1652
Requests/sec:  13590.39
Transfer/sec:      3.91MB

### Handling Transient errors with retries

Node - NGINX keep alive & retry with tight timeouts
wrk -L -t4 -c200 -d60s http://localhost:8082/ping
Running 1m test @ http://localhost:8082/ping
  4 threads and 200 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    15.88ms   14.80ms 241.09ms   94.03%
    Req/Sec     3.72k   210.05     4.17k    84.06%
  Latency Distribution
     50%   13.20ms
     75%   17.21ms
     90%   19.35ms
     99%   99.23ms
  888744 requests in 1.00m, 255.87MB read
  Non-2xx or 3xx responses: 10743
Requests/sec:  14809.45
Transfer/sec:      4.26MB

Go -  NGINX keep alive & retry with tight timeouts
wrk -L -t4 -c200 -d60s http://localhost:8082/ping
Running 1m test @ http://localhost:8082/ping
  4 threads and 200 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    15.80ms   14.23ms 135.89ms   94.13%
    Req/Sec     3.70k   239.15     4.12k    87.21%
  Latency Distribution
     50%   13.23ms
     75%   17.23ms
     90%   19.41ms
     99%   87.12ms
  884256 requests in 1.00m, 254.56MB read
  Non-2xx or 3xx responses: 10560
Requests/sec:  14733.94
Transfer/sec:      4.24MB

Go p99 has big drop! 
Because timeout caused the long running requests to retry

### Load balancing
Round robin vs Least conn
Latency = waiting time + service time
Users don’t care about fairness of work — they care about tail latency.

proxy_buffering off - REMOVED!

CDNs don’t use Round robin. 
Least connections - Least conn considers more connections = more load. If you add more CPU burn to fast backend, still connections are served to the fast backend

#### One service, different costs

MacOS command: lsof -iTCP:8081 -sTCP:ESTABLISHED | wc -l
* Gives out a number specifying number of active connections
Will be over 64
Shared pool contention

wrk -t4 -c50 -d30s http://localhost:8082/fast & wrk -t2 -c10 -d30s http://localhost:8082/slow 
[1] 45878 Running 30s test @ http://localhost:8082/slow 2 threads and 10 connections Running 30s test @ http://localhost:8082/fast 4 threads and 50 connections Thread Stats Avg Stdev Max +/- Stdev Latency 500.86ms 0.93ms 507.93ms 95.59% Req/Sec 9.16 1.16 20.00 94.96% 590 requests in 30.05s, 96.22KB read Requests/sec: 19.64 Transfer/sec: 3.20KB raghul.s@Raghuls-MacBook-Pro ~ % Thread Stats Avg Stdev Max +/- Stdev Latency 5.99ms 700.69us 23.51ms 92.54% Req/Sec 2.01k 84.16 2.27k 85.50% 240541 requests in 30.07s, 38.31MB read Requests/sec: 7999.73 Transfer/sec: 1.27MB [1] + done wrk -t4 -c50 -d30s http://localhost:8082/fast

Observations:
/slow
Latency ~500ms
RPS ~20

/fast
Latency ~6ms
p99 ~23ms
RPS ~8000
-------------------
Single backend + single upstream creates shared contention at 4 layers:

Clients -> NGINX worker -> Request queue (per worker) -> Upstream connection pool[slow req sit here] -> Backend threads / event loop

Slow requests:
Hold upstream connections
Hold backend workers
Increase queue wait time
Inflate p99 for unrelated requests

Fast requests:
Are fast only after they get a slot
Spend most of their time waiting

So yes — queueing is the enemy, not slowness by itself.

Isolation - Same binary, diff nginx upstreams, diff deployments, diff autoscaling rules
* Separate queues
* Separate pools
* Separate saturation domains
Isolation is also about avoiding thread exhaustion & failure containment, capacity planning, backpressure correctness (Aggressive timeouts on /slow, tight limit_conn on /fast, retries only on /fast)  

### With Buffering on 
NGINX must,
* Read the entire request body
* Store it in:
    memory buffers (and possibly temp files, but memory first)
    Only then forward to backend

With buffering off, NGINX reads the chunk and immediately forwards it. Doesn't store.

Run lua script with wrk to simulate large request body,
    wrk -t2 -c20 -d30s -s large_body.lua http://localhost:8083/buffered  

Watch live memory usage with command,
    top -pid WORKER-PROCESS-ID


# Takeaways
Slow downstream → Resource occupancy increases → Upstream overload
Place backpressure at the edge; Decide when to reject traffic
Always retry for idempotent ops only when failure is transient with jitter + exp backoff and with a global retry budget 

Is latency rising?
    |
    ├── Yes → Is traffic spike?
    |       ├── Yes → Rate limit / autoscale
    |       └── No → Backend slow
    |               ├── Disable retries
    |               ├── Lower timeout
    |               └── Enable breaker
    |
    └── No → Errors high?
            ├── Yes → Backend unhealthy → fail fast
            └── No → Likely resource saturation

