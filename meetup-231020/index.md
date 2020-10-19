# Deploying a smart contract to a channel, the fabric chaincode lifecycle

In this tutorial we are going to install the predefined abstore chaincode.

1. Start the network, create and join a channel
2. Package the smart contract
3. Install the chaincode package
4. Approve a chaincode definition
5. Committing the chaincode definition to the channel

## (1) Start the network, create and join a channel

tmux control
```bash
# start a new tmux session
tmux new -s fabric

# attach to existing session
tmux add -t fabric

# show all logs
docker-compose -f docker/docker-compose-test-net.yaml logs -f -t

# open a new panel
CTRL + b \" 

# jump between panels
CTRL + b + q 1

# detach from session
CTRL + b + d
```

```bash 
# clear the system
docker system prune 

# start the network with channel1
./network.sh createChannel -c channel1
```

Make sure that all comming steps are executed by a Org Admin user.

## (2) Package the chaincode
Before we package the chaincode, we need to install the chaincode dependences. Navigate to the folder that contains the Go version of the chaincode.

To install the smart contract dependencies, run the following command from the asset-transfer-basic/chaincode-go directory.

```bash
cd chaincode/abstore
GO111MODULE=on go mod vendor
```

We need some environment variables.
```bash 
cd ../../
export FABRIC_CFG_PATH=$PWD/../config/
```

You can now create the chaincode package using the peer lifecycle chaincode package command:

```bash 
peer lifecycle chaincode package abstore.tar.gz --path chaincode/abstore/ --lang golang --label abstore_1

# inspect the tar package
tar tfz abstore.tar.gz

# extract the package and inspect
tar -xzf abstore.tar.gz

cat metadata.json | jq .
```

## (3) Install the chaincode package
After we packaged or chaincode, we can install the chaincode on our peers. The chaincode needs to be installed on every peer that will endorse a transaction. Because we are going to set the endorsement policy to require endorsements from both Org1 and Org2, we need to install the chaincode on the peers operated by both organizations:

- peer0.org1.example.com
- peer0.org2.example.com

Let’s install the chaincode on the Org1 peer first. Set the following environment variables to operate the peer CLI as the Org1 admin user. The CORE_PEER_ADDRESS will be set to point to the Org1 peer, peer0.org1.example.com.

### Install chaincode Org1
```bash 
#  create a new env file and fill in the following env vars
vi org1.env

export FABRIC_CFG_PATH=$PWD/../config/
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_LOCALMSPID="Org1MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
export CORE_PEER_ADDRESS=localhost:7051

# execute the env file
source org1.env
```

Issue the peer lifecycle chaincode install command to install the chaincode on the peer:

```bash
peer lifecycle chaincode install abstore.tar.gz
```

As a result we will receive the chaoncode package identifier.
```bash
Chaincode code package identifier: abstore_1:3918d0438fd2ebe48ed1bde01533513a14f788846fd2d72ef054482760e73409
```

### Install chaincode Org2
```bash 
#  create a new env file and fill in the following env vars
vi org2.env

export FABRIC_CFG_PATH=$PWD/../config/
export CORE_PEER_LOCALMSPID="Org2MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/organizations/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt
export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/organizations/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp
export CORE_PEER_ADDRESS=localhost:9051

# execute the env file
source org2.env
```

Issue the peer lifecycle chaincode install command to install the chaincode on the peer:

```bash
peer lifecycle chaincode install abstore.tar.gz
```

As a result we will receive the chaoncode package identifier.
```bash
Chaincode code package identifier: abstore_1:3918d0438fd2ebe48ed1bde01533513a14f788846fd2d72ef054482760e73409
```

## (4) Approve a chaincode definition
After you install the chaincode package, you need to approve a chaincode definition for your organization. If an organization has installed the chaincode on their peer, they need to include the packageID in the chaincode definition approved by their organization. The package ID is used to associate the chaincode installed on a peer with an approved chaincode definition, and allows an organization to use the chaincode to endorse transactions. You can find the package ID of a chaincode by using the peer lifecycle chaincode queryinstalled command to query your peer.

```bash
peer lifecycle chaincode queryinstalled
```

To apprive to the chaincode we need one more environment variable, the so-called CC_PACKAGE_ID.
```bash
export CC_PACKAGE_ID=abstore_1:3918d0438fd2ebe48ed1bde01533513a14f788846fd2d72ef054482760e73409
```

### Approve the chaincode for Org2
```bash
# command to explain
peer lifecycle chaincode approveformyorg 
  -o localhost:7050 
  --ordererTLSHostnameOverride orderer.example.com 
  --channelID channel1 
  --name asbtore 
  --version 1
  --package-id $CC_PACKAGE_ID 
  --sequence 1 
  --tls 
  --cafile ${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
```

