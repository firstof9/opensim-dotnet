# opensim-dotnet

[![Build and Push Docker Image](https://github.com/firstof9/opensim-dotnet/actions/workflows/docker-build-push.yml/badge.svg)](https://github.com/firstof9/opensim-dotnet/actions/workflows/docker-build-push.yml)

Docker container configurations for running [OpenSim](https://github.com/opensim/opensim) (Simulator and Robust grid services) on the modern **.NET 8.0 runtime**.

Images are automatically built and published to the GitHub Container Registry (GHCR):
- **Robust Grid Services**: `ghcr.io/firstof9/robust:latest`
- **OpenSim Simulator**: `ghcr.io/firstof9/sim:latest`

---

## Directory Structure

```text
opensim-dotnet/
├── .github/workflows/
│   └── docker-build-push.yml  # GitHub Actions automated build/push pipeline
├── robust/
│   ├── Dockerfile             # Docker recipe for Robust Grid Services
│   └── Robust.exe.config      # Default log4net config for Robust
├── sims/
│   ├── Dockerfile             # Docker recipe for the Simulator
│   └── OpenSim.exe.config     # Default log4net config for OpenSim
├── docker-compose.yml         # Docker Compose orchestration
├── .gitignore
├── LICENSE
└── README.md
```

---

## Configuration & Volumes

To run these containers, prepare a directory structure on the host for mounting configurations and persistent data.

### 1. Configuration Files
Create a local `./config/` directory and populate it with the following configuration files tailored to your grid layout:
- **`config/Robust.ini`**: Configuration for Robust grid services.
- **`config/OpenSim.ini`**: Main simulator configurations.
- **`config/Regions.ini`**: Definitions of simulator regions.
- **`config/GridCommon.ini`**: Shared configuration (database connection, service URLs) included by both OpenSim and Robust.

### 2. Persistent Storage Directories
- **`./fsassets/`**: Local directory mounted to store Robust binary asset data.
- **`./persistence/`**: Local directory for simulator data (SQLite databases or local storage).
- **`./imports/`** *(Optional)*: Commented out in `docker-compose.yml`; uncomment to import/export OAR/IAR files from/to the host.

---

## Orchestration (Docker Compose)

Spin up both services with the provided `docker-compose.yml` file:

```yaml
services:
  robust:
    image: ghcr.io/firstof9/robust:latest
    container_name: robust-server
    restart: unless-stopped
    ports:
      - "8003:8003/tcp"
    environment:
      - TZ=UTC # Replace with your local timezone, e.g., America/Phoenix
      - DOTNET_GCConserveMemory=4
      - DOTNET_GCHighMemPercent=4B
    volumes:
      - ./config/Robust.ini:/home/opensim/opensim/bin/Robust.ini:ro
      - ./config/GridCommon.ini:/home/opensim/opensim/bin/config-include/GridCommon.ini:ro
      - ./fsassets:/home/opensim/opensim/bin/fsassets

  sim:
    image: ghcr.io/firstof9/sim:latest
    container_name: sim-server
    restart: unless-stopped
    ports:
      - "9000:9000/tcp"
      - "9000-9005:9000-9005/udp"
    environment:
      - TZ=UTC # Replace with your local timezone, e.g., America/Phoenix
      - DOTNET_GCConserveMemory=4
      - DOTNET_GCHighMemPercent=4B
    volumes:
      - ./config/Regions.ini:/home/opensim/opensim/bin/Regions/Regions.ini:ro
      - ./config/OpenSim.ini:/home/opensim/opensim/bin/OpenSim.ini:ro
      - ./config/GridCommon.ini:/home/opensim/opensim/bin/config-include/GridCommon.ini:ro
      - ./persistence:/home/opensim/opensim/bin/persistence
      # - ./imports:/home/opensim/opensim/bin/imports
```

Run the stack in the background:
```bash
docker compose up -d
```

### Environment Parameters (.NET Tuning)
The compose file includes .NET Garbage Collector settings configured to optimize resource utilization:
- `DOTNET_GCConserveMemory=4`: Aggressively conserves memory usage (GC aggressiveness level 4).
- `DOTNET_GCHighMemPercent=4B`: Restricts the GC high memory threshold to 75% (`4B` in hexadecimal) of host memory limits to prevent container OOM termination.
- `TZ`: Defines the container timezone, aligning timestamps inside logs.

---

## CI/CD Pipeline

The GitHub Actions workflow automatically:
1. Rebuilds and pushes the images on every push to the `main` branch.
2. Performs a daily cron build at **04:00 UTC** to pull the latest upstream OpenSim release from the repository releases redirect link, ensuring you are always up to date.

---

## License

Distributed under the **MIT License**. See the `LICENSE` file for details.
