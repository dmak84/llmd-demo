# Architecture Design

This document compares the architecture design of the following LLM-D configuration

**gpt-oss-20b (single pool)**

https://github.com/redhat-na-ssa/demo-ocp-llm-d/blob/main/gitops/instance/llm-d/gpt-oss-20b/llm-infra.yaml

**qwen-pd (prefill/decode split)**

https://github.com/redhat-na-ssa/demo-ocp-llm-d/blob/main/gitops/instance/llm-d/pd-disaggregation/pd-deployment.yaml

### gpt-oss-20b (single pool)

Client → Router → [Single LLM Pod] → Response

### qwen-pd (prefill/decode split)

Client → Router → Prefill Pool (prompt encoding)
↓ (KV cache via RDMA)
Decode Pool (token generation)
↓
Response


| Aspect                | `gpt-oss-20b`YAML                                                 | `qwen-pd`YAML                                                                |
| --------------------- | ----------------------------------------------------------------- | ---------------------------------------------------------------------------- |
| **Architecture Type** | **Single pool**(monolithic inference)                             | Prefill/Decode (PD) separated(two-stage pipeline)                            |
| **Purpose**           | Simpler LLM serving — one pod does all inference work.           | Optimized for**parallelism and KV cache reuse**across two specialized pools. |
| **Result**            | Easier to deploy but slower for long context or high concurrency. | More complex, but scales better for multi-request or large-token workloads.  |

**Explanation:**

* The **single-pool setup** (gpt-oss-20b) means every request is processed entirely by one model server (prefill + decode).
* The **PD-separated setup** (qwen-pd) splits these stages:
  * *Prefill pool*: Handles initial prompt embeddings (context).
  * *Decode pool*: Handles token-by-token generation.

This PD split allows **KV cache transfer** between stages, improving throughput on large models.

# Router Scheduler Config (`EndpointPickerConfig`)


| Component               | `gpt-oss-20b`                                                                                      | `qwen-pd`                                                                                      |
| ----------------------- | -------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| **Plugin Types**        | `single-profile-handler`,`max-score-picker`,`queue-scorer`,`kv-cache-scorer`,`prefix-cache-scorer` | `pd-profile-handler`,`prefill-header-handler`,`prefill-filter`,`decode-filter`,`random-picker` |
| **Purpose**             | Focus on**cache-aware routing**(prefix/kv scoring for token reuse).                                | Focus on**stage separation and routing** between prefill and decode pools.                 |
| **Scheduling Profiles** | Single`"default"`profile (all handled together).                                                   | Two profiles:`"prefill"`and`"decode"`.                                                         |

**Explanation:**

* `gpt-oss-20b` router tries to route requests to servers with **matching prefixes or cached KV states** — optimizing for *latency and cache reuse* within a single model.
* `qwen-pd` uses a **PD pipeline**, where the router first decides if a request is prefill or decode and routes accordingly.

# Model Deployment Resource


| Aspect             | `gpt-oss-20b`            | `qwen-pd`                                   |
| ------------------ | ------------------------ | ------------------------------------------- |
| **CPU/Memory**     | 1 CPU / 8 Gi             | 4 CPU / 32 Gi                               |
| **GPU**            | 1 GPU                    | 1 GPU (per pool)                            |
| **Extra Resource** | None                     | `rdma/roce_gdr`(for**RDMA over RoCE**)      |
| **Volume**         | Uses PVC (`gpt-oss-20b`) | None                                        |
| **Networking**     | Normal TCP/IP            | **RDMA network**for low-latency KV transfer |

**Explanation:**

* `qwen-pd` leverages **RDMA (Remote Direct Memory Access)** to move KV cache data between prefill and decode pods *without CPU involvement* — drastically improving speed for distributed inference.
* `gpt-oss-20b` runs entirely in one pod — simpler, but lacks distributed acceleration.

# Environment Variable


