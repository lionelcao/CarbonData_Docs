## Implement Partition Feature Proposal

### Why need partition table
Partition table provide an option to divide table into some smaller pieces. 
With partition table:

      1. Data could be better managed, organized and stored. 
      2. We can avoid full table scan in some scenario and improve query performance. (partition column in filter, 
      multiple partition tables join on the same partition column etc.)

### Partitioning design
#### Range Partitioning           
range partitioning maps data to partitions according to the range of partition column values, operator '<' defines non-inclusive upper bound of current partition.
#### List Partitioning
list partitioning allows you map data to partitions with specific value list
#### Hash Partitioning
hash partitioning maps data to partitions with hash algorithm and put them to the given number of partitions
#### ~~Composite Partitioning(2 levels at most for now)~~
   ~~Range-Range, Range-List, Range-Hash, List-Range, List-List, List-Hash, Hash-Range, Hash-List, Hash-Hash~~

### DDL-Create 
```
Create table sales(
     itemid long, 
     logdate datetime, 
     customerid int,
     country string,
     ...
     ...)
[partition by  (PARTITION_COLUMN)］
Stored By 'carbondata'
[tblproperties('PARTITION_TYPE'='XXX',
              ['RANGE_INFO'='XXX, XXXX, XXXXX'],
              ['LIST_INFO'='XXX, (XXXX, XX), XXXXX'],
              ['HASH_NUMBER'='XX'])];
```
PartitionName is OPTIONAL when user create table. If user specified PartitionName then use it else System will auto-generate the PartitionName. The number of PartitionNames must equal to the number of value list.

range partition: 
     
1. Fixed Range Partition（Static)
```
  PARTITIONED BY (logdate)
  ...
  TBLPROPERTIES('PARTITION_TYPE'='RANGE','RANGE_INFO'='20160101, 20160201, 20160301,20160401')
```
2. Interval Range Partition(Dynamic)
```     
  PARTITIONED BY (logdate)
  ...
  TBLPROPERTIES('PARTITION_TYPE'='RANGE_INTERVAL','PARTITION_VALUE_LIST'='(2016-01-01,YEAR),(2017-05-01,MONTH),(OTHER,DAY)'
```         
list partition:

1. Single List Partition

       PARTITIONED BY (area)
       ...
       TBLPROPERTIES('PARTITION_TYPE'='LIST','LIST_INFO'=(Asia,Europe,North America,Africa,Oceania)
2. Array List Partition

       PARTITIONED BY (country)
       ...
       TBLPROPERTIES('PARTITION_TYPE'='LIST','LIST_INFO'=((China,India),Japan,(Canada,America),Russia, (England,France),Australia))  

hash partition:

       PARTITION BY (itemid)
       ...
       TBLPROPERTIES('PARTITION_TYPE'='HASH','HASH_NUMBER'='9')
       
~~composite partition:~~

 ~~partition by range logdate(<  '2016- -01', < '2017-01-01', < '2017-02-01', < '2017-03-01', < '2099-01-01')~~
  
 ~~subpartition by list area('Asia', 'Europe', 'North America', 'Africa', 'Oceania')~~

### Show PartitionInfo
```
SHOW PARTITIONS #TABLENAME;

----------------------------
ID , NAME, VALUE(Country=)
0, Part0, china
1, Part1, usa
2, Part2, uk
3, Part3, india
...

```

### DDL-~~Rebuild~~, Add, Delete
~~Alter table sales rebuild partition by (range|list|hash)(...);~~
```
add/delete only support range partitioning, list partitioning
Alter table sales add partition ([PartitionName](PartitionValue));
Alter table sales delete partition ({PartitionName});

Example:
Alter table sales add partition (Part5(<'2018-01-01'));    
Alter table sales add partition ('South America');
Alter table sales delete partition (Part5);
```

~~Note: No delete operation for partition, please use rebuild.~~

~~If need delete data, use delete statement, but the definition of partition will not be deleted.~~

### Partition Table Data Store
#### ~~[Option One]~~

~~Use the current design, keep partition folder out of segments~~
```
Fact
   |___Part0
   |     |___Segment_0
   |     |___ *******-[bucketId]-.carbondata
   |     |___ *******-[bucketId]-.carbondata
   |     |___Segment_1
   |     ...
   |___Part1
   |     |___Segment_0
   |     |___Segment_1
   |...
```

#### [Option Two]
remove partition folder, add partition id into file name and build btree in driver side.
```
Fact
   |___Segment_0
   |     |___ *******-[bucketId]-[partitionId].carbondata
   |     |___ *******-[bucketId]-[partitionId].carbondata
   |___Segment_1
   |___Segment_2
   ...
```

#### Pros & Cons: 
Option two could reduce some metadata to be stored. Prefered Option Two for now. 

### Partition Table MetaData Store
partition info should be stored in table schema and load into memory when user initiate carbon/spark session.

### Relationship with Bucket
Bucket should be the lower level of partition.

### Query on Partition Table
```
Example:
Select * from sales
where logdate <= date '2016-12-01';

Select * from sales
join sales_product
on sales.logdate = sales_product.logdate
where sales.logdate <= date '2016-12-01'
;  
#if sales_product is also partitioned on logdate, then optimizer will also add filter 
on sales_product and then do the join.
```
User should remember to add a partition filter when write SQL on a partition table.
