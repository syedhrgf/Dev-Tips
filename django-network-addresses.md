# Understanding Django Network Addresses

## 1. Starting Django

Run the following command in your Django project directory:

```bash
python manage.py runserver 0.0.0.0:8000
```

This tells Django to listen on **all available IPv4 network interfaces** on port `8000`.

**Important:** `0.0.0.0` is a bind address. It is not the address you enter in your browser.

## 2. Accessing the Application

After starting Django, you can generally access the application through these URLs on your laptop.

### A. Localhost

```text
http://127.0.0.1:8000/
```

- `127.0.0.1` is the loopback address.
- It refers to your own laptop.
- It works locally without needing another device on the network.

### B. Local Network IP

```text
http://192.168.1.37:8000/
```

- `192.168.1.37` is the laptop's local network IP in this example.
- Other devices on the same reachable network may access Django through this address.
- The laptop's firewall and network configuration must allow the connection.
- The IP may change if the router assigns a different address.

## 3. What Does `0.0.0.0` Mean?

When you run:

```bash
python manage.py runserver 0.0.0.0:8000
```

Django listens on all available IPv4 interfaces instead of listening only on localhost.

| Address | Purpose |
|---|---|
| `127.0.0.1:8000` | Access the application locally |
| `192.168.1.37:8000` | Access it through the laptop's local network interface |
| `0.0.0.0:8000` | Tell Django which interfaces to listen on |

The same Django application can be reachable through both `127.0.0.1` and `192.168.1.37` when it is bound to `0.0.0.0`, provided the relevant interfaces and network configuration permit it.

## 4. Can Another Laptop Access It?

Suppose your laptop runs Django at:

```text
192.168.1.37:8000
```

Your friend's laptop has a different IP, for example:

```text
192.168.1.42
```

Your friend would open:

```text
http://192.168.1.37:8000/
```

Your friend uses **the server laptop's IP**, not their own IP.

This works only if both devices can communicate over the network and the firewall allows port `8000`.

## 5. Does This Make Django Public on the Internet?

No. Binding Django to `0.0.0.0` does not automatically expose it to the internet.

For internet access, you need an appropriately configured public server or network setup, along with suitable firewall rules and security controls.

For production, do not expose Django's development server directly to the internet. Use a production deployment setup with HTTPS, authentication, authorization, and appropriate network restrictions.

## 6. Relevance to the ZKTeco K45 Pro

During local ADMS testing, the biometric device can send requests to your laptop's local IP, for example:

```text
http://192.168.1.37:8000/iclock/cdata
```

This assumes the endpoint path matches your implementation and the device can reach the laptop over the network.

For a deployed environment, the device would instead use the configured public receiver hostname and supported protocol, for example:

```text
https://erp.example.com/iclock/cdata
```

The actual hostname and path must match your deployment and device firmware.

## Key Takeaway

**`0.0.0.0` controls where Django listens; `127.0.0.1` and `192.168.1.37` are addresses clients can use to reach it.** The application remains on your laptop until you deploy it elsewhere.
