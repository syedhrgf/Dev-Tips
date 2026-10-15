# Biometric Management — Django + ADMS Integration

## 1. Purpose

This document explains the biometric management module, the ADMS communication flow, the Django project structure, device configuration, attendance logs, diagnostics, troubleshooting, and a practical learning roadmap.

The goal is to build a maintainable Django application that can:

- Register and manage biometric devices.
- Receive attendance events from compatible devices.
- Store and review punch records.
- Diagnose communication and parsing problems.
- Safely integrate validated punch data with official HR attendance calculations.

> **Important:** Device protocol details can vary by model and firmware. Confirm endpoint behavior, payload field meanings, and response requirements against the actual device traffic and the applicable ZKTeco documentation.

---

## 2. What Is ADMS?

[**ADMS**](https://github.com/syedhrgf/Dev-Tips/blob/main/ADMS-Based%20vs%20Custom%20HTTP%20Approach.md) is commonly expanded in the ZKTeco device ecosystem as **Automatic Data Master Server**. It describes a device-to-server communication arrangement used by compatible devices to exchange data with a server. Many ADMS integrations use HTTP requests.

In this project:

- The **biometric device** records a punch event.
- The **ADMS receiver** accepts the device's HTTP request.
- The **Django service layer** validates and parses the request.
- The **database** stores the attendance record and relevant diagnostics.
- The **management UI** allows authorized users to review devices, punches, and communication issues.

ADMS is not the same thing as Django, HTTP, WebSocket, or the ZKTeco SDK.

### 2.1 ADMS compared with related technologies

| Term | Meaning | Role in this project |
|---|---|---|
| ADMS | Device-to-server communication arrangement used by compatible devices | Defines how the device exchanges data with the server |
| HTTP | Request/response protocol | Carries device requests and server responses in common ADMS setups |
| Django | Python web framework | Routes requests, enforces permissions, and provides application logic |
| WebSocket | Persistent, bidirectional communication protocol | Different from ordinary ADMS HTTP polling |
| ZKTeco SDK / `pyzk` | Library/protocol interface for supported devices | A different communication option; support depends on the device and firmware |
| Port 8000 | A network port | The configured ADMS destination port in this local test setup |
| Port 4370 | Commonly used by some ZKTeco device SDK/protocol connections | Not interchangeable with the ADMS HTTP port |

A device using ADMS on port `8000` does not need to expose port `4370` for that HTTP integration to work.

---

## 3. High-Level Architecture

```text
Employee
   |
   | Fingerprint / configured authentication method
   v
Biometric Device
   |
   | HTTP request using the device's configured ADMS destination
   v
Django URL Router
   |
   v
ADMS Endpoint / View
   |
   | Validate request parameters and payload
   v
Biometrics Service Layer
   |
   +---- Parse attendance line
   +---- Identify device
   +---- Validate and normalize fields
   +---- Prevent duplicate records
   +---- Record diagnostics
   |
   v
Database
   |
   +---- Device registry
   +---- Attendance logs
   +---- Diagnostic/request records
   |
   v
Authorized Management UI
   |
   +---- Device Configuration
   +---- Attendance Logs
   +---- ADMS Diagnostics
```

This is a conceptual flow. Exact endpoints and payloads depend on device firmware and configuration.

---

## 4. Suggested Django Project Structure

```text
biometrics/
├── models.py
├── views.py
├── urls.py
├── forms.py
├── services/
│   ├── __init__.py
│   └── adms.py
├── templates/
│   └── biometrics/
│       ├── dashboard.html
│       ├── device_list.html
│       ├── device_form.html
│       ├── attendance.html
│       └── requests.html
└── admin.py
```

### 4.1 `models.py` — Data models

Responsibilities:

- Define the biometric device registry.
- Define attendance log records.
- Define request or diagnostic records if those are persisted.
- Define relationships, field constraints, and useful indexes.
- Store timestamps and device identity consistently.

Learn:

- Django models and fields.
- `ForeignKey` and relationships.
- `Meta.constraints` and unique constraints.
- Database indexes.
- Migrations.
- `select_related()` and `prefetch_related()`.

Example conceptual entities:

- `BiometricDevice`
- `AttendanceLog`
- `DeviceRequest` or an equivalent diagnostic model

Use the actual model names already present in your project; these are examples, not a requirement to create duplicate models.

### 4.2 `views.py` — Web request handling

Responsibilities:

- Render the dashboard.
- List, create, edit, activate, or deactivate devices.
- Show attendance logs with search, filters, and pagination.
- Show diagnostic records.
- Enforce authentication and authorization.
- Validate incoming web form data.

Views should coordinate a request rather than contain the entire ADMS parser and all attendance business rules.

Learn:

- Function-based or class-based views.
- `login_required` and permission checks.
- `render()`, redirects, and HTTP responses.
- QuerySet filtering.
- Pagination.
- Error handling.

### 4.3 `urls.py` — URL routing

Responsibilities:

- Map URL paths to the correct views.
- Keep management pages and device-facing endpoints clear.
- Avoid accidental route collisions.
- Make endpoint names consistent and testable.

Example conceptual routes:

```python
urlpatterns = [
    # Management pages
    path("biometrics/", views.dashboard, name="biometrics_dashboard"),
    path("biometrics/devices/", views.device_list, name="biometrics_device_list"),
    path("biometrics/attendance/", views.attendance_list, name="biometrics_attendance"),
    path("biometrics/requests/", views.request_list, name="biometrics_requests"),

    # Device-facing routes should point to the project's actual ADMS views.
    # Example: path("iclock/cdata", adms_views.biometrics_cdata),
]
```

This is an illustrative example. Adapt the names and imports to your real project.

### 4.4 `forms.py` — Form validation

Responsibilities:

- Validate serial numbers and device metadata.
- Validate IP addresses or hostnames if these are accepted.
- Validate location and activation settings.
- Display useful validation errors.
- Prevent users from creating conflicting device records.

Form validation improves the user experience, but database constraints should enforce critical uniqueness requirements too.

### 4.5 `services/adms.py` — ADMS protocol and processing logic

Responsibilities may include:

- Extracting the serial number and table name from request parameters.
- Parsing attendance payloads.
- Validating payload fields.
- Looking up the registered device.
- Saving attendance records transactionally.
- Preventing duplicate records.
- Returning processing counts or structured outcomes to the view.
- Recording useful diagnostic information without leaking sensitive data.

Keep the protocol-specific parsing here rather than duplicating it across multiple views.

### 4.6 `templates/biometrics/` — User interface

| Template | Purpose |
|---|---|
| `dashboard.html` | Summary cards, device health, latest activity, and links to key pages |
| `device_list.html` | Device registry, search, status, and device actions |
| `device_form.html` | Register or edit device details |
| `attendance.html` | Punch records, filters, event labels, and pagination |
| `requests.html` | ADMS request diagnostics and troubleshooting information |

Reuse the existing base template, design tokens, and UI components used by the main application.

### 4.7 `admin.py` — Django Admin

Responsibilities:

- Register relevant models.
- Provide useful list columns and filters.
- Support safe inspection of records by authorized administrators.
- Avoid exposing sensitive raw payloads to users who do not need them.

The Django Admin is a development and administration aid; it does not replace a purpose-built management UI.

---

## 5. Feature Area A — Device Configuration

### Purpose

Register biometric devices and maintain their identity and configuration.

### Suggested fields

| Field | Purpose |
|---|---|
| Device name | Human-readable name |
| Serial number | Device identity reported in ADMS requests |
| Network address | Configured or known device address, if applicable |
| Location | Office, branch, gate, or other installation location |
| Active flag | Indicates whether the application should accept data from the device |
| Last seen | Last recorded communication time |
| Created/updated timestamps | Audit and maintenance |
| Notes | Non-sensitive operational notes |

The exact fields should match the existing model and operational requirements.

### Recommended functionality

- Add a device.
- Edit its metadata.
- Activate or deactivate it.
- Search by name, serial number, or location.
- Show last communication time.
- Display a connection indicator based on recent requests, without claiming that a single old request proves the device is currently online.
- Reject or flag unknown serial numbers.
- Prevent duplicate serial numbers.
- Audit important configuration changes.

### Device identity versus IP address

The **serial number** identifies the device in the ADMS protocol. The **source IP address** is the network address from which a request was received. They serve different purposes.

A device may use DHCP or communicate through a network gateway. Do not blindly overwrite the configured device address every time a request arrives. If needed, store the observed source IP separately as `last_request_ip`.

---

## 6. Feature Area B — Attendance Logs

### Purpose

Review attendance events received from devices.

### Recommended columns

- Employee PIN or device user identifier.
- Device serial number or registered device.
- Punch timestamp reported by the device.
- Punch/event status.
- Verification mode.
- Work code, if present.
- Server received timestamp.
- Processing status or validation error, where useful.

### Recommended controls

- Search by employee PIN or serial number.
- Filter by date range and event type.
- Sort by punch time or received time.
- Paginate at the database QuerySet level.
- Display an empty state when no records match.
- Keep raw payload access restricted to authorized diagnostic users.
- Show a clear distinction between event status and authentication method.
- Prevent duplicate storage.
- Log malformed lines rather than silently treating them as valid attendance.

### Punch time versus received time

These are different timestamps:

- **Punch time:** time reported by the biometric device for the event.
- **Received time:** time the server received the request.

If they differ, investigate device clock configuration, timezone handling, parsing, and database storage. Do not automatically add or subtract a fixed offset without identifying the cause.

### Event status versus verification mode

- **Punch/event status** describes the type of event reported by the device, according to its configured protocol.
- **Verification mode** describes the authentication method, such as fingerprint, card, or password, when the device reports that information.

Numeric codes are firmware/protocol dependent. Verify the meaning of each code before presenting it as “Check In,” “Check Out,” “Break,” or “Overtime.”

An event code does not by itself calculate working duration or overtime. Official attendance calculations need validated punches, shift rules, breaks, missing-punch handling, and company policy.

---

## 7. Feature Area C — ADMS Diagnostics

### Purpose

Help developers and authorized administrators understand whether the device is communicating with the application and where processing failures occur.

### Useful diagnostic fields

| Field | Why it matters |
|---|---|
| Received timestamp | Establishes when a request arrived |
| HTTP method | Shows whether the device sent GET or POST |
| Endpoint/path | Identifies which route was called |
| Device serial | Associates the request with a device |
| Source IP | Helps diagnose network paths |
| Table/command parameters | Shows the request's declared purpose |
| HTTP response status | Indicates the server's response |
| Processing outcome | Shows accepted, rejected, ignored, or failed processing |
| Error summary | Explains a failure without exposing secrets |
| Duration | Helps identify slow processing |
| Raw payload | Helps debug protocol issues; restrict access and retention |

Not every field needs to be stored permanently. Choose an appropriate retention period and avoid logging credentials, biometric templates, or unnecessary personal information.

### Common diagnostic questions

1. Did the device reach the server?
2. Did Django resolve the requested URL?
3. Did the request include the expected serial number?
4. Did the endpoint receive the expected HTTP method?
5. Was the payload parsed correctly?
6. Was the device registered and active?
7. Did validation or database persistence fail?
8. Did the server return the response expected by the firmware?

---

## 8. Common ADMS Endpoints

These names are common in ZKTeco-style ADMS integrations, but exact behavior must be confirmed against the device and firmware.

| Endpoint | Typical purpose |
|---|---|
| `/iclock/cdata` | Data exchange, including attendance uploads in common integrations |
| `/iclock/getrequest` | Device polls the server for pending commands |
| `/iclock/devicecmd` | Command-response or result exchange in some integrations |
| `/iclock/registry` | Registration-related behavior in some implementations |
| `/iclock/ping` | Often a custom health-check route if the application implements it |

Do not assume every firmware uses every endpoint, or that every endpoint must be implemented exactly as described above. Observe real device requests and consult the applicable protocol documentation.

### Example request

A device may send a request resembling:

```http
POST /iclock/cdata?SN=WED3240400871&table=ATTLOG&Stamp=9999 HTTP/1.1
Content-Type: text/plain
```

The query string can identify the device and the data table. The request body contains the payload.

The server should:

1. Extract and validate the serial number.
2. Confirm that the device is registered and active.
3. Check the declared table or request type.
4. Parse and validate the payload.
5. Store valid records safely.
6. Record a useful processing outcome.
7. Return the correct protocol response for the device firmware.

The example above is illustrative; use the actual headers, request format, and response requirements observed from the device.

---

## 9. Parsing Attendance Payloads

A sample observed during local testing:

```text
ASD 2026-10-14 14:54:01 0 1 0 0 0 0 0 0 11204
```

A possible whitespace-delimited layout is:

```text
PIN DATE TIME STATUS VERIFY_MODE WORK_CODE EXTRA_FIELDS...
```

For that layout, the first fields would be:

| Position | Sample value | Tentative meaning |
|---|---|---|
| 1 | `ASD` | Device employee PIN/user identifier |
| 2 | `2026-10-14` | Date |
| 3 | `14:54:01` | Time |
| 4 | `0` | Event status |
| 5 | `1` | Verification mode |
| 6 | `0` | Work code |
| 7 onward | `0 ... 11204` | Additional fields whose meanings must be verified |

**These meanings are tentative until verified against the actual K45 Pro firmware/protocol.** Do not assume that every trailing value has a known meaning.

For this particular whitespace-delimited sample, a basic extraction might look like:

```python
def parse_whitespace_attendance_line(line: str) -> dict | None:
    parts = line.strip().split()

    # PIN + date + time + status are the minimum assumed fields.
    if len(parts) < 4:
        return None

    employee_id = parts[0]
    date_text = parts[1]
    time_text = parts[2]

    return {
        "employee_id": employee_id,
        "punch_time_text": f"{date_text} {time_text}",
        "punch_status": parts[3],
        "verify_mode": parts[4] if len(parts) > 4 else "",
        "work_code": parts[5] if len(parts) > 5 else "",
        "raw_line": line.rstrip("\r\n"),
    }
```

This is only an example for the whitespace-delimited layout above. It is not a universal ZKTeco parser. Some payloads use tab delimiters or combine date and time into one token, which changes the field positions. If the device can send multiple formats, detect and validate the format explicitly rather than guessing.

### Parsing best practices

- Preserve the raw line for troubleshooting.
- Validate date and time formats.
- Reject or quarantine malformed records.
- Do not silently swap status and verification fields.
- Handle multiple lines in one request.
- Treat blank lines appropriately.
- Keep parsing separate from database writes.
- Write unit tests from real, anonymized payload samples.
- Track accepted, rejected, duplicate, and failed record counts.
- Confirm the protocol's acknowledgement behavior before deciding what response to send.

---

## 10. Database Integrity and Duplicate Prevention

A device may retry requests, so the receiver should tolerate repeated payloads.

### Recommended safeguards

- Define a stable event identity if the protocol provides one.
- Otherwise, use a carefully selected composite key based on verified event fields.
- Add a database uniqueness constraint where the chosen identity is reliable.
- Use transactions when saving a batch.
- Handle integrity errors caused by duplicate submissions.
- Keep raw data available for authorized troubleshooting.
- Do not assume a `get_or_create()` call alone guarantees concurrency safety without an appropriate database constraint.

A composite key such as device, employee PIN, punch time, status, and verification mode may be useful, but evaluate whether two legitimate events can share those values before enforcing it.

---

## 11. Timezones and Timestamp Handling

The device configuration in the local test setup reports a timezone value of `5.5`, corresponding to UTC+05:30. That setting does not by itself guarantee that timestamps are interpreted or displayed correctly by Django.

### Recommended approach

1. Check the device's clock and timezone configuration.
2. Record the raw timestamp string during debugging.
3. Parse it according to the actual device format.
4. Decide whether the device timestamp represents local time or UTC.
5. Store timestamps consistently using Django's timezone-aware settings.
6. Display them in the intended local timezone.
7. Compare device punch time with server received time.
8. Test daylight-saving and timezone assumptions if the system will operate across regions.

Do not add a fixed two-hour correction just because two stored timestamps differ. Determine whether the discrepancy comes from parsing, timezone conversion, device clock settings, or display formatting.

---

## 12. Network and Local Test Configuration

The current local test configuration is:

| Setting | Value |
|---|---|
| Device | ZKTeco K45 Pro |
| Device serial | `WED3240400871` |
| ADMS server IP | `192.168.1.37` |
| ADMS port | `8000` |
| Attendance endpoint | `/iclock/cdata` |

These are local test values, not production settings. A local private IP is only reachable within the appropriate network path.

### Network checklist

- The computer and device can reach each other through the network.
- The server listens on the intended interface, not only on `127.0.0.1`, when another device must connect.
- The configured server IP matches the computer's current reachable IP.
- The configured ADMS port matches the listening server port.
- Firewall rules permit the required inbound traffic.
- Django URL routing includes the required endpoint paths.
- The device's serial number matches the registered device record.
- The device's HTTP requests appear in server logs.

For a development server, a command such as the following binds to all local interfaces:

```bash
python manage.py runserver 0.0.0.0:8000
```

Use Django's development server only for local development and controlled testing. Use an appropriately secured production application server and deployment configuration for production.

---

## 13. Troubleshooting Guide

| Symptom | Checks |
|---|---|
| Device cannot connect | Confirm server IP, port, routing, firewall, and that the server is listening |
| Device reaches IP but endpoint returns 404 | Verify the URL patterns and exact path, including trailing-slash behavior |
| Requests arrive but no attendance rows are saved | Check HTTP method, table parameter, raw body, parser assumptions, validation, and database errors |
| Duplicate attendance records | Inspect the event identity strategy and database constraints |
| Wrong status labels | Confirm payload field order and firmware-specific code mappings |
| Verification mode displayed as event type | Check that the template reads `punch_status` and `verify_mode` from the correct fields |
| Punch and received times differ | Check device clock, raw timestamp, timezone handling, and conversion/display logic |
| Device keeps polling | Check whether the endpoint returns the expected response and whether pending commands are handled correctly |
| Unknown device serial | Compare request `SN` with the registered serial and inspect the device lookup |
| Attendance list pagination is missing | Check that the view creates a `Paginator`, the template uses the page object, and the pagination include is rendered |
| A device appears online incorrectly | Define a documented last-seen threshold and distinguish last communication from live reachability |

Always diagnose from the raw request and logs before changing parsing or field mappings.

---

## 14. Security and Reliability

A biometric integration handles attendance data and device-facing endpoints. Treat it as a security-sensitive system.

- Require authentication and authorization for management pages.
- Restrict device registration and activation to authorized administrators.
- Validate serial numbers, parameters, payload lengths, and field formats.
- Avoid trusting a serial number alone as strong authentication.
- Use a supported device authentication or network security mechanism where available.
- Keep secrets out of source control and logs.
- Restrict access to raw payloads and diagnostic pages.
- Avoid storing biometric templates unless there is a specific, approved need.
- Add rate limits or network restrictions where appropriate.
- Use database constraints and transactions.
- Record failures with enough context to debug them without exposing unnecessary personal data.
- Define retention and cleanup policies for diagnostic logs.
- Use HTTPS where supported by the deployment and device configuration.
- Keep the device and application on a controlled network where practical.
- Test malformed payloads and duplicate requests.

A CSRF exemption may be required for a device endpoint that cannot submit Django CSRF tokens, but it should be scoped to that endpoint only. Compensate with suitable validation, authentication, network controls, and protocol-specific safeguards.

---

## 15. Testing Strategy

### Unit tests

Test the parser with:

- A valid whitespace-delimited line.
- A valid tab-delimited line if the device sends that format.
- Missing fields.
- Invalid dates or times.
- Empty lines.
- Extra fields.
- Unknown status and verification codes.
- Non-ASCII or unexpected employee identifiers where relevant.

### Service tests

Test:

- Registered active device.
- Unknown device serial.
- Inactive device.
- One valid attendance record.
- Multiple lines in one payload.
- Duplicate record.
- Malformed line mixed with valid lines.
- Database failure and transaction behavior.

### View and integration tests

Test:

- Expected HTTP methods.
- Correct endpoint routing.
- Required query parameters.
- Correct protocol response.
- Permission checks on management pages.
- Search, event filtering, and pagination.
- Device last-seen updates.
- Diagnostics for successful and failed requests.

Use anonymized real payload samples for regression tests. Do not use real biometric templates or unnecessary personal information in test fixtures.

---

## 16. Suggested Learning Roadmap

### Stage 1 — Django foundations

Learn models, QuerySets, migrations, views, URL routing, forms, templates, permissions, and pagination.

**Goal:** Be able to trace a browser request through a Django view to a database query.

### Stage 2 — HTTP fundamentals

Learn request methods, headers, query strings, request bodies, content types, status codes, ports, and network binding.

**Goal:** Explain the difference between the URL, query parameters, headers, and body of a device request.

### Stage 3 — ADMS communication

Observe actual requests from the device. Identify the serial number, endpoint, HTTP method, table parameter, body format, and expected response.

**Goal:** Explain one real device request from arrival to response.

### Stage 4 — Parsing and persistence

Implement a tested parser, validate timestamps, save records transactionally, and prevent duplicates.

**Goal:** Reliably turn an anonymized raw payload into a correctly stored attendance record.

### Stage 5 — Diagnostics

Add structured logs, processing outcomes, last-seen information, and a searchable diagnostics page.

**Goal:** Find whether a failure occurred in networking, routing, parsing, validation, or persistence.

### Stage 6 — HR attendance rules

Map device PINs to employee records and implement approved shift, break, missing-punch, late-arrival, and overtime rules.

**Goal:** Separate raw device events from calculated HR attendance.

### Stage 7 — Production readiness

Review permissions, network restrictions, authentication, logging, retention, database indexes, backups, monitoring, and deployment.

**Goal:** Operate the integration safely and reliably outside a local test environment.

---

## 17. Glossary

| Term | Definition |
|---|---|
| ADMS | Device-to-server communication mechanism used by compatible biometric devices |
| Device serial number | Device identity reported in communication requests |
| Employee PIN | Identifier for a user registered on the device; it must be mapped to the correct HR employee |
| Attendance event | A raw punch record reported by a device |
| Punch status | Device-reported event code |
| Verification mode | Device-reported authentication method code |
| Work code | Optional code reported in some attendance payloads |
| Raw payload | Original request body or line before parsing |
| Received timestamp | Server time when the request or record was received |
| Idempotency | Ability to process a retried request without creating unintended duplicate effects |
| QuerySet | Django's database query abstraction |
| Transaction | A group of database operations handled as a unit |
| Diagnostic log | Record used to understand request processing and failures |
| Device polling | Device periodically requesting information or commands from a server |

---

## 18. Final Design Principles

1. **Keep protocol logic in the service layer.**
2. **Treat device payloads as untrusted input.**
3. **Verify field meanings using actual firmware behavior.**
4. **Preserve raw samples when debugging parsing errors.**
5. **Separate event status from verification mode.**
6. **Separate punch time from received time.**
7. **Make record persistence safe against retries and concurrency.**
8. **Keep raw events separate from official HR calculations.**
9. **Protect diagnostic information and management actions.**
10. **Test the full path from network request to database record.**

The most useful first milestone is to trace one real punch through the complete path:

**Device → HTTP request → Django route → ADMS parser → validation → database → attendance page.**
