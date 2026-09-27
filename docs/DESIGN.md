# InfrAI Design Document

Status: initial direction  
Date: 2026-09-27  
Source baseline: `Mongo-Hackathon` branch `feat/demo-branch`, commit `4e6ebcb`

## 1. Executive summary

InfrAI is an open-source, evidence-driven control plane for self-hosted AI inference infrastructure.

It should help a team answer five questions:

1. What models and serving architecture fit the hardware and workload we have?
2. What should we deploy, and why is that plan valid?
3. Is the running system healthy, efficient, and meeting its service objectives?
4. Which change is worth testing next, and what evidence supports it?
5. Can that change be verified, promoted, and rolled back safely?

The existing hackathon code already contains prototypes of all five capabilities. Its most important design choice is also the right long-term one: AI may propose or explain, but deterministic code owns validation, experiment execution, promotion gates, persistence, and rollback.

The project should initially remain read-only and human-in-the-loop. It should generate reviewable plans, evidence, diffs, and deployment bundles before it gains any ability to change a real cluster.

## 2. Problem and opportunity

Running generative AI on owned or rented accelerators is a systems problem, not only a model-launch problem. A useful deployment has to coordinate:

- model weights, precision, quantization, and license constraints;
- GPU memory, topology, interconnect, CPU, RAM, storage, and network constraints;
- tensor, pipeline, data, and expert parallelism;
- batching, KV-cache capacity, prefix reuse, prefill, and decode behavior;
- replicas, request routing, admission control, and autoscaling;
- Kubernetes resources, operators, gateways, secrets, and observability;
- latency, throughput, reliability, and cost objectives;
- upgrades, evidence collection, experiments, promotion, and rollback.

The ecosystem has strong components for parts of this problem. vLLM and SGLang are inference engines. KServe, llm-d, and NVIDIA Dynamo provide different combinations of orchestration, routing, deployment, and scaling. Envoy AI Gateway provides an AI-aware ingress and policy layer. The remaining usability gap is not another inference engine: it is a neutral layer that turns user intent and evidence into explainable, reproducible, reviewable infrastructure decisions across those systems.

That is the space InfrAI should occupy.

## 3. Product thesis

### 3.1 Positioning

> InfrAI turns hardware, workloads, and runtime evidence into safe, explainable plans for self-hosted AI infrastructure.

It is a control and decision layer above existing inference engines and deployment platforms. It should integrate with upstream projects rather than obscure or reimplement them.

### 3.2 Initial users

- Engineers learning how production inference infrastructure works.
- Small teams that own NVIDIA GPUs but do not have a dedicated inference-platform group.
- Platform engineers evaluating vLLM, SGLang, KServe, llm-d, or Dynamo.
- Operators who need an evidence-backed diagnosis and a reviewable change, not an opaque autonomous action.
- Contributors experimenting with planners, evaluators, workload replay, and safe infrastructure agents.

### 3.3 Why this project can be useful

The current code combines three capabilities that are usually separated:

- a hardware-aware deployment planner;
- an operations workbench that normalizes evidence and proposes bounded changes;
- a restart-safe experimentation loop with deterministic evaluation and memory.

Keeping those in one coherent system creates a learnable path from “what should I deploy?” to “how should this deployment evolve?”

## 4. Product principles

1. **Deterministic core, probabilistic advisor.** Sizing, validation, policy checks, IDs, experiment comparison, and promotion decisions must be deterministic. Models may classify, rank, explain, and propose within a typed boundary.
2. **Evidence before action.** Every diagnosis and recommendation must identify its inputs, assumptions, missing evidence, and confidence.
3. **Reviewable by default.** The normal output is a plan, diff, benchmark report, or GitOps change. Direct mutation is a later, optional execution mode.
4. **Reproducible experiments.** Incumbent and candidate must see the same persisted workload trace. Results must be content-addressed and resumable.
5. **Safe failure.** Missing data, model-provider errors, invalid model output, version drift, or stale state should produce a blocked plan or rules-based fallback, never an unreviewed action.
6. **Upstream-native artifacts.** Emit ordinary Kubernetes resources, Helm values, engine arguments, and reports that remain useful without InfrAI.
7. **Pluggable ecosystem.** Models, accelerators, engines, orchestrators, gateways, telemetry sources, evaluators, and state stores need explicit adapter contracts.
8. **Learn in public.** Architecture decisions, support matrices, benchmark methods, and limitations should be documented as part of the open-source product.

