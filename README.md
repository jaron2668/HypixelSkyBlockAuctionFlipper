# Hypixel SkyBlock Auction Flipper

This repository is the main project for a set of backend services and a Minecraft Fabric mod used to collect Hypixel SkyBlock auction data, analyse potential flips, and display them in-game. The shared models library, updater, flipper and the mod are maintained as Git submodules.

The services communicate through Kafka-compatible topics. PostgreSQL stores auction data, while the shared models library keeps the message formats consistent between services.

## Repository structure

The four directories marked as submodules are separate repositories checked out at fixed commits by this main repository. The backend projects consume the shared models artifact from GitHub Packages; the shared-models submodule is only needed when developing or publishing that library itself.

```text
HypixelSkyblockBackend/
├── lib/
│   └── skyblock-shared-models/   # Git submodule: shared Java models
├── services/
│   ├── skyblock-updater/         # Git submodule: auction data ingestion
│   └── skyblock-flipper/         # Git submodule: flip-analysis template
├── mod/
│   └── skyblock-auction-mod/      # Fabric client mod for displaying flips
├── docker-compose.yml
├── .env                          # local database overrides, optional
└── README.md
```

-   `lib/skyblock-shared-models` - Shared Java models used in Kafka messages.
-   `services/skyblock-updater` - Fetches auction data from the Hypixel API, stores it in PostgreSQL, and publishes auction events.
-   `services/skyblock-flipper` - A template service that consumes auction events and publishes flip events after a private flip engine has been added.
-   `mod/skyblock-auction-mod` - A Fabric client mod that consumes `flipper-newflip` and displays clickable flip notifications in Minecraft chat.
-   `docker-compose.yml` - Starts PostgreSQL, Redpanda, and the backend services together.

The flipper implementation is intentionally incomplete in this repository. The private implementation of `FlipperEngineService` is not included and must be supplied separately before the service can provide useful flip results.

## Requirements

-   Docker and Docker Compose
-   Java 21 and Maven, when building a module outside Docker
-   Java 25, Minecraft 26.1.2, and Fabric, when building or running the auction mod

## Quick start

### Backend

Clone the main repository together with its submodules:

```bash
git clone --recurse-submodules https://github.com/jaron2668/HypixelSkyBlockAuctionFlipper.git
cd HypixelSkyBlockAuctionFlipper
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

This starts Redpanda, PostgreSQL, the updater, and the flipper template. The first updater run loads the current auction data. New and ended auction events are then published for downstream consumers, including the auction mod when it is running locally.

### Fabric mod

Build the fabric mod and configure Kafka. See the [mod README](mod/skyblock-auction-mod/README.md) for the complete client setup and Kafka configuration.

Then start Minecraft with Fabric and the built mod. The mod connects to Kafka when the client starts and shuts down the consumer gracefully when the client exits.

## Runtime details

Kafka is available to the containers at `redpanda:9092`. PostgreSQL is available to the containers at `postgres_db:5432` and is exposed on `127.0.0.1:5432` for local access. The Redpanda admin API is not exposed by default.

The main Kafka topics are:

-   `updater-newauction` - A newly observed active auction, serialized as JSON.
-   `updater-endedauction` - The UUID of an auction that has ended.
-   `flipper-newflip` - A flip identified by the private flipper implementation.
-   `flipper-endedflip` - The UUID of a flip whose auction has ended.

For local Minecraft clients, Kafka is exposed at `localhost:19092`. The auction mod consumes `flipper-newflip`, displays the item name and estimated profit in chat, and links to the auction with `/ah view <auctionUuid>`.

View service output with commands such as:

```bash
docker logs -f skyblock-updater
docker logs -f skyblock-flipper
```

## Building the submodules without Docker

The services resolve `io.github.jaron2668:skyblock-shared-models:x.x.x` from GitHub Packages. Build either service directly from its own directory:

```bash
cd services/skyblock-updater
mvn clean verify

cd ../skyblock-flipper
mvn clean verify
```

Each submodule has its own README with details about its responsibilities, events, and local build requirements.

## License

This project is licensed under the **GNU General Public License v3.0 only (GPL-3.0-only)**. See LICENSE.txt included with each submodule for more details.

## Disclaimer

This project is not affiliated with, endorsed by, or associated with Hypixel Inc. "Hypixel" and related names are trademarks of Hypixel Inc. This is an independent community project intended for educational and personal use.
