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

Records will be stored sequentially on a disk file with the following format:
keylen | vallen | crc32 | flag | key | value | magic

- [4 bytes] keylen: size of the key in bytes
- [4 bytes] vallen: size of value in bytes
- [4 bytes] crc32: hash of the flag, key, and value
- [4 bytes] flag: special attributes about the record
- [keylen bytes] key: string of key in bytes
- [vallen bytes] value: string of the value in bytes
- [4 bytes] magic: a special number indicating end of a record

Detecting truncated lines is done as follows:
- The line length should be keylen + vallen + 20 bytes
- The crc32 value should be equal to the crc(flag, key, value, magic)
- The magic value should be present


### Write path (PUT)

Write updates the in-memory index and the database file. The index tracks keys
and their file offsets, whereas the database file has the actual value. To
create a record, we compute the elements of the format above and serialize them
as a stream of bytes. Once written to disk, the offset is written-back to the
in-memory index for the next read.

We have 3 possible synchronization strategies:
- flush to disk on each write; for simplicity this is our default scheme
- rely on the OS synchronization
- periodic background flushing

### Read path (GET)

The in-memory index stores the key and its respective offset within the
database file. Initialized at startup, the in-memory index tracks all items
present in the database. To read a given key:
- Lookup the key within the index, return 404 if not found
- Use the offset to read the value in the database file


### DELETE and tombstones

Deleting items consist in inserting a special `tombstone` entry indicating the
item has been deleted. Essentially, a tombstone record has the particular bit
flag set, and the `vallen` is set to 0. After the item is serialized to disk,
we can remove the entry from the in-memory index.

The approach above indicates the file keeps growing on delete, and would require
compaction to reclaim disk space.

### Startup / crash recovery

Startup creates the in-memory index by reading the entries from the database
file. The file is read top to bottom, each record is parsed and inserted into
the in-memory index:
- When an existing entry is found, we overwrite its stored offset.
- On a tombstone, we remove the entry from the in-memory index.
- When incomplete, we stop scanning as this indicates an integrity violation

### Torn writes

Every entry has a CRC value, which is used to check the truncation or
modification of the record line. In addition, we have a magic number at the end
of each record to track when a record is fully written.

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
