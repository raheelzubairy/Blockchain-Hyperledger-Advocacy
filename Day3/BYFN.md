
# Building Your First Network

>These steps follows the guide [here](https://hyperledger-fabric.readthedocs.io/en/latest/build_network.html)

The build your first network (BYFN) scenario provisions a sample Hyperledger Fabric network consisting of two organizations, each maintaining two peer nodes, and a “solo” ordering service.

## Steps
1. [Install Samples, Binaries and Docker Images](#1-install-samples,-binaries-and-docker-images)
2. [Generate Network artifacts](#2-generate-network-artifacts)
3. [Create a Channel Configuration Transaction](#3-create-a-channel-configuration-transaction)
4. [Start the network](#4-start-the-network)
5. [Create and join Channel](#5-create-and-join-channel)
6. [Update the anchor peers](#6-update-the-anchor-peers)
7. [Install and Instantiate Chaincode](#7-install-and-instantiate-chaincode)
8. [Interact with network](#8-interact-with-network)

Lets get started!



## 1. Install Samples, Binaries and Docker Images

```
docker kill $(docker ps -q)
docker rm $(docker ps -aq)
docker rmi $(docker images dev-* -q)
```

* Create a folder `BYFN`.  Navigate into the folder.
```
mkdir BYFN
cd BYFN
```

* The command that follows will perform the following steps:

   * If needed, clone the hyperledger/fabric-samples repository
   * Checkout the appropriate version tag
   * Install the Hyperledger Fabric platform-specific binaries and config files for the version specified into the root of the fabric-samples repository
   * Download the Hyperledger Fabric docker images for the version specified

Run the command:
```
curl -sSL http://bit.ly/2ysbOFE | bash -s 1.1.0
```

* Add binaries to your PATH environment variable
```
cd fabric-samples/bin/
export PATH=$(pwd):$PATH
```


## 2. Generate Network Artifacts

This first step generates all of the certificates and keys for our various network entities, the genesis block used to bootstrap the ordering service, and a collection of configuration transactions required to configure a Channel.

```
cd ../first-network
./byfn.sh generate
```

You will see a brief description as to what will occur, along with a yes/no command line prompt. Respond with a y or hit the return key to execute the described action.

### Bring up the network

Next, you can bring the network up with one of the following commands:

```
./byfn.sh up
```

or

```
./byfn.sh -m up
```

This will ensure correct setup of Fabric dependencies.

### Bring Down the Network

Finally, let’s bring it all down so we can explore the network setup one step at a time. The following will kill your containers, remove the crypto material and four artifacts, and delete the chaincode images from your Docker Registry:

```
./byfn.sh down
```

### Manually generate the artifacts

We will use the [cryptogen tool](https://hyperledger-fabric.readthedocs.io/en/latest/build_network.html#crypto-generator) to generate the cryptographic material (x509 certs and signing keys) for our various network entities. These certificates are representative of identities, and they allow for sign/verify authentication to take place as our entities communicate and transact.


The [configtxgen tool](https://hyperledger-fabric.readthedocs.io/en/latest/build_network.html#configuration-transaction-generator) is used to create four configuration artifacts:

        * orderer genesis block,
        * channel configuration transaction,
        * and two anchor peer transactions - one for each Peer Org.

First let’s run the cryptogen tool. Our binary is in the bin directory, so we need to provide the relative path to where the tool resides.

```
../bin/cryptogen generate --config=./crypto-config.yaml
```

You should see the following in your terminal:

```
org1.example.com
org2.example.com
```

The certs and keys (i.e. the MSP material) will be output into a directory - `crypto-config` - at the root of the `first-network` directory.

Next, we need to tell the `configtxgen` tool where to look for the `configtx.yaml` file that it needs to ingest. We will tell it look in our present working directory:

```
export FABRIC_CFG_PATH=$PWD
```

Then, we’ll invoke the `configtxgen` tool to create the orderer genesis block:

```
../bin/configtxgen -profile TwoOrgsOrdererGenesis -outputBlock ./channel-artifacts/genesis.block
```

You should see an output similar to the following in your terminal:

```
2017-10-26 19:21:56.301 EDT [common/tools/configtxgen] main -> INFO 001 Loading configuration
2017-10-26 19:21:56.309 EDT [common/tools/configtxgen] doOutputBlock -> INFO 002 Generating genesis block
2017-10-26 19:21:56.309 EDT [common/tools/configtxgen] doOutputBlock -> INFO 003 Writing genesis block
```

## 3. Create a Channel Configuration Transaction

Next, we need to create the channel transaction artifact.

```
export CHANNEL_NAME=mychannel  && ../bin/configtxgen -profile TwoOrgsChannel -outputCreateChannelTx ./channel-artifacts/channel.tx -channelID $CHANNEL_NAME
```

You should see an output similar to the following in your terminal:
```
2017-10-26 19:24:05.324 EDT [common/tools/configtxgen] main -> INFO 001 Loading configuration
2017-10-26 19:24:05.329 EDT [common/tools/configtxgen] doOutputChannelCreateTx -> INFO 002 Generating new channel configtx
2017-10-26 19:24:05.329 EDT [common/tools/configtxgen] doOutputChannelCreateTx -> INFO 003 Writing new channel tx
```

Next, we will define the anchor peer for Org1 on the channel that we are constructing.

```
../bin/configtxgen -profile TwoOrgsChannel -outputAnchorPeersUpdate ./channel-artifacts/Org1MSPanchors.tx -channelID $CHANNEL_NAME -asOrg Org1MSP
```

Now, we will define the anchor peer for Org2 on the same channel:
```
../bin/configtxgen -profile TwoOrgsChannel -outputAnchorPeersUpdate ./channel-artifacts/Org2MSPanchors.tx -channelID $CHANNEL_NAME -asOrg Org2MSP
```


## 4. Start the network

We will leverage a script to spin up our network. The docker-compose file references the images that we have previously downloaded, and bootstraps the orderer with our previously generated `genesis.block`. We want to go through the commands manually in order to expose the syntax and functionality of each call.

First let’s start our network:

```
docker-compose -f docker-compose-cli.yaml up -d
```

You should see:

```
Creating peer1.org2.example.com ... done
Creating peer0.org1.example.com ... done
Creating peer1.org1.example.com ... done
Creating orderer.example.com    ... done
Creating peer0.org2.example.com ... done
Creating cli                    ... done
```


## 5. Create & Join Channel

Recall that we created the channel configuration transaction using the `configtxgen` tool in the `Create a Channel Configuration Transaction` section, above. You can repeat that process to create additional channel configuration transactions, using the same or different profiles in the `configtx.yaml` that you pass to the `configtxgen` tool. Then you can repeat the process defined in this section to establish those other channels in your network.


We will enter the CLI container using the docker exec command:

```
docker exec -it cli bash
```

If successful you should see the following:
```
root@0d78bb69300d:/opt/gopath/src/github.com/hyperledger/fabric/peer#
```

For the CLI commands against `peer0.org1.example.com` to work, we need to preface our commands with the four environment variables given below. These variables for `peer0.org1.example.com` are baked into the CLI container, therefore we can operate without passing them. HOWEVER, if you want to send calls to other peers or the orderer, then you can provide these values accordingly by editing the `docker-compose-base.yaml` before starting the container. Modify the following four environment variables to use a different peer and org.

```
export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
export CORE_PEER_ADDRESS=peer0.org1.example.com:7051
export CORE_PEER_LOCALMSPID="Org1MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt
```

Next, we are going to pass in the generated channel configuration transaction artifact that we created in the Create a Channel Configuration Transaction section (we called it channel.tx) to the orderer as part of the create channel request.

```
export CHANNEL_NAME=mychannel

peer channel create -o orderer.example.com:7050 -c $CHANNEL_NAME -f ./channel-artifacts/channel.tx --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
```

This command returns a genesis block - `<channel-ID.block>` - which we will use to join the channel. It contains the configuration information specified in `channel.tx` If you have not made any modifications to the default channel name, then the command will return you a proto titled `mychannel.block`.

Now let’s join `peer0.org1.example.com` to the channel.

```
peer channel join -b mychannel.block
```

You should see:

```
2018-07-26 23:39:14.824 UTC [channelCmd] InitCmdFactory -> INFO 001 Endorser and orderer connections initialized
2018-07-26 23:39:14.851 UTC [channelCmd] executeJoin -> INFO 002 Successfully submitted proposal to join channel
```

Rather than join every peer, we will simply join `peer0.org2.example.com` so that we can properly update the anchor peer definitions in our channel. Since we are overriding the default environment variables baked into the CLI container, this full command will be the following:

```
CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp CORE_PEER_ADDRESS=peer0.org2.example.com:7051 CORE_PEER_LOCALMSPID="Org2MSP" CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt peer channel join -b mychannel.block
```

This should return:

```
2018-07-26 23:40:52.345 UTC [channelCmd] InitCmdFactory -> INFO 001 Endorser and orderer connections initialized
2018-07-26 23:40:52.373 UTC [channelCmd] executeJoin -> INFO 002 Successfully submitted proposal to join channel
```


## 6. Update the anchor peers

The following commands are channel updates and they will propagate to the definition of the channel. In essence, we adding additional configuration information on top of the channel’s genesis block to define the anchor peers.

Update the channel definition to define the anchor peer for Org1 as `peer0.org1.example.com`:

```
peer channel update -o orderer.example.com:7050 -c $CHANNEL_NAME -f ./channel-artifacts/Org1MSPanchors.tx --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
```

You should see:

```
2018-07-26 23:42:02.138 UTC [channelCmd] InitCmdFactory -> INFO 001 Endorser and orderer connections initialized
2018-07-26 23:42:02.149 UTC [channelCmd] update -> INFO 002 Successfully submitted channel update
```


Now update the channel definition to define the anchor peer for Org2 as `peer0.org2.example.com`. Identically to the `peer channel join` command for the Org2 peer, we will need to preface this call with the appropriate environment variables.

```
CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp CORE_PEER_ADDRESS=peer0.org2.example.com:7051 CORE_PEER_LOCALMSPID="Org2MSP" CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt peer channel update -o orderer.example.com:7050 -c $CHANNEL_NAME -f ./channel-artifacts/Org2MSPanchors.tx --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
```

You should see:

```
2018-07-26 23:43:03.603 UTC [channelCmd] InitCmdFactory -> INFO 001 Endorser and orderer connections initialized
2018-07-26 23:43:03.614 UTC [channelCmd] update -> INFO 002 Successfully submitted channel update
```


## 7. Install & Instantiate Chaincode

Applications interact with the blockchain ledger through chaincode. As such we need to install the chaincode on every peer that will execute and endorse our transactions, and then instantiate the chaincode on the channel.

First, install the sample Go chaincode onto one of the four peer nodes. These commands place the specified source code flavor onto our peer’s filesystem.

```
peer chaincode install -n mycc -v 1.0 -p github.com/chaincode/chaincode_example02/go/
```
You should see:

```
2018-07-26 23:43:51.950 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 001 Using default escc
2018-07-26 23:43:51.950 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 002 Using default vscc
2018-07-26 23:43:52.126 UTC [chaincodeCmd] install -> INFO 003 Installed remotely response:<status:200 payload:"OK" >
```

Next, instantiate the chaincode on the channel. This will initialize the chaincode on the channel, set the endorsement policy for the chaincode, and launch a chaincode container for the targeted peer. Take note of the `-P` argument. This is our policy where we specify the required level of endorsement for a transaction against this chaincode to be validated.

In the command below you’ll notice that we specify our policy as `-P "AND ('Org1MSP.peer','Org2MSP.peer')"`. This means that we need “endorsement” from a peer belonging to Org1 AND Org2 (i.e. two endorsement). If we changed the syntax to OR then we would need only one endorsement.

```
peer chaincode instantiate -o orderer.example.com:7050 --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C mychannel -n mycc -l golang -v 1.0 -c '{"Args":["init","a","100","b","200"]}' -P 'OR	('\''Org1MSP.peer'\'','\''Org2MSP.peer'\'')'
```

This should return:
```
2018-07-26 23:44:48.868 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 001 Using default escc
2018-07-26 23:44:48.868 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 002 Using default vscc
```

## 8. Interact with network

### Query

Let’s query for the value of `a` to make sure the chaincode was properly instantiated and the state DB was populated. The syntax for query is as follows:

```
peer chaincode query -C mychannel -n mycc -c '{"Args":["query","a"]}'
```

This should return:
```
100
```

### Invoke

Now let’s move `10` from `a` to `b`. This transaction will cut a new block and update the state DB. The syntax for invoke is as follows:  


<!-- ```
peer chaincode invoke -o orderer.example.com:7050 --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C $CHANNEL_NAME -n mycc --peerAddresses peer0.org1.example.com:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt --peerAddresses peer0.org2.example.com:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt -c '{"Args":["invoke","a","b","10"]}'
``` -->
```
peer chaincode invoke -o orderer.example.com:7050 --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C mychannel -n mycc -c '{"Args":["invoke","a","b","10"]}'
```

### Query

Let’s confirm that our previous invocation executed properly. We initialized the key `a` with a value of `100` and just removed `10` with our previous invocation. Therefore, a query against `a` should reveal `90`. The syntax for query is as follows.


<!-- ```
peer chaincode query -C $CHANNEL_NAME -n mycc -c '{"Args":["query","a"]}'
``` -->
```
peer chaincode query -C mychannel -n mycc -c '{"Args":["query","a"]}'
```

This should return:
```
90
```

### Additional Detail

Get more detail on [what's happening behind the scenes](https://hyperledger-fabric.readthedocs.io/en/latest/build_network.html#what-s-happening-behind-the-scenes)
