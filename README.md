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

## Input data (from stratum server)

**TODO describe field formats and sizes thoroughly**

* JobID - ID of the job. Used as-is when submitting a share back to the server.
* PrevHash - Hash of previous block.  Used when constructing a share.
* CoinBase1 - Initial part of coinbase transaction.
* CoinBase2 - **Not finalized at this time.**
* MerkleBranches - Unused at this time.
* Version - Decred block version in little endian.
* NBits - Encoded current network difficulty.
* Ntime - Server's time at the time of job transmission.
* CleanJobs - When true, server indicates that submitting shares from previous jobs don't have a sense and such shares will be rejected. When this flag is set, miner should also drop all previous jobs, so job_ids can be eventually rotated.

## Hashing setup and process

* construct blockheader documentation + pseudo code?
* calculate midstate + pseudo code?
* submitting to GPU
* difficulty/target checking + pseudo code?

## Submitting a solution

* document how to construct a mining.submit message + pseudo code

## Example Code

* Actual compilable and working code examples in golang (Make a minimal example
by copying & pasting out of gominer?)

# haste 2.0

## Overview

Future work may include a paring down or outright replacement of the stratum
portion of the protocol to make it more efficient.  However, no priority or
timeline has been established.