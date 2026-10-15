# LAN (Local Area Network)

## 1. What Is a LAN?

**LAN stands for Local Area Network.**

It connects devices within a limited area, such as a home, office, school, or building.

Devices on the same LAN can communicate with each other when routing and firewall settings allow it.

Examples of devices connected to a LAN:
- Laptop or desktop computer
- Wi-Fi router
- Printer
- Biometric attendance machine
- Local server

## 2. Example: Our Biometric Integration

In the current local development setup:

| Component | Example IP | Purpose |
|---|---|---|
| ZKTeco K45 Pro | `192.168.1.18` | Sends attendance data |
| Laptop running Django | `192.168.1.37` | Receives ADMS requests |
| Django port | `8000` | Application listener |

Django is started using:

```bash
python manage.py runserver 0.0.0.0:8000
```

The biometric device sends an HTTP request to an endpoint such as:

```text
http://192.168.1.37:8000/iclock/cdata
```

This example assumes the endpoint exists in the application and is configured correctly.

### What Does `0.0.0.0` Mean?

`0.0.0.0` tells Django's development server to listen on all available IPv4 network interfaces.

It is not the IP address to configure on the biometric device.

The device uses the laptop's reachable LAN IP, such as `192.168.1.37`.

## 3. How Does a LAN Work?

```text
       Office Wi-Fi Router
                |
        ┌───────┴────────┐
        |                |
        ▼                ▼
 Biometric Machine   Laptop
 192.168.1.18        192.168.1.37
                          |
                          ▼
                    Django :8000
                          |
                          ▼
                    Attendance DB
```

The router or network switch connects devices. IP addresses identify devices, while routing and firewall rules determine whether communication is permitted.

Being connected to the same Wi-Fi does not always guarantee communication. Guest Wi-Fi isolation, VLANs, and firewall rules can block traffic.

## 4. What Is a Private IP Address?

A private IP address is used inside a private network, such as an office LAN.

Common private IPv4 ranges are:

- `10.0.0.0` – `10.255.255.255`
- `172.16.0.0` – `172.31.255.255`
- `192.168.0.0` – `192.168.255.255`

For example, `192.168.1.18` is a private IP address.

It is ordinarily not directly reachable from an unrelated network over the public internet.

## 5. LAN vs Internet

| LAN | Internet |
|---|---|
| Connects devices within a local network. | Connects networks around the world. |
| Commonly uses private IP addresses. | Uses public addressing and routing between networks. |
| Example: biometric machine communicating with a laptop in the same office. | Example: biometric machine communicating with a cloud server. |
| Usually managed by a local router or network administrator. | Depends on internet connectivity, routing, DNS, and firewall rules. |

## 6. Why Does Production Not Require the Same LAN?

### Local development

The biometric device reaches Django through the laptop's private IP address.

```text
Biometric device
       |
       | Same LAN
       ▼
Laptop: 192.168.1.37:8000
       |
       ▼
Django ADMS endpoint
```

### Production deployment

The application runs on a production server with a reachable public domain or IP address.

```text
Biometric device
       |
       ▼
Office router
       |
       ▼
Internet
       |
       ▼
Production domain / public IP
       |
       ▼
Nginx → Django → Production database
```

The biometric machine and production server can be on different networks because the device can send requests through the internet to the server's reachable address.

**The requirement is network reachability, not a shared LAN.**

For this to work:

1. The device must be configured with the actual production ADMS address and port.
2. DNS must resolve correctly if a domain is used.
3. Routing and firewall rules must allow the connection.
4. The production web server must route ADMS requests to Django.
5. Django and the production database must remain available.

A VPN or another private network connection can also connect separate networks when needed.

## 7. Does the Laptop Need to Stay Open?

**During local development:** Yes. The laptop must remain powered on, connected to the network, and running Django because it acts as the server.

**During production:** No. The developer's laptop and browser do not need to remain open. The deployed server runs the backend independently.

Use a suitable production application server rather than Django's development server.

## 8. Final Takeaway

A LAN allows nearby devices to communicate within a local network.

The current biometric integration works because the device can reach the laptop through the LAN.

After deployment, the device can communicate with a server on another network through the internet or a suitable private network.

**Local development uses LAN reachability. Production uses public or private network reachability.**

The essential requirement is that the biometric device can reach the configured ADMS endpoint.