| Variable               | `gpt-oss-20b`                     | `qwen-pd`                                            |
| ---------------------- | --------------------------------- | ---------------------------------------------------- |
| `VLLM_ADDITIONAL_ARGS` | Only disables uvicorn access log. | Enables RDMA KV cache transfer using`NixlConnector`. |
| `KSERVE_INFER_ROCE`    | —                                | `"true"`(activates RDMA mode).                       |
| `UCX_*`                | —                                | UCX transport settings for RDMA communication.       |

**Explanation:**
The `qwen-pd` setup uses **UCX** (Unified Communication X) — a high-performance library used in HPC and distributed AI inference — to facilitate fast data transfer.

# LLM Serving Strategy Summary


| Strategy                      | Description                                          | Pros                                                                            | Cons                                                |
| ----------------------------- | ---------------------------------------------------- | ------------------------------------------------------------------------------- | --------------------------------------------------- |
| **Single-pool (gpt-oss-20b)** | Each pod handles full requests.                      | Simpler, fewer moving parts.                                                    | Lower throughput for long or parallel prompts.      |
| **PD-split (qwen-pd)**        | Dedicated prefill and decode pods exchange KV cache. | Higher concurrency, faster generation for long sequences.When to Use Each<br /> | Requires RDMA networking and complex orchestration. |

# When to Use Each


| Use Case                                                              | Choose        |
| --------------------------------------------------------------------- | ------------- |
| Simple LLM inference (dev/test, small models)                         | gpt-oss-20b`  |
| High-performance inference (long prompts, high QPS, distributed GPUs) | qwen-pd`      |
| Non-RDMA environment                                                  | `gpt-oss-20b` |
| RDMA-enabled HPC cluster (NVIDIA + InfiniBand/RoCE)                   | `qwen-pd`     |

# Plugin Types Overview

The **`plugins` section** inside the `EndpointPickerConfig` is the *brain* of KServe’s LLM router scheduler.
Each plugin type contributes a different logic for **how incoming inference requests are classified, scored, filtered, and routed** to backend inference pods (endpoints).

In `KServe LLMInferenceService`, the router scheduler defines:

```
apiVersion: inference.networking.x-k8s.io/v1alpha1
kind: EndpointPickerConfig
plugins:
  - type: <plugin-type>
    parameters: ...
schedulingProfiles:
  - name: <profile>
    plugins:
      - pluginRef: <plugin-type>
```

These plugins fall into four functional groups:


| Category | Function                                       |
| -------- | ---------------------------------------------- |
| Handler  | Classify or pre-process the request.           |
| Filter   | Select which endpoints are eligible.           |
| Scorer   | Score eligible endpoints to decide best match. |
| Picker   | Choose the final endpoint among candidates.    |

## **Handler Plugins**

Handlers interpret or classify the request context before any scoring or filtering happens.

## `single-profile-handler`

**Purpose:**
Simplest handler — routes all requests through a single scheduling profile (`default`).

**Used in:**`gpt-oss-20b`

**Function:**

* Doesn’t differentiate between request types.
* Suitable for *single-stage* inference (no prefill/decode separation).

**Analogy:** “Every request goes through the same queue.”

## `pd-profile-handler`

**Purpose:**
Separates requests into *prefill* and *decode* phases for **two-stage LLM serving**.

**Used in:**`qwen-pd`

**Parameters:**

```
parameters:
  threshold: 0
```

**threshold:** Controls how to classify requests.

0 → Always separate (every request has prefill + decode).

Higher values → separate only when request size exceeds threshold.

**Effect:**
Routes requests to:

prefill profile → for initial prompt embedding

decode profile → for token-by-token generation

## `prefill-header-handler`

**Purpose:**
Reads HTTP/gRPC headers to detect prefill requests explicitly (when client sets a header flag).

**Used in:**`qwen-pd`

**Example:**
If client sends a request with header `X-Prefill: true`, this handler assigns it to the `prefill` profile.

**Effect:**
Useful for explicit phase control by the client instead of auto-detection.

## Filter Plugin

Filters decide *which endpoints* are even eligible to serve a given request.

