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