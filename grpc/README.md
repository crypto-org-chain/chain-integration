# gRPC

`chain-maind` comes with a RESTful API on top of the gRPC.

Alternatively, it also comes with gRPC proxy RESTful server, which can be access via HTTP. You can find it under folder `grpc-proxy-test`.

## Enable API on server

Update your chain configuration file `app.toml` and enable gRPC server by
```toml
[grpc]

# Enable defines if the gRPC server should be enabled.
enable = true

# Address defines the gRPC server address to bind to.
address = "0.0.0.0:9090"
```

Additionally you can update the address to your need

## Running Query

All the proto files are available under `proto` folder (organized by modules), additionally, a gRPC documentation is generated as `grpc-reference.md`.

Generally you will need to
1. Install a gRPC client
2. Load the `proto` files, look for files with name `query.proto`, for example `proto/bank/v1beta1/query.proto`.
3. Query to the specific endpoint

Note that some of the query request accepts a buffer. So you will need to use tool like [this](https://repl.it/repls/BogusPreciousEvents#index.js) to convert string to a buffer.
