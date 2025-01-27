# Spark-SqoopCDC

Change data capture realization using Spark and Sqoop

[Sqoop documentation link](https://sqoop.apache.org/docs/1.4.7/SqoopUserGuide.html)

[Spark documentation link](https://spark.apache.org/docs/latest/)

## Problem
HDFS table needs to capture changes from the source RDB table. For example source RDB table has 1 million records and half of them have been changed, so we need to synchronize changed records with our HDFS database.

## Solution
1) We're going to use Sqoop in incremental mode to import only those records that were modified
2) We will store modified records in parquet file 
3) Using Spark SQL, create new tables: one for extracted data and another for updated original table.
4) Using Spark SQL, insert unchanged data merged with modified data using LEFT JOIN

## Requirements
1) Installed Spark and Sqoop
2) JDBC driver for Sqoop
3) Running hadoop cluster
4) Running spark-shell

## Instruction for Sqoop data import
Standart template for sqoop import as parquet file:

`sqoop import --connect <your jdbc driver with DB> --username <username> --password <password> --table <table name> --target-dir <path to HDFS directory> --as-parquetfile`

Sqoop incremental import can capture both new and modified records. If we need to import new records, we need to add next parameter:

`--check-column <name of column> --incremental append --last-value <value of the last record, that wasn't changed>`

If we need to capture updates:

`--check-column <name of column> --incremental lastmodified --last-value <value of the last record, that wasn't changed>`

## Using Sqoop job to simplify CDC
We can save our incremental import command for multiple usage without specifying --last-value attribute. Example: 

`sqoop job --create MY_JOB_NAME --import --connect` etc.

Sqoop will identify last-value attribute on first run and will change it if there will be larger value, so we don't need to specify it manually. To execute job run

`sqoop job --exec JOB_NAME`

## Using Spark for synchronization
Let our original table be called Customers. After sqoop operation there will be parquet file with changed data. To make updated variant of original table, follow next steps.
1) Create temporary table called customers_extract

  `val parquetFile = sqlContext.read.parquet("path/to/your/parquet")`
  
  `parquetFile.registerTempTable("customers_extract")`

2) Create table called customers_updated, where we will merge updates and original table

  `sql = "DROP TABLE if exists customers_updated"`
  
  `sqlContext.sql(sql)`
  
  `sql = "CREATE TABLE customers_updated ( cust_no int ,birth_date date ,first_name string ,last_name string ,gender string ,join_date date ,created_date timestamp ,modified_date timestamp ) STORED AS PARQUET"`
  
  `sqlContext.sql(sql)`
  
 3) Get exisiting_records_without_recent_query
 
 `exisiting_records_without_recent_query = "create table db.exisiting_records_without_recent_query as SELECT a.cust_no, a.birth_date, a.first_name, a.last_name, a.gender, a.join_date, a.created_date, a.modified_date FROM customer a LEFT OUTER JOIN customer_extract b ON a.cust_no = b.cust_no WHERE b.cust_no IS NULL"`
 
 `spark.sql(exisiting_records_without_recent_query)`
 
 4) Now Update the customers table with new data and exisiting_records_without_recent_query

 `  final_query = insert over customers select a.* from (select * from customer_extract union all select * from exisiting_records_without_recent_query) a `

  `spark.sql(final_query)`
