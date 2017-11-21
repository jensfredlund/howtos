# mysqldump howto

#### Dump database to file
```
time sudo -H mysqldump --single-transaction mydatabase | gzip > mydatabase-$(date '+%Y%m%d').sql.gz
```

#### Dump single table of database over MySQL socket
```
sudo -H mysqldump -S /var/run/mysql/mysqlsnap.sock mydatabase tablename | gzip > tablename-$(date '+%Y%m%d').sql.gz
```

#### Dump database with data but ignore these tables
```
sudo -H mysqldump -S /var/run/mysql/mysqlsnap.sock mydatabase --ignore-table=databasename.tablename1 \
                                                              --ignore-table=databasename.tablename2 \
                                                              --ignore-table=databasename.tablename3 | gzip > mydatabase-$(date '+%Y%m%d').sql.gz
```
#### Dump database structure (schema) without any data
```
sudo -H mysqldump -S /var/run/mysql/mysqlsnap.sock --no-data mydatabase > mydatabase.sql
```
