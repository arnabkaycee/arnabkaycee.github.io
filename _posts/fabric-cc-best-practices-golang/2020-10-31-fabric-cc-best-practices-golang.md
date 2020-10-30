---
title: Hyperledger Fabric GoLang Chaincode - Best Practices
date: 2020-10-31 00:00:00 +05:30
tags: [blockchain, fabric, coding, smart-contract, best-practices]
description: Some best practices learnt while coding Smart contracts in Fabric with GoLang
---

I believe smart-contracts (chaincodes) are the heart of any Blockchain Network. Written correctly can yeild many benefits and truly bring out the best in a secure blockchain network and have distratous effects if written inefficiently. Although I will not discuss about any specific goods and bads of Chaincode design, I will touch down a few guidelines which I experienced while doing some POC applications during my work. Without further ado, let's start.

Below are a few general guidelines / caveats that can be adhered to (although there are exceptions) while writing chaincodes. These I have particularly written for chaincodes written for Hyperledger fabric v.1.0 network in golang.  But, I believe they can be extrapolated to chaincodes written in any language for Hyperledger Fabric. 

# Use Chaincode DevMode

To kickoff your development process, use the DevMode. I cannot emphasise enough, how important it is to use dev mode and save time and effort in changing your code and restarting the network again and again.

Reference : https://github.com/hyperledger/fabric-samples/tree/release/chaincode-docker-devmode

*P.S. - Although the tutorial is for Golang, using it for other languages should not be different except for building the chaincode part.* 

# Use Chaincode Logging
Well, this is perhaps the first useful thing that you can do to debug your chaincode and find bugs quickly. Using logging is simple and easy. Use Fabric's inbuilt logger. Fabric provides logging mechanism as follows:

**For Golang:**
https://godoc.org/github.com/hyperledger/fabric/core/chaincode/shim#ChaincodeLogger
**For NodeJS:**
https://fabric-shim.github.io/Shim.html#.newLogger__anchor
**For Java:**
You can use any standard logging framework like Log4J

# Avoid using Global Keys

While writing chaincode, we often find our hands tied when finding data. To keep track of keys registered in the Key Value Store, we try and use some sort of Global Collection.

For example, when keeping track of registered marbles in your application, you might want to make a global counter and keep counting the number of all the marbles and generate the next ID of the marble too. But while doing so, you are introducing dependency on a single variable to write to when you add a new user.
This might not seem a problem at first, but underlying is a bug waiting to surprise you when you do concurrent transactions. How? Let me explain.

Consider this code :

```go
    package main
	import (
		//other imports
		"github.com/hyperledger/fabric/core/chaincode/shim"
	  	 pb "github.com/hyperledger/fabric/protos/peer"
	)

	//DON'T DO THIS	
    totalNumberOfMarbles := 0
    
    func (t *SimpleChaincode) initMarble(stub shim.ChaincodeStubInterface, args []string) pb.Response {
	    var err error
	    
		marbleId := fmt.Sprintf("MARBLE_%06d",totalNumberOfMarbles)
		marbleName := args[0]
		color := strings.ToLower(args[1])
		owner := strings.ToLower(args[3])
		size, err := strconv.Atoi(args[2])
		
		//other code to initialize
		objectType := "marble"
		marble := &marble{objectType, marbleId, marbleName, color, size, owner}
		
		//--------------CODE SMELL----------------
		//BIG source of Non-determinism as well as performance hit.
		totalNumberOfMarbles = totalNumberOfMarbles + 1 
		//--------------CODE SMELL----------------
		

		//regular stuff...		
		err = stub.PutState(marbleId, marbleJSONasBytes)
		if err != nil {
			return shim.Error(err.Error())
		}
    }
```


Well. Why do I hate this?

**Reason 1:** Consider you have written this code and all is going well and one fine day, one of the peers running this chaincode, crashes. Well, the ledger data is still there, but something dreaded has happened behind the scenes. You might start the peer and everything might seem normal at first. But suddenly all transactions that this peer was endorsing started to fail. Why? You had started the peer. But wait, the global counter variable you had kept, has now lost track of the last counted value. All the peers had counted till say 15K and this peer suddenly starts counting from zero. And your `marbleId` starts giving you IDs from zero again. 
So, when you send this transaction to the orderer and it reaches the committing peer, the Validation system on the committing peer compare the proposal responses from all the endorsed transactions as well as checks if there are sufficient signatures present as per the endorsement policy when the chaincode was instantiated. If there is a single proposal response which does not match, it throws an ENDORSEMENT_POLICY_FAILURE.


