---
name: kafka-troubleshooting
description: Debug, diagnose, and give best-practice recommendations for Amazon MSK (Managed Streaming for Apache Kafka), both provisioned and Serverless. Use whenever the user is investigating an MSK or Kafka problem — consumer lag, under-replicated or offline partitions, disk filling up, broker CPU, throttling, connection failures, auth/TLS errors, rebalancing, slow producers/consumers — or wants a cluster reviewed against Kafka/MSK best practices (sizing, replication, retention, monitoring, security, client tuning). Trigger even when the user just names a symptom on an MSK cluster ("brokers keep going unhealthy", "why is my consumer group lagging", "producers getting NotEnoughReplicas") without saying the words "debug" or "MSK". Works from live AWS access (AWS CLI/boto3, CloudWatch, Kafka admin tools) and stays read-only unless the user explicitly approves a change.
---

# Kafka Troubleshooting

Diagnose and advise on Amazon MSK clusters. Two things this skill does: **debug live problems**, and **review a cluster against best practices**. Both lean on the same evidence — cluster metadata, CloudWatch metrics, and Kafka admin output — and both end in a short, skimmable answer the user can act on.

This file is self-contained. The top half is the workflow (what to do); the bottom half is lookup material you consult as needed — a metric catalog, symptom playbooks, a command cookbook, and best practices. Jump to the relevant section rather than reading it all top to bottom.

## The one rule that matters most: read before you write

MSK clusters are usually production. Reading state is safe; changing it is not. So:

- **Freely run read-only commands** — `describe-*`, `list-*`, `get-*`, CloudWatch reads, and Kafka `--describe` calls. These can't hurt anything, so don't ask permission for them.
- **Never run a mutating command on a live cluster on your own initiative.** That means anything that reassigns partitions, alters configs, changes topic settings, creates/deletes topics, reboots brokers, updates cluster config, or scales storage/brokers. Instead, put the exact command in front of the user as a recommendation, explain what it does and its blast radius, and wait for an explicit yes.

The reason isn't bureaucracy — a partition reassignment or config change on a busy cluster can move a lot of data or drop a lot of messages, and the user owns that decision. Your job is to make the right action obvious and safe to run, not to run it for them. The **Command cookbook** below tags every command read-only `[R]` or mutating `[M]` so you never have to guess.

## How to run a debug session

Work like a good on-call engineer: start from the symptom, form a small number of hypotheses, then pull just enough evidence to confirm or kill each one. Don't dump every metric — pull the ones that discriminate between your hypotheses.

1. **Pin down the symptom and the cluster.** What's actually wrong, since when, and which cluster? Get the cluster ARN (or list clusters) and note whether it's **provisioned or Serverless** — the two debug very differently, so branch early.

2. **Get the lay of the land.** For provisioned: broker count, instance type, AZs, Kafka version, storage, monitoring level, auth. For Serverless: throughput usage against quotas, partition count, IAM. The Command cookbook has the exact calls.

3. **Form hypotheses from the symptom.** Find the matching entry in **Symptom playbooks** below. It lists the usual causes in rough order of likelihood, the metrics/commands that distinguish them, and the fix direction. Use it to aim your evidence-gathering, not as a script to run top to bottom.

4. **Pull the discriminating evidence.** Use the **Metrics** section to pick the right CloudWatch metrics and know their healthy ranges — a value only means something against its baseline. Check whether the cluster's monitoring level even exposes the metric you want (some live only at `PER_TOPIC_PER_BROKER` or finer); if it doesn't, say so and note what enabling it would show.

5. **Land on a root cause and a fix.** State what the evidence supports, how confident you are, and what to do about it. If the fix is a mutating command, present it per the read-before-you-write rule above.

## How to run a best-practice review

When the user wants a cluster reviewed rather than a fire put out, gather the same metadata plus current config, then work through the **Best practices** section — it's organized so you can check sizing, HA/replication, storage/retention, monitoring, security, and client config in turn, with provisioned and Serverless called out separately. Report only the gaps that matter, worst-first, each with the concrete change and why it's worth making. Skip the areas that are already fine rather than padding the report to look thorough.

