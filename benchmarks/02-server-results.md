# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=4` ·
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 29 | 0.50 | 16000 | 24000 | 24000 | 8.4 | 0.0% |
| 50 | 31 | 0.53 | 30000 | 53000 | 57000 | 16.0 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **1.06x** (21% of linear) |
| P95 latency | **2.21x** |
| Effective concurrency at 50 users | 16.0 vs `--parallel 4` slots (occupancy/slot ratio 4.01) |

**Saturated.** Throughput delivered only 1.06x for 5x the offered load, and effective concurrency (16.0) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 1.06x while P95 moved 2.21x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Your reading

The server is already close to saturation at 10 users and is clearly saturated
before 50 users. Increasing offered load by 5x raised delivered throughput only
from 0.50 to 0.53 RPS (1.06x, or 21% of linear scaling), while P95 increased from
24 s to 53 s (2.21x). At 50 users, effective concurrency was 16.0 against only four
decode slots; together with the earlier peak of four processing and 46 deferred
requests, this shows that most additional latency is queue time rather than useful
compute.

I would use a 30-second latency SLO. At 10 users, at least about 95% of requests met
it because P95 was 24 s, giving approximately 0.48 good requests/s. At 50 users,
the median itself was 30 s, so only about half of the 0.53 RPS was within the SLO,
or roughly 0.27 good requests/s. My first experiment would be increasing
`--parallel` above 4 while keeping enough context per slot, then re-measuring RPS
and P95. This directly targets the observed slot queue; changing CPU threads is
unlikely to help because the thread sweep was flat, and switching to Q2 is also
unlikely to help because Q2 decoded more slowly than Q4 on this machine.
