---
layout: default
title: It's always DNS
#permalink: /docs/articles/
---

# Docker Networking: Understanding Bridge vs IPvlan Drivers

When working with Docker, choosing the right network driver can significantly impact your application's behavior and performance. In this post, we'll explore two important Docker network drivers: `bridge` and `ipvlan`, understand their differences, and learn when to use each one.

## The Bridge Driver

The bridge driver is Docker's default network driver. It creates a virtual bridge on the host system, and containers connected to it get their own IP addresses in a private subnet.

### How Bridge Networking Works

- Creates a virtual bridge (docker0 by default)
- Containers get private IP addresses (typically in 172.17.0.0/16 range)
- Uses NAT for outbound traffic
- Port mapping required for inbound traffic

### Bridge Network Example

```bash
# Create a custom bridge network
docker network create --driver bridge my-bridge-network

# Run containers on this network
docker run -d --name web --network my-bridge-network nginx
docker run -d --name db --network my-bridge-network postgres
```

## The IPvlan Driver

The ipvlan driver allows containers to have their own IP addresses on the same subnet as the host's physical network, making them appear as separate physical devices on the network.

### How IPvlan Works

- No bridge required
- Containers get IPs from the host's network subnet
- Lower overhead than bridge networking
- Two modes: L2 (layer 2) and L3 (layer 3)

### IPvlan Network Example

```bash
# Create an ipvlan network in L2 mode
docker network create -d ipvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  -o parent=eth0 \
  my-ipvlan-network

# Run a container with a specific IP
docker run -d --name web \
  --network my-ipvlan-network \
  --ip 192.168.1.100 \
  nginx
```

## When to Use Each Driver

### Use Bridge When:

- Running typical containerized applications
- You need container isolation from the host network
- Port mapping is acceptable
- You want simple container-to-container communication
- Running development environments

### Use IPvlan When:

- Containers need to be directly accessible on the physical network
- You're running network appliances or services that need their own IPs
- Avoiding NAT overhead is important
- Integrating with existing network infrastructure
- Running applications that don't work well with NAT

## Practical Example Scenarios

### 1. Web Application (Bridge)
A typical web app with nginx, app server, and database would work perfectly with bridge networking. Internal services communicate freely, and only nginx needs port 80/443 exposed.

### 2. Network Monitoring Tool (IPvlan)
A containerized network monitoring tool that needs to appear as a real device on the network to properly capture traffic would benefit from ipvlan.

### 3. Microservices (Bridge)
Internal microservices that only need to talk to each other work great with bridge networking's isolation.

### 4. VoIP Server (IPvlan)
A containerized VoIP/SIP server often needs a real IP on the network to handle RTP streams properly without NAT complications.

## Case Study: DNS Servers

DNS servers provide an excellent example of when to choose ipvlan over bridge networking.

### Why IPvlan is Better for DNS

**Direct network access benefits:**
- DNS servers need to be directly accessible by all devices on the network
- No NAT translation overhead for UDP/TCP port 53
- Appears as a real device on the network, which is how traditional DNS servers operate

### IPvlan DNS Setup Example

```bash
# Create ipvlan network
docker network create -d ipvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  -o parent=eth0 \
  dns-network

# Run BIND9 DNS server with a specific IP
docker run -d --name dns-server \
  --network dns-network \
  --ip 192.168.1.53 \
  --dns 127.0.0.1 \
  --restart always \
  internetsystemsconsortium/bind9
```

**Benefits of this approach:**
- Clients can use `192.168.1.53` directly as their DNS server
- No port mapping complications
- Better performance (no NAT overhead)
- Works seamlessly with DHCP option 6 (DNS server assignment)

### When Bridge Might Work for DNS

Bridge can work for DNS in specific scenarios, particularly for internal-only DNS:

```bash
# Bridge network for internal services
docker network create --driver bridge internal-dns

# Run DNS server
docker run -d --name dns \
  --network internal-dns \
  -p 53:53/udp \
  -p 53:53/tcp \
  --restart always \
  coredns/coredns
```

**Limitations with bridge for DNS:**
- Requires port mapping (`-p 53:53`)
- Only one container can bind to port 53 on the host
- NAT can cause issues with some DNS features
- Clients need to use the host's IP, not the container's IP

### Real-World Example: Pi-hole with IPvlan

```bash
docker run -d --name pihole \
  --network dns-network \
  --ip 192.168.1.53 \
  -e TZ=America/New_York \
  -e WEBPASSWORD=admin \
  -v pihole-data:/etc/pihole \
  -v dnsmasq-data:/etc/dnsmasq.d \
  --restart unless-stopped \
  pihole/pihole
```

This gives Pi-hole its own IP address (192.168.1.53) that all network devices can use directly as their DNS server, making it behave exactly like a traditional DNS appliance.

## Key Takeaways

1. **Bridge networking** provides isolation and simplicity, perfect for typical containerized applications
2. **IPvlan networking** provides direct network integration, ideal for network services and appliances
3. For DNS servers specifically, ipvlan usually provides a cleaner, more traditional setup
4. Choose based on your specific requirements: isolation vs. direct network access

## Conclusion

Understanding the differences between Docker's network drivers helps you make better architectural decisions. While bridge networking covers most use cases with its simplicity and isolation, ipvlan shines when you need containers to be first-class citizens on your network. 

For network services like DNS servers, load balancers, or monitoring tools, ipvlan often provides a more natural integration with your existing infrastructure. However, for typical web applications and microservices, bridge networking's isolation and port mapping capabilities are usually the better choice.

Remember: the best network driver is the one that matches your specific use case and requirements.