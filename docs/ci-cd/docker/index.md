
## Docker

### Docker-compose

#### 2022

<details><summary>Click to expand..</summary>


### `docker-compose.yml`

```yaml
networks:
  default:
    name: localdev
    external: true

volumes:
  # Docker-managed volume for MSSQL (works reliably on Windows/WSL2 + Linux)
  mssql-data:

services:
  # One-shot init container to fix permissions on the MSSQL volume for SQL Server 2022 non-root.
  # SQL Server runs as user `mssql` (UID 10001), so the data directory must be writable by 10001.
  mssql-init:
    image: alpine:3.19
    container_name: mssql-dev-init
    user: "0:0"
    command:
      - sh
      - -c
      - >
        mkdir -p /var/opt/mssql/data /var/opt/mssql/log /var/opt/mssql/secrets &&
        chown -R 10001:0 /var/opt/mssql &&
        chmod -R g=u /var/opt/mssql
    volumes:
      - mssql-data:/var/opt/mssql
    restart: "no"

  # One-shot post-start config for SQL Server memory tuning.
  # Why: Under heavy imports SQL Server can trigger memory pressure (Error 701) and close TCP connections.
  # This container runs once after MSSQL becomes healthy and sets `max server memory (MB)`.
  #
  # Reference: https://hub.docker.com/r/microsoft/mssql-server
  mssql-config:
    image: mcr.microsoft.com/mssql/server:2022-latest
    container_name: mssql-dev-config
    depends_on:
      mssql:
        condition: service_healthy
    environment:
      MSSQL_SA_PASSWORD: "Test1234!"
      # Default is intentionally moderate (safe-by-default for dev machines).
      # Override by setting MSSQL_MAX_SERVER_MEMORY_MB (e.g. in your shell env or a .env file).
      MSSQL_MAX_SERVER_MEMORY_MB: "${MSSQL_MAX_SERVER_MEMORY_MB:-6144}"
      # Import-friendly defaults (optional overrides)
      MSSQL_MAXDOP: "${MSSQL_MAXDOP:-4}"
      MSSQL_COST_THRESHOLD_FOR_PARALLELISM: "${MSSQL_COST_THRESHOLD_FOR_PARALLELISM:-50}"
    entrypoint:
      - /bin/bash
      - -lc
    command:
      - >
        /opt/mssql-tools18/bin/sqlcmd -S mssql -U sa -P "$${MSSQL_SA_PASSWORD}" -No -Q
        "EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
         EXEC sp_configure 'max server memory (MB)', $${MSSQL_MAX_SERVER_MEMORY_MB}; RECONFIGURE;
         EXEC sp_configure 'optimize for ad hoc workloads', 1; RECONFIGURE;
         EXEC sp_configure 'max degree of parallelism', $${MSSQL_MAXDOP}; RECONFIGURE;
         EXEC sp_configure 'cost threshold for parallelism', $${MSSQL_COST_THRESHOLD_FOR_PARALLELISM}; RECONFIGURE;
         DBCC FREESYSTEMCACHE('SQL Plans');
         DBCC FREEPROCCACHE;"
    restart: "no"

  mssql:
    depends_on:
      mssql-init:
        condition: service_completed_successfully
    extends:
      file: services/mssql/service.yml
      service: mssql
    # Override volume strategy to ensure we *only* use the Docker-managed volume.
    volumes:
      - mssql-data:/var/opt/mssql
    # Memory settings (defaults are intentionally moderate / safe-by-default for dev machines).
    # Override by setting MSSQL_MEM_LIMIT / MSSQL_MEM_RESERVATION (e.g. in your shell env or a .env file).
    mem_limit: ${MSSQL_MEM_LIMIT:-8g}
    mem_reservation: ${MSSQL_MEM_RESERVATION:-4g}
```

### `services/mssql/service.yml`

```yaml
# https://hub.docker.com/r/microsoft/mssql-server#environment-variables
services:
  mssql:
    image: mcr.microsoft.com/mssql/server:2022-latest
    container_name: mssql-dev

    environment:
      ACCEPT_EULA: "Y"
      MSSQL_PID: "Developer"
      MSSQL_SA_PASSWORD: "Test1234!"

    ports:
      - "1433:1433"

    volumes:
      # Use a Docker-managed named volume by default (reliable on Windows/WSL2 + non-root MSSQL).
      # NOTE: The root compose file declares the volume and also runs a one-shot init container
      # that sets the correct ownership/permissions for SQL Server 2022 (user mssql, UID 10001).
      - mssql-data:/var/opt/mssql

    healthcheck:
      test: ["CMD-SHELL", "/opt/mssql-tools18/bin/sqlcmd -S localhost -U sa -P Test1234! -No -Q 'SELECT 1' || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
```

</details>
