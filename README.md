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
