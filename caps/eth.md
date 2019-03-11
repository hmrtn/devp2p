# The eth/63 protocol

'eth' is protocol on the [RLPx](../rlpx.md) transport that facilitates exchange of
Ethereum blockchain information between peers. The current protocol version is **eth/63**.
See end of document for a list of changes in past protocol versions.

### Protocol Semantics

Upon an active session, a Status message must be sent. Following the reception of the
peer's Status message, the Ethereum session is active and any other messages may be sent.

All known transactions should be sent following the Status exchange with one or more
Transactions messages.

Transactions messages should also be sent periodically as the node has new transactions to
disseminate. A node should never send a transaction back to the peer that it can determine
already knows of it (either because it was previously sent or because it was informed from
this peer originally).

New blocks are propagated using the NewBlock and NewBlockHashes messages. The NewBlock
message includes the full block and is sent to a small fraction of connected peers
(usually sqrt(peers)). All other peers are sent a NewBlockHashes message containing just
the hash of the new block. Those peers may request the full block later if they fail to
receive it from anyone within reasonable time.

Blocks are typically re-propagated to all connected peers as soon as basic validity of the
announcement has been established (e.g. after the proof-of-work check).

### Basic Chain Synchronization

Two peers connect and send their Status message. Status includes the Total Difficulty(TD)
and hash of their best block.

The client with the worst TD then proceeds to download block headers using the
BlockHeadersFromNumber message. It verifies proof-of-work values in received headers and
fetches block bodies using the GetBlockBodies message. Received blocks are executed using
the Ethereum Virtual Machine, recreating the state tree.

Note that header downloads, block body downloads and block execution may happen
concurrently.

### State Synchronization (a.k.a. "fast sync")

eth/63 also allows synchronizing transaction execution results ("state"). This may be
faster than re-executing all transactions but comes at the expense of some security.

State synchronization typically proceeds by downloading the chain of block headers,
verifying their proof-of-work values. Block bodies are requested as in the Basic Chain
Synchronization section but block transactions aren't executed. Instead, the client picks
a block near the head of the chain and downloads merkle tree nodes and contract code
incrementally by requesting the root node, it's children, grandchildren, ... until the
entire tree is synchronized.

## Protocol Messages

### Status (0x00)

`[protocolVersion: P, networkId: P, td: P, bestHash: B_32, genesisHash: B_32, number: P]`

Inform a peer of its current ethereum state. This message should be sent _after_ the
initial handshake and _prior_ to any **ethereum** related messages.

* `protocolVersion`: the current protocol version, 63
* `networkId`: Integer identifying the blockchain, see table below
* `td`: total difficulty of the best chain. Integer, as found in block header.
* `bestHash`: The hash of the best (i.e. highest TD) known block.
* `genesisHash`: The hash of the Genesis block.
* `number`: The block number of the latest block in the chain.

This table lists common Network IDs and their corresponding networks. Other IDs exist
which aren't listed, i.e. clients should not require that any particular network ID is
used.

