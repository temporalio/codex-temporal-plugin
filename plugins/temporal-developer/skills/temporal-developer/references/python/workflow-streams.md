# Workflow Streams (Python SDK)

> **Public Preview** — The `temporalio.contrib.workflow_streams` module is in [Public Preview](https://docs.temporal.io/evaluate/development-production-features/release-stages#public-preview). Only the Python client is available today; cross-language client support is on the roadmap.

## Overview

Workflow Streams gives a Workflow a durable, offset-addressed event channel built on Signals, Updates, and Queries. It batch-publishes events to amortize per-Signal cost, deduplicates batches for exactly-once delivery, supports topic filtering, and carries state across Continue-As-New. <!-- docs/develop/python/workflows/workflow-streams.mdx:22-23 -->

Use it when outside observers need to follow Workflow progress: updating a UI as an AI agent works, surfacing status from a pipeline, or reporting intermediate results from a data job. It targets modest fan-out (tens of publishers and subscribers per Workflow, not thousands) and is not suited to ultra-low-latency cases like real-time voice. <!-- docs/develop/python/workflows/workflow-streams.mdx:25 -->

The Workflow hosts the event log. Publishers append events (the Workflow itself, Activities, or external processes). Subscribers attach to the Workflow ID, optionally filter by topic, and consume events by long-polling from an offset they store. <!-- docs/develop/python/workflows/workflow-streams.mdx:27 -->

## Stream hosting

A `WorkflowStream` lives inside a Workflow, so the first choice is whether the work-Workflow also hosts the stream, or a separate Workflow exists only for the stream. <!-- docs/develop/python/workflows/workflow-streams.mdx:49-50 -->

**Host the stream on the work-Workflow** when events come from what that Workflow orchestrates (agent run, order pipeline, chat session). The Workflow ID used to start the work is the same one subscribers attach to. This is the common shape for AI agents and progress streaming. <!-- docs/develop/python/workflows/workflow-streams.mdx:53 -->

**Use a dedicated stream Workflow** when the stream should outlive any single producer, accept fan-in from multiple unrelated sources, or be subscribable before work starts. Trade-off: you need explicit lifecycle management (signal-driven shutdown or Continue-As-New). <!-- docs/develop/python/workflows/workflow-streams.mdx:55 -->

## Enable streaming on a Workflow

Construct a `WorkflowStream` from the Workflow's `@workflow.init` method. Construction must happen there because the stream's handlers must be registered before the first publish Signal arrives; doing it from `@workflow.run` raises `RuntimeError`. <!-- docs/develop/python/workflows/workflow-streams.mdx:61 -->

Constructing more than one stream on the same Workflow also raises `RuntimeError`. <!-- docs/develop/python/workflows/workflow-streams.mdx:82 -->

```python
from dataclasses import dataclass

from temporalio import workflow
from temporalio.contrib.workflow_streams import WorkflowStream


@dataclass
class OrderInput:
    order_id: str


@workflow.defn
class OrderWorkflow:
    @workflow.init
    def __init__(self, input: OrderInput) -> None:
        self.stream = WorkflowStream()
```
<!-- docs/develop/python/workflows/workflow-streams.mdx:63-80 -->

## Publish from a Workflow

Bind a topic name to its event type once via `self.stream.topic("name", type=Type)`, then call `publish()` on the returned handle. The `type=` argument is optional (defaults to `Any`); pass it to record the binding so re-binding the same name to an unequal type raises. <!-- docs/develop/python/workflows/workflow-streams.mdx:88-89,125-126 -->

```python
@dataclass
class StatusEvent:
    state: str
    progress: int = 0
    detail: str = ""


@workflow.defn
class OrderWorkflow:
    @workflow.init
    def __init__(self, input: OrderInput) -> None:
        self.stream = WorkflowStream()
        self.status = self.stream.topic("status", type=StatusEvent)

    @workflow.run
    async def run(self, input: OrderInput) -> None:
        self.status.publish(StatusEvent(state="validating", detail="checking inventory"))
        await validate_order(input.order_id)

        self.status.publish(StatusEvent(state="charging", progress=33))
        await charge_payment(input.order_id)

        self.status.publish(StatusEvent(state="completed", progress=100))
```
<!-- docs/develop/python/workflows/workflow-streams.mdx:94-120 -->

`publish()` runs the payload converter per item. The codec chain (encryption, compression) runs once on the Signal/Update envelope, not per item. <!-- docs/develop/python/workflows/workflow-streams.mdx:122 -->

## Publish from a client

Any process with a Temporal `Client` and the target Workflow ID can publish via `WorkflowStreamClient`. This covers HTTP backends, starters, scripts, other Workflows' Activities, and standalone Activities. <!-- docs/develop/python/workflows/workflow-streams.mdx:128 -->

```python
from datetime import timedelta

from temporalio.client import Client
from temporalio.contrib.workflow_streams import WorkflowStreamClient


async def publish_status(workflow_id: str) -> None:
    temporal_client = await Client.connect("localhost:7233")
    stream_client = WorkflowStreamClient.create(
        temporal_client,
        workflow_id=workflow_id,
        batch_interval=timedelta(milliseconds=200),
    )
    async with stream_client:
        status = stream_client.topic("status", type=StatusEvent)
        status.publish(StatusEvent(state="started"))
        ...
        # Buffer is flushed on context manager exit.
```
<!-- docs/develop/python/workflows/workflow-streams.mdx:133-151 -->

### Publish from an Activity

Inside an Activity scheduled by a Workflow, `WorkflowStreamClient.from_within_activity()` infers the Temporal `Client` and parent Workflow ID from Activity context: <!-- docs/develop/python/workflows/workflow-streams.mdx:153-154 -->

```python
from temporalio import activity
from temporalio.contrib.workflow_streams import WorkflowStreamClient


@activity.defn
async def stream_deltas(order_id: str) -> None:
    client = WorkflowStreamClient.from_within_activity()
    async with client:
        deltas = client.topic("delta", type=Delta)
        for delta in generate_deltas(order_id):
            deltas.publish(delta)
            activity.heartbeat()
        # Buffer is flushed on context manager exit.
```
<!-- docs/develop/python/workflows/workflow-streams.mdx:156-169 -->

For a standalone Activity (started directly via `Client.start_activity`), there is no parent Workflow context, so `from_within_activity()` raises. Fall back to the general pattern with `activity.client()` and the target Workflow ID threaded through the Activity's input. <!-- docs/develop/python/workflows/workflow-streams.mdx:171 -->

When events originate in an Activity, publish from the Activity directly rather than returning them for the Workflow to forward. The Workflow processes the Activity's return value and emits its own lifecycle events; keeping Workflow state independent of streamed output lets retried Activity attempts surface to subscribers without polluting durable state. <!-- docs/develop/python/workflows/workflow-streams.mdx:130-131 -->

### Flush control

- `force_flush=True` on a publish wakes the background flusher so the buffer ships without waiting for the next interval. Use for latency-sensitive events (first delta of a response, `RETRY` events). The call returns immediately; it does not wait for delivery. <!-- docs/develop/python/workflows/workflow-streams.mdx:175-179 -->
- `await client.flush()` is a mid-stream barrier. Successful completion proves the Temporal server has received all prior publications. Exiting `async with client` already flushes, so explicit flush is only for barriers in the middle. <!-- docs/develop/python/workflows/workflow-streams.mdx:183-184 -->

```python
async with client:
    deltas = client.topic("delta", type=Delta)
    for delta in first_phase():
        deltas.publish(delta)

    await client.flush()
    checkpoint_id = await record_phase_one_complete()

    for delta in second_phase(checkpoint_id):
        deltas.publish(delta)
```
<!-- docs/develop/python/workflows/workflow-streams.mdx:185-196 -->

`publish()` is non-blocking and applies no backpressure. If a publisher emits faster than batches can ship, the buffer grows unboundedly. Apply backpressure upstream of `publish()` if needed — the library does not pick a policy for you. <!-- docs/develop/python/workflows/workflow-streams.mdx:198-201 -->

## Subscribe

Subscribing uses `WorkflowStreamClient.create(client, workflow_id)` or `from_within_activity()` inside an Activity. Subscribing from inside the host Workflow is intentionally unsupported — the Workflow would mix durable state with partial output from failed Activity attempts. <!-- docs/develop/python/workflows/workflow-streams.mdx:204-207 -->

```python
from temporalio.client import Client
from temporalio.contrib.workflow_streams import WorkflowStreamClient


async def watch_order(order_id: str) -> None:
    temporal_client = await Client.connect("localhost:7233")
    stream = WorkflowStreamClient.create(temporal_client, workflow_id=order_id)

    status = stream.topic("status", type=StatusEvent)
    async for item in status.subscribe():
        evt = item.data
        print(f"[{evt.progress:3d}%] {evt.state}: {evt.detail}")
        if evt.state == "completed":
            break
```
<!-- docs/develop/python/workflows/workflow-streams.mdx:212-227 -->

The iterator handles re-polling, pagination (poll responses capped at ~1 MB), and Workflow-side log truncation transparently. A subscriber whose offset falls below the log base after a `truncate()` is silently advanced to the current base. <!-- docs/develop/python/workflows/workflow-streams.mdx:228-229,419 -->

Any process bridging events to the outside world (SSE proxy, forwarding Activity) can stay stateless — store the last delivered `item.offset` and reconnect from there. <!-- docs/develop/python/workflows/workflow-streams.mdx:208 -->

### Heterogeneous topics

To consume multiple topics with different payload types, call `client.subscribe()` directly with a list of names and pass `result_type=temporalio.common.RawValue`. Dispatch on `item.topic` and decode with the payload converter: <!-- docs/develop/python/workflows/workflow-streams.mdx:233-234 -->

```python
from temporalio.common import RawValue

converter = temporal_client.data_converter.payload_converter

async for item in stream.subscribe(["status", "progress"], result_type=RawValue):
    if item.topic == "status":
        evt = converter.from_payload(item.data.payload, StatusEvent)
        print(f"[status] {evt.state}: {evt.detail}")
    elif item.topic == "progress":
        evt = converter.from_payload(item.data.payload, ProgressEvent)
        print(f"[progress] {evt.message}")
```
<!-- docs/develop/python/workflows/workflow-streams.mdx:236-248 -->

### Closing the stream

The `async for` does not know when the publisher is done — end-of-stream is an application-level concern. Without coordination, a subscriber keeps polling until the Workflow reaches a terminal state. <!-- docs/develop/python/workflows/workflow-streams.mdx:255-256 -->

Two approaches for clean shutdown: <!-- docs/develop/python/workflows/workflow-streams.mdx:262 -->

**Fixed sleep** — publish a sentinel, sleep before returning so in-flight polls can fetch it:

```python
self.status.publish(StatusEvent(state="completed", progress=100))
await workflow.sleep(timedelta(seconds=30))
return result
```
<!-- docs/develop/python/workflows/workflow-streams.mdx:266-270 -->

**Acknowledgment handshake** — subscriber signals the Workflow on receipt, Workflow waits with a timeout fallback:

```python
@workflow.signal
async def subscriber_acknowledged_terminator(self) -> None:
    self.subscriber_done = True

@workflow.run
async def run(self, input: ChatInput) -> str:
    ...
    try:
        await workflow.wait_condition(
            lambda: self.subscriber_done,
            timeout=timedelta(seconds=30),
        )
    except TimeoutError:
        pass
    return result
```
<!-- docs/develop/python/workflows/workflow-streams.mdx:277-295 -->

After the `async for` exits, call `await temporal_client.get_workflow_handle(workflow_id).describe()` to inspect the Workflow's terminal status if needed. <!-- docs/develop/python/workflows/workflow-streams.mdx:297 -->

## Continue-As-New

Skip this section for short-lived Workflows (single chat completion, order pipeline). CAN is for streams that run for hours or accumulate thousands of events. <!-- docs/develop/python/workflows/workflow-streams.mdx:301 -->

Subscribers automatically follow Continue-As-New chains — the Workflow ID is stable, so the iterator fetches a fresh handle and continues polling from the carried offset. <!-- docs/develop/python/workflows/workflow-streams.mdx:303 -->

Carry both application state and stream state across the boundary. Add a `WorkflowStreamState | None` field to your Workflow input (the `| None` is required — with `Any`, the data converter rebuilds the field as a plain `dict` and `WorkflowStream(prior_state=...)` raises `AttributeError`): <!-- docs/develop/python/workflows/workflow-streams.mdx:305,347 -->

```python
from dataclasses import dataclass, field

from temporalio import workflow
from temporalio.contrib.workflow_streams import WorkflowStream, WorkflowStreamState


@dataclass
class AppState:
    items_processed: int = 0


@dataclass
class WorkflowInput:
    app_state: AppState = field(default_factory=AppState)
    stream_state: WorkflowStreamState | None = None


@workflow.defn
class LongRunningWorkflow:
    @workflow.init
    def __init__(self, input: WorkflowInput) -> None:
        self.app_state = input.app_state
        self.stream = WorkflowStream(prior_state=input.stream_state)

    @workflow.run
    async def run(self, input: WorkflowInput) -> None:
        while True:
            await do_one_iteration(self)
            if workflow.info().is_continue_as_new_suggested():
                await self.stream.continue_as_new(
                    lambda stream_state: [
                        WorkflowInput(
                            app_state=self.app_state,
                            stream_state=stream_state,
                        )
                    ]
                )
```
<!-- docs/develop/python/workflows/workflow-streams.mdx:307-345 -->

To pass other Continue-As-New parameters (`task_queue`, `retry_policy`, `run_timeout`), use the explicit recipe: <!-- docs/develop/python/workflows/workflow-streams.mdx:349 -->

```python
self.stream.detach_pollers()
await workflow.wait_condition(workflow.all_handlers_finished)
workflow.continue_as_new(
    args=[WorkflowInput(app_state=self.app_state, stream_state=self.stream.get_state())],
    task_queue="other-tq",
)
```
<!-- docs/develop/python/workflows/workflow-streams.mdx:352-358 -->

The carried `WorkflowStreamState` includes the entire in-memory log, so streams with large items can hit the per-payload size limit at rollover. Offload large payloads via External Storage and combine with `truncate()` to keep the carried log small. <!-- docs/develop/python/workflows/workflow-streams.mdx:360 -->

## Tuning

The core trade-off: a more responsive UI means more messages and more history per second. Messages drive workload (and on metered deployments, billing), while history accumulates against per-run limits. For long-running streams, plan a Continue-As-New policy from the start. <!-- docs/develop/python/workflows/workflow-streams.mdx:364-366 -->

### Key settings

| Setting | Default | Description |
|---------|---------|-------------|
| `batch_interval` | 2 seconds | Max time between automatic flushes from the client. Lower for live feel; raise to amortize Signal cost. For LLM token streaming, 200 ms is a good starting point. Below 100 ms, per-Signal RPC overhead starts to dominate. |
| `max_batch_size` | Unbounded | Caps items per batch. Set when a hot publisher could exceed the gRPC payload limit between intervals. |
| `poll_cooldown` | 100 ms | Subscriber sleep between polls. Skipped only when a poll response was capped at ~1 MB and more items remain. |
| `max_retry_duration` | 10 minutes | How long the client retries a failed publish batch before raising `TimeoutError`. |
| `publisher_ttl` | 15 minutes | How long the Workflow retains per-publisher dedup state. At each CAN, entries older than this are dropped. |
<!-- docs/develop/python/workflows/workflow-streams.mdx:370-381 -->

Keep `max_retry_duration < publisher_ttl` so a long-running retry cannot outlast its dedup record and produce a duplicate. If you tune one, tune the other. <!-- docs/develop/python/workflows/workflow-streams.mdx:382-383 -->

## Delivery semantics

**Exactly-once at the execution layer.** Each `(publisher_id, sequence)` batch lands in the Workflow's event log at most once, even if retried by the SDK or the network. Dedup state is carried across Continue-As-New (subject to `publisher_ttl`). <!-- docs/develop/python/workflows/workflow-streams.mdx:387 -->

**Ordering.** The log imposes a single total order. Within one publisher, events appear in publish order. Across concurrent publishers, interleaving is whatever the Workflow saw when serializing inbound Signals — stable once recorded but not under application control. <!-- docs/develop/python/workflows/workflow-streams.mdx:389 -->

**Activity retries surface to subscribers.** When an Activity fails partway through and Temporal retries it, *both* attempts' events appear in the stream. The conventional pattern: publish a `RETRY` event with `force_flush=True` when `activity.info().attempt > 1`, and have the consumer clear or annotate prior-attempt output when it sees one. <!-- docs/develop/python/workflows/workflow-streams.mdx:391-393 -->

**Other failure modes.** Events in a publisher's buffer are lost if the process crashes before they ship. Subscribers that crash before persisting their offset will reprocess on resume. <!-- docs/develop/python/workflows/workflow-streams.mdx:398 -->

## Gotchas

- **`WorkflowStreamClient` is asyncio-only.** Don't call `publish()` from a worker thread. <!-- docs/develop/python/workflows/workflow-streams.mdx:431 -->
- **Custom handlers on first activation.** `WorkflowStream` registers its publish-Signal handler dynamically from `__init__`, so on the first activation a publish Signal can be queued before class-level `@workflow.signal`/`@workflow.update` handlers have run. Make the handler `async def` and `await asyncio.sleep(0)` before reading state. Don't use `workflow.sleep(0)` (records a timer event). <!-- docs/develop/python/workflows/workflow-streams.mdx:432-433 -->
- **Type bindings aren't shared across publishers.** Each instance records topic types for itself only. Two publishers binding the same topic name to different types produces a decode error at subscribe time. <!-- docs/develop/python/workflows/workflow-streams.mdx:434 -->

## Stream LLM output

The headline use case: an Activity calls the model and publishes deltas; the Workflow waits for the consumer to ack end-of-stream; the consumer subscribes and clears accumulated state on `RETRY`.

### Activity (publisher)

```python
from openai import AsyncOpenAI


@dataclass
class TextDelta:
    text: str


@activity.defn
async def stream_completion(prompt: str) -> str:
    stream_client = WorkflowStreamClient.from_within_activity(
        batch_interval=timedelta(milliseconds=200),
    )
    openai_client = AsyncOpenAI(max_retries=0)

    async with stream_client:
        deltas = stream_client.topic("delta", type=TextDelta)
        retry = stream_client.topic("retry", type=dict)
        close = stream_client.topic("close")

        if activity.info().attempt > 1:
            retry.publish({"attempt": activity.info().attempt}, force_flush=True)

        full: list[str] = []
        first = True
        oai_stream = await openai_client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[{"role": "user", "content": prompt}],
            stream=True,
        )
        async for chunk in oai_stream:
            if not chunk.choices:
                continue
            text = chunk.choices[0].delta.content
            if not text:
                continue
            deltas.publish(TextDelta(text=text), force_flush=first)
            first = False
            full.append(text)
        close.publish({})
    return "".join(full)
```
<!-- docs/develop/python/workflows/workflow-streams.mdx:442-487 -->

### Workflow

```python
@workflow.defn
class ChatWorkflow:
    @workflow.init
    def __init__(self, input: ChatInput) -> None:
        self.stream = WorkflowStream()
        self.subscriber_done: bool = False

    @workflow.signal
    async def subscriber_acknowledged_terminator(self) -> None:
        self.subscriber_done = True

    @workflow.run
    async def run(self, input: ChatInput) -> str:
        result = await workflow.execute_activity(
            stream_completion,
            input.prompt,
            start_to_close_timeout=timedelta(minutes=5),
        )
        try:
            await workflow.wait_condition(
                lambda: self.subscriber_done,
                timeout=timedelta(seconds=30),
            )
        except TimeoutError:
            pass
        return result
```
<!-- docs/develop/python/workflows/workflow-streams.mdx:491-520 -->

### Consumer

```python
async def stream_chat(chat_id: str) -> str:
    stream = WorkflowStreamClient.create(temporal_client, workflow_id=chat_id)
    converter = temporal_client.data_converter.payload_converter
    output: list[str] = []

    def render() -> None:
        ...  # display the accumulated output

    async for item in stream.subscribe(
        ["delta", "retry", "close"], result_type=RawValue
    ):
        if item.topic == "retry":
            output.clear()
            render()
        elif item.topic == "delta":
            delta = converter.from_payload(item.data.payload, TextDelta)
            output.append(delta.text)
            render()
        elif item.topic == "close":
            await temporal_client.get_workflow_handle(chat_id).signal(
                ChatWorkflow.subscriber_acknowledged_terminator
            )
            break

    return "".join(output)
```
<!-- docs/develop/python/workflows/workflow-streams.mdx:523-552 -->

Key design choices in this example: <!-- docs/develop/python/workflows/workflow-streams.mdx:555-560 -->
- The Activity is the publisher because it owns the non-deterministic LLM call.
- The Activity publishes `RETRY` when `activity.info().attempt > 1` so the UI can clear stale deltas.
- Termination uses an ack handshake so the Workflow returns as soon as the subscriber confirms.
- `force_flush=True` is used only on the first delta and on `RETRY` — subsequent deltas batch at 200 ms.

## See also

- [Workflow Streams samples (samples-python)](https://github.com/temporalio/samples-python/tree/main/workflow_streams)
- [`temporalio.contrib.workflow_streams` API reference](https://python.temporal.io/temporalio.contrib.workflow_streams.html)
- `references/python/patterns.md` — Signals, Updates, and Queries that Workflow Streams is built on
- `references/python/ai-patterns.md` — AI/LLM integration patterns
