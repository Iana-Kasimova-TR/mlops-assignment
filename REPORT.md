# MLOps HW3 — text-to-SQL agent + observability (report)

## what I built

A text-to-SQL agent with LangGraph. The graph is **generate → execute → verify → revise**,
and it loops back to revise when the verifier is not happy, capped by `MAX_ITERATIONS`.
It talks to **Qwen3-30B-A3B** served by **vLLM**.

I worked in two places: on my Mac against Nebius Token Factory while building the agent,
and on a Nebius **H100** with my own vLLM for the real serving numbers. The agent sits
behind a small FastAPI server (`/answer`). Prometheus + Grafana watch vLLM, and Langfuse
traces the agent.

## serving config and why

I run vLLM from the official docker image (`vllm/vllm-openai:v0.22.1`) so I don't have to
build it myself. The flags:

- `--max-model-len 8192` — this is the important one. Without it vLLM tries to reserve KV
  cache for Qwen's full 262144 context and the container crashes on boot. 8192 is more than
  enough for my prompts (schema + question).
- `--gpu-memory-utilization 0.90` — let vLLM use most of the H100 so there is more room for
  KV cache.
- `--gpus all`, `--ipc=host` — give the container the GPU and enough shared memory.
- prefix caching is on by default in this version, and it turned out to matter a lot (below).

Agent side:
- `uvicorn --workers 3` — I explain how I landed on 3 in the load test part.
- `temperature=0`, `max_tokens=256` — a SQL query is short, no reason to let the model ramble.
- `MAX_ITERATIONS=2`.

## baseline

First eval on the H100, 30 questions, **execution accuracy** (compare the rows my SQL returns
to the rows the gold SQL returns):

- overall **30%**
- per iteration: **[26.7%, 33.3%, 30%]**

So the loop helped on the second attempt (26.7 → 33.3) and then actually *hurt* on the third
(33.3 → 30). That last drop is the verifier being too strict — it rejected answers that were
already correct and the revise step made them worse.

I opened the **Langfuse traces** to see why, and found two patterns:

1. **duplicates** — a question asking for circuit coordinates returned 11 identical rows
   because a JOIN fanned out. Missing DISTINCT.
2. **verifier too strict** — on one question the average fastest lap came out as 1.0 and the
   verifier rejected it with "implausible, F1 lap times are 80–90 seconds". But that is the
   verifier using its own world knowledge instead of trusting the data. That kind of rejection
   triggers a revise that usually makes things worse.

## load test and the SLO

SLO: **p95 end-to-end latency under 5s at 10 RPS.** The first run was a disaster, and then it
got a lot better in steps:

| stage | p50 | p95 | p99 | timeouts | conn refused | 500s | ok / 3000 |
|---|---|---|---|---|---|---|---|
| original (1 process, iter 3) | 31.1s | **96.8s** | 116.8s | 1446 | 941 | 237 | 376 |
| + workers, iter 2, max_tokens | 2.94s | 16.5s | 23.3s | 0 | 0 | 379 | 2621 |
| workers = 3 (sweet spot) | 1.59s | 9.3s | 13.0s | 1 | 1 | 381 | 2617 |
| + schema-first prompts | **0.89s** | **2.66s** | 6.4s | 0 | 13 | 381 | 2606 |

How I diagnosed it, step by step, using the Grafana dashboard:

