# Hypixel SkyBlock Backend

This repository is the main project for a set of backend services used to collect Hypixel SkyBlock auction data and pass it to consumers that analyse potential flips. The shared models library, updater, and flipper are maintained as Git submodules.

The services communicate through Kafka-compatible topics. PostgreSQL stores auction data, while the shared models library keeps the message formats consistent between services.

## Repository structure

The three directories marked as submodules are separate repositories checked out at fixed commits by this main repository.

```text
HypixelSkyblockBackend/
├── lib/
│   └── skyblock-shared-models/   # Git submodule: shared Java models
├── services/
│   ├── skyblock-updater/         # Git submodule: auction data ingestion
│   └── skyblock-flipper/         # Git submodule: flip-analysis template
├── docker-compose.yml
├── .env                          # local database overrides, optional
└── README.md
```

-   `lib/skyblock-shared-models` - Shared Java models used in Kafka messages.
-   `services/skyblock-updater` - Fetches auction data from the Hypixel API, stores it in PostgreSQL, and publishes auction events.
-   `services/skyblock-flipper` - A template service that consumes auction events and publishes flip events after a private flip engine has been added.
-   `docker-compose.yml` - Starts PostgreSQL, Redpanda, and the backend services together.

The flipper implementation is intentionally incomplete in this repository. The private implementation of `FlipperEngineService` is not included and must be supplied separately before the service can provide useful flip results.

## Requirements

-   Docker and Docker Compose
-   Java 21 and Maven, when building a module outside Docker

## Quick start

Clone the main repository together with its submodules:

```bash
git clone --recurse-submodules https://github.com/jaron2668/HypixelSkyblockBackend.git
cd HypixelSkyblockBackend
```

If the main repository was already cloned without its submodules, initialize them with:

```bash
git submodule update --init --recursive
```

The submodules are checked out at the commits recorded by the main repository. To update them to newer commits, update each submodule separately and then commit the resulting submodule changes in the main repository.

The default PostgreSQL settings are suitable for local development. To override them, create a `.env` file in the repository root:

```dotenv
POSTGRES_DB=skyblock_db
POSTGRES_USER=skyblock_user
POSTGRES_PASSWORD=change_me
```

Build and start the stack:

```bash
docker compose up --build
```

This starts Redpanda, PostgreSQL, the updater, and the flipper template. The first updater run loads the current auction data. New and ended auction events are then published for downstream consumers.

## Building the submodules

The services use the shared models artifact as a local Maven dependency. Build and install the library first, then build the services:

```bash
cd lib/skyblock-shared-models
mvn clean install
```

Then build either service from its own directory:

```bash
cd services/skyblock-updater
mvn clean verify

cd ../skyblock-flipper
mvn clean verify
```

Each submodule has its own README with details about its responsibilities, events, and local build requirements.

## Runtime details

Kafka is available to the containers at `redpanda:9092`. PostgreSQL is available to the containers at `postgres_db:5432` and is exposed on `127.0.0.1:5432` for local access. The Redpanda admin API is not exposed by default.

The main Kafka topics are:

-   `updater-newauction` - A newly observed active auction, serialized as JSON.
-   `updater-endedauction` - The UUID of an auction that has ended.
-   `flipper-newflip` - A flip identified by the private flipper implementation.
-   `flipper-endedflip` - The UUID of a flip whose auction has ended.

View service output with commands such as:

```bash
docker logs -f skyblock-updater
docker logs -f skyblock-flipper
```

## License

See [LICENSE.txt](LICENSE.txt) and the third-party license files included with each submodule.

## Disclaimer

This project is not affiliated with, endorsed by, or associated with Hypixel Inc. "Hypixel" and related names are trademarks of Hypixel Inc. This is an independent community project intended for educational and personal use.
