# ADR 001: Use a Lambda architecture for Brightwave's platform

## Status
Accepted

Date: 2026-11-21

## Context
Launch customer: **Brightwave Electronics (SHP03)**. It flies new devices by air for launch days announced months ahead, and a missed launch makes the news.

Targets:
- **Freshness:** 5 minutes
- **Reliability:** 98% on time on every lane
- **Budget:** $35 per month

Lanes: PVG → ORD (Transpacific), DXB → FRA (Middle East-Europe), SIN → LHR (Asia-Europe).

Evidence (`docs/metrics.md`, October 2026): all three lanes were **100% on time** (58, 70 and 61 shipments), with average delays of 1.8 h, 1.3 h and 0.7 h. Carriers: Meridian Air Cargo, Skybridge Cargo, Aquila Air Freight and Falcon Cargo, all 100% on time.

The data arrives in two different ways. Shipment tracking events are a continuous stream. Rates, bookings and schedules arrive as **daily files**, which is how carriers send them. The problem is to catch the rare late flight within 5 minutes, and keep reporting on carrier reliability, while staying inside $35 per month.

## Options considered
1. **Lambda:** a speed layer (Pub/Sub → Dataflow → BigQuery live) for shipment events, plus a batch layer (Cloud Storage → BigQuery curated) for the daily files. Each kind of data is processed the way it arrives.
2. **Kappa:** everything goes through one streaming pipeline, including the daily files, which would be replayed as events. There's one codebase, but daily files have to be turned into a stream, and the streaming job is the only path for all data.

## Decision
We chose **Lambda**, because Brightwave's data really does arrive in two forms. Only shipment events need the 5-minute freshness target. Rates and the delay-per-lane metric come from daily files and are needed for a monthly reliability trend, not minute-by-minute. A cheap daily batch load into BigQuery fits them better than forcing them through a streaming job. The evidence supports this: reliability is already at 100%, so the live layer only has to catch the rare exception quickly, and the batch layer can stay simple and cheap.

## Consequences
- **Cost:** the batch layer is nearly free at about 2 MB a month (Cloud Storage plus BigQuery loads and queries in the free tier). The main cost is the always-on streaming path. A Dataflow streaming job running all month with even one small worker may cost about as much as the whole $35 budget, so it is the main budget risk. If the cost check shows it is too high, we will replace Dataflow with a Pub/Sub BigQuery subscription that writes events straight into the live table. Kafka and Dataproc were removed from the reference for the same reason.
- **Freshness:** shipment events reach BigQuery live within seconds to minutes, which meets the 5-minute target. Rates and delay per lane are up to a day old, which is acceptable for a monthly reliability trend.
- **Complexity:** two pipelines to build, run and monitor instead of one, plus the new late-flight alert.
- **What gets harder:** the same logic, such as "what counts as late", can exist in two places (the streaming job and the batch SQL), and the two must be kept consistent. If the two layers disagree, the ops view and the exec view can show different numbers.

## Cost check (stretch)

Rough monthly estimate, box by box, from Google Cloud's published prices (November 2026). Brightwave's data volume: about 190 shipments a month on 3 lanes, about 2 MB of batch files.

| Box | Layer | How it's billed | Estimate per month |
|---|---|---|---|
| Pub/Sub | Live | Throughput; a few MB of tracking events | < $0.01 |
| Dataflow (1 vCPU, 3.75 GB worker, 730 h) | Live | Per second, always on: $0.056/vCPU-h + $0.003557/GB-h | ≈ $51 (lower bound) |
| BigQuery live + curated storage | Both | First 10 GiB free | $0 |
| BigQuery queries (ops view, exec view, delay per lane, alert check) | Both | First 1 TiB scanned free | $0 |
| Cloud Storage (raw files) | Batch | About 2 MB stored | < $0.01 |
| Late-flight alert (scheduled query every 5 min) | Live | Covered by the BigQuery free tier | $0 |
| Ops view and exec view (Data Studio) | Both | Free | $0 |
| **Total as drawn** | | | **≈ $51: over the $35 budget** |

**Does the design fit the budget?** No. The always-on Dataflow worker alone costs more than Brightwave's $35 per month, and every other box is close to free.

**What we would change:** replace Dataflow with a **Pub/Sub BigQuery subscription**, which writes events straight into the live table. It is billed at $50 per TiB of throughput, so Brightwave's few MB a month cost under a cent, and there are no BigQuery ingestion charges. The new total is **under $1 per month**, well inside the budget. Events still reach BigQuery within seconds, so the 5-minute freshness target is still met. This is still a Lambda architecture: the speed layer just becomes Pub/Sub → BigQuery live.

**Trade-off:** without Dataflow we can't transform events in flight (windowing, deduplication, enrichment). The late-flight check moves into a scheduled BigQuery query instead. If event volume or logic grows, we revisit this in a new ADR.