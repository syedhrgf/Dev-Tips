# ZKTeco K45 Pro Integration: ADMS-Based vs Custom HTTP Approach

## 1. Purpose

This note compares two ways to receive biometric attendance data in a Django application:

1. **ADMS-compatible receiver** — implement the device-facing endpoints and request/response behavior expected by a compatible ZKTeco device.
2. **Custom HTTP API** — define your own endpoints and payload contract, then send data to them using a client or a device that explicitly supports that custom contract.

These approaches are not interchangeable. A device configured to send ADMS requests will not automatically understand an arbitrary custom API.

> **Important terminology:** ADMS is a device communication protocol/workflow, not one universal Python SDK package. An ADMS-compatible receiver can be implemented directly in Django. A separate SDK is optional and must specifically support the device, firmware, and operations you need.

---

## 2. The Difference

| Area | ADMS-compatible receiver | Custom HTTP API |
|---|---|---|
| Endpoint paths | Uses paths and request patterns expected by the device, often `/iclock/...` | Uses routes you define, such as `/api/biometrics/punches/` |
| Request format | Follows the device's actual ADMS payload format | Follows your own documented JSON or other payload format |
| Responses | Must match the device firmware's expected response format | Defined by your API contract |
| Device compatibility | Suitable when the device supports the ADMS workflow you implement | Suitable only if the device can be configured to call your custom API, or if a separate gateway/client translates the device protocol |
| Control over schema | Constrained by device payloads and protocol behavior | You define the schema |
| SDK required? | Not necessarily; a compatible receiver can be implemented directly | Not necessarily; standard Django HTTP views are enough |
| Best use | Direct device-to-server integration | Application-to-application integration or a custom gateway |

### Important distinction

Writing a Django function that receives `/iclock/cdata` is not automatically a complete ADMS implementation. The handler must also correctly support the relevant device request formats, parameters, acknowledgements, and any required polling/command behavior.

Likewise, a custom endpoint is not an ADMS endpoint just because it receives attendance data over HTTP.

---

## 3. Approach A — ADMS-Compatible Receiver

### 3.1 Typical flow

```text
ZKTeco K45 Pro
    |
    | HTTP request expected by the device's ADMS workflow
    v
/iclock/cdata
    |
    v
Django ADMS view
    |
    v
ADMS service/parser
    |
    v
AttendanceLog database table
    |
    v
Protocol-compatible response
```

Common ZKTeco-style ADMS integrations may use routes such as `/iclock/cdata` for data exchange and `/iclock/getrequest` for polling. Exact endpoints, payload formats, and response text must be verified against the specific device and firmware.

### 3.2 Example parser for the observed whitespace format

The following sample is based on a payload observed in your test environment:

```text
ASD 2026-10-14 14:54:01 0 1 0 0 0 0 0 0 11204
```

For this specific assumed format:

```text
PIN DATE TIME STATUS VERIFY_MODE WORK_CODE [EXTRA FIELDS...]
```

Use a parser like this only after confirming the field order for the device:

```python
# biometrics/services/adms.py

import re
from datetime import datetime

from django.utils import timezone


DATE_RE = re.compile(r"^\d{4}-\d{2}-\d{2}$")
TIME_RE = re.compile(r"^\d{2}:\d{2}:\d{2}$")


def parse_adms_attendance_line(line: str) -> dict | None:
    """Parse one verified whitespace-delimited ATTLOG line."""
    raw_line = line.rstrip("\r\n")
    parts = raw_line.strip().split()

    # Expected: PIN DATE TIME STATUS [VERIFY_MODE] [WORK_CODE] ...
    if len(parts) < 4:
        return None

    if not DATE_RE.fullmatch(parts[1]):
        return None

    if not TIME_RE.fullmatch(parts[2]):
        return None

    timestamp_text = f"{parts[1]} {parts[2]}"

    try:
        naive_timestamp = datetime.strptime(
            timestamp_text,
            "%Y-%m-%d %H:%M:%S",
        )
    except ValueError:
        return None

    # This assumes the device timestamp represents Asia/Kolkata local time.
    # Confirm this assumption and your Django TIME_ZONE/USE_TZ settings.
    aware_timestamp = timezone.make_aware(
        naive_timestamp,
        timezone=timezone.get_current_timezone(),
    )

    return {
        "employee_id": parts[0],
        "punch_time": aware_timestamp,
        "punch_status": parts[3],
        "verify_mode": parts[4] if len(parts) > 4 else "",
        "work_code": parts[5] if len(parts) > 5 else "",
        "raw_line": raw_line,
    }
```