| ID | chain                              |
|----|------------------------------------|
| 0  | Olympic (disused)                  |
| 1  | Frontier (now mainnet)             |
| 2  | Morden (disused)                   |
| 3  | Ropsten (current PoW testnet)      |
| 4  | [Rinkeby](https://www.rinkeby.io/) |

### NewBlockHashes (0x01)

`[[hash_0: B_32, number_0: P], [hash_1: B_32, number_1: P], ...]`

Specify one or more new blocks which have appeared on the network. To be maximally
helpful, nodes should inform peers of all blocks that they may not be aware of. Including
hashes that the sending peer could reasonably be considered to know (due to the fact they
were previously informed of because that node has itself advertised knowledge of the
hashes through NewBlockHashes) is considered bad form, and may reduce the reputation of
the sending node. Including hashes that the sending node later refuses to honour with a
proceeding GetBlockHeaders message is considered bad form, and may reduce the reputation
of the sending node.

### Transactions (0x02)

`[[nonce: P, receivingAddress: B_20, value: P, ...], ...]`

Specify (a) transaction(s) that the peer should make sure is included on its transaction
queue. The items in the list (following the first item `0x12`) are transactions in the
format described in the main Ethereum specification. Nodes must not resend the same
transaction to a peer in the same session. This packet must contain at least one (new)
transaction.

### GetBlockHeaders (0x03)

`[block: {P, B_32}, maxHeaders: P, skip: P, reverse: P in {0, 1}]`

Require peer to return a BlockHeaders message. Reply must contain a number of block
headers, of rising number when `reverse` is `0`, falling when `1`, `skip` blocks apart,
beginning at block `block` (denoted by either number or hash) in the canonical chain, and
with at most `maxHeaders` items.

### BlockHeaders (0x04)

`[blockHeader_0, blockHeader_1, ...]`

Reply to GetBlockHeaders. The items in the list (following the message ID) are block
headers in the format described in the main Ethereum specification, previously asked for
in a GetBlockHeaders message. This may validly contain no block headers if no block
headers were able to be returned for the GetBlockHeaders query.

### GetBlockBodies (0x05)

`[hash_0: B_32, hash_1: B_32, ...]`

Require peer to return a BlockBodies message. Specify the set of blocks that we're
interested in with the hashes.

### BlockBodies (0x06)

`[[transactions_0, uncles_0] , ...]`

Reply to GetBlockBodies. The items in the list are some of the blocks, minus the header,
in the format described in the main Ethereum specification, previously asked for in a
GetBlockBodies message. This may validly contain no items if no blocks were able to be
returned for the GetBlockBodies query.

### NewBlock (0x07)

`[[blockHeader, transactionList, uncleList], totalDifficulty]`

Specify a single block that the peer should know about. The composite item in the list
(following the message ID) is a block in the format described in the main Ethereum
specification.

- `totalDifficulty` is the total difficulty of the block (aka score).

### BlockHashesFromNumber (0x08)

`[number: P, maxBlocks: P]`

Requires peer to reply with a BlockHashes message. Message should contain block with
that of number `number` on the canonical chain. Should also be followed by subsequent
blocks, on the same chain, detailing a number of the first block hash and a total of
hashes to be sent. Returned hash list must be ordered by block number in ascending order.

### GetNodeData (0x0d)

`[hash_0: B_32, hash_1: B_32, ...]`

Require peer to return a NodeData message containing state trie nodes or contract code
matching the requested hashes.

### NodeData (0x0e)

`[value_0: B, value_1: B, ...]`

Provide a set of state trie nodes or contract code blobs which correspond to previously
asked hashes from GetNodeData. Does not need to contain all; best effort is fine. If it
contains none, then has no information for previous GetNodeData hashes.

### GetReceipts (0x0f)

`[blockHash_0: B_32, blockHash_1: B_32, ...]`

Require peer to return a Receipts message containing the receipts of the given block
hashes.

### Receipts (0x10)

`[[receipt_0, receipt_1], ...]`

Provide a set of receipts which correspond to previously asked in GetReceipts.

## Changelog

### eth/63 (2016)

Version 63 added the GetNodeData (0x0d), NodeData (0x0e), GetReceipts (0x0f) and Receipts
(0x10) messages which allow synchronizing transaction execution results.

### eth/62 (2015)

In version 62, the NewBlockHashes (0x01) message was extended to include block numbers
alongside the announced hashes. Messages GetBlockHashes (0x03), BlockHashes (0x04),
GetBlocks (0x05) and Blocks (0x06) were replaced by messages that fetch block headers and
bodies.

Previous encodings of the reassigned message codes were:

GetBlockHashes (0x03) `[hash: B_32, maxBlocks: P]`

BlockHashes (0x04) `[hash_0: B_32, hash_1: B_32, ...]`

GetBlocks (0x05) `[hash_0: B_32, hash_1: B_32, ...]`

Blocks (0x06) `[[blockHeader, transactionList, uncleList], ...]`

### eth/61 (2015)

Version 61 added the BlockHashesFromNumber (0x08) message.

### eth/60 and below

Version numbers below 60 were used during the Ethereum PoC development phase.

- `0x00` for PoC-1
- `0x01` for PoC-2
- `0x07` for PoC-3
- `0x09` for PoC-4
- `0x17` for PoC-5
- `0x1c` for PoC-6