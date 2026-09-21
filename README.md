# AIOps Payment Service Assessment

## Scenario

This project monitors a synthetic `payment-service`. The operational problem is
that payment requests can become slow or fail when the service is overloaded or
when its database connection times out. AIOps is used here to inspect telemetry,
identify abnormal behaviour, and move actionable anomaly events through a small
event-processing workflow.

## Repository Components

- `data/service_data.json` contains the operational observations.
- `src/anomaly_detector.py` evaluates metrics and log severity against fixed
	thresholds and creates readable anomaly events.
- `src/event_producer.py` publishes events to an `EventTopic` in memory.
- `src/event_topic.py` simulates the event-streaming topic and message store.
- `src/event_consumer.py` receives messages from the topic.
- `src/aiops_pipeline.py` coordinates data loading, detection, publishing, and
	consumption. The consumed events are the downstream AIOps output.
- `src/calculations.py` and its tests are the unrelated starter exercise.

## Operational Data Analysis

Each record has an ISO-like timestamp, service name, three metrics, and log
information. `response_time_ms`, `cpu_percent`, and `memory_percent` are
metrics. `log_level` and `message` are log fields. Timestamps are ordered at
one-minute intervals and identify when each observation occurred.

The observations from 10:00 through 10:04 and 10:07 through 10:09 are normal:
response time is 120-150 ms, CPU is 42-50%, memory is 51-57%, and the log level
is `INFO`. The 10:05 observation is unusual because response time is 610 ms and
the log reports a payment timeout. The 10:06 observation is the most severe:
response time is 640 ms, CPU is 94%, memory is 91%, and the message reports a
database connection timeout.

## Detection Findings

The detector thresholds are response time over 500 ms, CPU over 80%, or memory
over 80%. `ERROR` log records are also treated as anomalous. It detected:

- `2026-09-20T10:05:00`: high response time and an error log (`Payment service
	timeout`).
- `2026-09-20T10:06:00`: high response time, high CPU, high memory, and an error
	log (`Database connection timeout`).

No expected anomaly was missed, and no normal record was flagged for this data.
The approach is threshold-based, so it may miss gradual degradation below a
threshold and does not learn service-specific baselines. A possible improvement
would be a historical baseline or adaptive thresholding combined with richer
log classification.

## Event Flow

The pipeline follows:

`Operational data -> AnomalyDetector -> Event -> EventProducer -> anomaly-events
EventTopic -> EventConsumer -> downstream AIOps result`

The producer publishes each non-empty anomaly event to the `anomaly-events`
topic. The consumer reads from that same topic and returns the processed events
as the final pipeline result. The event retains its timestamp, service, type,
reasons, and original source record, so the downstream result explains why it
was flagged.

## Issues Found and Corrected

1. The detector checked for `WARNING`, but the supplied concerning records use
	 `ERROR`. The check now identifies the data's actual error severity.
2. The producer used `service-events` while the consumer used a separate
	 `anomaly-events` topic instance. Both now share one `anomaly-events` topic.
3. Source modules used top-level imports, which failed when imported as the
	 `src` package by the tests. Imports now use package-relative paths, and the
	 module entry point is used for execution.

## Verified Execution

From the repository root:

```bash
python -m pytest -q
python -m src.aiops_pipeline
```

The final run processed 10 records, detected 2 anomalies, published 2 anomaly
events, consumed 2 events, and printed both payment-service incidents with
their timestamps and reasons. The test suite passes, including the anomaly,
producer, and consumer checks.

The repository is a fork: `origin` is the exercise copy and `upstream` is the
original repository. After validation, commit and push the changes with:

```bash
git add README.md src tests
git commit -m "Complete AIOps workflow assessment"
git push origin main
```

---

&copy; 2025 GitHub &bull; [Code of Conduct](https://www.contributor-covenant.org/version/2/1/code_of_conduct/code_of_conduct.md) &bull; [MIT License](https://gh.io/mit)