**Timezone warning:** The example uses Django's configured current timezone. Ensure it is set to the intended timezone before using this in production. Do not assume the device timestamp is UTC or local time without checking device configuration.

### 3.3 Example ADMS data endpoint

This is an illustrative Django view. It shows the shape of a receiver, not a guarantee of the exact response required by every K45 Pro firmware.

```python
# biometrics/views.py

import logging

from django.http import HttpResponse
from django.views.decorators.csrf import csrf_exempt
from django.views.decorators.http import require_http_methods

from .models import BiometricDevice
from .services.adms import parse_adms_attendance_line, save_parsed_records

logger = logging.getLogger(__name__)


@csrf_exempt  # Scope this exemption to the device-facing endpoint only.
@require_http_methods(["GET", "POST"])
def adms_cdata(request):
    serial = request.GET.get("SN", "").strip()

    if not serial:
        return HttpResponse(
            "Missing serial number",
            status=400,
            content_type="text/plain; charset=utf-8",
        )

    device = BiometricDevice.objects.filter(
        serial_number=serial,
        is_active=True,
    ).first()

    if device is None:
        logger.warning("ADMS request from unregistered/inactive device: %s", serial)
        return HttpResponse(
            "Unknown device",
            status=403,
            content_type="text/plain; charset=utf-8",
        )

    if request.method == "GET":
        # Return the actual device configuration response required by
        # the firmware. Do not copy a generic response without testing it.
        return HttpResponse(
            "OK",
            content_type="text/plain; charset=utf-8",
        )

    table = request.GET.get("table", "").strip().upper()
    if table != "ATTLOG":
        # Other tables may need their own handlers, depending on firmware.
        return HttpResponse(
            "OK",
            content_type="text/plain; charset=utf-8",
        )

    if len(request.body) > 1024 * 1024:
        return HttpResponse("Payload too large", status=413)

    body = request.body.decode("utf-8", errors="replace")

    records = []
    rejected_lines = 0

    for line in body.splitlines():
        if not line.strip():
            continue

        parsed = parse_adms_attendance_line(line)
        if parsed is None:
            rejected_lines += 1
            logger.warning(
                "Rejected malformed ADMS attendance line for serial=%s",
                serial,
            )
            continue

        records.append(parsed)

    # Implement this service with the actual model fields and a tested
    # duplicate strategy. It should save records transactionally.
    save_parsed_records(device=device, serial=serial, records=records)

    logger.info(
        "ADMS ATTLOG processed: serial=%s accepted=%s rejected=%s",
        serial,
        len(records),
        rejected_lines,
    )

    # The acknowledgement must be confirmed against the actual firmware.
    return HttpResponse("OK", content_type="text/plain; charset=utf-8")
```

The imports and helper `save_parsed_records()` above are illustrative; you must implement that helper to match your actual models and duplicate policy. If your current service already has `save_attendance(serial, body)`, you can call it instead, after adding registered-device validation and confirming the protocol response.

**Do not deploy the generic `OK` responses blindly.** Your current device may require a particular configuration payload on a GET request, and a specific acknowledgement format for data uploads. Use observed device traffic and reliable device documentation to confirm the response.

### 3.4 Example URL routing

```python
# biometrics/urls.py

from django.urls import path
from . import views

urlpatterns = [
    path("iclock/cdata", views.adms_cdata, name="adms_cdata"),
]
```

Include this URL configuration from the project's root URL configuration. Confirm whether the device requests paths with or without a trailing slash.

### 3.5 What a complete ADMS receiver may need

Depending on firmware and enabled features, you may need to implement:

