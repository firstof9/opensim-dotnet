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

## Getting Started: Host Setup

Before starting the containers, pre-create the host directories and configuration file placeholders. If you do not create them first, Docker Compose will create them automatically as directories owned by `root`, resulting in permission errors.

Run the following commands in the root of your project directory:

```bash
# Create directories for configuration, persistent simulator data, and assets
mkdir -p config persistence fsassets

# Create empty configuration placeholders (which you will populate)
touch config/Robust.ini config/OpenSim.ini config/Regions.ini config/GridCommon.ini
```

---

## Configuration & Volumes

Tailor the following mounted files and directories to your grid layout:

### 1. Configuration Files (in `./config/`)
- **`Robust.ini`**: Main configuration for Robust grid services.
- **`OpenSim.ini`**: Main simulator configuration.
- **`Regions.ini`**: Definitions of simulator regions.
- **`GridCommon.ini`**: Shared configuration (database connection strings, service URLs) included by both OpenSim and Robust.

### 2. Persistent Storage Directories
- **`./fsassets/`**: Mapped directory where Robust stores binary asset data.
- **`./persistence/`**: Mapped directory for simulator state and local databases (if using SQLite).
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
    # Run as a non-root user (match your host user's UID and GID to avoid permission issues)
    user: "1000:1000"
    ports:
      - "8003:8003/tcp"
    environment:
      - TZ=UTC # Replace with your local timezone, e.g., America/Phoenix
      # DOTNET_GCConserveMemory specifies the Garbage Collector memory conservation level (1-9, where 9 is most aggressive).
      - DOTNET_GCConserveMemory=4
      # DOTNET_GCHighMemPercent specifies the GC memory limit high threshold percent in hexadecimal (0x4B = 75 in decimal).
      - DOTNET_GCHighMemPercent=4B
    volumes:
      - ./config/Robust.ini:/home/opensim/opensim/bin/Robust.ini:ro
      - ./config/GridCommon.ini:/home/opensim/opensim/bin/config-include/GridCommon.ini:ro
      - ./fsassets:/home/opensim/opensim/bin/fsassets

  sim:
    image: ghcr.io/firstof9/sim:latest
    container_name: sim-server
    restart: unless-stopped
    # Run as a non-root user (match your host user's UID and GID to avoid permission issues)
    user: "1000:1000"
    ports:
      - "9000:9000/tcp"
      - "9000-9005:9000-9005/udp"
    environment:
      - TZ=UTC # Replace with your local timezone, e.g., America/Phoenix
      # DOTNET_GCConserveMemory specifies the Garbage Collector memory conservation level (1-9, where 9 is most aggressive).
      - DOTNET_GCConserveMemory=4
      # DOTNET_GCHighMemPercent specifies the GC memory limit high threshold percent in hexadecimal (0x4B = 75 in decimal).
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

## Interactive Console Administration

Since both services run in background detached mode, direct access to the command-line interfaces requires attaching to their interactive Docker TTYs.

### Attaching to the Console
To attach to the simulator console or Robust grid services console:
```bash
docker attach sim-server
# OR
docker attach robust-server
```

### Safely Detaching
> [!WARNING]
> Do NOT use `Ctrl + C` to exit the attached console; doing so will send a kill signal and terminate the entire OpenSim/Robust server.
>
> To safely detach and leave the server running in the background, press:
> **`Ctrl + P`**, followed by **`Ctrl + Q`**.

---

## Advanced Configuration: Remote Console

For a more robust and secure administration method, you can set up OpenSim's built-in remote console (Telnet-based server) which allows you to log in without attaching to the container's standard input.

1. In your `OpenSim.ini` (or `Robust.ini`), configure the `[Network]` section:
   ```ini
   [Network]
     ConsoleUser = "admin"
     ConsolePass = "securepass123"
     ConsolePort = 9000
   ```
2. Set `Console = "Connector"` in the `[Startup]` section of the config.
3. Access the console via any standard Telnet or SSH client pointing to the respective container port.

---

## Database Setup (SQLite vs. MariaDB/MySQL)

By default, standalone OpenSim instances utilize SQLite, storing database files inside the `./persistence/` directory.

- **For production, multi-region, or grid deployments**, it is highly recommended to migrate/configure OpenSim and Robust to connect to an external **MariaDB/MySQL** database.
- You can configure the database connection string in `GridCommon.ini` under the database settings:
  ```ini
  [DatabaseService]
      StorageProvider = "OpenSim.Data.MySQL.dll"
      ConnectionString = "Data Source=db_host;Database=opensim;User ID=opensim;Password=secure_password;"
  ```
- If desired, you can add a database service (such as `mariadb:10.11`) directly to your `docker-compose.yml` to keep the database container managed within the same stack.

---

## CI/CD Pipeline

The GitHub Actions workflow automatically:
1. Rebuilds and pushes the images on every push to the `main` branch.
2. Performs a daily cron build at **04:00 UTC** to pull the latest upstream OpenSim release from the repository releases redirect link, ensuring you are always up to date.

---

## License

Distributed under the **MIT License**. See the `LICENSE` file for details.
