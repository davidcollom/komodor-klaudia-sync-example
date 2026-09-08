---
name: kafka-troubleshooting
description: Debug, diagnose, and give best-practice recommendations for Amazon MSK (Managed Streaming for Apache Kafka), both provisioned and Serverless. Use whenever the user is investigating an MSK or Kafka problem — consumer lag, under-replicated or offline partitions, disk filling up, broker CPU, throttling, connection failures, auth/TLS errors, rebalancing, slow producers/consumers — or wants a cluster reviewed against Kafka/MSK best practices (sizing, replication, retention, monitoring, security, client tuning). Trigger even when the user just names a symptom on an MSK cluster ("brokers keep going unhealthy", "why is my consumer group lagging", "producers getting NotEnoughReplicas") without saying the words "debug" or "MSK". Works from live AWS access (AWS CLI/boto3, CloudWatch, Kafka admin tools) and stays read-only unless the user explicitly approves a change.
---

# MSK Doctor

Diagnose and advise on Amazon MSK clusters. Two things this skill does: **debug live problems**, and **review a cluster against best practices**. Both lean on the same evidence — cluster metadata, CloudWatch metrics, and Kafka admin output — and both end in a short, skimmable answer the user can act on.

## The one rule that matters most: read before you write

MSK clusters are usually production. Reading state is safe; changing it is not. So:

- **Freely run read-only commands** — `describe-*`, `list-*`, `get-*`, CloudWatch reads, and Kafka `--describe` calls. These can't hurt anything, so don't ask permission for them.
- **Never run a mutating command on a live cluster on your own initiative.** That means anything that reassigns partitions, alters configs, changes topic settings, creates/deletes topics, reboots brokers, updates cluster config, or scales storage/brokers. Instead, put the exact command in front of the user as a recommendation, explain what it does and its blast radius, and wait for an explicit yes.

The reason isn't bureaucracy — a partition reassignment or config change on a busy cluster can move a lot of data or drop a lot of messages, and the user owns that decision. Your job is to make the right action obvious and safe to run, not to run it for them.

`references/commands.md` marks every command read-only or mutating so you never have to guess.

## How to run a debug session

Work like a good on-call engineer: start from the symptom, form a small number of hypotheses, then pull just enough evidence to confirm or kill each one. Don't dump every metric — pull the ones that discriminate between your hypotheses.

1. **Pin down the symptom and the cluster.** What's actually wrong, since when, and which cluster? Get the cluster ARN (or list clusters with `describe-cluster-v2`/`list-clusters-v2`) and note whether it's **provisioned or Serverless** — the two debug very differently, so branch early.

2. **Get the lay of the land.** For provisioned: broker count, instance type, AZs, Kafka version, storage, monitoring level, auth. For Serverless: throughput usage against quotas, partition count, IAM. `references/commands.md` has the exact calls.

3. **Form hypotheses from the symptom.** Open `references/symptom-playbooks.md` and find the matching symptom. It lists the usual causes in rough order of likelihood, the metrics/commands that distinguish them, and the fix direction. Use it to aim your evidence-gathering, not as a script to run top to bottom.

4. **Pull the discriminating evidence.** Use `references/metrics.md` to pick the right CloudWatch metrics and know their healthy ranges — a value only means something against its baseline. Check whether the cluster's monitoring level even exposes the metric you want (some live only at `PER_TOPIC_PER_BROKER` or finer); if it doesn't, say so and note what enabling it would show.

5. **Land on a root cause and a fix.** State what the evidence supports, how confident you are, and what to do about it. If the fix is a mutating command, present it per the read-before-you-write rule above.

## How to run a best-practice review

When the user wants a cluster reviewed rather than a fire put out, gather the same metadata plus current config, then work through `references/best-practices.md` — it's organized so you can check sizing, HA/replication, storage/retention, monitoring, security, and client config in turn, with provisioned and Serverless called out separately. Report only the gaps that matter, worst-first, each with the concrete change and why it's worth making. Skip the areas that are already fine rather than padding the report to look thorough.

## Anything version- or limit-dependent: verify, don't recite

MSK broker instance limits (partitions per broker, throughput ceilings), Serverless quotas, default port numbers per auth type, and supported Kafka versions all change over time and vary by region and instance type. When a recommendation depends on a specific number or limit, check it against the current cluster (`describe-cluster-v2`, `describe-configuration`) or current AWS documentation rather than quoting a figure from memory — a confidently stated stale limit is worse than saying "let me confirm the current ceiling for this instance type." The references give you the right metric, command, and direction; they deliberately avoid hardcoding numbers that drift.

## Output: short, ranked, and actionable

The user wants signal, not a transcript. Structure a debug answer as:

- **What's wrong** — one or two lines, plainly.
- **Why** — the root cause, with the specific evidence that points to it (the metric value, the describe output), and your confidence.
- **Fix** — the concrete step(s), most impactful first. Mark anything that changes the cluster as needing their approval, with the exact command ready to run.
- **Watch** — if useful, what to keep an eye on to confirm recovery or catch recurrence.

Fold in the commands you actually ran so the user can re-run them, but keep the prose lean — no restating metric definitions they didn't ask for, no narrating every step you took. If the picture is genuinely ambiguous, say what you'd need (a metric that isn't enabled, a client-side log) rather than forcing a false-confident answer.

## Reference files

- `references/metrics.md` — CloudWatch `AWS/Kafka` metrics: what each means, healthy ranges, and which symptom it speaks to. Load when interpreting metrics.
- `references/symptom-playbooks.md` — Common MSK/Kafka symptoms mapped to causes, evidence, and fixes. Load at the start of a debug session.
- `references/commands.md` — AWS CLI + Kafka admin command cookbook, each tagged read-only or mutating. Load when you need the exact call.
- `references/best-practices.md` — MSK/Kafka best practices for sizing, HA, storage, monitoring, security, and clients. Load for a best-practice review or when a fix touches configuration.
