# Automatic Table Partitioning (Date Range) in Oracle Database using Bash shell script.
Linux Bash shell script to Automatically create new date range partitions for a table every month and delete older partitions.

Lets say, I have a large table called Orders, which I need to partition based on each Month data.
Each partition will contains data for a specific month, and partitions older than 1 year will be deleted automatically.
Each partitions will reside in its own tablespace with same name.


##### The Bash script will 
- Create New Tablespace & Partion for each month.
- Drop Older than 1 year Partion & tablespace including datafiles.
- Rebuild Indexes and execute gather stats on the Table.

Below is an example of Partition & Tablespace naming structure

- Table Name           = ORDERS
- New Partition Name   = ORDERS_202302
- New Tablespace Name  = ORDERS_202302

```
select PARTITION_NAME,
	tablespace_name,
	NUM_ROWS
from dba_tab_partitions 
where TABLE_OWNER = 'schema-name'
and TABLE_NAME='ORDERS' order by partition_name desc;

PARTITION_NAME            TABLESPACE_NAME             NUM_ROWS
------------------------- ------------------------- ----------
ORDERS_202302		  ORDERS_202302						        -- > To be created, new partiotion for new month
ORDERS_202301         ORDERS_202301                 4074603
ORDERS_202212         ORDERS_202212                 5204062
ORDERS_202211         ORDERS_202211                 4981191
ORDERS_202210         ORDERS_202210                 6456684
ORDERS_202209         ORDERS_202209                 4821135
ORDERS_202208         ORDERS_202208                 5066927
ORDERS_202207         ORDERS_202207                 5824346
ORDERS_202206         ORDERS_202206                 6477975
ORDERS_202205         ORDERS_202205                 5091088
ORDERS_202204         ORDERS_202204                 5774805
ORDERS_202203         ORDERS_202203                 7063802
ORDERS_202202         ORDERS_202202                 5876999
ORDERS_202201         ORDERS_202201                 5098684	    -- > To be Dropped, 1 year old partition
OTHER                 ORDERS_OTHER                 102375659
```
