# Riak LevelDB

![LevelDB Status](https://github.com/OpenRiak/leveldb/actions/workflows/build.yml/badge.svg?branch=openriak-3.4)

The original Google README, AUTHORS, LICENSE, NEWS, and TODO files have
been renamed with the extension `.google`.

## Introduction

This repository contains the Google source code as modified to benefit
the Riak environment.  The typical Riak environment has two attributes
that necessitate `leveldb` adjustments, both in options and code:

### Production Servers

Riak often runs in heavy Internet environments on servers with many CPU
cores, lots of memory, and 24x7 disk activity.
Riak LevelDB takes advantage of the environment by adding hardware CRC
calculation, increasing Bloom filter accuracy, and defaulting to
integrity checking enabled.

### Multiple Databases:

Riak often opens up to 128 databases simultaneously.
Google's leveldb supports this, but its background compaction thread can
fall behind.
LevelDB will "stall" new user writes whenever the compaction thread gets
too far behind.
Basho's modifications include multiple thread blocks that each contain
prioritized threads for specific compaction activities.

## Wiki

Details of Riak-specific customizations are found in the
[wiki](https://github.com/OpenRiak/leveldb/wiki).

## Basic Options Needed

Those wishing to truly savor the benefits of Basho's modifications
need to initialize a new `leveldb::Options` structure similar to the
following before each call to `leveldb::DB::Open`:

```cplusplus
    leveldb::Options * options;

    options=new Leveldb::Options;

    options.filter_policy=leveldb::NewBloomFilterPolicy2(16);
    options.write_buffer_size=62914560;     // 60MB
    options.total_leveldb_mem=2684354560;   // 2.5GB (details below)
    options.env=leveldb::Env::Default();
```

## Memory Plan

Riak LevelDB dramatically departed from Google's original internal memory
allotment plan with Riak 2.0, using a methodology called flexcache.
The technical details are
[here](https://github.com/OpenRiak/leveldb/wiki/mv-flexcache)

The key points are:

* `options.total_leveldb_mem` is an allocation for the entire process,
  not a single database

* Giving different values to `options.total_leveldb_mem` on subsequent `Open`
  calls causes memory to rearrange to current value across all databases

* Recommended minimum for Riak LevelDB is 340MB per database.  

* Performance improves rapidly from 340MB to 2.5GB per database (3.0GB if
  using Riak's Active Anti-Entropy).
  Even more is nice, but not as helpful.

* **Never** assign more than 75% of available RAM to `total_leveldb_mem`.
  There is too much unaccounted memory overhead (worse if you use `tcmalloc`
  library).

* `options.max_open_files` and `options.block_cache` should not be used.

## To Do

Since the [leveldb_ee](https://github.com/OpenRiak/leveldb_ee) code is now
open-source, it should be incorporated into this repository and the build
(here and in [eleveldb](https://github.com/OpenRiak/eleveldb)) adjusted
accordingly.
