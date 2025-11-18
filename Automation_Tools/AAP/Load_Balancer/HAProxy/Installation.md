```
---
title: The HAProxy Installation
author: <karshi.hasanov@utoronto.ca>
date: October 25, 2025
decription: I'll guide you through setting up HAProxy (ver. 2.8.14) to load balance your two AAP controllers. 
            This example assumes you're installing on a RHEL 9.6 server.
---
```
***


# Step 1: Install HAProxy
On your HAProxy server (a separate VM/server):
```bash

# Install HAProxy
sudo dnf install haproxy -y

# Enable and start the service
sudo systemctl enable --now haproxy

# Check status
sudo systemctl status haproxy
```

# Step 2: Configure HAProxy
Edit the main config file:
```bash
# Backup the original config:
sudo mv /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg_orig
# Create the new one:
sudo vim /etc/haproxy/haproxy.cfg
```
**Replace the entire content** with this configuration tuned for AAP:
```configuration
global
    log         127.0.0.1 local2 info
    chroot      /var/lib/haproxy
    #pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon

    # SSL/TLS settings (if terminating SSL at HAProxy)
    ssl-default-bind-options ssl-min-ver TLSv1.2
    ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384
    ssl-default-server-options ssl-min-ver TLSv1.2

defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          5m
    timeout server          5m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000

# Frontend for external traffic
frontend main_frontend
    bind *:80
    bind *:443 ssl crt /etc/haproxy/certs/main.karshi.ca.pem  # We'll create this
    default_backend AAP_Controllers
    timeout client 30s

    # Set headers for main.karshi.ca
    http-request add-header X-Forwarded-Proto https if { ssl_fc }
    http-request set-header X-Forwarded-Port %[dst_port]

    # Redirect HTTP to HTTPS
    redirect scheme https if !{ ssl_fc }

    # AAP uses WebSockets for some features
    acl is_websocket hdr(Upgrade) -i websocket
    use_backend AAP_Controllers if is_websocket

    # Default backend for all other traffic
    default_backend AAP_Controllers

# Backend pool of AAP controllers
backend AAP_Controllers
    mode http
    balance roundrobin  # Options: source, leastconn

    # The "http-check":
    option httpchk GET /api/v2/ping/
    http-check connect port 443
    http-check send meth GET uri /api/v2/ping/
    http-check expect status 200

    # Optional: Enable session persistence (sticky sessions)
    # AAP controllers share session state, but this adds extra safety
    cookie AAP_SERVER insert indirect nocache

    # Backend servers:

    # The "check-ssl" flag tells the health check to inherit the server's SSL settings,
    # including "verify none"

    # Controller
    server controller controller.karshi.ca:443 cookie ctrl1 ssl verify none check-ssl

    # Controller2
    server controller2 controller2.karshi.ca:443 cookie ctrl2 ssl verify none check-ssl


    # Websocket persistence
    timeout tunnel 1h

# Statistics page (optional, secure this in production!)
listen stats
    bind *:8404
    mode http
    stats enable
    stats uri /haproxy_stats
    stats refresh 30s
    stats auth admin: hassan3
```

# Step 3: Configure SSL/TLS Certificate

### Option A: Terminate SSL at HAProxy (Recommended)
#### Copy your aap.karshi.ca certificate and key to the HAProxy server:
```bash
# Create cert directory
sudo mkdir -p /etc/haproxy/certs

# Combine cert + key into one file (HAProxy requirement)
# You need to be superuser in order the ">" work:
sudo -i
cat /etc/certs/main_karshi_ca.crt /etc/certs/key_main_karshi_ca.pem > /etc/haproxy/certs/main.karshi.ca.pem
# Logout from root
exit
# Secure permissions
sudo chmod 600 /etc/haproxy/certs/main.karshi.ca.pem
sudo chown haproxy:haproxy /etc/haproxy/certs/main.karshi.ca.pem

```
#### OR create a self-signed cert for testing:
sudo mkdir -p /etc/haproxy/certs
sudo openssl req -x509 -newkey rsa:4096 -keyout /etc/haproxy/certs/key.pem -out /etc/haproxy/certs/cert.pem -days 365 -nodes -subj "/CN=aap.karshi.ca"
sudo cat /etc/haproxy/certs/cert.pem /etc/haproxy/certs/key.pem > /etc/haproxy/certs/aap.karshi.ca.pem
sudo chmod 600 /etc/haproxy/certs/aap.karshi.ca.pem

## Option B: Pass-through SSL
If you prefer SSL pass-through (controllers handle SSL), use a simpler TCP-based config. Let me know if you need this.

# Step 4: Update AAP Controller Settings
On both controllers, configure them to trust HAProxy's IP:
```bash
# On controller and controller2
sudo vim /etc/tower/conf.d/custom.py
```
Add:
```python
# Trust HAProxy's IP address
REMOTE_HOST_HEADERS = ['X-Forwarded-For', 'REMOTE_ADDR']
SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')
ALLOWED_HOSTS = ['*']  # Or use: ['aap.karshi.ca', 'controller.karshi.ca', 'controller2.karshi.ca']
```
Restart AAP services on both controllers:
```bash
sudo automation-controller-service restart
```

# Step 5: Configure DNS
In your DNS server (or /etc/hosts for testing):
```
# Point the external name to HAProxy's IP
aap.karshi.ca    <IP_of_your_haproxy_server>
```
# Configure SELinux & Firewall (RHEL)
## Firewall Rules
```bash
# Allow ports in firewall
sudo firewall-cmd --permanent --add-port=80/tcp
sudo firewall-cmd --permanent --add-port=443/tcp
sudo firewall-cmd --permanent --add-port=9000/tcp
sudo firewall-cmd --reload
```
## SELinux policy
```bash
sudo setsebool -P haproxy_connect_any 1
```


# Step 7: Restart HAProxy and Verify
```bash
# Check config syntax
sudo haproxy -c -f /etc/haproxy/haproxy.cfg

# Restart if syntax is OK
sudo systemctl restart haproxy

# Watch logs for issues
sudo tail -f /var/log/haproxy.log
```
# Step 8: Test the Setup
From a client machine:
```bash
# Test HTTPS access
curl -k https://main.karshi.ca/api/v2/ping/

# Check HAProxy stats page
# Open browser: http://<haproxy_ip>:9000/haproxy_stats
# Login: admin / your_secure_password_here
```
**Important Security Notes**
1. **Firewall:** Open only ports 80/443 on HAProxy server
2. **Stats Page:** Change the password and consider restricting access to specific IPs
3. **Certificates:** Use proper CA-signed certs in production (not self-signed)
4. **Controller Access:** Consider blocking direct access to controller IPs and forcing all traffic through HAProxy