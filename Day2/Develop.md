
# Develop and deploy the business network locally

>These steps follows the guide [here](https://hyperledger.github.io/composer/latest/tutorials/developer-tutorial)

This tutorial will walk you through building a Hyperledger Composer blockchain solution from scratch. In the space of a few hours you will be able to go from an idea for a disruptive blockchain innovation, to executing transactions against a real Hyperledger Fabric blockchain network.

## Sections
* [Set up your dev environment](#-set-up-your-dev-environment)
* [Step One: Creating a business network structure](#-step-one:-creating-a-business-network-structure)
* [Step Two: Defining a business network](#-step-two:-defining-a-business-network)
* [Step Three: Generate a business network archive](#-step-three:-generate-a-business-network-archive)
* [Step Four: Deploying the business network](#-step-four:-deploying-the-business-network)
* [Step Five: Generating a REST server](#-step-five:-generating-a-REST-server)
* [Step Six: Generating an application](#-step-six:-generating-an-application)

## Set up your dev environment

Follow these instructions to obtain the Hyperledger Composer development tools (primarily used to create Business Networks) and stand up a Hyperledger Fabric (primarily used to run/deploy your Business Networks locally).

### Install Composer Packages

Install the CLI tools

Essential CLI tools:
```
npm install -g composer-cli@0.19.5
```

Utility for running a REST Server on your machine to expose your business networks as RESTful APIs:
```
npm install -g composer-rest-server@0.19.5
```

Useful utility for generating application assets:
```
npm install -g generator-hyperledger-composer@0.19.5
```

Yeoman is a tool for generating applications, which utilises generator-hyperledger-composer:
```
npm install -g yo
```

This command will remove all composer cards
```
rm -rf ~/.composer
```

### Install Composer Fabric Locally

This step gives you a local Hyperledger Fabric runtime to deploy your business networks to.

In a directory of your choice (we will assume ~/fabric-dev-servers), get the .tar.gz file that contains the tools to install Hyperledger Fabric:

```
mkdir ./fabric-dev-servers && cd ./fabric-dev-servers

curl -O https://raw.githubusercontent.com/hyperledger/composer-tools/master/packages/fabric-dev-servers/fabric-dev-servers.tar.gz
tar -xvf fabric-dev-servers.tar.gz
```

>A zip is also available if you prefer: just replace the .tar.gz file with fabric-dev-servers.zip and the tar -xvf command with a unzip command in the preceding snippet.

Use the scripts you just downloaded and extracted to download a local Hyperledger Fabric runtime:

```
cd ./fabric-dev-servers
./downloadFabric.sh
./startFabric.sh
./createPeerAdminCard.sh
```

### Starting and stopping Hyperledger Fabric

* The first time you start up a new runtime, you'll need to run the start script, then generate a PeerAdmin card:

```
cd ./fabric-dev-servers
./startFabric.sh
./createPeerAdminCard.sh
```

* You stop your runtime using:
```
 ./stopFabric.sh,
```

* And can teardown Fabric and Docker setup with:
```
./teardownFabric.sh
./stopFabric.sh
```


## Step One: Creating a business network structure

Create a skeleton business network using Yeoman. This command will require a business network name, description, author name, author email address, license selection and namespace.

```
yo hyperledger-composer:businessnetwork
```

* Enter `tutorial-network` for the network name, and desired information for description, author name, and author email.
* Select `Apache-2.0` as the license.
* Select `org.example.mynetwork` as the namespace.
* Select No when asked whether to generate an empty network or not.


## Step Two: Defining a business network

A business network is made up of assets, participants, transactions, access control rules, and optionally events and queries. In the skeleton business network created in the previous steps, there is:
* a model (`.cto`) file which will contain the class definitions for all assets, participants, and transactions in the business network
* an access control (`permissions.acl`) document with basic access control rules,
* a script (`logic.js`) file containing transaction processor functions
* a `package.json` file containing business network metadata


### Modelling assets, participants, and transactions

The first document to update is the model (`.cto`) file. This file is written using the [Hyperledger Composer Modelling Language](https://hyperledger.github.io/composer/latest/reference/cto_language.html). The model file contains the definitions of each class of asset, transaction, participant, and event.

1. Open the org.example.mynetwork.cto model file.

2. Replace the contents with the following:
```
/**
 * My commodity trading network
 */
namespace org.example.mynetwork
asset Commodity identified by tradingSymbol {
    o String tradingSymbol
    o String description
    o String mainExchange
    o Double quantity
    --> Trader owner
}
participant Trader identified by tradeId {
    o String tradeId
    o String firstName
    o String lastName
}
transaction Trade {
    --> Commodity commodity
    --> Trader newOwner
}

```

3. Save your changes to org.example.mynetwork.cto.


### Adding JavaScript transaction logic

In the model file, a `Trade` transaction was defined, specifying a relationship to an asset, and a participant. The transaction processor function file contains the JavaScript logic to execute the transactions defined in the model file.

The `Trade` transaction is intended to simply accept the identifier of the `Commodity` asset which is being traded, and the identifier of the `Trader` participant to set as the new owner.

1. Open the logic.js script file.

2. Replace the contents with the following:
```
/**
 * Track the trade of a commodity from one trader to another
 * @param {org.example.mynetwork.Trade} trade - the trade to be processed
 * @transaction
 */
async function tradeCommodity(trade) {
    trade.commodity.owner = trade.newOwner;
    let assetRegistry = await getAssetRegistry('org.example.mynetwork.Commodity');
    await assetRegistry.update(trade.commodity);
}
```
3. Save your changes to `logic.js`.


### Adding access control


1. Replace the following access control rules in the file `permissions.acl` :


```
/**
 * Access control rules for tutorial-network
 */
rule Default {
    description: "Allow all participants access to all resources"
    participant: "ANY"
    operation: ALL
    resource: "org.example.mynetwork.*"
    action: ALLOW
}

rule SystemACL {
  description:  "System ACL to permit all access"
  participant: "ANY"
  operation: ALL
  resource: "org.hyperledger.composer.system.**"
  action: ALLOW
}
```

2. Save your changes to `permissions.acl`.


## Step Three: Generate a business network archive

Now that the business network has been defined, it must be packaged into a deployable business network archive (`.bna`) file.

1. Using the command line, navigate to the tutorial-network directory.

2. From the tutorial-network directory, run the following command:
```
composer archive create -t dir -n .
```

After the command has run, a business network archive file called `tutorial-network@0.0.1.bna` has been created in the `tutorial-network` directory.


## Step Four: Deploying the business network


After creating the .bna file, the business network can be deployed to the instance of Hyperledger Fabric. Normally, information from the Fabric administrator is required to create a PeerAdmin identity, with privileges to install chaincode to the peer as well as start chaincode on the composerchannel channel. However, as part of the development environment installation, a PeerAdmin identity has been created already.

### Retrieving the correct credentials

A `PeerAdmin` business network card with the correct credentials is already created as part of development environment installation.

### Deploying the business network

Deploying a business network to the Hyperledger Fabric requires the Hyperledger Composer business network to be installed on the peer, then the business network can be started, and a new participant, identity, and associated card must be created to be the network administrator. Finally, the network administrator business network card must be imported for use, and the network can then be pinged to check it is responding.

1. To install the business network, from the `tutorial-network` directory, run the following command:
```
composer network install --card PeerAdmin@hlfv1 --archiveFile tutorial-network@0.0.1.bna
```

The `composer network install` command requires a PeerAdmin business network card (in this case one has been created and imported in advance), and the the file path of the `.bna` which defines the business network.


2. To start the business network, run the following command:

```
composer network start --networkName tutorial-network --networkVersion 0.0.1 --networkAdmin admin --networkAdminEnrollSecret adminpw --card PeerAdmin@hlfv1 --file networkadmin.card

```

The `composer network start` command requires a business network card, as well as the name of the admin identity for the business network, the name and version of the business network and the name of the file to be created ready to import as a business network card.


3. To import the network administrator identity as a usable business network card, run the following command:

```
composer card import --file networkadmin.card
```
The `composer card import` command requires the filename specified in composer network start to create a card.


4. To check that the business network has been deployed successfully, run the following command to ping the network:
```
composer network ping --card admin@tutorial-network
```

The `composer network ping` command requires a business network card to identify the network to ping.


## Step Five: Generating a REST server

Hyperledger Composer can generate a bespoke REST API based on a business network. For developing a web application, the REST API provides a useful layer of language-neutral abstraction.

To create the REST API, navigate to the `tutorial-network` directory and run the following command:
```
composer-rest-server
```

* Enter `admin@tutorial-network` as the card name.
* Select `never use namespaces` when asked whether to use namespaces in the generated API.
* Select `No` when asked whether to secure the generated API.
* Select `No` when asked whether to enable authentication using Passport.
* Select `Yes` when asked whether to enable event publication.
* Select `No` when asked whether to enable TLS security.

The generated API is connected to the deployed blockchain and business network.


## Step Six: Generating an application

Hyperledger Composer can also generate an Angular 4 application running against the REST API.


To create your Angular 4 application, navigate to `tutorial-network` directory and run the following command:
```
yo hyperledger-composer:angular
```

* Select `Yes` when asked to connect to running business network.
* Enter standard `package.json` questions (project name, description, author name, author email, license)
* Enter `admin@tutorial-network` for the business network card.
* Select `Connect to an existing REST API`
* Enter `http://localhost` for the REST server address.
* Enter `3000` for server port
* Select `Namespaces are not used`

The Angular generator will then create the scaffolding for the project and install all dependencies. To run the application, navigate to your angular project directory and run:
```
npm start
```
This will fire up an Angular 4 application running against your REST API at `http://localhost:4200` .
