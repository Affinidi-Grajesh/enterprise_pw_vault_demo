# Temasek Example Enterprise Password Vault

An example demo using DIDComm showing the ease of retrieving a password easily and
securely.

## Components

This demo is made up of two separate applications:

1. Service App
   - Connects to a DIDComm Mediator and listens for incoming requests from clients
     and responds with a password.
2. Client App
   - Issues secured password requests to a Password Service App and displays the
     result

## Service Application

```bash
cargo run --bin service
```

The first time you run this app, it will run a setup wizard that will help generate
a default config.

This Config file is placed in the current directory as `config.json`

Deleting this Config file will cause the Service App to rerun the setup wizard.

## Client Application

```bash
cargo run --bin client --  --help
```

The client will auto-generate a DID each time (random) and send the password
request to the [Service Application](#service-application).

**Features:**

- Message forwarding via a DIDComm Mediator is handled for you
- DID Support including full Public Key Infrastructure (PKI) is included
- End-to-End encryption and Authorization is handled for you