## 5. Scope

### 5.1 In scope

- Read-only hardware and cluster discovery.
- Model, engine, and platform compatibility checks.
- Capacity screening and topology selection.
- Deployment artifact generation and schema validation.
- Runtime evidence import from files and APIs.
- Secret redaction, normalization, bounded storage, and provenance.
- Performance and reliability diagnoses.
- Repeatable workload capture, replay, and comparison.
- Policy-driven promotion recommendations and rollback plans.
- Durable campaigns, context manifests, lessons, and experiment history.
- Web UI, API, and CLI workflows.
- GitOps integration as the first real change-delivery mechanism.

### 5.2 Not initially in scope

- Building a new inference engine, scheduler, gateway, Kubernetes distribution, or observability backend.
- Training or fine-tuning models.
- Claiming production readiness from synthetic simulation alone.
- Direct cluster changes without explicit policy, credentials, audit history, and user approval.
- A universal cloud cost optimizer in the first releases.
- Hiding the selected upstream stack behind a proprietary abstraction.

## 6. Domain model

The long-term domain should use one shared vocabulary. The current code has a simulator-specific `Arch` and a production-oriented `ProdArch`; these should converge.

| Entity | Meaning |
|---|---|
| Inventory | Observed hardware, cluster, network, storage, and software capabilities with provenance. |
| Workload | Model task, traffic distribution, context/output lengths, concurrency, quality, and SLOs. |
| Model profile | Immutable model metadata: revision, architecture, memory characteristics, license, supported tasks, and formats. |
| Stack | A versioned composition of engine, runtime, orchestrator, router/gateway, and dependencies. |
| Deployment plan | Inputs, assumptions, selected stack, allocation, rendered artifacts, commands, checks, and blockers. |
| Evidence | Bounded, redacted, content-addressed telemetry or operational data with time range and provenance. |
| Change proposal | A typed patch to a plan plus diagnosis, trade-offs, files, diff, verification conditions, and rollback. |
| Workload trace | Immutable request timing and shape used for repeatable comparison. Payload content should be excluded or irreversibly sanitized by default. |
| Trial | One architecture executing one trace under a declared environment and benchmark method. |
| Evaluation | Deterministic gates comparing trial sets against policy. |
| Campaign | A durable sequence of observations, proposals, trials, decisions, and checkpoints. |
| Revision | A versioned, immutable desired architecture and its generated artifacts. |
| Lesson | A scoped claim linked to the evidence and experiments that support or contradict it. |

## 7. User workflows

### 7.1 Plan a first deployment

```text
discover/import inventory
        -> validate observations
        -> select model candidates
        -> enumerate compatible stacks/topologies
        -> size and rank candidates
        -> render native artifacts
        -> schema/policy/security checks
        -> reviewable bundle
```

The model may explain workload traits and rank already-valid candidates. It may not invent an unvalidated stack or override a blocker.

### 7.2 Diagnose a running deployment

```text
select deployment revision
        -> import evidence
        -> bound + redact + normalize
        -> deterministic fault hypotheses
        -> identify missing evidence
        -> bounded patch proposal
        -> render + validate + diff
        -> verification checklist
```

### 7.3 Compare a candidate

```text
capture or synthesize workload trace
        -> persist immutable trace
        -> run alternating incumbent/candidate repeats
        -> exclude declared warmup
        -> evaluate sample, error, SLO, cost, and regression gates
        -> recommend reject/promote
        -> record scoped lesson
```

### 7.4 Evolve safely

