# Why Prefer ORM Queries Over Python Loops?

When working with database-backed data, prefer **database-side filtering, aggregation, joining, and existence checks** over loading large datasets into Python and processing them with loops.

The key principle is:

> **Let the database process data whenever the operation is naturally a database operation. Keep Python responsible for application and business logic.**

---

## 1. Python Loop vs ORM

### Python-side filtering

```python
employees = Employee.objects.all()

active_employees = []

for employee in employees:
    if employee.is_active and employee.department_id == department_id:
        active_employees.append(employee)
```

The application may retrieve a large number of rows and then perform the filtering in Python.

### ORM filtering

```python
active_employees = Employee.objects.filter(
    is_active=True,
    department_id=department_id,
)
```

Conceptually, Django generates:

```sql
SELECT *
FROM employee
WHERE is_active = TRUE
  AND department_id = ...;
```

The database performs the filtering before the result reaches Python.

---

## 2. Why Database-Side Processing Is Usually Better

### Less data transferred

Instead of:

```text
Database
   ↓
Many rows
   ↓
Python
   ↓
Filter
```

we get:

```text
Database
   ↓
Filter
   ↓
Only required rows
   ↓
Python
```

This reduces:

* Database → application data transfer
* Python object creation
* Application memory usage
* Python CPU processing
* Garbage collection overhead

---

## 3. Database Indexes

Databases can use indexes to efficiently locate matching records.

For example:

```python
Employee.objects.filter(
    department_id=10,
    is_active=True,
)
```

With suitable indexes, PostgreSQL may avoid scanning the entire table.

The database query planner can choose strategies such as:

* Index Scan
* Bitmap Index Scan
* Sequential Scan
* Nested Loop
* Hash Join
* Merge Join

A Python loop cannot directly benefit from PostgreSQL's query planner after the data has already been retrieved.

---

## 4. Avoid Fetching Unnecessary Columns

If only a few fields are required, don't retrieve complete model objects unnecessarily.

Instead of:

```python
employees = Employee.objects.all()
```

use:

```python
employees = Employee.objects.values(
    "id",
    "name",
    "email",
)
```

Or:

```python
employee_ids = Employee.objects.values_list(
    "id",
    flat=True,
)
```

This reduces memory usage and data transfer.

---

## 5. Use Database Aggregation

Avoid calculating totals in Python:

```python
total_salary = 0

for employee in Employee.objects.filter(is_active=True):
    total_salary += employee.salary
```

Prefer:

```python
from django.db.models import Sum

total_salary = Employee.objects.filter(
    is_active=True
).aggregate(
    total=Sum("salary")
)["total"]
```

Conceptually:

```sql
SELECT SUM(salary)
FROM employee
WHERE is_active = TRUE;
```

Instead of transferring all salaries to Python, the database can return a single aggregated value.

---

## 6. Use `count()` Instead of Counting With a Loop

Avoid:

```python
count = 0

for employee in employees:
    count += 1
```

Prefer:

```python
count = employees.count()
```

This allows the database to perform the counting operation.

---

## 7. Use `exists()` for Existence Checks

Avoid:

```python
found = False

for employee in employees:
    if employee.email == email:
        found = True
        break
```

Prefer:

```python
found = Employee.objects.filter(
    email=email
).exists()
```

The application only needs a boolean result:

```text
True / False
```

There is no reason to retrieve an entire dataset.

---

## 8. Avoid the N+1 Query Problem

ORM itself does not guarantee good performance.

This can be problematic:

```python
employees = Employee.objects.all()

for employee in employees:
    print(employee.department.name)
```

If the relationship is not optimized, this can result in:

```text
1 query → employees

+ 1 query → department of employee 1
+ 1 query → department of employee 2
+ 1 query → department of employee 3
...
```

For 1,000 employees, this could become approximately:

```text
1 + 1,000 = 1,001 queries
```

Use:

```python
employees = Employee.objects.select_related(
    "department"
)
```

For many-to-many or reverse relationships:

```python
employees = Employee.objects.prefetch_related(
    "roles"
)
```

The goal is to minimize unnecessary database round trips.

---

## 9. ORM Does Not Mean "Always Faster"

Do not use ORM blindly.

Performance depends on:

* Table size
* Indexes
* Query complexity
* Number of joins
* Query plan
* Data volume
* Network latency
* Number of queries
* Python processing
* Database configuration

A complex ORM query can also be inefficient.

Therefore:

> **Measure before and after optimization.**

For example:

```python
queryset = Employee.objects.filter(
    department_id=10,
    is_active=True,
)

print(queryset.explain())
```

For PostgreSQL, `EXPLAIN ANALYZE` can be used for deeper investigation.

---

## 10. When Python Loops Are Appropriate

Python loops are not bad.

Use Python when the operation is genuinely application-level.

For example:

```python
for employee in employees:
    employee.display_name = format_employee_name(employee)
```

or:

```python
for employee in employees:
    calculate_complex_business_rule(employee)
```

Complex business logic, external API calls, custom algorithms, and application orchestration often belong in Python.

The preferred architecture is:

```text
Database
   ↓
Filter / Join / Aggregate
   ↓
Required data
   ↓
Python
   ↓
Business/Application Logic
```

Not:

```text
Database
   ↓
Large dataset
   ↓
Python
   ↓
Filter everything
   ↓
Aggregate everything
   ↓
Business Logic
```

---

## 11. Practical Decision Rule

Before writing:

```python
for item in queryset:
```

ask:

### Can the database do this?

Consider:

```python
filter()
exclude()
annotate()
aggregate()
count()
exists()
order_by()
distinct()
```

### Do I need related objects?

Consider:

```python
select_related()
prefetch_related()
```

### Do I need complete model objects?

If not:

```python
values()
values_list()
```

### Do I need all records?

If not:

```python
filter()
```

### Is the query slow?

Measure it:

```python
queryset.explain()
```

---

## 12. Complexity Is Not the Only Factor

Both approaches may appear to be `O(n)` from a simple algorithmic perspective.

However, database-backed applications have additional costs:

```text
Total Cost ≈

Database execution
+ Network transfer
+ Object creation
+ Memory usage
+ Python processing
+ Number of database queries
```

Therefore, simply saying:

> "Both are O(n), so performance is the same"

is incorrect.

The amount of data transferred and where the computation occurs are equally important.

---

## 13. Recommended Responsibility Boundary

### Database

Use the database for:

```text
WHERE
JOIN
GROUP BY
COUNT
SUM
AVG
MIN
MAX
EXISTS
ORDER BY
DISTINCT
Pagination
```

### Python

Use Python for:

```text
Complex business rules
Custom algorithms
External API calls
Application orchestration
Non-SQL-compatible transformations
ML/LLM inference
```

---

## 14. Golden Rule

> **Push data-intensive operations as close to the data as practical, and keep Python responsible for application-level logic.**

The goal is **not** to eliminate Python loops.

The goal is to avoid unnecessarily moving large amounts of database data into Python just to perform work that the database can efficiently perform itself.

A good Django application generally follows:

```text
                 PostgreSQL
                     │
       ┌─────────────┼─────────────┐
       │             │             │
    Filtering       Join       Aggregation
       │             │             │
       └─────────────┼─────────────┘
                     ↓
              Required Data
                     ↓
                  Django
                     ↓
          Business/Application Logic
                     ↓
                  Response
```

### Final Principle

**Don't ask Python to process data that the database can efficiently filter, join, aggregate, or validate before sending it to the application.**
