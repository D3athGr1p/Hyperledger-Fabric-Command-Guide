# How add org to the channel.

## create the new org artifacts
```javascript
cryptogen generate --config=org3-crypto.yaml --output="../organizations"
```

- create configtx.yaml file and fill up the details and genrate the appropriate json file.
```javascript
export FABRIC_CFG_PATH=$PWD
configtxgen -printOrg Org3MSP > ../organizations/peerOrganizations/org3.example.com/org3.json
```

## bring up the org3 peer
```javascript
docker-compose -f docker/docker-compose-org3.yaml up -d
```

## Fetch the Configuration

- fetch the most recent config block for the channel – channel1

#### Note :  Org3 is not yet a member of the channel, we need to operate as the admin of another organization to fetch the channel config. Because Org1 is a member of the channel, the Org1 admin has permission to fetch the channel config from the ordering service. Issue the following commands to operate as the Org1 admin.

```javascript
export PATH=${PWD}/../bin:$PATH
export FABRIC_CFG_PATH=${PWD}/../config/
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_LOCALMSPID="Org1MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
export CORE_PEER_ADDRESS=localhost:7051
```

- We can now issue the command to fetch the latest config block:

```javascript
peer channel fetch config channel-artifacts/config_block.pb -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com -c channel1 --tls --cafile "${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem"
```
- it will give `config_block.pb` in `channel-artifacts` folder.

## Convert the Configuration to JSON and Trim It Down

- decode this channel configuration block into JSON format

```javascript
configtxlator proto_decode --input config_block.pb --type common.Block --output config_block.json
```

- strip away all of the headers, metadata, creator signatures, and so on that are irrelevant to the change we want to make.
```javascript
jq .data.data[0].payload.data.config config_block.json > config.json
```

## Add the Org3 Crypto Material

- append the Org3 configuration definition – org3.json

```javascript
jq -s '.[0] * {"channel_group":{"groups":{"Application":{"groups": {"Org3MSP":.[1]}}}}}' config.json ../organizations/peerOrganizations/org3.example.com/org3.json > modified_config.json
```

- we have two JSON files of interest – config.json and modified_config.json. The initial file contains only Org1 and Org2 material, whereas the "modified" file contains all three Orgs. At this point it’s simply a matter of re-encoding these two JSON files and calculating the delta.

- translate `config.json` back into a protobuf called `config.pb`

```javascript
configtxlator proto_encode --input config.json --type common.Config --output config.pb
```

- encode `modified_config.json` to `modified_config.pb`
```javascript
configtxlator proto_encode --input modified_config.json --type common.Config --output modified_config.pb
```

- use configtxlator to calculate the delta between these two config protobufs. This command will output a new protobuf binary named `org3_update.pb`
```javascript
configtxlator compute_update --channel_id channel1 --original config.pb --updated modified_config.pb --output org3_update.pb
```

- decode this object into editable JSON format and call it `org3_update.json`

```javascript
configtxlator proto_decode --input org3_update.pb --type common.ConfigUpdate --output org3_update.json
```

- we have a decoded update file – org3_update.json – that we need to wrap in an envelope message. This step will give us back the header field that we stripped away earlier. We’ll name this file org3_update_in_envelope.json
```javascript
echo '{"payload":{"header":{"channel_header":{"channel_id":"'channel1'", "type":2}},"data":{"config_update":'$(cat org3_update.json)'}}}' | jq . > org3_update_in_envelope.json
```

- `org3_update_in_envelope.json` – we will leverage the configtxlator tool one last time and convert it into the fully fledged protobuf format that Fabric requires. We’ll name our final update object `org3_update_in_envelope.pb`
```javascript
configtxlator proto_encode --input org3_update_in_envelope.json --type common.Envelope --output org3_update_in_envelope.pb
```
- we need signatures from the requisite Admin users before the config can be written to the ledger.
- switch to org1 or org2 admin.
```javascript
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_LOCALMSPID="Org2MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/organizations/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp
export CORE_PEER_ADDRESS=localhost:9051
```

## Sign and Submit the Config Update

- `peer channel signconfigtx` command will sign the update
```javascript
peer channel signconfigtx -f channel-artifacts/org3_update_in_envelope.pb
```
- sign the `org3_update_inenvelope.pb`  which require number of approvel before sending to update call

```javascript
peer channel update -f channel-artifacts/org3_update_in_envelope.pb -c channel1 -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com --tls --cafile "${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem"
```

## Join Org3 to the Channel

- Now join the org3 to channel 
- Export the following environment variables to operate as the Org3 Admin:
```javascript
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_LOCALMSPID="Org3MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/organizations/peerOrganizations/org3.example.com/peers/peer0.org3.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org3.example.com/users/Admin@org3.example.com/msp
export CORE_PEER_ADDRESS=localhost:11051
```

- Org3 peers can join channel1 by either the genesis block or a snapshot that is created after Org3 has joined the channel.

- To join by the genesis block, send a call to the ordering service asking for the genesis block of channel1. As a result of the successful channel update, the ordering service will verify that Org3 can pull the genesis block and join the channel. If Org3 had not been successfully appended to the channel config, the ordering service would reject this request.

- Use the peer channel fetch command to retrieve this block:
```javascript
peer channel fetch 0 channel-artifacts/channel1.block -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com -c channel1 --tls --cafile "${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem"
```

- Notice, that we are passing a 0 to indicate that we want the first block on the channel’s ledger; the genesis block. If we simply passed the peer channel fetch config command, then we would have received block 3 – the updated config with Org3 defined. However, we can’t begin our ledger with a downstream block – we must start with block 0.

- If successful, the command returned the genesis block to a file named channel1.block. We can now use this block to join the peer to the channel. Issue the peer channel join command and pass in the genesis block to join the Org3 peer to the channel:

```javascript
peer channel join -b channel-artifacts/channel1.block
```