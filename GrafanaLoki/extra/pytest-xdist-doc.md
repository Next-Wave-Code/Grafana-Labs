# Parallel Testing with pytest-xdist: A Beginner's Guide

## What is pytest-xdist?

pytest-xdist is a plugin that lets you run your tests in parallel across multiple CPUs. This means your tests can finish much faster!

```
 Normal pytest (Sequential)       pytest-xdist (Parallel)
 ┌─────────────────────┐          ┌─────────────────────┐
 │                     │          │                     │
 │    Test Runner      │          │    Test Runner      │
 │                     │          │    (Controller)     │
 └─────────┬───────────┘          └──┬─────┬─────┬─────┘
           │                         │     │     │     
           ▼                        ▼    ▼    ▼     
 ┌─────────────────────┐     ┌───────┐ ┌───────┐ ┌───────┐
 │ test_1 → test_2 → │     │Worker1│ │Worker2│ │Worker3│
 │ test_3 → test_4 → │     │test_1 │ │test_2 │ │test_3 │
 │ test_5 → test_6    │     │test_4 │ │test_5 │ │test_6 │
 └─────────────────────┘     └───────┘ └───────┘ └───────┘

 Runtime: 60 seconds           Runtime: ~20 seconds
```

## Getting Started

To run tests in parallel:

```bash
# Install pytest-xdist
pip install pytest-xdist

# Run tests using 4 CPU cores
pytest -n 4

# Run tests using as many cores as your CPU has
pytest -n auto
```

That's it! Your tests will now run in parallel, which can dramatically speed up your test suite.

## Features Currently Used in Our Project

### Understanding Workers

When pytest-xdist runs, it creates multiple "worker" processes to run your tests. Each worker gets a portion of the tests to run:

```
┌──────────────────────────────────────────────────────────────────┐
│                           Controller                             │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐            │
│  │   Worker 1   │  │   Worker 2   │  │   Worker 3   │            │
│  │   (gw0)      │  │   (gw1)      │  │   (gw2)      │            │
│  │              │  │              │  │              │            │
│  │ test_login.py│  │ test_payment │  │ test_profile │            │
│  └──────────────┘  └──────────────┘  └──────────────┘            │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Identifying the Worker in Your Tests

Sometimes you need to know which worker is running a particular test. This is useful when you need to use different resources for each worker.

```python
@pytest.fixture()
def test_account(worker_id):
    """Create a different test account for each worker"""
    return f"test_user_{worker_id}@example.com"

def test_login(test_account):
    # Each worker will use a different test account
    # For example: test_user_gw0@example.com, test_user_gw1@example.com, etc.
    assert login_with_account(test_account) is True
```

### What's My Worker ID?

When pytest-xdist is running, each worker gets an ID like `gw0`, `gw1`, `gw2`, etc. (gw stands for "gateway").

If you're not using parallel mode (i.e., running with `-n0`), the worker_id will be `master`.

### Environment Variables

Worker processes also have these environment variables defined:

- `PYTEST_XDIST_WORKER`: The name of the worker, e.g., `gw2`
- `PYTEST_XDIST_WORKER_COUNT`: The total number of workers, e.g., `4` when running with `-n 4`

You can use these in your code like this:

```python
import os

def setup_something():
    worker = os.environ.get("PYTEST_XDIST_WORKER", "master")
    print(f"Setting up resources for worker {worker}")
```

### Test Run Unique ID (testrun_uid)

Each time you execute the pytest command, a **single unique ID** is generated for the entire test run. All tests and all workers in that run share the same ID.

```
┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│  Test Run 1                      Test Run 2                      │
│  testrun_uid: abc123             testrun_uid: def456             │
│  ┌─────────┐ ┌─────────┐         ┌─────────┐ ┌─────────┐         │
│  │ Worker1 │ │ Worker2 │         │ Worker1 │ │ Worker2 │         │
│  └─────────┘ └─────────┘         └─────────┘ └─────────┘         │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

**Important**: The `testrun_uid` is the same for all tests in a single run. It only changes when you start a new pytest execution.

This is useful when you need resources that are:
- Shared by all workers in the same test run
- Different between separate test runs

Example:
```python
@pytest.fixture(scope="session")
def create_unique_database(testrun_uid):
    """Create a unique database for this test run that all workers will share"""
    database_url = f"db://test-{testrun_uid}"
    
    # Use a lock to ensure only one worker creates the database
    with FileLock(f"db-{testrun_uid}.lock"):
        if not database_exists(database_url):
            create_database(database_url)
            print(f"Created database: {database_url}")
    
    return database_url
```

The `testrun_uid` is also available as an environment variable: `PYTEST_XDIST_TESTRUNUID`

