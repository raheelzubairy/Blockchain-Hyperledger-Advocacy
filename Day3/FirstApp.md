
# Writing Your First Application

>These steps follows the guide [here](http://hyperledger-fabric.readthedocs.io/en/latest/write_first_app.html)

In this section we’ll be looking at an application to provide a broad demonstration of Fabric functionality. Notably, we will show the process for interacting with a Certificate Authority and generating enrollment certificates, after which we will leverage these identities to query and update a ledger.

## Steps
1. [Install Samples, Binaries and Docker Images](#1-install-samples,-binaries-and-docker-images)
2. [Setting up your Dev Environment](#2-setting-up-your-dev-environment)
3. [Install the clients & launch the network](#3-install-the-clients-&-launch-the-network)
4. [Enrolling the Admin User](#4-enrolling-the-admin-user)
5. [Register and Enroll user1](#5-register-and-enroll-user1)
6. [Querying the Ledger](#6-guerying-the-ledger)
7. [Updating the Ledger](#7-updating-the-ledger)

Lets get started!

## 1. Install Samples, Binaries and Docker Images

* Create a a new folder.  Navigate into the folder.

* The command that follows will perform the following steps:

   * If needed, clone the hyperledger/fabric-samples repository
   * Checkout the appropriate version tag
   * Install the Hyperledger Fabric platform-specific binaries and config files for the version specified into the root of the fabric-samples repository
   * Download the Hyperledger Fabric docker images for the version specified

Run the command:
```
curl -sSL http://bit.ly/2ysbOFE | bash -s 1.2.0
```

* Add binaries to your PATH environment variable
```
cd fabric-samples/bin/
export PATH=$(pwd):$PATH
```


## 2. Setting up your Dev Environment

 Navigate to the `fabcar` subdirectory within your `fabric-samples` repository and take a look at what’s inside

```
cd ../../
cd fabric-samples/fabcar  && ls
```

You should see the following:
```
enrollAdmin.js     invoke.js       package.json    query.js        registerUser.js startFabric.sh
```

Before starting we also need to do a little housekeeping. Run the following command to kill any stale or active containers:

```
docker rm -f $(docker ps -aq)
```

Clear any cached networks:
```
docker network prune
```


And lastly if you’ve already run through this tutorial, you’ll also want to delete the underlying chaincode image for the `fabcar` smart contract. If you’re a user going through this content for the first time, then you won’t have this chaincode image on your system:

```
docker rmi dev-peer0.org1.example.com-fabcar-1.0-5c906e402ed29f20260ae42283216aa75549c571e2e380f3615826365d8269ba
```


## 3. Install the clients & launch the network

Run the following command to install the Fabric dependencies for the applications. We are concerned with `fabric-ca-client` which will allow our app(s) to communicate with the CA server and retrieve identity material, and with `fabric-client` which allows us to load the identity material and talk to the peers and ordering service.

```
npm install
```

Launch your network using the `startFabric.sh` shell script. This command will spin up our various Fabric entities and launch a smart contract container for chaincode written in Golang:

```
./startFabric.sh
```

## 4. Enrolling the Admin User

To stream your CA logs, split your terminal or open a new shell and issue the following:

```
docker logs -f ca.example.com
```


When we launched our network, an admin user – `admin` – was registered with our Certificate Authority. Now we need to send an enroll call to the CA server and retrieve the enrollment certificate (eCert) for this user. We won’t delve into enrollment details here, but suffice it to say that the SDK and by extension our applications need this cert in order to form a user object for the admin. We will then use this admin object to subsequently register and enroll a new user. Send the admin enroll call to the CA server:

```
node enrollAdmin.js
```

You should see:
```
Successfully enrolled admin user "admin"
Assigned the admin user to the fabric client ::{"name":"admin","mspid":"Org1MSP","roles":null,"affiliation":"","enrollmentSecret":"","enrollment":{"signingIdentity":"94cd4cccf215f797f00e521e904b1acfe6bd544534d7b2a2741c6cc7d7615dc9","identity":{"certificate":"-----BEGIN CERTIFICATE-----\nMIICATCCAaigAwIBAgIUJoqe45ojCRvoI27uuOI3RSP8xakwCgYIKoZIzj0EAwIw\nczELMAkGA1UEBhMCVVMxEzARBgNVBAgTCkNhbGlmb3JuaWExFjAUBgNVBAcTDVNh\nbiBGcmFuY2lzY28xGTAXBgNVBAoTEG9yZzEuZXhhbXBsZS5jb20xHDAaBgNVBAMT\nE2NhLm9yZzEuZXhhbXBsZS5jb20wHhcNMTgwNzI2MjIyMDAwWhcNMTkwNzI2MjIy\nNTAwWjAhMQ8wDQYDVQQLEwZjbGllbnQxDjAMBgNVBAMTBWFkbWluMFkwEwYHKoZI\nzj0CAQYIKoZIzj0DAQcDQgAE8vsF8d2zqERhDn1Rvraa/Kjx2Iyo99S9PpI5kbWK\nRAV46l1LJKohB657zUfuIcfvqKRDLcR207z4KRseLCmWbaNsMGowDgYDVR0PAQH/\nBAQDAgeAMAwGA1UdEwEB/wQCMAAwHQYDVR0OBBYEFIyi3ExFBsb4Sk0sY6ACWLLh\nNJ8MMCsGA1UdIwQkMCKAIEI5qg3NdtruuLoM2nAYUdFFBNMarRst3dusalc2Xkl8\nMAoGCCqGSM49BAMCA0cAMEQCIGQgO8WJJg56KMqsnvZfJ2jnOhxJlTmJ3wqW7oMB\n2aM2AiA2Cnspw00FHarP4I/Sy4uaOp3hFG6p7EvM4oc6c0O9Bg==\n-----END CERTIFICATE-----\n"}}}
```

## 5. Register and Enroll user1

With our newly generated admin eCert, we will now communicate with the CA server once more to register and enroll a new user. This user – `user1` – will be the identity we use when querying and updating the ledger. It’s important to note here that it is the `admin` identity that is issuing the registration and enrollment calls for our new user (i.e. this user is acting in the role of a registrar). Send the register and enroll calls for `user1`:

```
node registerUser.js
```

Similar to the admin enrollment, this program invokes a CSR and outputs the keys and eCert into the `hfc-key-store` subdirectory. So now we have identity material for two separate users – `admin` & `user1`. You should see:
```
Successfully loaded admin from persistence
Successfully registered user1 - secret:bnZvpvpwNXaf
Successfully enrolled member user "user1"
User1 was successfully registered and enrolled and is ready to interact with the fabric network
```

## 6. Querying the Ledger

Let’s run our `query.js` program to return a listing of all the cars on the ledger. We will use our second identity – `user1` – as the signing entity for this application. The following line in our program specifies `user1` as the signer:

```
fabric_client.getUserContext('user1', true);
```

This is the chunk where we construct our query:

```
// queryCar chaincode function - requires 1 argument, ex: args: ['CAR4'],
// queryAllCars chaincode function - requires no arguments , ex: args: [''],
const request = {
  //targets : --- letting this default to the peers assigned to the channel
  chaincodeId: 'fabcar',
  fcn: 'queryAllCars',
  args: ['']
};
```

We can run the program as is:

```
node query.js
```

It should return something like this:

```
Successfully loaded user1 from persistence
Query has completed, checking results
Response is  [{"Key":"CAR0", "Record":{"colour":"blue","make":"Toyota","model":"Prius","owner":"Tomoko"}},{"Key":"CAR1", "Record":{"colour":"red","make":"Ford","model":"Mustang","owner":"Brad"}},{"Key":"CAR2", "Record":{"colour":"green","make":"Hyundai","model":"Tucson","owner":"Jin Soo"}},{"Key":"CAR3", "Record":{"colour":"yellow","make":"Volkswagen","model":"Passat","owner":"Max"}},{"Key":"CAR4", "Record":{"colour":"black","make":"Tesla","model":"S","owner":"Adriana"}},{"Key":"CAR5", "Record":{"colour":"purple","make":"Peugeot","model":"205","owner":"Michel"}},{"Key":"CAR6", "Record":{"colour":"white","make":"Chery","model":"S22L","owner":"Aarav"}},{"Key":"CAR7", "Record":{"colour":"violet","make":"Fiat","model":"Punto","owner":"Pari"}},{"Key":"CAR8", "Record":{"colour":"indigo","make":"Tata","model":"Nano","owner":"Valeria"}},{"Key":"CAR9", "Record":{"colour":"brown","make":"Holden","model":"Barina","owner":"Shotaro"}}]
```

These are the 10 cars. A black Tesla Model S owned by Adriana, a red Ford Mustang owned by Brad, a violet Fiat Punto owned by Pari, and so on. The ledger is key-value based and, in our implementation, the key is `CAR0` through `CAR9`. This will become particularly important in a moment.


Update `query.js` program for request to look like this:

```
const request = {
  //targets : --- letting this default to the peers assigned to the channel
  chaincodeId: 'fabcar',
  fcn: 'queryCar',
  args: ['CAR4']
};
```

Save the program and navigate back to your `fabcar` directory. Now run the program again:

```
node query.js
```

You should see the following:

```
{"colour":"black","make":"Tesla","model":"S","owner":"Adriana"}
```

## 7. Updating the Ledger

Now that we’ve done a few ledger queries and added a bit of code, we’re ready to update the ledger. There are a lot of potential updates we could make, but let’s start by creating a car.


#### Create Car

Our first update to the ledger will be to create a new car. We have a separate Javascript program – invoke.js – that we will use to make updates. Just as with queries, use an editor to open the program and navigate to the code block where we construct our invocation:


```
// createCar chaincode function - requires 5 args, ex: args: ['CAR12', 'Honda', 'Accord', 'Black', 'Tom'],
// changeCarOwner chaincode function - requires 2 args , ex: args: ['CAR10', 'Barry'],
// must send the proposal to endorsing peers
var request = {
  //targets: let default to the peer assigned to the client
  chaincodeId: 'fabcar',
  fcn: '',
  args: [''],
  chainId: 'mychannel',
  txId: tx_id
};
```

You’ll see that we can call one of two functions – `createCar` or `changeCarOwner`. First, let’s create a red Chevy Volt and give it to an owner named Nick. We’re up to `CAR9` on our ledger, so we’ll use `CAR10` as the identifying key here. Edit this code block to look like this:

```
var request = {
  //targets: let default to the peer assigned to the client
  chaincodeId: 'fabcar',
  fcn: 'createCar',
  args: ['CAR10', 'Chevy', 'Volt', 'Red', 'Nick'],
  chainId: 'mychannel',
  txId: tx_id
};
```

Save it and run the program:

```
node invoke.js
```

There will be some output in the terminal about `ProposalResponse` and promises. However, all we’re concerned with is this message:

```
Successfully committed the change to the ledger by the peer
```



#### Query new car

To see that this transaction has been written, go back to `query.js` and change the argument from `CAR4` to `CAR10`.

In other words, change this:

```
const request = {
  //targets : --- letting this default to the peers assigned to the channel
  chaincodeId: 'fabcar',
  fcn: 'queryCar',
  args: ['CAR4']
};
```

To this:

```
const request = {
  //targets : --- letting this default to the peers assigned to the channel
  chaincodeId: 'fabcar',
  fcn: 'queryCar',
  args: ['CAR10']
};
```

Save once again, then query:

```
node query.js
```

Which should return this:

```
Response is  {"colour":"Red","make":"Chevy","model":"Volt","owner":"Nick"}
```

Congratulations. You’ve created a car!

#### Change car owner

So now that we’ve done that, let’s say that Nick is feeling generous and he wants to give his Chevy Volt to someone named Dave.

To do this go back to `invoke.js` and change the function from `createCar` to `changeCarOwner` and input the arguments like this:

```
var request = {
  //targets: let default to the peer assigned to the client
  chaincodeId: 'fabcar',
  fcn: 'changeCarOwner',
  args: ['CAR10', 'Dave'],
  chainId: 'mychannel',
  txId: tx_id
};
```

The first argument – `CAR10` – reflects the car that will be changing owners. The second argument – `Dave` – defines the new owner of the car.

Save and execute the program again:

```
node invoke.js
```

Now let’s query the ledger again and ensure that Dave is now associated with the CAR10 key:

```
node query.js
```

It should return this result:

```
Response is  {"colour":"Red","make":"Chevy","model":"Volt","owner":"Dave"}
```

The ownership of CAR10 has been changed from Nick to Dave.


### Troubleshoot

* If error during `invoke`:

```
Failed to invoke successfully :: TypeError: fabric_client.newEventHub is not a function
```
Update the `invoke.js` file with following:

on line 105:
```
let event_hub = channel.newChannelEventHub('localhost:7051');
// event_hub.setPeerAddr('grpc://localhost:7053');
```
on line 130:
```
console.log('The transaction has been committed on peer ' + event_hub.getPeerAddr());
```
