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
docker volume prune


#############################
## create crypto materials ##
#############################

# remove the old materials
sudo rm -fr ./network/organizations/ordererOrganizations/*
sudo rm -fr ./network/organizations/peerOrganizations/*
sudo rm -fr ./network/system-genesis-block/*

# export binaries to be available from any directory
export PATH=${PWD}/bin:$PATH

# generate crypto materials
cryptogen generate --config=./network/organizations/cryptogen/crypto-config-org1.yaml --output="./network/organizations"
cryptogen generate --config=./network/organizations/cryptogen/crypto-config-org2.yaml --output="./network/organizations"
cryptogen generate --config=./network/organizations/cryptogen/crypto-config-orderer.yaml --output="./network/organizations"

# set the cfg path
export FABRIC_CFG_PATH=$PWD/network/configtx/

# create the genesis block
configtxgen -profile TwoOrgsApplicationGenesis -channelID system-channel -outputBlock ./network/system-genesis-block/genesis.block/genesis.block 


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

