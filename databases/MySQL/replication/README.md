# How to set up MySQL master slave replication
This howto will demonstate how you can set up MySQL master slave replication

# MySQL master
#### If needed create a LVM logical volume where you can store the MySQL-dumps on the MySQL master server (Master)

```
sudo lvcreate -L 180G -n MySQL-dumps vg
sudo mkfs.ext4 /dev/mapper/vg-mysqldumps
sudo mkdir /mnt/mysqldumps
sudo mount -t ext4 /dev/mapper/vg-mysqldumps /mnt/mysqldumps
```

#### Create a MySQL replication user on the MySQL master server which the slave will use when connecting to the master (Master)
```
sudo mysql -H
CREATE USER 'repl'@'172.21.10.155' IDENTIFIED BY '***PASSWORD***';
GRANT REPLICATION CLIENT, REPLICATION SLAVE ON *.* TO 'repl'@'172.21.10.155';
FLUSH PRIVILEGES;
quit
```

#### Make the MySQL master server read only and stop the binary log position with "flush tables with read lock" (Master)
!!! IMPORTANT !!! You need to do this in a screen which you will detach during the dump. Do not type anything else than this otherwise the tables will unlock automatically.
```
screen
sudo -H mysql
root@mysqlmaster.hostname.com> FLUSH TABLES WITH READ LOCK;
root@mysqlmaster.hostname.com> SHOW MASTER STATUS;
+------------------+----------+--------------+------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB |
+------------------+----------+--------------+------------------+
| mysql-bin.004490 | 96629243 |              |                  |
+------------------+----------+--------------+------------------+
1 row in set (0.00 sec)

COMMENT: Store the information above in a text file so you can use it laster on the slave when starting the replication.
COMMENT: Press ctrl a+d to detach the screen !!! Important !!!
```

#### Dump the database(s) to disk (Master)
Now when we have made the MySQL master read-only and stopped the binary logging position we can safely dump the database(s) we want to replicate in a new screen session.
```
cd /mnt/mysqldumps
screen
time sudo -H mysqldump --single-transaction MyDatabase | gzip > mydatabase-$(date '+%Y%m%d').sql.gz

# COMMENT: Press ctrl a+d to detach the screen. Use screen -r pid to see the progress of the dump
```

#### When mysqldump is completed it is time to set MySQL master server to non read-only to get back to normal database operation mode. (master)
```
sudo -H mysql
UNLOCK TABLES;
```
# MySQL slave
Now it is time to configure the MySQL slave server. To be able to do this you first need to upload the mysqldump file(s) to the slave server. Or alternatively read in the dump on the slave server over the network.

#### Create a empty database on the MySQL slave (slave)
```
sudo -H mysql
root@mysqlslave.hostname.com> create MyDatabase;
root@mysqlslave.hostname.com> quit;
```

#### Load in the database dump from the MySQL master on the MySQL slave (Slave)
```
screen
gunzip mydatabase-20171121.sql.gz
time mysql -u root -p MyDatabase < mydatabase-20171121.sql

COMMENT: ctrl a+d to detach screen. Use screen -r pid to see the progress
```

#### Now it is time to start the replication from the slave (Slave)
```
root@mysqlslave.hostname.com> CHANGE MASTER TO
                              MASTER_HOST='172.100.100.10',
                              MASTER_USER='repl',
                              MASTER_PASSWORD='***PASSWORD***',
                              MASTER_LOG_FILE='mysql-bin.004490',
                              MASTER_LOG_POS=96629243;
```

#### Now we can start the replication from the slave server (Slave)
```
root@mysqlslave.hostname.com> START SLAVE;
```

#### Verify that replication works as expected on the slave (Slave)
```
 root@mysqlslave.hostname.com> SHOW SLAVE STATUS\G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event <--------- Yes we are connected
                  Master_Host: 172.100.100.10
                  Master_User: repl
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.004513
          Read_Master_Log_Pos: 88787183
               Relay_Log_File: relaylog.000215
                Relay_Log_Pos: 772031
        Relay_Master_Log_File: mysql-bin.004513
             Slave_IO_Running: Yes <--------- Slave is running
            Slave_SQL_Running: Yes <--------- Slave is running
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table: mysql.plugin,mysql.rds_monitor,mysql.rds_sysinfo,mysql.rds_replication_status,mysql.rds_history,innodb_memcache.config_options,innodb_memcache.cache_policies
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 88787183
              Relay_Log_Space: 4391527
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File:
           Master_SSL_CA_Path:
              Master_SSL_Cert:
            Master_SSL_Cipher:
               Master_SSL_Key:
        Seconds_Behind_Master: 0 <----------- Great we are in the same state as the master
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error:
               Last_SQL_Errno: 0
               Last_SQL_Error:
  Replicate_Ignore_Server_Ids:
             Master_Server_Id: 4000
                  Master_UUID:
             Master_Info_File: mysql.slave_master_info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates <--------- Perfect
           Master_Retry_Count: 86400
                  Master_Bind:
      Last_IO_Error_Timestamp:
     Last_SQL_Error_Timestamp:
               Master_SSL_Crl:
           Master_SSL_Crlpath:
           Retrieved_Gtid_Set:
            Executed_Gtid_Set:
                Auto_Position: 0
         Replicate_Rewrite_DB:
                 Channel_Name:
           Master_TLS_Version:
1 row in set (0.00 sec)
```

#### It is not required but recommended to set the MySQL slave in read only-mode so the data not get corrupted on the slave (Slave)
You can do this in two ways. Either by editing the my.cnf file on the slave server or running the sql-queries below . Setting the slave server to read only will NOT affect the replication only protect the databases from accidental writes from any users.
```
root@mysqlslave.hostname.com> SET GLOBAL read_only = 1;
Query OK, 0 rows affected (0.00 sec)

root@mysqlslave.hostname.com>SET GLOBAL super_read_only = 1;
Query OK, 0 rows affected (0.00 sec)
```

#### Extra if the slave is an AWS RDS instance you need to execute the following on the slave (Slave)
```
CALL mysql.rds_set_external_master ('172.100.100.10', 3306,
    'repl', '***PASSWORD***', 'mysql-bin.004490', 96629243, 0);

CALL mysql.rds_start_replication;
```
