# personal-cloud-infrastructure
Self-hosted personal cloud infrastructure based on Nextcloud and MariaDB, orchestrated via Docker Compose and secured through a Tailscale Mesh VPN.

Remote access is structured on a *Zero-Trust* model that leverages a point-to-point mesh VPN network, eliminating the need to expose ports on the host or configure Port Forwarding/DMZ on the perimeter router.

---

## Technology Stack & Components

### 1. Application Layer & Storage: Nextcloud Hub
* **Container**: `nextcloud:latest` (Image based on Apache web server and PHP-FPM).
* **Persistence**: Managed through independent Docker Volumes to isolate the application root (`/var/www/html`) from user data, ensuring persistence during image upgrades.
* **Networking**: Exposure of the container's HTTP port `80` mapped to port `8080` of the host.

### 2. Database Layer: MariaDB Enterprise (10.11 LTS)
* **Container**: `mariadb:10.11` (Long Term Support version for maximum transactional stability).
* **Engine Optimizations**: Configured with `READ-COMMITTED` transactional isolation and enabling binary logging (`ROW` format) to ensure metadata consistency and prevent concurrent locks on Nextcloud table records.

### 3. Network Layer & Security: Tailscale (WireGuard®)
* **Protocol**: Mesh VPN based on the WireGuard® protocol.
* **Security**: End-to-end encryption (ChaCha20-Poly1305). Allows bypassing Carrier-Grade NAT (CGNAT) and firewalls via NAT traversal techniques (STUN/DERP).
* **Routing**: Access limited exclusively to authorized clients within the private `100.x.x.x` subnet, forcing the resolution of the instance over an unencrypted socket (HTTP) since Tailscale's transport layer natively guarantees packet confidentiality.

### 4. Management Layer: RustDesk
* **Deployment**: Installed as a native Windows service (Daemon) for intercepting system-level privileges and pre-login remote access (UAC screen and Windows lock screen).

---

## Logical Architecture and Data Flow

```text
[Smartphone Client (Nextcloud App)]
                │
          (Tailscale IPv4)
                │
                ▼ (Port 8080)
┌────────────── Windows Host ────────────────────────┐
│                                                    │
│   ┌─── Docker Engine (WSL2 Backend) ───────────┐   │
│   │                                            │   │
│   │  [Nextcloud Container] ──(Internal Net)──┐ │   │
│   │         │                                │ │   │
│   │         ▼ (Volume)                       ▼ │   │
│   │   ┌───────────┐                    [MariaDB]   │
│   │   │ User Data │                        │       │
│   │   └───────────┘                        ▼       │
│   │                                  ┌───────────┐ │
│   │                                  │ DB Vol    │ │
│   │                                  └───────────┘ │
│   └────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────┘


## Automation & Systemic Resilience
To ensure headless operation (without input/output peripherals connected) and tolerance to power interruptions, the host has been configured wi th the following policies:

* AC Power Loss Recovery (Hardware): Configuration of the BIOS/UEFI ACPI register to Always On or Last State. Upon power restoration by the smart plug, the motherboard boots automatically without hardware input.

* Autologin & Init Services (OS): Configuration of the Windows registry using the Sysinternals Autologin utility to bypass the interactive Login Screen. When the user session starts, the Docker Desktop daemon         initializes the Docker socket.

* Container Restart Policy: Setting the restart: always directive in the Compose orchestration file, forcing the Docker engine to restart the db and app containers in case of a crash or upon the engine's boot.

* Graceful Shutdown: To prevent data corruption of MariaDB InnoDB tables and the loss of unpersisted transactions (dirty pages in memory), shutdown is managed by sending a SIGTERM signal to the operating system via   RustDesk before the smart plug cuts the hardware power.


## Quick Maintenance Commands

Configuration of the trusted origin domain to enable the mobile app handshake:
docker compose exec -u www-data app php occ config:system:set trusted_domains 2 --value=100.x.x.x:8080

Clean stop of the stack:
docker compose down
