# PostgreSQL Transactions, Locks, ROLLBACK, and Timesheet Cleanup

**Project:** HRGF Portal — Django and PostgreSQL  
**Purpose:** Understand what happened during timesheet deletion and how to debug database behaviour safely.

## 1. What happened?

We were cleaning pending timesheets from:

```sql
public.timesheet_unifiedtimesheet
```

We encountered three issues:

1. A `DELETE` query appeared to hang.
2. PostgreSQL rejected deletion of parent timesheets that had related child segments.
3. Running `ROLLBACK;` returned `WARNING: there is no transaction in progress`.

These are separate database behaviours.

## 2. Why did the query become slow?

We inspected PostgreSQL sessions through `pg_stat_activity`. One pgAdmin connection was `idle in transaction`, while other queries were waiting for transactions to finish.

The approximate blocking chain was:

```text
pgAdmin connection
    |
    | Open transaction
    v
Another transaction trying to COMMIT
    |
    | Waiting
    v
INSERT into timesheet_payrollcutoffsnapshot
    |
    | Waiting
    v
Other database queries
```

The important lesson is that a query can be waiting rather than actively processing data.

### What does `idle in transaction` mean?

A connection starts a transaction, executes a query, and then remains open without committing or rolling back.

Example:

```sql
BEGIN;

SELECT *
FROM public.timesheet_unifiedtimesheet;

-- Transaction remains open.
```

Depending on what the transaction did, it may retain locks or interfere with other transactions.

### How to investigate blocked queries

```sql
SELECT
    pid,
    state,
    wait_event_type,
    wait_event,
    now() - xact_start AS transaction_age,
    pg_blocking_pids(pid) AS blocking_pids,
    query
FROM pg_stat_activity
WHERE datname = current_database()
  AND (
      state = 'active'
      OR state = 'idle in transaction'
  )
ORDER BY xact_start NULLS LAST;
```

Key columns:

- `pid`: PostgreSQL connection/process ID.
- `state`: connection activity state.
- `wait_event_type`: category of the wait.
- `wait_event`: specific wait event.
- `transaction_age`: how long the transaction has been open.
- `blocking_pids`: process IDs blocking this process.
- `query`: current or most recently executed SQL, depending on the session state.

Do not assume a slow query is the blocker. Use `pg_blocking_pids(pid)` to identify the actual blocking sessions.

## 3. What is a transaction?

A transaction groups database operations into one unit.

| Command | Meaning |
|---|---|
| `BEGIN;` | Starts a transaction. |
| `COMMIT;` | Commits its changes. |
| `ROLLBACK;` | Cancels its uncommitted changes. |

### Example: Undo a deletion

```sql
BEGIN;

DELETE FROM public.timesheet_unifiedtimesheet
WHERE id = 2149;

SELECT *
FROM public.timesheet_unifiedtimesheet
WHERE id = 2149;

ROLLBACK;
```

If the deletion was performed inside this still-open transaction, `ROLLBACK` undoes it. The record should be present again.

### Example: Commit a deletion

```sql
BEGIN;

DELETE FROM public.timesheet_unifiedtimesheet
WHERE id = 2149;

COMMIT;
```

The deletion is now committed. A later `ROLLBACK` cannot ordinarily reverse it. Recovery may require a backup or another recovery mechanism.

## 4. Why did ROLLBACK report no transaction?

We executed:

```sql
ROLLBACK;
```

PostgreSQL returned:

```text
WARNING: there is no transaction in progress
```

This means the connection executing `ROLLBACK` did not have an open transaction to cancel.

The earlier investigation showed a different pgAdmin connection that was idle in a transaction.

**Each PostgreSQL connection has its own transaction state.** Running `ROLLBACK` in one query tab does not roll back a transaction belonging to another query tab or connection.

If a transaction is blocking other sessions, first identify the correct connection. Do not terminate a database session without understanding its query and transaction.

## 5. What is a foreign-key constraint?

The database contains two related tables:

```text
timesheet_unifiedtimesheet
          |
          | id
          v
timesheet_employeetimesheetsegment
          |
          | unified_timesheet_id
          |
          +---- references the parent timesheet
```

A child segment references its parent timesheet through `unified_timesheet_id`.

For example:

```text
Parent timesheet:
    id = 2151

Child segment:
    unified_timesheet_id = 2151
```

If we try to delete parent `2151` while a child segment still references it, PostgreSQL may reject the deletion.

This protects the database from leaving a child row with a reference to a nonexistent parent.

## 6. How we deleted 130 records safely from the parent table

We ran:

```sql
DELETE FROM public.timesheet_unifiedtimesheet t
WHERE LOWER(t.status) = 'pending'
  AND NOT EXISTS (
      SELECT 1
      FROM public.timesheet_employeetimesheetsegment s
      WHERE s.unified_timesheet_id = t.id
  );
```

