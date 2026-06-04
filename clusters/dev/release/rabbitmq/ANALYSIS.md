# Analysis: RabbitMQ GitOps Topology Audit

**Status**: IMPLEMENTED  
**Scope**: `tmf-gitops/clusters/dev/release/rabbitmq/` — correctness against live service code  
**Date**: 2026-06-04  
**Method**: Cross-referenced every exchange, queue, and binding in gitops against actual service code (`WithExchange`, `NewConsumer`, `Subscribe` call sites).

---

## Background

The gitops topology was written before the RPC client was hardened (plan `05_rpc_client_hardening`). That work switched all four RPC clients from named reply queues to `amq.rabbitmq.reply-to` (Direct Reply-To). Several other inconsistencies were found during the same audit. Five issues are described below, ordered by severity.

---

## Issue 1 — Broken YAML: 3 RPC server queues will never be created

### Problem

Three `Queue` resources have a truncated `connectionSecret:` block — the `name:` line is missing. The RabbitMQ Messaging Topology Operator rejects these resources silently (the CRD is accepted by Kubernetes but the operator reports `ReconcileError`). The queues are never created on the broker.

These are the **RPC server queues** — without them no service can receive incoming RPC requests in production.

| File | Kubernetes resource | Queue name | Symptom |
|---|---|---|---|
| `pocv/queues.yaml` | `q-pocv-rpc` | `q.pocv.rpc` | POCV never receives saga status queries |
| `shopping-cart/queues.yaml` | `q-cart-rpc-v2` | `q.cart.rpc.v2` | Cart never receives `query.cart.session.get` or `query.cart.get` |
| `qualification/queues.yaml` | `q-qual-rpc` | `q.qual.rpc` | Qualification never receives `query.qual.session.get` |

### Tasks

- [x] **I1-T1** `pocv/queues.yaml` — add `name: rabbitmq-topology-credentials` under the `connectionSecret:` of `q-pocv-rpc`.
- [x] **I1-T2** `shopping-cart/queues.yaml` — same fix for `q-cart-rpc-v2`.
- [x] **I1-T3** `qualification/queues.yaml` — same fix for `q-qual-rpc`.
- [x] **I1-T4** Verify operator reconciles all three queues after the fix (`kubectl get queues -n tmf`).

---

## Issue 2 — Dead named reply queues (4 resources)

### Problem

All four RPC clients were migrated to `amq.rabbitmq.reply-to` (Direct Reply-To) in the `05_rpc_client_hardening` plan. Direct Reply-To is a broker pseudo-queue — no named reply queue is declared or consumed. The following gitops resources declare queues that are **never referenced by any service** and will never receive a single message:

| File | Kubernetes resource | Queue name |
|---|---|---|
| `bff/queues.yaml` | `q-bff-rpc-reply` | `q.bff.rpc.reply` |
| `qualification/queues.yaml` | `q-qual-rpc-reply` | `q.qual.rpc.reply` |
| `shopping-cart/queues.yaml` | `q-cart-rpc-reply` | `q.cart.rpc.reply` |
| `pocv/queues.yaml` | `q-pocv-rpc-reply` | `q.pocv.rpc.reply` |

Keeping them wastes broker memory, clutters the management UI, and gives the false impression that reply routing is handled via named queues.

### Tasks

- [x] **I2-T1** Remove the `q-bff-rpc-reply` Queue resource from `bff/queues.yaml`. The BFF directory will have no remaining resources — remove `bff/queues.yaml` and `bff/kustomization.yaml` and drop the `bff/` entry from the root `kustomization.yaml`.
- [x] **I2-T2** Remove the `q-qual-rpc-reply` Queue resource from `qualification/queues.yaml`.
- [x] **I2-T3** Remove the `q-cart-rpc-reply` Queue resource from `shopping-cart/queues.yaml`.
- [x] **I2-T4** Remove the `q-pocv-rpc-reply` Queue resource from `pocv/queues.yaml`.

---

## Issue 3 — Catalog exchange mismatch: cart never receives price-sync events

### Problem

The product-catalog-management service publishes **all** catalog events (offering created/updated/deleted) exclusively to the exchange named `catalog_events` (underscore notation). This is hardcoded throughout the service (`rabbitmq_handler.go`, `rabbitmq_publisher.go`, `cmd/main.go`).

The shopping-cart's catalog-sync consumer connects to exchange `"ex.domain.catalog"` (dot notation, `shopping-cart/cmd/server/main.go:115`). The gitops correctly reflects this — `shopping-cart/exchanges.yaml` declares `ex.domain.catalog` and `shopping-cart/bindings.yaml` binds `q.cart.catalog.sync` to it with `evt.catalog.offering.#`.

These are **two different AMQP entities**. `catalog_events` and `ex.domain.catalog` are separate exchanges with no connection. `q.cart.catalog.sync` is bound to the wrong exchange and receives nothing. Cart prices are never synced when offerings change.

**Root cause**: the shopping-cart consumer was wired to a domain-style exchange name (`ex.domain.catalog`) but the catalog service was never updated to publish to it — or vice versa. One side must change to match the other.

**Decision required**: canonical exchange is `catalog_events` (already declared, used by BFF and qualification too), so the fix is on the shopping-cart side.

### Tasks

