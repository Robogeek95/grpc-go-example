# gRPC

gRPC is a high-performance opensource tool for making remote procedure calls. It allows for communications between applications by exposing their functionalities.

gRPC is language-neutral, meaning it doesn’t require both ends of the communication process to be implemented in the same programming language. But it requires a contract on both ends for them to communicate with, otherwise, the procedures wouldn’t make any sense to one end. For the contract definition, gRPC uses the declarative language “protocol buffer” or “protobuf” for short.

Although gRPC is still young, it has made its way into several industries and has already been adopted by several enterprise companies like Netflix, Cisco and more. gRPC typical uses cases include

- Large scale microservice communication
- High performance and byte efficient data streaming
- Cloud-native services communication
- Bi-directional streaming capabilities for gaming
- Mobile device communication with servers using protocol buffers

In this article, you’ll learn how to communicate with a Golang server using the gRPC Golang library.

## What is gRPC

Rest is the default choice for API communication and it works pretty well, so why gRPC? Well, REST still has a few caveats:

- You are limited to the REST HTTP verbs, while these verbs might work for generic use cases, they might not scale well. RPC allows you to express more verbs in form of procedures.
- Rest is schemaless and this happens to be good and bad at the same time. You might not need to define schemas in a simple CRUD application like a to-do list application. But it makes to explicitly define the schema various interfaces of a larger application  that expects a larger payload so you don’t mix them up over time

gRPC uses ptotobuf and HTTP 2.0 to enable you to achieve fast, simple and idiomatic procedure calls. Protobuf is a tool for serializing structured data to be sent over the wire.

With official support for 10+ programming languages which includes Go, Javascript(Node.js), Kotlin, Python, C/C++, Scala, C# and more. You can have your client in JavaScript and the server in Golang and have them communicate in a byte efficient manner with gRPC. REST achieves the same but gRPC is protobuf over HTTP 2 as opposed to REST API’s JSON over HTTP 1.x.

For this article, we would build a server-side application with gRPC in Go. We would make use of the gRPC-Go library which is the official package provided and maintained by gRPC for the implementation of gRPC in the Go programming language

### Implementing gRPC

In this tutorial, we would implement a basic 

### Prerequisites

- Go installed
- Familiarity with some Go concepts

### Setup

- Install protocol buffer compiler, protoc version 3. Head over to gRPC documentation and follow this guide to install the compiler for your operating system.
- Install the protocol compiler plugins for Go
- Update your path for the plugins

### Service definition

gRPC and many other RPC based protocols are based around the idea of a service definition, where you declaratively define the methods that can be called, the parameters they expect and their return types. gRPC relies on protocol buffer as the interface definition language for it's service definition, it's responsible for describing both service and the message payload structure.

gRPC supports the following four connectivity methods; 

- **Unary RPC** - This method of connectivity works in a similar fashion to the REST APIs. The client simply sends a request to the server and waits for a single response and closes the connection.
- **Server-side streaming** - The client sends a single request to the server and expects a stream of responses back. The connection is kept open until there are no more messages.
- **Client-side streaming** - Just as in server-side streaming connectivity methods, the connection is kept open but in this method, the client sends the stream of messages to the server and waits for the server to finish reading them before closing the connection.
- **Bidirectional streaming** - with the two previous streaming methods, one side is reading and the other is writing. Bidirectional streaming allows both ends to send a sequence of streams in a read-write fashion.

Create a file with a .proto extension and define the methods along with their Request and Response types in the service definition file:

```protobuf
syntax = "proto3";

option go_package = "example.com/grpc-go/welcome";

package welcome;

// The greeting service definition.
service WelcomeService {
  // Sends a greeting
  rpc SendWelcome (WelcomeRequest) returns (WelcomeResponse) {}
}

// The request message containing the user's name.
message WelcomeRequest {
  string name = 1;
}

// The response message containing the greetings
message WelcomeResponse {
  string message = 1;
}
```

The syntax keyword signifies the protobuf version which in this case is version 3. The definition defines `WelcomeService` with the service keyword and its functionality `SendWelcome` using the `rpc` keyword. The service functionality accepts a request parameter and returns the response type both defined in the file.

### Generating client and server code