**Reason 2:**  Well lets try and solve the above problem by adding the statement `stub.PutState("marble_count", totalNumberOfMarbles)` at the end. Is it any better? NO.  

Consider two concurrent transactions trying to insert a marble. 

For example, one transaction is updating the value of `marble_count` to `34` with a `new_version(marble_count) = 10` and another to `35` again with a `new_version(marble_count) = 10`. Remember, since they are concurrent both transactions see that the `current_version(marble_count) = 09`.  

Now one transaction will reach the orderer before the other and the key `marble_count` will have already updated it to a new value with `current_version(marble_count) = 10`. Therfore the transaction that arrives later will fail because the `current_version(marble_count) = 10`  now, and the late transaction was supposed to read version `09` and update it to version `10`. This is a classical problem of **double spending**.

Hyperledger Fabric uses an Optimistic Locking Model while committing transactions. As I have explained, that first the proposal responses are collected from the endorsing peers by the client and then sent to the Orderer for ordering and finally orderer delivers it to the Committing Peers. 
In this two stage process, if some versions of the keys that you had read in the Endorsement has changed till your transactions reach the  committing stage, you get an MVCC_READ_CONFLICT error. This often is a probability when one or more concurrent transactions is updating the same key.

You can read more here: https://medium.com/wearetheledger/hyperledger-fabric-concurrency-really-eccd901e4040

**[P.S. This is also applicable even when you are not doing concurrent transactions but your block determination criteria is such that one or more transactions updating the same key is landing up in the same block. Because, transactions in Hyperledger Fabric are not committed until the block is committed.]**

# Use Couch DB Queries wisely

Couch DB queries [a.k.a Mongo Queries] is such a boon when searching for data in the Key Value store. But there are a few caveats one needs to take care.

## Couch DB Queries DO NOT alter the READ SET of a transaction.

Mongo Queries are for querying the Key Value store aka StateDB only. It does not alter the read set of a transaction. This might lead to phantom reads in the transaction.

## Only the DATA that you have stored in the couchDB is searchable

Do not be tempted to search for a key by its name using the MangoQuery. Although you can access the Fauxton console of the CouchDB, you cannot access a key by querying a key by which it is stored in the database. Example : Querying by **channelName\0000KeyName** is not allowed. It is better to store your key as a property in your data itself.

# Write Deterministic Chaincode

Never write chaincode that is not deterministic. It means that if I execute the chaincode in 2 or more different environments at different times, result should always be the same, like setting the value as the current time or setting a random number.
For example: Avoid statements like calling  `rand.New(...)` , `t := time.Now()` or even relying on a global variable (check ) that is not persisted to the ledger.

This is because, that if the read write sets generated are not the same, the Validation System chaincode might reject it and throw an ENDORSEMENT_POLICY_FAILURE.

# Be cautions when calling Other Chaincodes from your chaincode.

Invoking a chaincode from another is okay when both chaincodes are on the same channel. But be aware that if it is on the other channel then you get only what the chaincode function returns (only if the current invoker has rights to access data on that channel). NO data will be committed in the other channel, even if it attempts to write some. Currently, cross channel chaincode chaincode invocation does not alter data (change writesets) on the other channel. So, it is only possible to write to one channel at a time per transaction.

# Remember to Set Chaincode Execution Timeout

Often it might so happen that during high load your chaincode might not complete its execution under 30s. It is a good practice to custom set your timeout as per your needs. This is goverened by the parameter in the core.yaml of the peer. You can override it by setting the environment variable in your docker compose file :

*Example:* CORE_CHAINCODE_EXECUTETIMEOUT=60s

#  Refrain from Accessing External Resources

Accessing external resources (http) might expose vulnerability and security threats to your chaincode. You do not want malicous code from external sources to influence your chaincode logic in any way. So keep away from external calls as much as possible.



----------
Official Golang Interfaces Definitions for different Functions: https://godoc.org/github.com/hyperledger/fabric-chaincode-go/shim

----------


