# breakwater-grpc

Breakwater-grpc is a gRPC implementation for the Breakwater microservice overload control framework, written in Go as a package for better generalizability. It is designed to control demand spikes with receiver-driven credit based rate limting, as demonstrated in the [breakwater paper](https://www.usenix.org/conference/osdi20/presentation/cho).

## Impementation

The key design features of the Breakwater framework via credit admission were closely followed in this implementation, including Demand Speculation and Overcommitment of credits. The primary difference between the original framework and the gRPC implementation is the measure of delay. 
In breakwater-grpc, delay is measured on the server-side via `/sched/latencies:seconds`. This is the waiting time of the goroutine in the scheduler queue, an analog to the thread waiting time in the original framework. The delay is then used to calculate the credit balance and determine whether to accept or reject a request.

## Installation

To use the breakwater-grpc package, you need to have Go installed on your system. You can then install the package using the following command:

```go get -u github.com/Jiali-Xing/breakwater-grpc```

## How to use

The `breakwater-grpc` package provides both client-side and server-side interceptors for overload control in gRPC applications. Below is a simple example demonstrating how to set up a server and client using the package.

#### Server Example

```go
import (
	"github.com/Jiali-Xing/breakwater-grpc"
	"google.golang.org/grpc"
)

// Initialize the Breakwater interceptor with default parameters
breakwater := bw.InitBreakwater(bw.BWParametersDefault)

// Setup a new gRPC server with Breakwater Unary Interceptor
grpcServer := grpc.NewServer(grpc.UnaryInterceptor(breakwater.UnaryInterceptor))

// Register services and start the server
pb.RegisterGreetingServiceServer(grpcServer, &greetingServiceServer{})
grpcServer.Serve(listener)
```

#### Client Example

```go
import (
	"github.com/Jiali-Xing/breakwater-grpc"
	"google.golang.org/grpc"
)

// Set up a connection to the gRPC server with Breakwater client-side interceptor
conn, err := grpc.Dial("server_address", grpc.WithUnaryInterceptor(breakwater.UnaryInterceptorClient))
if err != nil {
	log.Fatalf("Failed to connect: %v", err)
}

// Initialize the client and make requests
client := pb.NewGreetingServiceClient(conn)
response, err := client.Greeting(ctx, &pb.GreetingRequest{})
```

### Configurations

You can customize Breakwater's behavior by passing specific parameters to `InitBreakwater`. For example, you can adjust the Service Level Objective (SLO), initial credits, load shedding, and other factors to fit your application’s needs.

```go
bwConfig := bw.BWParameters{
	SLO:            200,		// Service Level Objective for response times. in microseconds
	InitialCredits: 1000,		// Initial credit balance
	AFactor:        0.5,		// Load shedding factor
	BFactor:        0.5,		// Load shedding factor
}
breakwater := bw.InitBreakwater(bwConfig)
```

For more detailed examples and advanced usage, please refer to the `client_interceptor.go` and `server_interceptor.go` files in the `breakwater` folder.

### BreakwaterD

BreakwaterD is used to manage overload control when your service makes downstream calls to other services. It allows you to configure separate Breakwater instances for each downstream service, customizing the SLO, credits, and load-shedding behavior for each.

#### Example of BreakwaterD

```go
import (
	"github.com/Jiali-Xing/breakwater-grpc"
	"google.golang.org/grpc"
)


// InitializeBreakwaterd initializes Breakwater instances for each downstream service
func (s *greetingServiceServer) InitializeBreakwaterd(bwConfig bw.BWParameters) {
	s.breakwaterd = make(map[string]*bw.Breakwater)
	for _, downstream := range s.downstreams {
		// Customize the Breakwater config per downstream if needed
		// For example, you might have different SLOs or other parameters per downstream service
		downstreamConfig := bwConfig
		bwConfig.ServerSide = false
		addr := getURL(downstream)
		s.breakwaterd[addr] = bw.InitBreakwater(downstreamConfig)
	}
}

func run() error {
	// ...
	breakwater := &bw.Breakwater{}

	if intercept == "breakwater" || (intercept == "breakwaterd" && isFrontend) {
		bwConfig.Verbose = logLevel == "Debug"
		bwConfig.SLO = breakwaterSLO.Microseconds()
		bwConfig.ClientExpiration = breakwaterClientTimeout.Microseconds()
		bwConfig.InitialCredits = breakwaterInitialCredit
		bwConfig.LoadShedding = breakwaterLoadShedding
		bwConfig.ServerSide = true
		bwConfig.AFactor = breakwaterA
		bwConfig.BFactor = breakwaterB
		bwConfig.RTT_MICROSECOND = breakwaterRTT.Microseconds()
		bwConfig.TrackCredits = breakwaterTrackCredit
		
		breakwater = bw.InitBreakwater(bwConfig)
		// print the breakwater config for debugging
		log.Printf("Breakwater Frontend Config: %v", bwConfig)
	} else if intercept == "breakwaterd" {
		bwConfig.Verbose = logLevel == "Debug"
		bwConfig.SLO = breakwaterdSLO.Microseconds()
		bwConfig.ClientExpiration = breakwaterdClientTimeout.Microseconds()
		bwConfig.InitialCredits = breakwaterdInitialCredit
		bwConfig.LoadShedding = true
		bwConfig.ServerSide = true
		bwConfig.AFactor = breakwaterdA
		bwConfig.BFactor = breakwaterdB
		bwConfig.LoadShedding = breakwaterdLoadShedding
		bwConfig.RTT_MICROSECOND = breakwaterdRTT.Microseconds()
		bwConfig.TrackCredits = breakwaterTrackCredit

		// Initialize the Breakwater instances for each downstream service
		breakwater = bw.InitBreakwater(bwConfig)
		log.Printf("BreakwaterD Backend Config: %v", bwConfig)
	}
	
	// ...

	if intercept == "breakwaterd" {
		// Initialize the Breakwater instances for each downstream service
		server.InitializeBreakwaterd(bwConfig)
		log.Printf("Initialized multiple instances of Breakwaters for downstream services of %s: %v", serviceName, server.breakwaterd)
	}

	var grpcServer *grpc.Server
	switch intercept {
	case "breakwater", "breakwaterd":
		grpcServer = grpc.NewServer(grpc.UnaryInterceptor(breakwater.UnaryInterceptor))
	default:
		grpcServer = grpc.NewServer()
	}

	// ...
}
```

BreakwaterD is used to manage overload control within the service graph. It allows you to configure separate Breakwater instances for each downstream service, customizing the SLO, credits, and load-shedding behavior for each. Well in our example, we did not use different configs for different services, only two sets of configs are used, for the frontend services and all other services, respectively.
