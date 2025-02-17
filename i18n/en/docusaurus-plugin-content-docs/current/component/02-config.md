---
id: config
title: Config
description: Kratos configuration source can be specified multiple, and config will be merged into map[string]interface{}, and then get the value content through Scan or Value.
keywords:
  - Go 
  - Kratos
  - Toolkit
  - Framework
  - Microservices
  - Protobuf
  - gRPC
  - HTTP
---
In the Kratos project, multiple configuration sources can be specified, and config will be merged into key/value. Then the user obtains the corresponding key-value content through `Scan` or `Value`, the main features are:

- The local file data source is implemented by default
- Users can customize the data source implementation
- Supports configuration hot-reloading, and change the existing `Value` through `Atomic`
- Supports custom data-source decoding implementation
- Ability to get the value of environment variables or existing fields through the `$` placeholder

### Define configuration through proto

In the Kratos project, we recommend using proto to define the configuration file by default. The main benefits are:

- A unified template configuration can be defined
- Add corresponding configuration verification
- Better management of configuration
- Cross-language support

#### Configuration file

```yaml
server:
  http:
    addr: 0.0.0.0:8000
    timeout: 1s
  grpc:
    addr: 0.0.0.0:9000
    timeout: 1s
data:
  database:
    driver: mysql
    source: root:root@tcp(127.0.0.1:3306)/test
  redis:
    addr: 127.0.0.1:6379
    read_timeout: 0.2s
    write_timeout: 0.2s
```

#### Proto declaration

```protobuf
syntax = "proto3";
package kratos.api;

option go_package = "github.com/go-kratos/kratos-layout/internal/conf;conf";

import "google/protobuf/duration.proto";

message Bootstrap {
  Server server = 1;
  Data data = 2;
}

message Server {
  message HTTP {
    string network = 1;
    string addr = 2;
    google.protobuf.Duration timeout = 3;
  }
  message GRPC {
    string network = 1;
    string addr = 2;
    google.protobuf.Duration timeout = 3;
  }
  HTTP http = 1;
  GRPC grpc = 2;
}

message Data {
  message Database {
    string driver = 1;
    string source = 2;
  }
  message Redis {
    string network = 1;
    string addr = 2;
    google.protobuf.Duration read_timeout = 3;
    google.protobuf.Duration write_timeout = 4;
  }
  Database database = 1;
  Redis redis = 2;
}
```

### Build configuration

Head to the project's root and execute the command below:

```shell
make config
```

### Usage

One or more config sources can be applied.

They will be merged into `map[string]interface{}`, then you could use `Scan` or `Value` to get the values.

- file
- env

```go
c := config.New(
    config.WithSource(
        file.NewSource(path),
    ),
    config.WithDecoder(func(kv *config.KeyValue, v map[string]interface{}) error {
        // kv.Key
        // kv.Value
        // kv.Metadata
        // Configuration center can use the metadata to determine the type of the config.
        return yaml.Unmarshal(kv.Value, v)
    }),
    config.WithResolver(func(map[string]interface{}) error {
        // The default resolver provides processing
        // for two $(key:default) and $key placeholders.
        //
        // Customize the processing method after loading the configuration data...
    })
)
// load config source
if err := c.Load(); err != nil {
    log.Fatal(err)
}
// Get the corresponding value
name, err := c.Value("service").String()

/*
  // The structure generated by the proto file can also be directly declared for analysis
  var v struct {
      Service string `json:"service"`
      Version string `json:"version"`
  }
*/
var bc conf.Bootstrap
if err := c.Scan(&bc); err != nil {
	log.Fatal(err)
}
// watch the changing of the value
c.Watch("service.name", func(key string, value config.Value) {
    // callback of this event
})
```

Kratos can read the value of **environment variable** or **existing field** through placeholders in the configuration file

```yaml
service:
  name: "kratos_app"
http:
  server:
    # Use the value of service.name
    name: "${service.name}"
    # Replace with the environment variable PORT, if it does not exist, use the default value 8080
    port: "${PORT:8080}"
    # Use environment variable TIMEOUT to replace, no default value
    timeout: "$TIMEOUT"
```

When loading configuration sources from environment variables **no need to load in advance**, the analysis of the environment configuration will be performed after all sources are loaded

```go
c := config.New(
    config.WithSource(
        // Add environment variables prefixed with KRATOS_
        env.NewSource("KRATOS_"),
        // Add configuration file
        file.NewSource(path),
    ))
    
// 加载配置源:
if err := c.Load(); err != nil {
    log.Fatal(err)
}

// Get the value of the environment variable KRATOS_PORT
port, err := c.Value("PORT").String()
```
