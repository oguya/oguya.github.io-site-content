---
date: '2016-01-07'
tags:
- mysql
- linux
title: "Enabling MySQL Binary Logs"
---

MySQL binary log contains records of all changes to a databases—both data & structure—as well as how long each statement took to execute. It logs SQL statements such as CREATE, ALTER, INSERT, UPDATE & DELETE  with the exception of SELECT & SHOW which have no effect on the data.
<!--more-->

The purpose of binary log is to allow replication whereby data is sent from a one server(master) to another one(slave) as well assisting in certain data recovery operations. Binary logs are stored in binary format, therefore, you have to use [mysqlbinlog utility](https://dev.mysql.com/doc/refman/5.5/en/mysqlbinlog.html) to view its contents.

In most MySQL setups, binary logging is disabled by default, thus you'll end up with the following error:
```
MariaDB [(none)]> show binary logs;
ERROR 1381 (HY000): You are not using binary logging
MariaDB [(none)]> show variables like 'log_bin';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| log_bin       | OFF   |
+---------------+-------+
1 row in set (0.00 sec)
```

To fix this, you have to enable binary logs by adding `log_bin` parameter under `[mysqld]` section in your `my.cnf` config. file:
```
log_bin = mysql-bin
```

## Further Reading

1. [MySQL Binary Log](https://dev.mysql.com/doc/refman/5.5/en/binary-log.html)
