# 01 - Tune: thread-count sweep

Model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` · host `Windows-AMD64` · llama.cpp `b10488`
CPU: **4 physical · 8 logical** cores · `ngl=99` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 52.0 | 100% |
| 2 | 50.7 | 98% |
| 4 | 50.8 | 98% |
| 8 | 50.5 | 97% |
| 16 | 50.5 | 97% |

**Best**: `-t 1` at 52.0 tok/s
**Slowest tested**: `-t 8` at 50.5 tok/s (1.03x spread)
**Against the physical-core default** (`-t 4`, 50.8 tok/s): 1.02x

Use this in your run:

```bash
LAB_N_THREADS=1 make bench
```

## Your explanation

The curve is effectively flat and reaches its knee at 1 thread: `-t 1` achieved
52.0 tok/s, while the 4-physical-core default achieved 50.8 tok/s and even 16
threads only fell to 50.5 tok/s. Therefore, choosing 1 thread gives only a 1.02x
speedup over the default, so the small difference may partly be run-to-run noise.
This run used `ngl=99`, which offloaded the model layers to the GPU; decode was
therefore constrained mainly by GPU memory movement and execution rather than CPU
core count. Extra CPU threads could not add useful decode throughput and instead
introduced minor scheduling and synchronization overhead while sharing the same
memory and GPU resources.
