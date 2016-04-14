---
date: '2016-04-13'
tags:
- mysql
- mariadb
- linux
title: "Safely Rotating MySQL Slow Query Logs"
---

MySQL slow query log consists of SQL statements that took more than [long_query_time](https://dev.mysql.com/doc/refman/5.6/en/server-system-variables.html#sysvar_long_query_time) seconds to complete execution & required atleast [min_examined_row_limit](https://dev.mysql.com/doc/refman/5.6/en/server-system-variables.html#sysvar_min_examined_row_limit) to be examined. By default, administrative queries & those that don't use indexes for lookups are not logged.
<!--more-->

Two common techniques used by [Logrotate](http://linux.die.net/man/8/logrotate) are:

- **copytruncate**: Instead of moving the old log file & optionally creating a new one, logrotate truncates the original log file in place after creating a copy.
- **nocopytruncate**: Do not truncate the original log file in place after creating a copy.

Truncating log files can block MySQL because the OS serializes access to the inode during the truncate operation. Therefore, it is recommended to temporarily stop slow query logging, flush slow logs, rename 
the old log file & finally enable slow query logging.

Flushing logs might take a considerable amount of time, therefore, to avoid filling slow log buffer, it's advisable to temporarily disable MySQL slow query logging & re-enabling it once the rotation is complete.


## Manual Rotation

To manually rotate slow query logs, we'll temporarily disable slow query logging, flush slow logs, rename the original file & finally re-enable slow query logging.

- get the path to slow query log file

        MariaDB [(none)]> show variables like '%slow_query%';
        +---------------------+-------------------------------+
        | Variable_name       | Value                         |
        +---------------------+-------------------------------+
        | slow_query_log      | ON                            |
        | slow_query_log_file | /var/lib/mysql/mysql-slow.log |
        +---------------------+-------------------------------+
        2 rows in set (0.00 sec)

- temporarily disable slow query logging

        MariaDB [(none)]> set global slow_query_log=off;
        Query OK, 0 rows affected (0.00 sec)

- flush only slow logs

        MariaDB [(none)]> flush slow logs;
        Query OK, 0 rows affected (0.00 sec)

- rename the old log file and or compress it

        root@db01:~# mv /var/lib/mysql/mysql-slow.log /var/lib/mysql/mysql-slow-$(date +%Y-%m-%d).log
        root@db01:~# gzip -c /var/lib/mysql/mysql-slow-$(date +%Y-%m-%d).log > /var/lib/mysql/mysql-slow-$(date +%Y-%m-%d).log.gz

- finally, re-enable slow query logging

        MariaDB [(none)]> set global slow_query_log=on;
        Query OK, 0 rows affected (0.00 sec)


## Using Logrotate

Instead of manual rotation, you can use a lograte config file to acheive the same effect by using logrotate: `/etc/logrotate.d/mysql-slow-logs`
```
/var/lib/mysql/mysql-slow.log {
    size 1G
    dateext
    compress
    missingok
    rotate 52
    notifempty
    delaycompress
    sharedscripts
    nocopytruncate
    create 660 mysql mysql
    postrotate
        /usr/bin/mysql -e 'select @@global.slow_query_log into @sq_log_save; set global slow_query_log=off; select sleep(5); FLUSH SLOW LOGS; select sleep(10); set global slow_query_log=@sq_log_save;'
    endscript
    rotate 150
}
```

### Further Reading

1. [MySQL Slow Query Log](https://dev.mysql.com/doc/refman/5.5/en/slow-query-log.html)