```text
OBSERVE -> BUILD_CONTEXT -> PROPOSE -> VALIDATE -> PREPARE_REPLAY
        -> RUN_TRIALS -> EVALUATE -> PROMOTE or REJECT
        -> VERIFY_LIVE -> LEARN -> CHECKPOINT
```

The controller owns the state machine. Each transition uses compare-and-swap semantics, an expiring lease, an incrementing revision, and an immutable checkpoint. Model calls remain stateless.

## 8. Target architecture

```text
                         Web UI / CLI / API
                                  |
                        Application services
                                  |
       +--------------------------+--------------------------+
       |                          |                          |
  Planning service         Operations service        Campaign controller
       |                          |                          |
 candidate enumeration      evidence pipeline         experiment runner
 sizing + ranking           diagnosis + patches       deterministic evaluator
 render + validate          verify conditions         promotion recommender
       |                          |                          |
       +--------------------------+--------------------------+
                                  |
                  Typed domain + policy + provenance
                                  |
        +-------------------------+-------------------------+
        |                         |                         |
  Capability adapters       Runtime adapters          Persistence adapters
  models/hardware/stacks    telemetry/benchmark/GitOps MongoDB/local store
        |                         |                         |
   Upstream projects        Clusters and metrics       Durable evidence
```

### 8.1 Control plane versus data plane

InfrAI's durable product should be a control plane. It should not proxy production inference traffic. The current `gateway/` is valuable as a simulation and test fixture, but real deployments should use upstream gateways and routers. This separation reduces security exposure and keeps benchmark behavior distinct from the production data plane.

### 8.2 Core services

**Inventory service**

- Accept signed/read-only probe output and optional cluster API discovery.
- Record whether every field was observed, supplied, estimated, or unknown.
- Normalize GPU identities and capabilities using a versioned hardware database.
- Treat mixed nodes, multi-node topology, MIG, and non-NVIDIA accelerators as explicit capabilities rather than special cases.

**Capability registry**

- Replace hard-coded dictionaries with versioned manifests for models, engines, platforms, gateways, and compatibility rules.
- Keep install sources, image digests, chart versions, CRD versions, supported accelerators, and known incompatibilities together.
- Validate registry entries in CI and support community-contributed adapters.

**Planner**

- Separate hard feasibility constraints from ranking preferences.
- Generate multiple candidates rather than one implicit default.
- Add quantization, pipeline/data/expert parallelism, heterogeneous clusters, model download/storage, network bandwidth, and failure-domain constraints over time.
- Preserve assumptions and uncertainty in the result.

**Renderer and validator**

- Use stack-specific adapters to render native manifests.
- Validate against exact upstream CRDs, Kubernetes schemas, policy rules, and compatibility matrices.
- Produce a software bill of materials and pin images by digest for release artifacts.
- Make every bundle reproducible from a normalized request and registry version.

**Evidence service**

- Keep the current bounded, redacted, content-addressed normalization model.
- Add adapters for Prometheus, OpenTelemetry, Kubernetes events/status, DCGM, engine metrics, and benchmark reports.
- Preserve units, counters versus gauges, labels, time ranges, sampling, and source identity.
- Store raw evidence only when explicitly configured; default to the minimal normalized record.

**Experiment service**

- Support synthetic traces, sanitized captured traces, and standard benchmark suites.
- Record environment fingerprints so results from different drivers, images, kernels, or hardware are not silently compared.
- Distinguish screening simulation from measured trials.
- Add quality and correctness gates when a proposal changes model, precision, quantization, or engine behavior.

**Campaign controller**

- Retain persisted stages, leases, revisions, idempotency keys, and immutable checkpoints.
- Run one side effect per transition and make retries safe.
- Require explicit policies for budgets, allowed changes, approval boundaries, and rollback.
- Deliver real changes through a Git branch or pull request before direct reconciliation is considered.

**Memory and retrieval**

