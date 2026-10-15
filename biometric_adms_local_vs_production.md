# Biometric ADMS Integration: Local vs Production

## 1. Overview

This document explains how the ZKTeco K45 Pro biometric device communicates with a Django application during local development and after production deployment.

**Key principle:** The biometric device and Django server do not need to be on the same local network if the device can reach the server over the internet or a suitable private network.

In production, nobody needs to keep the HRMS website open on a laptop. The production backend must remain running and reachable.

---

## 2. Current Local Setup

The Django development server is started using:

```bash
python manage.py runserver 0.0.0.0:8000
```

Example network configuration:

| Component | IP address | Purpose |
|---|---|---|
| ZKTeco K45 Pro | `192.168.1.18` | Biometric device |
| Laptop running Django | `192.168.1.37` | Local ADMS receiver |
| Django port | `8000` | HTTP development server |

The biometric device is configured to send ADMS requests to the laptop's LAN address:

```text
192.168.1.37:8000
```

### What does `0.0.0.0` mean?

`0.0.0.0` tells Django to listen on all available IPv4 network interfaces.

It is a listening address, not the IP address to configure on the biometric machine.

The machine must use an address it can reach, such as `192.168.1.37` on the same LAN.

### Why does the current setup work?

- The laptop and biometric device are connected to the same local network.
- The device can reach the laptop's private IP.
- Django is listening on port `8000`.
- Firewall rules permit the connection.
- Django's ADMS endpoints receive and process the HTTP requests.

**Limitation:** If the laptop shuts down, disconnects from the network, or stops running Django, the device cannot reach this local receiver.

---

## 3. Local Architecture

```text
┌───────────────────────────────┐
│ ZKTeco K45 Pro                │
│ IP: 192.168.1.18              │
└──────────────┬────────────────┘
               │
               │ HTTP / ADMS over LAN
               ▼
┌───────────────────────────────┐
│ Developer laptop              │
│ IP: 192.168.1.37              │
│ Django: port 8000             │
│ ADMS endpoints                │
└──────────────┬────────────────┘
               │
               ▼
┌───────────────────────────────┐
│ Development database          │
│ Attendance records            │
└───────────────────────────────┘
```

This architecture is suitable for testing but depends on the laptop and local network remaining available.

---

## 4. Production Architecture

In production, the deployed server replaces the laptop as the ADMS receiver.

```text
┌───────────────────────────────┐
│ ZKTeco K45 Pro                │
│ Company office network        │
└──────────────┬────────────────┘
               │
               │ Internet or private network/VPN
               ▼
┌───────────────────────────────┐
│ Production domain / public IP │
│ DNS, firewall, routing, TLS   │
└──────────────┬────────────────┘
               ▼
┌───────────────────────────────┐
│ Production application server │
│ Nginx + Gunicorn + Django     │
│ ADMS endpoints                │
└──────────────┬────────────────┘
               ▼
┌───────────────────────────────┐
│ Production database           │
│ Attendance records            │
└───────────────────────────────┘
```

Example production domain:

```text
https://hrms.example.com
```

This is a placeholder domain. Replace it with the actual production domain or reachable server address.

### Does the biometric device need the same network?

**No.** If the production server is reachable over the internet, the device can communicate with it from the office network.

For example:

- The biometric device is in the company office.
- Django runs on a cloud server.
- The device sends ADMS requests to the server's public domain or IP.
- Django processes attendance records and saves them to the production database.

Both systems do not need to share the same private LAN.

If the server is private-only, an appropriate VPN or private network route may be required.

---

## 5. Does Someone Need to Keep the Website Open?

**No.**

The website being open in a browser does not control whether the backend can receive attendance data.

| Local development | Production |
|---|---|
| Laptop runs Django. | Production server runs Django. |
| Laptop must remain available. | Production server must remain available. |
| Device connects to laptop's private IP. | Device connects to a reachable production address. |
| Current setup relies on the local network. | Internet routing or private connectivity can be used. |
| Closing the development process stops the receiver. | Closing a browser does not stop the backend service. |

You can shut down your laptop and leave the office. The biometric device can continue synchronizing attendance, provided the production server, network, database, and device remain operational.

---

