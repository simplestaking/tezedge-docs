
### **The TezEdge Debugger**

Whenever you are developing a node in OCaml or Rust and your software runs into a bug, an error or an otherwise undesired state, you want to be able to identify and locate the instance when and how the problem happened.

However, when you are working with an encrypted network, such as the Tezos peer to peer (P2P) network, locating the problem is not so easy. The messages must first be decrypted and deserialized before you can make sense of them.

During the development of the TezEdge node, we wanted to have a debugger tool that could operate equally well with OCaml as well as Rust. In order to thoroughly debug and simulate any state in the node, we need to access the P2P network, RPC endpoints and the storage.

For this purpose, we’ve created the TezEdge Debugger, a tool that records messages sent and received by your Tezos node via the P2P and RPC layers, decrypts and (if necessary) deserializes them, and finally providing them in a human-readable format in its own user interface (UI).

Through the TezEdge Debugger’s UI, users can view:



*   Who sent the P2P message(s) to your node
*   The content of the message(s)
*   The time when they were made
*   RPCs made in connection with your node (both requests and responses)

All of this information is recorded in the storage, allowing users to examine all of the aforementioned details whenever they want to.

The TezEdge Debugger is node-agnostic, meaning that it will run on any kind of Tezos node, whether it is the native OCaml version, the Rust-based TezEdge node or other alternatives.

As this is an early version, certain features, including full text search and pagination, have yet to be developed. We plan on adding them in future releases.


### **How it works**

### **How it works**

The core function of the debugger is to intercept all network communication made between the node (local) and the rest of the network (remote). We want to have a tool that is capable of capturing all of communication intended for the local node.

There are multiple methods of intercepting network communication. In our case, we chose to 
a raw TCP socket.

The internet operates as a "layered" system, with multiple protocols on each layer. Internet protocol (IP) is protocol of the 3rd layer while user datagram protocol (UDP) and the transmission control protocol (TCP) belong to the 4th layer of the TCP/IP model.

Peer to peer (P2P) communication between Tezos nodes operates under the aforementioned TCP protocol. The internal node uses a TCP socket. From the point of view of the application, the TCP socket works as a channel, as you input data, they are released on the other end in the order that they were received. Everything else is handled by the operating system. 

A Raw TCP socket works differently-it captures packets with headers, but you have to compose the message yourself and make sure it's in the right order. However, it allows you to capture all of the communication, not just the one that is intended for your application.

Applications commonly use standard TCP sockets, with the TCP stack being held internally by the operating system. We, on the other hand, use a “raw” TCP socket, which is received from the operating system and operates on TCP packets (instead of already processed bytes) but does not contain a TCP stack. 

This allows it to capture all TCP packets, including their TCP and IP headers. Using the raw socket, you can intercept all of the packets that constitute TCP communication. Based on the data in the packet headers, you can then filter the packets that are used in communication with the Tezos node. 

**The raw TCP socket**

In addition to the message, a TCP header contains two more items: an IP header and the TCP header itself.

The IP header contains the IP address, which informs us about the packet’s sender and receiver.
The TCP header provides us with the TCP port, which informs us which application the packet belongs to. 

Using Docker, you can create a shared network - there were two containers on the same network, with one running a Tezos node and the other running the Debugger. Since all of the communication is contained within this network, you can use the Debugger to analyze it. This removes the obligation to set up a tun device, which makes it a more user-friendly experience. 

**Simulating network traffic**

In order to perform replay, we need to simulate a running node by literally connecting the node onto another node. Now that you have a local node running, you launch the Debugger
If you want to perform replay on the running node, you need to open another TCP socket locally. In this case, it is a common TCP socket (not raw). Through this socket, the Debugger communicates with the node as if it were another node, sending messages based on what is stored in its database.

In order to perform replay, we need to create a snapshot of communication we want to simulate (by using the debugger). Using the snapshot and plain TCP socket, we can use the debugger as if it is a node itself, but instead of actual P2P communication logic, it uses the created snapshot as a guide to drive the P2P communication.


**Operating within the Tezos network**

When you launch the TezEdge Debugger, it will run parallel to your Tezos node, recording the messages, packet and RPCs made in connection with your node. The TezEdge Debugger makes use of the identity.json which is generated by a node before it makes its first connection to the Tezos network. It then uses the private key from the identity.json in combination with the public keys and nonces it intercepts as they are being sent across the network

At a broad level, the P2P Explorer operates by setting up an interception layer between two peers before they exchange the public keys and nonces.

Let’s say Alice is running a node on the Tezos P2P network. She wants to run the TezEdge P2P Explorer to examine the messages sent between her node and Bob’s node. The communication between Alice and Bob is unencrypted since they haven’t exchanged the public keys and nonces necessary for encryption.



1. **Alice** is the local user. She launches the **TezEdge debugger**, creating an interception layer between Alice (local) and Bob (remote).
2. **Alice** sends a **ConnectionMessage** to Bob. This message contains (among other items) her **public key** and a **nonce**.
3. The **debugger** intercepts and records the message, creating a copy of it and storing it onto its storage, providing the data in a readable format through an exposed application programming interface (API). The original **ConnectionMessage** with **Alice’s public key** is delivered to Bob.
4. Bob sends his ConnectionMessage (including his public key and a nonce) to **Alice**, the **debugger** intercepts and records Bob’s message, copying it onto its storage, providing the data in a readable format through its API. The original message with Bob’s public key is delivered to Alice.
5. Now that both **Alice** and Bob have received the **public keys** and **nonces** (thus completing the handshake), their communication can now be **encrypted.**