- Retain exact context manifests, included/excluded evidence, and token budgets.
- Scope lessons by model, model revision, hardware, engine/version, topology, workload regime, and objective.
- Treat a lesson as a claim with supporting and contradicting evidence, not free-form chat memory.
- Make lexical retrieval the reliable baseline and embeddings an optional enhancement.

## 9. Safety model

### 9.1 Authority levels

| Level | Capability | Default |
|---|---|---|
| 0 | Inspect local files and imported evidence. | Enabled |
| 1 | Generate plans, reports, manifests, and diffs. | Enabled |
| 2 | Run isolated benchmarks or simulations. | Explicit setup |
| 3 | Open a GitOps change for review. | Opt-in |
| 4 | Apply approved changes to a non-production cluster. | Future, opt-in |
| 5 | Automated production promotion and rollback. | Future, policy-gated |

Each deployment must declare its maximum authority. Model output can never raise it.

### 9.2 Required controls before real mutation

- Strong authentication and role-based authorization.
- Separate credentials per cluster and environment.
- Allowlisted resources, namespaces, fields, and commands.
- Policy-as-code checks and server-side dry runs.
- Tamper-evident audit log with actor, evidence, decision, and artifact hashes.
- Secret-safe logs and external secret references.
- Signed artifacts, image digests, dependency and vulnerability scanning.
- Canary or shadow rollout, health gates, timeouts, and tested rollback.
- Kill switch and per-campaign cost/experiment budgets.
- Human approval at a configurable boundary.

## 10. Current codebase map

The imported code is a strong prototype, but several product generations coexist in it.

| Path | Current responsibility | Long-term treatment |
|---|---|---|
| `common/` | Environment configuration, Mongo access, simulator and campaign contracts, routing helpers. | Split domain contracts, settings, policies, and persistence interfaces. |
| `production/planner.py` | Hardware/model sizing, YAML generation, schema checks, commands, bundle content. | Become planner core plus renderer adapters. |
| `production/catalog.py` | Two pinned Qwen models, six projects, three recipes, install commands. | Replace with the versioned capability registry. |
| `production/inventory.py` | Read-only discovery script and JSON/text import. | Expand into inventory adapters and capability probes. |
| `production/api.py` | Planner HTTP endpoints and bundle download. | Retain behind versioned API models. |
| `production/evidence.py` | Bounded redaction, normalization, diagnosis, verification, fixtures. | Preserve as evidence core; separate format adapters from diagnoses. |
| `production/operations.py` | Performance/reliability roles, bounded patches, re-rendering, diffs. | Generalize to typed change plugins and policy checks. |
| `production/ops_api.py` | Evidence, analysis, changes, verification, campaign, and revision endpoints. | Split application services from HTTP transport. |
| `production/evolution.py` | Simulated long-horizon evolution of provisioned plans. | Merge with the generic campaign controller and real benchmark adapters. |
| `production/sim.py` | Production-plan simulator and `ProdArch`. | Keep as a clearly labelled screening backend. |
| `production/harness.py` | Namespaced production collections, context, lessons, indexes. | Move to repository interfaces and migrations. |
| `agent/` | Architect, state store, replay runner, evaluator, promotion, and durable workflow. | Keep the deterministic workflow; merge duplicate production campaign logic. |
| `memory/` | Regime bandit, retrieval, embeddings, compaction, evidence-linked lessons. | Keep concepts; formalize scopes, claims, and storage interfaces. |
| `docs/ingest.py` | Hash-addressed local/web documentation ingestion. | Add source allowlists, content limits, provenance, and parser adapters. |
| `gateway/` | Demo OpenAI proxy, routing, telemetry. | Move under simulation/test support; do not make it the production gateway. |
| `infra/` | Fake/Docker simulator deployment, replay, blue/green demo reconciler. | Rename as simulation/benchmark adapters; add real isolated runners separately. |
| `traffic/` | Poisson load generator and scripted traffic phases. | Become workload generation and trace tooling. |
| `ui/` | Plan, performance, reliability, campaign, and simulator views. | Retain, rebrand, and consume versioned API schemas. |
| `scripts/` | Database bootstrap, doc ingestion, all-in-one demo. | Replace with CLI commands and migrations while retaining demo entry points. |
| `api/` + `vercel.json` | Serverless planner entry point. | Keep only if a stateless hosted planner remains a product goal. |
| `tests/` | Contracts, state, workflow, evidence, planner, and safety behavior. | Preserve and expand into adapter, migration, integration, and end-to-end suites. |

