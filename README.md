# Star Schema Benchmark on Azure Data Explorer versus Clickhouse

## What is Star Schema Benchmark(SSB)
Star Schema Benchmark, aks SSB, is designed to measure performance of database products in support of data warehousing
application. It is developed based on [TPC-H benchmark](https://www.tpc.org/tpch/) but with a curated schema version.
Simply speaking, SSB benchmarking schema is easier for developer to verify the performance of major commercial
databases with a concise schema and queries.

**TPC-H Schema**

![TPCH Schema](pics/tpch.png)

**SSB Schema**

![SSB Schema](pics/ssb.png)

If you are interested in the details of Star Schema Benchmark, please visit the [Paper](https://www.cs.umb.edu/~poneil/StarSchemaB.pdf)
for more details.

## 2. Generate Data

1. Compile SSB-Benchmark

   ```shell
   git clone https://github.com/vadimtk/ssb-dbgen.git
   cd ssb-dbgen
   make clean
   make
   ```

2. Generate SSB data files

   ```shell
   ./dbgen -s 100 -T a
   ```
   
3. Create table schema in Clickhouse

    ```shell
    CREATE DATABASE ssb;
   
    CREATE TABLE ssb.customer
    (
    C_CUSTKEY       UInt32,
    C_NAME          String,
    C_ADDRESS       String,
    C_CITY          LowCardinality(String),
    C_NATION        LowCardinality(String),
    C_REGION        LowCardinality(String),
    C_PHONE         String,
    C_MKTSEGMENT    LowCardinality(String)
    )
    ENGINE = MergeTree ORDER BY (C_CUSTKEY);
    
    ## Create Distributed Table in Clickhouse cluster
    CREATE TABLE ssb.dist_customer ON CLUSTER benchmark as ssb.customer engine = Distributed(benchmark, ssb, customer, rand());
    
    CREATE TABLE ssb.lineorder
    (
    LO_ORDERKEY             UInt32,
    LO_LINENUMBER           UInt8,
    LO_CUSTKEY              UInt32,
    LO_PARTKEY              UInt32,
    LO_SUPPKEY              UInt32,
    LO_ORDERDATE            Date,
    LO_ORDERPRIORITY        LowCardinality(String),
    LO_SHIPPRIORITY         UInt8,
    LO_QUANTITY             UInt8,
    LO_EXTENDEDPRICE        UInt32,
    LO_ORDTOTALPRICE        UInt32,
    LO_DISCOUNT             UInt8,
    LO_REVENUE              UInt32,
    LO_SUPPLYCOST           UInt32,
    LO_TAX                  UInt8,
    LO_COMMITDATE           Date,
    LO_SHIPMODE             LowCardinality(String)
    )
    ENGINE = MergeTree PARTITION BY toYear(LO_ORDERDATE) ORDER BY (LO_ORDERDATE, LO_ORDERKEY);
    
    CREATE TABLE ssb.dist_lineorder ON CLUSTER benchmark as ssb.lineorder engine = Distributed(benchmark, ssb, lineorder, rand());
    
    CREATE TABLE ssb.part
    (
    P_PARTKEY       UInt32,
    P_NAME          String,
    P_MFGR          LowCardinality(String),
    P_CATEGORY      LowCardinality(String),
    P_BRAND         LowCardinality(String),
    P_COLOR         LowCardinality(String),
    P_TYPE          LowCardinality(String),
    P_SIZE          UInt8,
    P_CONTAINER     LowCardinality(String)
    )
    ENGINE = MergeTree ORDER BY P_PARTKEY;
    
    CREATE TABLE ssb.dist_part ON CLUSTER benchmark as ssb.part engine = Distributed(benchmark, ssb, part, rand());
    
    CREATE TABLE ssb.supplier
    (
    S_SUPPKEY       UInt32,
    S_NAME          String,
    S_ADDRESS       String,
    S_CITY          LowCardinality(String),
    S_NATION        LowCardinality(String),
    S_REGION        LowCardinality(String),
    S_PHONE         String
    )
    ENGINE = MergeTree ORDER BY S_SUPPKEY;
    
    CREATE TABLE ssb.dist_supplier ON CLUSTER benchmark as ssb.supplier engine = Distributed(benchmark, ssb, supplier, rand());
    
    
    CREATE TABLE dates
    (
    D_DATEKEY          Date,
    D_DATE             String,
    D_DAYOFWEEK        String,
    D_MONTH            String,
    D_YEAR             UInt8,
    D_YEARMONTHNUM     UInt8,
    D_YEARMONTH        String,
    D_DAYNUMINWEEK     UInt8,
    D_DAYNUMINMONTH    UInt8,
    D_DAYNUMINYEAR     UInt8,
    D_MONTHNUMINYEAR   UInt8,
    D_WEEKNUMINYEAR    UInt8,
    D_SELLINGSEASON    String,
    D_LASTDAYINWEEKFL  UInt8,
    D_LASTDAYINMONTHFL UInt8,
    D_HOLIDAYFL        UInt8,
    D_WEEKDAYFL        UInt8
    )
    ENGINE = MergeTree ORDER BY (D_DATEKEY);
    
    CREATE TABLE ssb.dist_dates ON CLUSTER benchmark as ssb.dates engine = Distributed(benchmark, ssb, dates, rand());
    ```
4. Load data into Clickhouse

   ```shell
    clickhouse-client --query "INSERT INTO ssb.customer FORMAT CSV" < customer.tbl
    clickhouse-client --query "INSERT INTO ssb.part FORMAT CSV" < part.tbl
    clickhouse-client --query "INSERT INTO ssb.supplier FORMAT CSV" < supplier.tbl
    clickhouse-client --query "INSERT INTO ssb.lineorder FORMAT CSV" < lineorder.tbl
    clickhouse-client --query "INSERT INTO ssb.dates FORMAT CSV" < dates.tbl
   ```
5. Create a flat lineorder fact table for benchmark

    ```shell
    SET max_memory_usage = 17179869184;
       
    CREATE TABLE ssb.lineorder_flat on cluster benchmark
    ENGINE = ReplicatedMergeTree
    PARTITION BY toYear(LO_ORDERDATE)
    ORDER BY (LO_ORDERDATE, LO_ORDERKEY) AS
    SELECT
    l.LO_ORDERKEY AS LO_ORDERKEY,
    l.LO_LINENUMBER AS LO_LINENUMBER,
    l.LO_CUSTKEY AS LO_CUSTKEY,
    l.LO_PARTKEY AS LO_PARTKEY,
    l.LO_SUPPKEY AS LO_SUPPKEY,
    l.LO_ORDERDATE AS LO_ORDERDATE,
    l.LO_ORDERPRIORITY AS LO_ORDERPRIORITY,
    l.LO_SHIPPRIORITY AS LO_SHIPPRIORITY,
    l.LO_QUANTITY AS LO_QUANTITY,
    l.LO_EXTENDEDPRICE AS LO_EXTENDEDPRICE,
    l.LO_ORDTOTALPRICE AS LO_ORDTOTALPRICE,
    l.LO_DISCOUNT AS LO_DISCOUNT,
    l.LO_REVENUE AS LO_REVENUE,
    l.LO_SUPPLYCOST AS LO_SUPPLYCOST,
    l.LO_TAX AS LO_TAX,
    l.LO_COMMITDATE AS LO_COMMITDATE,
    l.LO_SHIPMODE AS LO_SHIPMODE,
    c.C_NAME AS C_NAME,
    c.C_ADDRESS AS C_ADDRESS,
    c.C_CITY AS C_CITY,
    c.C_NATION AS C_NATION,
    c.C_REGION AS C_REGION,
    c.C_PHONE AS C_PHONE,
    c.C_MKTSEGMENT AS C_MKTSEGMENT,
    s.S_NAME AS S_NAME,
    s.S_ADDRESS AS S_ADDRESS,
    s.S_CITY AS S_CITY,
    s.S_NATION AS S_NATION,
    s.S_REGION AS S_REGION,
    s.S_PHONE AS S_PHONE,
    p.P_NAME AS P_NAME,
    p.P_MFGR AS P_MFGR,
    p.P_CATEGORY AS P_CATEGORY,
    p.P_BRAND AS P_BRAND,
    p.P_COLOR AS P_COLOR,
    p.P_TYPE AS P_TYPE,
    p.P_SIZE AS P_SIZE,
    p.P_CONTAINER AS P_CONTAINER
    FROM ssb.lineorder AS l
    INNER JOIN ssb.customer AS c ON c.C_CUSTKEY = l.LO_CUSTKEY
    INNER JOIN ssb.supplier AS s ON s.S_SUPPKEY = l.LO_SUPPKEY
    INNER JOIN ssb.part AS p ON p.P_PARTKEY = l.LO_PARTKEY
    ;
    ```

6. Unload data into csv files 

   ```shell
   clickhouse-client --query "SELECT * from ssb.lineorder_flat" --format CSV > lineorder_flat.csv
   clickhouse-client --query "SELECT * from ssb.lineorder" --format CSV > lineorder.csv
   clickhouse-client --query "SELECT * from ssb.customer" --format CSV > customer.csv
   clickhouse-client --query "SELECT * from ssb.part" --format CSV > part.csv
   clickhouse-client --query "SELECT * from ssb.supplier" --format CSV > supplier.csv
   clickhouse-client --query "SELECT * from ssb.dates" --format CSV > dates.csv                         
   ```

   Truncate table data for data backfill into shards in Clickhouse cluster
    ```
   truncate table lineorder_flat;
   truncate table lineorder;
   truncate table customer;
   truncate table part;
   truncate table supplier;
   truncate table dates;
   ```

7. Load data back to Distributed table in multiple shards in Clickhouse cluster

   ```shell
    clickhouse-client --query "INSERT INTO ssb.dist_customer FORMAT CSV" < customer.csv
    clickhouse-client --query "INSERT INTO ssb.dist_part FORMAT CSV" < part.csv
    clickhouse-client --query "INSERT INTO ssb.dist_supplier FORMAT CSV" < supplier.csv
    clickhouse-client --query "INSERT INTO ssb.dist_lineorder FORMAT CSV" < lineorder.csv
    clickhouse-client --query "INSERT INTO ssb.dist_lineorder_flat FORMAT CSV" < lineorder_flat.csv
    clickhouse-client --query "INSERT INTO ssb.dist_dates FORMAT CSV" < dates.csv
   ```

8. Split the fact tables (lineorder/lineorder_flat) data file into multiple segments less than 4GB, which is the limit file size for Azure Data Explorer Lightingest

    For scale factor 100, the total line number of lineorder is 600,038,145, we will evenly distribute into multiple files.

    ``` shell
    split -d -l 35000000 lineorder.csv lineorder_part_
    split -d -l 10020000 /data3/clickouse/data/lineorder_flat.csv lineorder_flat_part_
    ```

9. Upload segment files to ADLS Gen2 for ADX data ingestion

   ```shell
   az storage fs directory upload -f ssb --account-name <storage account> -s lineorder/* -d 100G/lineorder/csv/ --recursive
   az storage fs directory upload -f ssb --account-name <storage account> -s lineorder_flat/* -d 100G/lineorder_flat/csv/ --recursive
   ```

## 3. Use Light Ingest to ingest segment data into Azure Data Explorer

Please refer to below official document to create ADX cluster and database for preparation
https://docs.microsoft.com/en-us/azure/data-explorer/create-cluster-database-portal

Please refer to below official document to ingest data into ADX cluster
1. [Install Light Ingest](https://docs.microsoft.com/en-us/azure/data-explorer/lightingest)
2. [Use wizard for one-time ingestion of historical data with LightIngest](https://docs.microsoft.com/en-us/azure/data-explorer/generate-lightingest-command)

## 4. Query

Here is a list of SSB queries, the query parameter may be different between different *scale factor*. The sample data is generated randomly.

*Notice that the queries may be slightly different for different scale factor in the filtering constants. The queries below are tested with scale factor 10.*

##### Q1.1

Flat table
```sql
lineorder_flat
| where LO_ORDERDATE between(datetime(1993-01-01) .. datetime(1993-12-31))
    and LO_DISCOUNT  between (1 .. 3)
    and LO_QUANTITY < 25
| summarize revenue = sum(LO_EXTENDEDPRICE * LO_DISCOUNT)
```

Multi table join

```sql
lineorder_daily_partition
| lookup  kind=leftouter  ['dates'] on $left.LO_ORDERDATE == $right.D_DATEKEY
| where LO_ORDERDATE between(datetime(1993-01-01) .. datetime(1993-12-31))
    and LO_DISCOUNT between (1 ..3)
    and LO_QUANTITY < 25
| summarize revenue = sum(LO_EXTENDEDPRICE * LO_DISCOUNT)
```

##### Q1.2

Flat table
```sql
lineorder_flat
| where LO_ORDERDATE between(datetime(1994-01-01) .. datetime(1994-01-31))
    and LO_DISCOUNT between (4 .. 6)
    and LO_QUANTITY between (26 .. 35)
| summarize revenue = sum(LO_EXTENDEDPRICE * LO_DISCOUNT)
```

Multi table join

```sql
lineorder_daily_partition
| lookup  kind=leftouter  ['dates'] on $left.LO_ORDERDATE == $right.D_DATEKEY
| where LO_ORDERDATE between(datetime(1994-01-01) .. datetime(1994-01-31))
    and LO_DISCOUNT between (4 ..6)
    and LO_QUANTITY between (26 .. 35)
| summarize revenue = sum(LO_EXTENDEDPRICE * LO_DISCOUNT)
```
##### Q1.3

Flat table
```sql
lineorder_flat
| where LO_ORDERDATE between(datetime(1994-01-01) .. datetime(1994-12-31))
    and week_of_year(LO_ORDERDATE) == 6
    and LO_DISCOUNT between (5 .. 7)
    and LO_QUANTITY between (26 .. 35)
| summarize revenue = sum(LO_EXTENDEDPRICE * LO_DISCOUNT)
```

Multi table join

```sql
lineorder_daily_partition
| lookup  kind=leftouter ['dates'] on $left.LO_ORDERDATE == $right.D_DATEKEY
| where LO_ORDERDATE between(datetime(1994-01-01) .. datetime(1994-12-31))
    and D_WEEKNUMINYEAR == 6
    and LO_DISCOUNT between (5 ..7)
    and LO_QUANTITY between (26 .. 35)
| summarize revenue = sum(LO_EXTENDEDPRICE * LO_DISCOUNT)
```
##### Q2.1

Flat table
```sql
lineorder_flat
| extend order_year = getyear(LO_ORDERDATE)
| where P_CATEGORY == 'MFGR#12'
    and S_REGION == 'AMERICA'
| summarize revenue = sum(LO_REVENUE) by order_year, P_BRAND
| order by order_year, P_BRAND
| project revenue, order_year, P_BRAND

```

Multi table join

```sql
lineorder_daily_partition
| lookup  kind=inner  ['dates'] on $left.LO_ORDERDATE == $right.D_DATEKEY
| lookup  kind=inner  (['part'] | where P_CATEGORY == 'MFGR#12') on $left.LO_PARTKEY == $right.P_PARTKEY
| lookup  kind=inner  (['supplier'] | where S_REGION == 'AMERICA') on $left.LO_SUPPKEY == $right.S_SUPPKEY
| summarize revenue = sum(LO_REVENUE) by D_YEAR, P_BRAND
| order by D_YEAR, P_BRAND
| project revenue, D_YEAR, P_BRAND

```
##### Q2.2

Flat table
```sql
lineorder_flat
| extend order_year = getyear(LO_ORDERDATE)
| where P_BRAND matches regex "MFGR#222[1-8]"
    and S_REGION == 'ASIA'
| summarize revenue = sum(LO_REVENUE) by order_year, P_BRAND
| order by order_year, P_BRAND
| project revenue, order_year, P_BRAND
```

Multi table join

```sql
lineorder_daily_partition
| lookup  kind=inner  ['dates'] on $left.LO_ORDERDATE == $right.D_DATEKEY
| lookup  kind=inner  (['part'] | where P_BRAND matches regex "MFGR#222[1-8]") on $left.LO_PARTKEY == $right.P_PARTKEY
| lookup  kind=inner  (['supplier'] | where S_REGION == 'ASIA') on $left.LO_SUPPKEY == $right.S_SUPPKEY
| summarize revenue = sum(LO_REVENUE) by D_YEAR, P_BRAND
| order by D_YEAR, P_BRAND
| project revenue, D_YEAR, P_BRAND

```
##### Q2.3

Flat table
```sql
lineorder_flat
| extend order_year = getyear(LO_ORDERDATE)
| where P_BRAND == "MFGR#2239"
    and S_REGION == 'EUROPE'
| summarize revenue = sum(LO_REVENUE) by order_year, P_BRAND
| order by order_year, P_BRAND
| project revenue, order_year, P_BRAND
```

Multi table join

```sql
lineorder_daily_partition
| lookup  kind=inner  ['dates'] on $left.LO_ORDERDATE == $right.D_DATEKEY
| lookup  kind=inner  (['part'] | where P_BRAND == "MFGR#2239") on $left.LO_PARTKEY == $right.P_PARTKEY
| lookup  kind=inner  (['supplier'] | where S_REGION == 'EUROPE') on $left.LO_SUPPKEY == $right.S_SUPPKEY
| summarize revenue = sum(LO_REVENUE) by D_YEAR, P_BRAND
| order by D_YEAR, P_BRAND
| project revenue, D_YEAR, P_BRAND
```
##### Q3.1

Flat table
```sql
lineorder_flat
| extend order_year = getyear(LO_ORDERDATE)
| where C_REGION == 'ASIA'
    and S_REGION == 'ASIA'
    and order_year >= 1992
    and order_year <= 1997
| summarize revenue = sum(LO_REVENUE) by C_NATION, S_NATION, order_year
| order by order_year asc , revenue desc 
| project C_NATION, S_NATION, order_year, revenue
```

Multi table join

```sql
lineorder_daily_partition
| lookup  kind=inner   (['dates'] | where D_YEAR >= 1992 and D_YEAR <= 1997) on $left.LO_ORDERDATE == $right.D_DATEKEY
| lookup  kind=inner  (['customer'] | where C_REGION == "ASIA") on $left.LO_CUSTKEY == $right.C_CUSTKEY
| lookup  kind=inner  (['supplier'] | where S_REGION == 'ASIA') on $left.LO_SUPPKEY == $right.S_SUPPKEY
| summarize revenue = sum(LO_REVENUE) by C_NATION, S_NATION, D_YEAR
| order by D_YEAR asc , revenue desc
| project C_NATION, S_NATION, D_YEAR, revenue
```
##### Q3.2

Flat table
```sql
lineorder_flat
| extend order_year = getyear(LO_ORDERDATE)
| where C_NATION == 'UNITED STATES'
    and S_NATION == 'UNITED STATES'
    and order_year >= 1992
    and order_year <= 1997
| summarize revenue = sum(LO_REVENUE) by C_CITY, S_CITY, order_year
| order by order_year asc , revenue desc
| project C_CITY, S_CITY, order_year, revenue 
```

Multi table join

```sql
lineorder_daily_partition
| lookup  kind=inner   (['dates'] | where D_YEAR >= 1992 and D_YEAR <= 1997) on $left.LO_ORDERDATE == $right.D_DATEKEY
| lookup  kind=inner  (['customer'] | where C_NATION == "UNITED STATES") on $left.LO_CUSTKEY == $right.C_CUSTKEY
| lookup  kind=inner  (['supplier'] | where S_NATION == 'UNITED STATES') on $left.LO_SUPPKEY == $right.S_SUPPKEY
| summarize revenue = sum(LO_REVENUE) by C_CITY, S_CITY, D_YEAR
| order by D_YEAR asc , revenue desc
| project C_CITY, S_CITY, D_YEAR, revenue
```
##### Q3.3

Flat table
```sql
lineorder_flat
| extend order_year = getyear(LO_ORDERDATE)
| where (C_CITY == 'UNITED KI1'
    or C_CITY == 'UNITED KI5')
    and (S_CITY == 'UNITED KI1'
    or S_CITY == 'UNITED KI5')
    and order_year >= 1992
    and order_year <= 1997
| summarize revenue = sum(LO_REVENUE) by C_CITY, S_CITY, order_year
| order by order_year asc , revenue desc 
| project C_CITY, S_CITY, order_year, revenue
```

Multi table join

```sql
lineorder_daily_partition
| lookup  kind=inner   (['dates'] | where D_YEAR >= 1992 and D_YEAR <= 1997) on $left.LO_ORDERDATE == $right.D_DATEKEY
| lookup  kind=inner  (['customer'] | where (C_CITY == 'UNITED KI1' or C_CITY == 'UNITED KI5')) on $left.LO_CUSTKEY == $right.C_CUSTKEY
| lookup  kind=inner  (['supplier'] | where (S_CITY == 'UNITED KI1' or S_CITY == 'UNITED KI5')) on $left.LO_SUPPKEY == $right.S_SUPPKEY
| summarize revenue = sum(LO_REVENUE) by C_CITY, S_CITY, D_YEAR
| order by D_YEAR asc , revenue desc
| project C_CITY, S_CITY, D_YEAR, revenue

```
##### Q3.4

Flat table
```sql
lineorder_flat
| extend order_year = getyear(LO_ORDERDATE)
| where (C_CITY == 'UNITED KI1'
    or C_CITY == 'UNITED KI5')
    and (S_CITY == 'UNITED KI1'
    or S_CITY == 'UNITED KI5')
    and format_datetime(LO_ORDERDATE, 'yyyyMM') == 199712
| summarize revenue = sum(LO_REVENUE) by C_CITY, S_CITY, order_year
| order by order_year asc , revenue desc 
| project C_CITY, S_CITY, order_year, revenue

```

Multi table join

```sql
lineorder_daily_partition
| lookup  kind=inner   (['dates'] | where D_YEARMONTH == 'Dec1997') on $left.LO_ORDERDATE == $right.D_DATEKEY
| lookup  kind=inner  (['customer'] | where (C_CITY == 'UNITED KI1' or C_CITY == 'UNITED KI5')) on $left.LO_CUSTKEY == $right.C_CUSTKEY
| lookup  kind=inner  (['supplier'] | where (S_CITY == 'UNITED KI1' or S_CITY == 'UNITED KI5')) on $left.LO_SUPPKEY == $right.S_SUPPKEY
| summarize revenue = sum(LO_REVENUE) by C_CITY, S_CITY, D_YEAR
| order by D_YEAR asc , revenue desc
| project C_CITY, S_CITY, D_YEAR, revenue
```
##### Q4.1

Flat table
```sql
lineorder_flat
| extend order_year = getyear(LO_ORDERDATE)
| where C_REGION == 'AMERICA'
    and S_REGION == 'AMERICA'
    and (P_MFGR == 'MFGR#1' or P_MFGR == 'MFGR#2')
| summarize profit = sum(LO_REVENUE - LO_SUPPLYCOST) by order_year, C_NATION
| order by order_year asc, C_NATION asc
| project order_year, C_NATION, profit
```

Multi table join

```sql
lineorder_daily_partition
| lookup  kind=inner  ['dates'] on $left.LO_ORDERDATE == $right.D_DATEKEY
| lookup  kind=inner  (['customer'] | where C_REGION == 'AMERICA') on $left.LO_CUSTKEY == $right.C_CUSTKEY
| lookup  kind=inner  (['supplier'] | where S_REGION == 'AMERICA') on $left.LO_SUPPKEY == $right.S_SUPPKEY
| lookup  kind=inner  (['part'] | where P_MFGR == 'MFGR#1' or P_MFGR == 'MFGR#2') on $left.LO_PARTKEY == $right.P_PARTKEY
| summarize profit = (sum(LO_REVENUE) - sum(LO_SUPPLYCOST)) by D_YEAR, C_NATION
| order by D_YEAR, C_NATION
| project D_YEAR, C_NATION, profit
```
##### Q4.2

Flat table
```sql
lineorder_flat
| extend order_year = getyear(LO_ORDERDATE)
| where C_REGION == 'AMERICA'
    and S_REGION == 'AMERICA'
    and (order_year == 1997 or order_year == 1998)
    and (P_MFGR == 'MFGR#1' or P_MFGR == 'MFGR#2')
| summarize profit = sum(LO_REVENUE) - sum(LO_SUPPLYCOST) by order_year, S_NATION, P_CATEGORY
| order by order_year asc, S_NATION asc, P_CATEGORY asc
| project order_year, S_NATION, P_CATEGORY, profit
```

Multi table join

```sql
lineorder_daily_partition
| lookup  kind=inner  (['dates'] | where D_YEAR == 1997 or D_YEAR == 1998) on $left.LO_ORDERDATE == $right.D_DATEKEY
| lookup  kind=inner  (['customer'] | where C_REGION == 'AMERICA') on $left.LO_CUSTKEY == $right.C_CUSTKEY
| lookup  kind=inner  (['supplier'] | where S_REGION == 'AMERICA') on $left.LO_SUPPKEY == $right.S_SUPPKEY
| lookup  kind=inner  (['part'] | where P_MFGR == 'MFGR#1' or P_MFGR == 'MFGR#2') on $left.LO_PARTKEY == $right.P_PARTKEY
| summarize profit = (sum(LO_REVENUE) - sum(LO_SUPPLYCOST)) by D_YEAR, S_NATION, P_CATEGORY
| order by D_YEAR, S_NATION, P_CATEGORY
| project D_YEAR, S_NATION, P_CATEGORY, profit
```
##### Q4.3

Flat table
```sql
lineorder_flat
| extend order_year = getyear(LO_ORDERDATE)
| where S_NATION == 'UNITED STATES'
    and (order_year == 1997 or order_year == 1998)
    and P_CATEGORY == 'MFGR#14'
| summarize profit = sum(LO_REVENUE) - sum(LO_SUPPLYCOST) by order_year, S_CITY, P_BRAND
| order by order_year asc, S_CITY asc, P_BRAND asc
| project order_year, S_CITY, P_BRAND, profit
```

Multi table join

```sql
lineorder_daily_partition
| lookup  kind=inner  (['dates'] | where D_YEAR == 1997 or D_YEAR == 1998) on $left.LO_ORDERDATE == $right.D_DATEKEY
| lookup  kind=inner  (['customer'] | where C_REGION == 'AMERICA') on $left.LO_CUSTKEY == $right.C_CUSTKEY
| lookup  kind=inner  (['supplier'] | where S_NATION == 'UNITED STATES') on $left.LO_SUPPKEY == $right.S_SUPPKEY
| lookup  kind=inner  (['part'] | where P_CATEGORY == 'MFGR#14') on $left.LO_PARTKEY == $right.P_PARTKEY
| summarize profit = (sum(LO_REVENUE) - sum(LO_SUPPLYCOST)) by D_YEAR, S_CITY, P_BRAND
| order by D_YEAR, S_CITY, P_BRAND
| project D_YEAR, S_CITY, P_BRAND, profit
```

## 5. Result

ADX Cluster info:
- 3 nodes; Standard_E8d_v5, 8 cores, 64 GB memory for each node

Clickhouse Cluster info:
- 3 nodes; Standard E8ds v5, 8 cores, 64 GB memory for each node