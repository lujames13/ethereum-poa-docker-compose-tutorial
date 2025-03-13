[English](./README.md) | [中文](./README.zh-TW.md)

# Setting up a Private Ethereum Network with Docker Compose and Geth 1.13

This document provides complete steps for setting up a private Ethereum network using Docker Compose and Geth 1.13, implementing the Clique PoA (Proof of Authority) consensus mechanism.

## Table of Contents

1. [Introduction to Clique Consensus Mechanism](#1-introduction-to-clique-consensus-mechanism)
   - [1.1 What is Clique PoA](#11-what-is-clique-poa)
   - [1.2 How Clique Works](#12-how-clique-works)
   - [1.3 Voting and Governance Mechanisms](#13-voting-and-governance-mechanisms)
   - [1.4 Programmatic Implementation of Voting](#14-programmatic-implementation-of-voting)
2. [Implementing a Private Ethereum Network with Docker Compose](#2-implementing-a-private-ethereum-network-with-docker-compose)
   - [2.1 File Structure](#21-file-structure)
   - [2.2 Prerequisites](#22-prerequisites)
   - [2.3 Complete Installation Steps](#23-complete-installation-steps)
   - [2.4 Creating a Working Directory](#24-creating-a-working-directory)
   - [2.5 Creating Password Files](#25-creating-password-files)
   - [2.6 Creating Accounts for Nodes](#26-creating-accounts-for-nodes)
   - [2.7 Creating the Genesis Block Configuration File](#27-creating-the-genesis-block-configuration-file)
   - [2.8 Initializing the Nodes](#28-initializing-the-nodes)
   - [2.9 Getting Node 1's enode ID](#29-getting-node-1s-enode-id)
   - [2.10 Creating the Docker Compose Configuration File](#210-creating-the-docker-compose-configuration-file)
   - [2.11 Starting the Private Network](#211-starting-the-private-network)
   - [2.12 Connecting to the Node Console](#212-connecting-to-the-node-console)
   - [2.13 Verifying Network Connections](#213-verifying-network-connections)
   - [2.14 Checking Accounts and Balances](#214-checking-accounts-and-balances)
3. [Testing Network Functionality](#3-testing-network-functionality)
   - [3.1 Testing Transaction Functionality](#31-testing-transaction-functionality)
   - [3.2 Monitoring Block Generation](#32-monitoring-block-generation)
   - [3.3 Stopping the Network](#33-stopping-the-network)
4. [Troubleshooting](#4-troubleshooting)
   - [4.1 Nodes Cannot Connect to Each Other](#41-nodes-cannot-connect-to-each-other)
   - [4.2 Cannot Mine](#42-cannot-mine)
   - [4.3 Common Error: Invalid IP Address](#43-common-error-invalid-ip-address)
5. [Advanced Uses](#5-advanced-uses)
   - [5.1 Smart Contract Deployment Example](#51-smart-contract-deployment-example)
6. [Important Notes](#6-important-notes)
7. [References](#7-references)

## 1. Introduction to Clique Consensus Mechanism

### 1.1 What is Clique PoA

Clique is a Proof of Authority (PoA) consensus mechanism in the Ethereum ecosystem, designed specifically for private and consortium chains. Unlike Proof of Work (PoW), Clique doesn't require significant computational power for mining. Instead, blocks are produced by pre-selected authorized nodes (called validators) in rotation.

**Key Features:**

- **High Performance**: No need to solve complex mathematical problems, resulting in faster block generation
- **Low Resource Consumption**: No specialized mining hardware required, significantly reducing energy consumption
- **Predictable Block Generation**: Stable block interval (5 seconds in this configuration)
- **Identity-Oriented**: Relies on validator identity and reputation to ensure network security
- **No Fork Risk**: With a clear set of validators, most fork scenarios are effectively eliminated

**Suitable Scenarios:**

- Enterprise internal blockchain applications
- Consortium chain networks
- Development and testing environments
- Applications requiring high performance and determinism

### 1.2 How Clique Works

The Clique consensus mechanism operates as follows:

**Block Generation Process:**

1. **Validator Rotation**: The system determines the block producer for each block based on a deterministic algorithm
2. **Signature Ordering**: Validators take turns producing blocks in a pseudo-random order based on their addresses
3. **Block Signing**: The current validator signs the block header, placing it in the `extradata` field
4. **Block Interval**: Blocks are produced regularly based on the configured `period` parameter (5 seconds in this example)

**Key Parameter Analysis:**

In the `genesis.json` Clique configuration:

```json
"clique": {
  "period": 5,    // Block generation interval (seconds)
  "epoch": 30000  // Number of blocks for recalculating votes and validators
}
```

- **period**: Controls the block generation interval; shorter intervals mean faster transaction confirmation but increased network load
- **epoch**: Defines the number of blocks in a complete governance cycle; at the end of each epoch:
  - The system processes and settles all accumulated votes
  - Updates the validator set
  - Clears votes that did not reach consensus
  - Recalculates block signing order

**Genesis Block Validator Setup:**

The initial validators are included in the `extradata` field of the `genesis.json`:

```
"extradata": "0x0000...0000NODE1_ADDRESS_WITHOUT_0x0000...0000"
```

Format: 32-byte prefix + validator address (without 0x prefix) + 65-byte suffix

### 1.3 Voting and Governance Mechanisms

Clique uses a majority voting mechanism to dynamically adjust the validator set:

**Voting Rules:**

- Each existing validator has one vote
- Proposals need majority support to pass
- Validators cannot vote to add or remove themselves
- Votes take effect at the end of the next epoch
- Votes that don't achieve a majority within an epoch are discarded

**Using the Geth Console for Voting:**

1. **Proposing to add a new validator**:
   ```javascript
   clique.propose("0xNEW_VALIDATOR_ADDRESS", true)
   ```

2. **Proposing to remove an existing validator**:
   ```javascript
   clique.propose("0xEXISTING_VALIDATOR_ADDRESS", false)
   ```

3. **Viewing all current proposals**:
   ```javascript
   clique.proposals()
   ```
   Example output:
   ```
   {
     "0xabc123...": true,    // Proposal to add this address
     "0xdef456...": false    // Proposal to remove this address
   }
   ```

4. **Viewing the current validator set**:
   ```javascript
   clique.getSigners()
   ```
   Example output:
   ```
   ["0xabc123...", "0xdef456..."]
   ```

5. **Viewing the validator for a specific block**:
   ```javascript
   clique.getSnapshot()
   ```
   Example output:
   ```
   {
     hash: "0x...",
     number: 100,
     recents: {...},
     signers: {...},
     votes: [...]
   }
   ```

   ### 1.4 Programmatic Implementation of Voting

In addition to the Geth console, voting operations can also be performed programmatically:

**Using Web3.js for Voting**:

```javascript
// Import Web3 library
const Web3 = require('web3');
const web3 = new Web3('http://localhost:8545');  // Connect to Node 1's RPC

// Prepare sender account
const sender = "0xYOUR_VALIDATOR_ADDRESS";
const privateKey = "YOUR_PRIVATE_KEY_WITHOUT_0x";

// Unlock account (or use signed transactions)
async function proposeValidator(validatorAddress, add) {
  try {
    // Method 1: Using unlocked account
    await web3.eth.personal.unlockAccount(sender, "password", 600);
    
    // Send RPC request
    return await web3.currentProvider.send({
      method: 'clique_propose',
      params: [validatorAddress, add],  // add=true means add, false means remove
      jsonrpc: "2.0",
      id: Math.floor(Math.random() * 1000)
    });
  } catch (error) {
    console.error("Vote failed:", error);
  }
}

// Get current proposals
async function getProposals() {
  try {
    return await web3.currentProvider.send({
      method: 'clique_proposals',
      params: [],
      jsonrpc: "2.0",
      id: Math.floor(Math.random() * 1000)
    });
  } catch (error) {
    console.error("Failed to get proposals:", error);
  }
}

// Get current validators
async function getSigners() {
  try {
    return await web3.currentProvider.send({
      method: 'clique_getSigners',
      params: [],
      jsonrpc: "2.0",
      id: Math.floor(Math.random() * 1000)
    });
  } catch (error) {
    console.error("Failed to get validators:", error);
  }
}

// Usage example
async function main() {
  // Propose adding a new validator
  const result = await proposeValidator("0xNEW_VALIDATOR_ADDRESS", true);
  console.log("Vote result:", result);
  
  // View current proposals
  const proposals = await getProposals();
  console.log("Current proposals:", proposals);
  
  // View current validators
  const signers = await getSigners();
  console.log("Current validators:", signers);
}

main();
```

**Monitoring Validator Change Events**:

Although Clique itself doesn't have dedicated events, you can monitor changes to the `extradata` field in block headers to track validator set changes:

```javascript
// Monitor validator changes
let knownSigners = new Set();

// Subscribe to new blocks
web3.eth.subscribe('newBlockHeaders', async (error, blockHeader) => {
  if (error) {
    console.error("Subscription error:", error);
    return;
  }
  
  // Check at each epoch boundary (blockNumber % epoch === 0)
  if (blockHeader.number % 30000 === 0) {
    // Get the latest validator set
    const signers = await getSigners();
    const newSigners = new Set(signers.result);
    
    // Detect added validators
    for (const signer of newSigners) {
      if (!knownSigners.has(signer)) {
        console.log(`New validator added: ${signer}`);
      }
    }
    
    // Detect removed validators
    for (const signer of knownSigners) {
      if (!newSigners.has(signer)) {
        console.log(`Validator removed: ${signer}`);
      }
    }
    
    // Update known validator set
    knownSigners = newSigners;
  }
});
```

## 2. Implementing a Private Ethereum Network with Docker Compose

The following sections detail how to set up a private Ethereum network based on the Clique consensus using Docker Compose.

### 2.1 File Structure

```
.
├── README.md
├── docker-compose.yml
├── genesis.json
├── node1
│   ├── geth
│   ├── geth.ipc
│   ├── history
│   └── keystore
├── node1_password.txt
├── node2
│   ├── geth
│   ├── geth.ipc
│   └── keystore
└── node2_password.txt
```

### 2.2 Prerequisites

- Docker and Docker Compose installed
- Basic terminal operation knowledge

### 2.3 Complete Installation Steps

The following sections provide step-by-step instructions for creating a private Ethereum network.

### 2.4 Creating a Working Directory

```bash
mkdir -p ethereum-private-net/{node1,node2}
cd ethereum-private-net
```

### 2.5 Creating Password Files

```bash
# Create password file for node1
echo "password1" > node1_password.txt

# Create password file for node2
echo "password2" > node2_password.txt
```

### 2.6 Creating Accounts for Nodes

```bash
# Create account for node1
docker run --rm -v $(pwd)/node1:/root/.ethereum -v $(pwd)/node1_password.txt:/password.txt ethereum/client-go:v1.13.14 account new --password /password.txt

# Create account for node2
docker run --rm -v $(pwd)/node2:/root/.ethereum -v $(pwd)/node2_password.txt:/password.txt ethereum/client-go:v1.13.14 account new --password /password.txt
```

Record the account address from each command output, for example: `0x1234567890abcdef1234567890abcdef12345678`
(In this example, node1 address is `0xC0A55ae58fb8E26f7874E865eE143f033D445927`, and node2 address is `0x8c59707CcF4c996bDB6163A3a759baADf82dAe6A`)

### 2.7 Creating the Genesis Block Configuration File

Create a `genesis.json` file:

```bash
cat > genesis.json << EOF
{
  "config": {
    "chainId": 123454321,
    "homesteadBlock": 0,
    "eip150Block": 0,
    "eip155Block": 0,
    "eip158Block": 0,
    "byzantiumBlock": 0,
    "constantinopleBlock": 0,
    "petersburgBlock": 0,
    "istanbulBlock": 0,
    "muirGlacierBlock": 0,
    "berlinBlock": 0,
    "londonBlock": 0,
    "arrowGlacierBlock": 0,
    "grayGlacierBlock": 0,
    "clique": {
      "period": 5,
      "epoch": 30000
    }
  },
  "difficulty": "1",
  "gasLimit": "800000000",
  "extradata": "0x0000000000000000000000000000000000000000000000000000000000000000NODE1_ADDRESS_WITHOUT_0x0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000",
  "alloc": {
    "NODE1_ADDRESS_WITHOUT_0x": { "balance": "1000000000000000000" },
    "NODE2_ADDRESS_WITHOUT_0x": { "balance": "1000000000000000000" }
  }
}
EOF
```

**Important**: Replace the following with your actual values:
- `NODE1_ADDRESS_WITHOUT_0x`: Node1's address (without the 0x prefix)
  (In this example: `C0A55ae58fb8E26f7874E865eE143f033D445927`)
- `NODE2_ADDRESS_WITHOUT_0x`: Node2's address (without the 0x prefix)
  (In this example: `8c59707CcF4c996bDB6163A3a759baADf82dAe6A`)

### 2.8 Initializing the Nodes

```bash
# Initialize node1
docker run --rm -v $(pwd)/node1:/root/.ethereum -v $(pwd)/genesis.json:/genesis.json ethereum/client-go:v1.13.14 init --datadir /root/.ethereum /genesis.json

# Initialize node2
docker run --rm -v $(pwd)/node2:/root/.ethereum -v $(pwd)/genesis.json:/genesis.json ethereum/client-go:v1.13.14 init --datadir /root/.ethereum /genesis.json
```

### 2.9 Getting Node 1's enode ID

Temporarily start node1 to get its enode ID:

```bash
docker run --rm -it --name temp-node1 -v $(pwd)/node1:/root/.ethereum -v $(pwd)/node1_password.txt:/password.txt ethereum/client-go:v1.13.14 --datadir /root/.ethereum --port 30306 --networkid 123454321 console
```

In the geth console, execute:

```javascript
admin.nodeInfo.enode
```

This will output a string similar to `enode://7e302aa...@127.0.0.1:30306`. Copy the part between `enode://` and `@`; this is node1's enode ID.
(In this example: `enode://67cbdce9f4d0a82cfb76053f948bb467d5acb96175a31330b99df0907e65a0468946f2ba2680a850e72a00ab92bbed53d417765b0e32e7afb3523453399edd45@111.241.136.73:30306`)

Type `exit` to exit the console.

### 2.10 Creating the Docker Compose Configuration File

Create a `docker-compose.yml` file:

```yaml
version: '3'

services:
  node1:
    container_name: ethereum-node1
    image: ethereum/client-go:v1.13.14
    volumes:
      - ./node1:/root/.ethereum
      - ./node1_password.txt:/password.txt
      - ./genesis.json:/genesis.json
    ports:
      - "30306:30306"
      - "30306:30306/udp"
      - "8545:8545"
    command: --datadir /root/.ethereum --port 30306 --networkid 123454321 --unlock 0xC0A55ae58fb8E26f7874E865eE143f033D445927 --password /password.txt --mine --miner.etherbase 0xC0A55ae58fb8E26f7874E865eE143f033D445927 --http --http.api eth,net,web3,personal,admin --http.addr 0.0.0.0 --http.port 8545 --http.corsdomain "*" --allow-insecure-unlock
    networks:
      ethnet:
        ipv4_address: 172.20.0.2

  node2:
    container_name: ethereum-node2
    image: ethereum/client-go:v1.13.14
    depends_on:
      - node1
    volumes:
      - ./node2:/root/.ethereum
      - ./node2_password.txt:/password.txt
      - ./genesis.json:/genesis.json
    ports:
      - "30307:30307"
      - "30307:30307/udp"
      - "8546:8546"
    command: --datadir /root/.ethereum --port 30307 --networkid 123454321 --unlock 0x8c59707CcF4c996bDB6163A3a759baADf82dAe6A --password /password.txt --http --http.api eth,net,web3,personal,admin --http.addr 0.0.0.0 --http.port 8546 --http.corsdomain "*" --allow-insecure-unlock --bootnodes enode://67cbdce9f4d0a82cfb76053f948bb467d5acb96175a31330b99df0907e65a0468946f2ba2680a850e72a00ab92bbed53d417765b0e32e7afb3523453399edd45@172.20.0.2:30306
    networks:
      ethnet:
        ipv4_address: 172.20.0.3

networks:
  ethnet:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16
```

Replace the following with your actual values:
- `0xNODE1_ADDRESS`: Node1's full address (including the 0x prefix)
  (In this example: `0xC0A55ae58fb8E26f7874E865eE143f033D445927`)
- `0xNODE2_ADDRESS`: Node2's full address (including the 0x prefix)
  (In this example: `0x8c59707CcF4c996bDB6163A3a759baADf82dAe6A`)
- `NODE1_ENODE_WITHOUT_IP`: Node1's enode ID
  (In this example: `67cbdce9f4d0a82cfb76053f948bb467d5acb96175a31330b99df0907e65a0468946f2ba2680a850e72a00ab92bbed53d417765b0e32e7afb3523453399edd45`)

### 2.11 Starting the Private Network

```bash
docker-compose up
```

If you want to run it in the background:

```bash
docker-compose up -d
```

### 2.12 Connecting to the Node Console

In a new terminal window:

```bash
# Connect to node1
docker exec -it ethereum-node1 geth attach /root/.ethereum/geth.ipc

# Connect to node2
docker exec -it ethereum-node2 geth attach /root/.ethereum/geth.ipc
```

Architecture relationship:

```
[Your Terminal] → [Docker Container] → [Geth JavaScript Console] → [Ethereum Node] → [Blockchain Network]
```

### 2.13 Verifying Network Connections

After connecting to the node console, you'll see a welcome message similar to:

```
Welcome to the Geth JavaScript console!
instance: Geth/v1.13.14-stable-2bd6bd01/linux-amd64/go1.21.7
coinbase: 0xc0a55ae58fb8e26f7874e865ee143f033d445927
at block: 9 (Wed Mar 12 2025 21:39:08 GMT+0000 (UTC))
 datadir: /root/.ethereum
 modules: admin:1.0 clique:1.0 debug:1.0 engine:1.0 eth:1.0 miner:1.0 net:1.0 rpc:1.0 txpool:1.0 web3:1.0
To exit, press ctrl-d or type exit
```

Run the following commands in the node console to verify network connections:

```javascript
// Check block height - the number will increase as mining progresses
> eth.blockNumber
10  // Indicates that 10 blocks have been generated so far

// Check the number of peer nodes
> net.peerCount
2  // Indicates connections to 2 nodes, typically the other local node and an external node

// View peer details
> admin.peers
[{
    caps: ["eth/68", "snap/1"],
    enode: "enode://aa6c5c109f9cd6c4d2fa95ed61b1a158c14d0da3934ea3fb744ad28b902a1771a97f2f3eb342cb85385e1d8c8ef9390d3fc8fb8fb948aa767a60622c36bf7d55@172.20.0.3:45978",
    id: "4f674866b3b7ca76ca3650ddf81a138ed1bd68aa6b311a04571a613119e06111",
    name: "Geth/v1.13.14-stable-2bd6bd01/linux-amd64/go1.21.7",  // Shows this is node2
    network: {
      inbound: true,  // Indicates this is an inbound connection
      localAddress: "172.20.0.2:30306",  // Local node address
      remoteAddress: "172.20.0.3:45978",  // Remote node address (node2)
      static: false,
      trusted: false
    },
    protocols: {
      eth: {
        version: 68  // Ethereum protocol version
      },
      snap: {
        version: 1  // Snapshot sync protocol version
      }
    }
}, {
    caps: ["eth/66", "eth/67", "eth/68", "snap/1"],
    enode: "enode://8ffcf8ba02dc25d1fd0cafeeecea2f6ee6c9f5c12199178ac61ee548684b728018ba686a4a4112076d3c8b7ec94c3eb58e7cbb4a9d59a03edf61fb65b37f5dfe@51.195.39.168:30303",
    id: "cb5260385409e7741a7d16ec47c24d4da4c1c6b5f979d9d2df4ca29458bb7267",
    name: "Geth/v1.14.13-stable-eb00f169/linux-amd64/go1.23.5",  // This is an external Ethereum node
    network: {
      inbound: false,  // Indicates this is an outbound connection
      localAddress: "172.20.0.2:46520",
      remoteAddress: "51.195.39.168:30303",  // External node address
      static: false,
      trusted: false
    },
    protocols: {
      eth: "handshake",  // Handshake in progress
      snap: "handshake"
    }
}]
```

After successful connection, you should see node1 and node2 connected to each other. The above output clearly shows node1 (172.20.0.2) is connected to node2 (172.20.0.3).

### 2.14 Checking Accounts and Balances

Execute the following commands in either node console:

```javascript
// Display account addresses
> eth.accounts
["0xc0a55ae58fb8e26f7874e865ee143f033d445927"]  // Lists all account addresses for the node

// Check account balance (in Wei)
> eth.getBalance(eth.accounts[0])
1000000000000000000  // Indicates a balance of 1 ETH (10^18 Wei)

// Convert Wei to a more readable Ether unit
> web3.fromWei(eth.getBalance(eth.accounts[0]), "ether")
1  // Confirms the balance is 1 ETH
```

## 3. Testing Network Functionality

### 3.1 Testing Transaction Functionality

Execute the following commands in node1's console to send a transaction:

```javascript
// Send 0.1 Ether from node1 to node2
> eth.sendTransaction({
  from: "0xC0A55ae58fb8E26f7874E865eE143f033D445927",  // Node1 address
  to: "0x8c59707CcF4c996bDB6163A3a759baADf82dAe6A",    // Node2 address
  value: web3.toWei(0.1, "ether")
})
"0x58b6458922733da50d6230560cd033d147532beb64107e75dc246853cdb6a8ec"  // Returns the transaction hash
```

Use the returned transaction hash to check transaction details:

```javascript
// Query transaction details
> eth.getTransaction("0x58b6458922733da50d6230560cd033d147532beb64107e75dc246853cdb6a8ec")
{
  accessList: [],
  blockHash: "0x669ca015346779881a8cf844dba3038211e68842bc0b017c584ad137f448bcaa",  // Hash of the block containing this transaction
  blockNumber: 72,  // Transaction is included in block 72
  chainId: "0x75bc371",  // Chain ID, corresponds to decimal 123454321
  from: "0xc0a55ae58fb8e26f7874e865ee143f033d445927",  // Sender address (node1)
  gas: 21000,  // Gas limit used
  gasPrice: 1000066773,  // Gas price
  hash: "0x58b6458922733da50d6230560cd033d147532beb64107e75dc246853cdb6a8ec",  // Transaction hash
  input: "0x",  // Transaction input data (empty indicates a pure transfer transaction)
  maxFeePerGas: 1000174426,  // EIP-1559 transaction max fee
  maxPriorityFeePerGas: 1000000000,  // EIP-1559 transaction max priority fee
  nonce: 0,  // Sender's transaction count
  r: "0x1d2bb3ce66485791e1de1ecbaa2d06b7eb643b21a425c16bd690d011971c3d54",  // Signature component r
  s: "0x28209c4bf004642f4a12863093a1f0e4a8dca71995e0775bf5e20a43bc5c1f1f",  // Signature component s
  to: "0x8c59707ccf4c996bdb6163a3a759baadf82dae6a",  // Recipient address (node2)
  transactionIndex: 0,  // Transaction index in the block
  type: "0x2",  // Transaction type (0x2 indicates an EIP-1559 transaction)
  v: "0x1",  // Signature component v
  value: 100000000000000000,  // Transaction amount (0.1 ETH)
  yParity: "0x1"  // Signature y-parity
}
```

Check the balance change in node2's console:

```javascript
web3.fromWei(eth.getBalance(eth.accounts[0]), "ether")
```

### 3.2 Monitoring Block Generation

In either node console, set up a block monitor to observe new blocks in real-time:

```javascript
// Set up a block monitor
> eth.filter("latest").watch(function() {
  console.log("New block: ", eth.blockNumber);
})
{
  callbacks: [function()],
  filterId: "0x69b1cfd917f61250c96e114bd68cfcef",  // Filter ID to identify this monitor
  getLogsCallbacks: [],
  implementation: {
    getLogs: function(),
    newFilter: function(),
    poll: function(),
    uninstallFilter: function()
  },
  options: "latest",  // Monitor latest blocks
  pollFilters: [],
  requestManager: {
    polls: {
      0x69b1cfd917f61250c96e114bd68cfcef: {
        data: {...},
        id: "0x69b1cfd917f61250c96e114bd68cfcef",
        callback: function(error, messages),
        uninstall: function bound()
      }
    },
    provider: {
      send: function github.com/ethereum/go-ethereum/internal/jsre.MakeCallback.func1(),
      sendAsync: function github.com/ethereum/go-ethereum/internal/jsre.MakeCallback.func1()
    },
    timeout: {},
    poll: function(),
    reset: function(keepIsSyncing),
    send: function(data),
    sendAsync: function(data, callback),
    sendBatch: function(data, callback),
    setProvider: function(p),
    startPolling: function(data, pollId, callback, uninstall),
    stopPolling: function(pollId)
  },
  formatter: function(log),
  get: function(callback),
  stopWatching: function(callback),  // Can be used to stop monitoring
  watch: function(callback)
}

// As new blocks are generated, the console will output block heights
> New block:  768
New block:  769
New block:  770
New block:  771
New block:  772
New block:  773
New block:  774
New block:  775
```

You can see blocks are continuously being generated, indicating that the mining function is working properly. A new block is generated approximately every 5 seconds (according to the configured Clique PoA consensus period).

Confirm the balance change (after receiving the transaction, node2's balance should increase):

```javascript
// Check balance in node2's console
> web3.fromWei(eth.getBalance(eth.accounts[0]), "ether")
1.1  // Shows that 0.1 ETH has been received, total balance is now 1.1 ETH
```

### 3.3 Stopping the Network

To stop the network:

```bash
# If running in the foreground, press Ctrl+C
# If running in the background
docker-compose down
```

## 4. Troubleshooting

### 4.1 Nodes Cannot Connect to Each Other

1. Check if the network IDs match (123454321 in the example)
2. Verify that the enode URL is correct
3. Try restarting the containers
4. Check Docker logs for error messages

```bash
docker logs ethereum-node1
docker logs ethereum-node2
```

### 4.2 Cannot Mine

Ensure that node1's account is unlocked and configured as a miner:

```javascript
// In node1's console
eth.coinbase
miner.getHashrate()
```

### 4.3 Common Error: Invalid IP Address

If you encounter the `Fatal: Option nat: invalid IP address` error, remove the `--nat extip:node1` parameter and restart.

## 5. Advanced Uses

This private network can be used for:
- Smart contract development and testing
- DApp development
- Learning and experimenting with blockchain concepts
- Performance testing

### 5.1 Smart Contract Deployment Example

In the node console:

```javascript
// Unlock account (if not already unlocked)
personal.unlockAccount(eth.accounts[0], "password1", 3600)

// Sample contract code example
var contractSource = `
pragma solidity ^0.8.0;

contract SimpleStorage {
    uint256 private storedData;
    
    function set(uint256 x) public {
        storedData = x;
    }
    
    function get() public view returns (uint256) {
        return storedData;
    }
}
`

// Compilation and deployment steps will vary depending on your development environment
```

## 6. Important Notes

- This setup is only for development and testing, not suitable for production environments
- Password files should be stored securely; in production environments, more secure key management methods should be used
- The default configuration exposes RPC ports; appropriate security measures should be set in a production environment
- In actual applications, it's recommended to use more nodes to increase network resilience

## 7. References

- [Geth Official Documentation](https://geth.ethereum.org/docs/)
- [Ethereum Developer Documentation](https://ethereum.org/developers/)
- [Clique PoA Consensus Algorithm](https://eips.ethereum.org/EIPS/eip-225)