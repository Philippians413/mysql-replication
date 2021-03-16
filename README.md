### Preparations

Start test environment: `docker-compose up -d`

Install python requirements: `pip install -r requirements.txt`

Run master configs: `python3 master.py`

Run generator: `python3 generator.py`, that will insert values into our table every 5 seconds.

```
$ mysql -h 127.0.0.1 -P 3306 -u root -p

mysql> SELECT count(*) FROM number;
+----------+
| count(*) |
+----------+
|       11 |
+----------+
1 row in set (0.00 sec)

mysql> FLUSH TABLES WITH READ LOCK;
Query OK, 0 rows affected (0.01 sec)

mysql> SHOW MASTER STATUS;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000003 |     4415 | mydb         |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)

# mysqldump -u root -p mydb > mydb.sql
Enter password: 

$ docker cp master:mydb.sql .

On master: UNLOCK TABLES;
```

### Configure slave1
```
docker cp mydb.sql slave1:/

mysql> CREATE DATABASE mydb;

# mysql -u root -p mydb < mydb.sql
Enter password: 

mysql> use mydb
mysql> SELECT count(*) FROM number;
+----------+
| count(*) |
+----------+
|       11 |
+----------+
1 row in set (0.00 sec)

mysql> CHANGE MASTER TO MASTER_HOST='10.0.0.10',MASTER_USER='slave',MASTER_PASSWORD='password',MASTER_LOG_FILE = 'mysql-bin.000003',MASTER_LOG_POS=4415,GET_MASTER_PUBLIC_KEY=1;
Query OK, 0 rows affected, 8 warnings (0.09 sec)

mysql> START SLAVE;
Query OK, 0 rows affected, 1 warning (0.01 sec)

mysql> SELECT count(*) FROM number;
+----------+
| count(*) |
+----------+
|      213 |
+----------+
1 row in set (0.01 sec)

```

### Configure slave2
All steps the same as for slave1. Replica start successfully.
```
mysql> SELECT count(*) FROM number;
+----------+
| count(*) |
+----------+
|      266 |
+----------+
1 row in set (0.00 sec)
```

### Turn off slave1
```
$ docker stop slave1
slave1
```
Check on master:
```
mysql> select count(*) from number;
+----------+
| count(*) |
+----------+
|      349 |
+----------+
1 row in set (0.00 sec)
```
Check on slave2:
```
mysql> SELECT count(*) FROM number;
+----------+
| count(*) |
+----------+
|      349 |
+----------+
1 row in set (0.00 sec)
```
As we can see nothing changed in our chain.

### Try to remove column on slave2 replica
```
mysql> ALTER TABLE number DROP COLUMN number;
Query OK, 0 rows affected (0.28 sec)
Records: 0  Duplicates: 0  Warnings: 0

```
Our `number` column was removed, but `id` column still exist and get new data from master.