### Creating Worker-Specific Log Files

To create a separate log file for each worker:

```python
# In your conftest.py
import os
import logging

def pytest_configure(config):
    worker_id = os.environ.get("PYTEST_XDIST_WORKER")
    if worker_id is not None:
        # Create a separate log file for each worker
        logging.basicConfig(
            format="%(asctime)s [%(levelname)s] %(message)s",
            filename=f"test_log_{worker_id}.log",
            level=logging.INFO
        )
```

With this setup, running tests with `-n3` will create three log files: 
- `test_log_gw0.log`
- `test_log_gw1.log`
- `test_log_gw2.log`

### Common Command Examples

```bash
# Run on 4 CPUs
pytest -n 4

# Run on all available CPUs
pytest -n auto

# Disable parallel testing
pytest -n 0
```

## Additional Features for Future Use

### Distribution Modes

```bash
# Group tests by module for better performance with shared fixtures
pytest -n 4 --dist=loadscope

# Run each test file on a different worker
pytest -n 4 --dist=loadfile
```

### Configuration in pytest.ini

You can set default options in your pytest.ini file:

```ini
[pytest]
# Always use 4 workers by default
addopts = -n4

# Directories to sync to remote machines (if using --tx)
rsyncdirs = . mypkg helperpkg

# Directories to exclude from syncing
rsyncignore = .git build
```

### Tracking Workers in System Tools

```
┌─────────────────────────────────────────────┐
│                                             │
│  $ ps aux | grep pytest                     │
│                                             │
│  user  1234  ... [pytest-xdist running] ... │
│  user  1235  ... [pytest-xdist idle] ...    │
│                                             │
└─────────────────────────────────────────────┘
```

If you install the `setproctitle` package, pytest-xdist will update the process title of workers to show:
- `[pytest-xdist running] file.py/node::id` - When a worker is running a test
- `[pytest-xdist idle]` - When a worker is waiting for the next test

This helps you identify which worker is running which test when you use tools like `ps` or `top` (on Linux/Mac) or Process Explorer (on Windows).

### Session-Scoped Fixtures in Parallel Testing

A challenge with parallel testing is that fixtures marked as `scope="session"` normally run once per worker, not once per test run. This can be a problem if you need something to happen exactly once.

Here's a pattern to make a session fixture run only once across all workers:

```python
import json
import pytest
from filelock import FileLock

@pytest.fixture(scope="session")
def session_data(tmp_path_factory, worker_id):
    """A fixture that executes only once regardless of how many workers are used"""
    
    # If not using xdist, just run as normal
    if worker_id == "master":
        return produce_expensive_data()

    # Get a temp directory shared by all workers
    root_tmp_dir = tmp_path_factory.getbasetemp().parent
    
    # Use a file to share data between workers
    fn = root_tmp_dir / "shared_data.json"
    
    # Use a lock file to coordinate between workers
    with FileLock(str(fn) + ".lock"):
        if fn.is_file():
            # Another worker already did the work, just read the data
            data = json.loads(fn.read_text())
        else:
            # We're the first worker, do the expensive operation
            data = produce_expensive_data()
            fn.write_text(json.dumps(data))
    
    return data
```

### Accessing Command Line Arguments in Workers

Workers don't automatically have access to the command line arguments passed to pytest. Use `request.config.workerinput["mainargv"]` to access them:

```python
@pytest.fixture
def cli_args(request):
    try:
        # In a worker
        return request.config.workerinput["mainargv"]
    except AttributeError:
        # In the master process
        return sys.argv
```

## Troubleshooting Tips

1. **Tests interfere with each other**: Make sure tests don't depend on shared state. Each test should be isolated.

2. **Database conflicts**: Use different database names for each worker or use a locking mechanism.

3. **Slower than expected**: Try different distribution modes:
   ```bash
   # Default mode - tests distributed evenly
   pytest -n 4
   
   # Group by module (helps with module-level fixtures)
   pytest -n 4 --dist=loadscope
   
   # Group by file (helps with file-level fixtures)
   pytest -n 4 --dist=loadfile
   ```

4. **Session fixtures running multiple times**: Use the FileLock pattern shown above.

5. **Random failures**: Some tests may not be thread-safe. Try running specific failing tests without xdist to confirm.

## Conclusion

pytest-xdist dramatically speeds up your test suite by running tests in parallel. With the patterns and fixtures described above, you can handle the challenges that come with parallel testing.

Remember:
- Use `-n` to specify the number of workers
- Use the `worker_id` fixture when you need worker-specific resources
- Use the `testrun_uid` fixture when you need resources shared across workers but unique to this test run
- Use FileLock when you need to coordinate between workers
