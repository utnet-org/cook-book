# Overview

This document describes the high-level architecture of node. The focus here
is on the protocol itself.

## Bird's Eye View

If we put the entirety of framework onto one picture, we get something like this:

![](../images/architecture.svg)

Don't worry if this doesn't yet make a lot of sense: hopefully, by the end of
this document the above picture would become much clearer!

## Overall Operation

`node` is a blockchain node -- it's a single binary (`unc-node`) which runs on
some machine and talks to other similar binaries running elsewhere. Together,
the nodes agree (using a distributed consensus algorithm) on a particular
sequence of transactions. Once transaction sequence is established, each node
applies transactions to the current state. Because transactions are fully
deterministic, each node in the network ends up with identical state. To allow
greater scalability

Major components of arch:

* **node**. node blockchain.

* **miner**.  miner.

* **container-cloud**.  container...
