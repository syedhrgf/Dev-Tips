# ADMS Explained  — ZKTeco K45 Pro (Machine)

## 1. What Is ADMS?

**ADMS** stands for **Automatic Data Master Server** in the context of ZKTeco devices.

It is a device-management and communication mechanism that allows a compatible biometric device to communicate with a server, commonly using HTTP-based requests.

For example, a ZKTeco K45 Pro can be configured to send attendance data to a server running a Django application.

ADMS is not a Python library. It is also not simply another name for HTTP. The device firmware determines the particular ADMS endpoints, request formats, and responses it supports.

> **Important:** ADMS implementations can vary by device model and firmware. Always verify the exact request format against the device's documentation or actual network traffic.

---

## 2. Understand the Difference Between ADMS, HTTP, and a Library

These terms describe different things.

| Term           | Meaning                                          | Example                                     |
| -------------- | ------------------------------------------------ | ------------------------------------------- |
| ADMS           | Device-management and communication mechanism    | Biometric device communicates with a server |
| HTTP           | Protocol used to exchange requests and responses | `POST /iclock/cdata`                        |
| Django         | Python web framework                             | Receives HTTP requests                      |
| Python library | Reusable code that provides functionality        | A package that parses attendance records    |
| API endpoint   | URL where an application receives a request      | `/iclock/cdata`                             |
| Database       | Stores attendance records                        | PostgreSQL or SQLite                        |

### Simple analogy

Imagine an office:

- **ADMS** is the communication arrangement between the biometric machine and the server.
- **HTTP** is the language used to exchange requests and responses.
- **Django** is the receptionist that receives the requests.
- **Your service functions** process the information.
- **The database** stores the attendance records.

You do not necessarily need a dedicated ADMS Python library. You can implement the required HTTP endpoints yourself, provided they match the device's expectations.

---

## 3. How the Biometric Device Communicates with Django

Suppose your setup is:

- Biometric device: ZKTeco K45 Pro
- Laptop IP: `192.168.1.37`
- Device IP: `192.168.1.18`
- Server port: `8000`
- Django application: running on the laptop

The device is configured with the laptop's IP address and port.

The communication flow looks like this:

```text
ZKTeco K45 Pro
      |
      | HTTP request
      | POST /iclock/cdata
      | Attendance data
      v
Django Application
      |
      v
URL Routing
      |
      v
biometrics_cdata()
      |
      v
save_attendance()
      |
      v
AttendanceLog Model
      |
      v
Database
```

In this setup, the biometric device initiates the HTTP request to the server. Your application does not need to connect directly to the device's attendance database.

The precise request sequence depends on the device firmware and configuration.

---

## 4. ADMS vs Traditional ZKTeco Communication

ZKTeco devices may support different communication methods.

| Feature                     | ADMS-style communication                 | Traditional device communication                                |
| --------------------------- | ---------------------------------------- | --------------------------------------------------------------- |
| Communication pattern       | Device sends requests to a server        | Application connects to the device                              |
| Common transport            | HTTP                                     | Vendor-specific protocol over TCP                               |
| Commonly encountered port   | `8000`, depending on configuration       | `4370`, depending on configuration                              |
| Django HTTP endpoints       | Typically used                           | Not used directly for the traditional protocol                  |
| Possible Python integration | Django views and custom parsing          | A compatible library such as `pyzk`                             |
| Typical use                 | Attendance uploads and device management | Reading device data or performing supported operations directly |

**Ports are not universal standards.** Confirm the actual communication method and port supported by your model and configuration.

For example, a timeout when using `pyzk` on port `8000` does not automatically mean the device is offline. It may mean the application is using the wrong protocol for that port.

---

## 5. What Is an ADMS Endpoint?

An endpoint is a URL that receives requests.

Your Django application may expose endpoints such as:

| Endpoint             | Possible purpose                                           |
| -------------------- | ---------------------------------------------------------- |
| `/iclock/cdata`      | Receive attendance data or provide configuration responses |
| `/iclock/getrequest` | Allow the device to request pending commands               |
| `/iclock/devicecmd`  | Handle device command-related communication                |
| `/iclock/registry`   | Handle registration-related communication if supported     |
| `/biometrics/ping`   | Custom endpoint to test server connectivity                |

The first four paths resemble endpoints used in some ZKTeco communication implementations. Whether a particular device calls them, and what it expects from them, must be verified against the firmware.

The `/biometrics/ping` endpoint is a custom application endpoint, not necessarily a standard ADMS endpoint.

### Example

A device might send:

```http
POST /iclock/cdata?SN=WED3240400871&table=ATTLOG HTTP/1.1
Host: 192.168.1.37:8000
Content-Type: text/plain
```