## Anything version- or limit-dependent: verify, don't recite

MSK broker instance limits (partitions per broker, throughput ceilings), Serverless quotas, default port numbers per auth type, and supported Kafka versions all change over time and vary by region and instance type. When a recommendation depends on a specific number or limit, check it against the current cluster (`describe-cluster-v2`, `describe-configuration`) or current AWS documentation rather than quoting a figure from memory — a confidently stated stale limit is worse than saying "let me confirm the current ceiling for this instance type." The material below gives you the right metric, command, and direction; it deliberately avoids hardcoding numbers that drift.

## Output: short, ranked, and actionable

The user wants signal, not a transcript. Structure a debug answer as:

- **What's wrong** — one or two lines, plainly.
- **Why** — the root cause, with the specific evidence that points to it (the metric value, the describe output), and your confidence.
- **Fix** — the concrete step(s), most impactful first. Mark anything that changes the cluster as needing their approval, with the exact command ready to run.
- **Watch** — if useful, what to keep an eye on to confirm recovery or catch recurrence.

Fold in the commands you actually ran so the user can re-run them, but keep the prose lean — no restating metric definitions they didn't ask for, no narrating every step you took. If the picture is genuinely ambiguous, say what you'd need (a metric that isn't enabled, a client-side log) rather than forcing a false-confident answer.

---

# Metrics — CloudWatch `AWS/Kafka` namespace

Metrics only mean something against a baseline and against the cluster's monitoring level. Use this to pick the metric that discriminates between your hypotheses and to know what "bad" looks like.

## Monitoring levels — check this first

MSK emits metrics at one of four levels, set per cluster: `DEFAULT`, `PER_BROKER`, `PER_TOPIC_PER_BROKER`, `PER_TOPIC_PER_PARTITION`. Higher levels expose more granular metrics but cost more. If a metric you want isn't showing up, the cluster is probably below the level that emits it — confirm with `describe-cluster-v2` (`EnhancedMonitoring` field). Note this to the user rather than assuming the metric is zero. **Serverless** exposes a much smaller set (no broker CPU/disk — those are managed away); debug it from throughput, partition, and client-side signals instead.

Dimensions available depend on level: `Cluster Name` always; `Broker ID` at `PER_BROKER`+; `Topic` at `PER_TOPIC_PER_BROKER`+.

## Cluster-health metrics — the "is it on fire" set

Check these first on almost any provisioned-cluster incident.

| Metric | Healthy | What a bad value means |
|---|---|---|
| `ActiveControllerCount` | sums to **exactly 1** across brokers | 0 = no controller (cluster can't do metadata ops); >1 = split brain. Either is urgent. |
| `OfflinePartitionsCount` | **0** | Partitions with no leader = data unavailable for those partitions. Urgent, user-visible. |
| `UnderReplicatedPartitions` | **0** | Replicas falling behind leaders — usually a broker down, overloaded, or network-bound. Durability risk. |
| `UnderMinIsrPartitionCount` | **0** | ISR below `min.insync.replicas`; producers with `acks=all` get `NotEnoughReplicas` and fail. |
| `KafkaDataLogsDiskUsed` | comfortably **< ~70–85%** | Approaching 100% makes brokers read-only then unhealthy. This is the single most common cause of MSK broker failure. |

## Saturation — the "why is it slow" set

| Metric | Reading | Signals |
|---|---|---|
| `CpuUser` + `CpuSystem` | keep total **under ~60%** | Above that you have no headroom for rolling patches (which take a broker out) or spikes. Sustained high CPU = under-provisioned or hot brokers. |
| `RequestHandlerAvgIdlePercent` | 0–1; **higher is better** | Low (approaching 0) = request handler threads saturated; the broker can't keep up with request volume. |
| `NetworkProcessorAvgIdlePercent` | 0–1; **higher is better** | Low = network threads saturated; often connection churn or large payloads. |
| `RootDiskUsed` | low | Distinct from data-logs disk; high root disk is rarer but breaks the broker OS. |
| `MemoryUsed` / heap-related | stable, no steady climb | Steady climb or GC pressure shows up as latency spikes. |

## Throughput

| Metric | Use |
|---|---|
| `BytesInPerSec` / `BytesOutPerSec` | Ingress/egress per broker or topic. Compare across brokers to spot hot brokers / partition skew. |
| `MessagesInPerSec` | Message rate; pair with BytesIn to reason about message size. |
| `ReplicationBytesInPerSec` / `ReplicationBytesOutPerSec` | Replication traffic; a spike often accompanies under-replication recovery or reassignment. |
| `PartitionCount` (per broker) / `GlobalPartitionCount` | Too many partitions per broker hurts recovery time and controller load — check against the instance type's guidance. |
| `LeaderCount` (per broker) | Uneven leader counts = leader imbalance; a few brokers doing most of the work. |

## Latency

| Metric | Use |
|---|---|
| `ProduceTotalTimeMsMean` | End-to-end produce latency. Rising = broker-side pressure (disk, replication, CPU). |
| `FetchConsumerTotalTimeMsMean` | Consumer fetch latency. |
| `RequestThrottleTime` / throttle metrics | Non-zero = quotas are throttling clients (client or user quotas configured on the cluster). |

## Consumer lag

Lag metrics require MSK's consumer-lag monitoring to be enabled; if absent, fall back to `kafka-consumer-groups.sh --describe` (see Command cookbook).

| Metric | Use |
|---|---|
| `SumOffsetLag` (per group) | Total messages behind across the group's partitions. |
| `MaxOffsetLag` | Worst single partition — isolates a stuck consumer or a hot partition. |
| `EstimatedMaxTimeLag` / `EstimatedTimeLag` | Lag expressed as time, which is usually what SLAs care about. |

## Connections

| Metric | Use |
|---|---|
| `ConnectionCount` / `ClientConnectionCount` | Baseline vs spike. Connection storms saturate network threads and can look like a broker problem. |
| `TcpConnections` | OS-level connection count per broker. |

## Reading metrics well

- **Compare across brokers**, not just against a threshold — one broker hot while others idle points to skew or a bad node, not global overload.
- **Look at the trend**, not the instant — a single spike differs from a sustained climb.
- **Correlate in time** — under-replication that starts exactly when disk crossed 85% or when a broker rebooted tells you the cause directly.
- Pull metrics with `get-metric-data` (see Command cookbook) using `Average` or `Maximum` as fits; for the health set, `Maximum` catches transient badness that `Average` hides.

---

# Symptom playbooks

Each playbook: the usual causes in rough likelihood order, the evidence that distinguishes them, and the fix direction. Use it to aim your investigation — confirm or kill causes with evidence, don't just run down the list.

## Brokers going unhealthy / cluster degraded

**Most common cause by far: disk full.** Check `KafkaDataLogsDiskUsed` first — as it approaches 100% brokers go read-only then unhealthy.
- **Confirm:** disk-used metric per broker; `kafka-log-dirs.sh` to see what's consuming space; retention settings and any topic with huge/growing partitions.
- **Fix direction:** short term, expand EBS storage (mutating — propose it) or let storage auto-scaling do it if enabled; reduce retention on the offending topics; longer term, enable **storage auto-scaling** and/or **tiered storage**, and alarm on disk at ~70–85%.

**Next: a broker actually down or restarting.** `ActiveControllerCount` ≠ 1, or `UnderReplicatedPartitions` > 0 concentrated on one broker.
- **Confirm:** `list-nodes`, `describe-cluster-operation-v2` (is a rolling update / patch in flight?), broker-level metrics for the suspect node.
- **Fix direction:** if it's an in-progress managed operation, wait it out; if a broker is genuinely stuck, a reboot-broker is mutating — propose it.

**Also consider:** CPU sustained near 100% (under-provisioned), or a recent Kafka-version/config change that correlates in time.

## Under-replicated / under-min-ISR partitions

`UnderReplicatedPartitions` > 0 means replicas can't keep up with leaders; `UnderMinIsrPartitionCount` > 0 means `acks=all` producers are already failing.
- **Broker down or overloaded** — check per-broker CPU, disk, and whether under-replication clusters on one broker. → relieve load / recover the broker.
- **Replication can't keep up with ingress** — `ReplicationBytesInPerSec` high, ingress spiking. → the cluster is under-provisioned for the write rate; scale brokers/instance type (propose it) or shed load.
- **Network saturation** — `NetworkProcessorAvgIdlePercent` low. → connection storm or oversized payloads.
- **Misconfigured replication** — replication factor or `min.insync.replicas` set such that normal broker maintenance breaks ISR. → see Best practices (RF=3, minISR=2 on a 3-AZ cluster).

## Consumer group lagging

- **Consumers too slow / too few** — `MaxOffsetLag` spread evenly, all partitions behind. → scale consumers up to (but not beyond) the partition count; profile consumer processing time.
- **One stuck partition / hot key** — `MaxOffsetLag` on one partition while others are fine. → investigate that partition's key distribution or a poisoned message wedging the consumer.
- **Frequent rebalances** — group keeps rebalancing (client logs; `--describe --members`). → tune `max.poll.interval.ms`, `session.timeout.ms`, `heartbeat.interval.ms`; long processing between polls is the usual trigger.
- **Broker-side slowness** — `FetchConsumerTotalTimeMsMean` rising. → back to broker saturation/disk.
- **Confirm current state:** `kafka-consumer-groups.sh --describe --group <g>` shows per-partition lag, current offset, and assigned member.

## Producers failing or throttled

- **`NotEnoughReplicas` / `NotEnoughReplicasAfterAppend`** — ISR below minISR. → same as under-min-ISR above; the durability guarantee is doing its job by refusing the write.
- **Throttling** — `RequestThrottleTime` non-zero. → a client/user quota is set; either the client exceeds it or the quota is too tight for legitimate load.
- **Timeouts / high produce latency** — `ProduceTotalTimeMsMean` high. → broker saturation (disk, CPU, replication).
- **`acks`/`retries` misconfig on the client** — producer errors with a healthy cluster. → client tuning, see Best practices.

## Can't connect / auth failures

Almost always networking or auth-method mismatch, not the cluster being down.

- **Wrong bootstrap string for the auth method** — IAM, TLS, and SASL/SCRAM each have their own bootstrap brokers and port. → `get-bootstrap-brokers` returns all of them; match the string and port to the client's auth.
- **Port blocked** — security group / NACL / routing between client and broker subnets. → common ports: 9092 plaintext (often disabled), 9094 TLS, 9096 SASL/SCRAM, 9098 IAM (verify current values). Check the SG on the cluster ENIs allows the client's source on the right port.
- **IAM auth** — client role lacks `kafka-cluster:*` permissions, or wrong signing region. → check the IAM policy and that the client uses the IAM SASL mechanism over `SASL_SSL`.
- **SASL/SCRAM** — secret not associated with cluster, or not encrypted with a customer-managed KMS key (required). → `list-scram-secrets`; verify the secret's KMS key.
- **mTLS** — client cert not trusted / expired; wrong CA. → check the ACM PCA and the client keystore.
- **DNS / VPC** — client can't resolve broker DNS (cross-VPC without proper DNS, or no route). → resolve the broker hostname from the client's network.

## Serverless-specific

Serverless hides brokers, CPU, and disk, so debugging shifts to quotas, partitions, and clients.

- **Throttling against per-partition throughput quotas** — traffic hitting the documented ingress/egress-per-partition ceiling. → add partitions to spread load, or smooth producer bursts. Verify current quota numbers from AWS docs rather than assuming.
- **Partition-count limits** — cluster or per-topic partition ceilings. → consolidate topics or request a limit increase.
- **IAM only** — Serverless supports IAM auth only; SASL/mTLS won't apply. → check client IAM setup as above.
- **No broker metrics** — if the user asks for CPU/disk on Serverless, explain those are managed and point them at throughput, throttling, and consumer-lag signals instead.

## Leader imbalance / uneven load

`LeaderCount` uneven across brokers, or a few brokers with much higher `BytesIn/Out`.
- **Confirm:** per-broker leader count and throughput; `kafka-topics.sh --describe` for partition/leader placement.
- **Fix direction:** trigger preferred-leader election, or reassign partitions to rebalance (both mutating — propose with the exact plan; reassignment moves data, so schedule for low-traffic windows and throttle it).

---

# Command cookbook

Every command is tagged **[R]** read-only (safe, run freely) or **[M]** mutating (needs explicit user approval — present it, don't run it). Placeholders in `<angle brackets>`.

## AWS CLI — cluster metadata (all read-only)

```bash
# [R] List clusters (covers provisioned + Serverless)
aws kafka list-clusters-v2 --region <region>

# [R] Full cluster detail: type, brokers, AZs, version, monitoring level, auth, storage
aws kafka describe-cluster-v2 --cluster-arn <arn>

# [R] Is a managed operation (patch, scale, config update) in flight or recent?
aws kafka list-cluster-operations-v2 --cluster-arn <arn>
aws kafka describe-cluster-operation-v2 --cluster-operation-arn <op-arn>

# [R] Broker nodes and their roles
aws kafka list-nodes --cluster-arn <arn>

# [R] Bootstrap brokers — returns a string per auth method (TLS, SASL/SCRAM, IAM, plaintext)
aws kafka get-bootstrap-brokers --cluster-arn <arn>

# [R] Cluster configuration (server.properties-style settings)
aws kafka describe-configuration --arn <config-arn>
aws kafka describe-configuration-revision --arn <config-arn> --revision <n>

# [R] SASL/SCRAM secrets associated with the cluster
aws kafka list-scram-secrets --cluster-arn <arn>
```

## AWS CLI — CloudWatch metrics (read-only)

Prefer `get-metric-data` for pulling several metrics at once. Namespace is `AWS/Kafka`; dimension `Cluster Name` always applies, add `Broker ID` / `Topic` when the monitoring level supports it.

```bash
# [R] One metric, quick look (get-metric-statistics)
aws cloudwatch get-metric-statistics \
  --namespace AWS/Kafka \
  --metric-name KafkaDataLogsDiskUsed \
  --dimensions Name="Cluster Name",Value=<cluster-name> Name="Broker ID",Value=<n> \
  --start-time <ISO8601> --end-time <ISO8601> \
  --period 300 --statistics Maximum
```

For a health snapshot, pull this set with `get-metric-data` and read against the Metrics section:
`ActiveControllerCount`, `OfflinePartitionsCount`, `UnderReplicatedPartitions`, `UnderMinIsrPartitionCount`, `KafkaDataLogsDiskUsed`, `CpuUser`, `CpuSystem`, `RequestHandlerAvgIdlePercent`. Use `Maximum` for the health set (catches transient badness), `Average` for throughput/latency trends.

## Kafka admin CLI — run from a client inside the VPC

These need a Kafka client host with network + auth to the brokers, and a `client.properties` matching the cluster's auth (TLS/SASL/IAM). Get the bootstrap string from `get-bootstrap-brokers`.

```bash
BS=<bootstrap-string-for-this-auth>
CFG=--command-config client.properties   # omit only if plaintext

# [R] Topic layout: partitions, replicas, ISR, leaders
kafka-topics.sh --bootstrap-server $BS $CFG --describe
kafka-topics.sh --bootstrap-server $BS $CFG --describe --topic <topic>

# [R] Under-replicated / unavailable partitions, fast
kafka-topics.sh --bootstrap-server $BS $CFG --describe --under-replicated-partitions
kafka-topics.sh --bootstrap-server $BS $CFG --describe --unavailable-partitions

# [R] Consumer group lag, offsets, and member assignment
kafka-consumer-groups.sh --bootstrap-server $BS $CFG --describe --group <group>
kafka-consumer-groups.sh --bootstrap-server $BS $CFG --describe --group <group> --members --verbose
kafka-consumer-groups.sh --bootstrap-server $BS $CFG --list

# [R] Effective configs for a topic / broker
kafka-configs.sh --bootstrap-server $BS $CFG --describe --entity-type topics --entity-name <topic>
kafka-configs.sh --bootstrap-server $BS $CFG --describe --entity-type brokers --entity-name <broker-id>

# [R] Disk usage per log dir / partition — pinpoints what's filling a broker
kafka-log-dirs.sh --bootstrap-server $BS $CFG --describe --broker-list <ids>
```

## Mutating commands — present, don't run

Show these to the user with what they do and their blast radius, and run only on an explicit yes.

```bash
# [M] Expand broker storage (EBS). Can't be reversed/shrunk. Propose a target size.
aws kafka update-broker-storage --cluster-arn <arn> \
  --current-version <ver> --target-broker-ebs-volume-info '...'

# [M] Update cluster to a new configuration revision (triggers a rolling change)
aws kafka update-cluster-configuration --cluster-arn <arn> \
  --configuration-info '...' --current-version <ver>

# [M] Scale brokers / change instance type (rolling; capacity + cost impact)
aws kafka update-broker-count --cluster-arn <arn> --current-version <ver> --target-number-of-broker-nodes <n>
aws kafka update-broker-type  --cluster-arn <arn> --current-version <ver> --target-instance-type <type>

# [M] Reboot a broker (takes it out of service briefly; partitions it leads fail over)
aws kafka reboot-broker --cluster-arn <arn> --broker-ids <id>

# [M] Alter a topic/broker config live
kafka-configs.sh --bootstrap-server $BS $CFG --alter --entity-type topics --entity-name <topic> --add-config <k=v>

# [M] Preferred-leader election — rebalances leadership
kafka-leader-election.sh --bootstrap-server $BS $CFG --election-type preferred --all-topic-partitions

# [M] Partition reassignment — MOVES DATA. Throttle it and run in a low-traffic window.
kafka-reassign-partitions.sh --bootstrap-server $BS $CFG --reassignment-json-file plan.json --execute --throttle <bytes/s>

# [M] Create / delete topics
kafka-topics.sh --bootstrap-server $BS $CFG --create --topic <t> --partitions <p> --replication-factor 3
kafka-topics.sh --bootstrap-server $BS $CFG --delete --topic <t>
```

Note: `describe-cluster-v2` returns `CurrentVersion` — most mutating `update-*` calls require the current version string, so read it first.

---

# Best practices

For a best-practice review, work these areas in turn and report only the gaps that matter, worst-first, each with the concrete change and the reason. Provisioned and Serverless differ — the callouts say which applies. Where a recommendation depends on a number that drifts (instance limits, quotas, versions), verify it against the live cluster or current AWS docs rather than hardcoding.

## High availability & replication

- **Three AZs.** Run brokers across 3 Availability Zones so a single-AZ failure is survivable. (Serverless is multi-AZ by design.)
- **Replication factor 3.** Every production topic should have RF=3. RF=1 or 2 risks data loss on broker failure.
- **`min.insync.replicas = 2`** with RF=3. This lets one replica be down while `acks=all` writes still succeed — the sweet spot between durability and availability. minISR=3 with RF=3 means any single broker maintenance blocks writes; minISR=1 gives up durability.
- **Producers use `acks=all`** for data you can't lose. `acks=1` or `0` trades durability for latency — fine for some telemetry, wrong for anything of record.
- These three (RF=3, minISR=2, acks=all) work as a set; a review should check they're consistent, not just individually present.

## Sizing & capacity (provisioned)

- **Keep total CPU (`CpuUser`+`CpuSystem`) under ~60%.** Headroom matters because rolling patches remove a broker at a time, and the survivors must absorb its load without tipping over.
- **Right-size the instance type** to throughput; newer Graviton (`m7g`) types give better price/performance than older ones. Scaling up instance type or broker count are both rolling operations.
- **Watch partitions per broker.** Every instance type has a guideline ceiling; exceeding it slows recovery and leader election and loads the controller. Verify the current per-instance number rather than assuming.
- **Spread partitions and leaders evenly** so no broker is a hotspot (`LeaderCount`, per-broker `BytesIn/Out`).

## Storage & retention

- **Enable EBS storage auto-scaling** with a sensible target utilization — running out of disk is the top cause of MSK broker failure, and auto-scaling turns a 2am page into a non-event. Storage can grow but never shrink, so don't wildly over-provision either.
- **Alarm on `KafkaDataLogsDiskUsed`** at ~70–85% as a backstop even with auto-scaling.
- **Set retention deliberately** per topic (`retention.ms` / `retention.bytes`) to what the consumers actually need — over-long retention silently fills disk.
- **Use tiered storage** for topics that need long retention cheaply; it offloads older segments off broker disk (requires a supporting Kafka version).

## Monitoring & operations

- **Set the monitoring level to what you'll actually debug with** — `PER_TOPIC_PER_BROKER` is a reasonable default for a cluster you operate; `DEFAULT` is too coarse to diagnose most real issues, `PER_TOPIC_PER_PARTITION` costs more and is worth it only when you need partition-level granularity.
- **Enable consumer-lag monitoring** so lag is visible without shelling into a client.
- **Alarm on the cluster-health set**: `OfflinePartitionsCount` > 0, `UnderReplicatedPartitions` sustained > 0, `ActiveControllerCount` ≠ 1, disk high, CPU high. These are the ones that page.
- **Open Monitoring (Prometheus)** is available if the team wants JMX/node metrics in their own stack (JMX exporter on 11001, node exporter on 11002 — verify current ports).
- **Keep the Kafka version current** and leave automatic minor-version upgrades enabled; running an unsupported version blocks features and fixes.

## Security

- **Encrypt in transit and at rest.** TLS for client-broker (and in-cluster) traffic; at-rest encryption with a KMS key — a customer-managed key if the org needs to control rotation/access.
- **Use a real auth method, not plaintext.** IAM access control is the simplest on AWS (no secret management, policies as code). SASL/SCRAM (secrets in Secrets Manager, encrypted with a customer-managed KMS key) or mTLS are the alternatives. Disable the plaintext listener.
- **Least-privilege client policies** — IAM roles scoped to the topics/groups a client needs (`kafka-cluster:` actions), not blanket access.
- **Lock down networking** — brokers in private subnets, security groups allowing only the client sources that need them on the right port.

## Client configuration

- **Producers:** enable compression (`compression.type`, e.g. `lz4`/`zstd`), batch (`linger.ms` + `batch.size`) to trade a little latency for a lot of throughput, set `enable.idempotence=true` to avoid duplicates on retry, and size `retries`/`delivery.timeout.ms` for your durability needs.
- **Consumers:** set `max.poll.records` and `max.poll.interval.ms` so slow processing doesn't trigger rebalances; commit offsets deliberately (after processing, for at-least-once).
- **Connections:** reuse clients/connections rather than reconnecting per request — connection storms saturate broker network threads and look like a broker fault.
- **Partition count** should match target consumer parallelism (a group can't have more active consumers than partitions), balanced against the cost of too many partitions cluster-wide.

## Serverless — what changes

- No broker/CPU/disk/storage decisions — AWS manages them. Sizing shifts to **partitions and throughput quotas**.
- **Design around per-partition throughput quotas**: partition topics enough to spread load under the ingress/egress-per-partition ceiling, and smooth bursty producers. Verify current quota values from AWS docs.
- **IAM auth only** — no SASL/SCRAM or mTLS to configure; put the effort into least-privilege IAM policies.
- **Mind partition-count limits** per cluster/topic; consolidate or request increases rather than sprawling topics.
- HA, encryption, and client tuning guidance above still applies.