![Image](../../static/images/Debugger1.svg "Intercepting the connection message")

_Now that the TezEdge debugger has received the public keys and nonces, it is ready to intercept, decrypt and deserialize the messages that flow between the two peers:_

6. **Alice** uses Bob’s public key, her **own** **private key** and **her nonce** to encrypt a message which she then sends to Bob.

7. The **TezEdge debugger** intercepts, decrypts (and, if necessary, deserializes) **Alice’s message**, providing the data into the TezEdge debugger which displays it via its UI.

8. Bob receives **Alice’s** original message.

9. **Alice** can access the **UI** to browse the data intercepted by the **TezEdge debugger** at any time


![Image](../../static/images/Debugger2.svg "Decrypting and deserializing messages")

**How to run the TezEdge Debugger**

**1.** Open shell and type this code into the command line and then press Enter

git clone[ https://github.com/simplestaking/tezedge-debugger.git](https://github.com/simplestaking/tezedge-debugger.git)

**cd[ tezedge-debugger](https://github.com/simplestaking/tezedge-debuger)**

**2.** Download and install Docker and Docker Compose

Open shell and type this code into the command line and then press Enter:

**sudo ./docker.sh**

**3.** Open the TezEdge Explorer in your browser

You can view the status of the debugger in your browser by entering this address into your browser’s URL bar:

**[http://localhost:8080](http://localhost:8080/)**


### **Filtering and visualizing the data**

When a Tezos node is up and running, there is a large amount of data flowing between the node and the rest of the network. Although we can record the data, browsing through it is difficult due to its sheer volume.

We want to be able to quickly find certain events or items within the data. For example, when the node exchanges messages with other peers via the P2P network, we want to know which peers sent a connection message to our node, as well as the message details in its metadata.

Utilizing _filters_, we can examine a particular section of the data from various angles in order to accelerate the debugging process. Using a conventional database system would significantly hinder the performance of the node. We must consider faster, albeit simpler, storage solutions.

In order to store such a large amount of data, we chose **RocksDB**, which is a high performance _key-value store_. By key value, we mean that there is no high level structure of data (such as tables in relational databases or structures in NoSQL databases). A key value store is an association of a byte array representing the key with another byte array representing a value.

**Utilizing indexes for filtering**

To create a high performing filter, we need to utilize our own indexes. Conveniently for us, RocksDB holds its data sorted by its key, which provides us with a functionality that is necessary for fast data filtering.

RocksDB contains one more powerful mechanism called _column families_, which allows us to group data of the same type into their own “named columns”. This way, we can separate data indexes, allowing us to efficiently look up data with specific properties from the database itself.

At its core, the index is a very simple idea: combine part of the data with its associated key to carefully create an “index key” that will be inserted and sorted into its own column family. We use sequentially generated ID (meaning that each message is given a number representing order they were received) for actual data.

By combining, we mean prefixing the ID of the data with binary representation of the property we want to sort by, such as a particular type of a P2P message. Thanks to the sorting of RocksDB, all P2P messages with the same type are grouped together (and correctly ordered). We only need to retrieve messages of a specific type. For this, we utilize the “prefix iterator”, a special type of iterator that returns values with a specified prefix value.

But to perform more advanced filters and queries, like finding messages that are either connection messages (used for establishing encrypted connection between nodes) OR metadata message (containing details about nodes after establishing encrypted connection), an algorithmic approach is required. But in such a situation, we can just use the merge algorithm (used in the merge sort) to correctly join and sort multiple prefix iterators.

To search by multiple properties, such as searching for messages that are connection messages AND are incoming, we should imagine how our data is structured in the database. All keys are **unique, sorted** and we constructed the index, so they would also be **grouped** if they share a property. This allows us to use prefix iterators like **sorted sets** of IDs and performing complex filters by building an set intersection (finding IDs that are present in both iterators).

After we find our desired IDs from our indexes, all we need is just to load complete data from primary column families and present them to the user.

You can browse through the P2P network utilizing a variety of filters, for example:



![Image](../../static/images/Debugger3.gif)


**Visualizing log files from the OCaml and Rust Tezos nodes**

The logs are information about the node’s actions. Some logs are error logs, which are of particular interest to developers as they directly affect the node’s operation. They inform us about the severity of the problem, its approximate location within the code and the time when it happened.

When you have a Tezos (either OCaml or Rust-based) node up and running, a log will be recorded in the terminal that runs the node. In case of an issue, we want to be able to quickly browse through the logs and find the root cause. The TezEdge debugger processes the logs into a more manageable format, stores them inside our database and builds indexes on top of them.

The TezEdge Explorer allows us to easily switch and compare data between multiple Tezos nodes.

In the lower left corner of the interface, you can select from a drop-down menu of servers (Rust and two OCaml nodes) and examine the logs, network and RPCs of each node.


![Image](../../static/images/Debugger4.gif)


To try out the aforementioned features, visit the[ TezEdge.com](http://tezedge.com) website.

