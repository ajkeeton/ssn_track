
Blue Green Hash (bgh) and Dirt Simple Hash (dsh) are two solutions for 
TCP/IP session tracking. Each allows arbitrary data to be associated with
a session. To account for missed teardowns, session data is automatically
timed out. 

# BGH

BGH derives its implementation from connection draining blue-green deployments.
It uses two threads, one of which controls a periodic refresh. During a 
refresh:

    - A new hash is allocated and is optionally sized to meet past resource 
      requirements
    - When a lookup is performed and the data is found in the old table, it is
      automatically transitioned to the new table
    - All inserts go into the new table
    - After the timeout period, the old hash is destroyed. Any sessions 
      remaining in this hash are removed

Since hash reallocation and cleanup are performed in their own thread, the 
performance impact of timing out old sessions is minimal.

# DSH

DSH maintains an LRU of sessions that is updated for each lookup and insert. 
Old sessions are timed out once the timeout limit is hit, unlike BGH where 
timeouts happen after the refresh period starts and the timeout is reached.

DSH does not automatically resize.

Used by https://github.com/ajkeeton/pack_stat for TCP session stats

# Building

    mkdir build ; cd build ; cmake .. ; make
    
# Basic usage (prefix dsh for Dirt Simple Hash):

    bgh_new(...)
    bgh_insert(...)
    bgh_lookup(...)
    bgh_clear(...) - optional, as sessions are timed out automatically
    bgh_free(...)
 
Hashes must be provided with a callback to free data:

    void free_cb(void *data_to_free) { ... }

# Sample

    ./sample/pcap_stats <pcap>

# Configuring BGH

To use with defaults (see bgh.h), just provide bgh_new with a callback to free
the data you insert. This can not be null.

    bgh_new(free_cb)

For more control over BGH's behavior, pass in a bgh_config_t to bgh_config_new. 

Initialize a config to the defaults:

    bgh_config_t config;
    bgh_init_config(&config);

Configuration options are as follows:

    // Seconds between refresh periods. 0 to disable refreshes (and thereby
    //  timeouts)
    config.refresh_period = 120; 
    // Seconds to wait for active sessions to transition. Anything left in the
    // old hash after this timeout will be removed
    config.timeout = 30;
    // Initial number of rows. Should be prime
    config.initial_rows = 100003;
    // Lower bounds to shrink to. If 0, the initial size is used
    // Should be prime
    config.min_rows = 26003;
    // Max number of rows we can grow to
    // Should be prime
    config.max_rows = 15485867;

    bgh_t *tracker = bgh_config_new(&config, free_cb);

# BGH Autoscaling

prime.cc contains a partial list of prime numbers. When scaling up or down, BGH
selects the next prime in the list in the direction of scaling.


Test:

    ./tests/test

And for extra Sanity:

    valgrind --tool=memcheck tests/test