- **First run:** p95 96.8s, 1446 timeouts, 941 connection refused. But the dashboard showed
  vLLM was basically idle — queue 0, KV cache ~40%, TTFT ~230ms. So vLLM was not the problem,
  it was *starving*. (My first instinct was actually to go tune vLLM flags like `max-num-seqs` — the dashboard is the only reason I didn't waste time on that, it clearly showed vLLM sitting idle.) The bottleneck was my agent server: it's a sync endpoint, FastAPI runs
  those in a ~40 thread pool, and each request makes 2–3 sequential vLLM calls, so it could
  only handle ~40 requests at once. At 10 RPS that's nowhere near enough, so everything piled
  up at the door (that's the timeouts and the connection-refused).
- **Fix 1 — more workers.** The agent has no model in it (the model is in the vLLM container),
  so extra worker processes are cheap. I also cut the work *per request* — dropped `MAX_ITERATIONS` 3 → 2 (fewer LLM calls per
  question), capped `max_tokens` at 256, and made the generator output only the SQL instead of
  "think step by step" prose. All of these shrink how many tokens get generated, and generating
  tokens (decode) is the slow phase, so they help latency directly. p95 went 96.8 → 16.5s and timeouts/connection errors went to 0. Now the
  dashboard flipped: KV cache hit **100%** and the queue spiked to ~87. The bottleneck moved
  from my server to the GPU — which is exactly where you want it. (Honest caveat: I changed three
  things in this one step — more workers, `MAX_ITERATIONS` 3→2, and `max_tokens` — so I can't
  cleanly say how much each one helped on its own. Workers was clearly the big one though, since
  that's what took the timeouts and connection-refused to zero.)
- **Fix 2 — find the worker sweet spot.** I had gone to 4 workers, but at KV 100% the GPU was
  over-fed and started preempting, which adds latency. So I swept the worker count down. **3
  workers** gave KV ~40% and p95 9.3s. Busy GPU, but not thrashing. (I did think about turning
  on FP8 KV cache for more memory headroom, but once the worker sweep brought KV down to ~40% I
  didn't need it — no point adding an accuracy risk to fix a problem that was already gone. So KV
  cache was a lever I *read* on the dashboard to make the worker decision, and deliberately chose
  not to pull.)
- **Fix 3 — schema-first prompts (the big one).** My prompt had the *question* first and the
  *schema* after. The schema is identical for every question on the same database, but because
  the changing question came first, vLLM couldn't cache the schema as a prefix and re-processed
  it on every single call. I flipped the order — **schema first, question last** — so the
  schema becomes a stable prefix the cache can reuse. p95 went 9.3 → **2.66s**. That's the
  change that put me under the SLO. A prompt-ordering change that paid off at the serving layer. One caveat on this win though: it
  leans on a lot of my questions hitting the same database, so the cached schema gets reused
  across them. On a more spread-out workload (a different DB every request) prefix caching would
  help much less — so my p95 partly reflects the shape of the test traffic, not only the fix.

Final: **p95 2.66s, p50 0.89s, 0 timeouts at 10 RPS — SLO met.** (p99 is 6.4s, just over 5s,
but the target is on p95.)

## the errors I did not fully fix

Through all the runs there were ~**381 HTTP 500s** (about 12.7%), and they did not move no
matter what I changed — same at 4 workers and at 3 workers. But when I ran the eval, which goes
one question at a time with no concurrency, there were **zero failures** over all 30 questions.
So these errors only show up under concurrent load, not on specific questions. My guess is it's
something on the concurrency side (the agent's HTTP client connection pool, or vLLM under
sustained load) rather than a bad question. I ran out of time to pin it down exactly — it's
first on the "more time" list.

## quality tuning and the second eval

After the trace analysis I changed the prompts:

- **generate:** ask for only the SQL, drop "think step by step" (less to generate = faster and
  cleaner to parse), and a stronger rule about avoiding duplicate rows from JOINs.
- **verify:** made it lenient. Only reject on a concrete SQL mistake (errored, wrong columns,
  obvious duplicates). Explicitly told it *not* to reject just because a number looks
  surprising — trust the data, default to ok.
- **skip verify on a SQL error:** if the query didn't even run, there's no point spending an
  LLM call to "verify" it — go straight to revise with the DB error.
- **schema-first ordering** (the same change that helped latency).

Second eval, same 30 questions:

- overall **36.7%** (up from 30%)
- per iteration: **[36.7%, 36.7%]**

(note: the full per-question file for this last run was on the H100 and I lost it when the
VM got torn down — I kept the summary above, and the baseline plus an earlier prompt-tuning
round (33.3%) are saved in full in `results/`.)

Two things to read here. First, the first-attempt accuracy went from 26.7% to 36.7% — that's
the better generation prompt working. Second, the two iterations are *identical*, which means
the revise loop added nothing this time. With the lenient verifier it rarely says "not ok", so
revise rarely fires, and when it did fire it didn't turn a wrong answer into a right one on
this set.

## what value does the agent loop actually add

On my eval, the revise loop **did not improve accuracy** — the number is the
same at iteration 1 and 2. In the strict-verifier baseline the loop even *hurt* (it revised
correct answers into wrong ones). What actually moved accuracy was a **better first attempt**
(the generation prompt) and a verifier that **stops breaking correct answers**.

The verify step still has some value — it catches execution errors and gives a signal you could
log or surface to a user ("I'm not confident about this one") — but the self-correction part
didn't pay off here. If I deployed this with latency in mind I'd set `MAX_ITERATIONS=1`, because
the second call only adds latency without buying accuracy. Maybe the prompt for verifier should be tuned further so that to give the profit from this self reflection loop

## what I'd do with more time

- **Pin down the 381 concurrency errors** — reproduce under load with logging turned on in the
  except block, and check the HTTP client connection pool and the vLLM logs.
- **Give the agent a tool to look up real distinct values of a column.** A lot of failures were
  casing/value mismatches (e.g. `'M'` vs `'m'`, or a made-up category name). If the agent could
  sample the actual values it would stop guessing.
- **A better verifier, or schema-linking** so on big databases it only looks at the relevant
  tables.
- **Set `MAX_ITERATIONS=1`** in a latency-sensitive deployment, since revise wasn't helping —
  that would also pull p99 under 5s.
- **For higher RPS, run more vLLM replicas behind a load balancer.** One H100 has a ceiling, and
  a 2-call agent at 10 RPS is already close to it.
