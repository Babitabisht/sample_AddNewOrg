# Hyperledger Fabric network setup to add a new org in existing running network  

Pre Requisite - Hyperledger Binaries and HLF Pre-Requisites software are installed

# Following are the steps to run the setup
1. create a working folder, change directory to working folder
2. git clone https://github.com/ashwanihlf/sample_AddNewOrg.git
3. sudo chmod -R 755 sample_AddNewOrg/
4. cd sample_AddNewOrg  
5. mkdir config  
	<remove config and crypto-config if they are existing before creation of config folder (Optional)>
	5a. sudo rm -rf config
	5b  sudo rm -rf crypto-config

6. export COMPOSE_PROJECT_NAME=net
7. sudo ./generate.sh
8. sudo ./start.sh
9. docker exec -it cli /bin/bash
10. peer chaincode invoke -C mychannel -n samplecc -c '{"function":"initCar","Args":["Ashwani","Blue","BMW"]}'
11. exit
11. docker exec -e "CORE_PEER_ADDRESS=peer0.org2.example.com:7051" -e "CORE_PEER_LOCALMSPID=Org2MSP" -e "CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp" cli peer chaincode query -C mychannel -n samplecc -c '{"function":"readCar","Args":["Ashwani"]}'      
12. sudo chmod -R 755 config
13. cd Org4
14. sudo cryptogen generate --config=./org4-crypto.yaml
15. export FABRIC_CFG_PATH=$PWD
16. sudo configtxgen -printOrg Org4MSP > org4.json
17. sudo chmod 755 org4.json
18. sudo cp org4.json ../config/
19. cd ../ && sudo cp -r crypto-config/ordererOrganizations Org4/crypto-config/
20. docker exec -it cli bash
21. export CHANNEL_NAME=mychannel
22. peer channel fetch config config_block.pb -o orderer.example.com:7050 -c $CHANNEL_NAME
23. configtxlator proto_decode --input config_block.pb --type common.Block | jq .data.data[0].payload.data.config > config.json
24. jq -s '.[0] * {"channel_group":{"groups":{"Application":{"groups": {"Org4MSP":.[1]}}}}}' config.json ./channel-artifacts/org4.json > modified_config.json
25. configtxlator proto_encode --input config.json --type common.Config --output config.pb
26. configtxlator proto_encode --input modified_config.json --type common.Config --output modified_config.pb
27. configtxlator compute_update --channel_id $CHANNEL_NAME --original config.pb --updated modified_config.pb --output org4_update.pb
28. configtxlator proto_decode --input org4_update.pb --type common.ConfigUpdate | jq . > org4_update.json
29. echo '{"payload":{"header":{"channel_header":{"channel_id":"mychannel", "type":2}},"data":{"config_update":'$(cat org4_update.json)'}}}' | jq . > 	    	org4_update_in_envelope.json
30. configtxlator proto_encode --input org4_update_in_envelope.json --type common.Envelope --output org4_update_in_envelope.pb	
31. peer channel signconfigtx -f org4_update_in_envelope.pb
32. export CORE_PEER_LOCALMSPID="Org2MSP"
33. export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp
34. export CORE_PEER_ADDRESS=peer0.org2.example.com:7051
35. peer channel signconfigtx -f org4_update_in_envelope.pb
36. export CORE_PEER_LOCALMSPID="Org3MSP"
37. export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org3.example.com/users/Admin@org3.example.com/msp
38. export CORE_PEER_ADDRESS=peer0.org3.example.com:7051
39. peer channel update -f org4_update_in_envelope.pb -c $CHANNEL_NAME -o orderer.example.com:7050
Open a new terminal 
40. docker-compose -f docker-compose-org4.yaml up -d
41. docker exec -it Org4cli bash
42. export CHANNEL_NAME=mychannel
43. peer channel fetch 0 mychannel.block -o orderer.example.com:7050 -c $CHANNEL_NAME
44. peer channel join -b mychannel.block
45. peer chaincode install -n samplecc -v 1.0 -p github.com/ >&installlog.txt
46. peer chaincode query -C mychannel -n samplecc -c '{"function":"readCar","Args":["Ashwani"]}'    