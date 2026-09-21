# GitHub Challenge

<img src="https://octodex.github.com/images/Professortocat_v2.png" align="right" height="200px" />

Hey there!

Your challenge is ready.
Follow the instructions provided for this challenge and complete the required tasks in this repository.

Make sure your work is committed and pushed to your repository before submission.

Good luck!

## AIOps scenario

This exercise monitors a synthetic `payment-service`. The operational problem is
to identify a short period of degraded service caused by slow requests, high
resource usage, and timeout errors. AIOps combines the metric and log signals,
creates an actionable anomaly event, and moves it through a simulated event
stream so that downstream processing can report the issue.

## Repository components

- `data/service_data.json` contains 10 observations, one per minute.
- `src/anomaly_detector.py` applies fixed thresholds and checks error logs.
- `src/event_producer.py` publishes detected events.
- `src/event_topic.py` is the in-memory topic carrying messages.
- `src/event_consumer.py` receives messages from the topic.
- `src/aiops_pipeline.py` loads data, detects anomalies, and reports the
        consumed events as the final AIOps output.
- `tests/` validates the calculations and the AIOps workflow.

## Operational data analysis

Each observation has seven attributes. `timestamp` identifies the observation in
ISO 8601 format and is ordered at one-minute intervals from
`2026-09-20T10:00:00` through `2026-09-20T10:09:00`. This makes the data a time
series and shows the incident between otherwise healthy observations.

The metrics are `response_time_ms`, `cpu_percent`, and `memory_percent`. The log
information is `service`, `log_level`, and `message`; `service` identifies the
source, while the other two describe severity and the operational event.

The eight records at 10:00-10:04 and 10:07-10:09 appear normal. Latency is
120-150 ms, CPU is 42-50%, memory is 51-57%, and each log is `INFO` with a
successful payment message. The records at 10:05 and 10:06 are unusual:

| Timestamp | Response | CPU | Memory | Log | Message |
| --- | ---: | ---: | ---: | --- | --- |
| 10:05 | 610 ms | 75% | 70% | ERROR | Payment service timeout |
| 10:06 | 640 ms | 94% | 91% | ERROR | Database connection timeout |

The service returns to the normal range at 10:07, so the abnormal behavior is
concentrated in these two observations.

## Detection findings

The detector thresholds are response time above 500 ms, CPU above 80%, and
memory above 80%. It also flags `ERROR` logs. It detected both expected
anomalies and no normal observations:

- `2026-09-20T10:05:00`: high response time and an error log.
- `2026-09-20T10:06:00`: high response time, high CPU, high memory, and an
        error log.

The original detector checked for `WARNING`, although the supplied data uses
`ERROR`. I corrected that condition, so timeout logs now contribute a reason to
the event. A limitation is that the thresholds are fixed and do not learn a
service-specific baseline. A useful improvement would be a rolling baseline or
statistical detector that adapts to normal traffic patterns and reduces both
false positives and false negatives.

## Event-processing flow

For every detected anomaly, the `EventProducer` publishes the event to the
`anomaly-events` `EventTopic`. The `EventConsumer` reads from that same topic
and returns the processed messages to `run_pipeline`, which prints the final
AIOps output. The event contains the timestamp, service, type, detection
reasons, and original source record.

The original pipeline created separate producer and consumer topics, so the
consumer always saw zero events. I corrected the pipeline to share one
`anomaly-events` topic. This keeps the provided producer, topic, consumer, and
event architecture intact.

## Final execution result

Command:

```text
python src/aiops_pipeline.py
```

Result:

```text
Records processed: 10
Anomalies detected: 2
Events consumed: 2
2026-09-20T10:05:00: High response time, Error log detected
2026-09-20T10:06:00: High response time, High CPU utilization,
High memory utilization, Error log detected
```

This demonstrates the complete path: operational data, anomaly detection,
event generation, producer publication, topic delivery, consumer receipt, and
final AIOps output. No expected anomaly was missed and no normal record was
flagged for this supplied dataset.

## Reproduce and validate

From the repository root, install the dependencies and run the tests and
pipeline:

```bash
pip install -r requirements.txt
python -m pytest -q
python src/aiops_pipeline.py
```

The validation currently reports `9 passed`. The end-to-end regression test
asserts that all 10 records are processed, exactly two anomaly events are
generated, and both events are consumed in timestamp order.

&copy; 2025 GitHub &bull; [Code of Conduct](https://www.contributor-covenant.org/version/2/1/code_of_conduct/code_of_conduct.md) &bull; [MIT License](https://gh.io/mit)