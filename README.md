# LogKV

An append-only log-structured key-value store, built from scratch in Go.
In-memory hash index, single data file, Bitcask-style. Built as a learning
project: every design decision is documented below, including the wrong ones.

## What it does

- String keys, JSON values, entire dataset fits in memory
- `GET` / `PUT` / `DELETE` over HTTP
- Persistence and crash recovery via an append-only log

## Design

### Record format on disk

Each record would be represented by each line within the file. The record would
be composed of the following fields: 

keylen | vallen | crc32 | flag | key | value

- [4 bytes] keylen: size of the key in bytes
- [4 bytes] vallen: size of value in bytes
- [4 bytes] crc32: non cryptographic hash of the record, including flag, key, and val
- [4 bytes] flag: special attributes about the record
- [keylen bytes] key: string of key in bytes
- [vallen bytes] value: string of the value in bytes

Detecting truncated lines is done as follows:
- The line must have at least 12 bytes containing the keylen, vallen, and crc32
- The sum of keylen and vallen should be at least the length of the line
- The crc32 value should match the computed crc32 of the key and value


### Write path (PUT)

First, we append a new record in the format described above on a newline. We
escape newline characters in the key and value by converting them to base64.
The offset of the write is then written back to the in-memory index. Initially,
we would flush the database to disk on each write, but would later compare to
OS-controlled synchronization or periodic background flushing.

### Read path (GET)

The in-memory index stores the key and its respective offset within the
database file. Initialized at startup, the in-memory index is the list of
all items present in the database. Thus, to read a given item:
- Find the item in memory, return 404 or other fail error code on miss
- Use the item offset to read its corresponding value in the database file


### DELETE and tombstones

Deleting items consist in inserting a special `tombstone` entry indicating the
item has been deleted. This would be done by changing a specific bit within the
flags. After checking the item within the in-memory database, we will append
a new record in the file with the tombstone flag on.

Note that compaction would be triggered as a separate process, and the space
would be reclaimed accordingly.

### Startup / crash recovery

Startup creates the in-memory index by reading the entries from the database
file. The file is read top to bottom, each line is a parsed into a record, and
inserted into the in-memory index. When an existing entry is found, the offset
is overriden with the new entry. On a tombstone, we remove the entry from the 
in-memory index.


### Torn writes

Every entry has a CRC value, which is used to check the truncation or
modification of the record line.

## Open questions (stage 2+)

- When to compact, and how, without stopping writes?
    + Compaction is done when the log file is considered "too large"
    + We serialize in-memory entries into a new DB file and update pointer
- When is the log "too large"?
    + The ratio of the #lines in the log to the #entries in the in-memory
      index; Once this gets to 2x, we can assume the log file is too large.
- Should PUT wait for the append before responding? (Currently: yes.)

## Status

- [ ] Stage 1: single-file log, in-memory index, HTTP API, crash recovery
- [ ] Stage 2: compaction
- [ ] Stage 3: TBD

## Non-goals (for now)

Multi-node anything, values larger than memory, concurrent writers.
