# GRPC go

This is a very simple implementation of gRPC using go.

## Project initialization
```shell
mkdir grpc-go \
  && cd grpc-go \
  && go mod init github.com/oskiegarcia/grpc-go \
  && go mod tidy
```

## Create 3 sub directories
```shell
mkdir client helloworld server
```

## In helloworld directory, create protobuf definition
```protobuf
syntax = "proto3";

option go_package = "github.com/oskiegarcia/grpc-go/helloworld";

package helloworld;

service HelloWorldService {
  rpc SayHello(HelloWorldRequest) returns (HelloWorldResponse) {}
}

message HelloWorldRequest {}

message HelloWorldResponse {
  string message = 1;
}
```

## Generate protocol buffer code (helloworld.pb.go) and stub (helloworld_grpc.pb.go)
```shell
cd helloworld
```

```shell
protoc --go_out=. --go_opt=paths=source_relative \
    --go-grpc_out=. --go-grpc_opt=paths=source_relative \
    helloworld.proto
```

## tidy to download dependencies
```shell
cd ..
go mod tidy
```

## Implement the gRPC server
```go
package main

import (
	"context"
	"fmt"
	pb "github.com/oskiegarcia/grpc-go/helloworld"
	"google.golang.org/grpc"
	"log"
	"net"
)

type server struct {
	pb.UnimplementedHelloWorldServiceServer
}

func (s *server) SayHello(ctx context.Context, in *pb.HelloWorldRequest) (*pb.HelloWorldResponse, error) {
	return &pb.HelloWorldResponse{Message: fmt.Sprintf("Hello, %s!", in.Name)}, nil
}

func main() {
	lis, err := net.Listen("tcp", ":50051")
	if err != nil {
		log.Fatalf("failed to listen on port 50051: %v", err)
	}

	s := grpc.NewServer()
	pb.RegisterHelloWorldServiceServer(s, &server{})
	log.Printf("gRPC server listening at %v", lis.Addr())
	if err := s.Serve(lis); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}
}

```

## Implement the gRPC client
```go
package main

import (
	"context"
	pb "github.com/oskiegarcia/grpc-go/helloworld"
	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
	"log"
	"time"
)

func main() {
	conn, err := grpc.Dial("localhost:50051", grpc.WithTransportCredentials(insecure.NewCredentials()))
	if err != nil {
		log.Fatalf("failed to connect to gRPC server at localhost:50051: %v", err)
	}
	defer conn.Close()
	c := pb.NewHelloWorldServiceClient(conn)

	ctx, cancel := context.WithTimeout(context.Background(), time.Second)
	defer cancel()

	r, err := c.SayHello(ctx, &pb.HelloWorldRequest{Name: "Oscar"})
	if err != nil {
		log.Fatalf("error calling function SayHello: %v", err)
	}

	log.Printf("Response from gRPC server's SayHello function: %s", r.GetMessage())
}

```


## In the project root directory, run server
```shell
go run server/main.go
```

## In a new terminal window, run client
```shell
go run client/main.go
```

It should display the following output in the console.
```shell
2024/04/10 20:18:21 Response from gRPC server's SayHello function: Hello, Oscar!
```
