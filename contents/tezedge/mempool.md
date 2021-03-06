---
title: Mempool
sidebar: Docs
showTitle: false
---


## **The mempool**

Nodes are the pillars of a blockchain network. Each node plays its part in maintaining the network and ensuring consensus about the blockchain’s state. Managing the constant stream of unconfirmed operations that flow across the network is one of a node’s most important duties.

The purpose of the mempool, short for **mem**ory **pool**, is to temporarily store and manage the movement of new operations before they are validated and baked into blocks.

Without a mempool, it would be impossible to control the vast amount of operations that are being constantly made on the Tezos blockchain.

As the number of operations increases, the mempool must manage a greater volume of tasks. One of the most important duties of the mempool is to efficiently manage resources in order to prevent accidental overloading or intentional flooding by adversaries.

To improve overall resiliency, the node uses the actor model, a conceptual model for software architecture in which tasks are performed by individual actors. An actor is a primitive unit of computation that is capable of performing tasks, but is completely separated from other actors and can use various strategies to automatically recover from failure.

### **How operations are processed inside the mempool**

To understand what happens inside the mempool, here is a scenario in which an operation enters and exits the mempool, starting when it is received from the network and ending when it has been baked into a block.



![Image](../../static/images/mempool1.svg "Operations arriving in mempool via network")



**1. A P2P message arrives from the network**

A P2P message of the _CurrentHead_ type arrives from the network.

