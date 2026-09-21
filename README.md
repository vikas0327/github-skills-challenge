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


# GitHub Challenge

<img src="https://octodex.github.com/images/Professortocat_v2.png" align="right" height="200px" />

Hey there!

Your challenge is ready.
Follow the instructions provided for this challenge and complete the required tasks in this repository.

Make sure your work is committed and pushed to your repository before submission.

Good luck!


---

&copy; 2025 GitHub &bull; [Code of Conduct](https://www.contributor-covenant.org/version/2/1/code_of_conduct/code_of_conduct.md) &bull; [MIT License](https://gh.io/mit)

---

# AIOps Practical Assessment

## 1. Overview
This project is a lightweight AIOps simulation for monitoring application service health. It reads operational data, detects abnormal behaviour, creates event records, and passes those events through a producer/topic/consumer flow before producing the final output.

The main idea is to follow the workflow:

Operational Data → Anomaly Detection → Event Generation → Producer → Topic → Consumer → AIOps Output

## 2. Repository components
The repository contains a small set of files that represent the full flow:

- `data/service_data.json` contains the operational records used by the workflow.
- `src/anomaly_detector.py` identifies abnormal service behaviour.
- `src/aiops_pipeline.py` coordinates the full AIOps process.
- `src/event_producer.py` publishes anomaly events.
- `src/event_topic.py` simulates the event topic or queue.
- `src/event_consumer.py` reads the events from the same topic.
- `src/calculations.py` contains simple utility functions unrelated to the AIOps flow.
- `tests/test_aiops_pipeline.py` validates the pipeline logic.
- `tests/calculations_test.py` validates the calculation utilities.

## 3. Operational data analysis
I reviewed the data in `data/service_data.json` and identified the main fields:

- `timestamp`: time when the record was created
- `service`: the service being monitored
- `response_time_ms`: service response time
- `cpu_percent`: CPU usage
- `memory_percent`: memory usage
- `log_level`: INFO, WARNING, or ERROR
- `message`: textual log message

The normal records had values such as low response time, normal CPU and memory usage, and INFO log entries. The unusual records had high response time, high CPU usage, high memory usage, and ERROR log entries. These were the records that clearly represented abnormal service behaviour.

## 4. Anomaly detection findings
The anomaly detector in `src/anomaly_detector.py` flags a record when any of the following conditions are met:

- response time is above 500 ms
- CPU usage is above 80%
- memory usage is above 80%
- log level is WARNING or ERROR

This means the detector combines both metric thresholds and log severity to judge whether an observation is abnormal. In the repository data, the records around `2026-09-20T10:05:00` and `2026-09-20T10:06:00` are the clear anomaly examples.

## 5. Event flow and AIOps processing
The event flow in the project is:

1. operational data is loaded
2. each record is checked by the anomaly detector
3. if an anomaly is detected, an event is created
4. the event is passed to the producer
5. the producer publishes it to the in-memory topic
6. the consumer reads it from the same topic
7. the final AIOps pipeline output is displayed

The simulated topic is implemented in `src/event_topic.py`, and it acts as the in-memory queue used by the producer and consumer.

## 6. Issues found and corrected
I identified three main issues during the assessment.

### Issue 1: pytest import issue
The test file in `tests/test_aiops_pipeline.py` originally imported modules using `from src...`, but the test environment was not adding the repository root to Python’s import path. This caused a `ModuleNotFoundError` during test collection.

The fix was to add the project root to `sys.path` before importing the project modules.

### Issue 2: producer/consumer topic mismatch
In `src/aiops_pipeline.py`, the producer and consumer were created with different topic objects. That meant the event was published to one topic and consumed from another, so the event never reached the consumer.

The fix was to use the same `EventTopic` instance for both producer and consumer.

### Issue 3: log-level anomaly rule
The log condition in `src/anomaly_detector.py` originally only checked `WARNING`. The actual operational data uses `ERROR` records in the abnormal cases, so the check needed to include both `WARNING` and `ERROR` to match the real telemetry.

## 7. Validation
I validated the project using the available tests and the end-to-end pipeline.

Commands used:

```bash
cd /workspaces/github-skills-challenge
pytest -q
python src/aiops_pipeline.py
```

Results:

- 12 tests passed
- 10 records processed
- 2 anomalies detected
- 2 events consumed

The pipeline produced the expected AIOps output for the payment service anomaly records.

## 8. Final workflow result
The final execution showed the anomaly records being identified and passed through the event flow successfully.

The output included:

- `Service: payment-service`
- `Timestamp: 2026-09-20T10:05:00`
- `Type: ANOMALY`
- `Reasons: High response time, Error log detected`

and a second anomaly event for the later record with high response time, high CPU, high memory, and error log information.

## 9. Reproduction steps
To reproduce the project:

1. Open the repository in GitHub Codespaces or a local environment.
2. Install dependencies if required:

```bash
pip install -r requirements.txt
```

3. Run the tests:

```bash
pytest -q
```

4. Run the pipeline:

```bash
python src/aiops_pipeline.py
```

5. Check the output to confirm that anomalies and events are processed successfully.

## 10. Conclusion
This project demonstrates a small but complete AIOps workflow in Python. It reads service telemetry, detects abnormal behaviour, transforms the result into a simulated event, moves that event through an in-memory event topic, and produces a final output. After fixing the identified issues, the repository works successfully and the validation passes.

---