## 11. What is already strong

- Pydantic contracts constrain model output and domain inputs.
- Planner artifacts are generated without applying them.
- Invalid or unavailable LLM advice falls back to deterministic rules.
- Plans expose blockers and validate YAML against pinned schema projections.
- Evidence is bounded, redacted, normalized, and content-addressed.
- Changes are limited to renderer-supported fields with explicit bounds.
- Campaign state uses leases, compare-and-swap revisions, idempotent budget debits, and immutable checkpoints.
- Replays are deterministic and persisted.
- Trial order alternates to reduce order bias.
- Evaluation uses hard sample/error/SLO gates and a paired bootstrap latency-regression bound.
- Promotion checks the expected incumbent version and rollback creates a new version rather than rewriting history.
- Context manifests record exactly what the model could and could not see.
- Offline rules, lexical retrieval, in-memory Mongo, and fake simulators make the project approachable.

These are the foundations to preserve during refactoring.

## 12. Current limitations and design debt

1. **Hard-coded support matrix.** Only two Qwen models, two simulated GPU classes, three recipes, and a small set of knobs are modeled.
2. **Screening estimates are too simple.** BF16 weight plus KV estimates omit many runtime, quantization, fragmentation, communication, and architecture-specific effects.
3. **Version drift is manual.** Upstream APIs and releases move quickly. The catalog includes versions that must be revalidated together; compatibility cannot be inferred from independent latest versions.
4. **Two architecture models and two workflows.** Simulator `Arch`/`DurableOptimizationWorkflow` and production `ProdArch`/`EvolutionCampaign` duplicate concepts.
5. **Simulation is not production evidence.** Fake and inference-simulator results demonstrate the workflow but do not establish real performance or reliability.
6. **The production evolution loop is still simulation-backed.** It records plan revisions but does not benchmark on an isolated real cluster or deliver a GitOps change.
7. **Persistence is tightly coupled to MongoDB collection shapes.** There is no migration framework or repository boundary.
8. **HTTP/application/domain boundaries are mixed.** Several modules combine transport, persistence, decisions, and rendering.
9. **Security is demo-level.** There is no user authentication, authorization, tenant isolation, signed artifacts, or cluster credential model.
10. **Remote documentation ingestion needs hardening.** URL policy, network boundaries, maximum response size, content type, and provenance controls need to be explicit.
11. **Packaging and contributor infrastructure are absent.** There is no `pyproject.toml`, lockfile, container image, CI workflow, license, contribution guide, security policy, or release process.
12. **The hosted planner and long-running controller have different operational needs.** Treating both as one deployment obscures state, identity, background work, and availability requirements.

## 13. Proposed package boundaries

```text
infrai/
  domain/          # immutable models, policies, decisions, provenance
  registry/        # versioned capabilities and compatibility data
  planning/        # constraints, candidate enumeration, ranking
  rendering/       # stack-specific artifacts and validation
  evidence/        # formats, normalization, redaction, storage
  experiments/     # traces, runners, metrics, evaluation
  campaigns/       # durable state machine and approvals
  memory/          # manifests, retrieval, claims, compaction
  adapters/
    inventory/
    models/
    stacks/
    telemetry/
    benchmarks/
    delivery/
    persistence/
  api/             # versioned transport only
  cli/             # local-first workflows
  web/             # UI assets/client
  simulation/      # fake gateway, traffic, screening backends
```

Refactoring should be incremental. Existing behavior should first receive characterization tests, then move behind these interfaces without a rewrite.

## 14. Roadmap