- Device identification and registration checks.
- Data upload and the expected acknowledgement.
- GET configuration/options response.
- Device polling for server commands.
- Command-result handling.
- Device time/configuration behavior if required.
- Additional data tables if enabled.
- Duplicate protection, error logging, and payload validation.

Implement only the protocol behavior required by the device, but verify what it expects before declaring the integration complete.

---

## 4. Approach B — Custom HTTP API

A custom API is appropriate when your own client or a gateway sends data to Django in a schema you control. It is **not automatically compatible with the K45 Pro's ADMS mode**.

### 4.1 Typical flow

```text
Custom client or gateway
    |
    | POST JSON to your own API contract
    v
/api/biometrics/punches/
    |
    v
Django validates JSON and authorization
    |
    v
Application service
    |
    v
AttendanceLog database table
```

For example, a custom client could send:

```http
POST /api/biometrics/punches/ HTTP/1.1
Content-Type: application/json
Authorization: Bearer <token>
```

```json
{
  "device_serial": "WED3240400871",
  "employee_id": "ASD",
  "punch_time": "2026-10-14T14:54:01+05:30",
  "punch_status": "0",
  "verify_mode": "1",
  "work_code": "0"
}
```

This JSON structure is **your own API schema**, not the raw ADMS payload. A client or gateway must transform the device's data into this shape.

### 4.2 Example custom Django endpoint

This simple example validates a JSON body and demonstrates where application persistence belongs. It assumes a bearer-token authentication layer is added by your project; it does not implement authentication itself and should not be exposed publicly as-is.

```python
# biometrics/custom_api_views.py

import json
from datetime import datetime

from django.http import JsonResponse
from django.views.decorators.http import require_POST
from django.views.decorators.csrf import csrf_exempt
from django.utils.dateparse import parse_datetime

from .models import BiometricDevice


@csrf_exempt  # Only suitable when replaced/supplemented by explicit API authentication.
@require_POST
def custom_punch_api(request):
    if len(request.body) > 64 * 1024:
        return JsonResponse({"error": "Payload too large"}, status=413)

    try:
        payload = json.loads(request.body)
    except (json.JSONDecodeError, UnicodeDecodeError):
        return JsonResponse({"error": "Invalid JSON"}, status=400)

    if not isinstance(payload, dict):
        return JsonResponse({"error": "JSON object required"}, status=400)

    required = (
        "device_serial",
        "employee_id",
        "punch_time",
        "punch_status",
        "verify_mode",
    )
    missing = [key for key in required if not str(payload.get(key, "")).strip()]
    if missing:
        return JsonResponse(
            {"error": "Missing required fields", "fields": missing},
            status=400,
        )

    serial = str(payload["device_serial"]).strip()
    device = BiometricDevice.objects.filter(
        serial_number=serial,
        is_active=True,
    ).first()

    if device is None:
        return JsonResponse({"error": "Unknown or inactive device"}, status=403)

    punch_time = parse_datetime(str(payload["punch_time"]))
    if punch_time is None or punch_time.utcoffset() is None:
        return JsonResponse(
            {"error": "punch_time must be an ISO-8601 timestamp with timezone"},
            status=400,
        )

    # Authenticate the client before accepting data in production.
    # Then pass validated data to a shared persistence service that:
    #   1. maps the employee identifier,
    #   2. applies a defined duplicate policy,
    #   3. writes in a transaction,
    #   4. records a safe diagnostic outcome.
    #
    # Do not write directly until the actual AttendanceLog fields and
    # uniqueness constraints have been confirmed.

    return JsonResponse(
        {
            "accepted": True,
            "device_serial": serial,
            "message": "Validated; connect this point to the persistence service.",
        },
        status=202,
    )
```

This example deliberately returns `202 Accepted` after validation to show the API contract. It does **not** save an attendance row yet. Connect a tested persistence service before treating it as a working attendance endpoint.

### 4.3 Example custom API URL

```python
# biometrics/urls.py

from django.urls import path
from . import custom_api_views

urlpatterns = [
    path(
        "api/biometrics/punches/",
        custom_api_views.custom_punch_api,
        name="custom_punch_api",
    ),
]
```

