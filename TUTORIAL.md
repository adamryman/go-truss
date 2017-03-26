# Tutorial: Getting Started with Truss

We will build a simple service based on [echo.proto](./_example/echo.proto)

# Installation tips

1. Follow instructions in the [README](./README.md)
  - Can use `brew install` to get protobuf, golang, but not other packages (`go get` the rest).
2. Go to truss installation folder and run `make test`
  If everything passes you’re good to go.
  If you see any complaints about packages not installed, `go get` those packages
  If you encounter any other issues - ask the developers
3. To update to newer version of truss, do `git pull`, or `go get -u github.com/TuneLab/go-truss/...` truss again.

# Writing your first service

Define the communication interface for your service in the *.proto file(s).
Let's start with [echo.proto](./_example/echo.proto) and read the helpful comments.

## What is in the *.proto file definitions?

The name of the generated service will be based on the package name. (Readability-wise, it is good practice to use the same name for the service definition.)
```
package echo;
...
service Echo{...}
```
The service API is defined through RPC calls and a set of corresponding request and response messages.
The fields in the message definition are enumerated - this is a requirement for protobuf to serialize the data.
```
message LouderRequest {
  string In = 1;          // In is the string to echo back
  int32  Loudness = 2;    // Loudness is the number of exclamations marks to add to the echoed string
}
```
The RPC calls can be annotated with HTTP transport option (endpoint name and type of request). For this we must import the google annotations library.

```
import "github.com/TuneLab/go-truss/deftree/googlethirdparty/annotations.proto";

service Echo {
...
  rpc Echo (EchoRequest) returns (EchoResponse) {
    option (google.api.http) = {
        // All message fields are query parameters of the http request unless otherwise specified
        get: "/echo"
      };
  }
...
}
```
Most of the time a `get` request is sufficient. If you wish to transmit parameters via the body, use `post`.
Currently, if your message contains fields of type `map` they must be transmitted in the body. You would annotate it similarly to the Louder function in our example.
```
  rpc Louder (LouderRequest) returns (EchoResponse) {
    option (google.api.http) = {
        post: "/louder/{Loudness}"    // Loudness is accepted in the http path
        body: "*"                     // All other fields (In) are located in the body of the http/json request
      };
  }
```

The service definition can be split across multiple proto files. However, it is a good practice to keep all the RPC definitions in the same file, to make sure there are no naming conflicts.

## Understanding generated file structures

In your terminal, go to the folder containing echo.proto and run `truss *.proto` command. This will generate the service folder (by default at the same level as the proto files). The command will succeed silently. Your directory will now look like this:

```
.
├── echo
│   ├── docs
│   │   └── docs.md
│   ├── generated
│   │   └── ...
│   ├── handlers
│   │   └── server
│   │       └── server_handler.go
│   ├── middlewares
│   │   └── ...
│   ├── echo-cli-client
│   │   └── client_main.go
│   ├── echo-server
│   │   └── server_main.go
│   └── echo.pb.go
├── echo.proto
```
From the top down, within echo/:
  - docs/ contains the generated documentation of the service API
  - generated/ contains the wiring and encoding protocols necessary for service communication
  - handlers/server/server_handler.go is populated with stubs where you will add the business logic
  - middlewares/ is where you can put the middlewares (NOP by default)
  - echo-cli-client/ contains the client side CLI (useful for testing)
  - echo-server/ contains the service main, which you will build and run shortly
  - echo.pb.go contains the RPC interface definitions and supporting structures that have been translated from echo.proto to golang

If you try to build and run your service now, it will respond with empty messages. There is no business logic yet! We shall add it in the next step.

You can safely modify only the files in handlers/ and middlewares/. Changes to any other files will be lost the next time you re-generate the service with truss.

## Implement business logic

Open `handlers/server/server_handler.go` using your favorite editor. Find the Echo function stub. It should look like this:
```
func (s echoService) Echo(ctx context.Context, in *pb.EchoRequest) (*pb.EchoResponse, error) {
	var resp pb.EchoResponse
	resp = pb.EchoResponse{
	// Out:
	}
	return &resp, nil
}
```
Notice that the stub has created an empty `EchoResponse` structure and suggests that we should fill in the `Out` field (commented field). Let's do this! Remember that we defined EchoResponse.Out as a string in out echo.proto, or you can verify what the structures are by looking at the golang definitions in echo.pb.go. In the case of echo, let's say back what we heard.

```
func (s echoService) Echo(ctx context.Context, in *pb.EchoRequest) (*pb.EchoResponse, error) {
	var resp pb.EchoResponse
	resp = pb.EchoResponse{
	   Out: in.In,
	}
	return &resp, nil
}
```

## Build/Run the client and server executables

From the directory containing echo.proto run
`go build echo/echo-cli-client/client_main.go` and
`go build echo/echo-server/server_main.go`

Create another terminal window to run the server in, navigate to the same directory and launch the server:
`./server_main`
When server starts up, you should see something like this:
```
ts=2016-12-06T23:25:14Z caller=server_main.go:55 msg=hello
ts=2016-12-06T23:25:14Z caller=server_main.go:106 transport=HTTP addr=:5050
ts=2016-12-06T23:25:14Z caller=server_main.go:98 transport=debug addr=:5060
ts=2016-12-06T23:25:14Z caller=server_main.go:124 transport=gRPC addr=:5040

```
The server is now waiting for incoming messages.
At this point we can send a request to the server via networking tools (telnet, curl) and construct message directly, or we can use the client CLI.

Let's do the latter, in your first terminal. To learn how to launch client with proper parameters run `./client_main -help`. The printout will tell you what methods the service supports (-method flag options) and all the additional flags that must be set to call a certain method (flags of format -method_name.parameter_name).

Now run `./client_main -method echo -echo.in “hello microservices”`
The client terminal will display messages that were sent and received.

You can also specify the address to send messages to via -grpc.addr or -http.addr flags (e.g. `-grpc.addr localhost:5040`), should you want to change the port the server runs on, or test it out on separate machines.

To shutdown the server, press Ctrl+C in the server terminal

## Implement more things!

The following is left as an exersise to the reader:
  - Implement logic for the Louder call
    - code the logic inside the stub
    - now separate this logic into an unexported helper function
  - Define a new RPC call in echo.proto
    - regenerate service with truss, check that your old logic remains
    - implement the logic for your new call in a separate package, place it ouside of echo
    - wire in the new logic by importing the package in the server_handler.go
  Suggestion: Save everything the service hears and echo all of it back. See repeated types (protobuf), package variables and init() function (golang).
  - Remove an RPC call definition from echo.proto
  	- regenerate service with truss, verify that the call no longer exists
  - Break things
  - Launch the server on a different port, or different machine, and talk to it (hint: run `./server_main -h`)
  - Try running multiple servers at once

# Additional features

## File placement

You can control the location of the output folders for your service by specifying the following flags when running truss
```
  -svcout {go-style-package-path to where you want the {Name} folder to be}
  -pbout {go-style-package-path to where you want the *.pb.go interface definitions to be}
```

Note: “go-style-package-path” means exactly the style you use in your golang import statements, relative to your $GOPATH. This is not your system file path, nor it is relative to location of the *.proto file; the start of the path must be accessible from your $GOPATH. Also no “/” at the end.
For example:
```
truss -pbout truss-demo/interface-defs -svcout truss-demo/service echo.proto
```
Executing this command will place the *.pb.go files into `$GOPATH/truss-demo/interface-defs/`, and the entire echo directory (excepting the *.pb.go files) to `$GOPATH/truss-demo/service/`.

## Middlewares

 TODO