### prefill-filter`

**Purpose:**
Selects endpoints belonging to the **prefill pool** only.

**Used in:**`qwen-pd`

**Function:**
When the active profile is `prefill`, this filter ensures only pods in the prefill deployment are candidates.

###decode-filter`

**Purpose:**
Selects endpoints belonging to the **decode pool** only.

**Used in:**`qwen-pd`

**Function:**
Ensures decode-phase requests go only to decode pool pods (which are configured differently, e.g. lower latency, smaller batch sizes).

## Scorer Plugins

Scorers assign numerical scores to candidate endpoints to decide *which one* is best.
Higher scores mean better routing suitability.

### 🔹 `queue-scorer`

**Purpose:**
Scores endpoints based on current queue depth (load).

**Used in:**`gpt-oss-20b`

**Logic:**

* Endpoints with fewer queued requests score higher (preferred).
* Prevents overloading a single pod.

### 🔹 `kv-cache-scorer`

**Purpose:**
Prefers endpoints with **matching KV cache entries** (cached context from previous tokens).

**Used in:**`gpt-oss-20b`

**Function:**

* Looks for partial token history matches in the endpoint’s in-memory KV cache.
* Reduces latency by avoiding recomputation of previously cached attention blocks.

**Result:**
Improves performance for streaming or continuation requests.

### 🔹 `prefix-cache-scorer`

**Purpose:**
More advanced cache scorer — matches longer *prompt prefixes* between new and previous requests.

**Used in:**`gpt-oss-20b`

**Parameters:**

```
parameters:
  hashBlockSize: 64
  maxPrefixBlocksToMatch: 256
  lruCapacityPerServer: 39744

```

**Meaning:**

* `hashBlockSize`: Size of token blocks used for hashing prefix segments.
* `maxPrefixBlocksToMatch`: Max number of blocks to compare (affects token limit).
* `lruCapacityPerServer`: Size of prefix cache LRU per endpoint.

**Effect:**
Allows router to route requests to pods that already computed a similar prompt prefix, leveraging prefix reuse to save compute time.

### 🔹 `max-score-picker`

**Purpose:**
Final decision maker — picks the endpoint with the highest total score.

**Used in:**`gpt-oss-20b`

**Function:**
Takes scores from all scorers and picks the endpoint with the best composite score (after weights).

## Picker Plugins

Pickers are the last step — they actually **choose** which endpoint to send the request to, based on scoring or randomness.

### 🔹 `random-picker`

**Purpose:**
Randomly picks an endpoint among filtered candidates.

**Used in:**`qwen-pd`

**Parameters:**

```
parameters:
  maxNumOfEndpoints: 1

```

* `maxNumOfEndpoints`: Limit how many endpoints to pick from.

**Use Case:**
Used in simpler or low-latency routing profiles where deterministic scoring is unnecessary (e.g., random load balancing among decode pods).

### Scheduling Profiles

Each `schedulingProfile` defines **which plugins** apply to each class of requests:

**Example — gpt-oss-20b**

```
schedulingProfiles:
- name: default
  plugins:
    - pluginRef: prefix-cache-scorer
      weight: 2.0
    - pluginRef: queue-scorer
      weight: 1.0
    - pluginRef: kv-cache-scorer
      weight: 1.0
    - pluginRef: max-score-picker

```

**Meaning:**
All requests go through:

Cache-based scorers (prefix, kv)

Queue load scorer

Pick the endpoint with max weighted score.

**Example — qwen-pd**

```
schedulingProfiles:
- name: prefill
  plugins:
    - pluginRef: prefill-filter
    - pluginRef: random-picker
- name: decode
  plugins:
    - pluginRef: decode-filter
    - pluginRef: random-picker

```

**Meaning:**
Requests first classified as prefill or decode by the handlers, then routed randomly among eligible endpoints in that pool.

`gpt-oss-20b` — *Single Pool, Cache-Aware Routing*

