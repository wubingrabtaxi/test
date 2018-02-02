Thank you for choosing [Grab-Kit](https://wiki.grab.com/display/TECH/Grab-Kit) to kickstart your new service! Read on to learn more about what's just been created.

You can find the most recent version of this guide [here](https://gitlab.myteksi.net/gophers/go/blob/master/commons/util/grab-kit/templates/readme.md).

## Table of Contents

- [Updating Grab-Kit](#updating-grabkit)
- [Getting Started](#getting-started)
- [Directory Structure](#directory-structure)
- [Starting the Service](#starting-the-service)
- [The Service object](#the-service-object)
- [DTO Generation](#dto-generation)
- [Reading Headers](#reading-headers-(metadata))
- [Adding a New Endpoint](#adding-a-new-endpoint)
- [Using the gRPC client package](#using-the-grpc-client-package)
- [Regenerating the Code](#regenerating-the-code)
- [Adding a Database](#adding-a-database)
- [Returning Errors](#returning-errors)
- [Customizing the Middleware Stack](#customizing-the-middleware-stack)
- [HTTP / RESTful routing](#http-/-restful-routing)
- [Features](#features)
- [Profiling](#profiling)
- [Caching](#caching)
- [Support](#support)

## Updating GrabKit

The `grab-kit` script located in `scripts/` will always refer to the latest version in your checkout. Simply merge your checkout with master to make sure you're on the latest version.

## Getting Started

### Writing a service

#### Create service proto

The primary requirement of Grab-Kit is a service definition, defined using Protobuf in `pb/service.proto`. The first time you run `grab-kit create MyService`, a 'hello world' endpoint will be created for you. It defines `HelloRequest` and `HelloResponse`, each with one field.

```
service MyService {
  rpc Hello(HelloRequest) returns (HelloResponse) {
    option (google.api.http) = {
      post: "/hello"
      body: "*"
    };
  }
}

message HelloRequest {
  string Name = 1;
}

message HelloResponse {
  string Message = 1;
}
```

#### Create the service

`grab-kit create` creates the basic server code in `server/serve.go` from scaffold:

```
// Serve starts the server
func Serve() {
	// Load app config
	appCfg := config.Load()

	// Create service implementation
	service := handlers.NewGktestServiceImpl()

	// Initialize Grab dependencies
	grabConf, _ := gkconfig.FromStruct(appCfg)

	// Create the Grab-Kit Service
	s := grabkit.NewService(
		grabkit.GrabConfig(grabConf),
		grabkit.Name("gktest"),
		grabkit.ServiceDiscovery(enableServiceDiscovery()),
	)

	// Register the app handlers
	RegisterHandlers(s.Server(), service)

	// Start the service
	_ = s.Start(context.Background())
}
```

This code:

* Loads your application's config from `config-ci.json`
* Creates an instance of your API (from `server/handlers`)
* Initializes Grab dependencies from your application's config (eg. sitevar, statsd, logging)
* Constructs a `grabkit.Service`
* Registers your handlers using the generated code
* Starts the server.

#### Create the handlers

In the above code, `NewGktestServiceImpl()` should return an object that implements your app's `Service` as defined in `services/myapp/z_service.go`. The default implementation of this service is in `server/handlers`, along with function stubs. You can either implement these stubs, or create your own object that implements the same API.


#### Run the server

Run:

```
$ grab-kit start
```

Output:

```
grab-kit start

 _______  ______    _______  _______         ___   _  ___   _______    _______  _______
|       ||    _ |  |   _   ||  _    |       |   | | ||   | |       |  |       ||       |
|    ___||   | ||  |  |_|  || |_|   | ____  |   |_| ||   | |_     _|  |    ___||   _   |
|   | __ |   |_||_ |       ||       ||____| |      _||   |   |   |    |   | __ |  | |  |
|   ||  ||    __  ||       ||  _   |        |     |_ |   |   |   |    |   ||  ||  |_|  |
|   |_| ||   |  | ||   _   || |_|   |       |    _  ||   |   |   |    |   |_| ||       |
|_______||___|  |_||__| |__||_______|       |___| |_||___|   |___|    |_______||_______|

•  Running go build in ./cmd/server
•  Running ./cmd/server/server
17:17  grabkit.server.grpc: started : address=:8087
17:17  grabkit.server.http: started : address=0.0.0.0:8088
17:17 middleware.profiling: Serving flamegraphs at /debug/profiling : binary=/Users/mike/go/src/gitlab.myteksi.net/gophers/go/gktest/cmd/server/server
2017-12-15 17:17:13.439871730 +0800 SGT : INFO : simplehttpserver : Starting simple HTTP server on 0.0.0.0:8089
17:17       grabkit.server: Debug webserver started. Open http://localhost:8089/ to access. : address=0.0.0.0:8089
```

#### Create the client

An example client is included in `examples/sampleclient.go`

```
func main() {
	pool := pool.NewV1WithOptions(context.Background(),
		"someservice"
		pool.WithServerAddress("localhost:8087"),
		pool.WithServiceDiscoveryEnabled(false),
	)

	// Create grabkit client
	s := grabkit.NewService(
		grabkit.Client(
			grpc.NewClient(
				gkclient.WithGRPCPool(pool),
			),
		),
	)

	// Create service using grabkit client
	circuitConfig := map[string]hystrix.CommandConfig{}
	client := client.NewGktestClient("gktest", circuitConfig, s.Client())
	res, err := client.Hello(ctx, "anonymous")
}
```

## Directory Structure

Your project structure should look like this:

```
./bin
./client
./cmd
./cmd/server
./dto
./examples
./pb
./server
./server/config
./server/handlers
./services
./services/my-app
./storage

```

### ./bin

This contains scripts used internally by Grab-Kit. You should not commit this directory to version control

### ./client

This is the code to be used by clients to call your service. By default the client will use gRPC, unless your service is
generated without gRPC (via `--http-only`)

### ./cmd/server

This is the entry point to your application, where the main binary is built.

### ./dto

These are your DTO types, and the bindings to convert them to/from protobuf types. By default Grab-Kit will generate both the DTO types and the bindings in `z_dto.go` and `z_grpc_bindings.go`.

If you want to override the generated code, simply define types and/or functions with the same names in the `dto` package (but **not** in the z_ files), and Grab-Kit will no longer generate them the next time it is run. We recommend using `dto.go` and `grpc_bindings.go` for this purpose.

### ./examples

This contains example code and scripts to demonstrate the newly created service.

### ./pb

This is where your main service definition lives, typically in `service-name.proto`. This file is important: it is the source of truth used by Grab-Kit to generate all the other code.

The compiled Go version of the proto file is also built here.

### ./server

This directory contains the main server code in the `serve*.go` files. You are free to edit these files, as they are only generated once on startup and do not change afterwards.

The `z_myapp_endpoints.go` file contains the generated endpoint metadata, used for abstracting the transport (HTTP or GRPC) from the endpoint implementation.

### ./server/config

This package just contains your service's configuration variables and some basic initialization. Again, you are free to edit this package.

### ./server/handlers

Ah, finally! This directory contains your actual service implementation. You'll need to write code here to implement your service.

### ./services/my-app

This directory contains the bulk of the generated code, and provides the 'glue' between your endpoints and the server or client. You won't need to edit this code, but feel free to browse to see how Grab-Kit works under the hood.

### ./storage

This contains data entities that represent tables in a backend database, and functions for interacting with them.

### Updating the scaffolded code

The 'create' command generates some initial scaffold for your app, including the `server/serve.go` file which is used to
start the server. By default, this file is small and starts a server with some standard settings. We recommend you keep
this file as it is, so that you can take advantage of any updates to the Grab-Kit server without modifying your server
code.

However, you might like to customize the HTTP or GRPC server or perform additional intialization. You can do this by
directly editing this file; it is only generated once and never updated afterwards.

In particular, you can hook into the initialization of each server by specifying the `OnInit` callback options to the Server, eg:


```
func FooHandler(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte(`foo`))
}

func main() {
	svc := grabkit.NewService(
		grabkit.Server(
			gkserver.NewServer(gkserver.WithHTTPInit(func(ctx context.Context, _ *http.Server, r *mux.Router) error {
				r.HandleFunc("/foo", FooHandler)
				return nil
			})),
		),
	)
	svc.Start(context.Background())
}

```

## Starting the Service

### Setting the config environment variable

`grab-kit create` will modify `scripts/env-vars.sh` to add an environment variable `MYAPP_CONF` which points to your configuration file. Source this file using `source scripts/env-vars.sh` before starting your service.

### Start the service

To start your service, run `grab-kit start` from the main directory. Your service should start on port 8088.

You can test it out by calling `curl http://localhost:8088/myapp/hello --data '{"name": "Anonymous"}'`.

## The Service object

The core object in Grab-Kit is the `Service`, created with `grabkit.NewService()`. Calling this by default creates a service with a default Server and Client:

```
  service := grabkit.NewService()
  service.Start(ctx)
```

You can pass options at any time using either parameters with `NewService` or calling `Init(options)`, which can be called multiple times:

```
  service := grabkit.NewService(grabkit.WithName("my service"))
  service.Init(grabkit.WithName("my new name"))
```

The same pattern can be used with the Server and Client:

```
  service := grabkit.NewService(
  	grabkit.Server(
  	  gkserver.NewServer(
  	    gkserver.WithHTTPAddr("0.0.0.0:8888"),
  	  ),
  	),
  )
  
  // or
  
  service.Server().Init(gkserver.WithHTTPAddr("0.0.0.0:8888"))
 
  ```

### Access to logger and other handlers

The `grabkit.Service`  can be retrieved at any time in your handlers with:

```
  func myHandler(ctx context.Context) {
    grabkit := grabkit.FromContext(ctx)
  }
```

This can be used to access many utility objects such as the logger and stats:

```
  func myHandler(ctx context.Context) {
    gk := grabkit.FromContext(ctx)
    logger := gk.Logger()
    statsClient := gk.StatsClient()
    cfg := gk.ConfigHandler()
  }
```  

## DTO Generation

Grab-Kit will automatically generate DTOs for custom message types in the .proto file. It will also generate the protobuf bindings for these types, so they can be converted between the Go DTO and protobuf types.

### Types Supported

* All basic types (string, int etc.)
* Arrays
* Maps (converted to custom type)
* Messages (converted to structs)
* `google.protobuf.Timestamp` (converted to time.Time)
* `grab.kit.RawJSON` (maps to `json.RawMessage`, internally marshals to a message with a single string field for
  protobuf)
* `google.protobuf.Struct` (converted to gktypes.Struct, maps to JSON object for HTTP)
* Enums (converted to string constants)

### Unsupported Types

* `google.protobuf.Any`
* `google.protobuf.OneOf`

### Custom types

If you want to use an unsupported type, or you want to handle the DTO conversion yourself, it is easy to override the Grab-Kit generated code:

* Create a type with the same name in `dto.go` (not `z_dto.go`)
* Create the binding code in `grpc_bindings.go`. The simplest way is probably to copy the functions from `z_grpc_bindings.go` and modify them to suit your needs. The functions are named `<Type>ToPB` and `<Type>FromPB`.

## Reading headers (metadata)

Grab-Kit saves HTTP headers and gRPC metadata as-is inside the context. To retrieve it:

```
# Metadata is a map[string][]string
md := metadata.FromIncomingContext(ctx)
```

Because HTTP headers and gRPC metadata have different naming conventions, Grab-Kit provides a standard mapping for common keys such as `Authorization`. To access these keys, use the constants in `middleware`:

```
	md := metadata.FromIncomingContext(ctx)
	auth := md[metadata.KeyRequestAuthorization]
```

### Sending request from gRPC clients

This is explained in detail in [the official docs](https://github.com/grpc/grpc-go/blob/master/Documentation/grpc-metadata.md).

```
  md := metadata.Pairs("authorization", "some-auth-key")
  ctx := metadata.NewOutgoingContext(context.Background(), md)
```

Note you should use the GRPC and HTTP header names in this case, not the Grab-Kit helpers.

### Sending response metadata from servers

To do this, you must implement the `metadata.Getter` interface and set it in `grabkit.Config`, eg:

```
type mdGetter struct{}

func (m *mdGetter) Get(key string, _ metadata.Protocol) metadata.Metadata {
	return metadata.Metadata{
		"X-Foo": []string{"Bar"},
	}
}


func Serve() {
  // ...
  grabkit.Config.MDGetter = &mdGetter{}
}
```

### Setting ad-hoc metadata / request headers

In your handler, call `grabkit.AddHeader(ctx, key, val)` eg:

```
	grabkit.AddHeader(ctx, "X-My-Header", "10")

```


### Consistent naming of headers

Grab-Kit will automatically translate known response headers into their equivalents for HTTP or GRPC if you use the `metadata.Key*` constants. Otherwise, the headers will remain unchanged.

For incoming request headers, the metadata will be populated both with the original header name and the Grab-Kit equivalent name (if any), so you can choose which key to access. This is for compatibility with other libraries that might expect the standard HTTP/gRPC headers.

## Adding a New Endpoint

To add a new endpoint, start with modifying the proto file. For example, let's say I want to add an API `SortPeople`,
which will sort people according to their surname. I'll first want to extend the `service` definition in my proto file:

```
service MikeTest {
  rpc Hello(HelloRequest) returns (HelloResponse);

  // This was added.
  rpc SortPeople(SortPeopleRequest) returns (SortPeopleResponse);
}
```

Note the request and response types, by convention, should be defined as a single message each. I'm also going to need to define a new type here, since `person` isn't a built-in data type. I do that in the proto file:

```
message SortPeopleRequest {
  repeated Person People = 1;
  int64 Limit = 2;
}

message SortPeopleResponse {
  repeated Person People = 1;
}

message Person {
  string FirstName = 1;
  string LastName = 2;
}
```

This is standard protobuf stuff, which will become familiar the more you do it.

Now I want to start implementing this new API. But first, I run `go generate` to create the DTOs from the .proto file.
Let's look at the generated code:

```
// dto/z_dto.go
// NOTE: THIS IS GENERATED. You don't need to edit this file.
package dto

// Person ...
type Person struct {
	FirstName string

	LastName string
}
```

Now that's done, I can get on with implementing the SortPeople endpoint. GrabKit has created a stub for me in
`server/handlers/z_miketest.go` which looks like this:

```
// SortPeople implements Service.
func (s *miketestServiceImpl) SortPeople(ctx context.Context, people []*dto.Person, limit int64) ([]*dto.Person, error) {
	return sortPeople(ctx, people, limit)
}
```

Note how there's no HTTP or GRPC code to worry about here; I just need to write `sortPeople`. Neat! I'll create `server/handlers/sortpeople.go`. It looks like this:

```
// ByLastName implements sort.Interface for []Person based on
// the LastName field.
type ByLastName []*dto.Person

func (a ByLastName) Len() int           { return len(a) }
func (a ByLastName) Swap(i, j int)      { a[i], a[j] = a[j], a[i] }
func (a ByLastName) Less(i, j int) bool { return a[i].LastName < a[j].LastName }

func sortPeople(_ context.Context, people []*dto.Person, limit int64) ([]*dto.Person, error) {
	sort.Sort(ByLastName(people))
	sortedPeople := make([]*dto.Person, limit)
	for i := int64(0); i < limit; i++ {
		sortedPeople[i] = people[i]
	}
	return sortedPeople, nil
}
```

That's it. No need to run `go generate` again now, I just start the server with `go run cmd/server/main.go` and give it a try:

```
$ curl http://localhost:8088/miketest/sortPeople --data '{"People":[{"FirstName":"John", "LastName":"Smith"}, {"FirstName":"Bob", "LastName":"Archer"}, {"FirstName":"Sam", "LastName":"Davis"}], "Limit":2}'

{"people":[{"FirstName":"Bob","LastName":"Archer"},{"FirstName":"Sam","LastName":"Davis"}]}
```

...and that seems to work! Well, the HTTP part at least. Testing the GRPC client is left as an exercise for the reader :)

## Using the gRPC client package

The `client` package is designed to be used directly by clients of your service. You first need to configure a grabrpc
client Pool object before calling client.NewABCService - see ./examples/sampleclient for an example.

Note that service discovery is enabled by default and will not work locally, so you must explicitly disable it in the
client code.

For more information about service discovery, see [the grabrpc
documentation](https://gitlab.myteksi.net/gophers/go/tree/master/commons/util/grabrpc/csdp).


## Regenerating the Code

Simply run `go generate` from the same directory as this file. This magic works by the comment in `doc.go`, and will run
`grab-kit gen` with the settings you specified when you created the app.


## Adding a Database

Use the `grab-kit entity` command:

```
$ grab-kit entity Person id name:string age:int
```

This will create an initial `storage/person.go` and `storage/z_person.go`. The `person.go` is the initial schema and
should be modified when you want to add new columns or custom queries.

There are two helper functions which can be customized:

### Wrapping database errors

To wrap DB errors, call the `storagesettings.SetErrorCallback` function, preferably in `init` in a `storage/storage.go`
file.

To customize circuitbreaker please look at [AutoConfigure Circuitbreaker](#auto-configure-circuitbreaker)

OR

To customize the circuitbreaker settings, call `storagesettings.SetCBSettingCallback`:

```
// in storage/storage.go

func init() {
  // log any errors
  storagesettings.SetErrorCallback(func (op, entity string, err error) error {
    if err == nil {
      return nil
    }

    logging.Info("Got db error: %v")
    return err
  })

  storagesettings.SetCBSettingCallback(func(strategy, tableName string) circuitbreaker.SettingHandler {
    return &circuitbreaker.Setting{
      Timeout: 3000,
      TagStr: fmt.Sprintf("storage-%s-%s", strategy, tableName),
    }
  })
}
```

If you modify the schema file, simply run `go generate` from the storage directory to regenerate the code.

See https://wiki.grab.com/display/TECH/Data for general information about Grab's data layer.

## Returning Errors

Grab-Kit provides an [errors package](https://gitlab.myteksi.net/gophers/go/tree/master/commons/util/grab-kit/errors) which can be used to return standard errors, which map to the correct HTTP and gRPC status codes. See the available [helper functions](https://gitlab.myteksi.net/gophers/go/blob/master/commons/util/grab-kit/errors/const.go).

For example:

```
  return errors.InvalidArgument("missing 'name' parameter")
```

If you return a different kind of error, Grab-Kit will wrap it using `errors.Internal()`, and assume it is an internal server error (ie. HTTP 500).

### Rendering custom errors

If for some reason you don't want the standard error format to be returned by the API (eg. for legacy clients), you can do so by passing the custom error payload to the `Wrap()` function, eg:

```
  return errors.InvalidArgument("missing 'name'").Wrap(myErr)
```

This will cause `myErr` to be rendered from the API instead of the Grab-Kit error format.

**Use with caution!** Do not simply wrap every internal error here, since it could create a security risk by exposing internal error messages to the end-user. Instead, only use this when you have defined a custom error format and verified that it 

### Best practices

There are two ways to return errors to your clients: either return an error directly from the endpoint, or by including an 'error' field in your protobuf response. Both methods work.

However, we recommend that you use an error field in your response struct wherever possible. This will ensure the best compatibility across HTTP and gRPC and give you full control over how the error can be handled. This approach can be considered a 'soft' error.

Returning an error directly from the endpoint will expose the error to processing by Grab-Kit middleware and clients. This means it could have potential side-effects on things like circuit breakers and error handling, so take this into account.

As a guide, you should use 'soft' errors if:

* You want to return partial errors
* You want to use an error as a 'status', eg. for multi-step or asynchronous operations
* You want to return multiple errors, or use a complex error response format
* You have complex rules for handling errors, and your clients require full control

You should return errors directly if:

* The error is simple and has a standard meaning (ie. you can use the built-in error helpers like `InvalidArgument` or `NotFound`). These are not likely to affect circuit breakers.
* The error is unexpected or 'fatal'
* The error is related to transport (eg. timeouts)
* The error can't / shouldn't be handled by clients

### Localizing error messages

If you want to provide a localized error message to your clients, you should do so by using `Wrap()` and use a custom error response format, or by embedding the error message in your response struct. Another option is localizing the message in the client.

### Developer error messages

If your service is user-facing, be sure not to include sensitive information in the error messages.

### Status codes

It is not possible to change the status codes returned by Grab-Kit besides using the built-in error codes. This is because Grab-Kit relies on the status codes for consistent error handling. If you need a new status code, please discuss with the Grab-Kit developers first.

The two rules will always remain true:

* A non-nil error will never return an OK status code
* An OK status code will always result in a nil error
  
## Customizing the Middleware Stack

Grab-Kit ships with a minimal middleware stack, with only logging, stats enabled in the default stack. In development
mode, panic recovery and profiling are also included in the default stack. Other middleware can be added on
demand in `server/serve.go` with the use of the `gkserver.WithMiddleware(...endpoint.Middleware)` option.

You may write your own custom middleware by using the following function signature:

```
type Middleware func(Meta, Endpoint) Endpoint
```

They are added from the outside (top) to inner (bottom of the stack) from start-to-end of the list, before the default
stack, so the first endpoint in the list will become the outermost middleware function, called first. For example, if
you call `gkserver.WithMiddleware(A, B, C)`, the middleware stack will look like:

```
PanicRecovery <-- outer
Logging
Stats
A
B
C <--
(handler)
```

That means:

* A will be called first
* C will be first to receive the response from the handler

## HTTP / RESTful routing

Grab-Kit allows you to customize the generated routes and use variables in URL paths by using annotations as in Google's
RPC specification: https://cloud.google.com/service-management/reference/rpc/google.api#httprule. By default, all routes
are `POST`.

For example:

```
service TodoList {
  rpc Hello(HelloRequest) returns (HelloResponse) {
      option (google.api.http).get = '/myhello';
  }
}
```

This would change the route from the default (`POST /hello`) to `GET /myhello`. For `GET` requests only, you can use
query parameters like `GET /myhello?name=Bob`. The parameters are mapped using the same `json` struct tags that are used
for POST requests.

To use URL vars, simply add the variable to the path enclosed in `{}`s, eg:

```
service TodoList {
  rpc Hello(HelloRequest) returns (HelloResponse) {
    option (google.api.http).get = '/hello/{Name}';
  }
}
```

Note this does not use the `json` struct tag. The name of the variable must match the message field in the .proto, and is important
for compatibility with other Protobuf tools.

The priority order is:

1. URL variables
2. Query parameters (only available for `GET`)
3. Request body (can be used for all methods, including `GET`)

That is, the request body will always be used if present.

## Features

### Distributed tracing support

See [the wiki page](https://wiki.grab.com/display/SRE/Introduction+to+Grab%27s+Distributed+Tracing+System) for a great
introduction to tracing at Grab.

Grab-Kit supports tracing by default for both the HTTP and the GRPC server via the
[commons/util/tracer](https://gitlab.myteksi.net/gophers/go/tree/master/commons/util/tracer) library. To enable, make sure the
`tracing` feature is enabled in the Grab-Kit section of the config:

```
  "grabkit": {
    "features": ["tracing"]
  }
```

You don't need to do anything else for the basic tracing of endpoints. However, to make the most of this functionality you will want to propagate
the context within your own application. To do this, you should read the existing tracing information from the Context
and create a child span, using the helper function:

```
    span, childCtx := tracer.CreateSpanFromContext(ctx, "httpHandler")
    defer span.Finish()

    otherFunction(childCtx)
```

Tracing can even be propagated to other HTTP services - see the above wiki page for how to do this.

### Configurator

```
  "grabkit": {
    "features": ["ucm"]
  }
```

Grab-kit understands that you need to get configuration for your application to run. Within Grab there are multiple configuration system.
Grab-kit provides a simple and uniform interface which can work with any of the available configurator (ucm/sitevar).
To get the handle to the configurator from you app, after initializing grabkit call `grabkit.Handler.ConfigHandler()`.
The config handler is already initialized for you with correct parameters. If you want to still use the native functions
provided by the (ucm/sitevar) sdk you can do a type-assertion to get hold of the actual SDK `configHandler.(ucm.Client).On(...)`.

```
    gkConf := gkInit.Init(Config)

    configHandler := grabkit.Handler.ConfigHandler()
    configurableString := configHandler.GetString("my-config-key")
    configurableInt := configHandler.GetInt("my-int")
```

### Auto configure circuitbreaker
Once you have setup a configurator (either via sitevar2 or UCM). You may also want to setup auto configuration of hystrix.
How this works is, client needs to pass a config handler (configurer) which will listen to all the updates on `circuitbreaker` config key
When any content changes, it will flush hystrix with older configuration and initialize with the new values.
The object that needs to be stored in ucm/flagsview should adhere to the format of `circuitbreaker.Setting`
If all the properties are not defined, it will take the default values. Please refer 
`myteksi/go/commons/util/resilience/circuitbreaker/setting.go` for defaults.

```
    configHandler := grabkit.ConfigHandler()
    grabkit.AutoConfigureCircuitbreaker(configHandler)
    
    // Setting struct used for unmarshal
    type Setting struct {
    	TagStr              string `json:"tag"`
    	Timeout             int    `json:"timeoutInMs"`
    	MaxConcurrentReq    int    `json:"maxConcurrentReq"`
    	ErrPercentThreshold int    `json:"errorPercentThreshold"`
    	VolPercentThreshold int    `json:"volumePercentThreshold"`
    	SleepWindow         int    `json:"sleepWindowInMs"`
    	MaxQueueSize        int    `json:"maxQueueSize"`
    	CommandGroup        string `json:"commandGroup"`
    }
    
    // content in ucm/flagsview:
    key: "circuitbreaker"
    
    value:
     "daxApiCB": {
        "timeoutInMs": 100,
        "maxConcurrentReq": 10,
        "commandGroup": "api"
     }, "paxApiCB": {
        "timeoutInMs": 100,
        "maxConcurrentReq": 10,
        "commandGroup": "api"
     }, "chronosCB": {
        "timeoutInMs": 100,
        "maxConcurrentReq": 10,
        "commandGroup": "geo"
     } 
``` 

  

### Swagger documentation

When using the above specification to define HTTP endpoints, Grab-Kit will generate Swagger documentation for you in
`pb/<app>.swagger.json`. It can also be generated using the `grab-kit swagger` command.

By default, this will also be served at `/swagger.json` on port 8088 (eg. http://localhost:8088/swagger.json).

Note that in order to generate the Swagger documentation, you need to have the `protoc-gen-swagger` tool installed:

```
  go get -u github.com/grpc-ecosystem/grpc-gateway/protoc-gen-swagger
```

### Support for multiple services

Grab-Kit now supports multiple services in a single proto file, eg:

```
service Gktest {
  rpc Hello(HelloRequest) returns (HelloResponse);
}

service Foo {
  rpc Bar(BarRequest) returns (BarResponse);
}
```

Each service will generate its own set of files. Note that you would then need to register the handlers for each service separately:

```
	RegisterGktestHandlers(s.Server(), service)
	RegisterFooHandlers(s.Server(), foo)
```

The hierarchy in Grab-Kit is `project -> service -> endpoint`. In the circuitbreaker configuration and gRPC, they are namespaced like `project.service.endpoint`, eg: `myservice.greeter.hello`.

## Profiling

Grab-Kit enables a lot of profiling features by default. When the service is running, they are available at
http://localhost:8089 (the debug port). View that page to see instructions on the different kinds of profiling
available.

A full guide on profiling Grab-Kit applications is provided in a [separate
document](https://wiki.grab.com/display/TECH/Profiling+Grab-Kit+services), but here are some quick guidelines:

* To debug **goroutine leaks**, use the goroutine profiler ("All goroutines"). The numbers show you how many of each goroutine are running.
* To debug **memory leaks**, use the memory/heap profiler, best viewed in SVG or with pprof.
* To debug **code bottlenecks**, use the CPU profiler or the per-endpoint flamegraph, but note this works only for CPU-heavy workloads. It may be better to do this offline using specially crafted benchmark code instead.
* To debug **latency issues** caused by blocked goroutines, use the execution tracer to visualize your goroutines over time. This includes blocking network calls, syscalls, GC pauses etc. 
* To debug **poor parallelism**, the execution tracer can also be used, showing which goroutines are executed on each processor *over time*.

## Caching

Support for caching is integrated in both the client and server on an opt-in basis.

For a server endpoint to support caching, it must:

* Specify cache key field(s) to use in the .proto:

```
  rpc GetTodo(GetTodoRequest) returns (GetTodoResponse) {
    option (grab.kit.cache) = {
      keys: ["ID"]
    };
  }
```

  * **or** implement a `CacheKey` method on the response type
* Return a TTL in the header with eg. `grabkit.AddHeader(ctx, metadata.KeyCacheControl, "max-age=60")`



For the client, they must explicitly add the `cachemiddleware.Client` to the stack:

```
	s := grabkit.NewService(
	   grabkit.Client(
	     grpc.NewClient(
	       gkclient.WithMiddleware(cachemiddleware.Client),
	     ),
	   ),
	)
	client := client.NewHelloClient("hello", circuitConfig, s.Client())
```

By default, the cache used is a 100mb in-memory cache provided by `end/common/kv.NewGMemCache`, and the responses are gob-encoded. This can be overridden with options to `Client()`:

```
  cachemiddleware.Client(
    cachemiddleware.WithCache(cache kv.Cache),
    cachemiddleware.WithSerializer(s cachemiddleware.Serializer)
  )
```


# Support

Grab-Kit is fully supported by the commons (Spartan) team. To get help, feel free to drop in at #grab-kit, #spartan or just ask in #grab-gopher-family on Slack.
