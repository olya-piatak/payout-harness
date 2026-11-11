# payout-harness

[![ci](https://github.com/olya-piatak/payout-harness/actions/workflows/ci.yml/badge.svg)](https://github.com/olya-piatak/payout-harness/actions/workflows/ci.yml)

A test harness for asynchronous money movement, and proof that it catches the
things it was built to catch.

The flow it covers is the one a tips platform lives on: a shift closes, the
pool is split between the people who worked it, and the payouts settle
asynchronously. The API answers before the money has moved, which is where the
defects that matter actually live.

```
POST /shifts                      a shift closes with a pool and minutes worked
POST /shifts/{id}/payouts   202   shares are calculated, payouts are pending
                                  nothing has settled yet
POST /_worker/run                 the worker moves the money
GET  /shifts/{id}/ledger          the ledger total equals the pool, or something is wrong
```

## Why this exists

The most expensive defect I have met in production was not a broken screen. It
was a system that reported success while the work behind it never happened. The
request returned 200, the interface showed a tick, and the downstream event was
never published.

In a payments flow the same shape means somebody did not get paid. This repo is
how I test for it: hold the worker still, look at the gap between the promise
and the ledger, and assert on stored state rather than on what the API said.

## Testing layers

Each layer answers a different question, which is how the suite stays fast
without losing coverage of the part that moves money.

| Layer | What it checks | Count | Speed |
| --- | --- | --- | --- |
| `unit` | pool arithmetic, including a property test that no cent is created or lost | 10 | milliseconds |
| `integration` | the API against its own storage, worker held still | 6 | fast |
| `e2e` | the full flow through the worker to the ledger | 5 | fast |
| `contract` | real responses against the OpenAPI schema the service publishes | 4 | fast |

The whole suite runs in well under a second and is meant to block a merge, not to
run overnight. Anything slower than that belongs behind a separate job.

The property test is the one worth reading first. The pool split uses the
largest remainder method, so the interesting cases are the pools that do not
divide cleanly. Hypothesis generates those for me instead of me guessing which
awkward numbers to hardcode.

## Fault drills

A suite that has never failed is not a gate. The service can be told to
reintroduce three real defects, and CI requires the suite to fail on each one.

```
$ python scripts/verify_faults.py

CAUGHT  double_pay: the idempotency key is ignored on retry
        tests/integration/test_payout_api.py::test_a_retried_request_pays_once
CAUGHT  ghost_success: payout is marked paid before the ledger entry exists
        tests/e2e/test_payout_flow.py::test_nothing_is_settled_before_the_worker_runs
        tests/integration/test_payout_api.py::test_a_payout_request_is_accepted_but_not_yet_paid
CAUGHT  rounding: pool split drops the remainder cents
        tests/e2e/test_payout_flow.py::test_the_pool_reaches_the_ledger
        tests/e2e/test_payout_flow.py::test_two_shifts_do_not_leak_into_each_other
        tests/integration/test_payout_api.py::test_the_shares_add_up_to_the_pool
        and 3 more

all 3 faults were caught
```

`ghost_success` is the fault this repo is really about. With it switched on the
service marks a payout settled the moment the request arrives, before any
ledger entry exists. Every status endpoint answers correctly, every response
validates against the schema, and the money is still sitting in the queue. Only
a test that looks at stored state between the API call and the worker run can
see it.

## Test data

Factories in `tests/factories.py` describe shifts by shape: an even one, an
awkward one that does not divide cleanly, one with a given number of people.
Each test states only the part it cares about. Nothing is random unless a test
asks for randomness, nothing is shared between tests, and no test reads data
from anywhere but its own fixtures.

Each test gets its own service, its own storage and its own queue, so tests can
run in any order and in parallel.

## Running it

```
make install      # virtualenv and dependencies
make test         # the whole suite
make unit         # the fast layer only
make drills       # break the service three ways, require the suite to fail
make run          # the service itself, on localhost:8000
```

Python 3.11 or newer. No Docker and no external services, because a harness
that needs an afternoon of setup does not get run.

## What the service is

`service/` is a deliberately small stand-in, not a product: FastAPI, SQLite and
an in-process queue that only moves when a test tells it to. It exists so the
tests have something real to run against, and it is modelled on the parts of a
tip payout flow where accuracy matters: weighted pool splits, idempotent payout
requests, asynchronous settlement and a ledger.

In production that queue is Celery and that database is PostgreSQL. What does
not change is the question the tests ask, which is whether the ledger agrees
with the pool.

## Not in scope

No authentication, no currencies other than integer cents, no partial payouts,
no retries with backoff. Those belong in the real platform, and adding stubs for
them here would only make the harness harder to read.

MIT licensed. Written by Olya Piatak.
