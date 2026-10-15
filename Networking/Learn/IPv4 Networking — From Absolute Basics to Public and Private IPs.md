# IPv4 Networking — From Absolute Basics to Public and Private IPs

Let's start from zero and build up to the point where you can understand exactly how your laptop, ZKTeco K45 Pro biometric machine, router, and production Django server communicate.

The most important idea to understand first is this:

IPv4 is a system for addressing devices on networks. A LAN is a network. Private and public IP addresses are categories of addresses. They are related, but they are not the same thing.

## 1. First, why do we need a network?

Imagine you have three devices in your office:

Laptop

Printer

Biometric

Network switch or router

Connects devices and/or other networks

You might want to:

- Print a document from your laptop.
- Send attendance data from your biometric machine to Django.
- Access a server running on another computer.
- Browse a website on the internet.

For these devices to communicate, they need a way to send data, identify destinations, and determine where that data should go.

That's where networking comes in.

## 2. What is an IP address?

IP stands for Internet Protocol.

An IP address is a logical network address used to identify a network interface as a source or destination for IP packets.

For example:

```
Laptop:             192.168.1.37
Biometric machine:  192.168.1.18
Router:             192.168.1.1
```

These are example private IPv4 addresses.

When your biometric machine sends a request to your laptop, the destination IP tells the network where the packet is intended to go.

Think of an IP address as the destination address on a parcel. The analogy is not perfect, but it helps explain why addressing matters.

An IP address alone does not guarantee delivery. The devices also need appropriate network connectivity, routing, and permission to communicate.

## 3. What exactly is IPv4?

IPv4 stands for Internet Protocol version 4.

It is one version of the Internet Protocol, which provides addressing and packet delivery between networks.

An IPv4 address contains 32 bits.

For example:

```
192.168.1.37
```

This address is written in a human-readable format consisting of four decimal numbers separated by dots. Each number is called an octet.

An octet contains 8 bits.

Therefore:

\\[ 4 \times 8 = 32 \text{ bits} \\]

### How does a computer represent this address?

Every bit can be either `0` or `1`.

The address `192.168.1.37` can be represented in binary as:

```
192       168       1         37
11000000.10101000.00000001.00100101
```

Each octet contains eight bits.

For example, the decimal number 192 is calculated as:

\\[ 128+64=192 \\]

The binary representation is:

```
11000000
```

The eight bit positions represent these values:

| Bit position | Decimal value |
| ------------ | ------------- |
| 1            | 128           |
| 2            | 64            |
| 3            | 32            |
| 4            | 16            |
| 5            | 8             |
| 6            | 4             |
| 7            | 2             |
| 8            | 1             |

Each octet can represent a number from 0 to 255.

That is why an ordinary dotted-decimal IPv4 address has four numbers, each between 0 and 255.

### How many IPv4 addresses exist?

Because IPv4 contains 32 bits, there are:

\\[ 2^{32}=4,294,967,296 \\]

That is approximately 4.29 billion possible bit patterns.

However, not all of these addresses are available for ordinary public internet use. Some are reserved for private networks, loopback, multicast, documentation, and other special purposes.

Also, an address's public or private classification is not determined merely by its appearance as four decimal numbers. The ranges have defined meanings.

## 4. Does every IPv4 address represent a private network?

No. This is where the distinction becomes important.

IPv4 is the addressing system. Within IPv4, different address ranges have different purposes.

## IPv4 address space

The overall set of 32-bit IPv4 addresses

Private IPv4

Used inside private networks

Example: `192.168.1.37`

Public IPv4

Globally routable address space

Example: an assigned cloud-server IP

Loopback

A computer communicating with itself

Example: `127.0.0.1`

Other special ranges

Link-local, multicast, documentation, and more

The categories above are not all interchangeable. For example, loopback addresses are not used to identify remote internet servers, and multicast addresses have a different purpose from ordinary unicast addresses.

## 5. What is a private IPv4 address?

A private IPv4 address is an address reserved for use within private networks. The three private IPv4 ranges are defined in RFC 1918.

