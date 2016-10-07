# gengokit

gengokit is a `truss` plugin that from a `deftree` and `[]truss.SimpleFile`

1. Generates Golang code for a gokit microservice that includes:
	- Logging
	- Metrics/Instrumentation
	- gRPC transport
	- http/json transport (including all encoding/decoding)
	- no-op handler methods for each *service* rpc, ready for business logic to be added
2. Generates Golang code for a cli gokit microservice client that includes:
	- gRPC transport
	- http/json transport (including all encoding/decoding)
	- handler methods that marshal command line arguments into server requests
3. Parses code previously generated by `protoc-gen-truss-gokit` and inserts/removes handler methods/rpcs that are defined/removed from the definition

## Development

### Build

` $ go generate github.com/TuneLab/go-truss/...`
` $ go install github.com/TuneLab/go-truss/...`

### Test

`make test` from the `$GOPATH/src/github.com/TuneLab/go-truss/truss` directory runs the integration tests


### Structure

`./template/service` contain go text/template files that represent a gokit microservice

`./astmodifier` provides functions to modify source code already generated and/or user modified. 

`gengokit.go` executes the template files using `deftree`'s representation of the `protoc` AST. `./generator` also uses `./astmodifier` to rewrite code to insert/remove handler methods/rpcs that are defined/removed from a definition file without touching user written logic.


# NOTE:
- No Service rpc methods named "NewBasicService" allowed for generation reasons 
