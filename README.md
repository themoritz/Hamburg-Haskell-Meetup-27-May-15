# Trading Engine in Haskell

*Moritz Drexl - May 27, 2015*

Snippets can be found in [snippets.md](snippets.md).

## Background / history

- Started for fun as Haskell learning project
- Extended and used for testing purposes
- Optimized for speed (work in progress)

## Introduction: Trading engines

- What they do
  - Match orders between (stock) market participants
  - Execute trades
  - Manage accounts
- Challenges
  - Correct integer arithmetic
  - ACID guarantees
    - Transactions need to be atomic
    - Accounts and orderbook need to be consistent
  - Rollback
  - Speed

## How to write an engine in Haskell

- Integer arithmetic
  - Define units by wrapping integer in newtype
  - Define precision for each unit
  - Use type system to constrain which units can be multiplied and define the resulting unit
- ACID
  - Engine as a state transition system
    - Market as state
    - Orders as commands
    - Command -> Market -> (Event, Market)
    - => Engine monad transformer
  - Journalling of commands
- Rollback
  - Journalling of commands ensures state at any point in time can be rebuilt
- Speed
  - Efficient datatypes
  - Disruptor to handle concurrency without locking

## Program structure

- Get message from ZMQ
- Deserialize message + journal command
- Perform engine action depending on command/query
- Emit events telling what happened and answering queries

## Experience

- Few bugs, mainly business logic
- No crashes
- 115k transactions per second (ATM)
- While coding you see all the cases you haven't specified