| Private range                     | CIDR notation    | Number of addresses |
| --------------------------------- | ---------------- | ------------------- |
| `10.0.0.0` – `10.255.255.255`     | `10.0.0.0/8`     | 16,777,216          |
| `172.16.0.0` – `172.31.255.255`   | `172.16.0.0/12`  | 1,048,576           |
| `192.168.0.0` – `192.168.255.255` | `192.168.0.0/16` | 65,536              |

These ranges can be reused by many unrelated organizations because they are not globally unique public addresses.

For example, your office might use:

```
Laptop:              192.168.1.37
Biometric machine:   192.168.1.18
Router:              192.168.1.1
```

Another company can also use `192.168.1.37` inside its own network. That does not create a conflict because the two private networks are separate.

### Why can't the internet directly route these private addresses?

Internet routers do not ordinarily forward packets across the public internet when their destination is an RFC 1918 private address.

Suppose your biometric machine tries to access:

```
192.168.1.37
```

This works when that address refers to your laptop on a reachable local network.

But a biometric machine in a different city cannot simply send a public internet request to `192.168.1.37` and expect to reach your laptop. That private address could exist in thousands of different networks.

The remote machine needs another route, such as a public endpoint, a VPN, or another private network connection.

## 6. What is a public IPv4 address?

A public IPv4 address belongs to address space intended for global routing, subject to allocation and routing policies.

A hosting provider might assign a public IPv4 address to a virtual machine, or a network provider might assign one to a router.

For example, suppose your deployed Django server has a real public IP assigned by its provider.

```
Production server
Public IPv4: assigned by hosting provider
Application: Django
Web server: Nginx
```

A biometric device in your office can potentially send an HTTP request to that server, even if the office and server are on entirely different networks.

But a public IP alone does not guarantee that a connection will work. The server needs a reachable route, a listening service, correct firewall rules, and an application that handles the request.

Public IP means globally routable address space, not unrestricted access to every service.

A public server can still block every incoming connection using a firewall.

## 7. What are the other important IPv4 ranges?

To understand IPv4 properly, it helps to know a few additional categories.

| Range             | Purpose                                                     |
| ----------------- | ----------------------------------------------------------- |
| `127.0.0.0/8`     | Loopback addresses                                          |
| `169.254.0.0/16`  | IPv4 link-local addresses                                   |
| `100.64.0.0/10`   | Shared address space often used by carrier-grade NAT        |
| `224.0.0.0/4`     | IPv4 multicast                                              |
| `240.0.0.0/4`     | Reserved for future or special use, with limited exceptions |
| `192.0.2.0/24`    | Documentation examples                                      |
| `198.51.100.0/24` | Documentation examples                                      |
| `203.0.113.0/24`  | Documentation examples                                      |

A few important distinctions:

- `127.0.0.1` means the current computer.
- `169.254.x.x` can appear when a device configures a link-local address, for example when DHCP is unavailable.
- `100.64.x.x` is shared address space and is not the same as RFC 1918 private space.
- The three documentation ranges should not be used as real production server addresses.

There are additional special-purpose ranges beyond this list.

## 8. How does a computer know whether an address is local?

This is one of the most important networking concepts.

A computer uses its IP address and subnet mask (or equivalent prefix information) to determine which destinations are on its local IP subnet.

Consider this example:

```
Laptop IP:       192.168.1.37
Subnet mask:     255.255.255.0
Network prefix:  192.168.1.0/24
```

The `/24` means that the first 24 bits identify the network prefix.

In this example, the local subnet is:

```
192.168.1.0/24
```

The usable host addresses commonly used in this ordinary subnet are:

```
192.168.1.1 – 192.168.1.254
```

The address `192.168.1.0` identifies the subnet, and `192.168.1.255` is its broadcast address under this subnet configuration.

Now consider your biometric machine:

```
Biometric IP:    192.168.1.18
Laptop IP:       192.168.1.37
Subnet mask:     255.255.255.0
```

Both addresses belong to the same `/24` subnet. The laptop can normally communicate with the biometric machine directly over the local network, assuming the devices and network permit it.

If the laptop wants to reach `8.8.8.8`, that destination is outside its local subnet. It normally sends the packet to its configured default gateway, which is often the office router.

