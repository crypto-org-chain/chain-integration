# Crypto.com Chain Integration documentation

## Useful Links

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

## API Client (JavaScript)

Cosmosjs (Stargate ready): https://github.com/cosmos/cosmjs/tree/master/packages/stargate
chain-jslib: https://github.com/crypto-com/chain-jslib

## Node Monitoring (Prometheus)

The Ansible playbook for deploying Prometheus and some of the rules we are using are under `prometheus`.

## Croeseid Testnet Public Nodes

Tendermint: https://testnet-croeseid.crypto.com:26657/
Cosmoe RESTful gRPC: https://testnet-croeseid.crypto.com:1317/

## Block Explorer

https://chain.crypto.com/explorer

## Community

Discord: https://discord.gg/5JTk2ppsY3