# Hadoop

## Batch
### Source files to HDFS using CLI
```sh
hadoop fs -mkdir -p /tmp/file/sink
hadoop fs -put /src_sys_batch/customer.csv /tmp/file/sink
```
### Create a structure table over a file for a query
- LOCATION '/tmp/file/sink/'
```sh
hive -f /init_tbl/init_hive_customers_tbl.sql
```

### Spark (cleansing and transformation)
- open spark session
- table to df
- clean data
    - withColumn
        - trim
        - to_date
            - ex. `to_date`(col("..."),`"yyyy-MM-dd"`).cast("string")
        - regexp_replace
        - expr("`case` when ... then ... when ... then ... else __ `end`")
- selectExpr
    - ex. selectExpr("customer_id `as` cust_id",\\
          "gender `as` cust_gender")
- write to destination path
- submit script
```sh
spark-submit /spark/spark_batch.py
```

### Create a structure table over a file for a query
- LOCATION '/tmp/default/customers_cln/'
```sh
hive -f /init_tbl/init_hive_customers_cln_tbl.sql
```

## Stream Layer
### Prepare path source and sink
- source path
```sh
mkdir /flume/src/hdfs
mkdir /flume/src/hbase
```
- sink path (HDFS)
```sh
hadoop fs -mkdir -p /tmp/flume/sink
```
- sink table (Hbase)
```sh
hbase shell
#hbase shell (sync with Hue → Hbase Browser)
create ‘spooled_table’, ‘spool_cf’
list
exit
```

### log files as a source & `HDFS` as a sink
- config path: /flume/config/flume_hdfs.conf
```sh
tier1.sources.src-1.spoolDir = /flume/src/hdfs
tier1.sinks.snk-1.type = hdfs
tier1.sinks.snk-1.hdfs.path = /tmp/flume/sink/
```

### log files as a source & `Hbase` as a sink
- config path: /flume/config/flume_hbase.conf
```sh
tier2.sources.src-2.spoolDir = /flume/src/hbase
tier2.sinks.snk-2.type = org.apache.flume.sink.hbase.HBaseSink
tier2.sinks.snk-2.table = spooled_table
tier2.sinks.snk-2.columnFamily = spool_cf
```

### Run flume
- HDFS
```sh
nohup flume-ng agent -n tier1 -f /flume/config/flume_hdfs.conf &
# nohup … & = run as a bg
# exit: ^c
# list jobs: jobs -l
```
- Hbase
```sh
nohup flume-ng agent -n tier2 -f /flume/config/flume_hbase.conf &
```
