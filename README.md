# Scenario 1

## Problem 1: Connection Issue

I found that Docker Compose was not installed on the server, so I started the installation process:

```bash
sudo apt update
```

### APT Update Issue

The `apt update` command failed.

I checked the APT sources configuration and verified that the repository entries were correct. I also changed the configured mirror to `arvan` to rule out a mirror-related issue. However, the problem remained, which indicated that the mirror was not the root cause.

### Network and DNS Investigation

Since I was unable to ping external addresses, I suspected a DNS resolution issue.

I checked:

* `/etc/hosts`

  * The configuration was correct.

* `/etc/resolv.conf`

  * The file was misconfigured and was resolving DNS requests back to the server itself.

During this investigation, I lost my SSH connection and was temporarily disconnected from the server.

---

After reconnecting to the server, I checked the network interfaces:

```bash
ip --br a s
```

Output:

```text
lo               UNKNOWN        127.0.0.1/8 ::1/128
eth0             UP             95.38.188.152/23 fe80::f816:3eff:fe6a:f315/64
docker0          DOWN           172.17.0.1/16 fe80::c70:f7ff:feba:75d7/64
```

The network interfaces appeared to be configured correctly.

I checked whether any DNS resolver service was listening on port 53:

```bash
sudo ss -lntup | grep ':53'
sudo ss -lnup | grep ':53'
```

No service was listening on port 53.

I then checked the status of `systemd-resolved`:

```bash
resolvectl status
```

The command returned:

```text
Failed to get global data: Unit dbus-org.freedesktop.resolve1.service not found.
```

---

### DNS Resolution Fix

To verify whether the issue was only related to DNS, I tested network connectivity directly using an IP address:

```bash
ping -c 3 8.8.8.8
```

The test was successful.

This confirmed that:

* Internet connectivity was working.
* The issue was only with local DNS resolution.

I found that `systemd-resolved` was not running, so I started the service:

```bash
sudo systemctl start systemd-resolved
```

The `/etc/resolv.conf` file was linked to an immutable file, so I removed the immutable attribute:

```bash
sudo chattr -i /run/systemd/resolve/stub-resolv.conf
```

Then I restarted the resolver:

```bash
sudo systemctl restart systemd-resolved
```

After this change, DNS resolution started working correctly.

---

## Problem 2: Docker Compose Installation

I tried to install `docker compose plugin`, but its repo was not defined. So i found that the server was using `docker.io` instead of `Docker CE`.

```sh
apt list --installed | grep docker

# output:

# docker-buildx/noble-updates,now 0.30.1-0ubuntu1~24.04.1 amd64 [installed]
# docker.io/noble-updates,noble-security,now 29.1.3-0ubuntu3~24.04.2 amd64 [installed]
```

I installed Docker Compose V1:

```bash
sudo apt install docker-compose
```

---

## Problem 3: Application Troubleshooting

### Nginx 502 Bad Gateway Error

I tested the application endpoint:

```bash
curl http://localhost/graph
```

The request returned an Nginx `502 Bad Gateway` error.

I confirmed that:

* Nginx was running correctly.
* Nginx was unable to communicate with the backend service.

---

### Backend Container Investigation

I checked the backend container logs:

```bash
docker-compose logs backend
```

I found the following error:

```text
could not translate host name "db" to address: Temporary failure in name resolution
```

The issue was caused by the backend container not being connected to the same Docker network as the database container.

I fixed this by adding the required network configuration to the backend service.

---

### Container Connectivity Testing

After applying the network fix, the issue was still present.

I tested connectivity from inside both containers:

* Executed `curl` from the backend container.
* Executed `curl` from the Nginx container.

Both tests were successful, confirming that container-to-container communication was working.

---

### Nginx Configuration Investigation

I reviewed the Nginx configuration and found that the backend port was incorrectly configured. The correct port was defined in `Dockerfile` of backend as `5000`.

The configuration was forwarding traffic to:

```text
8080
```

while the backend service was listening on:

```text
5000
```

I updated the configuration:

```text
8080 → 5000
```

Then I restarted Nginx:

```bash
docker-compose restart nginx
```

---

### Backend Service Name Correction

The issue still persisted after the port correction.

I checked the Nginx upstream configuration and found that the backend service name was incorrect.

The configuration was using:

```text
backend-api
```

but the actual Docker Compose service name was:

```text
backend
```

I updated the Nginx configuration:

```text
backend-api → backend
```

The upstream configuration was then aligned with the actual Docker Compose service name.

```
```

# Extra problems

## 1. Initial Access Issue

I was only able to SSH into the server when connected through VPN.

---

## 2. Connection Loss to Server
During this investigation, I lost my SSH connection and was temporarily disconnected from the server. So I decided to work on the Scenario 2 using Vagrant. 