### Phase 0 — Make the hackathon prototype a healthy open-source project

- Rebrand user-facing Harness names to InfrAI while preserving attribution and history.
- Add an OSI-approved license, `CONTRIBUTING.md`, `SECURITY.md`, code of conduct, governance notes, and architecture decision records.
- Add `pyproject.toml`, a reproducible dependency lock, supported Python versions, lint/type/test commands, and CI.
- Establish database migrations and sample/demo data.
- Create a versioned `/api/v1` contract and a local CLI.
- Mark simulator results clearly in every UI and API response.
- Add a compatibility audit for all pinned projects, charts, CRDs, images, and commands.
- Consolidate the two architecture models and the two campaign controllers.

Exit criterion: a new contributor can clone, run tests, launch the offline demo, and understand which paths are simulation versus real planning.

### Phase 1 — Reliable read-only planner

- Introduce the capability registry and adapter schemas.
- Expand GPU inventory and normalize real device identities.
- Add model metadata ingestion and immutable model revisions.
- Model quantization and more realistic memory headroom.
- Enumerate and compare multiple valid plans.
- Add compatibility, policy, security, and artifact checks.
- Ship CLI workflows such as `infrai inventory import`, `infrai plan`, `infrai validate`, and `infrai bundle`.

Exit criterion: plans are reproducible, explain assumptions, and are validated against an explicitly supported version matrix.

### Phase 2 — Measured benchmarking and evidence

- Add isolated real-engine runners for vLLM and SGLang.
- Add Prometheus, OpenTelemetry, DCGM, Kubernetes, and engine-metric adapters.
- Record driver, CUDA, image digest, engine, model, and hardware fingerprints.
- Add sanitized workload trace capture and standard synthetic suites.
- Calibrate or reject simulator estimates against real measurements.
- Add model-quality/correctness gates for model, precision, and quantization changes.

Exit criterion: InfrAI can produce a reproducible comparison report from measured trials, not only simulation.

### Phase 3 — GitOps operations assistant

- Convert diagnoses into typed, policy-checked changes.
- Create branches or pull requests with manifests, evidence, test results, verification conditions, and rollback.
- Track approvals and deployment observations without holding broad cluster credentials.
- Add canary/shadow result ingestion and close the loop after a human-approved deployment.

Exit criterion: a team can review and merge an InfrAI-generated change using its normal GitOps process.

### Phase 4 — Optional guarded reconciliation

- Add a separate, minimal in-cluster executor.
- Enforce namespace/resource/field allowlists and short-lived identity.
- Require signed plans and policy decisions.
- Support progressive rollout, automatic halt, and tested rollback.
- Keep human approval configurable and on by default.

Exit criterion: a narrowly scoped non-production environment can safely execute a complete campaign with an auditable kill switch.

### Phase 5 — Ecosystem and research platform

- Stable adapter SDK and compatibility test kit.
- Community registry for stacks and benchmark profiles.
- Pluggable evaluators and policies.
- Multi-cluster and heterogeneous accelerator support.
- Public, reproducible benchmark datasets and decision traces.

## 15. Near-term release definition

The first serious release should be smaller than the full vision. A useful `0.1` should provide:

- offline installation and demo;
- inventory import;
- multi-candidate planning for a small, declared support matrix;
- native manifest and report bundles;
- schema and policy validation;
- evidence import and deterministic diagnosis;
- simulation clearly separated from measured results;
- durable local or Mongo-backed history;
- no direct cluster mutation.

Success is not the number of supported stacks. Success is that every supported path is reproducible, understandable, and honest about uncertainty.

## 16. Testing strategy

### Unit and property tests

- Domain invariants, normalization, redaction, compatibility, sizing, rendering, idempotency, and policies.
- Property tests for input bounds, deterministic IDs, replay order, and forbidden mutations.
- Golden files for generated manifests and reports.

### Contract tests

- Exact CRD/OpenAPI schemas for every supported upstream version.
- Adapter fixtures for telemetry and inventory formats.
- Backward/forward migration tests for persisted records.

