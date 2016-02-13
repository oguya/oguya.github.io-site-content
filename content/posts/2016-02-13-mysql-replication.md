---
date: '2016-02-13'
tags:
- mysql
- mariadb
- linux
title: "MySQL Replication: What is it? How does it work? How can I set it up?"
---

## What is it?

MySQL replication allows you to 'mirror' databases on one or more servers(master) to one or more servers(slave). You can control what to replicate such as one or more databases, or even tables within a database i.e. selective replication.
<!--more-->

## How does it work?

Replication relies heavily on [binary logs](https://dev.mysql.com/doc/refman/5.6/en/binary-log.html). If binary logging is enabled, all updates—data manipulation & data definition—to a database are written to binary log as a binary log event. A slave server can then read it's master's binary log to access data for replication. The slave writes master's binary log events in a [relay log](https://dev.mysql.com/doc/refman/5.6/en/slave-logs-relaylog.html) file.

A slave server keeps track of it's master's binary log position that it last applied, therefore allowing it to re-connect and resume replication if it was temporarily stopped. Basically, masters & slaves don't have to be in constant communication with each other.

Replication can come in handy in the following scenarios:

- Taking backups: backups can easily be taken if a server is not being actively used. Its always recommended to take backups on a slave server instead of a production(master) server that is actively under use.
- Scalability: For high-read/low-write environments, you can have one master server where all writes occur and replicated to multiple slaves which in turn handle the reads.
- Load reduction: Instead of running some sort of data analysis which might increase the load on a master server, this can be done on a slave server therefore reducing load on the master.
- Data distribution: replication can be used to create a local copy of data on a remote master.

## How can I set it up?

Having 2 servers—`db01` as the master and `db02` as the slave—both running Ubuntu 14.04, we'll setup replication on [MariaDB](https://mariadb.com/)—a fork of MySQL:

### On both servers

- add mariadb repo & install [mariadb-server-10.0](https://mariadb.com/kb/en/mariadb/what-is-mariadb-100/)

        # aptitude install software-properties-common
        # apt-key adv --recv-keys --keyserver hkp://keyserver.ubuntu.com:80 0xcbcb082a1bb943db
        # echo 'deb http://ftp.wa.co.za/pub/mariadb/repo/10.0/ubuntu trusty main' > /etc/apt/sources.list.d/mariadb.list
        # aptitude update
        # aptitude install mariadb-server-10.0

### On db01—master server

- set mysql to bind to a non-localhost IP address, otherwise remote slave connections will fail:

        bind-address = 192.168.1.5

- Give the master a unique `server_id`. It can be a number from 1 to 2<sup>32</sup>-1 & must be unique in the replication group; `/etc/mysql/conf.d/master-replication.cnf`

        server_id               = 1
        master_verify_checksum  = 1

- [Enable binary logging](/posts/2016-01-07-enable-mysql-binlog/), if not yet enabled.

        log_bin = mysql-bin

- General master's mysql config file: `/etc/mysql/conf.d/master-replication.cnf` Restart mysql service for the config changes to take effect

        [mysqld]
        log_bin                 = mysql-bin
        server_id               = 1
        bind-address            = 192.168.1.5
        master_verify_checksum  = 1

- Create a slave user & grant it replication privilege; remember this user will connect to the master from the slave(`192.168.1.6`)

        create user 'slave'@'192.168.1.6' identified by '6^%ys3a^A7&bpQWmR=*A';
        grant replication slave on *.* to 'slave'@'192.168.1.6';
        flush privileges;

- get the binary log file name & position which the slave will use it as a starting point for the replication

        MariaDB [(none)]> show master status;
        +------------------+----------+--------------+------------------+
        | File             | Position | Binlog_Do_DB | Binlog_Ignore_DB |
        +------------------+----------+--------------+------------------+
        | mysql-bin.001054 |   55117  |              |                  |
        +------------------+----------+--------------+------------------+
        1 row in set (0.00 sec)

- rsync mysql data dir from `db01` to `db02`, preferably when mysql service is stopped on both hosts to avoid inconsistencies. You can also use `mysqldump` if you don't have large databases

        # service mysql stop
        # rsync -aPvz --human-readable /var/lib/mysql root@db02:/var/lib/


### On db02—slave

- Give db02 a unique `server_id` too. To avoid data inconsistencies between the master & slave, verify binlog checksums when reading events from the relay log by setting `slave_sql_verify_checksum` to 1. Restart mysql for config changes to take effect: `/etc/mysql/conf.d/slave-replication.cnf`

        [mysqld]
        server_id                   = 2
        slave_sql_verify_checksum   = 1

- Once data from the master has been copied you can start replication by running [CHANGE MASTER TO](https://dev.mysql.com/doc/refman/5.6/en/change-master-to.html) mysql command. Ensure that `MASTER_LOG_FILE` & `MASTER_LOG_POS` matches the filename & position returned by `SHOW MASTER STATUS` ran earlier on the master:

        CHANGE MASTER TO
          MASTER_HOST='db01',
          MASTER_USER='slave',
          MASTER_PASSWORD='6^%ys3a^A7&bpQWmR=*A',
          MASTER_PORT=3306,
          MASTER_LOG_FILE='mysql-bin.001054',
          MASTER_LOG_POS=55117,
          MASTER_CONNECT_RETRY=60;

- finally, start slave using [START SLAVE](https://dev.mysql.com/doc/refman/5.6/en/start-slave.html) mysql command:

        START SLAVE;

- You can view slave status by running [SHOW SLAVE STATUS](https://dev.mysql.com/doc/refman/5.6/en/show-slave-status.html) mysql command:

        MariaDB [(none)]> show slave status\G
        *************************** 1. row ***************************
                       Slave_IO_State: Waiting for master to send event
                          Master_Host: db01
                          Master_User: slave
                          Master_Port: 3306
                        Connect_Retry: 60
                      Master_Log_File: mysql-bin.001054
                  Read_Master_Log_Pos: 55117
                       Relay_Log_File: mysqld-relay-bin.000274
                        Relay_Log_Pos: 55100
                Relay_Master_Log_File: mysql-bin.001054
                     Slave_IO_Running: Yes
                    Slave_SQL_Running: Yes
                      Replicate_Do_DB: 
                  Replicate_Ignore_DB: 
                   Replicate_Do_Table: 
               Replicate_Ignore_Table: 
              Replicate_Wild_Do_Table: 
          Replicate_Wild_Ignore_Table: 
                           Last_Errno: 0
                           Last_Error: 
                         Skip_Counter: 0
                  Exec_Master_Log_Pos: 55117
                      Relay_Log_Space: 1239660
                      Until_Condition: None
                       Until_Log_File: 
                        Until_Log_Pos: 0
                   Master_SSL_Allowed: No
                   Master_SSL_CA_File: 
                   Master_SSL_CA_Path: 
                      Master_SSL_Cert: 
                    Master_SSL_Cipher: 
                       Master_SSL_Key: 
                Seconds_Behind_Master: 0
        Master_SSL_Verify_Server_Cert: No
                        Last_IO_Errno: 0
                        Last_IO_Error: 
                       Last_SQL_Errno: 0
                       Last_SQL_Error: 
          Replicate_Ignore_Server_Ids: 
                     Master_Server_Id: 1
                       Master_SSL_Crl: 
                   Master_SSL_Crlpath: 
                           Using_Gtid: No
                          Gtid_IO_Pos: 
        1 row in set (0.00 sec)


## Extras

1. Don't mix & match server names & IP addresses: I prefer adding hostnames & their respective IP addresses to `/etc/hosts` which gives me some sort of flexibility in changing IP address & it's more readable. To be on the safe side, either use only hostnames or IP addresses.
2. Replication is not a complete backup solution: Remember that all updates—data manipulation & definition—to a database are written as binlog events to a binary log, which is then replicated to a slave. Therefore, if you drop a database or truncate a table on master, the same will be done on the slave too! Replication, to some extent, can assist in protection against hardware failure on the master, but not against data loss—intentional or unintentional—on the master.


## Further Reading

1. [MySQL Binary Log](https://dev.mysql.com/doc/refman/5.6/en/binary-log.html)
2. [MySQL Relay Log](https://dev.mysql.com/doc/refman/5.6/en/slave-logs-relaylog.html)
3. [MariaDB 10.0 Server](https://mariadb.com/kb/en/mariadb/what-is-mariadb-100/)
4. [MySQL `CHANGE MASTER TO` Command](https://dev.mysql.com/doc/refman/5.6/en/change-master-to.html)
5. [MySQL `SHOW SLAVE STATUS` Command](https://dev.mysql.com/doc/refman/5.6/en/show-slave-status.html)
