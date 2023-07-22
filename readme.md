# Documentation for debugging pixels

## 1. Create user-defined tpch data

In this example, we create tpch0.1 because it is small enough for us to debug pixels. 

### a. The sql command to create mysql metadata database

The metadata database can be created by the file `/scratch/liyu/opt/pixels/scripts/sql/metadata_schema.sql`

Parameter: `pixels_metadata_liyu` is the metadata database name


### b. Check Mysql service is running

```
liyu@diascld31:/usr/bin$ pgrep mysqld
326792
```

### c. Create mysql metadata database

The information of Mysql is [here](https://github.com/pixelsdb/evaluations/blob/master/Pixels.md):
```
user=pixels_liyu
password=@WSXcft6
port=13306
```
The command to log in Mysql is:
```
mysql -u pixels_liyu -p < /scratch/liyu/opt/pixels/scripts/sql/metadata_schema.sql
```
And then type the password `@WSXcft6`.


The metadata is stored here. 
```
mysql> show databases;
+----------------------+
| Database             |
+----------------------+
| information_schema   |
| performance_schema   |
| pixels_metadata_liyu |
+----------------------+
3 rows in set (0.00 sec)

mysql> use pixels_metadata_liyu;

mysql> show tables;
+--------------------------------+
| Tables_in_pixels_metadata_liyu |
+--------------------------------+
| COLS                           |
| DBS                            |
| LAYOUTS                        |
| TBLS                           |
| VIEWS                          |
+--------------------------------+
5 rows in set (0.00 sec)
```

### d. generate tpch data
In server 31, we use [dbgen](https://github.com/electrum/tpch-dbgen) to generate tpch data. 

```
cd /scratch/liyu/opt/tpch-dbgen
make
dbgen -vf -s 0.1
mkdir -p /scratch/liyu/opt/data/tpch-0_1g
cp *.tbl /scratch/liyu/opt/data/tpch-0_1g/
```

The expected output files are:

```
liyu@diascld31:/scratch/liyu/opt/data/tpch-0_1g$ ls
customer.tbl  lineitem.tbl  nation.tbl  orders.tbl  partsupp.tbl  part.tbl  region.tbl  supplier.tbl
```

Based on the instruction [here](https://github.com/pixelsdb/pixels), **The file(s) of each table are stored in a separate directory named by the table name.** Thus, we organize the original data as the following way:
```
mkdir customer
mv customer.tbl customer/
mkdir lineitem
mv lineitem.tbl lineitem/
mkdir nation
mv nation.tbl nation/
mkdir orders
mv orders.tbl orders/
mkdir partsupp
mv partsupp.tbl partsupp/
mkdir part
mv part.tbl part/
mkdir region
mv region.tbl region/
mkdir supplier
mv supplier.tbl supplier/
```
The expected data organization is:
```
liyu@diascld31:/scratch/liyu/opt/data/tpch-0_1g$ ls
customer  lineitem  nation  orders  part  partsupp  region  supplier
```

### e. Start etcd
run the command:
```
tmux a -t etcd
./start-etcd.sh
```

## f. Start pixels
Copy the mysql-connector to lib/ and then 
Run the command
```
liyu@diascld31:/scratch/liyu/opt/pixels$ ./sbin/start-pixels.sh
```

## g. Start trino
The trino repo is [here](https://trino.io/docs/405/installation/deployment.html).

Edit the file `trino-server/etc/config.properties`:
```
coordinator=true
node-scheduler.include-coordinator=true
http-server.http.port=7080
query.max-memory=20GB
query.max-memory-per-node=20GB
discovery.uri=http://diascld31.iccluster.epfl.ch:7080
task.max-worker-threads=48
#task.min-drivers=1
#task.concurrency=1
```

`trino-server/etc/jvm.config`:

```
-server
-Xmx32G
-XX:+UseNUMA
-XX:InitialRAMPercentage=80
-XX:MaxRAMPercentage=80
-XX:G1HeapRegionSize=32M
-XX:+ExplicitGCInvokesConcurrent
-XX:+ExitOnOutOfMemoryError
-XX:+HeapDumpOnOutOfMemoryError
-XX:-OmitStackTraceInFastThrow
-XX:ReservedCodeCacheSize=512M
-XX:PerMethodRecompilationCutoff=10000
-XX:PerBytecodeRecompilationCutoff=10000
-Djdk.attach.allowAttachSelf=true
-Djdk.nio.maxCachedBufferSize=2000000
-XX:+UnlockDiagnosticVMOptions
-XX:+UseAESCTRIntrinsics
# Disable Preventive GC for performance reasons (JDK-8293861)
-XX:-G1UsePreventiveGC
-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:7005
```

And then change `etc/catalog/pixels.properties`:
```
connector.name=pixels
pixels.config=/scratch/liyu/opt/pixels/pixels.properties

# serverless config
# lambda.switch can be on, off, auto
lambda.switch=off
local.scan.concurrency=40
clean.local.result=true
```
And run the command 
```
tmux a -t trino
./bin/launcher start
```

## h. Create table in trino

The command to create the table is in the file `/scratch/liyu/opt/pixels/scripts/sql/tpch_0_1_schema.sql`

Notice: the parameter `storage` in this file should be `s3`, `hdfs` or `file`. 

```
./bin/trino --server localhost:7080 --catalog pixels < /scratch/liyu/opt/pixels/scripts/sql/tpch_0_1_schema.sql
```

The expected output:
```
liyu@diascld31:/scratch/liyu/opt/trino-server$ ./bin/trino --server localhost:7080 --catalog pixels
trino> show schemas;
       Schema
--------------------
 information_schema
 tpch
(2 rows)

Query 20230224_134826_00121_n4ycf, FINISHED, 1 node
Splits: 35 total, 35 done (100.00%)
0.06 [2 rows, 32B] [33 rows/s, 542B/s]

trino> use tpch;
USE
trino:tpch> show tables;
  Table
----------
 customer
 lineitem
 nation
 orders
 part
 partsupp
 region
 supplier
(8 rows)

Query 20230224_134833_00125_n4ycf, FINISHED, 1 node
Splits: 35 total, 35 done (100.00%)
0.18 [8 rows, 166B] [43 rows/s, 907B/s]
```

### i. write the LAYOUTS in mysql
The command to write the LAYOUTS in mysql is in the file `/scratch/liyu/opt/pixels/scripts/sql/tpch_0_1_layouts.sql`. 

Notice: the parameter `file:///scratch/liyu/opt/pixels_file/pixels-tpch-0_1/customer/v-0-order/` is the pixel data in the local file system. 

```
mysql -u pixels_liyu -p < /scratch/liyu/opt/pixels/scripts/sql/tpch_0_1_layouts.sql
```
The password: `@WSXcft6`

Create the pixel data directory in the local file system:
```
mkdir -p /scratch/liyu/opt/pixels_file/pixels-tpch-0_1/lineitem/v-0-order/
mkdir -p /scratch/liyu/opt/pixels_file/pixels-tpch-0_1/lineitem/v-0-compact/

mkdir -p /scratch/liyu/opt/pixels_file/pixels-tpch-0_1/customer/v-0-order/
mkdir -p /scratch/liyu/opt/pixels_file/pixels-tpch-0_1/customer/v-0-compact/

mkdir -p /scratch/liyu/opt/pixels_file/pixels-tpch-0_1/nation/v-0-order/
mkdir -p /scratch/liyu/opt/pixels_file/pixels-tpch-0_1/nation/v-0-compact/

mkdir -p /scratch/liyu/opt/pixels_file/pixels-tpch-0_1/orders/v-0-order/
mkdir -p /scratch/liyu/opt/pixels_file/pixels-tpch-0_1/orders/v-0-compact/

mkdir -p /scratch/liyu/opt/pixels_file/pixels-tpch-0_1/part/v-0-order/
mkdir -p /scratch/liyu/opt/pixels_file/pixels-tpch-0_1/part/v-0-compact/

mkdir -p /scratch/liyu/opt/pixels_file/pixels-tpch-0_1/partsupp/v-0-order/
mkdir -p /scratch/liyu/opt/pixels_file/pixels-tpch-0_1/partsupp/v-0-compact/

mkdir -p /scratch/liyu/opt/pixels_file/pixels-tpch-0_1/region/v-0-order/
mkdir -p /scratch/liyu/opt/pixels_file/pixels-tpch-0_1/region/v-0-compact/

mkdir -p /scratch/liyu/opt/pixels_file/pixels-tpch-0_1/supplier/v-0-order/
mkdir -p /scratch/liyu/opt/pixels_file/pixels-tpch-0_1/supplier/v-0-compact/
```

### j. Run pixels-cli
Under `PIXELS` directory, run 
```
cd /scratch/liyu/opt/pixels
java -jar ./sbin/pixels-cli-*-full.jar
```

And then type the following command

Enable encoding:
```
LOAD -o file:///data/tpch/100g/customer -s tpch -t customer -n 319150 -r \| -c 1
LOAD -o file:///data/tpch/100g/lineitem -s tpch -t lineitem -n 600040 -r \| -c 1
LOAD -o file:///data/tpch/100g/nation -s tpch -t nation -n 100 -r \| -c 1
LOAD -o file:///data/tpch/100g/orders -s tpch -t orders -n 638300 -r \| -c 1
LOAD -o file:///data/tpch/100g/part -s tpch -t part -n 769240 -r \| -c 1
LOAD -o file:///data/tpch/100g/partsupp -s tpch -t partsupp -n 360370 -r \| -c 1
LOAD -o file:///data/tpch/100g/region -s tpch -t region -n 10 -r \| -c 1
LOAD -o file:///data/tpch/100g/supplier -s tpch -t supplier -n 333340 -r \| -c 1
```

Disable encoding:
```
LOAD -o file:///scratch/liyu/opt/data/tpch-1/customer -s tpch_1 -t customer -n 319150 -r \| -c 1 -e 0
LOAD -o file:///scratch/liyu/opt/data/tpch-1/lineitem -s tpch_1 -t lineitem -n 600040 -r \| -c 1 -e 0
LOAD -o file:///scratch/liyu/opt/data/tpch-1/nation -s tpch_1 -t nation -n 100 -r \| -c 1 -e 0
LOAD -o file:///scratch/liyu/opt/data/tpch-1/orders -s tpch_1 -t orders -n 638300 -r \| -c 1 -e 0
LOAD -o file:///scratch/liyu/opt/data/tpch-1/part -s tpch_1 -t part -n 769240 -r \| -c 1 -e 0
LOAD -o file:///scratch/liyu/opt/data/tpch-1/partsupp -s tpch_1 -t partsupp -n 360370 -r \| -c 1 -e 0
LOAD -o file:///scratch/liyu/opt/data/tpch-1/region -s tpch_1 -t region -n 10 -r \| -c 1 -e 0
LOAD -o file:///scratch/liyu/opt/data/tpch-1/supplier -s tpch_1 -t supplier -n 333340 -r \| -c 1 -e 0
```


Now the pixel data is in the directory `/scratch/liyu/opt/pixels_file/pixels-tpch-0_1`.


### k. run tpch query


```
cd /scratch/liyu/opt/trino-server
./bin/trino --server localhost:7080 --catalog pixels
```

The expected output:

```
trino> use tpch;
USE
trino:tpch> show tables;
  Table
----------
 customer
 lineitem
 nation
 orders
 part
 partsupp
 region
 supplier
(8 rows)

Query 20230224_140422_00131_n4ycf, FINISHED, 1 node
Splits: 35 total, 35 done (100.00%)
0.13 [8 rows, 166B] [62 rows/s, 1.27KB/s]

trino:tpch> select count(*) from customer;
 _col0
-------
 45000
(1 row)

Query 20230224_140432_00132_n4ycf, FINISHED, 1 node
Splits: 34 total, 34 done (100.00%)
0.08 [45K rows, 0B] [600K rows/s, 0B/s]

trino:tpch> select count(*) from lineitem;
 _col0
--------
 600572
(1 row)
```

### l. summary of adding a new pxl dataset
* schema.sql: modify schema name
* layout.sql: modify path and tbl_id
* pixels-sink: modify path and schema

## 2. Debug `pixels-sink`

### a. change `pixels.properties`
```
pixel.stride=5
row.group.size=1
```

The unit of `row.group.size` is byte, not row. 

### d. edit pixels repo

Change `DEFAULT_SIZE` in `VectorizedRowBatch.java` to 5. This parameter means that each time we process 5 rows. The 5 rows cannot be seperated to different row group. This is why we set `row.group.size` as 1 byte. Now we designate the row count in each row group as 5. 

Then run `run.sh` to compile the pixels

### c. run `pixels-sink`

```
java -jar -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:30001 ./sbin/pixels-sink-*-full.jar
```

### d. run JVM debugger on Intellij
![8a42804bd717101df834cf6141637d8](https://user-images.githubusercontent.com/41314695/230728196-5be3e695-1f41-4bb8-b300-405178a5f56c.png)


### e. command on `pixels-sink`

```
LOAD -f pixels -o file:///scratch/liyu/opt/data/tpch-0_1g/nation -s tpch -t nation -n 10 -r \| -c 1 -e 0
```

Here we set `-n` as 10. `n` represents the maximal row in a pixels file. 

## 3. Start vtune
```
cd /opt/intel/oneapi/vtune/lastest
source vtune-vars.sh
vtune-backend --web-port=9080 --enable-server-profiling --allow-remote-access --reset-passphrase --data-directory /scratch/liyu/opt/vtune

```

## 4. Check page cache

Under Linux, you can check the page cache size via the command `free -w -h`. The page cache size is indicated in the cache column. 