## 6. Production Requirements

| Requirement | Local development | Production |
|---|---|---|
| ADMS address | Laptop LAN IP | Public domain/IP or reachable private address |
| Port | `8000` | Commonly HTTPS `443`, or another configured port |
| Application server | Django development server | Gunicorn behind Nginx or equivalent |
| Network | Usually the same LAN | Internet routing, VPN, or private network |
| Database | Development database | Production database |
| Availability | Laptop and development process | Production services must remain available |
| Security | Isolated testing environment | TLS, firewall restrictions, and access controls |
| Monitoring | Manual checks | Logs, health checks, and device connectivity monitoring |

The port is not automatically `443`. The device, reverse proxy, TLS configuration, and ADMS endpoint must use compatible settings. Verify the K45 Pro firmware's supported ADMS configuration.

### Required steps

1. Deploy Django to a production server.
2. Configure a production application server.
3. Configure the production domain and DNS, if applicable.
4. Configure Nginx or an equivalent reverse proxy.
5. Enable HTTPS where supported.
6. Configure firewall rules and network routing.
7. Point the biometric device to the actual production endpoint.
8. Verify ADMS requests reach Django.
9. Confirm attendance is stored in the production database.

---

## 7. Network and Security Considerations

### Private IP addresses

Addresses such as `192.168.1.37` and `192.168.1.18` are private LAN addresses.

A cloud server cannot ordinarily reach these addresses directly across the public internet. Likewise, a biometric device cannot reach a cloud server using its private IP without a suitable private route.

### Prefer device-initiated connections

In an ADMS push/polling architecture, the device commonly initiates HTTP requests to the configured server.

This generally avoids exposing the biometric device itself directly to the public internet. Confirm the exact behavior supported by your device and firmware.

### Avoid exposing the development server

This command is useful for local testing:

```bash
python manage.py runserver 0.0.0.0:8000
```

It is not the recommended production serving configuration.

Do not expose arbitrary ports or enable port forwarding without understanding the security implications. Prefer a secured production endpoint and least-privilege firewall rules.

---

## 8. Production Testing Checklist

- [ ] The production application is deployed and running.
- [ ] The production domain resolves to the correct server.
- [ ] The intended ADMS endpoint is reachable from the device's network.
- [ ] The device is configured with the correct server address and port.
- [ ] Firewall and routing rules permit the required requests.
- [ ] The reverse proxy forwards ADMS paths correctly.
- [ ] Django receives requests from the biometric device.
- [ ] The device serial number maps to the correct device record.
- [ ] Attendance records are saved to the production database.
- [ ] Duplicate submissions and retries are handled safely.
- [ ] Logs and last-seen monitoring help identify connection failures.
- [ ] A test punch succeeds when the HRMS website is closed.
- [ ] The device's buffering and retry behavior is tested during temporary outages.

For this application, inspect the implemented endpoints, such as:

```text
/iclock/cdata
/iclock/getrequest
/iclock/devicecmd
```

These paths must match the routes actually configured in the production application.

---

## 9. Troubleshooting

| Problem | What to check |
|---|---|
| Device cannot connect | Server address, port, DNS, internet connectivity, firewall, TLS compatibility |
| Website works but no attendance arrives | ADMS routing, device settings, device polling behavior, proxy logs |
| Requests arrive but records are missing | Django logs, payload parsing, serial-number mapping, database errors |
| Duplicate punches appear | Idempotency and deduplication logic |
| Sync stops when laptop is shut down | Device may still point to the laptop's private IP |
| Works locally but fails in production | Public routing, DNS, firewall, proxy configuration, or HTTPS compatibility |

Opening the HRMS website successfully does not prove that the device can reach the ADMS endpoints. Test the actual device-to-server request path.

---

## 10. Final Conclusion

The current local setup confirms that the biometric device can communicate with the Django ADMS receiver over the local network.

Production uses the same general integration flow, but the biometric device must point to a server address it can reach.

**Production does not require the laptop or browser to remain open.** It requires a continuously available production backend, database, network connection, and correctly configured ADMS endpoint.

The critical requirement is not that both systems share the same network. It is:

**Reliable network reachability between the biometric device and the production ADMS endpoint.**
