
# Chaincode for Developers

>These steps follows the guide [here](https://hyperledger-fabric.readthedocs.io/en/latest/chaincode4ade.html)

In the following sections, we will explore chaincode through the eyes of an application developer. We’ll present a simple chaincode sample application and walk through the purpose of each method in the Chaincode Shim API.

We will demonstrate the use of **Chaincode APIs** for **Go** language, by implementing a simple chaincode application that manages simple “assets”.  Next, we'll build our code and create a testing environment to test the chaincode.

This guide is divided in two sections:
1. [**Writing Chaincode**](#1-writing-chaincode)
2. [**Build and Test Chaincode**]((#2-build-and-test-chaincode))

Lets get started!

## 1. Writing Chaincode

Here we will go through steps on writing chaincode. Our application is a basic sample chaincode to create assets (key-value pairs) on the ledger.

### Choose a Location for the Code

First, ensure that your `GOPATH` and `GOBIN` path are pointing correctly:

```
export GOPATH=$HOME/go
export GOBIN=$HOME/go/bin
```

To keep things simple, let’s use the following command:
```
mkdir -p $GOPATH/src/sacc && cd $GOPATH/src/sacc
```

Now, let’s create the source file that we’ll fill in with code:

```
touch sacc.go
```

### Dependencies

First, let’s start with some housekeeping.  Let’s add the go import statements for the necessary dependencies for our chaincode. We’ll import the chaincode shim package and the [peer protobuf package](https://godoc.org/github.com/hyperledger/fabric/protos/peer).


```
package main

import (
    "fmt"

    "github.com/hyperledger/fabric/core/chaincode/shim"
    "github.com/hyperledger/fabric/protos/peer"
)
```

### Asset Struct

Next, let’s add a struct SimpleAsset as a receiver for Chaincode shim functions.

```
// SimpleAsset implements a simple chaincode to manage an asset
type SimpleAsset struct {
}
```

### Initializing the Chaincode

Next, we’ll implement the Init function:
```
// Init is called during chaincode instantiation to initialize any data.
func (t *SimpleAsset) Init(stub shim.ChaincodeStubInterface) peer.Response {

}
```

Next, we’ll retrieve the arguments to the `Init` call using the [ChaincodeStubInterface.GetStringArgs](https://godoc.org/github.com/hyperledger/fabric/core/chaincode/shim#ChaincodeStub.GetStringArgs) function and check for validity. In our case, we are expecting a key-value pair.

```
// Init is called during chaincode instantiation to initialize any
// data. Note that chaincode upgrade also calls this function to reset
// or to migrate data, so be careful to avoid a scenario where you
// inadvertently clobber your ledger's data!
func (t *SimpleAsset) Init(stub shim.ChaincodeStubInterface) peer.Response {
  // Get the args from the transaction proposal
  args := stub.GetStringArgs()
  if len(args) != 2 {
    return shim.Error("Incorrect arguments. Expecting a key and a value")
  }
}
```

Next, now that we have established that the call is valid, we’ll store the initial state in the ledger. To do this, we will call [ChaincodeStubInterface.PutState](https://godoc.org/github.com/hyperledger/fabric/core/chaincode/shim#ChaincodeStub.PutState) with the key and value passed in as the arguments. Assuming all went well, return a `peer.Response` object that indicates the initialization was a success.


```
// Init is called during chaincode instantiation to initialize any
// data. Note that chaincode upgrade also calls this function to reset
// or to migrate data, so be careful to avoid a scenario where you
// inadvertently clobber your ledger's data!
func (t *SimpleAsset) Init(stub shim.ChaincodeStubInterface) peer.Response {
  // Get the args from the transaction proposal
  args := stub.GetStringArgs()
  if len(args) != 2 {
    return shim.Error("Incorrect arguments. Expecting a key and a value")
  }

  // Set up any variables or assets here by calling stub.PutState()

  // We store the key and the value on the ledger
  err := stub.PutState(args[0], []byte(args[1]))
  if err != nil {
    return shim.Error(fmt.Sprintf("Failed to create asset: %s", args[0]))
  }
  return shim.Success(nil)
}
```


### Invoking the Chaincode

First, let’s add the Invoke function’s signature:

```
// Invoke is called per transaction on the chaincode. Each transaction is
// either a 'get' or a 'set' on the asset created by Init function. The 'set'
// method may create a new asset by specifying a new key-value pair.
func (t *SimpleAsset) Invoke(stub shim.ChaincodeStubInterface) peer.Response {

}
```


As with the Init function above, we need to extract the arguments from the ChaincodeStubInterface. The Invoke function’s arguments will be the name of the chaincode application function to invoke. In our case, our application will simply have two functions: set and get, that allow the value of an asset to be set or its current state to be retrieved. We first call [ChaincodeStubInterface.GetFunctionAndParameters](https://godoc.org/github.com/hyperledger/fabric/core/chaincode/shim#ChaincodeStub.GetFunctionAndParameters) to extract the function name and the parameters to that chaincode application function.

```
// Invoke is called per transaction on the chaincode. Each transaction is
// either a 'get' or a 'set' on the asset created by Init function. The Set
// method may create a new asset by specifying a new key-value pair.
func (t *SimpleAsset) Invoke(stub shim.ChaincodeStubInterface) peer.Response {
    // Extract the function and args from the transaction proposal
    fn, args := stub.GetFunctionAndParameters()

}
```

Next, we’ll validate the function name as being either `set` or `get`, and invoke those chaincode application functions, returning an appropriate response via the `shim.Success` or `shim.Error` functions that will serialize the response into a gRPC protobuf message.

```
// Invoke is called per transaction on the chaincode. Each transaction is
// either a 'get' or a 'set' on the asset created by Init function. The Set
// method may create a new asset by specifying a new key-value pair.
func (t *SimpleAsset) Invoke(stub shim.ChaincodeStubInterface) peer.Response {
    // Extract the function and args from the transaction proposal
    fn, args := stub.GetFunctionAndParameters()

    var result string
    var err error
    if fn == "set" {
            result, err = set(stub, args)
    } else {
            result, err = get(stub, args)
    }
    if err != nil {
            return shim.Error(err.Error())
    }

    // Return the result as success payload
    return shim.Success([]byte(result))
}
```


### Implementing the Chaincode Application

As noted, our chaincode application implements two functions that can be invoked via the `Invoke` function. Let’s implement those functions now. Note that as we mentioned above, to access the ledger’s state, we will leverage the [ChaincodeStubInterface.PutState](https://godoc.org/github.com/hyperledger/fabric/core/chaincode/shim#ChaincodeStub.PutState) and [ChaincodeStubInterface.GetState](https://godoc.org/github.com/hyperledger/fabric/core/chaincode/shim#ChaincodeStub.GetState) functions of the chaincode shim API.

```
// Set stores the asset (both key and value) on the ledger. If the key exists,
// it will override the value with the new one
func set(stub shim.ChaincodeStubInterface, args []string) (string, error) {
    if len(args) != 2 {
            return "", fmt.Errorf("Incorrect arguments. Expecting a key and a value")
    }

    err := stub.PutState(args[0], []byte(args[1]))
    if err != nil {
            return "", fmt.Errorf("Failed to set asset: %s", args[0])
    }
    return args[1], nil
}

// Get returns the value of the specified asset key
func get(stub shim.ChaincodeStubInterface, args []string) (string, error) {
    if len(args) != 1 {
            return "", fmt.Errorf("Incorrect arguments. Expecting a key")
    }

    value, err := stub.GetState(args[0])
    if err != nil {
            return "", fmt.Errorf("Failed to get asset: %s with error: %s", args[0], err)
    }
    if value == nil {
            return "", fmt.Errorf("Asset not found: %s", args[0])
    }
    return string(value), nil
}
```

### Pulling it All Together

Finally, we need to add the `main` function, which will call the [shim.Start](https://godoc.org/github.com/hyperledger/fabric/core/chaincode/shim#Start) function. Here’s the whole chaincode program source.


```
package main

import (
    "fmt"

    "github.com/hyperledger/fabric/core/chaincode/shim"
    "github.com/hyperledger/fabric/protos/peer"
)

// SimpleAsset implements a simple chaincode to manage an asset
type SimpleAsset struct {
}

// Init is called during chaincode instantiation to initialize any
// data. Note that chaincode upgrade also calls this function to reset
// or to migrate data.
func (t *SimpleAsset) Init(stub shim.ChaincodeStubInterface) peer.Response {
    // Get the args from the transaction proposal
    args := stub.GetStringArgs()
    if len(args) != 2 {
            return shim.Error("Incorrect arguments. Expecting a key and a value")
    }

    // Set up any variables or assets here by calling stub.PutState()

    // We store the key and the value on the ledger
    err := stub.PutState(args[0], []byte(args[1]))
    if err != nil {
            return shim.Error(fmt.Sprintf("Failed to create asset: %s", args[0]))
    }
    return shim.Success(nil)
}

// Invoke is called per transaction on the chaincode. Each transaction is
// either a 'get' or a 'set' on the asset created by Init function. The Set
// method may create a new asset by specifying a new key-value pair.
func (t *SimpleAsset) Invoke(stub shim.ChaincodeStubInterface) peer.Response {
    // Extract the function and args from the transaction proposal
    fn, args := stub.GetFunctionAndParameters()

    var result string
    var err error
    if fn == "set" {
            result, err = set(stub, args)
    } else { // assume 'get' even if fn is nil
            result, err = get(stub, args)
    }
    if err != nil {
            return shim.Error(err.Error())
    }

    // Return the result as success payload
    return shim.Success([]byte(result))
}

// Set stores the asset (both key and value) on the ledger. If the key exists,
// it will override the value with the new one
func set(stub shim.ChaincodeStubInterface, args []string) (string, error) {
    if len(args) != 2 {
            return "", fmt.Errorf("Incorrect arguments. Expecting a key and a value")
    }

    err := stub.PutState(args[0], []byte(args[1]))
    if err != nil {
            return "", fmt.Errorf("Failed to set asset: %s", args[0])
    }
    return args[1], nil
}

// Get returns the value of the specified asset key
func get(stub shim.ChaincodeStubInterface, args []string) (string, error) {
    if len(args) != 1 {
            return "", fmt.Errorf("Incorrect arguments. Expecting a key")
    }

    value, err := stub.GetState(args[0])
    if err != nil {
            return "", fmt.Errorf("Failed to get asset: %s with error: %s", args[0], err)
    }
    if value == nil {
            return "", fmt.Errorf("Asset not found: %s", args[0])
    }
    return string(value), nil
}

// main function starts up the chaincode in the container during instantiate
func main() {
    if err := shim.Start(new(SimpleAsset)); err != nil {
            fmt.Printf("Error starting SimpleAsset chaincode: %s", err)
    }
}
```

## 2. Build and Test Chaincode

In this section we will build and test chaincode.

### Building Chaincode

Now let’s compile your chaincode.

```
go get -u github.com/hyperledger/fabric/core/chaincode/shim
go build
```

### Start testing environment


Navigate to the `chaincode-docker-devmode` directory of the `fabric-samples` clone:

```
cd chaincode-docker-devmode
```

Now open three terminals and navigate to your `chaincode-docker-devmode` directory in each.

### Terminal 1 - Start the network

```
docker-compose -f docker-compose-simple.yaml up
```

The above starts the network with the SingleSampleMSPSolo orderer profile and launches the peer in “dev mode”. It also launches two additional containers - one for the chaincode environment and a CLI to interact with the chaincode. The commands for create and join channel are embedded in the CLI container, so we can jump immediately to the chaincode calls.

### Terminal 2 - Build & start the chaincode
```
docker exec -it chaincode bash
```

You should see the following:
```
root@d2629980e76b:/opt/gopath/src/chaincode#
```

Now, compile your chaincode:
```
cd sacc
go build
```

Now run the chaincode:
```
CORE_PEER_ADDRESS=peer:7052 CORE_CHAINCODE_ID_NAME=mycc:0 ./sacc
```

The chaincode is started with peer and chaincode logs indicating successful registration with the peer. Note that at this stage the chaincode is not associated with any channel. This is done in subsequent steps using the `instantiate` command.

### Terminal 3 - Use the chaincode

Even though you are in `--peer-chaincodedev` mode, you still have to install the chaincode so the life-cycle system chaincode can go through its checks normally. This requirement may be removed in future when in `--peer-chaincodedev` mode.

We’ll leverage the CLI container to drive these calls.

```
docker exec -it cli bash
```

```
peer chaincode install -p chaincodedev/chaincode/sacc -n mycc -v 0
peer chaincode instantiate -n mycc -v 0 -c '{"Args":["a","10"]}' -C myc
```

## Call invoke and query to Chaincode

Now issue an invoke to change the value of “a” to “20”.

```
peer chaincode invoke -n mycc -c '{"Args":["set", "a", "20"]}' -C myc
```

Finally, query `a`. We should see a value of `20`.

```
peer chaincode query -n mycc -c '{"Args":["query","a"]}' -C myc
```

By default, we mount only sacc. However, you can easily test different chaincodes by adding them to the chaincode subdirectory and relaunching your network. At this point they will be accessible in your chaincode container.