The request body might contain attendance records.

The exact query parameters, headers, and body format depend on the device.

---

## 6. Understanding Your Django Implementation

Your implementation is divided into views and service functions.

### A. Django views

Views receive HTTP requests and return HTTP responses.

For example:

```python
@csrf_exempt
@require_http_methods(["GET", "POST"])
def biometrics_cdata(request):
    serial = request.GET.get("SN", "")
    table = request.GET.get("table", "").upper()

    record_device_request(request)
    mark_device_seen(serial)

    if request.method == "POST":
        if table in {"ATTLOG", "ATTLOGS"}:
            save_attendance(serial, request.body)

        return plain_response("OK")

    return plain_response("OK")
```

This is a simplified example of the request flow, not a complete replacement for your existing implementation. Your real GET response may need to return device configuration text rather than just `OK`.

### B. `plain_response()`

This function sends a plain-text response:

```python
from django.http import HttpResponse


def plain_response(text="OK"):
    return HttpResponse(
        text,
        content_type="text/plain; charset=utf-8",
    )
```

The response body must match what the device expects for that request.

### C. `record_device_request()`

This function records incoming requests for debugging and auditing.

It can help you inspect:

- Which endpoint the device contacted
- The HTTP method
- The serial number supplied in the request
- Query parameters
- The request body, subject to size limits

Be careful not to log sensitive employee information unnecessarily.

### D. `mark_device_seen()`

This function updates the registered device's last-seen time and, when available, its observed IP address.

It helps your application determine whether a device has recently communicated with the server.

### E. `parse_attendance_line()`

This function converts a raw attendance line into structured data.

For example, an illustrative positional record could look like:

```text
1001    2026-10-15 09:02:30    0    1    0
```

A parser might interpret it as:

- Employee PIN: `1001`
- Punch time: `2026-10-15 09:02:30`
- Status: `0`
- Verification mode: `1`
- Work code: `0`

These field meanings are illustrative. Confirm the column order and status codes for your device's actual payload.

Your implementation should also support the formats your device actually sends, rather than assuming all devices use the same layout.

### F. `save_attendance()`

This service function is responsible for:

1. Finding the registered biometric device.
2. Reading the received attendance payload.
3. Splitting the payload into individual records.
4. Parsing each record.
5. Validating employee identifiers and timestamps.
6. Saving attendance records.
7. Avoiding duplicate records.

Conceptually:

```text
Raw HTTP body
     |
     v
Split into lines
     |
     v
Parse each line
     |
     v
Validate record
     |
     v
Check duplicate
     |
     v
Save AttendanceLog
```

A production implementation should distinguish between records that were newly created, already existed, or failed validation.

---

## 7. Why Does the Device Send `SN` and `table`?

A request may contain parameters such as:

```text
SN=WED3240400871
table=ATTLOG
```

### `SN`

`SN` commonly identifies the device's serial number.

Your application can use it to find the corresponding `BiometricDevice` record.

Do not trust the serial number alone as authentication. Restrict access to registered devices and consider network-level controls or additional verification.

### `table`

The `table` parameter identifies the category of data involved in the request.

For example, `ATTLOG` commonly refers to attendance logs in compatible ZKTeco implementations.

Your view can use it to decide which parser or service should process the request.

---

## 8. Why Does the Server Respond with `OK`?

A device usually expects a response after sending an HTTP request.

For example:

```http
HTTP/1.1 200 OK
Content-Type: text/plain; charset=utf-8

OK
```

This indicates that the server returned a successful HTTP response with an `OK` body.

However, **returning `OK` does not prove that the attendance record was saved correctly**.

A robust implementation should:

- Validate the incoming data.
- Save valid records reliably.
- Log parsing and database errors.
- Return the response expected by the device firmware.
- Avoid acknowledging data as successfully processed when it has not been persisted, where the device's retry behavior permits this distinction.

Some devices expect a particular configuration response during GET requests. For those requests, returning only `OK` may be insufficient.

---

## 9. What Is `GET OPTION FROM`?

Some ZKTeco communication implementations use a configuration response resembling:

```text
GET OPTION FROM: WED3240400871
ErrorDelay=30
Delay=10
TransTimes=00:00;14:05
TransInterval=1
TransFlag=TransData AttLog
Realtime=1
Encrypt=None
TimeZone=5.5
```

This is an illustrative example, not a verified configuration for every K45 Pro firmware version.

Your Django view may return such a response when the device requests configuration.

The individual settings can affect communication behavior, data transfer, or scheduling. Their exact meanings and valid values should be confirmed against the device documentation.

In particular:

