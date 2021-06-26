docker-compose -f network/docker/docker-compose-test-net.yaml down

docker volume prune -f

sudo rm -fr ./network/organizations/ordererOrganizations/*
sudo rm -fr ./network/organizations/peerOrganizations/*
sudo rm -fr ./network/system-genesis-block/genesis.block/*
export PATH=${PWD}/bin:$PATH

cryptogen generate --config=./network/organizations/cryptogen/crypto-config-org1.yaml --output="./network/organizations"
cryptogen generate --config=./network/organizations/cryptogen/crypto-config-org2.yaml --output="./network/organizations"
cryptogen generate --config=./network/organizations/cryptogen/crypto-config-orderer.yaml --output="./network/organizations"

export FABRIC_CFG_PATH=$PWD/network/configtx/

configtxgen -profile TwoOrgsOrdererGenesis -channelID system-channel -outputBlock ./network/system-genesis-block/genesis.block

docker-compose -f ./network/docker/docker-compose-test-net.yaml up -d

docker ps

configtxgen -profile TwoOrgsChannel -outputCreateChannelTx ./network/channel-artifacts/channel1.tx -channelID channel1


export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_LOCALMSPID="Org1MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/network/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=${PWD}/network/organizations/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
export CORE_PEER_ADDRESS=localhost:7051

export FABRIC_CFG_PATH=$PWD/config

peer channel create -o localhost:7050  --ordererTLSHostnameOverride orderer.example.com -c channel1 -f ./network/channel-artifacts/channel1.tx --outputBlock ./network/channel-artifacts/channel1.block --tls --cafile ${PWD}/network/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
