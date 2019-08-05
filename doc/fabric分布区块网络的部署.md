# fabric分布区块网络的部署

## 基础环境

```
go 1.12及以上
docker
docker-compose
```

```
go get github.com/hyperledger/fabric
cd $GOPATH/src/github.com/hyperledger/fabric
git checkout release-1.4
go get github.com/hyperledger/fabric-samples
cd $GOPATH/src/github.com/hyperledger/fabric-samples
git checkout release-1.4
```

## 部署

### 生成bin文件

```
cd $GOPATH/src/github.com/hyperledger/fabric
make release
cd $GOPATH/src/github.com/hyperledger/fabric/release/darwin-amd64/bin
cp ./* /usr/local/bin
```

### 设置环境变量

```
export PATH=${PWD}/../bin:${PWD}:$PATH
export FABRIC_CFG_PATH=${PWD}
export CLI_TIMEOUT=10
export CLI_DELAY=3
export CHANNEL_NAME="mychannel"
export COMPOSE_FILE=docker-compose-cli.yaml
export COMPOSE_FILE_COUCH=docker-compose-couch.yaml
export COMPOSE_FILE_ORG3=docker-compose-org3.yaml
export LANGUAGE=golang
export IMAGETAG="latest"
export VERBOSE=true
```

### 生成证书

```
cryptogen generate --config=./crypto-config.yaml
```

### 生成创世区块

```
configtxgen -profile TwoOrgsOrdererGenesis -outputBlock ./channel-artifacts/genesis.block

(configtxgen -inspectBlock channel-artifacts/genesis.block > genesis.block.json   // 查看创世区块信息)
```

### 生成 Channel 配置 Transaction

```
configtxgen -profile TwoOrgsChannel -outputCreateChannelTx ./channel-artifacts/channel.tx -channelID $CHANNEL_NAME
```

### 生成锚节点配置

```
configtxgen -profile TwoOrgsChannel -outputAnchorPeersUpdate ./channel-artifacts/Org1MSPanchors.tx -channelID $CHANNEL_NAME -asOrg Org1MSP           //  Transaction for Org1
configtxgen -profile TwoOrgsChannel -outputAnchorPeersUpdate ./channel-artifacts/Org2MSPanchors.tx -channelID $CHANNEL_NAME -asOrg Org2MSP					 //  Transaction for Org2
```

### 启动网络

```
docker-compose -f $COMPOSE_FILE up -d 2>&1
docker exec -it cli bash  // 进入容器
```

### 设置环境变量（容器内）

```
export CHANNEL_NAME="mychannel"
export LANGUAGE=golang
export ORDERER_CA=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
export PEER0_ORG1_CA=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt
export PEER0_ORG2_CA=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt
export ORG1_MSP=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
export ORG2_MSP=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp
export CC_SRC_PATH="github.com/chaincode/chaincode_example02/go/"
export CORE_PEER_LOCALMSPID="Org1MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=$PEER0_ORG1_CA
export CORE_PEER_MSPCONFIGPATH=$ORG1_MSP
export CORE_PEER_ADDRESS=peer0.org1.example.com:7051
```

### 创建channel

```
peer channel create -o orderer.example.com:7050 -c $CHANNEL_NAME -f ./channel-artifacts/channel.tx --tls $CORE_PEER_TLS_ENABLED --cafile $ORDERER_CA
```

### 将节点加入channel

```
# joinChannelWithRetry 0 1
export CORE_PEER_LOCALMSPID="Org1MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=$PEER0_ORG1_CA
export CORE_PEER_MSPCONFIGPATH=$ORG1_MSP
export CORE_PEER_ADDRESS=peer0.org1.example.com:7051
peer channel join -b $CHANNEL_NAME.block
 
# joinChannelWithRetry 1 1
export CORE_PEER_LOCALMSPID="Org1MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=$PEER0_ORG1_CA
export CORE_PEER_MSPCONFIGPATH=$ORG1_MSP
export CORE_PEER_ADDRESS=peer1.org1.example.com:8051
peer channel join -b $CHANNEL_NAME.block
 
# joinChannelWithRetry 0 2
export CORE_PEER_LOCALMSPID="Org2MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=$PEER0_ORG2_CA
export CORE_PEER_MSPCONFIGPATH=$ORG2_MSP
export CORE_PEER_ADDRESS=peer0.org2.example.com:9051
peer channel join -b $CHANNEL_NAME.block
 
# joinChannelWithRetry 1 2
export CORE_PEER_LOCALMSPID="Org2MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=$PEER0_ORG2_CA
export CORE_PEER_MSPCONFIGPATH=$ORG2_MSP
export CORE_PEER_ADDRESS=peer1.org2.example.com:10051
peer channel join -b $CHANNEL_NAME.block
```

