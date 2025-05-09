# Python Logging with Loki in Airpay Money Test Framework

This guide explains how we use `python-logging-loki` in our test automation framework to send logs to Grafana Loki for centralized logging and analysis.

## What is python-logging-loki?

`python-logging-loki` is a Python library that connects Python's standard logging system to Grafana Loki, a log aggregation system. It allows us to:

- Send all test logs to a central Loki server
- Tag logs with useful metadata (environment, browser, test run ID, etc.)
- Query and filter logs using Grafana's powerful interface
- Store logs beyond the lifetime of test runs

## How Logging Works in Our Framework

```
                           ┌─────────────────┐
                           │                 │
                           │  Python Tests   │
                           │                 │
                           └────────┬────────┘
                                    │
                                    ▼
┌───────────────────────────────────────────────────────┐
│               Python Logging System                   │
│                                                       │
│  ┌─────────────┐      ┌─────────────┐                 │
│  │   Console   │      │  Queue      │                 │
│  │   Handler   │      │  Handler    │                 │
│  └─────────────┘      └──────┬──────┘                 │
│                              │                        │
└──────────────────────────────┼────────────────────────┘
                               │
                               ▼
                     ┌───────────────────┐
                     │                   │
                     │  Queue Listener   │
                     │   (Background)    │
                     │                   │
                     └─────────┬─────────┘
                               │
                               ▼
                     ┌───────────────────┐     ┌─────────────────┐
                     │                   │     │                 │
                     │   Loki Handler    │───▶│  Grafana Loki   │
                     │                   │     │                 │
                     └───────────────────┘     └─────────────────┘
```

## How We Set It Up

Our framework uses a queue-based approach for non-blocking logging. Here's how it works:

1. When tests start, we check if the `--loki` flag is enabled
2. If enabled, we set up a Queue and QueueHandler
3. Logs are sent to a background thread (QueueListener)
4. The QueueListener forwards logs to Loki
5. If setup fails, we fall back to a direct (synchronous) Loki handler

### Code Breakdown (from conftest.py)

```python
# 1. Create queue for log messages
from multiprocessing import Queue
import logging.handlers
log_queue = Queue(-1)  # Unlimited queue size

# 2. Create queue handler that routes logs to the queue
queue_handler = logging.handlers.QueueHandler(log_queue)

# 3. Create Loki handler for sending logs to Grafana
loki_handler = logging_loki.LokiHandler(
    url="http://localhost:3100/loki/api/v1/push",
    tags=loki_tags,
    version="1",
)

# 4. Create and start the queue listener
listener = logging.handlers.QueueListener(
    log_queue, loki_handler, respect_handler_level=True
)
listener.start()

# 5. Add queue handler to root logger
root_logger = logging.getLogger()
root_logger.addHandler(queue_handler)
```

## Log Tags: The Secret to Easy Filtering

We add special tags to each log message, which makes filtering in Grafana much easier:

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│  Log Message                                                        │
│  ───────────                                                        │
│                                                                     │
│  INFO  [TEST START]: test_login_with_valid_credentials              │
│                                                                     │
│  Tags:                                                              │
│  ─────                                                              │
│  ┌─────────────────┬───────────────────────────────────────────┐    │
│  │ service_name    │ airpay_money                              │    │
│  ├─────────────────┼───────────────────────────────────────────┤    │
│  │ environment     │ staging                                   │    │
│  ├─────────────────┼───────────────────────────────────────────┤    │
│  │ browser         │ chrome                                    │    │
│  ├─────────────────┼───────────────────────────────────────────┤    │
│  │ testrun_uid     │ 7ae536b3a92e4b1d9768d36b4abe6670          │    │
│  ├─────────────────┼───────────────────────────────────────────┤    │
│  │ headless        │ False                                     │    │
│  ├─────────────────┼───────────────────────────────────────────┤    │
│  │ worker_id       │ gw0                                       │    │
│  └─────────────────┴───────────────────────────────────────────┘    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

Here's what each tag means:

- **service_name**: Always "airpay_money" (identifies logs from our system)
- **environment**: Which test environment we're running against (dev/staging/prod)
- **browser**: Which browser is being used for the test
- **testrun_uid**: A unique ID for each test run, provided by pytest-xdist
- **headless**: Whether the browser is running in headless mode
- **worker_id**: Which worker is running the test (for parallel execution)

## Enabling Loki Logging

Loki logging is disabled by default to avoid sending logs when not needed. Add the `--loki` flag to enable it:

```bash
# Run tests without Loki logging
poetry run pytest

# Run tests with Loki logging enabled
poetry run pytest --loki
```

## Viewing Logs in Grafana

