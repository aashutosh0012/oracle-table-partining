# Automatic Table Partitioning (Date Range) in Oracle Database using Bash shell script.
Linux Bash shell script to Automatically create new date range partitions for a table every month and delete older partitions.

Lets say, I have a large table called Orders, which I need to partition based on each Month data.
Each partition will contains data for a specific month, and partitions older than 1 year will be deleted automatically.
Each partitions will reside in its own tablespace with same name.


#### The Bash script will 
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

Table Partitioning Script
-------------------------

```
#!/usr/bin/bash

# export ORACLE enviroment
export ORACLE_SID=SID; . oraenv

# Table Name to be partitioned
TABLE=ORDERS

# Partition Date Range
PARTITION_DATE_RANGE=`date +%m/01/%Y`

# NEW_TABLE_PARTITION = ORDERS_202302  (TABLENAME_YYYYMM, current year) 
NEW_TABLE_PARTITION=${TABLE}_`date +%Y%m`

# OLD_TABLE_PARTITION = ORDERS_202202 (TABLENAME_YYYYMM, last year)
OLD_TABLE_PARTITION=${TABLE}_`expr $(date +%Y%m) - 100`

# NEW_INDEX_TABLESPACE = ORDERS_IX_202302 (INDEXNAME_YYYYMM, current year)
NEW_INDEX_TABLESPACE=${TABLE}_IX_`date +%Y%m`

# OLD_INDEX_TABLESPACE = ORDERS_IX_202202 (INDEXNAME_YYYYMM, last year)
OLD_INDEX_TABLESPACE=${TABLE}_IX_`expr $(date +%Y%m) - 100`


# Get the old last year tablespace (ORDERS_202202) data_file name & location, 
# new tablespace (ORDERS_202302) data file, will be created in the same mount point/directory as old
# OLD_DATAFILE=/storage/APP/DATA001/oradata/orders_202202_01.dbf
OLD_DATAFILE=`sqlplus -s / as sysdba<<EOF
set heading off
set feedback off
set pagesize 0
select file_name from dba_data_files where tablespace_name = '$OLD_TABLE_PARTITION' and rownum=1 order by file_name;
EOF`

# convert names to lower case
OLD_FILE=`echo $OLD_TABLE_PARTITION | tr A-Z a-z`
NEW_FILE=`echo $NEW_TABLE_PARTITION | tr A-Z a-z`
# replace old partition_name to new partition_name in datafile name
# OLD_DATAFILE=/storage/APP/DATA001/oradata/orders_202202_01.dbf
# NEW_DATAFILE=/storage/APP/DATA001/oradata/orders_202302_01.dbf
NEW_DATAFILE=`echo $OLD_DATAFILE | sed "s/$OLD_FILE/$NEW_FILE/g"`

# get filename for index tablespace, and generate new index tablespace file name in same directory
OLD_INDEXFILE=`sqlplus -s / as sysdba<<EOF
set heading off
set feedback off
set pagesize 0
select file_name from dba_data_files where tablespace_name like '$OLD_INDEX_TABLESPACE' and rownum=1 order by file_name;
EOF`

OLD_FILE=`echo $OLD_TABLE_PARTITION | tr A-Z a-z`
NEW_FILE=`echo $NEW_TABLE_PARTITION | tr A-Z a-z`
NEW_INDEXFILE=`echo $OLD_INDEXFILE | sed "s/$OLD_FILE/$NEW_FILE/g"`


echo ""
echo "New_Partition        = $NEW_TABLE_PARTITION"
echo "Old_Partition        = $OLD_TABLE_PARTITION"
echo "New_Tablespace       = $NEW_TABLE_PARTITION"
echo "New_DB_File          = $NEW_DATAFILE"
echo "New_Index_DB_File    = $NEW_INDEXFILE"
echo "New_Index_Partition  = $NEW_INDEX_TABLESPACE"
echo "Old_Index_Partition  = $OLD_INDEX_TABLESPACE"
echo ""

# Login to database, drop old partition & tablespace, create new tablespace & split table paritions
sqlplus / as sysdba <<EOF
set time on
set timing on
set echo on

-- Truncate older Partition
alter table schema.ORDERS truncate partition $OLD_TABLE_PARTITION;

--drop older partition
alter table schema.ORDERS drop partition $OLD_TABLE_PARTITION;

--Offline older tablespace
alter tablespace $OLD_TABLE_PARTITION offline;

-- drop older tablespace including datafiles
drop  tablespace $OLD_TABLE_PARTITION including contents and datafiles;

--offline older index tablespace
alter tablespace $OLD_INDEX_TABLESPACE offline;

--drop older index tablespace including datafiles
drop  tablespace $OLD_INDEX_TABLESPACE including contents and datafiles;

-- create new tablespace
create tablespace $NEW_TABLE_PARTITION datafile '$NEW_DATAFILE' size 8000M;
create tablespace $NEW_INDEX_TABLESPACE datafile '$NEW_INDEXFILE' size 8000M;

-- grant quota to schema user on new tablespace
alter user schema quota unlimited on $NEW_TABLE_PARTITION;
alter user schema quota unlimited on $NEW_INDEX_TABLESPACE;

-- Add & split Parition to table
ALTER TABLE $TABLE SPLIT PARTITION ALL_OTHER AT (TO_DATE('$PARTITION_DATE_RANGE','MM/DD/YYYY')) INTO (PARTITION $NEW_TABLE_PARTITION TABLESPACE $NEW_TABLE_PARTITION, PARTITION OTHER TABLESPACE ORDERS);
EOF


# Generate SQL script to rebuild Indexes
sqlplus -s / as sysdba<<EOF
set lin 300
set heading off
set feedback off
set pagesize 0
spool rebuild_schema_indexes.sql
select 'alter index schema.'||index_name||' rebuild subpartition '||subpartition_name||' tablespace '||tablespace_name||';' from dba_ind_subpartitions where index_name in (select distinct(index_name) from dba_indexes where table_name = 'ORDERS') and status = 'UNUSABLE' order by index_name, partition_name, subpartition_name, tablespace_name;
spool off
EOF

# Rebuild Indexes & Gather Table statistics
sqlplus -s / as sysdba<<EOF
set time on
set timing on
set echo on
@rebuild_schema_indexes.sql
exec dbms_stats.gather_table_stats('schema', 'ORDERS', DEGREE => 8, CASCADE => TRUE);
EOF
```
