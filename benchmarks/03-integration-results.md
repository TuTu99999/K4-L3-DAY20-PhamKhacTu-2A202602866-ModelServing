# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.4 | 8120.9 | 8121.3 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.1 | 6611.9 | 6612.1 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.1 | 7117.2 | 7117.3 |

Mean per stage (ms): embed **0.0** · retrieve **0.2** ·
llm **7283.3** · total **7283.6**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Goodput@SLO counts only the requests per second that met the TTFT and TPOT targets. Throughput at saturation ignores SLOs.

**What problem does PagedAttention actually solve?**

> PagedAttention stores the KV cache in non-contiguous pages, removing the internal fragmentation that wasted most GPU memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps because prefill is compute-bound and decode is memory-bandwidth-bound.


## Which N16-N19 pieces are real

- **N16 Cloud/IaC: stubbed.** The pipeline and serving endpoint run on localhost,
  not on a Kubernetes cluster or Compose deployment.
- **N17 Data pipeline: stubbed.** Documents come from an in-memory Python list,
  without an Airflow DAG or batch ingestion job.
- **N18 Lakehouse: stubbed.** `TOY_DOCS` stands in for a Delta or Iceberg table.
- **N19 Vector and feature layer: stubbed.** Retrieval uses keyword overlap rather
  than a vector index, embedding service, or Feast feature view.

The N20 serving component is real: all three queries called the local
OpenAI-compatible `llama-server` and returned answers with server timings. The LLM
was the dominant stage at 7283.3 ms out of 7283.6 ms on average (approximately
100%), which is expected because embedding was disabled and keyword retrieval took
only 0.2 ms. To halve end-to-end latency, I would attack the LLM stage, especially
decode, by reducing the output-token budget and testing faster accelerator/runtime
settings. Optimizing retrieval cannot materially help while it accounts for less
than one millisecond, and the measured Q2 model is not the first choice because it
decoded more slowly than Q4 on this machine.
