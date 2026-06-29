# personal-cloud-infrastructure
Infrastruttura personal cloud self-hosted basata su Nextcloud e MariaDB, orchestrata tramite Docker Compose e protetta via Tailscale Mesh VPN.


L'accesso remoto è strutturato su un modello *Zero-Trust* che sfrutta una rete mesh VPN point-to-point, eliminando la necessità di esporre porte sull'host o configurare Port Forwarding/DMZ sul router perimetrale.

---

## Stack Tecnologico & Componenti

### 1. Application Layer & Storage: Nextcloud Hub
*   **Container**: `nextcloud:latest` (Image basata su server web Apache e PHP-FPM).
*   **Persistence**: Gestita tramite Docker Volumes indipendenti per isolare la root dell'applicazione (`/var/www/html`) dai dati utente, garantendo persistenza in caso di upgrade dell'immagine.
*   **Networking**: Esposizione della porta HTTP `80` del container mappata sulla porta `8080` dell'host.

### 2. Database Layer: MariaDB Enterprise (10.11 LTS)
*   **Container**: `mariadb:10.11` (Versione Long Term Support per massima stabilità transazionale).
*   **Ottimizzazioni Engine**: Configurato con isolamento transazionale `READ-COMMITTED` e abilitazione del binary logging (`ROW` format) per garantire la coerenza dei metadati e prevenire lock concorrenziali sui record delle tabelle di Nextcloud.

### 3. Network Layer & Security: Tailscale (WireGuard®)
*   **Protocollo**: Mesh VPN basata su protocollo WireGuard®.
*   **Sicurezza**: Crittografia end-to-end (ChaCha20-Poly1305). Consente il superamento di Carrier-Grade NAT (CGNAT) e firewall tramite tecniche di NAT traversal (STUN/DERP).
*   **Routing**: Accesso limitato ai soli client autorizzati nella subnet privata `100.x.x.x`, forzando la risoluzione dell'istanza su socket non cifrato (HTTP) in quanto il livello di trasporto di Tailscale garantisce nativamente la riservatezza dei pacchetti.

### 4. Management Layer: RustDesk
*   **Deployment**: Installato come servizio nativo di Windows (Daemon) per l'intercettazione dei privilege livelli di sistema e l'accesso remoto pre-login (schermata UAC e blocco schermo Windows).

---

## Architettura logica e Flusso Dati

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
## Automazione & Resilienza Sistemistica
Per garantire il funzionamento headless (senza periferiche di input/output collegate) e la tolleranza alle interruzioni di alimentazione, l'host è stato configurato con le seguenti policy:

AC Power Loss Recovery (Hardware): Configurazione del registro ACPI del BIOS/UEFI su Always On o Last State. Al ripristino dell'alimentazione da parte della presa smart, la scheda madre esegue il boot automatico senza input hardware.

Autologin & Init Services (OS): Configurazione del registro di Windows tramite utility Sysinternals Autologin per superare la Login Screen interattiva. All'avvio della sessione utente, il daemon di Docker Desktop inizializza il socket Docker.

Container Restart Policy: Impostazione della direttiva restart: always nel file di orchestrazione Compose, forzando il motore Docker a riavviare i container db e app in caso di crash o al boot del motore stesso.

Graceful Shutdown: Per prevenire la corruzione delle tabelle InnoDB di MariaDB e la perdita di transazioni non persistite (dirty pages in memoria), lo spegnimento viene gestito inviando un segnale di SIGTERM al sistema operativo tramite RustDesk prima dell'interruzione hardware della corrente.


## Comandi di Manutenzione Rapida

Configurazione del dominio di origine trusted per abilitare l'handshake dell'app mobile:

docker compose exec -u www-data app php occ config:system:set trusted_domains 2 --value=100.x.x.x:8080
Arresto pulito dello stack:

docker compose down