After running tests with `--loki` enabled, you can view the logs in Grafana:

```
┌────────────────────────────────────────────────────────────────────────────────┐
│                                                                                │
│   Grafana                                                                      │
│   ───────                                                                      │
│                                                                                │
│   ┌─────────────────┐  ┌──────────────────────────────────────────────────┐    │
│   │                 │  │                                                  │    │
│   │    Explore      │  │  Loki        ▼                                  │    │    
│   │                 │  │                                                  │    │
│   │    Dashboards   │  │  {testrun_uid="7ae536b3a92e4b1d9768d36b4abe6670"}│    │
│   │                 │  │                                                  │    │
│   │    Alerts       │  │  Run Query                                       │    │
│   │                 │  │                                                  │    │
│   └─────────────────┘  └──────────────────────────────────────────────────┘    │
│                                                                                │
│   ┌─────────────────────────────────────────────────────────────────────┐      │
│   │                                                                     │      │
│   │  2025-05-09 08:47:32 [INFO    ] tests.conftest -> [TEST RUN ID]:    │      │
│   │  7ae536b3a92e4b1d9768d36b4abe6670 [configure_loki 215]              │      │
│   │                                                                     │      │
│   │  2025-05-09 08:47:32 [DEBUG   ] tests.conftest -> Loki logging is   │      │
│   │  enabled for worker gw0. Logs will be sent to Grafana. [configure_  │      │
│   │  loki 271]                                                          │      │
│   │                                                                     │      │
│   │  2025-05-09 08:47:33 [INFO    ] tests.conftest -> [TEST START]:     │      │
│   │  test_login_with_valid_credentials [_setup_webdriver 609]           │      │
│   │                                                                     │      │
│   └─────────────────────────────────────────────────────────────────────┘      │
│                                                                                │
└────────────────────────────────────────────────────────────────────────────────┘
```

### Useful Queries

Here are some helpful queries to try in Grafana:

1. View all logs from a specific test run:
   ```
   {testrun_uid="7ae536b3a92e4b1d9768d36b4abe6670"}
   ```

2. View logs from a specific worker:
   ```
   {worker_id="gw0"}
   ```

3. View only error logs:
   ```
   {level="ERROR"}
   ```

4. View logs from Chrome tests in staging:
   ```
   {browser="chrome", environment="staging"}
   ```

5. View logs for a specific test:
   ```
   {testrun_uid="7ae536b3a92e4b1d9768d36b4abe6670"} |= "test_login_with_valid_credentials"
   ```

## Fallback Mechanism

If something goes wrong with the queue-based setup, we have a fallback:

```
┌─────────────────┐      ┌─────────────┐      ┌─────────────────┐
│                 │      │             │      │                 │
│  Python Tests   │────▶│  Loki       │────▶│  Grafana Loki   │
│                 │      │  Handler    │      │                 │
└─────────────────┘      └─────────────┘      └─────────────────┘
```

This direct approach is simpler but may block test execution if Loki is slow to respond. It's only used if the queue-based setup fails.

## Practical Tips for Beginners

1. **Always note the testrun_uid when running important tests**:
   The log shows it at the start: `[TEST RUN ID]: 7ae536b3a92e4b1d9768d36b4abe6670`

2. **Use Grafana's Live mode for real-time monitoring**:
   Enable the "Live" button in Grafana to see logs appear as tests run

3. **Combine log queries with dashboard panels**:
   Create a dashboard with panels for errors, warnings, and test starts/ends

4. **Check logs when tests fail**:
   If a test fails, look at the logs filtered by that test name to see what happened

5. **If Loki isn't working**:
   - Make sure Loki is running (default: http://localhost:3100)
   - Check that you're using the `--loki` flag
   - Verify your network can reach the Loki server

## Customizing Loki Settings

To change the Loki server URL, modify this line in `conftest.py`:

```python
loki_handler = logging_loki.LokiHandler(
    url="http://localhost:3100/loki/api/v1/push",  # Change this URL
    tags=loki_tags,
    version="1",
)
```

## Log Levels Explained

We use different log levels for various purposes:

- **DEBUG**: Detailed information for troubleshooting
- **INFO**: Confirmation that things are working as expected
- **WARNING**: Something unexpected happened, but tests can continue
- **ERROR**: A serious problem that prevented something from working
- **CRITICAL**: A very serious error that might stop the entire test run

Most test progress messages use INFO level, while setup details use DEBUG.

## Conclusion

`python-logging-loki` gives us powerful centralized logging capabilities for our test automation framework. By tagging logs with metadata about the test environment, browser, and test run ID, we can easily filter and analyze logs to debug test failures and understand test behavior.

