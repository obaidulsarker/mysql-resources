# MySQL Server Tuning

## MySQL 8.0

To let MySQL automatically configure InnoDB variables, set the innodb_dedicated_server = ON parameter. The following variables will be configured:
```
innodb_buffer_pool_size
innodb_log_file_size
innodb_log_files_in_group
innodb_flush_method
```
- Determines the combined size of all logs. If the available RAM is <2GB, set the value to 2, if it is >2GB, set the value to 4.
  ```
  innodb_log_files_in_group = 4 
  innodb_buffer_pool_instances = 8 # (or 1 if innodb_buffer_pool_size < 1GB)
  ```
- Use the following formula to calculate the value: 0.75*RAM. If the available RAM is 128GB, then 128*0.75 = 96.
  ```
  innodb_buffer_pool_size = 96G 
  innodb_log_buffer_size = 256M
  ```
  
- Using 2x the quantity of cores is recommended.
 ```
  innodb_thread_concurrency = 64 
  innodb_file_per_table = ON
  innodb_stats_on_metadata = OFF
  ```

- Commented out by default. Determines the method to flush data to InnoDB data files and logs. Using it can affect I/O throughput. (Recommended values for codebeamer, if needed: O_DIRECT: local/DAS, O_DSYNC: SAN/iSCSI)
  ```
  innodb_flush_method = O_DIRECT 
  ```

- The values below should be adjusted depending on the available physical memory (RAM) of the MySQL server (the values here are a general examples, it is advised to use the values automatically configured as a starting point):
```
# RAM: 2-8GB
innodb_log_file_size = 128M
# RAM: 8-24GB
innodb_log_file_size = 256M
# RAM: 24-64GB
innodb_log_file_size = 512M
# RAM: 64-128GB
innodb_log_file_size = 768M
# RAM: 128GM+
innodb_log_file_size = 1024M
```

### Example Configuration:
#### Server Size: 32 Core, 64 GB Memory, SSD Disk
```vi /etc/my.cnf```

```
[mysqld]
# General Settings
port                           = 3306
datadir                        = /var/lib/mysql
socket                         = /var/run/mysqld/mysqld.sock
pid_file                       = /var/run/mysqld/mysqld.pid
character_set_server           = utf8mb4
collation_server               = utf8mb4_unicode_ci
skip_name_resolve              = 1 # Avoids DNS lookups for client connections, faster but requires IP-based grants.

# Error Logging
log_error                      = /var/log/mysql/error.log
log_timestamps                 = SYSTEM # Or UTC for consistency

# Slow Query Log (Essential for optimization)
slow_query_log                 = 1
slow_query_log_file            = /var/log/mysql/mysql-slow.log
long_query_time                = 1     # Log queries taking longer than 1 second
log_queries_not_using_indexes  = 1     # Log queries that don't use indexes

# Binary Logging (for replication and point-in-time recovery)
log_bin                        = /var/lib/mysql/mysql-bin
binlog_format                  = ROW   # Recommended for safety and replication consistency
expire_logs_days               = 7     # Retain binary logs for 7 days (adjust based on backup strategy)
# For MySQL 8.0+, use binlog_expire_logs_seconds instead:
# binlog_expire_logs_seconds   = 604800 # 7 days in seconds

# Durability (Trade-off between durability and performance)
sync_binlog                    = 1     # 1 = highest durability (syncs binlog on every commit)
                                       # 0 = OS handles sync (fastest, but risk of data loss on crash)
                                       # N = syncs after N transactions (compromise)
innodb_flush_log_at_trx_commit = 1     # 1 = highest durability (syncs InnoDB log on every commit)
                                       # 0 = fastest (flushes every second), 2 = compromise

# InnoDB Settings (CRITICAL for performance)
innodb_buffer_pool_size        = 50G   # ~70-80% of 64GB RAM. Adjust based on other services on the server.
innodb_buffer_pool_instances   = 8     # For 64GB, 8 instances is a good start (1 per 8GB)
innodb_log_file_size           = 1G    # Larger files reduce checkpoint frequency. (e.g., 256M to 4G)
innodb_log_files_in_group      = 2     # Typically 2. Total size = innodb_log_file_size * innodb_log_files_in_group
innodb_file_per_table          = 1     # Recommended: each table has its own .ibd file
innodb_flush_method            = O_DIRECT # Recommended for Linux to avoid double buffering
innodb_io_capacity             = 2000  # Adjust based on your disk's IOPS (e.g., 2000 for good SSD, 200 for HDD)
innodb_io_capacity_max         = 4000  # Max burst IOPS
innodb_thread_concurrency      = 0     # 0 = let InnoDB manage (recommended for modern CPUs)
                                       # Or set to 2 * number of cores (e.g., 64 for 32 cores) if you see contention
innodb_read_io_threads         = 8     # Number of I/O threads for reads
innodb_write_io_threads        = 8     # Number of I/O threads for writes
innodb_lru_scan_depth          = 2048  # Adjust based on workload; default 1024, higher for more active systems

# Threading & Connections
max_connections                = 500   # Adjust based on application needs. Too high can cause issues.
thread_cache_size              = 100   # Cache for reusable threads
back_log                       = 1500  # Number of pending connections before MySQL stops accepting new ones

# Per-Thread Buffers (Allocated per connection, so keep these reasonable)
sort_buffer_size               = 2M    # For sorting operations
join_buffer_size               = 2M    # For join operations that can't use indexes
read_buffer_size               = 1M    # For sequential scans
read_rnd_buffer_size           = 1M    # For random reads (e.g., after sorting)

# Temporary Tables
tmp_table_size                 = 256M  # Max size for in-memory temporary tables
max_heap_table_size            = 256M  # Max size for user-created MEMORY tables
                                       # Ensure tmp_table_size <= max_heap_table_size

# Query Cache (DISABLE FOR MYSQL 5.7+ AND 8.0)
# The query cache is a known bottleneck for high concurrency and is removed in MySQL 8.0.
query_cache_type               = 0
query_cache_size               = 0

# Logging (General Log - for debugging, disable in production)
# general_log                  = 1
# general_log_file             = /var/log/mysql/mysql.log

# Authentication (MySQL 8.0 specific)
# default_authentication_plugin = mysql_native_password # If you need older clients to connect

# Other useful settings
skip_external_locking          = 1 # Prevents external locking, usually safe
# default_storage_engine       = InnoDB # Explicitly set default engine

[client]
port                           = 3306
socket                         = /var/run/mysqld/mysqld.sock

[mysqldump]
quick
max_allowed_packet             = 16M # For large dumps

[mysql]
no-auto-rehash

[isamchk]
key_buffer                     = 16M

# For Percona Toolkit
# [pt-query-digest]
# database=your_database
# user=your_user
# password=your_password
```
