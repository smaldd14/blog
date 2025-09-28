# Deploying Temporal with Podman Quadlets

I spent a weekend diving into Podman Quadlets, exploring how to deploy Temporal on a Linux machine using this relatively new container management approach. While Temporal's website has excellent [docs for self-hosting](https://docs.temporal.io/self-hosted-guide/deployment), I found a gap when it comes to [Podman Quadlets](https://www.redhat.com/en/blog/quadlet-podman). Since I've been working more with edge deployments lately, I thought this would be a perfect opportunity to bridge that gap.

## What You'll Learn

In this tutorial, we'll cover:
- Deploying multi-container applications with Podman Quadlets
- Creating systemd services from containers for production-like management
- Configuring an nginx reverse proxy with SSL
- Setting up Temporal for local development with PostgreSQL (Although better use just use [temporal cli](https://github.com/temporalio/cli) for local dev)

By the end, you'll have a fully functional Temporal deployment that's managed by systemd, making it ideal for production environments, especially those with high security requirements.

## Source Code

All the code for this tutorial is available on GitHub: **[temporal-quadlets](https://github.com/smaldd14/temporal-quadlets)**

You can clone the repository and follow along:
```bash
git clone https://github.com/smaldd14/temporal-quadlets.git
cd temporal-quadlets
```

## Prerequisites

- A RHEL 10 machine (or similar with Podman 4.4+)
- Basic familiarity with containers and systemd
- Linux/bash command line skills
- SSH access to your target machine

I initially wanted to deploy this on my Raspberry Pi, but discovered it was running Debian Bookworm with only Podman 4.3.1. Since Quadlets require Podman 4.4+, I set up a RHEL 10 VM to ensure we're working with the latest features.

## Technology Overview

### Temporal: Workflow Orchestration Made Simple

Temporal is a workflow orchestration platform that makes building reliable, scalable applications significantly easier. Instead of dealing with the complexities of distributed systems, timeouts, retries, and state management yourself, Temporal handles these concerns for you.

For our deployment, we're setting up three core Temporal components:
- **Temporal Server**: The core workflow engine that orchestrates your workflows
- **Temporal UI**: A web interface for monitoring workflows, debugging, and operational visibility
- **PostgreSQL**: The database backend for persistence and state management

### Podman Quadlets: Containers Meet systemd

Podman Quadlets represent a fundamental shift in how we deploy containers in production. Rather than relying on Docker's daemon architecture, Quadlets integrate directly with systemd, offering several compelling advantages:

**What are Quadlets?** They're configuration files that describe containers, networks, and volumes in a systemd-native way. Podman reads these `.container`, `.network`, and `.volume` files and automatically generates corresponding systemd services.

**Why they matter for production:**
- **No daemon dependency**: Unlike Docker, there's no central daemon that could become a single point of failure
- **Native systemd integration**: Automatic restarts, dependency management, and logging work out of the box (think journalctl)
- **Security by default**: Each container runs in its own systemd slice with proper isolation

### Edge and High-Security Benefits

This approach is particularly valuable for edge systems and high-security environments:

- **Reduced attack surface**: No container daemon running as root
- **Better compliance**: systemd's security features and audit trails
- **Simplified operations**: Standard systemd tools for monitoring and management
- **Rootless capabilities**: Can run entirely without root privileges when needed (I initially tried this, but ran into some quirks that I will explain later)

## Architecture Overview

Our deployment consists of four main components connected through a custom network:

*[Architecture diagram will be added here]*

- **PostgreSQL**: Provides persistent storage for Temporal's state
- **Temporal Server**: The core engine that executes workflows
- **Temporal UI**: Web interface accessible on port 8080
- **Nginx**: Reverse proxy providing external access and SSL termination

All components communicate over a dedicated `temporal-network`, allowing them to reach each other by container name while remaining isolated from other services.

## Implementation

### Step 1: Project Structure

Our Podman Quadlets are organized in a clear directory structure:

```
services/temporal/
├── temporal-postgres.container      # PostgreSQL database
├── temporal-server.container        # Temporal workflow engine
├── temporal-ui.container           # Web UI
├── temporal-nginx.container        # Reverse proxy
├── temporal.network               # Custom network
├── temporal-config.volume         # Configuration volume
├── nginx-config.volume           # Nginx config volume
├── nginx-ssl.volume             # SSL certificates volume
└── config/
    ├── temporal/
    │   └── development-sql.yaml  # Temporal dynamic config
    └── nginx/
        ├── conf.d/
        │   └── temporal.conf     # Nginx site config
        └── ssl/
            ├── temporal.crt      # SSL certificate
            └── temporal.key      # SSL private key
```

### Step 2: Database Foundation

PostgreSQL serves as our persistence layer. Here's the quadlet configuration:

```ini
[Unit]
Description=PostgreSQL Database for Temporal

[Container]
Image=docker.io/library/postgres:16
ContainerName=temporal-postgresql

# PostgreSQL configuration
Environment=POSTGRES_PASSWORD=temporal
Environment=POSTGRES_USER=temporal
Environment=POSTGRES_DB=temporal

# Persistent data storage using named volume
Volume=postgres-data:/var/lib/postgresql/data

Network=temporal-network
# Keep port published for external access if needed
PublishPort=5432:5432

# Health check
HealthCmd=pg_isready -U temporal
HealthInterval=10s
HealthTimeout=5s
HealthRetries=5

[Service]
Restart=always
TimeoutStartSec=60

[Install]
WantedBy=multi-user.target
```

> 📁 **Complete file**: [`services/temporal/temporal-postgres.container`](https://github.com/smaldd14/temporal-quadlets/blob/main/services/temporal/temporal-postgres.container)

Key points:
- Uses a named volume for data persistence across container restarts
- Connects to our custom network for service discovery
- Includes health checks to ensure PostgreSQL is ready before dependent services start

### Step 3: Temporal Server Configuration

The Temporal Server is the heart of our deployment. It connects to PostgreSQL and provides the workflow execution engine:

```ini
[Unit]
Description=Temporal Server
Requires=temporal-postgres.service
After=temporal-postgres.service

[Container]
Image=docker.io/temporalio/auto-setup:1.28.1
ContainerName=temporal-server
Network=temporal-network
PublishPort=7233:7233

# PostgreSQL configuration
Environment=DB=postgres12
Environment=DB_PORT=5432
Environment=POSTGRES_USER=temporal
Environment=POSTGRES_PWD=temporal
Environment=POSTGRES_SEEDS=temporal-postgresql
Environment=POSTGRES_DB=temporal

# Volume mounts for dynamic configuration
Volume=temporal-config-volume:/etc/temporal/config/dynamicconfig

# Health check
HealthCmd=temporal --address localhost:7233 operator cluster health
HealthInterval=30s
HealthTimeout=10s
HealthRetries=3

[Service]
Restart=always
TimeoutStartSec=300

[Install]
WantedBy=multi-user.target
```

> 📁 **Complete file**: [`services/temporal/temporal-server.container`](https://github.com/smaldd14/temporal-quadlets/blob/main/services/temporal/temporal-server.container)

The `auto-setup` image automatically creates the necessary database schemas on startup, simplifying deployment. The server connects to PostgreSQL using the container name `temporal-postgresql` thanks to our custom network.

In a production setup, especially with security in mind, you would probably want to use the dedicated [temporal-server image](https://github.com/temporalio/docker-compose?tab=readme-ov-file#using-temporal-docker-images-in-production), and most likely want to build off a trusted base image rather than using `auto-setup` in production.

### Step 4: UI Setup

The Temporal UI provides essential operational visibility:

```ini
[Unit]
Description=Temporal UI
Requires=temporal-server.service
After=temporal-server.service

[Container]
Image=docker.io/temporalio/ui:2.39.0
ContainerName=temporal-ui
Network=temporal-network

# Connect to temporal server
Environment=TEMPORAL_ADDRESS=temporal-server:7233
Environment=TEMPORAL_CORS_ORIGINS=http://localhost:8080

# UI configuration
Environment=TEMPORAL_UI_PORT=8080

[Service]
Restart=always
TimeoutStartSec=60

[Install]
WantedBy=multi-user.target
```

> 📁 **Complete file**: [`services/temporal/temporal-ui.container`](https://github.com/smaldd14/temporal-quadlets/blob/main/services/temporal/temporal-ui.container)

The UI connects directly to the Temporal Server and serves the web interface on port 8080. In our architecture, nginx will proxy requests to this internal port.

### Step 5: Nginx Reverse Proxy

Nginx provides external access with SSL termination. We use separate volumes for configuration and SSL certificates, following security best practices:

```ini
[Unit]
Description=Nginx Reverse Proxy for Temporal
Requires=temporal-ui.service
After=temporal-ui.service

[Container]
Image=docker.io/library/nginx:1.27-alpine
ContainerName=temporal-nginx
Network=temporal-network

# Volume mounts for configuration and certificates
Volume=systemd-nginx-config:/etc/nginx/conf.d:ro,Z
Volume=systemd-nginx-ssl:/etc/nginx/ssl:ro,Z

# Expose HTTP and HTTPS ports
PublishPort=80:80
PublishPort=443:443

# Health check
HealthCmd=curl -f http://temporal-server:7233/health || exit 1
HealthInterval=30s
HealthTimeout=5s
HealthRetries=3

[Service]
Restart=always
TimeoutStartSec=60

[Install]
WantedBy=multi-user.target
```

> 📁 **Complete file**: [`services/temporal/temporal-nginx.container`](https://github.com/smaldd14/temporal-quadlets/blob/main/services/temporal/temporal-nginx.container)

The nginx configuration proxies requests to the Temporal UI while providing SSL encryption for external access.

### Step 6: Network and Volume Management

Our custom network enables service discovery:

```ini
[Network]
NetworkName=temporal-network
Driver=bridge
Subnet=172.20.0.0/16
Gateway=172.20.0.1

[Install]
WantedBy=multi-user.target
```

> 📁 **Complete file**: [`services/temporal/temporal.network`](https://github.com/smaldd14/temporal-quadlets/blob/main/services/temporal/temporal.network)

Volume quadlets manage our configuration and SSL certificates. Here are the key volume definitions:

**Nginx Configuration Volume:**
```ini
[Unit]
Description=Nginx Configuration and Certificates Volume

[Volume]
Driver=local
Device=/opt/temporal/config/nginx/conf.d
Type=bind
Options=bind

[Install]
WantedBy=multi-user.target
```

**SSL Certificates Volume:**
```ini
[Unit]
Description=Nginx SSL Certificates Volume

[Volume]
Driver=local
Device=/opt/temporal/config/nginx/ssl
Type=bind
Options=bind

[Install]
WantedBy=multi-user.target
```

> 📁 **Complete files**: [`nginx-config.volume`](https://github.com/smaldd14/temporal-quadlets/blob/main/services/temporal/nginx-config.volume) and [`nginx-ssl.volume`](https://github.com/smaldd14/temporal-quadlets/blob/main/services/temporal/nginx-ssl.volume)

This separation follows security best practices:
- Configuration files are stored in `/opt/temporal/config/nginx/conf.d`
- SSL certificates are isolated in `/opt/temporal/config/nginx/ssl`

## Deployment Steps

### 1. Prepare Your Environment

SSH into your RHEL machine and switch to root for system-wide deployment:

```bash
ssh your-vm
sudo su -
```

### 2. Create Directory Structure

Set up the configuration directories where our volumes will mount:

```bash
mkdir -p /opt/temporal/config/{temporal,nginx/{conf.d,ssl}}
```

### 3. Copy Files

Transfer your quadlet files and configurations:

```bash
# Copy quadlet definitions
scp services/temporal/*.container services/temporal/*.volume services/temporal/*.network root@{VM_IP}:/etc/containers/systemd/

# Copy configuration files
scp services/temporal/config/temporal/development-sql.yaml vbvm:/opt/temporal/config/temporal/
scp services/temporal/config/nginx/conf.d/temporal.conf vbvm:/opt/temporal/config/nginx/conf.d/
scp services/temporal/config/nginx/ssl/* vbvm:/opt/temporal/config/nginx/ssl/

# Set proper ownership
chown -R root:root /opt/temporal/config/
```

### 4. Deploy Services

Reload systemd to recognize the new quadlets, then start services in dependency order:

```bash
# Make systemd aware of our new services
systemctl daemon-reload

# Verify our services are loaded
systemctl list-unit-files | grep temporal

# Start services in dependency order
systemctl start temporal.network
systemctl start temporal-postgres.service

# Wait for PostgreSQL to be ready
systemctl start temporal-server.service
# tail logs
journalctl -f -u temporal-server.service
# Wait for server initialization
systemctl start temporal-ui.service
systemctl start temporal-nginx.service

```

Because the systemd files are generated by Podman by being in `/etc/containers/systemd`, there is no need to `systemctl enable` them.

### 5. Verify Deployment

Check that all services are running correctly:

```bash
# Check service status
systemctl status temporal*.service

# Monitor Temporal Server logs to ensure it connected to PostgreSQL
journalctl -f -u temporal-server.service

# Test nginx health check
curl http://localhost/health
```

## Accessing Your Temporal Deployment

Once deployed, you can access Temporal through multiple endpoints:

- **Local access**: `http://localhost:8080` (direct to UI)
- **Via nginx**: `http://localhost:80` or `https://localhost:443`, or just `localhost` in your browser in the VM

### VirtualBox Port Forwarding (Optional)

If you're running this in a VM and want to access it from your host machine, set up port forwarding in VirtualBox:

1. Go to VM Settings → Network → Advanced → Port Forwarding
2. Add these rules:
   - **SSH**: Host Port 2222 → Guest Port 22
   - **Temporal HTTP**: Host Port 8080 → Guest Port 80
   - **Temporal HTTPS**: Host Port 8443 → Guest Port 443
   - **Temporal UI Direct**: Host Port 9080 → Guest Port 8080

Then access from your host machine:
- HTTP via nginx: `http://localhost:8080`
- HTTPS via nginx: `https://localhost:8443` (will show certificate warning)
- Direct UI: `http://localhost:9080`

## Troubleshooting

### Common Issues

**Temporal Server waiting for PostgreSQL**: Check that PostgreSQL is running and accessible:
```bash
systemctl status temporal-postgres.service
podman exec temporal-postgresql pg_isready -U temporal
```

**Permission errors with nginx volumes**: Ensure proper ownership of configuration directories:
```bash
chown -R root:root /opt/temporal/config/
```

**Services not starting**: Check systemd service status and logs:
```bash
systemctl status service-name.service
journalctl -u service-name.service -n 50
```

### Useful Debugging Commands

```bash
# Watch all Temporal services in real-time
journalctl -f -u temporal-postgres.service -u temporal-server.service -u temporal-ui.service -u temporal-nginx.service

# Check container logs directly
podman logs temporal-server
podman logs temporal-ui

# Verify network connectivity
podman network inspect temporal-network
```

## Production Considerations

This setup uses self-signed certificates for local development. For production deployment with real domains and Let's Encrypt certificates, see the [Temporal nginx guide](https://learn.temporal.io/tutorials/infrastructure/nginx-sqlite-binary/#deploying-an-nginx-reverse-proxy-with-an-ssl-certificate).

Additional production considerations:
- Implement proper backup strategies for PostgreSQL data
- Configure log rotation and monitoring
- Set up proper firewall rules
- Consider using separate machines for different components
- Implement proper secrets management instead of hardcoded passwords

## Conclusion

Podman Quadlets offers a great alternative to traditional container orchestration, especially for edge deployments and high-security environments. By integrating directly with systemd, we get production-grade container management without the complexity and security concerns of daemon-based solutions, like docker/kubernetes.

This Temporal deployment demonstrates how Quadlets can manage complex multi-container applications with proper networking, volume management, and service dependencies. The result is a robust, systemd-native deployment that's perfect for production environments where reliability and security are high priority.

The next step would be to start building and deploying Temporal workflows to take advantage of this solid foundation. The combination of Temporal's workflow capabilities with Podman's security-focused approach creates an excellent platform for building reliable distributed applications. Stay tuned for more tutorials on using Temporal effectively!