```bash
# command to copy
peer lifecycle chaincode approveformyorg -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com --channelID channel1 --name abstore --version 1 --package-id $CC_PACKAGE_ID --sequence 1 --tls --cafile ${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
```

Check the approve progress.
```bash
# command to explain
peer lifecycle chaincode checkcommitreadiness 
  --channelID channel1 
  --name abstore 
  --version 1
  --sequence 1 
  --tls 
  --cafile ${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem --output json
```

```bash
# command to copy
peer lifecycle chaincode checkcommitreadiness --channelID channel1 --name abstore --version 1 --sequence 1 --tls --cafile ${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem --output json
```

### Approve the chaincode for Org1

We have to set the environment variables for Org1.
```bash
source org1.env
```
The commands for the approve process and checkreadyness step are the same like Org2.

```bash
# command to copy
peer lifecycle chaincode approveformyorg -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com --channelID channel1 --name abstore --version 1 --package-id $CC_PACKAGE_ID --sequence 1 --tls --cafile ${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
```

Check the approve progress.
```bash
# command to copy
peer lifecycle chaincode checkcommitreadiness --channelID channel1 --name abstore --version 1 --sequence 1 --tls --cafile ${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem --output json
```

## (5) Committing the chaincode definition to the channel

```bash
# command to explain
peer lifecycle chaincode commit 
  -o localhost:7050 
  --ordererTLSHostnameOverride orderer.example.com 
  --channelID channel1 
  --name abstore 
  --version 1
  --sequence 1 
  --tls 
  --cafile ${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem --peerAddresses localhost:7051 --tlsRootCertFiles ${PWD}/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt --peerAddresses localhost:9051 --tlsRootCertFiles ${PWD}/organizations/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt

# command to copy
peer lifecycle chaincode commit -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com --channelID channel1 --name abstore --version 1 --sequence 1 --tls --cafile ${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem --peerAddresses localhost:7051 --tlsRootCertFiles ${PWD}/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt --peerAddresses localhost:9051 --tlsRootCertFiles ${PWD}/organizations/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt
```

You can use the peer lifecycle chaincode querycommitted command to confirm that the chaincode definition has been committed to the channel.

```bash
# command to explain
peer lifecycle chaincode querycommitted 
  --channelID channel1 
  --name abstore 
  --cafile ${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem

# command to copy
peer lifecycle chaincode querycommitted --channelID channel1 --name abstore --cafile ${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
```

## (6) Use the abstore chaincode

###  Methods
 - Init
 - Invoke
 - Query
 - Delete

#### Init the chaincode
```bash
# command to explain
peer chaincode invoke 
  -o localhost:7050 
  --ordererTLSHostnameOverride orderer.example.com 
  --tls 
  --cafile ${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem 
  -C channel1 
  -n abstore 
  --peerAddresses localhost:7051 --tlsRootCertFiles ${PWD}/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt --peerAddresses localhost:9051 --tlsRootCertFiles ${PWD}/organizations/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt -c '{"function":"Init","Args":["account1","1000","account2","10"]}'
 
# command to copy
peer chaincode invoke -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com --tls --cafile ${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C channel1 -n abstore --peerAddresses localhost:7051 --tlsRootCertFiles ${PWD}/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt --peerAddresses localhost:9051 --tlsRootCertFiles ${PWD}/organizations/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt -c '{"function":"Init","Args":["account1","1000","account2","10"]}'
```

#### Query the chaincode
```bash
peer chaincode query -C channel1 -n abstore -c '{"function":"Query","Args":["account1"]}'
peer chaincode query -C channel1 -n abstore -c '{"function":"Query","Args":["account2"]}'
```

#### Invoke the chaincode
```bash
# command to explain
peer chaincode invoke
  -o localhost:7050 
  --ordererTLSHostnameOverride orderer.example.com 
  --tls 
  --cafile ${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem 
  -C channel1 
  -n abstore 
  --peerAddresses localhost:7051 --tlsRootCertFiles ${PWD}/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt --peerAddresses localhost:9051 --tlsRootCertFiles ${PWD}/organizations/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt -c '{"function":"Invoke","Args":["account1","account2","100"]}'

# command to copy
peer chaincode invoke -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com  --tls --cafile ${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C channel1 -n abstore --peerAddresses localhost:7051 --tlsRootCertFiles ${PWD}/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt --peerAddresses localhost:9051 --tlsRootCertFiles ${PWD}/organizations/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt -c '{"function":"Invoke","Args":["account1","account2","100"]}'
```