- Do not assume every setting is supported.
- Do not assume every firmware version interprets settings identically.
- Verify whether the device expects this response at the specific endpoint.
- Confirm the timezone convention expected by the firmware.

---

## 10. What Are `getrequest` and `devicecmd`?

These endpoints may participate in command polling and device management.

### `/iclock/getrequest`

A device may periodically request pending commands from the server.

A basic implementation that always returns `OK` can acknowledge the request but does not provide a real command queue.

If you need server-to-device commands, you may need to implement:

1. A database model for queued commands.
2. A mechanism to associate commands with a device.
3. A polling response in the expected format.
4. Command identifiers and status tracking.
5. Acknowledgement and retry handling.

### `/iclock/devicecmd`

A device may use a command-related endpoint to communicate command results or acknowledgements.

Simply recording the request and returning `OK` does not necessarily process those results.

Implement the exact behavior only after confirming which command exchange your firmware supports.

---

## 11. What Is the Difference Between a Custom HTTP API and ADMS?

A custom HTTP API is an API whose paths and request formats you define.

For example:

```text
POST /api/attendance/
```

You can decide what JSON fields it accepts and how authentication works.

An ADMS-compatible integration is different because the device firmware expects specific communication behavior, request formats, endpoint paths, and response conventions.

For example:

```text
POST /iclock/cdata?SN=...&table=ATTLOG
```

You can write the endpoint yourself in Django, but the implementation must follow the behavior expected by the device.

**The key distinction is compatibility, not whether you use a library.**

You do not need to install a dedicated ADMS package to write Django endpoints. You do need to implement the required communication conventions correctly.

Also, an arbitrary custom HTTP API will not automatically work with the device unless the firmware supports that API or can be configured to use it.

---

## 12. Security and Reliability Checklist

Before using the integration with real employee attendance, check the following.

- [ ] Only registered, active devices can submit attendance.
- [ ] Incoming payloads have a reasonable maximum size.
- [ ] Serial numbers and employee identifiers are validated.
- [ ] Timestamps and status codes are parsed safely.
- [ ] Duplicate punches are handled consistently.
- [ ] Invalid records are logged without exposing unnecessary personal information.
- [ ] Database writes use appropriate transaction handling.
- [ ] Device time and application timezone are configured correctly.
- [ ] The server is reachable from the device's network.
- [ ] Firewall rules allow the required traffic.
- [ ] HTTP responses match the firmware's expectations.
- [ ] Command polling and acknowledgement behavior is implemented if required.
- [ ] Production deployments use suitable network restrictions and transport security where supported.

**Important:** Truncating `request.body` after accessing it does not prevent Django from first reading a large request into memory. Enforce appropriate request-size limits at the web server and application layers.

---

## 13. How to Test the Integration

### Step 1: Confirm network connectivity

From the laptop, test whether the device is reachable:

```powershell
ping 192.168.1.18
```

A successful ping indicates IP-level reachability for ICMP, not that the ADMS service is working.

### Step 2: Start Django on the correct interface

For a development environment:

```bash
python manage.py runserver 0.0.0.0:8000
```

This makes the development server listen on available network interfaces. It is not a production deployment recommendation.

### Step 3: Confirm the laptop address

On Windows:

```powershell
ipconfig
```

Verify that the laptop's active network interface uses the IP address configured in the biometric device.

### Step 4: Inspect Django logs

Check whether requests arrive at:

```text
/iclock/cdata
/iclock/getrequest
/iclock/devicecmd
```

Also inspect your stored `DeviceRequest` records.

### Step 5: Perform a test punch

Use an approved test employee and record the request body, serial number, table name, and response.

Compare the actual payload with the assumptions made by `parse_attendance_line()`.

### Step 6: Verify the database

Check that the attendance record exists in `AttendanceLog` with the correct employee, device, timestamp, and status.

Repeat a test if necessary to verify that duplicate submissions do not create duplicate attendance records.

---

## 14. Final Mental Model

Remember these five points:

1. **ADMS** describes a device-management and communication mechanism; exact behavior varies by implementation.
2. **HTTP** is the transport protocol used by your HTTP-based integration.
3. **Django views** receive the device's requests.
4. **Service functions** parse and validate the data before saving it.
5. **The database** stores the resulting attendance records.

Your architecture is therefore:

```text
Biometric Device
       |
       | HTTP / ADMS-style communication
       v
Django URL Routing
       |
       v
Django Views
       |
       v
Biometric Services
       |
       v
Django Models
       |
       v
Database
```

The most important next step is to capture the actual requests sent by your K45 Pro and confirm the firmware's expected response formats. That will tell you whether your current endpoints are genuinely compatible with the device's ADMS implementation, rather than merely accepting HTTP requests.
