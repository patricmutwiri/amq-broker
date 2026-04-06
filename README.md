# AMQ Broker Podman Compose Baseline

Production-oriented Podman Compose baseline for a single-node ActiveMQ Artemis / Red Hat AMQ Broker deployment on Linux hosts.

This repository provides:

- a hardened [compose.yaml](/Users/mutwiri/Projects/devops/amq/compose.yaml) for Podman Compose
- a sample [.env.example](/Users/mutwiri/Projects/devops/amq/.env.example) for environment-driven configuration
- an externalized [broker.xml](/Users/mutwiri/Projects/devops/amq/config/etc-override/broker.xml) baseline for secure broker behavior

The current baseline is designed for:

- single-node production deployments
- persistent storage
- TLS-enabled broker listeners
- constrained port exposure
- future extension toward HA and clustering

## Layout

```text
.
├── compose.yaml
├── .env.example
├── config/
│   └── etc-override/
│       └── broker.xml
├── certs/
│   ├── broker-keystore.p12
│   ├── broker-truststore.p12
│   ├── amq-broker-root-ca.crt
│   └── amq-broker-root-ca.key
└── broker-instance/
```

Notes:

- `certs/amq-broker-root-ca.crt`, `certs/broker-keystore.p12`, and `certs/broker-truststore.p12` are committed as dev bootstrap materials.
- `certs/amq-broker-root-ca.key` and `broker-instance/` are local runtime materials and should not be committed.
- `broker-instance/` contains persisted broker state and the active generated instance config.
- `config/etc-override/broker.xml` is the source-of-truth config baseline for this repo.

## What AMQ Uses At Runtime

The current broker TLS acceptor uses these files at runtime:

- `certs/broker-keystore.p12`
- `certs/broker-truststore.p12`

The root CA certificate is useful for client trust distribution:

- `certs/amq-broker-root-ca.crt`

The root CA private key should be treated as highly sensitive:

- `certs/amq-broker-root-ca.key`

In a real production setup, keep the CA private key off the broker host and manage certificates through your PKI process.

## Quick Start

1. Copy `.env.example` to `.env` if you want a local env file:

   ```bash
   cp .env.example .env
   ```

2. Create the required directories:

   ```bash
   mkdir -p broker-instance config/etc-override certs
   ```

3. Place your broker TLS stores in `certs/`:

   - `broker-keystore.p12`
   - `broker-truststore.p12`

4. Start the broker:

   ```bash
   podman compose -f compose.yaml --env-file .env up -d
   ```

5. Check service state:

   ```bash
   podman compose -f compose.yaml --env-file .env ps
   ```

