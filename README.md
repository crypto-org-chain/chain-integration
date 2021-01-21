# Crypto.com Chain Integration documentation

## Useful Links

- Crypto.com Chain website: https://chain.crypto.com/
- GitHub Repository: https://github.com/crypto-com/chain-main
- Official Documentation: https://chain.crypto.com/docs/

## Node and RPC setup notes

[node-and-rpc-setup-notes.md](./node-and-rpc-setup-notes.md)

## Setup Guide

https://chain.crypto.com/docs/getting-started/croeseid-testnet.html

## API Documentation

There are a few ways to access to the Crypto.com Chain

- Tendermint RPC
    - Raw but most-completed data
    - Hosted documentation (Latest master only): https://docs.tendermint.com/master/rpc/
    - Swagger file: https://github.com/tendermint/tendermint/blob/v0.34.3/rpc/openapi/openapi.yaml
- gRPC Based
    - Processed chain information. Old state maybe pruned from time-to-time. To avoid state pruning, update `pruning = "nothing"` in `app.toml`
    - There are two query methods based on gRPC:
        1. [gRPC Server](./grpc/README.md)
        2. [gRPC Proxy RESTful Server](./grpc-proxy-rest/README.md)
            - Swagger UI (Please switch the network to latest master): https://cosmos.network/rpc/v0.37.9
            - [Swagger file](./grpc-proxy-rest/swagger.yml)

## API Clients

- TypeScript library: https://github.com/crypto-com/chain-jslib
- Cosmosjs (Stargate ready): https://github.com/cosmos/cosmjs/tree/master/packages/stargate
- Python library: https://pypi.org/project/chainlibpy/#description for teams coding in python
- Rust library (note that it is not feature complete): https://github.com/crypto-com/chainlib-rs
## Indexing

- chain-indexing (indexing on-chain data): https://github.com/crypto-com/chain-indexing
## Node Monitoring (Prometheus)

The Ansible playbook for deploying Prometheus and some of the rules we are using are under `prometheus`.

## Croeseid Testnet Public Nodes

- Tendermint: https://testnet-croeseid.crypto.com:26657/
- Cosmos RESTful gRPC: https://testnet-croeseid.crypto.com:1317/

## Block Explorer

https://chain.crypto.com/explorer

## Community

Discord: https://discord.gg/5JTk2ppsY3