The subnet mask helps determine the local network. It does not, by itself, establish whether an address is private or public.

## 9. What is a default gateway?

A default gateway is the router or other routing device a computer uses when it needs to send traffic to a destination for which it has no more specific route.

For example:

```
Laptop
IP:       192.168.1.37
Gateway:  192.168.1.1
```

Suppose the laptop accesses an internet server.

The general flow is:

1. The laptop checks whether the destination is on its local subnet.
2. If the destination is outside that subnet, the laptop consults its routing table.
3. If the default route applies, it sends the packet to the gateway.
4. The router forwards it according to its routing table and firewall policy.
5. The packet travels through other networks toward its destination.

The default gateway does not need to be the public IP of the production server. It is the next hop used by the laptop to reach destinations outside its local network.

## 10. What happens when your laptop accesses the internet?

Suppose your laptop has:

```
Laptop private IP: 192.168.1.37
Router LAN IP:     192.168.1.1
```

The router's internet-facing address is determined by your ISP or upstream network. It may be a public IP, or the connection may use another layer of NAT.

A typical flow looks like this:

Laptop

`192.168.1.37`

Office router

May translate private addresses using NAT

ISP and internet routing

Forward traffic toward the destination

Remote server

Receives the request if reachable and permitted

### What is NAT?

NAT stands for Network Address Translation.

A common form, called source NAT, changes the source address of an outgoing packet as it crosses a network boundary.

For example, an outgoing connection might look conceptually like this:

Before NAT:

```
Source:      192.168.1.37:51500
Destination: Remote server:443
```

After NAT at the router:

```
Source:      Router's external address:mapped-port
Destination: Remote server:443
```

The router tracks the connection so that response traffic can be mapped back to the laptop.

The exact external address and mapped port depend on the router and ISP configuration.

NAT is common in IPv4 networks because it lets multiple devices share an external address. However, NAT is not what defines an IP address as private or public, and it is not a replacement for a firewall.

## 11. Why can the biometric machine reach your laptop locally?

Your current setup is approximately:

```
ZKTeco K45 Pro
192.168.1.18
        |
        | Local network
        v
Laptop
192.168.1.37:8000
        |
        v
Django ADMS endpoint
/iclock/cdata
        |
        v
Attendance database
```

The device sends a request to:

```
http://192.168.1.37:8000/iclock/cdata
```

The connection can work because:

- The laptop and device have compatible local network configuration.
- The laptop is reachable from the device.
- Django is listening on the correct interface and port.
- The firewall permits the request.
- The endpoint exists and handles the request correctly.

The laptop's `192.168.1.37` address is private. The device does not need a public IP for this local communication.

## 12. Why does production work without the same LAN?

Now suppose Django is deployed to a cloud server.

The device is in your office, while the server is in a cloud data center.

```
Office network                         Cloud network

Biometric machine                      Production server
192.168.1.18                           Private server IP
      |                                      ^
      v                                      |
Office router                        Nginx → Django
      |                                      ^
      v                                      |
    Internet ────────────────────────────────┘
                 Public endpoint
```

The actual network path may include DNS, a load balancer, a firewall, and a reverse proxy.

The biometric device sends an outgoing request to the configured production endpoint. Its office router forwards the traffic toward the internet, and routing delivers it to the production infrastructure if everything is configured correctly.

The production server may have both a public endpoint and a private IP. For example, a cloud load balancer can accept external requests and forward them to a server whose private IP is not directly accessible from the internet.

The machine and server do not need the same private subnet or the same LAN. They need a valid path for communication.

## 13. Does the server always need a public IP?

No.

There are several possible deployment designs:

| Deployment                                             | How the biometric machine reaches Django                             |
| ------------------------------------------------------ | -------------------------------------------------------------------- |
| Public cloud endpoint                                  | The device connects through the internet                             |
| Public load balancer with a private application server | The load balancer accepts the request and forwards it internally     |
| VPN between the office and cloud                       | The device reaches the server through private routing, if configured |
| Private server without a public route or VPN           | The device normally cannot reach it from an unrelated network        |
| Office server behind a public router                   | It may be reachable through carefully configured port forwarding     |
| Office server behind CGNAT                             | Ordinary inbound port forwarding may not work                        |

