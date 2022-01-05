- we will fetch the latest channel configuration to get started. Fetch the most recent config block for the channel, using the `peer channel fetch` command.

```javascript
peer channel fetch config channel-artifacts/config_block.pb -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com -c channel1 --tls --cafile "${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem"
```
- we will get `config_block.pb` file into `channel-artifacts` folder.

- When converting it we need to remove all the headers, metadata, and signatures that are not required to update Org3 to include an anchor peer by using the jq tool. 

```javascript
configtxlator proto_decode --input config_block.pb --type common.Block --output config_block.json
jq .data.data[0].payload.data.config config_block.json > config.json
```

- The config.json is the now trimmed JSON representing the latest channel configuration that we will update.

-  we will update the configuration JSON with the Org3 anchor peer we want to add.

```javascript
jq '.channel_group.groups.Application.groups.Org3MSP.values += {"AnchorPeers":{"mod_policy": "Admins","value":{"anchor_peers": [{"host": "peer0.org3.example.com","port": 11051}]},"version": "0"}}' config.json > modified_anchor_config.json
```

- We now have two JSON files, one for the current channel configuration, config.json, and one for the desired channel configuration modified_anchor_config.json

- convert each of these back into protobuf format and calculate the delta between the two.

- Translate config.json back into protobuf format as config.pb

```javascript
configtxlator proto_encode --input config.json --type common.Config --output config.pb
```

- Translate the modified_anchor_config.json into protobuf format as modified_anchor_config.pb

```javascript
configtxlator proto_encode --input modified_anchor_config.json --type common.Config --output modified_anchor_config.pb
```

- Calculate the delta between the two protobuf formatted configurations.

```javascript
configtxlator compute_update --channel_id channel1 --original config.pb --updated modified_anchor_config.pb --output anchor_update.pb
```

- Now that we have the desired update to the channel we must wrap it in an envelope message so that it can be properly read. To do this we must first convert the protobuf back into a JSON that can be wrapped.

- We will use the configtxlator command again to convert anchor_update.pb into anchor_update.json

```javascript
configtxlator proto_decode --input anchor_update.pb --type common.ConfigUpdate --output anchor_update.json
```

- we will wrap the update in an envelope message, restoring the previously stripped away header, outputting it to anchor_update_in_envelope.json

```javascript
echo '{"payload":{"header":{"channel_header":{"channel_id":"channel1", "type":2}},"data":{"config_update":'$(cat anchor_update.json)'}}}' | jq . > anchor_update_in_envelope.json
```

- we have reincorporated the envelope we need to convert it to a protobuf so it can be properly signed and submitted to the orderer for the update.

```javascript
configtxlator proto_encode --input anchor_update_in_envelope.json --type common.Envelope --output anchor_update_in_envelope.pb
```

- Now that the update has been properly formatted it is time to sign off and submit it.
- Since this is only an update to Org3 we only need to have Org3 sign off on the update. Run the following commands to make sure that we are operating as the Org3 admin:
```javascript
export CORE_PEER_LOCALMSPID="Org3MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/organizations/peerOrganizations/org3.example.com/peers/peer0.org3.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org3.example.com/users/Admin@org3.example.com/msp
export CORE_PEER_ADDRESS=localhost:11051
```

- We can now just use the peer channel update command to sign the update as the Org3 admin before submitting it to the orderer.
```javascript
peer channel update -f channel-artifacts/anchor_update_in_envelope.pb -c channel1 -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com --tls --cafile "${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem"
```