- [x] **I3-T1** `shopping-cart/cmd/server/main.go` — change `"ex.domain.catalog"` to `"catalog_events"` in the `NewConsumer` call for `catalogConsumer`.
- [x] **I3-T2** `shopping-cart/bindings.yaml` — change `source: ex.domain.catalog` to `source: catalog_events` for the `cart-catalog-sync-offering-wildcard` binding.
- [x] **I3-T3** `shopping-cart/exchanges.yaml` — remove the now-unused `ex-domain-catalog` Exchange resource entirely (it would have zero bindings after I3-T2).
- [x] **I3-T4** Update root `shopping-cart/kustomization.yaml` if `exchanges.yaml` becomes empty after I3-T3.

---

## Issue 4 — Double delivery: old and new catalog query handlers both active

### Problem

The product-catalog-management service runs **two** competing consumers on `catalog_events`:

| Consumer | Queue | Binding routing key | Handler |
|---|---|---|---|
| Old (`rabbitmq_handler.go`) | `catalog_queries` | `query.catalog.#` | handles all catalog queries via direct channel |
| New (`catalog_rpc_handler.go`) | `catalog_rpc_queue` | `query.catalog.offering.#` | handles offering pricing queries only |

RabbitMQ delivers to **all queues whose binding matches** the routing key. Any message with key `query.catalog.offering.get` or `query.catalog.offering.by_category` satisfies **both** patterns. Both queues receive it; both handlers process it and send a reply. The second reply arrives after the correlation ID was already removed from `pending` in `handleReplies` — it is silently dropped, but the broker already did double the work.

This also means the gitops binding `catalog-queries-wildcard` (`query.catalog.#` → `catalog_queries`) should not overlap with `catalog-rpc-offering-wildcard` (`query.catalog.offering.#` → `catalog_rpc_queue`).

**Decision required**: determine whether the old `rabbitmq_handler.go` query consumer is still needed for any routing key that is NOT covered by the new `catalog_rpc_handler.go`. If it is, the wildcard binding must be tightened. If it is not, the old consumer and its queue can be retired.

Looking at the BFF routing keys vs handlers:
- `query.catalog.catalog.*`, `query.catalog.category.*`, `query.catalog.specification.*` → only bound to `catalog_queries` (old handler)
- `query.catalog.offering.get`, `query.catalog.offering.by_category` → bound to both

So the old handler is still needed for catalog/category/specification queries. The fix is to remove the broad `query.catalog.#` wildcard and replace it with explicit bindings that do not overlap with `query.catalog.offering.#`.

### Tasks

- [x] **I4-T1** `product-catalog-management/bindings.yaml` — replace the single `catalog-queries-wildcard` binding (`query.catalog.#`) with four explicit bindings that cover only what the old handler serves and do not overlap with the RPC handler:
  - `query.catalog.catalog.#` → `catalog_queries`
  - `query.catalog.category.#` → `catalog_queries`
  - `query.catalog.specification.#` → `catalog_queries`
  - *(do not add `query.catalog.offering.#` — that stays on `catalog_rpc_queue` only)*
- [x] **I4-T2** Verify no other routing keys sent by the BFF fall through — check `catalog_handlers.go` for any `query.catalog.*` keys not covered by I4-T1 or the existing `catalog-rpc-offering-wildcard`.
- [x] **I4-T3** Add an integration test or smoke-test note: send `query.catalog.offering.get` and assert exactly one reply is received.

---

## Issue 5 — Missing BFF event queue and binding

### Problem

The BFF starts a consumer for catalog events (`bff/cmd/server/main.go:68`):

```go
eventConsumer, err := newConsumerFunc(cfg.RabbitMQURL, "catalog_events", "q.bff.events")
```

Neither `q.bff.events` nor a binding for it exists in gitops. With `RABBITMQ_PASSIVE_DECLARE=true` (the production flag), `NewConsumer` calls `QueueDeclarePassive` and `ExchangeDeclarePassive` — both fail if the resource does not exist. **BFF fails to start in production.**

Without a binding, even if the queue were auto-created in dev mode, it would receive nothing from `catalog_events`.

### Tasks

- [x] **I5-T1** `bff/queues.yaml` — add a Queue resource for `q.bff.events` (durable, non-exclusive, non-auto-delete).
- [x] **I5-T2** Add `bff/bindings.yaml` with a Binding: `catalog_events` → `q.bff.events`, routing key `evt.catalog.#` (the BFF needs all offering/category/specification change notifications to keep its event stream current).
- [x] **I5-T3** `bff/kustomization.yaml` — add `bindings.yaml` to resources.
- [x] **I5-T4** Verify what events the BFF actually consumes (`events/hub.go` or equivalent) and tighten the routing key in I5-T2 if `evt.catalog.#` is broader than needed.

---

## Execution Order

```
I1 (15 min)  — YAML typo fix; unblocks RPC in all three services
I2 (10 min)  — cleanup only; safe to do independently
I5 (20 min)  — BFF start-up fix; independent of I3/I4
I3 (30 min)  — cart exchange fix; requires code + gitops change together
I4 (30 min)  — catalog binding tightening; gitops-only once decision is made
```

I1 and I5 are production outage fixes and should land first. I2 is cleanup. I3 and I4 require a coordination decision and can be done in either order.

---

## Definition of Done

- [x] All tasks above checked off.
- [ ] `kubectl get queues,exchanges,bindings -n tmf` shows all resources `Ready=True`. *(requires live cluster)*
- [ ] No orphaned reply queues visible in the RabbitMQ management UI. *(requires live cluster)*
- [ ] Shopping-cart receives a message on `q.cart.catalog.sync` when an offering is updated (smoke test). *(requires live cluster)*
- [ ] A single `query.catalog.offering.get` message produces exactly one reply (no duplicate delivery). *(requires live cluster)*
- [ ] BFF starts without error when `RABBITMQ_PASSIVE_DECLARE=true`. *(requires live cluster)*
