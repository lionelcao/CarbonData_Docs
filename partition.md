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
     customerid int
     ...
     ...)
[partition by {PartitionType} {PartitionColumn}([PartitionName1](PartitionValue1),[PartitionName2](PartitionValue2) ...)]
Stored By 'carbondata'
[tblproperties(...)];
```
PartitionName is OPTIONAL when user create table. If user specified PartitionName then use it else System will auto-increase the partition number from 0 to CARBON_MAX_PARTITIONS(User can set in carbon.properties). Finally system will format the create statement and fill the PartitionName.

range partition: 
     
1. Fixed Range Partition（Static)
```
  partition by range logdate(
         part0(<'2016-01-01'), 
         part1(<'2017-01-01'), 
         part2(<'2017-02-01'), 
         part3(<'2017-03-01'), 
         part4(<'2017-04-01'))
```
2. Interval Range Partition(Dynamic)
```     
  partition by range logdate(
         case <'2016-01-01' interval '1' YEAR, 
         case <'2017-01-01' interval '1' MONTH, 
         case other interval '1' DAY)
```         
list partition:

1. Single List Partition

       partition by list area('Asia','Europe','North America','Africa','Oceania')
2. Array List Partition

       partition by list country(('China','India'),('Canada','America'),('England','France'),'Australia')
3. Full List Partition     #Any better name for this?

       partition by list area('Asia','Europe','North America','Africa','Oceania',&)   
       ##add symbol '&' in the end so that all values not in the list will be put in another partition
hash partition:

       partition by hash(itemid, 9)  #9 is partition number here
       
~~composite partition:~~

 ~~partition by range logdate(<  '2016- -01', < '2017-01-01', < '2017-02-01', < '2017-03-01', < '2099-01-01')~~
  
 ~~subpartition by list area('Asia', 'Europe', 'North America', 'Africa', 'Oceania')~~

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
partitioni info should be stored in file footer/index file and load into memory before user query.

### Relationship with Bucket
Bucket should be lower level of partition.

### Query on Partition Table
```
Example:
Select * from sales
where logdate <= date '2016-12-01';
```
User should remember to add a partition filter when write SQL on a partition table.
