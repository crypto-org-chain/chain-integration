# gRPC proxy RESTful API

`chain-maind` comes with a RESTful API on top of the gRPC.

## Enable API on server

Update your chain configuration file `app.toml` and enable gRPC proxy RESTful API server by
```toml
[api]

# Enable defines if the API server should be enabled.
enable = true

# Swagger defines if swagger documentation should automatically be registered.
swagger = true

# Address defines the API server to listen on.
address = "tcp://0.0.0.0:1317"
```

Additionally you can update the address to your need

## Running Query

A swagger file is provided along with this documentation. Also you can access to`127.0.0.1:1317/swagger` for the swagger UI.