A production application does not inherently require a public IP on the machine running Django. It requires a network path from the biometric device to the service.

For example, a private application server can be behind a public load balancer. The load balancer is the internet-facing endpoint, while Django remains on a private network.

## 14. How does DNS fit into this?

DNS stands for Domain Name System.

Suppose the configured server is:

```
attendance.example.com
```

DNS allows the client to resolve that name to the destination configured in DNS records.

A simplified request flow is:

```
Biometric device
        |
        v
Resolve domain (if supported/configured)
        |
        v
Obtain destination address
        |
        v
Connect to the configured service
        |
        v
Send ADMS HTTP request
```

DNS does not translate a private IP into a public IP automatically. It returns the records configured for the domain, and the resulting destination must be reachable.

For your K45 Pro, confirm whether its ADMS settings accept a hostname or require an IP address, and whether it supports the required transport protocol. Some devices accept a server address and port in separate fields.

Do not assume that configuring an HTTPS domain will work unless the device supports the corresponding HTTPS/TLS behavior.

## 15. How can you identify your own private and public IPs?

### On Windows: find your LAN IP

Open Command Prompt and run:

```
ipconfig
```

Look for your active Wi-Fi or Ethernet adapter.

You may see something like:

```
IPv4 Address:       192.168.1.37
Subnet Mask:        255.255.255.0
Default Gateway:    192.168.1.1
```

This tells you your laptop's local IPv4 configuration.

### Find your public-facing IP

You can check your router's WAN status or use a reputable public-IP lookup service from your browser.

However, the address observed by a public-IP lookup service is the address seen by that service. It may belong to an upstream NAT gateway rather than directly to your laptop or router.

If your router's WAN IP is in a private range or in `100.64.0.0/10`, your ISP may be using another layer of NAT. Compare the router's WAN address with the externally observed address to investigate.

Never assume the public IP belongs directly to your laptop simply because you can see it in a browser.

## 16. Common misunderstandings

Misunderstanding 1: Every IPv4 address is private.

Incorrect. IPv4 includes private, globally routable, and special-purpose address ranges.

Misunderstanding 2: Every address starting with `172` is private.

Incorrect. Only `172.16.0.0` through `172.31.255.255` is within the RFC 1918 private range.

Misunderstanding 3: A public IP means everyone can access the server.

Incorrect. Firewalls, routing, listeners, authentication, and application configuration determine access.

Misunderstanding 4: A domain is the same as a public IP.

Incorrect. A domain is a name. DNS resolves it to its configured destination.

Misunderstanding 5: A production server must be on the biometric machine's LAN.

Incorrect. A reachable public endpoint or an appropriately configured private connection can connect separate networks.

Misunderstanding 6: NAT and private IP mean the same thing.

Incorrect. Private IP describes an address range. NAT is a mechanism for translating addresses.

Misunderstanding 7: `0.0.0.0` is the server's public IP.

Incorrect. In a server bind configuration, `0.0.0.0` means listen on all available IPv4 interfaces. It is not the address the biometric device should use.

## 17. Final mental model

Think about the concepts in this order:

1. IPv4 defines a 32-bit addressing system.
2. An IP address identifies a network interface as a source or destination.
3. A subnet and routing table help a computer decide where to send packets.
4. A LAN connects devices within a local network.
5. A private IP is an address from a range reserved for private networks.
6. A public IP is an internet-routable address, provided the necessary routing and configuration exist.
7. A router and NAT commonly connect private devices to an internet-facing network.
8. DNS resolves names to configured destinations.
9. A firewall and listening service determine whether a request can reach the application.
10. Django handles the ADMS request once the request reaches the appropriate application endpoint.

For your biometric integration:

- Local development uses your laptop's reachable private LAN IP.
- Production can use a public endpoint, or a suitable private network connection.
- The biometric machine does not need its own public IP to send attendance data.
- Your laptop does not need to remain on when the production server runs independently.

The core principle is this: IP addressing identifies destinations, routing provides a path, and network/application configuration determines whether communication succeeds.
