# haste

## **PRELIMINARY WORK-IN-PROGRESS**

## Overview

Haste is a Decred Proof-of-Work (PoW) mining protocol which is used primarily
for pool mining.  This protocol was created to support the voting functionality
of Decred such as voteBits and proof-of-voting which are used to deploy
smooth hardforks via Proof-of-Stake (PoS).

# haste 1.0

## Overview

haste 1.0 is a slight alteration of the widely used stratum mining protocol.
By leveraging the familiar stratum protocol, this ensures faster uptake by
miners and mining pool operators.

## Modifications

The main modification from plain stratum is the use of the
Coinbase2 field to provide a path to the coinbase.  The use
of this field is described in more detail below.

## Stratum Session Startup

Stratum is a series of JSON messages with an incremented ID to pair
messages and replies.

### Client sends mining.subscribe message to the stratum server to initiate the session:

```no-highlight
{"id": 1, "method": "mining.subscribe", "params": ["gominer/0.2.0-decred"]}
```

### Server sends mining.notify combination message in response to mining.subscribe

```no-highlight
{"id":1,"result":[[["mining.set_difficulty","1"],["mining.notify","2bd595e34826a3b6271400920d4decb8"]],"0000000000000000e3014335",12],"error":null}
```

The fields are:

| Field Name               | Example                          | Notes                                                                      |
| ------------------------ | -------------------------------- | -------------------------------------------------------------------------- |
| Subscription Type ID     | 1                                | Never used in practice.                                                    |
| Stratum Session ID       | 2bd595e34826a3b6271400920d4decb8 | Never used in practice.                                                    |
| Extranonce1              | 0000000000000000e3014335         | Unique session string which will be used for coinbase serialization later. |
| Extranonce2 Length       | 12                               | Formatted length of Extra Nonce that the pool expects the miner to use.    |

### Client sends mining.authorize so work is correctly associated with an account/payment address

```no-highlight
{"id": 2, "method": "mining.authorize", "params": ["user.workerid", "password"]}
```

### Server sends true if authorization succeeded or false with error message if authorization failed

```no-highlight
{"id":2,"result":true,"error":null}
```

### **TODO** Should mention mining.extranonce.subscribe to be complete. Sent by ccminer/sgminer but no server messages are sent other than an initial result:false/true response.  Maybe not widely used?

## Stratum Work Data And Target

After subscribing to server pushes and performing authorization,
the server will send mining.difficulty and mining.notify messages.
Typically this occurs for the first time in a session directly after
the mining.subscribe reply so the miner can start performing work
immediately.  Then the messages are sent on as-needed basis.  For
example, the server will send new work if a new block has appeared on the
network.

These messages inform the mining software what work to perform
and the target/difficulty of the work.  These messages do not contain
an ID field because no message is sent in response to them.

### mining.difficulty

Stratum servers can be either be fixed or auto-adjusting difficulty.
Fixed difficulty pools will only set the difficulty at the beginning of a
session. Auto-adjusting difficulty pools adapt to the speed of the miner.
The pool will increase or decrease the difficulty in an attempt to have the
miner to submit at a fixed interval (typically at least once per minute).
An example of such a message is below:

```
{"id":null,"method":"mining.set_difficulty","params":[8]}
```

### mining.notify

This message contains everything necessary to derive work which will be passed
to a hashing device. An example message is below:

```
{"id":null,"method":"mining.notify","params":["76df","7817c24aa99f3999a57dcfc8a7a834f92ebb442f8d519dbd000009e000000000","d52f367013ddc74d61a4f50c0d47c4b8e87b6d89a603a04447bcd2b110c508e31d37308253d38bbe0464508f4eb1f12b92e6431d41b01f01518d2d9b9b64fe93010098588e9b1c650500030052a90000b778171a134e691e01000000b1c70000ce0f0000d052a1570000000000000000","",[],"01000000","1a1778b7","57a152d0",false]}
```

The fields contained in params are:

| Field Name     | Purpose           | Example  |
| -------------- | ------------- | ----- |
| JobID          | ID of the job. Used when submitting a solved shared to the server.                             | 187 |
| PrevHash       | Hash of the previous block.  Used when deriving work.                                          | 7817c24aa99f3999a57dcfc8a7a834f92ebb442f8d519dbd000009e000000000 |
| CoinBase1      | Initial part of the coinbase transaction.  Used when deriving work.                            | d52f367013ddc74d61a4f50c0d47c4b8e87b6d89a603a04447bcd2b110c508e31d37308253d38bbe0464508f4eb1f12b92e6431d41b01f01518d2d9b9b64fe93010098588e9b1c650500030052a90000b778171a134e691e01000000b1c70000ce0f0000d052a1570000000000000000 |
| CoinBase2      | **TODO** Explain haste use.                                                                    | **TODO** Empty for now |
| MerkleBranches | Array of merkle branches.  **TODO** Used anywhere?                                             | **TODO** Empty array |
| BlockVersion   | Decred block version.  Used when deriving work.                                                | 01000000 |
| Nbits          | Encoded current network difficulty.  Used for informational purposes.                          | 1a1778b7 |
| Ntime          | Server's time when the job was transmitted. Used when submitting a solved share to the server. | 57a152d0 |
| CleanJobs      | When true, discard current work and re-calculate derived work.                                 | false |

### **TODO** Explain Ntime is supposed to be changeable but Decred pools don't seem to support this?

### **TODO** Explain CleanJobs/work management some more but gominer doesn't do any work management at the moment which seems to work okay.  sgminer seems to aggressively queue and discard work.  ccminer seems to only discard current work when cleanjobs: true and roll work (re-use the same work but increment nonce) when no new work is available.

## Deriving Work And Performing Hashing

### Converting Stratum Work To "getwork" Style Work

#### Construct a Block Header 

Field          | Description                                                                            | Size
---            | ---                                                                                    | ---
Version        | Decred Block header version                                                            | 4 bytes
PrevHash       | Hash of the previous block                                                             | 32 bytes
CoinBase1      | Merkle tree hash calculated using all transactions in the block                        | 108 bytes
TimeStamp      | Job Ntime                                                                              | 4 bytes
Random Data    | Can be used as a miners mark or as an extra nonce for fast hashing devices like ASICs. | 4 bytes

### Calculate Midstate

The midstate is obtained by calling the BLAKE-256 hashing function on the first
two 64-byte units of the block header.

### Performing Hashing / Mining

The midstate is then passed to the device code.  Mining is performed by incrementing
the nonce and the device code currently returns whenever a difficulty 1 share
is found.  The target is checked by the mining program and false positives below
target are discarded and hashing resumes.

If a valid solution above target is found then the share is prepared and submitted.

### **TODO** Explain conversion from stratum difficulty to target somewhere.

### **TODO** Explain Extra Nonce formatting here or in next section.

### **TODO** Add data formats/pseudo-code for each step when proof-of-voting is complete.

## Submitting A Share

### Client sends mining.submit message with params from pool/calculated from above

```
> {"method": "mining.submit", "params": ["user.worker", "187", "0100000000188fece3014335", "5783c78e", "f6c4e01d"], "id":4}
```

The fields contained in params are:

| Field Name               | Example                  |
| ------------------------ | ------------------------ |
| Username/Payment Address | user.worker              |
| JobID (From Pool)        | 187                      | 
| Extra Nonce              | 0100000000188fece3014335 |
| Ntime (From Pool)        | 5783c78e                 |
| Nonce                    | f6c4e01d                 |

### **TODO** Not sure if the proof-of-voting mechanism will required an additional field or not.

### Server sends true if share accepted or false with error message if share rejected

```
{"id":4,"result":true,"error":null}
```

Less than 1% of shares submitted should be rejected.  The most typical error
message is "job not found" which happens when submitting an old share that was
solved using stale work.  This is typically a race condition which occurs when
a new block was just propagated across the network and the miner's work was not
updated prior to solving the share.

## Golang Example Code

### **TODO** Adapt (test) code from gominer as an example once proof-of-voting is functional.

# haste 2.0

## Overview

Future work may include a paring down or outright replacement of the stratum
portion of the protocol to make it more efficient.  However, no priority or
timeline has been established.