### 更新锚节点

```
# updateAnchorPeers 0 1
export CORE_PEER_LOCALMSPID="Org1MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=$PEER0_ORG1_CA
export CORE_PEER_MSPCONFIGPATH=$ORG1_MSP
export CORE_PEER_ADDRESS=peer0.org1.example.com:7051
peer channel update -o orderer.example.com:7050 -c $CHANNEL_NAME -f ./channel-artifacts/${CORE_PEER_LOCALMSPID}anchors.tx --tls $CORE_PEER_TLS_ENABLED --cafile $ORDERER_CA
 
# updateAnchorPeers 0 2
export CORE_PEER_LOCALMSPID="Org2MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=$PEER0_ORG2_CA
export CORE_PEER_MSPCONFIGPATH=$ORG2_MSP
export CORE_PEER_ADDRESS=peer0.org2.example.com:9051
peer channel update -o orderer.example.com:7050 -c $CHANNEL_NAME -f ./channel-artifacts/${CORE_PEER_LOCALMSPID}anchors.tx --tls $CORE_PEER_TLS_ENABLED --cafile $ORDERER_CA
```

### 安装 Chaincode

```
# installChaincode in peer 0 1
export CORE_PEER_LOCALMSPID="Org1MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=$PEER0_ORG1_CA
export CORE_PEER_MSPCONFIGPATH=$ORG1_MSP
export CORE_PEER_ADDRESS=peer0.org1.example.com:7051
export VERSION="1.0"
peer chaincode install -n mycc -v ${VERSION} -l ${LANGUAGE} -p ${CC_SRC_PATH}
 
# installChaincode in peer 0 2
export CORE_PEER_LOCALMSPID="Org2MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=$PEER0_ORG2_CA
export CORE_PEER_MSPCONFIGPATH=$ORG2_MSP
export CORE_PEER_ADDRESS=peer0.org2.example.com:9051
export VERSION="1.0"
peer chaincode install -n mycc -v ${VERSION} -l ${LANGUAGE} -p ${CC_SRC_PATH}
```

### 实例化 Chaincode

```
# Chaincode 安装后需要实例化才能使用，在channel中，一个chaincode 只需要进行一次 instantiate 操作即可。
export CORE_PEER_LOCALMSPID="Org2MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=$PEER0_ORG2_CA
export CORE_PEER_MSPCONFIGPATH=$ORG2_MSP
export CORE_PEER_ADDRESS=peer0.org2.example.com:9051
export VERSION="1.0"
peer chaincode instantiate -o orderer.example.com:7050 --tls $CORE_PEER_TLS_ENABLED --cafile $ORDERER_CA -C $CHANNEL_NAME -n mycc -l ${LANGUAGE} -v ${VERSION} -c '{"Args":["init","a","100","b","200"]}' -P "AND ('Org1MSP.peer','Org2MSP.peer')"
```

###查询

```
Chaincode –Query
peer chaincode query -C $CHANNEL_NAME -n mycc -c '{"Args":["query","a"]}'
```

### 交易

```
Chaincode--Invoke
export PEER_CONN_PARMS="--peerAddresses peer0.org1.example.com:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt --peerAddresses peer0.org2.example.com:9051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt"

peer chaincode invoke -o orderer.example.com:7050 --tls $CORE_PEER_TLS_ENABLED --cafile $ORDERER_CA -C $CHANNEL_NAME -n mycc $PEER_CONN_PARMS -c '{"Args":["invoke","a","b","10"]}'
```

### 再次查询

```
peer chaincode query -C $CHANNEL_NAME -n mycc -c '{"Args":["query","a"]}'
```