PostgreSQL returned:

```text
DELETE 130
```

This means 130 rows were deleted from the parent table.

The query deleted only timesheets that:
- Had a status matching `pending`, regardless of letter case.
- Had no matching child segment.

It did not delete the child segments.

## 7. Understanding the subquery and NOT EXISTS

Consider this section:

```sql
NOT EXISTS (
    SELECT 1
    FROM public.timesheet_employeetimesheetsegment s
    WHERE s.unified_timesheet_id = t.id
)
```

A **subquery** is a query inside another query.

For each candidate parent timesheet, PostgreSQL checks whether a matching child segment exists.

- If a matching segment exists, `NOT EXISTS` is false, so the parent is excluded.
- If no matching segment exists, `NOT EXISTS` is true, so the parent may be deleted if the other conditions also match.

`SELECT 1` does not mean we are deleting the number one. It simply checks whether a matching row exists.

### Why use LOWER(status)?

```sql
LOWER(t.status) = 'pending'
```

This allows values such as `pending`, `PENDING`, and `Pending` to match.

## 8. What did we find in the remaining 15 timesheets?

After deleting 130 records, 15 pending parent timesheets remained because each had a related segment.

The IDs were:

```text
2149–2162, excluding 2163, and 2164
```

They covered July, September, and October 2026.

Some related segments had workflow states such as `NOT_REACHED`; others had a workflow step marked `PENDING`. Record `2162` also showed an `APPROVED` value in one workflow-related column.

Therefore, a parent status of `pending` does not necessarily mean all related workflow, approval, or payroll states are pending.

Before deleting these records, check:

1. Whether the child segments can be discarded.
2. Whether other tables reference these records.
3. Whether payroll submissions or approval workflows use them.
4. Whether today's October records should be preserved.
5. Whether a backup or recovery path exists.

Do not automatically delete related segments merely to make a parent deletion succeed.

## 9. A safer debugging workflow

### Step 1: Preview affected records

```sql
SELECT id, employee_id, status, updated_at
FROM public.timesheet_unifiedtimesheet
WHERE LOWER(status) = 'pending'
ORDER BY id;
```

### Step 2: Count matching records

```sql
SELECT COUNT(*) AS pending_count
FROM public.timesheet_unifiedtimesheet
WHERE LOWER(status) = 'pending';
```

### Step 3: Check related segments

```sql
SELECT
    t.id,
    t.employee_id,
    COUNT(s.unified_timesheet_id) AS segment_count
FROM public.timesheet_unifiedtimesheet t
LEFT JOIN public.timesheet_employeetimesheetsegment s
    ON s.unified_timesheet_id = t.id
WHERE LOWER(t.status) = 'pending'
GROUP BY t.id, t.employee_id
ORDER BY t.id;
```

### Step 4: Investigate locks

Use `pg_stat_activity` and `pg_blocking_pids(pid)` to check whether queries are waiting for another transaction.

### Step 5: Use a transaction for reviewed cleanup

```sql
BEGIN;

-- Run reviewed cleanup operations here.

-- Check affected row counts and remaining records.

-- If anything is unexpected:
-- ROLLBACK;

-- If everything is verified:
-- COMMIT;
```

Run `ROLLBACK` or `COMMIT` in the same connection that started the transaction. Do not commit until the results have been verified.

For production data, check foreign-key relationships, arrange a suitable backup, and obtain the necessary approval before deleting records that could affect payroll or audit history.

## 10. Core concepts to remember

- **Slow query:** May be processing data or waiting for another transaction.
- **Lock:** Coordinates concurrent database operations and can make queries wait.
- **Transaction:** A group of operations with a shared commit or rollback decision.
- **COMMIT:** Makes transaction changes permanent.
- **ROLLBACK:** Undoes uncommitted changes in the current transaction and connection.
- **Foreign key:** Protects relationships between parent and child records.
- **Subquery:** A query nested inside another query.
- **NOT EXISTS:** Checks that no matching row exists.
- **DELETE 130:** Means 130 rows were deleted from the target table.
- **Safe cleanup:** Preview, count, inspect dependencies, delete approved records, validate, and commit.

## Final takeaway

This incident taught us three important parts of database behaviour:

1. **Transaction management** — how `BEGIN`, `COMMIT`, and `ROLLBACK` work.
2. **Concurrency and locking** — why one query may wait for another transaction.
3. **Referential integrity** — why PostgreSQL protects parent and child relationships.

Django's ORM ultimately relies on these same PostgreSQL mechanisms. Understanding them makes it easier to investigate slow queries, database locks, and failed deletions safely.
