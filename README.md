# Requirements and How to Use ForgeDB

## Requirements

- **CMake 3.16+**
- A **C++17-compatible compiler**
- **Bash**

## Build

If required, make the scripts executable:

```bash
chmod +x scripts/*.sh
```

Build the complete project:

```bash
./scripts/build.sh
```

This creates the `build/` directory and builds the ForgeDB server, CLI client, tests, and benchmarks.

## Run

Start the ForgeDB server:

```bash
./build/forgedb
```

Open a second terminal and start the CLI client:

```bash
./build/forgedb_cli
```

## Tests

Run all tests:

```bash
./scripts/run_tests.sh
```

## Benchmarks

```bash
./build/write_benchmark
./build/read_benchmark
./build/sync_vs_async
```

## Clean

Remove generated build and runtime data:

```bash
./scripts/clean.sh
```


# ForgeDB

A lightweight, persistent key-value database built from scratch in **C++17**.

ForgeDB is designed as an educational systems project that demonstrates how the core pieces of a database work together: command parsing, networking, in-memory storage, write-ahead logging, crash recovery, immutable sorted storage files, manifests, compaction, checksums, and concurrency.

The project is intentionally built from low-level components instead of relying on an existing database engine.

---

## Table of Contents

- [Features](#features)
- [Architecture](#architecture)
- [How ForgeDB Works](#how-forgedb-works)
- [Project Structure](#project-structure)
- [Building the Project](#building-the-project)
- [Running ForgeDB](#running-forgedb)
- [Command-Line Client](#command-line-client)
- [Supported Commands](#supported-commands)
- [Durability and Crash Recovery](#durability-and-crash-recovery)
- [Storage Engine](#storage-engine)
- [Write Path](#write-path)
- [Read Path](#read-path)
- [Delete Path](#delete-path)
- [SSTables](#sstables)
- [Manifest](#manifest)
- [Compaction](#compaction)
- [Checksums and Corruption Detection](#checksums-and-corruption-detection)
- [Concurrency](#concurrency)
- [Testing](#testing)
- [Fault Injection and Crash Tests](#fault-injection-and-crash-tests)
- [Technical Concepts Demonstrated](#technical-concepts-demonstrated)
- [Current Limitations](#current-limitations)
- [Future Improvements](#future-improvements)
- [License](#license)

---

## Features

ForgeDB currently includes:

### Database Features

- Persistent key-value storage
- `PUT`, `GET`, and `DELETE` operations
- In-memory **MemTable**
- Durable **Write-Ahead Log (WAL)**
- Automatic recovery after restart
- Recovery from an incomplete final WAL record
- Delete tombstones
- Immutable **SSTables**
- Metadata-based table tracking through a **MANIFEST**
- Automatic MemTable flushing
- SSTable lookup from newest table to oldest table
- Automatic compaction
- Tombstone removal during full compaction
- CRC32 checksums for WAL record validation
- Binary serialization and deserialization
- Configurable durability behavior
- Thread-safe database command execution

### Server and Client Features

- TCP server
- Client command-line interface
- Request/response protocol
- Command parsing
- Multiple supported client commands

### Reliability and Testing Features

- Unit tests
- Storage tests
- Restart persistence tests
- WAL recovery tests
- Compaction tests
- Concurrency tests
- Crash-after-write tests
- Crash-during-recovery tests
- Crash-during-compaction tests
- Fault injection support

All current tests pass:

```text
100% tests passed out of 11
```

---

# Architecture

ForgeDB follows a layered architecture:

```text
Client
  |
  v
Network Protocol
  |
  v
Command Parser
  |
  v
Database
  |
  +-----------------------------+
  |                             |
  v                             v
MemTable                     Storage
  |                             |
  |                             +--> WAL
  |                             |
  |                             +--> Recovery
  |                             |
  |                             +--> SSTables
  |                             |
  |                             +--> MANIFEST
  |                             |
  +-----------------------------+
                |
                v
           Compaction
```

The basic idea is:

1. A client sends a command.
2. The server receives and parses it.
3. The `Database` executes the operation.
4. Writes are first recorded in the WAL.
5. The newest data is kept in the MemTable.
6. When the MemTable becomes large enough, it is flushed to an SSTable.
7. The MANIFEST tracks all active SSTables.
8. Compaction merges multiple SSTables.
9. After a restart, ForgeDB reconstructs the latest state from SSTables and the WAL.

---

# How ForgeDB Works

## A Simple Example

Suppose the user executes:

```text
PUT name Mohit
```

ForgeDB performs approximately these steps:

```text
1. Parse the command
        |
        v
2. Append the operation to the WAL
        |
        v
3. Store the value in the MemTable
        |
        v
4. Return OK
```

Later, if the MemTable reaches its configured flush threshold:

```text
MemTable
   |
   v
Create SSTable
   |
   v
Add SSTable metadata to MANIFEST
   |
   v
Reset WAL
   |
   v
Clear MemTable
```

This means data eventually moves from:

```text
WAL + MemTable
        |
        v
     SSTables
```

---

# Project Structure

```text
ForgeDB/
├── CMakeLists.txt
│
├── include/
│   └── forgedb/
│       ├── command.hpp
│       ├── parser.hpp
│       ├── database.hpp
│       ├── memtable.hpp
│       ├── storage.hpp
│       ├── wal.hpp
│       ├── recovery.hpp
│       ├── crc32.hpp
│       ├── serialization.hpp
│       ├── snapshot.hpp
│       ├── sstable.hpp
│       ├── manifest.hpp
│       ├── compaction.hpp
│       ├── durability.hpp
│       └── ...
│
├── src/
│   ├── main.cpp
│   │
│   ├── command/
│   │   ├── parser.cpp
│   │   └── command.cpp
│   │
│   ├── database/
│   │   ├── database.cpp
│   │   ├── memtable.cpp
│   │   └── concurrency.cpp
│   │
│   ├── storage/
│   │   ├── storage.cpp
│   │   ├── wal.cpp
│   │   ├── recovery.cpp
│   │   ├── crc32.cpp
│   │   ├── serialization.cpp
│   │   ├── snapshot.cpp
│   │   ├── sstable.cpp
│   │   ├── compaction.cpp
│   │   └── manifest.cpp
│   │
│   ├── network/
│   │   ├── server.cpp
│   │   ├── connection.cpp
│   │   ├── protocol.cpp
│   │   └── event_loop.cpp
│   │
│   └── testing/
│       └── fault_injection.cpp
│
├── client/
│   ├── main.cpp
│   └── cli.cpp
│
└── tests/
    ├── unit/
    ├── integration/
    └── crash/
```

---

# Building the Project

## Requirements

You need:

- A C++17-compatible compiler
- CMake 3.16 or newer
- A Unix-like environment for the current project setup

### macOS

The project can be built with the compiler provided by Xcode Command Line Tools.

```bash
xcode-select --install
```

## Configure the Build

From the project root:

```bash
cmake -S . -B build
```

This tells CMake:

```text
-S .        Source directory = current directory
-B build    Build directory  = ./build
```

## Build

```bash
cmake --build build
```

After a successful build, the generated executables are placed in the `build/` directory.

---

# Running ForgeDB

From the project root:

```bash
./build/forgedb
```

The server starts and waits for client connections.

The exact server configuration and startup behavior are defined by the project's networking and entry-point code.

---

# Command-Line Client

Build the project:

```bash
cmake -S . -B build
cmake --build build
```

Then run the client:

```bash
./build/forgedb_cli
```

The client communicates with the ForgeDB server using the project's network protocol.

---

# Supported Commands

## PUT

Stores or updates a key.

```text
PUT <key> <value>
```

Example:

```text
PUT name Mohit
```

Response:

```text
OK
```

Values may contain spaces:

```text
PUT message Hello this is ForgeDB
```

---

## GET

Retrieves the latest value associated with a key.

```text
GET <key>
```

Example:

```text
GET name
```

If the key exists:

```text
VALUE Mohit
```

If the key does not exist:

```text
NOT_FOUND
```

---

## DELETE

Deletes a key.

```text
DELETE <key>
```

The short form is also supported:

```text
DEL <key>
```

Example:

```text
DEL name
```

Response:

```text
OK
```

ForgeDB does not immediately remove the key from all older storage files. Instead, it stores a **tombstone** indicating that the key has been deleted.

---

## EXIT

Ends the client session.

```text
EXIT
```

The parser also supports:

```text
QUIT
```

The database responds with:

```text
BYE
```

---

# Durability and Crash Recovery

One of the most important goals of ForgeDB is to demonstrate how a database can recover data after a restart or crash.

The central mechanism for this is the **Write-Ahead Log**.

## What Is a WAL?

A WAL is a file that records changes before those changes are considered committed to the in-memory database state.

For a write:

```text
PUT name Mohit
```

ForgeDB performs:

```text
Append PUT operation to WAL
        |
        v
Update MemTable
        |
        v
Return success
```

This ordering is important.

If the process crashes after the WAL write but before the MemTable data is permanently flushed, the operation can still be reconstructed from the WAL.

---

# Recovery After Restart

When ForgeDB starts, it reconstructs the database state in this order:

```text
1. Load MANIFEST
        |
        v
2. Load SSTables from oldest to newest
        |
        v
3. Replay WAL
        |
        v
4. WAL entries overwrite older SSTable entries
        |
        v
5. Replace the MemTable with the recovered latest state
```

Conceptually:

```text
Oldest SSTable
      |
      v
Newer SSTable
      |
      v
Newest SSTable
      |
      v
WAL
      |
      v
Latest database state
```

Newer values always override older values.

This also applies to delete tombstones.

---

# Incomplete WAL Tail Recovery

A crash can happen while the final WAL record is being written.

For example:

```text
Record 1: complete
Record 2: complete
Record 3: partially written
              ^
              crash
```

ForgeDB treats this as an incomplete final record.

The recovery logic:

- Keeps all earlier complete records.
- Ignores the incomplete final record.
- Returns `RECOVERED_INCOMPLETE_TAIL`.

This prevents a partial final write from destroying the earlier valid database state.

---

# Corruption Detection

Not every invalid WAL record is treated as a recoverable incomplete tail.

ForgeDB distinguishes between:

```text
Incomplete final write
        |
        v
Potentially recoverable
```

and:

```text
Invalid record / checksum mismatch
        |
        v
Corruption
```

A corrupted WAL returns an error instead of silently continuing with potentially incorrect data.

---

# Storage Engine

The storage subsystem is responsible for:

- WAL management
- Recovery
- SSTable creation
- SSTable reads
- MANIFEST management
- MemTable flushing
- Compaction
- WAL reset after a successful flush

The main abstraction is:

```text
Database
    |
    v
Storage
    |
    +--> WAL
    +--> Recovery
    +--> SSTables
    +--> MANIFEST
    +--> Compaction
```

---

# Write Path

For a `PUT` command:

```text
PUT key value
    |
    v
Append to WAL
    |
    v
Update MemTable
    |
    v
Check MemTable size
    |
    +--> Below threshold
    |        |
    |        v
    |      Return OK
    |
    +--> Above threshold
             |
             v
       Flush MemTable
             |
             v
       Create SSTable
             |
             v
       Update MANIFEST
             |
             v
          Reset WAL
             |
             v
       Clear MemTable
             |
             v
       Maybe compact
```

The important property is:

```text
WAL write happens before the MemTable update.
```

This is the write-ahead rule.

---

# Read Path

For a `GET key` command, ForgeDB first checks the newest in-memory state.

```text
GET key
   |
   v
Check MemTable
   |
   +--> Found?
   |      |
   |      +--> Deleted --> NOT_FOUND
   |      |
   |      +--> Value   --> Return value
   |
   +--> Not found
          |
          v
     Search SSTables
     newest --> oldest
          |
          +--> Found?
          |      |
          |      +--> Tombstone --> NOT_FOUND
          |      |
          |      +--> Value --> Return value
          |
          +--> Not found --> NOT_FOUND
```

The MemTable is checked first because it contains the newest unflushed state.

SSTables are searched from newest to oldest because newer tables contain newer versions of keys.

---

# Delete Path

When deleting a key:

```text
DELETE name
```

ForgeDB performs:

```text
Append DELETE to WAL
        |
        v
Store tombstone in MemTable
        |
        v
Flush later if required
```

The tombstone is represented conceptually as:

```text
key -> {
    value: "",
    deleted: true
}
```

This is important because an older SSTable may still contain:

```text
name -> Mohit
```

Without a tombstone, a later read could accidentally find and return that old value.

The tombstone tells ForgeDB:

```text
This key existed before, but its newest state is deleted.
```

---

# MemTable

The MemTable is the active in-memory storage structure.

Internally, it stores key metadata similar to:

```cpp
struct MemTableEntry {
    std::string value;
    bool deleted = false;
};
```

A key can therefore represent either:

```text
Live value:
key -> { "Mohit", false }
```

or:

```text
Deleted value:
key -> { "", true }
```

The MemTable also tracks its approximate size in bytes.

When the configured threshold is reached, the database flushes the current MemTable into an SSTable.

The current threshold is:

```text
1 MiB
```

defined as:

```cpp
1024 * 1024
```

---

# SSTables

An SSTable is an immutable on-disk storage file.

After a MemTable is flushed, the resulting data is written to an SSTable.

Conceptually:

```text
MemTable
    |
    v
+-------------------+
| key1 -> value1    |
| key2 -> value2    |
| key3 -> deleted   |
+-------------------+
    |
    v
SSTable
```

Once written, an SSTable is not modified in place.

Newer writes create newer storage state instead of modifying old SSTables directly.

This design simplifies crash recovery and is a core idea behind LSM-style storage engines.

---

# SSTable Metadata

Each SSTable has metadata containing information such as:

```text
Table ID
Filename
Smallest key
Largest key
Entry count
```

Conceptually:

```cpp
struct SSTableMetadata {
    uint64_t id;
    std::string filename;
    std::string smallest_key;
    std::string largest_key;
    uint64_t entry_count;
};
```

This metadata is stored in the MANIFEST.

---

# Manifest

The MANIFEST is the database's metadata file.

It keeps track of the currently active SSTables.

The MANIFEST contains information such as:

```text
Next table ID
Number of tables
Metadata for every active table
```

Example conceptually:

```text
next_table_id = 6

tables:
    table_1.sst
    table_2.sst
    table_3.sst
    table_5.sst
```

The MANIFEST is important because the database should know exactly which SSTables belong to the current valid database state.

---

# Atomic MANIFEST Replacement

When persisting MANIFEST changes, ForgeDB uses a temporary file:

```text
MANIFEST.tmp
```

The process is:

```text
Write new MANIFEST.tmp
        |
        v
Flush and close it
        |
        v
Rename it to MANIFEST
```

This is safer than overwriting the existing MANIFEST directly.

The goal is to reduce the risk of leaving a partially written MANIFEST as the active metadata file.

---

# MemTable Flush

When the MemTable reaches the configured threshold, ForgeDB flushes it.

The flush sequence is:

```text
1. Take a MemTable snapshot
        |
        v
2. Write a new SSTable
        |
        v
3. Add the new SSTable to the MANIFEST
        |
        v
4. Reset the WAL
        |
        v
5. Clear the MemTable
```

The ordering matters.

ForgeDB does not reset the WAL until the data has been successfully written to an SSTable and published through the MANIFEST.

This helps prevent losing data during the transition from memory/log storage to persistent table storage.

---

# Compaction

Over time, repeated flushes create multiple SSTables:

```text
table_1.sst
table_2.sst
table_3.sst
table_4.sst
table_5.sst
...
```

Too many tables make reads more expensive because the database may need to search several files.

Compaction solves this by merging tables.

The configured compaction threshold is:

```text
4 SSTables
```

When the threshold is reached, ForgeDB can merge the tables.

---

# Compaction Process

Conceptually:

```text
Old SSTables
    |
    +--> table_1
    +--> table_2
    +--> table_3
    +--> table_4
              |
              v
         Merge oldest
         to newest
              |
              v
        Newer values win
              |
              v
      Remove obsolete tombstones
              |
              v
        Write replacement table
              |
              v
       Publish new table in MANIFEST
              |
              v
      Remove old tables from MANIFEST
              |
              v
       Delete old physical files
```

The merge direction is important:

```text
oldest --> newest
```

When the same key appears multiple times, the newest version overwrites the older version.

Example:

```text
table_1:
    name = Alice

table_2:
    name = Mohit
```

After compaction:

```text
name = Mohit
```

---

# Tombstone Removal During Compaction

Suppose:

```text
table_1:
    name = Mohit

table_2:
    name = TOMBSTONE
```

During a full compaction of all active tables, the older value is no longer needed.

The result can remove the key completely.

Why is this safe?

Because there are no older active SSTables left that could contain an old value that might reappear.

This is why tombstone cleanup is performed after merging all active tables in the current compaction design.

---

# Crash Safety During Compaction

Compaction follows a publish-before-delete approach.

The simplified order is:

```text
1. Create replacement SSTable
        |
        v
2. Publish replacement in MANIFEST
        |
        v
3. Remove old table metadata
        |
        v
4. Delete old physical SSTable files
```

This ordering is designed so that a crash should not cause all copies of the data to disappear before the replacement has been published.

The project also includes a dedicated crash-during-compaction test.

---

# Write-Ahead Log Format

The WAL stores binary records.

A record is conceptually structured as:

```text
+------------------+
| Record Length    |
+------------------+
| Record Payload   |
+------------------+
| CRC32 Checksum   |
+------------------+
```

The payload contains the logical operation, such as:

```text
PUT
key
value
```

or:

```text
DELETE
key
```

The outer length and checksum allow recovery to validate records.

---

# CRC32 Checksums

ForgeDB uses CRC32 to detect WAL corruption.

When writing a record:

```text
Payload
   |
   v
Calculate CRC32
   |
   v
Store checksum with record
```

During recovery:

```text
Read payload
   |
   v
Calculate CRC32 again
   |
   v
Compare with stored checksum
```

If they do not match:

```text
WAL checksum mismatch
```

and recovery reports corruption.

This prevents ForgeDB from silently trusting damaged WAL data.

---

# Binary Serialization

ForgeDB includes dedicated serialization and deserialization logic for binary storage records.

The general responsibility is:

```text
In-memory object
      |
      v
Serialize
      |
      v
Binary bytes
```

and during reading:

```text
Binary bytes
      |
      v
Deserialize
      |
      v
In-memory object
```

Keeping this logic separate makes the storage format easier to reason about and test.

---

# Recovery States

WAL recovery can produce statuses conceptually including:

```text
SUCCESS
RECOVERED_INCOMPLETE_TAIL
CORRUPTED
IO_ERROR
```

### SUCCESS

The WAL was read successfully.

### RECOVERED_INCOMPLETE_TAIL

The final record was incomplete, but earlier complete records were recovered successfully.

### CORRUPTED

The WAL contained invalid or corrupted data.

### IO_ERROR

A storage I/O error occurred.

---

# Concurrency

ForgeDB protects database command execution with a mutex.

Conceptually:

```text
Thread A ----\
              \
Thread B ------> Database mutex --> Database state
              /
Thread C ----/
```

This prevents concurrent operations from corrupting shared database state.

The project also includes a concurrency test to verify behavior when multiple operations are executed concurrently.

This is a simple correctness-oriented concurrency model rather than a high-performance lock-free design.

---

# Networking

ForgeDB includes a networking layer with components for:

- Server startup
- Connections
- Request/response protocol
- Event loop management

The high-level flow is:

```text
Client
  |
  | TCP request
  v
Server
  |
  v
Connection handling
  |
  v
Protocol decoding
  |
  v
Command parsing
  |
  v
Database execution
  |
  v
Response
  |
  v
Client
```

This separates network communication from the core database logic.

The database itself does not need to know whether a command came from a local CLI, a socket, or another future interface.

---

# Testing

ForgeDB uses CTest for automated testing.

Run all tests with:

```bash
ctest --test-dir build --output-on-failure
```

The current test suite contains 11 tests.

---

## 1. CRC32 Test

```text
test_crc32
```

Tests checksum calculation behavior.

This verifies that the corruption-detection mechanism produces expected results.

---

## 2. Protocol Test

```text
test_protocol
```

Tests the network protocol serialization and parsing behavior.

---

## 3. Parser Test

```text
test_parser
```

Tests command parsing, including:

```text
PUT
GET
DEL
EXIT
Invalid commands
```

It also verifies values containing spaces.

Example:

```text
PUT message Hello this is ForgeDB
```

---

## 4. SSTable Test

```text
test_sstable
```

Tests writing and loading SSTable data.

This verifies that data can move correctly between in-memory structures and persistent table files.

---

## 5. Restart Test

```text
test_restart
```

Tests persistence across a database restart.

The general idea is:

```text
Start database
    |
    v
Write data
    |
    v
Destroy database
    |
    v
Create database again
    |
    v
Recover persisted data
```

---

## 6. Recovery Test

```text
test_recovery
```

Tests WAL recovery behavior.

This verifies that data written before a restart can be reconstructed.

---

## 7. Compaction Test

```text
test_compaction
```

Tests automatic SSTable compaction and verifies that values remain correct after tables are merged.

---

## 8. Concurrency Test

```text
test_concurrency
```

Tests database behavior under concurrent access.

The goal is to detect incorrect results or shared-state corruption caused by multiple threads.

---

## 9. Crash After Write

```text
crash_after_write
```

Tests that a successfully logged write can survive a simulated crash/restart scenario.

---

## 10. Crash During Recovery

```text
crash_during_recovery
```

Tests recovery behavior when WAL data contains an incomplete final record.

---

## 11. Crash During Compaction

```text
crash_during_compaction
```

Tests that data remains recoverable when a simulated crash occurs during compaction.

---

# Running Individual Tests

Examples:

```bash
./build/test_crc32
```

```bash
./build/test_parser
```

```bash
./build/test_restart
```

```bash
./build/test_recovery
```

```bash
./build/test_compaction
```

```bash
./build/test_concurrency
```

```bash
./build/crash_after_write
```

```bash
./build/crash_during_recovery
```

```bash
./build/crash_during_compaction
```

---

# Fault Injection

ForgeDB includes testing support for fault injection.

The purpose of fault injection is to simulate failures at controlled points in the storage lifecycle.

Examples of situations that can be tested include:

```text
Write succeeds, then process crashes
```

```text
Recovery sees an incomplete final record
```

```text
Compaction is interrupted
```

Fault injection is useful because many storage bugs are difficult to reproduce naturally.

Instead of waiting for a real crash at exactly the wrong moment, tests can deliberately simulate the failure.

---

# Technical Concepts Demonstrated

This project demonstrates practical knowledge of several important systems concepts.

## C++ Fundamentals

- RAII
- Smart pointers
- Move semantics
- `std::unique_ptr`
- `std::unordered_map`
- `std::vector`
- Mutexes
- File streams
- Binary I/O
- Error handling
- Object lifetime management

## Operating Systems Concepts

- Files and persistent storage
- Process crash scenarios
- Atomic rename behavior
- Directory creation
- File truncation
- Durable logging
- Failure ordering

## Database Concepts

- Key-value storage
- MemTables
- Write-Ahead Logging
- Recovery
- Tombstones
- SSTables
- LSM-style storage
- Compaction
- Manifest metadata
- Crash consistency
- Checksums

## Computer Networks

- TCP server/client communication
- Connections
- Request/response protocols
- Message serialization
- Event-loop based architecture

## Concurrency

- Shared mutable state
- Mutual exclusion
- Thread safety
- Concurrent testing

## Software Engineering

- Modular architecture
- Header/source separation
- CMake builds
- Unit tests
- Integration tests
- Crash tests
- Fault injection
- Clear subsystem boundaries

---

# Current Limitations

ForgeDB is an educational database and does not currently aim to compete with production systems such as RocksDB, LevelDB, Redis, or SQLite.

Some current limitations include:

- The MemTable uses a hash map rather than an ordered tree.
- SSTable lookup is not yet optimized with indexes.
- There are no Bloom filters.
- There are no multiple compaction levels.
- The entire database command path is protected by a single mutex.
- Compaction is synchronous.
- There is no background flush thread.
- There is no transaction support.
- There are no snapshots exposed as a full multi-version API.
- There is no replication.
- There is no authentication.
- There is no query language beyond the key-value commands.
- Storage formats are intended for learning and project development rather than long-term compatibility guarantees.

These limitations are intentional areas for future systems work.

---

# Future Improvements

Possible future improvements include:

## Storage

- Bloom filters for faster negative lookups
- Sparse indexes for SSTables
- Block-based SSTables
- Compression
- Prefix compression
- Multiple LSM levels
- More advanced compaction strategies
- Background compaction

## Concurrency

- Reader/writer locking
- Sharded locking
- Per-key or per-MemTable synchronization
- Background flush workers
- Reduced global lock contention

## Durability

- Stronger fsync semantics
- WAL segment rotation
- More detailed crash injection points
- Atomic directory syncing where appropriate
- Recovery metrics and diagnostics

## Networking

- More efficient event loop handling
- Better client protocol framing
- Connection pooling
- Request IDs
- Pipelined requests

## Database Features

- TTL support
- Expiration
- Range scans
- Prefix scans
- Iterators
- Batches
- Transactions
- Snapshots
- Metrics
- Configuration files

---

# Design Philosophy

ForgeDB is built with a focus on understanding the underlying systems rather than hiding them behind a large framework.

The project answers practical questions such as:

- How does a database remember data after restarting?
- Why write to a log before updating memory?
- How can incomplete writes be recovered?
- Why are immutable files useful?
- How are deleted keys represented?
- How do multiple storage files get merged?
- How can a database detect corrupted data?
- What happens if a crash occurs during compaction?
- How do concurrent commands avoid corrupting shared state?

The goal is to make these concepts visible in the implementation.

---

# Example Data Lifecycle

Suppose the following commands are executed:

```text
PUT name Alice
PUT city Delhi
PUT name Mohit
DEL city
```

The latest logical state is:

```text
name -> Mohit
city -> deleted
```

Initially, the state may exist in:

```text
WAL
+
MemTable
```

After a flush:

```text
SSTable
```

After more writes and more flushes:

```text
table_1.sst
table_2.sst
table_3.sst
table_4.sst
```

During compaction:

```text
table_1
table_2
table_3
table_4
    |
    v
Merge oldest -> newest
    |
    v
Keep newest value for each key
    |
    v
Remove obsolete tombstones
    |
    v
replacement table
```

The final state remains:

```text
name -> Mohit
```

and the deleted key remains absent.

---

# Why the Ordering Matters

A major theme of ForgeDB is operation ordering.

For example, this is safer:

```text
1. Write to WAL
2. Update MemTable
3. Flush to SSTable
4. Update MANIFEST
5. Reset WAL
6. Clear MemTable
```

than arbitrary ordering.

Why?

Because at every important transition, there should be at least one durable path from which the data can be recovered.

The project therefore treats ordering as part of correctness, not just implementation detail.

---

# Development Commands

## Configure

```bash
cmake -S . -B build
```

## Build

```bash
cmake --build build
```

## Run all tests

```bash
ctest --test-dir build --output-on-failure
```

## Rebuild after changes

```bash
cmake --build build
```

If CMake configuration changes:

```bash
cmake -S . -B build
cmake --build build
```

---

# Example Test Output

A successful test run looks like:

```text
100% tests passed out of 11
```

This confirms that the implemented parser, storage, recovery, compaction, concurrency, and crash-handling scenarios are passing in the current test suite.

---

# License

This project is currently intended for educational and portfolio purposes.

---

# Author

**Mohit Methi**

ForgeDB was built as a systems-focused C++ project to explore the internals of database storage engines, persistence, crash recovery, networking, and concurrency.

---

# Final Summary

ForgeDB is more than a basic in-memory key-value store.

It combines:

```text
Client
  +
Network Server
  +
Command Parser
  +
Thread-Safe Database Layer
  +
MemTable
  +
Write-Ahead Log
  +
CRC32 Validation
  +
Crash Recovery
  +
Immutable SSTables
  +
MANIFEST Metadata
  +
Automatic Flushing
  +
Compaction
  +
Fault Injection
  +
Unit and Integration Tests
  +
Crash Tests
```

into one database project.

The result is a practical learning implementation of the core ideas behind persistent LSM-style storage engines and crash-consistent database systems.
