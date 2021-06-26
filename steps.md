##################################
#                                #
#    CREATING CRYPTO MATERIAL    #
#                                #
##################################

#######################
## reset the network ##
#######################

# shut down the containers from the compose file
docker-compose -f network/docker/docker-compose-test-net.yaml down

# clear the volumes
docker volume prune -f


#############################
## create crypto materials ##
#############################

# remove the old materials
sudo rm -fr ./network/organizations/ordererOrganizations/*
sudo rm -fr ./network/organizations/peerOrganizations/*
sudo rm -fr ./network/system-genesis-block/genesis.block/*

# export binaries to be available from any directory
export PATH=${PWD}/bin:$PATH

# generate crypto materials
cryptogen generate --config=./network/organizations/cryptogen/crypto-config-org1.yaml --output="./network/organizations"
cryptogen generate --config=./network/organizations/cryptogen/crypto-config-org2.yaml --output="./network/organizations"
cryptogen generate --config=./network/organizations/cryptogen/crypto-config-orderer.yaml --output="./network/organizations"

# set the cfg path
#############################################
#                                           #
# <<<<<<<<<< TODO verify this!!! >>>>>>>>>> #
#                                           #
#############################################
export FABRIC_CFG_PATH=$PWD/network/configtx/

# create the genesis block
configtxgen -profile TwoOrgsOrdererGenesis -channelID system-channel -outputBlock ./network/system-genesis-block/genesis.block
# configtxgen -profile TwoOrgsOrdererGenesis -channelID system-channel -outputBlock ./network/channel-artifacts/genesis.block


##################################
#                                #
#    DEPLOYING THE NETWORK       #
#                                #
##################################

#######################
## start the network ##
#######################

# start the containers from the compose file
docker-compose -f ./network/docker/docker-compose-test-net.yaml up -d

# checkout the network
docker ps



##################################
#                                #
#    CREATING A CHANNEL          #
#                                #
##################################

#######################
## start the network ##
#######################

# set the cfg path
export FABRIC_CFG_PATH=$PWD/network/configtx/

# create channel artifacts folder
mkdir $PWD/network/channel-artifacts

# create the channel creation transaction
configtxgen -profile TwoOrgsChannel -outputCreateChannelTx ./network/channel-artifacts/channel1.tx -channelID channel1

# create the channel creation transaction
# configtxgen -profile TwoOrgsApplicationGenesis -outputCreateChannelTx ./network/channel-artifacts/genesisChannel.tx -channelID genesisChannel

# set environment to the admin user of Org1
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_LOCALMSPID="Org1MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/network/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=${PWD}/network/organizations/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
export CORE_PEER_ADDRESS=localhost:7051

# set the cfg path to the core file
export FABRIC_CFG_PATH=$PWD/config

# create the channel
peer channel create -o localhost:7050  --ordererTLSHostnameOverride orderer.example.com -c channel1 -f ./network/channel-artifacts/channel1.tx --outputBlock ./network/channel-artifacts/channel1.block --tls --cafile ${PWD}/network/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem

# join org1 to the channel
peer channel join -b ./network/channel-artifacts/channel1.block

# show the blockheight of the channel
peer channel getinfo -c channel1

# set environment to the admin user of Org2
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_LOCALMSPID="Org2MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/network/organizations/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=${PWD}/network/organizations/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp
export CORE_PEER_ADDRESS=localhost:9051

# fetch the first block from the ordering service
peer channel fetch 0 ./channel-artifacts/channel_org2.block -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com -c channel1 --tls --cafile ${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem

# join org2 to the channel
peer channel join -b ./channel-artifacts/channel_org2.block

# set the cfg path
export FABRIC_CFG_PATH=$PWD/configtx/

# create the anchor update transaction for org2
configtxgen -profile TwoOrgsChannel -outputAnchorPeersUpdate ./channel-artifacts/org2_anchor.tx -channelID channel1 -asOrg Org2MSP

# set the cfg path to the core file
export FABRIC_CFG_PATH=$PWD/../config

# send the anchor update to the ordering service
peer channel update -o localhost:7050 -c channel1 -f ./channel-artifacts/org2_anchor.tx --ordererTLSHostnameOverride orderer.example.com --tls --cafile ${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem

# set environment to the admin user of Org1
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_LOCALMSPID="Org1MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
export CORE_PEER_ADDRESS=localhost:7051

# set the cfg path
export FABRIC_CFG_PATH=$PWD/configtx/

# create the anchor update transaction for org1
configtxgen -profile TwoOrgsChannel -outputAnchorPeersUpdate ./channel-artifacts/org1_anchor.tx -channelID channel1 -asOrg Org1MSP

# set the cfg path to the core file
export FABRIC_CFG_PATH=$PWD/../config

# send the anchor update to the ordering service
peer channel update -o localhost:7050 -c channel1 -f ./channel-artifacts/org1_anchor.tx  --ordererTLSHostnameOverride orderer.example.com --tls --cafile ${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem

# show the blockheight of the channel
peer channel getinfo -c channel1



##################################
#                                #
#    DEPLOYING A SMART CONTRACT  #
#                                #
##################################

#######################
## reset the network ##
#######################


# shut down the containers from the compose file
docker-compose -f ./network/docker/docker-compose-test-net.yaml down

# clear the volumes
docker volume prune


#######################
## install chaincode ##
#######################

# go to the chaincode location
cd ./chaincode/go

# add the dependencies
go mod vendor

# return to the network folder
cd .. && cd ..

# set cfg path
export FABRIC_CFG_PATH=$PWD/config

# package the chaincode
peer lifecycle chaincode package chaincode1.tar.gz --path /chaincode/go --lang golang --label chaincode_1

# Set environment to Org1
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_LOCALMSPID="Org1MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/network/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=${PWD}/network/organizations/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
export CORE_PEER_ADDRESS=localhost:7051

# Install the chaincode
peer lifecycle chaincode install chaincode1.tar.gz

# Set environment to Org2
export CORE_PEER_LOCALMSPID="Org2MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/network/organizations/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt
export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/network/organizations/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=${PWD}/network/organizations/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp
export CORE_PEER_ADDRESS=localhost:9051

# Install the chaincode
peer lifecycle chaincode install chaincode1.tar.gz

#######################
## approve chaincode ##
#######################

# Get the package ID of our chaincode
peer lifecycle chaincode queryinstalled

# export the ID as a variable
export CC_PACKAGE_ID=<...replace with ID...>

# approve for Org2
peer lifecycle chaincode approveformyorg -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com --channelID channel1 --name chaincode1 --version 1.0 --package-id $CC_PACKAGE_ID --sequence 1 --tls --cafile ${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem

# Check the approvals
peer lifecycle chaincode checkcommitreadiness --channelID channel1 --name chaincode1 --version 1.0 --sequence 1 --tls --cafile ${PWD}/network/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem --output json

# Set environment to Org1
export CORE_PEER_LOCALMSPID="Org1MSP"
export CORE_PEER_MSPCONFIGPATH=${PWD}/network/organizations/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/network/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt
export CORE_PEER_ADDRESS=localhost:7051

# approve for Org1
peer lifecycle chaincode approveformyorg -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com --channelID channel1 --name chaincode1 --version 1.0 --package-id $CC_PACKAGE_ID --sequence 1 --tls --cafile ${PWD}/network/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem

# Check the approvals
peer lifecycle chaincode checkcommitreadiness --channelID channel1 --name chaincode1 --version 1.0 --sequence 1 --tls --cafile ${PWD}/network/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem --output json

# Commit the approved chaincode for both of the organizations
peer lifecycle chaincode commit -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com --channelID channel1 --name chaincode1 --version 1.0 --sequence 1 --tls --cafile ${PWD}/network/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem --peerAddresses localhost:7051 --tlsRootCertFiles ${PWD}/network/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt --peerAddresses localhost:9051 --tlsRootCertFiles ${PWD}/network/organizations/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt

# checkout all the chaincode defenitions on the channel
peer lifecycle chaincode querycommitted --channelID channel1 --name chaincode1 --cafile ${PWD}/network/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem

# initialize the chaincode
peer chaincode invoke -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com --tls --cafile ${PWD}/network/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C channel1 -n chaincode1 --peerAddresses localhost:7051 --tlsRootCertFiles ${PWD}/network/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt --peerAddresses localhost:9051 --tlsRootCertFiles ${PWD}/network/organizations/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt -c '{"function":"InitLedger","Args":[]}'

# query the chaincode
peer chaincode query -C channel1 -n chaincode1 -c '{"Args":["queryAllCars"]}'



