# Hadoop
## Prepare environment
### Create VM
- Enable Compute Engine API
- Create vm instance
- setup network firewall of vm
    - open firewall for all inbound for this practice only (not secure)
    - create a firewall rule for access via Web UI
        - allow tcp port : 7180 for Cloudera Manager
        - allow tcp port : 8888 for Cloudera HUE
- go to SSH terminal

### Cloudera Hadoop via Docker container
#### Install and Use Docker on Ubuntu 18.04
https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-18-04
- step 1-4
- In step 2, use this command for find user and set password

```bash
#own password
passwd

# change password for other user account
## login as the root user
su -
passwd ${USER}
```

#### Pull docker image to my vm

```bash
#pull image
docker pull mikelemikelo/cloudera-spark:latest

#check image
docker images

#run image
docker run --hostname=quickstart.cloudera --privileged=true -it -p 8888:8888 -p8080:8080 -p 7180:7180 -p 88:88/udp -p 88:88 mikelemikelo/cloudera-spark:latest /usr/bin/docker-quickstart-light
```

#### Access Cloudera Manager

```bash
sudo /home/cloudera/cloudera-manager --express && service ntpd start
```
- use Public / External IP of VM with port 7180
- ex. http://34.126.72.57:7180/ -> user: cloudera, pwd: cloudera


#### Set up Cloudera Manager

- set configuration of hive service is none
- delete useless sevice ex. Spark, KV Indexer
(Spark of CDH is version 1.x.x that cannot use spark structured streaming → use Spark Local in docker container instead)
    - Add service “Flume”
- start cluster
- access system via Cloudera HUE
    - use Public / External IP of VM with port 8888
    - ex. http://34.126.72.57:8888/ -> user: cloudera, pwd: cloudera
---
## Batch
### Source files to HDFS by using CLI
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
---
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


### Generate log files from source system

- log files stream to HDFS and Hbase by using Flume

    ```sh
    nohup sh /src_sys_stream/src_sys.sh &
    # if file syntax error 
    # rm and vi (again)
    ```
- check results
    - source
        ```sh
        ls -l /flume/src/hdfs
        ls -l /flume/src/hbase
        ```
    - sink
        - HDFS
            ```sh
                hadoop fs -ls /tmp/flume/sink
            ```
        - Hbase
            ```sh
                hbase shell
                scan ‘spooled_table’
            ```

##### Note
- if exit and start again
    ```sh
    docker container ls
    docker exec -it <container_id> /bin/bash
    
    nohup flume-ng agent -n tier1 -f /flume/config/flume_hdfs.conf &
    nohup flume-ng agent -n tier2 -f /flume/config/flume_hbase.conf &
    
    nohup sh /src_sys_stream/src_sys.sh &
    ```
    
### Create a structure table over a file for a query
- LOCATION '/tmp/flume/sink/'
    ```sh
    hive -f /init_tbl/init_hive_transactions_tbl.sql
    ```

### Spark Streaming (cleansing and transformation)
- open spark session
- declare checkpoint (for fault torrance)
- declare structure
- readStream csv to df
- clean data
    - withColumn
        - split
            - ex. 
                `split`(df['customer_order'], `' '`).`getItem(1)`.cast('integer')
        - from_unixtime
            - ex. `from_unixtime`(col("order_timestamp"),`'dd-MM-yyyy HH:mm:ss'`).cast('string')
        - drop
- selectExpr
- write stream file to destination path
- submit script
    ```sh
    nohup spark-submit /spark/spark_streaming.py &
    ```

### Create a structure table over a file for a query
- LOCATION '/tmp/default/transactions_cln/'
    ```sh
    hive -f /init_tbl/init_hive_transactions_cln_tbl.sql
    ```
---
## Join data
### Create target table
- LOCATION '/tmp/default/loyalty/'
    ```sh
    hive -f /init_tbl/init_hive_loyalty_tbl.sql
    ```
- Add permission for write path (target table)

    ```sh
    hadoop fs -chmod -R 777 /tmp/default/loyalty
    
    # or if error chmod: '/tmp/default/lotalty': No such file or directory
    hadoop fs -chmod -R 777 /tmp/
    ```
- Excecute  
    ```
    INSERT OVERWRITE TABLE loyalty
    PARTITION(data_dt='2022-01-01')
    select a.cust_id, a.cust_nm, a.cust_member_card_no, sum(b.odr_prc) as spnd_amt
    from customers_cln as a
    join transactions_cln as b
    on a.cust_id = b.cust_id
    group by a.cust_id, a.cust_nm, a.cust_member_card_no
    ```
---
## Oozie
- Control access by using Hue Web 
- Services: Workflow and Coordinator
- Put script from local to hadoop
    ```sh
    hadoop fs -put /tnfm_script/insert_hive_loyalty.sql /tmp/file
    ```
- If want to add parameters in script
    ```sh
    PARTITION(data_dt='${data_dt}')
    ```
### Workflow (pipeline)
1. Create workflow
2. Drag hive2 script
    - Choose script: insert_hive_loyalty.sql
    - Add parameters: data_dt=${data_dt_wf}
3. Save

### Coordinator (trigger)
1. Create coordinator
2. Choose workflow
3. How often? 
    - Every…
    - Timezone
    - From...To... (when you submit a coordinator with start date in the past it catches up)
4. Add Parameters: data_dt_wf = ${coord:formatTime(coord:nominalTime(), 'yyyyMMdd')}
5. Save
6. Submit
