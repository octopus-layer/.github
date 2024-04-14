# Octopus Layer

## A Fully Decentralized Chain Agnostic ZK Snark Proof Verification Layer

Octopus creates a decentralized and censorship resistant verification layer for ZK snark proofs that settles in Near and distributes ZK proofs to all blockchains through Near Multichain.

The layer is implemented as a light and censorship resistant network, thus provides proof availability when integrated into a blockchain.

As Near is very strong as a consensus and settlement layer, Near Multichain creates the perfect way of distributing settled proof verifications across the blockchain.

**Contract Addresses:**

[Near Verification Contract Testnet](https://testnet.nearblocks.io/txns/7FzGA2PLAtpBxiSLZKjZ4xN88ibEkGiPmc3Rrqix7SzL)

[Near Application Contract Testnet](https://testnet.nearblocks.io/txns/7fWvas97xAyu9YoVWhkgDRSRVw2H4XwF48sB4orkZ6Xr) 

## Background

ZK snark proofs are the only way of providing fully on chain privacy in constant verification time. However, the proof generation time and the proof size depends on the input.

o1js is a ZK snark proof generation framework in JS ([npm package](https://www.npmjs.com/package/o1js)) commonly used in the Mina blockchain. As it is essentially JS, it can be used in all machines supporting wasm. Moreover, o1js uses Plonk proofs on top of a layer called Pickles, which is responsible for proof recursion. As a result, proof aggregation is much more efficient in o1js. Developers may easily use o1js to create complex recursive ZK proofs.

Near is a chain abstraction layer with a very strong consensus. Through its Multichain technology, it allows developers to push TXs in any chain through their Near contract. However, computation is limited on the Near network, and it is not an ideal solution to store all of the data associated with a ZK proof verification in a Near contract.

## Problem Overview

Decentralized networks cannot provide full privacy without using ZK snark proofs. However, ZK snark proofs are quite complex to integrate and often costly to generate. Moreover, even though the verification time of a snark proof is always constant, the proof size may increase TX costs significantly.

In order to solve this issue, a method called proof aggregation can be utilized, where multiple ZK proofs are recursively combined together and proved at once. As ZK snark proofs are always verified in constant time, recursively combining proofs do not increase time costs. However, aggregated proof generation takes significant time in the client side, and thus affects the UX of a ZK application. Common ZK snark proof generation technologies are usually not usable with proof aggregation.

Combined with the limited computation power inside Near contracts and the complex underlying structure of proof verification, it becomes very difficult to provide on chain privacy on Near.

## Solution

**Important:** Please note that for now, only the proof of concept of the project is ready and not all of the functionality is implemented. However, the most important part of the architecture, Octopus layer verifier nodes are fully available for usage.

Instead of directly verifying ZK snark proofs in Near, Octopus implements a light o1js proof verification layer on top of Near. This layer is made up of light verifier nodes.

Verifier nodes are implemented as nodeJS servers that can access o1js and Near at the same time. The details of these verification nodes are as following:

- They are responsible for verifying o1js proofs and signing proofs public output with their Near private key. As snark proof verification has constant time complexity, this process is very light, thus making these nodes light as well: Octopus verifier nodes can be installed in machines with **1 core and 2 GB RAM**.

- The motivation of nodes is to sign a new coming proof as fast as possible and settle it in Near once 66% of all possible signatures are gathered (66% as a common prevention against possible majority attack vectors).

- Instead of every node signing and settling on Near one by one, it is enough **only one node** to send a settlement TX on Near with 66% of signatures. This decreases computation costs on Near significantly and optimize the usage of the consensus.

- Nevertheless, nodes do not compete to settle the proof, as all nodes that provided a signature during the settlement are **rewarded equally** once the settlement is complete. This makes the need of a consensus layer inside Octopus dissapear.

- As it is for the best of all verifier nodes to settle as fast as possible, they try reaching each other in the most optimized manner. This creates a **perfect game theory** and makes sure the settlement process will be complete in the smallest time possible.

- Moreover, nodes do not need to store anything. The URL of other nodes is provided from the Near contract. In a way, Near contract acts as a **communication layer** between the verifier nodes of Octopus. As a result, there is **no synchronization** during the installation of a new Octopus verifier node.

- The layer is instantiated with an initial set of verifier nodes, but anyone can instantly join or leave the network. In order to join the network, you need 66% signatures of current verifier nodes. An additional incentivization mechanism can be provided here to make sure the node count never goes below a certain level.

- As a result of all these properties, Octopus nodes can be installed in a browser during the usage of an decentralized application, just like a data availability chain, to provide 100% **censorship resistant proof verification**.

- In the future, if Near becomes a data availability layer by implementing light clients, then Octopus layer will also become **100% available**. Direct access to the layer will be possible without the need of an RPC.

## Architecture Diagram

Details of the diagram are explained below

![Octopus Layer Architecture](./architecture.png)

## Architecture Parts

### Verifier Contract

`Verifier Contract` is a Near contract deployed as a part of Octopus layer. It is responsible for keeping track of the current state of Octopus verifier nodes, adding a new verifier node when signatures are provided, and providing the current URL of any Octopus verifier node.

### Verifier Node

`Verifier Node` is a nodeJS light server that can be deployed anywhere. It communicates with other `Verifier Node`s through their public URL, which is provided by Near (to achieve full liveness in the system). Its job is to verify an o1js proof by asserting it to the public input in the `Application Contract`, and then signing the proof with its Near private key.

Once a proof is signed, it is gossiped to other `Verifier Node`s in the layer or settled in Near if 66% of the list is achieved.

### Application Contract

`Application Contract` is application specific and differs from project to project. If an application on top of Near wants to use the Octopus Layer, their `Application Contract` must follow a specific pattern (as in the example provided with this project).

The `Application Contract` is required to provide a public input in the state (generally a list of hashes, or a merkle tree root for more optimized data storage usage) and to implement a settlement function. The settlement function reaches the `Verifier Contract` to verify the validity of signatures as in the example.

Please note that the verification happening in Near is not the proof verification, but only signatures are checked to make sure the provided public output is trustless. If 66% is reached, then the public output is directly set to the state. This process is referred as the "settlement of the ZK proof".

The `Application Contract` may also implement Near Multichain, just like in the example provided, to pass authenticated ZK proof outputs into other chains.

### Other Chain Contract (e.g. Ethereum)

If Near Multichain is used, then the corresponding chain must also host a contract to be interacted by Near.

## Process for Deployment

Here, we decribe the process of the deployment of a private dApp on top of Near using the Octopus layer. As an example, we create a whistleblower application on Ethereum.

1. First, a developer uses o1js to create ZKPs needed in its dApp. As Near also supports JS for smart contract creation, this is very convenient for Near developers.

2. Then, we compile the ZK proof and obtain the verification key of the proof locally. We add the verification key of the proof in our `Application Contract` during the deployment. If the ZK proof is changed, then the verification key kept on Near should be as well.

3. In the example of whistleblower, we need to verify that some set of `{email, password}` pair's hash is in a certain list. By asserting the hash of this pair to be in the on chain pair, we make sure that only authenticated users can submit a message, without revealing who in the set sent the message at all.

4. We deploy our `Application Contract` by including an empty message list array, the verification key, and the list of `{email, password}` hash pair.

5. Now, the contract is ready to receive ZK proof verified updates on its message list.

6. Please note here that the Multichain interaction is handled by the application developer. The application developer should modify its contract accordingly to use ZK proofs in another chain.

## Process for Interaction

Here, we describe how a user interacts with a Near dApp using Octopus layer for privacy. Again, we continue with the example of whistleblower application.

1. The user provides their `{email, password}` pair and generate a ZK snark proof. The proof configuration is as following:

- Private Inputs: email, password, message

- Public Input: The list of `{email, password}` pair hashes that is kept on Near.

- Public Output: message

2. Then, user receives the location of Octopus `Verifier Node`s from the Near `Verifier Contract`. It submits a ZK verification TX to any of these nodes (or all of them at once, this does not affect the architecture).

3. Once a `Verifier Node` receives a proof, it verifies its correctness by asserting on the public input. The trustless public input is obtained from the `Application Contract` on Near. This makes these proofs to be considered as verified in the Near.

4. It is also important to verify the verification key of the proof to match the signature stored in the `Application Contract`. Once the verification key and the public input is matched, there is no more doubt on the public output of the prooof.

5. The now verified proof is gossiped to other `Verifier Node`s to achieve 66% of signatures as fast as possible. If 66% of signatures already exist in the TX, then the settlement happens on the `Application Contract`.

6. `Application Contract` checks for the validity of signatures by receiving the list of Octopus layer node keys from the `Verifier Contract`. If all signatures match and they pass the 66% limit, then the public output of the proof is included in the state of the contract.

7. The Near Multichain allows developers to update another chain in the step 6 as well as the proper state of their `Application Contract`. As the ZK proofs are verified by a decentralized light set based on the assertions on the `Application Contract`, the ZK proof can be considered to be verified in Near without loss of generality.

## Process for Adding a New `Verifier Node`

This process is completely separate from the example whistleblower application. It describes how new nodes are included in the state of the `Verifier Contract`.

1. The new `Verifier Node` installs the nodeJS application on its device and starts the server.

2. Then, it requests the location of other Octopus `Verifier Node`s from the Near `Verifier Contract`.

3. It submits its URL and public key into the network.

4. Similarly to the ZK proof settlement, the request is signed and gossiped among the `Verifier Node`s. Once the 66% of signatures is obtained, the new node is submitted into the `Verifier Contract`.

5. Now the new `Verifier Node` is a part of the Octopus Layer.

## Next Steps & Improvements

Some possible missing points and improvements on the system are as following:

- The Multichain part is not fully implemented yet and not working as expected. It should be finished before this project is fully ready.

- As this is a proof of concept project, the incentivization of `Verifier Node`s is not implemented. It should be added as described in the architecture.

- Instead of keeping a list of all public keys, keeping a merkle root on the `Verifier Contract` is much more efficient, but then an off chain storage solution should also be implemented to keep the liveness on its maximum.

- The joining process of a new node may be modeled better in terms of tokenomics to make sure the incentives keep the set at a specific size: The set should be big enough to always provide trustlessness, but also should be as small as possible to decrease the proof settlement time to its minimum.

- Wrapping all of the functionality inside a single NPM package is necessary to bring the best developer experience possible to chain agnostic ZK proof verification. This is not hard to achieve, but not included in this proof of concept.