This URL is for your own client or gateway. Do not change the K45 Pro's ADMS destination to this route unless the device can be configured to send the required custom JSON contract.

---

## 5. Which Approach Should You Use?

For your current setup, the device is configured to send ADMS-style requests to your local server:

- Server IP: `192.168.1.37`
- Port: `8000`
- Example device endpoint: `/iclock/cdata`
- Example serial number: `WED3240400871`

Therefore, **continue with the ADMS-compatible receiver** for direct communication with the device. You already have custom parsing and persistence functions; those can form the service layer underneath the ADMS endpoint.

Use a custom JSON API if you later build a separate gateway, a mobile/web client, or another service that submits normalized attendance events. You can reuse the same validation and persistence service for both inputs, while keeping each input adapter separate.

Suggested architecture:

```text
                         +--------------------------+
K45 Pro -- ADMS HTTP --->| ADMS view / ADMS adapter |
                         +-------------+------------+
                                       |
                                       v
                         +--------------------------+
Custom client -- JSON -->| Custom API adapter       |
                         +-------------+------------+
                                       |
                                       v
                         +--------------------------+
                         | Shared validation and    |
                         | attendance persistence   |
                         +-------------+------------+
                                       |
                                       v
                         +--------------------------+
                         | AttendanceLog database   |
                         +--------------------------+
```

This avoids duplicating database logic while keeping the device protocol separate from your custom API.

---

## 6. Important Improvements for Your Current Code

Your existing service functions are a useful starting point:

- `parse_attendance_line()` — payload parser.
- `record_device_request()` — diagnostic request logger.
- `save_attendance()` — attendance persistence.
- `mark_device_seen()` — last-seen tracking.

Before production, review these items:

1. **Validate the serial number and active device** before accepting attendance records.
2. **Confirm actual field positions** from K45 Pro payloads. Do not assume all formats are whitespace-delimited or that status codes have universal meanings.
3. **Validate timestamps**, including timezone interpretation. A string shaped like a date is not necessarily a valid date.
4. **Use database uniqueness constraints** for the chosen event identity; `get_or_create()` alone is not a substitute for a well-defined concurrency-safe uniqueness policy.
5. **Bound request size before fully processing the body.** Slicing `request.body` after Django has already read it does not prevent Django from receiving the whole body into memory.
6. **Protect diagnostics.** Query strings and raw bodies may contain sensitive information; limit access and retention.
7. **Do not overwrite configured IP information blindly** with the source IP of each request. Consider a separate `last_request_ip`.
8. **Confirm protocol responses** from the device documentation and observed requests. A response that works for one endpoint may be incorrect for another.
9. **Keep device events separate from HR calculations.** A raw punch code does not itself calculate worked hours or overtime.
10. **Use authentication and network controls** appropriate to the device's capabilities. A serial number alone is not strong authentication.

---

## 7. Key Terms

| Term | Meaning |
|---|---|
| ADMS receiver | Server-side component that implements the relevant device communication behavior |
| ADMS SDK | A software library, if available for the device, that abstracts some protocol operations; not required for every receiver |
| Protocol adapter | Code that converts a particular input protocol into your application's internal data format |
| Custom API | An endpoint and payload contract designed by your application |
| Parser | Code that converts raw text or bytes into structured fields |
| Persistence service | Code that validates and stores normalized attendance records |
| Idempotency | Processing retries without creating unintended duplicate records |
| Acknowledgement | Server response that tells the device how its request was handled |
| Gateway | A service that communicates with the device and forwards normalized data to another API |

---

## 8. Summary

- **ADMS-compatible approach:** implement the request paths, payload parsing, and response behavior expected by the K45 Pro.
- **Custom API approach:** design your own HTTP/JSON contract for a client or gateway that supports it.
- Both can use Django and Python; neither automatically requires a separate SDK.
- Your current code is a custom implementation of parts of an ADMS receiver, not a dedicated SDK integration.
- For your direct K45 Pro connection, finish and test the ADMS-compatible path first.
