# COMP5320 Lab 3: Design, Architecture and Patterns with AI Agents

| | |
| --- | --- |
| **Student** | Ho Thi Minh Ngan |
| **Student ID** | V202603134 |
| **Course** | COMP5320 Advanced Software Engineering |
| **Week** | 3: Software Design, Architecture and Patterns |
| **Submission type** | Individual |

Dear Professor,

This repository contains my submission for Lab 3. I repaired and extended the
task-assignment service according to requirements R1 to R5 and R7, completed the
sequence diagram, and wrote the design decision record, including my answer to
the changed requirement R6. All 12 provided tests pass on my machine, and I
explain my reasoning for each lab below.

## How to run the tests

The project uses Python 3.11 and the standard library only. From this folder:

```sh
python test_lab.py all
```

On Windows, `py -3 test_lab.py all` also works. To check a single lab, replace
`all` with `1`, `2`, `3` or `4`. The output of my final run is saved in
`test_results.txt` (Ran 12 tests, OK).

## Submitted files

| File | Purpose |
| --- | --- |
| `service.py` | Labs 1 and 3: corrected ordering and failure boundaries in `TaskService` |
| `push_channel.py` | Lab 2: new `PushChannel` notification strategy |
| `audit_channel.py` | Lab 4: `AuditedChannel` decorator |
| `sequence.mmd` | Sequence diagram for the success, `SaveError` and `NotificationError` paths |
| `DECISIONS.md` | Design decision record (within 200 words) |
| `test_results.txt` | Output of `python test_lab.py all` |

The remaining files (`models.py`, `adapters.py`, `test_lab.py`, `AGENTS.md`) are
the provided starter materials, and I did not modify them.

## Commit history

I committed each lab separately so that every step can be reviewed on its own
diff, as the lab instructions require:

1. Starter pack as provided
2. Lab 1: changes only `service.py`
3. Lab 2: changes only `push_channel.py`
4. Lab 3: changes only `service.py`
5. Lab 4: changes only `audit_channel.py`
6. Sequence diagram, decision record and test results

## Lab 1: Correcting the interaction (R1 to R3)

In the starter code, `assign_task` sent the notification before saving the task.
If the save then failed, the assignee would have been told about an assignment
that was never stored. I moved `self._repository.save(assigned)` before the call
to `channel.send(...)`, so the order is now `get`, `save`, `send`. If `save`
raises `SaveError`, the exception propagates and no send is attempted.

Responsibilities stay separated (SRP): `TaskService` owns the ordering and
failure policy, the repository owns storage, and the channel owns delivery. The
service still receives its collaborators through the injected interfaces (DIP).

**Evidence:** `test_save_happens_before_send` checks the trace
`['get', 'save', 'send']`, and `test_save_failure_propagates_without_any_send_attempt`
checks that `email.attempts` is 0 after a failed save.

## Lab 2: Extending delivery with Strategy (R5)

I implemented `PushChannel.send` so that it appends each `(recipient, message)`
pair to `self.sent`. No change to `TaskService` was needed: the test simply
registers `{'push': push}`, and the service selects the channel from the
assignee's preference. Strategy absorbs the change "add a new delivery method",
and registration belongs in the startup or composition code, not in the service.
This step's diff touches only `push_channel.py`, which shows the open/closed
principle at this boundary.

## Lab 3: Verifying failure boundaries (R4)

I narrowed the handler from `except Exception` to `except NotificationError`.
When delivery fails in the expected way, the service logs a warning that
contains only the task id (no contact details), keeps the saved assignment and
returns it normally. It does not retry.

A broad `except Exception` would also swallow programming errors. The test
`test_unexpected_provider_bug_remains_visible_after_save` uses a channel that
raises `RuntimeError`; with the broad handler this defect would be hidden and
only a warning would appear. With the narrow handler the error propagates, while
the assignment remains saved.

## Lab 4: Composing a channel with Decorator (R7)

`AuditedChannel.send` delegates exactly once to the wrapped channel and appends
`notification.accepted` to `self.events` only after the delegate returns
successfully. If the delegate raises, the exception propagates and no event is
recorded. The wrapper is injected as a normal channel, so neither `TaskService`
nor `PushChannel` changed.

I consider this a Decorator rather than a new Strategy because it does not
choose a different delivery behavior. It wraps a channel that has already been
chosen and adds behavior around it while keeping the same `send` contract. I
also note that this in-memory record is not a durable audit log.

## UML and architecture review

`sequence.mmd` shows the three paths of `assign_task`: success, `SaveError` with
no send, and `NotificationError` with a normal return. The participant names and
message order follow the code, and a note states that a returned `save` models a
completed commit.

`DECISIONS.md` defends a modular monolith for this project, distinguishes
Strategy, Decorator and Observer, and outlines a transactional outbox for R6,
including why a relay crash after provider acceptance can cause a duplicate
notification.

## Use of the AI agent and remaining limitations

I used Claude Code as an AI assistant during this lab. I checked its changes
against the requirements, inspected each diff, and ran the tests myself. The
offline tests use in-memory adapters, so they do not verify real database commit
durability, real provider behavior or crash recovery. R6 is addressed only at the
design level, as the lab specifies, and no outbox is implemented.

Thank you for reviewing my submission.

Ho Thi Minh Ngan