6. Open the management console:

   [http://127.0.0.1:8161/console](http://127.0.0.1:8161/console)

## Credentials

The bootstrap admin credentials come from environment variables:

- `ARTEMIS_USER`
- `ARTEMIS_PASSWORD`

Do not store real production passwords in a committed `.env` file.

## TLS Notes

The current broker config expects:

- keystore path: `/etc/amq-certs/broker-keystore.p12`
- truststore path: `/etc/amq-certs/broker-truststore.p12`
- keystore type: `PKCS12`
- truststore type: `PKCS12`

The sample certificate set used during validation was generated for:

- broker DNS: `amq-broker.dev`
- root CA CN: `amq-broker-root-ca`
- organization: `Patrick Org`
- organizational unit: `Software Engineering`

If you replace the certificate set, ensure the broker certificate SANs match the hostnames your clients will use.

## Validation Commands

Inspect the root CA certificate:

```bash
openssl x509 -in certs/amq-broker-root-ca.crt -text -noout
```

Inspect the root CA private key:

```bash
openssl rsa -in certs/amq-broker-root-ca.key -check -noout
```

Inspect the broker keystore:

```bash
openssl pkcs12 -in certs/broker-keystore.p12 -info -nokeys -passin pass:changeit
```

Inspect the broker truststore:

```bash
openssl pkcs12 -in certs/broker-truststore.p12 -info -nokeys -passin pass:changeit
```

Test the live TLS listener:

```bash
openssl s_client -connect 127.0.0.1:61616 -servername amq-broker.dev -showcerts
```

## Operational Notes

- Ports are bound to `127.0.0.1` by default.
- The management console is exposed on `8161`.
- The broker listener is exposed on `61616`.
- The healthcheck has a `90s` start period, so `starting` is expected immediately after boot.
- The current broker instance persists data under `broker-instance/`.
- The embedded web console on `apache/artemis:2.53.0` currently loads with an empty plugin list in this repo's tested setup, so Artemis queue administration is most reliable from the CLI commands below.

## Queue Administration From The CLI

The commands below were verified against the running Podman deployment in this repository.

Set these shell variables once before running the examples:

```bash
export AMQ_USER=mutwiri
export AMQ_PASSWORD=31344baee9daa086c1c2138857083a739f965ab57fcd7c060788d6ab74707d65
export AMQ_URL='tcp://localhost:61616?sslEnabled=true;trustStorePath=/etc/amq-certs/broker-truststore.p12;trustStorePassword=changeit;trustStoreType=PKCS12;verifyHost=false'
```

`verifyHost=false` is required for the current dev certificate set because the broker certificate is issued for `amq-broker.dev`, while these examples connect through `localhost`.

List queues:

```bash
podman exec amq-broker sh -lc '/var/lib/artemis-instance/bin/artemis queue stat --url "'"$AMQ_URL"'" --user "'"$AMQ_USER"'" --password "'"$AMQ_PASSWORD"'" --verbose'
```

Sample response:

```text
|NAME        |ADDRESS     |CONSUMER_COUNT|MESSAGE_COUNT|MESSAGES_ADDED|
|DLQ         |DLQ         |0             |0            |0             |
|ExpiryQueue |ExpiryQueue |0             |0            |0             |
```

Create a durable queue and its address:

```bash
podman exec amq-broker sh -lc '/var/lib/artemis-instance/bin/artemis queue create --name demoQueue --address demoQueue --anycast --auto-create-address --durable --url "'"$AMQ_URL"'" --user "'"$AMQ_USER"'" --password "'"$AMQ_PASSWORD"'" --silent'
```

Sample response:

```text
Queue demoQueue created successfully.
```

Send a test message:

```bash
podman exec amq-broker sh -lc '/var/lib/artemis-instance/bin/artemis producer --destination "demoQueue::demoQueue" --message-count 1 --message "hello from cli" --url "'"$AMQ_URL"'" --user "'"$AMQ_USER"'" --password "'"$AMQ_PASSWORD"'" --silent'
```

Sample response:

```text
Produced: 1 messages
```

Browse queued messages:

```bash
podman exec amq-broker sh -lc '/var/lib/artemis-instance/bin/artemis browser --destination queue://demoQueue --message-count 10 --url "'"$AMQ_URL"'" --user "'"$AMQ_USER"'" --password "'"$AMQ_PASSWORD"'" --verbose'
```

Sample response:

```text
Connection brokerURL = tcp://localhost:61616
browsing hello from cli
messages = 1
browsed: 1 messages
```

Consume a single message:

```bash
podman exec amq-broker sh -lc '/var/lib/artemis-instance/bin/artemis consumer --destination "demoQueue::demoQueue" --message-count 1 --break-on-null --receive-timeout 1000 --url "'"$AMQ_URL"'" --user "'"$AMQ_USER"'" --password "'"$AMQ_PASSWORD"'" --silent'
```

Sample response:

```text
Consumed: 1 messages
```

Purge a queue without deleting it:

```bash
podman exec amq-broker sh -lc '/var/lib/artemis-instance/bin/artemis queue purge --name demoQueue --url "'"$AMQ_URL"'" --user "'"$AMQ_USER"'" --password "'"$AMQ_PASSWORD"'" --silent'
```

Sample response:

```text
Queue demoQueue purged successfully.
```

Delete the queue and remove its address:

```bash
podman exec amq-broker sh -lc '/var/lib/artemis-instance/bin/artemis queue delete --name demoQueue --autoDeleteAddress --url "'"$AMQ_URL"'" --user "'"$AMQ_USER"'" --password "'"$AMQ_PASSWORD"'" --silent'
```

Sample response:

```text
Queue demoQueue deleted successfully.
```

## Important Config Behavior

On first broker startup, the container creates an Artemis instance under `broker-instance/` and copies config into the active `broker-instance/etc/` directory.

That means:

- changes to `config/etc-override/broker.xml` do not automatically replace the already-persisted active `broker-instance/etc/broker.xml`
- if you are iterating on config after first boot, sync the updated config into `broker-instance/etc/broker.xml` or recreate the broker instance directory

## Security Notes

- Keep host port exposure restricted unless you have a deliberate firewall or proxy design.
- Rotate sample passwords before any real deployment.
- Do not commit private keys, populated `.env` files, or broker runtime state.
- Consider moving from plain env secrets to file-backed or secret-manager-backed injection for production.
- Keep the root CA private key offline wherever possible.

## Future Improvements

- move keystore and truststore passwords to file-backed secrets
- add automated config sync or a cleaner first-boot override flow
- add explicit HA / clustering profiles
- add a systemd unit for Linux host startup
- add client-authenticated TLS if mutual TLS is required
