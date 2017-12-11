# Raft

## Foreword

The title of the Raft paper is "In Search of an Understandable Consensus Algorithm."
It was written by Diego Ongaro and John Ousterhout of Stanford University.

## Introduction

The primary goal of Raft was understandability.
Two techniques were used to do so:

* decomposition
* state space reduction

Main features include the following:

* Strong leaders
  * Logs only flow from the leader to other servers.
* Leader election
  * Raft uses randomized timers to elect leaders.
* Membership changes
  * A joint consensus mechanism allows for normal operationg during configuration changes

## Replicated State Machines

Most consensus algorithms rely on each server keeping a copy of the state log.
The state machine on each server runs through its state log in order.
Because the logs are kept in check across machines, the states can all be assumed to be the same.

## Paxos Drawbacks

Its single decree subest is likely the cause of difficulty.
There are two stages, and these stages are both dense and subtle.
When multi-Paxos is used, the difficulty is greatly compounded.

The second problem is that implementations are difficult becuase there is no widely agreed-upon algorithm for multi-Paxos.

## Designing for Understandability

The primary value in making design decisions was understandability.