### Integration tests

- Ephemeral MongoDB and a local persistence backend.
- Kind/K3s clusters for server-side dry run and operator installation.
- Engine smoke tests on CPU where possible and scheduled GPU CI where necessary.
- Failure injection for leases, restarts, stale promotion, missing evidence, and rollback.

### Benchmark validity

- Warmup, repeat count, arrival model, timeout, and exclusion rules must be explicit.
- Compare identical persisted traces and alternate execution order.
- Retain raw-enough outcome data to recompute statistics.
- Never compare trials with incompatible environment fingerprints without an explicit override.
- Publish methodology and limitations next to results.

## 17. Observability and audit

Every plan, change, trial, and campaign transition should expose:

- stable ID and schema version;
- normalized inputs and content hashes;
- actor and authority level;
- source provenance and time range;
- registry and policy versions;
- model-provider identity when AI advice was used;
- included/excluded context manifest;
- generated artifact hashes;
- validation and evaluation gates;
- approval, delivery, verification, and rollback events.

Operational telemetry should use OpenTelemetry conventions where possible and export Prometheus-compatible metrics. The audit record must remain useful if the model provider is unavailable.

## 18. Open-source project shape

- Use a permissive license if the goal is broad infrastructure adoption; Apache-2.0 is a natural candidate because it includes an explicit patent grant.
- Keep the capability registry, planner, evaluators, and schemas in the public repository.
- Document how commercial or hosted services could extend the project without making the open-source core incomplete.
- Publish support tiers: experimental, validated, and production-observed.
- Require benchmark provenance for performance claims.
- Use architecture decision records for major choices such as persistence interfaces, agent authority, and adapter stability.
- Provide “good first issue” adapters and fixtures so the repository also serves the learning goal.

## 19. Decisions to make next

1. Is the primary first user a learner with one workstation, a small on-prem team, or a Kubernetes platform team?
2. Should the first supported real path be direct vLLM/SGLang on a host, Kubernetes via KServe/llm-d, or Kubernetes via Dynamo?
3. Is MongoDB a required product dependency, the first persistence adapter, or only the original hackathon backend?
4. Which license and governance model should the project adopt?
5. Should `0.1` ship a CLI-first experience, a web-first experience, or both with one shared application service?
6. What is the initial support matrix that the maintainers can continuously test on real hardware?

Recommended defaults: learner/small-team first, CLI plus local web UI, MongoDB as an optional adapter, no direct cluster mutation, and one narrow end-to-end stack validated well before adding more.

## 20. Upstream references used for this direction

- [vLLM parallelism and scaling](https://docs.vllm.ai/en/latest/serving/parallelism_scaling/)
- [vLLM automatic prefix caching](https://docs.vllm.ai/en/latest/design/prefix_caching/)
- [KServe LLMInferenceService overview](https://kserve.github.io/website/docs/model-serving/generative-inference/llmisvc/llmisvc-overview)
- [llm-d architecture](https://llm-d.ai/docs/dev/architecture)
- [llm-d Endpoint Picker and request scheduling](https://llm-d.ai/docs/dev/architecture/core/router/epp)
- [NVIDIA Dynamo model deployment](https://docs.nvidia.com/dynamo/kubernetes/model-deployment/introduction)
- [NVIDIA Dynamo auto deployment](https://docs.nvidia.com/dynamo/kubernetes/auto-deployment/overview)
- [Envoy AI Gateway architecture](https://aigateway.envoyproxy.io/docs/0.4/concepts/architecture/system-architecture/)
- [Envoy AI Gateway usage-based rate limiting](https://aigateway.envoyproxy.io/docs/0.1/capabilities/usage-based-ratelimiting/)
- [SGLang project documentation](https://docs.sglang.ai/)

These projects evolve quickly. Links and compatibility claims must be reviewed as part of every InfrAI release; “latest” versions should never be combined without an explicitly tested compatibility matrix.