Using the gRPC protocol compiler plugin, you can generate the client and server-side code for up to 10 programing languages after you have described your services in the service definition file using protocol buffers.

Open your terminal and run the following command to generate the client and server interfaces from the service definition file you created earlier using the protoc compiler

```bash
protoc --go_out=. --go_opt=paths=source_relative \
    --go-grpc_out=. --go-grpc_opt=paths=source_relative welcome.proto
```

This command specifies to provide the output in Go, this uses the Go plugin you installed earlier.

You should have the following files generated for you:

- **.pb.go - contains the protobuf codes for populating, serializing and retrieving the request/response message types.
- *_grpc.pb.go - Contains the client stub for clients to call and interface type for servers to implement with the methods defined in the service definition.

### Creating the Server

Now you create a server responsible for interacting with your gRPC implementation. The server would accept a request from a client and return the response from the service.

Create a folder “server” and a new file “main.go” so the file is available at “server/main.go” and add the following code to create a server 

```go
package main

import (
	"context"
	"flag"
	"fmt"
	"log"
	"net"

	pb "example.com/grpc-go"
	"google.golang.org/grpc"
)

var (
	port = flag.Int("port", 50051, "The server port")
)

func main() {
	flag.Parse()
	lis, err := net.Listen("tcp", fmt.Sprintf(":%d", *port))
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}

	s := grpc.NewServer()
	pb.RegisterWelcomeServiceServer(s, &server{})

	log.Printf("server listening at %v", lis.Addr())
	if err := s.Serve(lis); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}
}
```

You need to first import the required packages including the protocol service handlers and the server struct as pb, this allows the gRPC services to be available in the file as a method of pb.

With the code above, the server is able to listen for gRPC connections on port 5051 but we need still need to define the method to implement the service

Add the following code after the “var” definition and before the “func main” declaration:

```go
type server struct {
	pb.UnimplementedWelcomeServiceServer
}

// SayHello implements helloworld.GreeterServer
func (s *server) SendWelcome(ctx context.Context, in *pb.WelcomeRequest) (*pb.WelcomeResponse, error) {
	log.Printf("Received: %v", in.GetName())
	return &pb.WelcomeResponse{Message: "Welcome onboard " + in.GetName()}, nil
}
```

Start the server by running the following command:

```bash
go run main.go
```

### Creating the client

Now that you have generated the gRPC protocols and we have a server waiting to accept connections the next step is to create a stub that sends requests to the server.

The concept of a stub in gRPC is the same as clients in other applications. gRPC is language-neutral so it doesn't matter the language you have your client and server in as long as they are both supported and are to make sense of the procedure calls.

Create a new file at `client/main.go`

```bash
package main

import (
	"context"
	"flag"
	"log"
	"time"

	pb "example.com/grpc-go"
	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
)

const (
	defaultName = "User"
)

var (
	addr = flag.String("addr", "localhost:50051", "the address to connect to")
	name = flag.String("name", defaultName, "Name to greet")
)

func main() {
	flag.Parse()
	// Set up a connection to the server.
	conn, err := grpc.Dial(*addr, grpc.WithTransportCredentials(insecure.NewCredentials()))
	if err != nil {
		log.Fatalf("did not connect: %v", err)
	}
	defer conn.Close()
	c := pb.NewWelcomeServiceClient(conn)

	// Contact the server and print out its response.
	ctx, cancel := context.WithTimeout(context.Background(), time.Second)
	defer cancel()
	r, err := c.SendWelcome(ctx, &pb.WelcomeRequest{Name: *name})
	if err != nil {
		log.Fatalf("could not greet: %v", err)
	}
	log.Printf("Greeting: %s", r.GetMessage())
}
```

Open another terminal and run the following command to start the client application:

```bash
go run main.go
Welcome onboard User
```

This runs the  the response “Welcome onboard User”. You can customize the message by providing the command with a name option

```bash
go run main.go --name=Lukman
```

<aside>
💡 Ensure you have both the client and the server app running

</aside>

## Conclusion

You have learnt how to define a protobuf service and use it to generate both client and server code using the protoc compiler. The resulting code uses the Go gRPC compiler to implement a simple client and server for your service.

Speedscale, a traffic replay framework for API testing in Kubernetes. Prepare APIs for real-world scenarios with stress testing, auto-generated test, traffic replay, and more.