```
                   ┌──────────────────────────┐
                   │ Incoming Inference Req.  │
                   └────────────┬─────────────┘
                                │
                                ▼
              ┌────────────────────────────────┐
              │  Handler: single-profile-handler │
              │  → All requests use "default"    │
              └────────────────────────────────┘
                                │
                                ▼
         ┌────────────────────────────────────────────┐
         │           Scoring Phase (default)          │
         ├────────────────────────────────────────────┤
         │ 1️⃣ prefix-cache-scorer                    │
         │     → Checks for prompt prefix matches     │
         │ 2️⃣ kv-cache-scorer                        │
         │     → Checks for cached KV state reuse     │
         │ 3️⃣ queue-scorer                           │
         │     → Prefers endpoints with shorter queue │
         └────────────────────────────────────────────┘
                                │
                                ▼
       ┌─────────────────────────────────────────────┐
       │ Picker: max-score-picker                   │
       │ → Combines weighted scores, selects best   │
       │   endpoint for the request                 │
       └─────────────────────────────────────────────┘
                                │
                                ▼
           ┌────────────────────────────────┐
           │ Route to best endpoint (Pod)   │
           │ e.g., gpu-node-1 (inference)   │
           └────────────────────────────────┘


```

**Summary:**


Single pool routing → Smart cache scoring → Highest score wins.
Good for smaller clusters or when cache locality matters more than multi-stage optimization.

`qwen-pd` — *Prefill/Decode Split with RDMA Cache Transfer*

```
                   ┌──────────────────────────┐
                   │ Incoming Inference Req.  │
                   └────────────┬─────────────┘
                                │
                                ▼
          ┌─────────────────────────────────────────────┐
          │ Handler Phase                               │
          ├─────────────────────────────────────────────┤
          │ pd-profile-handler                          │
          │   → Classifies as "prefill" or "decode"     │
          │ prefill-header-handler                      │
          │   → (Optional) reads header for phase info  │
          └─────────────────────────────────────────────┘
                                │
                                ▼
          ┌─────────────────────────────────────────────┐
          │ If Prefill Profile                         │
          ├─────────────────────────────────────────────┤
          │  prefill-filter → keeps only prefill pods   │
          │  random-picker → randomly pick one          │
          │  (Pod in Prefill Pool)                      │
          └─────────────────────────────────────────────┘
                                │
                                ▼
                     ┌────────────────────┐
                     │ Prefill Pod        │
                     │ - Builds prompt    │
                     │ - Generates KV cache│
                     └─────────┬───────────┘
                               │
                      (KV cache via RDMA)
                               │
                               ▼
          ┌─────────────────────────────────────────────┐
          │ Decode Profile                              │
          ├─────────────────────────────────────────────┤
          │  decode-filter → keeps only decode pods      │
          │  random-picker → randomly pick one           │
          │  (Pod in Decode Pool)                        │
          └─────────────────────────────────────────────┘
                                │
                                ▼
                     ┌────────────────────┐
                     │ Decode Pod         │
                     │ - Streams tokens   │
                     │ - Uses KV cache    │
                     └────────────────────┘
                                │
                                ▼
                   ┌──────────────────────────┐
                   │  Final Response to User  │
                   └──────────────────────────┘

```

**Summary:**
Two-stage serving:

Prefill stage builds the prompt & KV cache.

Decode stage uses RDMA (RoCE) to fetch that cache and generate tokens efficiently.

Router plugins ensure each request phase goes to the right pool.


| Phase      | `gpt-oss-20b`                       | `qwen-pd`                            |
| ---------- | ----------------------------------- | ------------------------------------ |
| Handler    | Single profile                      | Prefill/Decode split                 |
| Scorer     | Prefix/KV/Queue scoring             | None (simple random pick)            |
| Picker     | max-score-picker                    | random-picker****                    |
| Networking | Standard TCP/IP                     | RDMA (low-latency KV cache transfer) |
| Purpose    | Optimize cache reuse in single pool | Enable 2-stage distributed inference |
