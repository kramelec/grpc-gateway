# grpc-gateway

grpc-gateway is a plugin of [protoc](http://github.com/google/protobuf).
It reads [gRPC](http://github.com/grpc/grpc-common) service definition,
and generates a reverse-proxy server which translates a RESTful JSON API into gRPC.

It helps you to provide your APIs in both gRPC and RESTful style at the same time.

## Background
gRPC is great -- it generates API clients and server stubs in many programming languages,
it is fast, easy-to-use, bandwidth-efficient and its design is combat-proven by Google.
However, you might still want to provide classic RESTful APIs too for some reasons --
compatiblity to languages not supported by gRPC, API backward-compatibility or aesthetics
of RESTful architecture.

That's what grpc-gateway helps you to do. You just need to implement your gRPC service with a small amount of custom options.
Then the reverse-proxy generated by grpc-gateway serves RESTful API on top of the gRPC service.

## Installation
First you need to install ProtocolBuffers 3.0 or later.

```sh
mkdir tmp
cd tmp
git clone https://github.com/google/protobuf
./autogen.sh
./configure
make
make check
sudo make install
```

Then, `go get` as usual.

```sh
go get github.com/gengo/grpc-gateway/protoc-gen-grpc-gateway
go get github.com/golang/protobuf/protoc-gen-go
```
 
## Usage
Make sure that your `$GOPATH/bin` is in your `$PATH`.

1. Define your service in gRPC
   
   your_service.proto:
   ```protobuf
   syntax = "proto3";
   package example;
   message StringMessage {
     string value = 1;
   }
   
   service YourService {
     rpc Echo(StringMessage) returns (StringMessage) {}
   }
   ```
2. Add a custom option to the .proto file
   
   your_service.proto:
   ```protobuf
   syntax = "proto3";
   package example;
   
   import "github.com/gengo/grpc-gateway/options/options.proto";
   
   message StringMessage {
     string value = 1;
   }
   
   service YourService {
     rpc Echo(StringMessage) returns (StringMessage) {
       option (gengo.grpc.gateway.ApiMethodOptions.api_options) = {
         path: "/v1/example/echo"
         method: "POST"
       }
     }
   }
   ```
3. Generate gRPC stub
   
   ```sh
   protoc -I/usr/local/include -I. -I$GOPATH \
     --go_out=plugins=grpc:. \
     path/to/your_service.proto
   ```
   
   It will generate a stub file `path/to/your_service.pb.go`.
   Now you can implement your service on top of the stub.
4. Generate reverse-proxy
   
   ```sh
   protoc -I/usr/local/include -I. -I$GOPATH \
     --grpc-gateway_out=logtostderr=true:. \
     path/to/your_service.proto
   ```
   
   It will generate a reverse proxy `path/to/your_service.pb.gw.go`.
5. Write an entrypoint
   
   Now you need to write an entrypoint of the proxy server.
   ```go
   package main
   import (
     "flag"
     "net/http"
   
     "github.com/golang/glog"
     "github.com/zenazn/goji/web"
     "golang.org/x/net/context"
   	
     gw "path/to/your_service_package"
   )
   
   var (
     echoEndpoint = flag.String("echo_endpoint", "localhost:9090", "endpoint of YourService")
   )
   
   func run() error {
     ctx := context.Background()
     ctx, cancel := context.WithCancel(ctx)
     defer cancel()
   
     mux := web.New()
     err := gw.RegisterYourServiceHandlerFromEndpoint(ctx, mux, *echoEndpoint)
     if err != nil {
       return err
     }
   
     http.ListenAndServe(":8080", mux)
     return nil
   }
   
   func main() {
     flag.Parse()
     defer glog.Flush()
   
     if err := run(); err != nil {
       glog.Fatal(err)
     }
   }
   ```


## More Examples
More examples are available under `examples` directory.
* `echo_service.proto`, `a_bit_of_everything.proto`: service definition
  * `echo_service.pb.go`, `a_bit_of_everything.pb.go`: [generated] stub of the service
  * `echo_service.pb.gw.go`, `a_bit_of_everything.pb.gw.go`: [generated] reverse proxy for the service
* `server/main.go`: service implementation
* `main.go`: entrypoint of the generated reverse proxy

## Features
### Supported
* Generating JSON API handlers
* Method parameters in request body
* Method parameters in request path

### Want to support
But not yet.
* Integrated authentication
* Streaming APIs
* bytes and enum fields in path parameter
* Method parameters in query string
* Encoding request/response body in application/x-www-form-urlencoded
* Optinally generating the entrypoint
* Optionally emitting API definition for [Swagger](http://swagger.io)

### No plan to support
But patch is welcome.
* Method parameters in HTTP headers
* Handling header/trailer metadata
* Encoding request/response body in XML
* True bi-directional streaming. (Probably impossible?)

# License
grpc-gateway is licensed under the BSD 3-Clause License.
See `LICENSE.txt` for more details.
