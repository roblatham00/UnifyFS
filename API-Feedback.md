# Experiences with  Unify API

Using the Unify API to write a ROMIO driver


## Config Object

`unifyfs_initialize` : we can't turn MPI-IO info keys into config entities: initialize happens too early
- some things really need to be set up at Init
- can call it more than once: creates multiple clients
- separating "up front" from "deferable" items: could be a little bit of a headache?
- Unify expectation: not changing often, but defaults aren't great for every system


## Open vs close
- Trivial to look for `O_CREAT` flag but is the difference really necessary?
- expect excatly one person to create
- `unify_open` just a hash on the file
- everyone has to call `unify_open` or `unify_create`


## Semantics

- "commit" semantics vs "laminate" semantics
- no "close" only "laminate"
-- no state, so no need
-- lamination not required
-- "sync" -- make everything i've done locally visible to others
-- unify has "sync md about local writes" (enables remote reads) and "data sync"  (flush to storage)
- "A process may open and close a file many times" -- how?
- when does "move from local to permanent" happen?  Impossible through API, right?

## assorted minor build things

Pull requests on github

## server startup

- user provides some environment variables
- no way to select margo protocol
-- tcp/sockets fallback: can turn off the fallback

## mounting
- Transparent mounting not applicable to API right?
- Coordinating namespace between unifyfsd and ROMIO
- does "mounting" mean anything to API ?

## batched I/O
- what is the buffer reuse rule for `unify_io_req` memory buffer?  (do not touch until `unifyfs_wait_io` ?  Perhaps copied internally at `unifyfs_dispatch_io` ?)
-- don't touch until 'wait' -- no copies
- how many `unifyfs_io_request` can I post at once? (1000, hard coded)
-- not accesable programatically
- "use local extents" ?
-- a very specific use case:  "when you know remote clients will not make changes you wrote to"

- usleep in code instead of ABT primitives -- shouldn't these operations be done in a ULT?
- documentation says "asynchronous" but all the I/O happens in dispatch, right?
-- assume no i/o until `wait()`

- small writes require small log page:  big chunk will consume tons of data
-- future optimizations might handle an E3SM case better
-- "logio" file vs "striped file"
-- "try both ways"


## transfers
- not sure how to convey final location for Unify through ROMIO
-- no need to worry aobut that: 3rd party transfer
- probably makes sense to call `unifyfs_dispatch_transfer` and `unifyfs_wait_transfer` at mpi close time?

## userpsace utilities?

- Is there a `unifyfs-ls` ?
-- Staging out to verify the file

## Helpful notes to future-me

- "What is going wrong?"  -- set environment variable `UNIFYFS_LOG_VERBOSITY` to 5
- "How do I run on my laptop?"
  1. create `hostfile` similar to mpich machine file:  number of processes on a node and node name e.g. `1 localhost`
  1. set `UNIFYFS_SERVER_HOSTFILE` environment variable
  1. run `unifyfsd`

- `export UNIFYFS_LOGIO_CHUNK_SIZE=1024` needed to pass ROMIO tests.  What are the tradeoffs for such a small chunk size?
- `export  UNIFYFS_LOGIO_SHMEM_SIZE=0` when running the ROMIO test suite.  The
  shared memory regions are a kind of staging area.  The server removes them on
  its exit, not the clients.

## optimizations not yet implemented
- list-io optimization, but depends on how/when i/o happens
