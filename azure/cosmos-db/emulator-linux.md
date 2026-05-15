---
title: Linux-based emulator - vNext
description: Use the Azure Cosmos DB Linux-based emulator to test your applications against API for NoSQL endpoints.
author: abhirockzz
ms.author: guabhishek
ms.service: azure-cosmos-db
ms.topic: how-to
ms.date: 6/02/2026
# CustomerIntent: As a developer, I want to use the Linux-based Azure Cosmos DB emulator so that I can develop my application against a database during development.
appliesto:
  - ✅ NoSQL
ai-usage: ai-assisted
---

# Linux-based emulator - vNext

The next generation of the Azure Cosmos DB emulator is entirely Linux-based and is available as a Docker container. It supports running on a wide variety of processors and operating systems.

> [!IMPORTANT]
> This version of the emulator only supports the API for NoSQL in [gateway mode](sdk-connection-modes.md#available-connectivity-modes), with a select subset of features. For more information, see [feature support](#feature-support).

## Prerequisites

- [Docker](https://www.docker.com/)

## Installation

Get the Docker container image using `docker pull`. The container image is published to the [Microsoft Artifact Registry](https://mcr.microsoft.com/) as `mcr.microsoft.com/cosmosdb/linux/azure-cosmos-emulator:vnext-latest`.

```bash
docker pull mcr.microsoft.com/cosmosdb/linux/azure-cosmos-emulator:vnext-latest
```

## Running

To run the container, use `docker run`. Afterwards, use `docker ps` to validate that the container is running.

```bash
docker run --detach --publish 8081:8081 --publish 8080:8080 --publish 1234:1234 mcr.microsoft.com/cosmosdb/linux/azure-cosmos-emulator:vnext-latest

docker ps
```

```output
CONTAINER ID   IMAGE                                                             COMMAND                  CREATED         STATUS         PORTS                                                                                  NAMES
c1bb8cf53f8a   mcr.microsoft.com/cosmosdb/linux/azure-cosmos-emulator:vnext-latest  "/bin/bash -c /home/…"   5 seconds ago   Up 5 seconds   0.0.0.0:1234->1234/tcp, :::1234->1234/tcp, 0.0.0.0:8081->8081/tcp, :::8081->8081/tcp   <container-name>
```

The emulator includes two components:

- **Data Explorer** - Interactively explore the data in the emulator. By default, this component runs on port `1234`.
- **Azure Cosmos DB emulator** - A local version of the Azure Cosmos DB database service. By default, this component runs on port `8081`.

The emulator gateway endpoint uses port `8081` at the address <http://localhost:8081>. To navigate to the Data Explorer, use the address <http://localhost:1234> in your web browser. The gateway endpoint is typically available immediately, but Data Explorer might take a few seconds to start.

### Health probe

The emulator exposes a health probe endpoint on port `8080`. Use this endpoint to determine when the emulator is fully initialized and ready to accept requests.

The following endpoints are available:

- <http://localhost:8080/alive> — Liveness probe.
- <http://localhost:8080/ready> — Readiness probe.
- <http://localhost:8080/status> — Detailed status.

> [!NOTE]
> The legacy log message `System is now fully ready to accept requests` is still emitted for backward compatibility but might be removed in a future version. Use the health probe for readiness checks instead.

### HTTPS mode

The .NET and Java SDKs don't support HTTP mode in the emulator. Since this version of the emulator starts with HTTP by default, you will need to explicitly enable HTTPS when starting the container (see below). For the Java SDK, you will also need to [install certificates](#installing-certificates-for-java-sdk).

```bash
docker run --detach --publish 8081:8081 --publish 8080:8080 --publish 1234:1234 mcr.microsoft.com/cosmosdb/linux/azure-cosmos-emulator:vnext-latest --protocol https
```

When you use HTTPS with persisted data volumes, the emulator automatically regenerates SSL certificates at startup, so you don't need to manage certificate renewal.

## Docker commands

The following table summarizes the available Docker commands for configuring the emulator. This table details the corresponding arguments, environment variables, allowed values, default settings, and descriptions of each command.

| Requirement                                      | Arg                   | Env                     | Allowed values                                     | Default                        | Description                                                                                                                                                                                                                                                         |
| ------------------------------------------------ | --------------------- | ----------------------- | -------------------------------------------------- | ------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Print the settings to stdout from the container  | `--help`, `-h`        | N/A                     | N/A                                                | N/A                            | Display information on available configuration                                                                                                                                                                                                                      |
| Set the port of the Cosmos endpoint              | `--port [INT]`        | PORT                    | INT                                                | 8081                           | The port of the Cosmos endpoint on the container. You still need to publish this port (for example, `-p 8081:8081`).                                                                                                                                                |
| Specify the protocol used by the Cosmos endpoint | `--protocol`          | PROTOCOL                | `https`, `http`, `https-insecure`                  | `http`                         | The protocol of the Cosmos endpoint on the container.                                                                                                                                                                                                               |
| Enable the data explorer                         | `--enable-explorer`   | ENABLE_EXPLORER         | `true`, `false`                                    | `true`                         | Enable running the Cosmos Data Explorer on the same container.                                                                                                                                                                                                      |
| Set the port used by the data explorer           | `--explorer-port`     | EXPLORER_PORT           | INT                                                | 1234                           | The port of the Cosmos Data Explorer on the container. You still need to publish this port (for example, `-p 1234:1234`).                                                                                                                                           |
| Specify the protocol used by the data explorer   | `--explorer-protocol` | EXPLORER_PROTOCOL       | `https`, `http`, `https-insecure`                  | `<the value of --protocol>`    | The protocol of the Cosmos Data Explorer on the container. Defaults to the protocol setting on the Cosmos endpoint.                                                                                                                                                 |
| Customize the gateway public endpoint            | `--gateway-endpoint`  | GATEWAY_PUBLIC_ENDPOINT | N/A                                                | `localhost`                    | The public gateway endpoint. Defaults to `localhost`.                                                                                                                                                                                                               |
| Specify the key via file                         | `--key-file [PATH]`   | KEY_FILE                | PATH                                               | `<default secret>`             | Override default key with the key specified in the file. You need to mount this file into the container (for example, if KEY_FILE=/mykey, you'd add an option like the following to your docker run: `--mount type=bind,source=./myKey,target=/myKey`)              |
| Set the data path                                | `--data-path [PATH]`  | DATA_PATH               | PATH                                               | `/data`                        | Specify a directory for data. Frequently used with `docker run --mount` option (for example, if DATA_PATH=/usr/cosmos/data, you'd add an option like the following to your docker run: `--mount type=bind,source=./.local/data,target=/usr/cosmos/data`)            |
| Specify the cert path to be used for https       | `--cert-path [PATH]`  | CERT_PATH               | PATH                                               | `<default cert>`               | Specify a path to a certificate for securing traffic. You need to mount this file into the container (for example, if CERT_PATH=/mycert.pfx, you'd add an option like the following to your docker run: `--mount type=bind,source=./mycert.pfx,target=/mycert.pfx`) |
| Specify the cert secret to be used for https     | N/A                   | CERT_SECRET             | string                                             | `<default secret>`             | The secret for the certificate specified on CERT_PATH.                                                                                                                                                                                                              |
| Set the log level                                | `--log-level [LEVEL]` | LOG_LEVEL               | `quiet`, `error`, `warn`, `info`, `debug`, `trace` | `info`                         | The verbosity of logs that are emitted by the emulator and data explorer.                                                                                                                                                                                           |
| Enable OpenTelemetry OTLP exporter               | `--enable-otlp`       | ENABLE_OTLP_EXPORTER    | `true`, `false`                                    | `false`                        | Enable OpenTelemetry integration.                                                                                                                                                                                                                                   |
| Enable console exporter                          | `--enable-console`    | ENABLE_CONSOLE_EXPORTER | `true`, `false`                                    | `false`                        | Enable console output of telemetry data (useful for debugging).                                                                                                                                                                                                     |
| Enable verbose mode                              | `--verbose`           | VERBOSE                 | `true`, `false`                                    | `false`                        | Enable verbose mode to print PostgreSQL logs (pglog) to console. Useful for debugging.                                                                                                                                                                              |
| Set query buffer size                            | `--query-buffer-size` | QUERY_BUFFER_SIZE_KB    | INT                                                | 4096 (4 MB), max 65536 (64 MB) | The maximum size in KB for query result buffers. Increase this if you encounter HTTP 500 errors on large queries.                                                                                                                                                   |
| Enable seeding data on container start           | `--enable-init-data`  | ENABLE_INIT_DATA        | `true`, `false`                                    | `false`                        | Run any `.csh` scripts found at the top level of the init directory in alphabetical order before the emulator accepts requests. See [Azure Cosmos DB Shell integration](#use-azure-cosmos-db-shell-with-the-emulator).                                              |
| Set the directory for init scripts               | `--init-path [PATH]`  | INIT_PATH               | PATH                                               | `/init`                        | The directory the emulator scans for `.csh` seed scripts when `ENABLE_INIT_DATA=true`.                                                                                                                                                                              |
| Enable diagnostic info being sent to Microsoft   | `--enable-telemetry`  | ENABLE_TELEMETRY        | `true`, `false`                                    | `true`                         | Enable sending usage data to Microsoft to help us improve the emulator.                                                                                                                                                                                             |


## Feature support

Not all Azure Cosmos DB features are supported by the emulator, and some aren't planned for future support. This table includes the state of various features and their level of support.

| Feature                                            | Support               |
| -------------------------------------------------- | --------------------- |
| **Batch API**                                      | ✅ Supported           |
| **Bulk API**                                       | ✅ Supported           |
| **Change Feed**                                    | ✅ Supported           |
| **Create and read document with utf data**         | ✅ Supported           |
| **Create collection**                              | ✅ Supported           |
| **Create collection twice conflict**               | ✅ Supported           |
| **Create collection with custom index policy**     | ⚠️ No-op               |
| **Create collection with ttl expiration**          | ✅ Supported           |
| **Create database**                                | ✅ Supported           |
| **Create database twice conflict**                 | ✅ Supported           |
| **Create document**                                | ✅ Supported           |
| **Create partitioned collection**                  | ✅ Supported           |
| **Delete collection**                              | ✅ Supported           |
| **Delete database**                                | ✅ Supported           |
| **Delete document**                                | ✅ Supported           |
| **Get and change collection performance**          | ⚠️ Not yet implemented |
| **Insert large document**                          | ✅ Supported           |
| **Patch document**                                 | ✅ Supported           |
| **Query partitioned collection in parallel**       | ⚠️ Not yet implemented |
| **Query with aggregates**                          | ✅ Supported           |
| **Query with and filter**                          | ✅ Supported           |
| **Query with and filter and projection**           | ✅ Supported           |
| **Query with equality**                            | ✅ Supported           |
| **Query with equals on id**                        | ✅ Supported           |
| **Query with joins**                               | ✅ Supported           |
| **Query with order by**                            | ✅ Supported           |
| **Query with order by for partitioned collection** | ✅ Supported           |
| **Query with order by numbers**                    | ✅ Supported           |
| **Query with order by strings**                    | ✅ Supported           |
| **Query with paging**                              | ✅ Supported           |
| **Query with range operators date times**          | ✅ Supported           |
| **Query with range operators on numbers**          | ✅ Supported           |
| **Query with range operators on strings**          | ✅ Supported           |
| **Query with single join**                         | ✅ Supported           |
| **Query with string math and array operators**     | ✅ Supported           |
| **Query with subdocuments**                        | ✅ Supported           |
| **Query with two joins**                           | ✅ Supported           |
| **Query with two joins and filter**                | ✅ Supported           |
| **Read collection**                                | ✅ Supported           |
| **Read collection feed**                           | ⚠️ Not yet implemented |
| **Read database**                                  | ✅ Supported           |
| **Read database feed**                             | ⚠️ Not yet implemented |
| **Read document**                                  | ✅ Supported           |
| **Read document feed**                             | ✅ Supported           |
| **Replace document**                               | ✅ Supported           |
| **Request Units**                                  | ⚠️ Not yet implemented |
| **Stored procedures**                              | ❌ Not planned         |
| **Triggers**                                       | ❌ Not planned         |
| **UDFs**                                           | ❌ Not planned         |
| **Update collection**                              | ⚠️ No-op               |
| **Update document**                                | ✅ Supported           |
| **Offers endpoint**                                | ⚠️ No-op               |
| **Users endpoint**                                 | ⚠️ No-op               |
| **Permissions endpoint**                           | ⚠️ No-op               |
| **Client Encryption Keys (CEK)**                   | ⚠️ No-op               |

> [!NOTE]
> Features marked **No-op** accept requests and return valid HTTP status codes but don't execute the underlying operation. Your code won't break, but don't depend on these features for functional behavior. Custom index policies and collection updates are accepted for compatibility, but queries aren't optimized by custom indexes.

## Installing certificates for Java SDK

When using the [Java SDK for Azure Cosmos DB](sdk-java-v4.md) with this version of the emulator in https mode, it is necessary to install its certificates to your local Java trust store.

### Get certificate

In a `bash` window, run the following: 

```bash
# If the emulator was started with /AllowNetworkAccess, replace localhost with the actual IP address of it:
EMULATOR_HOST=localhost
EMULATOR_PORT=8081
EMULATOR_CERT_PATH=/tmp/cosmos_emulator.cert
openssl s_client -connect ${EMULATOR_HOST}:${EMULATOR_PORT} </dev/null | sed -ne '/-BEGIN CERTIFICATE-/,/-END CERTIFICATE-/p' > $EMULATOR_CERT_PATH
```

### Install certificate

Navigate to the directory of your java installation where `cacerts` file is located (replace below with correct directory):

```bash
cd "C:/Program Files/Eclipse Adoptium/jdk-17.0.10.7-hotspot/bin"
```

Import the cert (you may be asked for a password, the default value is "changeit"):

```bash
keytool -cacerts -importcert -alias cosmos_emulator -file $EMULATOR_CERT_PATH
```

If you get an error because the alias already exists, delete it and then run the above again:

```bash
keytool -cacerts -delete -alias cosmos_emulator
```

## Use Azure Cosmos DB Shell with the emulator

[Azure Cosmos DB Shell](shell/overview.md) is an open-source command-line interface (CLI) that enables you to interact with your Azure Cosmos DB databases using bash-like commands. The shell is included in the emulator container image, so you don't need to install it separately.

To start an interactive session against the running emulator, use `docker exec`:

```bash
docker exec -it <container-name> cosmoshell.sh
```

> [!NOTE]
> The `cosmoshell.sh` wrapper auto-detects the emulator endpoint and authenticates with the well-known account key. The underlying binary is located at `/usr/local/bin/cosmosdbshell` inside the container.

The shell can also run scripts when the container first starts, so databases, containers, seed documents, and any other state are ready before your application connects. To opt in, set `ENABLE_INIT_DATA=true` or pass `--enable-init-data=true`. The default is `false`. When enabled, the emulator processes any `.csh` files at the top level of `/init` in alphabetical order. To use a different directory, point `INIT_PATH` (or `--init-path`) at it.

### Use the included sample data

The image ships with example seed scripts under `/scripts/init_examples/`, which are copied to `/init` at build time. To load them, start the container with `ENABLE_INIT_DATA=true`:

```bash
docker run --name emulator --rm -e ENABLE_INIT_DATA=true -p 8081:8081 mcr.microsoft.com/cosmosdb/linux/azure-cosmos-emulator:vnext-latest
```

You can then connect to the shell and explore the data:

```bash
docker exec emulator cosmoshell.sh -c 'ls; cd SampleDB; ls'
```

### Seed your own data

You can seed your own data. For example, create three files (`01-create-db.csh`, `02-load-movies.csh`, and `03-verify.csh`) in a local folder `movies`.

```bash
mkdir movies && cd movies
```

`01-create-db.csh`:

```text
mkdb MovieDB
mkcon Movies /genre --database=MovieDB
```

`02-load-movies.csh`:

```text
mkitem -container Movies --database=MovieDB '{"id":"m1","genre":"scifi","title":"Inception","year":2010}'
mkitem -container Movies --database=MovieDB '{"id":"m2","genre":"scifi","title":"The Matrix","year":1999}'
mkitem -container Movies --database=MovieDB '{"id":"m3","genre":"drama","title":"The Godfather","year":1972}'
```

`03-verify.csh`:

```text
query "SELECT VALUE COUNT(1) FROM c" --database=MovieDB --container=Movies
```

Then run the emulator with that directory mounted at `/init`:

```bash
docker run --name emulator --rm -e ENABLE_INIT_DATA=true -v "$(pwd):/init" -p 8081:8081 mcr.microsoft.com/cosmosdb/linux/azure-cosmos-emulator:vnext-latest
```

You can verify the data was loaded by connecting to the shell and running a query:

```bash
docker exec emulator cosmoshell.sh -c 'ls; query "SELECT VALUE COUNT(1) FROM c" --database=MovieDB --container=Movies'
```

> [!NOTE]
> Seed data in `/init` is processed alphabetically. To ensure consistent ordering, prefix files with `01-`, `02-`, `03-`, and so on. Either always use 2-digit prefixes (`01-`, `02-`, ..., `99-`) or 3-digit prefixes.
>
> Filenames must contain only letters, digits, dots, underscores, and hyphens. For example, `01-create-catalog.csh` and `02_load_books.csh` are valid, but `01 create catalog.csh` (spaces) and `café.csh` (non-ASCII character) are not.

### Share state across init scripts

Each `.csh` file in `/init` runs in the same shell session, so state set in one file is still in effect in the next. This includes the current path set by `cd`, custom commands defined with `def`, and `for`-loop variables.

You can use this to avoid repeating `--database=MovieDB` on every line of the previous example. Add `cd MovieDB` to the end of `01-create-db.csh`:

`01-create-db.csh`:

```text
mkdb MovieDB
mkcon Movies /genre --database=MovieDB
cd MovieDB
```

Later files can then omit `--database=MovieDB`:

`02-load-movies.csh`:

```text
mkitem -container Movies '{"id":"m1","genre":"scifi","title":"Inception","year":2010}'
mkitem -container Movies '{"id":"m2","genre":"scifi","title":"The Matrix","year":1999}'
mkitem -container Movies '{"id":"m3","genre":"drama","title":"The Godfather","year":1972}'
```

`03-verify.csh`:

```text
query "SELECT VALUE COUNT(1) FROM c" --container=Movies
```

### Persist data across restarts

By default, every `docker run --rm` wipes the emulator's data when the container stops, so init scripts re-run from scratch on the next start. To keep your seeded data (and skip init on subsequent runs), bind-mount a host folder at `/data`:

```bash
mkdir -p ./cosmos-data

docker run --rm -p 8081:8081 -e ENABLE_INIT_DATA=true -v "$(pwd):/init" -v "$(pwd)/cosmos-data:/data" mcr.microsoft.com/cosmosdb/linux/azure-cosmos-emulator:vnext-latest
```

On any subsequent start with the same `-v "$(pwd)/cosmos-data:/data"` flag, the emulator detects the existing data and skips the init process, so your previously seeded state is preserved. To re-seed from scratch, delete the host folder (`rm -rf ./cosmos-data`) and start again.

### Connect from a local shell

You can also connect to the running emulator from a locally installed version of Azure Cosmos DB Shell. Point it at the emulator's endpoint and key:

```bash
cosmosdbshell --connect "AccountEndpoint=http://localhost:8081/;AccountKey=<emulator-account-key>"
```

> [!NOTE]
> Replace `<emulator-account-key>` with the well-known emulator account key.

## OpenTelemetry support

[OpenTelemetry](https://opentelemetry.io/) is an open-source observability framework that provides a collection of tools, APIs, and SDKs for instrumenting, generating, collecting, and exporting telemetry data. The OpenTelemetry Protocol (OTLP) is the protocol used by OpenTelemetry to transmit telemetry data between components.

You can use OpenTelemetry with the emulator to monitor and trace your application. The emulator supports telemetry options, which can be configured through environment variables or command-line flags when running the Docker container.

The emulator exports the following metrics. These are available through any metrics backend that supports OTLP and provides valuable insights into the database's performance and health:

- Request Rates: Shows the traffic patterns for different operation types
- Query Execution Times: Measures the time taken to execute different queries
- Resource Utilization: CPU, memory usage and connection pool metrics
- Error Rates: Tracking of errors by type and endpoint

> [!NOTE]
> The emulator supports conditional TLS for the OTLP exporter, so you can integrate with observability platforms that require secure connections.

Detailed instructions with examples [are available in the GitHub repository](https://github.com/Azure/azure-cosmos-db-emulator-docker/blob/master/docs/opentelemetry.md).

## Use in continuous integration workflow

There are lots of benefits to using Docker containers in CI/CD pipelines, especially for stateful systems like databases. This could be in terms of cost-effectiveness, performance, reliability and consistency of your test suites. 

The emulator can be incorporated as part CI/CD pipelines. You can refer to this [GitHub repository](https://github.com/AzureCosmosDB/cosmosdb-linux-emulator-github-actions) that provides examples of how to use the emulator as part of a GitHub Actions CI workflow for .NET, Python, Java, and Go applications on both `x64` and `ARM64` architectures (demonstrated for Linux runner using `ubuntu`).

Here is an example of a GitHub Actions CI workflow that shows how to configure the emulator as a [GitHub Actions service container](https://docs.github.com/en/actions/use-cases-and-examples/using-containerized-services/about-service-containers) as part of a job in the workflow. GitHub takes care of starting the Docker container and destroys it when the job completes, without the need for manual intervention (such as using the `docker run` command).

```yml
name: CI demo app

on:
  push:
    branches: [main]
    paths:
      - 'java-app/**'
  pull_request:
    branches: [main]
    paths:
      - 'java-app/**'

jobs:
  build-and-test:
    runs-on: ubuntu-latest

    services:
      cosmosdb:
        image: mcr.microsoft.com/cosmosdb/linux/azure-cosmos-emulator:vnext-latest
        ports:
          - 8081:8081
        env:
          PROTOCOL: https
        
    env:
      COSMOSDB_CONNECTION_STRING: ${{ secrets.COSMOSDB_CONNECTION_STRING }}
      COSMOSDB_DATABASE_NAME: ${{ vars.COSMOSDB_DATABASE_NAME }}
      COSMOSDB_CONTAINER_NAME: ${{ vars.COSMOSDB_CONTAINER_NAME }}

    steps:

      - name: Set up Java
        uses: actions/setup-java@v3
        with:
          distribution: 'microsoft'
          java-version: '21.0.0'

      - name: Export Cosmos DB Emulator Certificate
        run: |

          sudo apt update && sudo apt install -y openssl

          openssl s_client -connect localhost:8081 </dev/null | sed -ne '/-BEGIN CERTIFICATE-/,/-END CERTIFICATE-/p' > cosmos_emulator.cert

          cat cosmos_emulator.cert

          $JAVA_HOME/bin/keytool -cacerts -importcert -alias cosmos_emulator -file cosmos_emulator.cert -storepass changeit -noprompt
      
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Run tests
        run: cd java-app && mvn test
```

This job runs on an Ubuntu runner and uses the `mcr.microsoft.com/cosmosdb/linux/azure-cosmos-emulator:vnext-latest` Docker image as a service container. It uses environment variables to configure the connection string, database name, and container name. Since in this case the job is running directly on the GitHub Actions runner machine, the **Run tests** step in the job can access the emulator is accessible using `localhost:8081` (`8081` is the port exposed by the emulator).

The **Export Cosmos DB Emulator Certificate** step is specific to Java applications since the Azure Cosmos DB Java SDK currently doesn't support `HTTP` mode in emulator. The `PROTOCOL` environment variable is set to `https` in the `services` section and this step exports the emulator certificate and import it into the Java keystore. The same applies to .NET as well.

## Limitations

In addition to features not yet supported or not planned, the following list includes current limitations of the emulator.

- The .NET SDK for Azure Cosmos DB doesn't support bulk execution in the emulator.
- If you get HTTP 500 errors on large query results, increase the query buffer size with the `--query-buffer-size` flag or the `QUERY_BUFFER_SIZE_KB` environment variable. The default is `4096` KB (`4` MB), and the maximum is `65536` KB (`64` MB).

## Reporting issues

If you encounter issues with using this version of the emulator, open an issue in the GitHub repository (<https://github.com/Azure/azure-cosmos-db-emulator-docker>) and tag it with the label `cosmosEmulatorVnextPreview`.