#### Delete an asset
```bash
# command to explain
peer chaincode invoke
  -o localhost:7050 
  --ordererTLSHostnameOverride orderer.example.com 
  --tls 
  --cafile ${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem 
  -C channel1 
  -n abstore 
  --peerAddresses localhost:7051 --tlsRootCertFiles ${PWD}/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt --peerAddresses localhost:9051 --tlsRootCertFiles ${PWD}/organizations/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt -c '{"function":"delete","Args":["account2]}'

# command to copy
peer chaincode invoke -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com --tls --cafile ${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C channel1 -n abstore --peerAddresses localhost:7051 --tlsRootCertFiles ${PWD}/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt --peerAddresses localhost:9051 --tlsRootCertFiles ${PWD}/organizations/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt -c '{"function":"delete","Args":["account2"]}'
```

## Upgrade the chaincode

Modify the chaincode

Install the smart contract dependencies.
```bash
cd chaincode/abstore2
GO111MODULE=on go mod vendor
```

To package the chaincode, we need some environment variables and change our position.
```bash 
cd ../../
export FABRIC_CFG_PATH=$PWD/../config/
```

You can now create the new chaincode package using the peer lifecycle chaincode package command:

```bash 
peer lifecycle chaincode package abstore2.tar.gz --path chaincode/abstore2/ --lang golang --label abstore_1.5

# inspect the tar package
tar tfz abstore2.tar.gz
```

### Steps to go
- Install the new chaincode version on Org1
- Approve the version on Org1
- checkcommitreadiness
- Install the new chaincode version on Org2
- Approve the version on Org2
- checkcommitreadiness
- commit the chaincode


#### Org1

```bash
# execute the env file
source org1.env
```

Install the upgraded chaincode
```bash
peer lifecycle chaincode install abstore2.tar.gz
peer lifecycle chaincode queryinstalled

# copy the packageId hash
export NEW_CC_PACKAGE_ID=abstore_1.5:c4fbdec55ca793ea77b91e34384297d2e0cf69da24874f628da04b5e63f9c36a

peer lifecycle chaincode approveformyorg -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com --channelID channel1 --name abstore --version 1.5 --package-id $NEW_CC_PACKAGE_ID --sequence 4 --tls --cafile ${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem

peer lifecycle chaincode checkcommitreadiness --channelID channel1 --name abstore --version 1.5 --sequence 4 --tls --cafile ${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem --output json
```

#### Org2

```bash
# execute the env file
source org2.env
```

Install the upgraded chaincode
```bash
peer lifecycle chaincode install abstore2.tar.gz
peer lifecycle chaincode queryinstalled

# copy the packageId hash
export NEW_CC_PACKAGE_ID=abstore_1.1:0a9c434fe75915443f6eb95bb9d68da9706452d2ac320c8a0dda9c67d1759bec

peer lifecycle chaincode approveformyorg -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com --channelID channel1 --name abstore --version 1.1 --package-id $NEW_CC_PACKAGE_ID --sequence 2 --tls --cafile ${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem

peer lifecycle chaincode checkcommitreadiness --channelID channel1 --name abstore --version 1.1 --sequence 2 --tls --cafile ${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem --output json
```

#### Commit the upgraded chaincode
The chaincode will be upgraded on the channel after the new chaincode definition is committed. Until then, the previous chaincode will continue to run on the peers of both organizations. Org2 can use the following command to upgrade the chaincode:

```bash 
peer lifecycle chaincode commit -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com --channelID channel1 --name abstore --version 1.5 --sequence 4 --tls --cafile ${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem --peerAddresses localhost:7051 --tlsRootCertFiles ${PWD}/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt --peerAddresses localhost:9051 --tlsRootCertFiles ${PWD}/organizations/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt

# check with
docker ps
```

Try the new chaincode

```bash
# command to copy
peer chaincode invoke -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com  --tls --cafile ${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C channel1 -n abstore --peerAddresses localhost:7051 --tlsRootCertFiles ${PWD}/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt --peerAddresses localhost:9051 --tlsRootCertFiles ${PWD}/organizations/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt -c '{"function":"Invoke","Args":["account1","account2","50"]}'
```

Query the new assets
```bash
peer chaincode query -C channel1 -n abstore -c '{"function":"Query","Args":["account1"]}'
peer chaincode query -C channel1 -n abstore -c '{"function":"Query","Args":["account2"]}'
peer chaincode query -C channel1 -n abstore -c '{"function":"Query","Args":["bank"]}'
```