It is then processed by the[ ChainManager](https://github.com/simplestaking/tezedge/blob/v0.1.0/shell/src/chain_manager.rs#L159), an actor that is responsible for processing all of the messages that come in from the P2P network once a trusted connection has been established.

The _CurrentHead_ contains only the hashes of operations. In order to get the full details of these operations, Alice’s node needs to send _GetOperations_, another type of P2P message. When the network receives the _GetOperations_ message, it replies with _Operation._



![Image](../../static/images/screengrab1.gif)



**2. The ChainManager actor receives the Operation.**

The operation is[ stored](https://github.com/simplestaking/tezedge/blob/v0.1.0/shell/src/chain_manager.rs#L526) in the mempool’s storage. Please note that the mempool’s storage is different from the storage module that contains the current state.

**3. Checking operations from peers**

This triggers a[ CheckMempoolCompleteness](https://github.com/simplestaking/tezedge/blob/v0.1.0/shell/src/chain_manager.rs#L174) which checks whether all mempool operations were received from peers. If they were not received, Alice’s node sends a _GetOperations_ message.

**4. Notifying peers of received operation**

The _ChainManager_ sends a[ MempoolOperationReceived](https://github.com/simplestaking/tezedge/blob/v0.1.0/shell/src/chain_manager.rs#L532) message, which notifies all of the actors that are subscribed to the shell channel that a new operation was received

**5. Messaging the shell channel**

The _MempoolPrevalidator_ receives the[ MempoolOperationReceived](https://github.com/simplestaking/tezedge/blob/v0.1.0/shell/src/mempool_prevalidator.rs#L129) and sends a _ValidateOperation_ message to the shell channel (since the _MempoolPrevalidator_ is subscribed to the shell channel, it receives the message as well)

**6. Inserting into pending operations**

The _MempoolPrevalidator_ processes the received[ ValidateOperation](https://github.com/simplestaking/tezedge/blob/v0.1.0/shell/src/mempool_prevalidator.rs#L344) message and checks whether the operation is stored in the mempool storage and whether the operation was already validated. If the checks are successful, the operation is then inserted into the pending operations, thus modifying the current state of the mempool.

**7. Moving the operation towards validation**

The _MempoolPrevalidator_ then handles the pending operations.



*   The operation is sent to the protocol for validation.
*   The result is then handled and, if needed, the current mempool state is changed, removing the operation from pending and moving it to _applied_, _refused_, *branch_delayed* or *branch_refused*. The applied operations are also known as “known_valid” operations (in the context of peer messages).

We will take a closer look at how operations and blocks are validated in a future article.

### **Ensuring consistency between the Rust and OCaml nodes’ mempool**

_We want to be able to test whether operations are successfully and correctly propagated from the native OCaml node’s mempool into the TezEdge node’s mempool. To do this, we run the following test:_

First we run two nodes; the TezEdge node and the OCaml node, both are run in sandbox mode.

Using the Tezos-admin-client, we create a connection between the two nodes.

Using the Tezos-client, we activate a protocol inside the OCaml node, thus creating the first block.

We wait until TezEdge synchronizes with the OCaml node.

We call each node with the *pending_operations* RPC and compare the return values. We want both values to be empty, which means their mempools are empty.

Using a Tezos client, we inject a valid transaction into the OCaml node. We call each node with the *pending_operations* RPC again and compare the return values.

Again, we should see the same value from calling both nodes, but this time it will not be empty. We can see operation in the _applied_ field. This means that the transaction has been successfully propagated from the OCaml node into the TezEdge node.

_[Click here to see the results](http://ci.tezedge.com/simplestaking/tezedge/808/2/10) of our own continuous integration (CI) test._



![Image](../../static/images/screengrab2.gif)

## Adding operations via RPCs

### **The blockchain sandbox**

The blockchain is a high stakes environment. Once smart contracts are deployed on the live network, there is no turning back, and faulty code or errors may cause enormous financial damage or other serious real world consequences.

It is useful for developers to work in an environment which they can fully control. One in which they can set the parameters, create funds for testing purposes and try out the features of the node’s various modules, for example the mempool.

For this reason, we are creating the Tezos sandbox, an offline tool that simulates the Tezos blockchain. This allows you to safely develop, test and deploy your and smart contracts without needing to connect to the Tezos network, or even the internet itself.

We made use of the sandbox while developing the TezEdge node’s mempool.

When an operation is injected into the node’s **mempool**, there are two possible points of origin:



*   From other nodes in the network
*   Via **RPCs** from the node itself



![Image](../../static/images/mempool2.svg "Operations arriving in mempool via RPCs")



Since there currently is no client for sending custom messages to the node via the P2P network, we make use of remote procedure calls (RPCs) that allow us to inject operations that are created locally into the node’s mempool.


### **Using CI tests to demonstrate operation injection via RPCs**

The objective here is to demonstrate that operations are injected into the TezEdge node’s mempool via RPCs and are then broadcasted to other nodes (including OCaml nodes) across the Tezos network. We utilize **continuous integration (CI)** tests as proof of this mechanism.

CI is a practice in software development used to continuously check the quality of new changes made to a project. CI tests ensure the changes proposed by each pull request will not cause errors or otherwise endanger the software.

By using CI, we can easily track the exact moment where development goes wrong, meaning that we can quickly find out which pull request contained the faulty code and thus avoid merging it with the main branch.


### **Testing operation injection into the mempool and broadcasting between nodes**

When we run the sandbox, the genesis block already exists, which means we now have to activate the protocol. The Tezos client creates the first block and injects it into the Rust node, where it activates the protocol. From there, the block is broadcasted to the OCaml node where it also activates the protocol. Once the protocol is activated, we test the injection of operations.

Here you can see the aforementioned CI tests:

[https://github.com/simplestaking/tezedge/blob/master/.drone.yml#L99](https://github.com/simplestaking/tezedge/blob/master/.drone.yml#L99)

This test is similar to the one we described in our previous article. The difference is that now we can demonstrate the injection of the first block and an operation into the TezEdge node.

**1.** First, we run two nodes; the TezEdge node (_[tezedge-node-sandbox-run](https://github.com/simplestaking/tezedge/blob/master/.drone.yml#L119)_) and the OCaml node(_[ocaml-node-sandbox-run](https://github.com/simplestaking/tezedge/blob/master/.drone.yml#L146)_) , both are run in sandbox mode. This is done in the first four steps in the CI pipeline. After each run step there is a so-called *wait-for* step which ensures that the pipeline is held until each node has started successfully.

**2.** Using the Tezos-admin-client, we create a connection between the two nodes. You can see this in the_[ connect-ocaml-and-rust step](https://github.com/simplestaking/tezedge/blob/master/.drone.yml#L167)_

**3.** In the[ next step](https://github.com/simplestaking/tezedge/blob/master/.drone.yml#L180), we prepare the tezos-client. This means including the accounts used in the protocol activation and the transfer operation. Then, using the Tezos-client, we activate a protocol inside the TezEdge node, thus creating the first block. This is a distinct block that contains a block header and the field “content” in which there are subfields such as “command”, “hash”, “fitness” and “protocol_parameters”.


![Image](../../static/images/screengrab3.gif)

[http://ci.tezedge.com/simplestaking/tezedge/955/2/6](http://ci.tezedge.com/simplestaking/tezedge/955/2/6)

During the step [*wait-for-sync-on-level-1*](https://github.com/simplestaking/tezedge/blob/master/.drone.yml#L193), we wait until the OCaml node synchronizes with the TezEdge node which has the newly injected block on level 1.

**4.** The next step is a check to ensure that both nodes have an empty mempool. We call each node with the pending_operations RPC and compare the return values. We want both return values to be empty, which means their mempools are empty.

**5.** Using the Tezos client, we inject a valid transaction into the TezEdge node. This is demonstrated in step[ do-transfer-with-tezos_client](https://github.com/simplestaking/tezedge/blob/master/.drone.yml#L219)_._



![Image](../../static/images/screengrab4.gif)

[http://ci.tezedge.com/simplestaking/tezedge/955/2/9](http://ci.tezedge.com/simplestaking/tezedge/955/2/9)

To help you understand how RPCs are used to ‘inject’ operations into the mempool, we will explain through a hypothetical Inject Operation.

This is done by the tezos-client (via RPCs)

**5.1** Collecting data from the node

**5.2** Simulating the operation

**5.3** Running the pre-apply operation

**5.4** Injecting the transaction

**5.5** After injection, inside the node:

**5.5.1** Messaging the shell channel

**5.5.2** Inserting into pending operations

**5.5.3** Moving the operation towards validation

**5.5.4** Broadcasting the new mempool state

**6.** In the step[*check-mempool-after-transfer*](https://github.com/simplestaking/tezedge/blob/master/.drone.yml#L231), we call each node with the pending_operations RPC again and compare the return values. Again, we should see the same value from calling both nodes, but this time it will not be empty. We can see the operation in the applied field. This means that the transaction has been successfully propagated from the TezEdge node into the OCaml node.


![Image](../../static/images/screengrab5.gif)

[http://ci.tezedge.com/simplestaking/tezedge/955/2/10](http://ci.tezedge.com/simplestaking/tezedge/955/2/10)

You can see the results of the entire process here:

[http://ci.tezedge.com/simplestaking/tezedge/955/2/1](http://ci.tezedge.com/simplestaking/tezedge/955/2/1)


### **Using multiple CPU cores to pre-validate operations**

When networks handle financial movements or other data that affects items of value, they must have an accurate, fast and efficient system for either approving valid messages or rejecting invalid ones. Without such a system in place, the network would be unable to reliably assess data, making it unsuitable for handling such information.

1. First and foremost, the validation system must be accurate, with no room for error as approving a malicious or erroneous message can lead to serious real-world consequences, including financial loss.
2. Second, it must be fast. Speed is of the essence — when a network expands, the number of actions requiring validation increases, but waiting times must be kept short in order to provide satisfactory service to users.
3. Third, it needs to be efficient in the use of its resources in order to handle an ever-expanding volume of data. It should be able to prevent accidental overloading or intentional flooding by adversaries.

In the Tezos blockchain, all messages sent across the peer to peer (P2P) network (including transactions) are known as operations. The **mempool** plays a central role in the validation of new operations. The mempool manages and holds operations before they are validated and baked into blocks.

**Validation layer for operation injection**

New operations are injected into the mempool’s pending operations section, either from:



1. the node itself, via dedicated remote procedure calls (RPCs) that are called with the wallet or the tezos-client, or
2. other peers (Tezos nodes), via the P2P network.

_Note that operations can arrive **parallelly** from the RPC/P2P._

The mempool contains the accumulated state of all operations that have been subjected to (protocol) validation, which is known as the `validation_state`.

Operations are sorted into each of these categories `applied`, `refused`, `branch_refused`, `branch_delayed`, all of which are based on the results of their validation with the protocol. Note that the mempool also contains `unprocessed`, however, this category does not influence the `validation_state`.

In the background, the mempool uses one instance of the [OCaml FFI runtime](https://caml.inria.fr/pub/docs/manual-ocaml/runtime.html), which contains the accumulated `validation_state`. The current state of the mempool can be viewed via this RPC:
```
/chains/main/mempool/pending_operations
```
**Scaling pre-validation performance via multiple CPU cores**

We want to scale up our validation system, improve its performance and avoid flooding the mempool.

To achieve that, we needed to be capable of scaling and parallelizing **validations** by utilizing multiple CPU cores.

We have performed pre-validations on operations (from both RPC and P2P). When an operation arrives at either the[ RPC](https://github.com/simplestaking/tezedge/blob/v0.7.0/rpc/src/services/mempool_services.rs#L145) or[ P2P](https://github.com/simplestaking/tezedge/blob/v0.7.0/shell/src/chain_manager.rs#L604), we first send it into **pre-validation**. The[` pre-validation`](https://github.com/simplestaking/tezedge/blob/v0.7.0/shell/src/validation/mod.rs#L161) calls `begin_construction` and `apply_operation` on the OCaml FFI runtime. Each operation is individually validated with the protocol against the context. The context is dependent on the current mempool’s block.

Following this pre-validation, the mempool then validates operations. From the RPCs, only `applied` operations are allowed to pass to mempool validation, while from the P2P, all of the other categories may pass except for `refused`. This may change in future versions.

The main feature is that we can parallelize the pre-validations thanks to having parallelized access to the **OCaml FFI runtime** through dedicated **OCaml FFI runtime pools.** This is achieved by using the `ocaml-interop` (which we described in our[ previous article](https://medium.com/simplestaking/safely-mixing-rust-and-ocaml-in-the-tezedge-node-a2089cbef590?source=collection_home---6------0-----------------------)), as well as the dedicated pool `tezos_readonly_prevalidation_api`. For the pools, we implemented a[ version](https://github.com/simplestaking/tezedge/blob/v0.7.0/tezos/wrapper/src/pool.rs#L140) that uses the [r2d2 library](https://crates.io/crates/r2d2).

**Try out multi-CPU core support for the pre-validation layer**

1. Checkout git & go to directory

```
git clone [https://github.com/simplestaking/tezedge.git](https://github.com/simplestaking/tezedge.git)

cd tezedge
```

2. Please open this link, follow the instructions and check that you have all the prerequisites necessary for building the node from the source.

[https://github.com/simplestaking/tezedge#building-from-source](https://github.com/simplestaking/tezedge#building-from-source)

3. Start the sandbox launcher. We use the launcher to remotely manage the sandbox node. Wait until the launcher is ready.

`./run.sh sandbox`

4. Start the node in sandbox mode.

`./test.sh start_sandbox_node`

5. Run a stress test. We create several accounts (wallets) that inject multiple transactions to stress test the pre-validation layer.

`./test.sh send_transactions`

6. You can now see how the sub-processes are scaled on multiple CPU cores:

`htop`


![Image](../../static/images/multicpu1.gif)


