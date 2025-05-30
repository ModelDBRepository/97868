# Table of contents:

1. SPUD feature extraction algorithm
2. NEURON interface to MySQL

## 1. SPUD

SPUD feature extraction algorithm implementation as appearing in [1](see bottom of readme)

- spud.mod - main implementation
- spud.hoc - hoc utilities for ease of use
- mosinit.hoc - sets up a GUI showing figure 2 from chapter - run this to see SPUD in action
- rat_strobe_1.vec - single trace of electrocorticographic recordings from rat

Sample routine to demonstrate SPUD feature extraction algorithm (in spud.hoc):
To use: `testspud(vector,num_threshold_slices,log_spacing,[user-specified-thresholds])`
On return, output will store the extracted "bumps" as an NQS database
- `$o1` = input data vector
- `$2` = num threshold lines
- `$3` = threshold spacing, 0=linear,1=log (optional)
- `$o4` = user-specified thresholds to pass in to SPUD (optional)

```hoc
proc testspud()
```

## 2. NEURON interface to MySQL readme - S Neymotin , WW Lytton - 4/2007

(for questions/comments contact samn at neurosim dot downstate dot edu)

This is an interface that allows access to a MySQL server from directly
within the NEURON simulation environment [1]. It allows for performing SQL
queries and returning the results into NEURON data structures. It also
works with the Neural Query System (NQS)[2] and can convert between NQS databases
and MySQL tables.

The MySQL C API is required. The version used was mysql-5.0.37. Some modifications
were made to it in order to compile it as a NEURON module. The main change was mysql/my_list.h
had `#undef LIST` so it wouldn't conflict with NEURON's list type. The C header files for the
modified version are available in this package. You'll also need to compile the API to a lib file and
link to it. The MySQL server must be running when using this interface.

### NEURON mod files:

- MySQL.mod - main interface to MySQL
- vecst.mod - used by NQS

### HOC files:

- mosinit.hoc - demo file
- declist.hoc - used by NQS
- decnqs.hoc - used by NQS
- decvec.hoc - used by NQS
- grvec.hoc - graphics utils.
- drline.hoc - graphics utils.
- mysql_utils.hoc - MySQL interface utilities
- nqs.hoc - NQS
- setup.hoc - setup simulation utils.

### mysql directory:

MySQL API header files for version 5.0.37

For help with compilation/usage, contact samn at neurosim dot downstate dot edu .

The interface has only been tested on Linux machines with version 5.0.37 of MySQL. If you are using
a different version of MySQL , this is not guaranteed to work/compile, and you may need to make
some small changes to get it to compile.

### To build:

Make sure you are in the directory containing mod files, and have the mysql header
files in a subdir named `mysql` (or a symbolic link will be fine).

Then run:

```
nrnivmodl -loadflags "-L/usr/local/src/mysql-5.0.37-linux-x86_64-glibc23/lib -lmysqlclient -lz"
```

- `-L` should have the full path to the mysql lib files (that you already compiled).
- `-lz` is for zlib
- mysql include dir must be in mod subdir (with a link is fine)

Note: build can only be done once MySQL has been built on the system.

### Sample usage

```hoc
load_file("grvec.hoc")
load_file("nqs.hoc")
load_file("mysql_utils.hoc")

Init_mysql("localhost","username","password") //connect
Query_mysql("show databases") //perform a sql query
ListDBs_mysql()
```

See below for more example code and function descriptions.

### MySQL.mod function descriptions:

All functions described below should have `_mysql` suffix added
to them when running from NEURON.

There is one main MYSQL object: `MYSQL g_mysql;`
Since only one connection allowed.

- **Close()**
  Closes any open connection to MySQL server

- **Init()**
  Initialize MySQL engine & connect to MySQL server
  User must supply host-name, user-name, password
  Returns 1.0 if successful
  Usage: `Init(host,user,pass)`

- **SelectDB()**
  Select DB by name
  Returns 1.0 if successful
  Usage: `SelectDB(dbname)`

- **FreeResults()**
  Frees results of Select, responsibility of hoc user

- **NumCols()**
  Check number of columns from previous Select call

- **NumRows()**
  Check number of rows from previous Select call

- **GetRows()**
  Get rows from previous Select
  Into list of vectors (each vector is dimension/column)
  Returns -1.0 on error, otherwise number of rows

- **Find()**
  Takes vector and returns number of times it exists as a row in table_name
  Also allows partial row match on first min(vector.size, table.columns) columns
  Stores results in g_result for later retrieval
  Returns -1.0 on error, otherwise number of rows found matching vector
  Usage: `Find(table_name,Vector)`

- **UpdateCol()**
  Updates a single column of a table.
  Usage: `UpdateCol(table_name,col_name,order_by_column_name,vector_of_values,start_idx)`
  - `col_name`: column that will be updated
  - `order_by_column_name`: column that stores ids
  - `start_idx`: starting value of order by column index; it is incremented for each row of a column.

Example update statements executed:
```
update table set col_name = vec[0] where order_by_column_name=start_idx;
update table set col_name = vec[1] where order_by_column_name=start_idx+1;
...
update table set col_name = vec[n] where order_by_column_name=start_idx+n;
```

- **Insert()**
  Inserts data into existing table
  Usage: `Insert(table_name,list_of_vectors or vector)`
  Vector should have same size as number of columns in table, so Insert will add 1 row for Vector arg.
  If argument is List, it should have number of columns vectors, and vector sizes rows will be inserted into table.
  Returns 1.0 if success.

- **Select()**
  Does a SQL select and keeps results around for hoc user to retrieve.
  Hoc user must free results at a certain point.
  Returns -1.0 on error, otherwise number of rows found.
  **Note:** If you do the select from NEURON with `Select_mysql`, it doesn't display all the rows onto screen. After that you can do `GetRows_mysql` to get the rows or `NumRows_mysql` to see the number of rows returned from the select.

- **GetColNames()**
  Gets column names from last SQL select. Must pass in correct number of `char*`'s which must have sufficient length to store column names.

- **ListDBs()**
  Lists all available databases. Returns -1 if error, otherwise number of databases.

- **Query()**
  Executes a SQL command but doesn't store results for hoc user.
  Can execute any type of SQL command, e.g. create, select, insert, etc.
  Displays results on screen.
  Returns -1.0 on error.
  Usage: `Query(query_string)`

- **VersionInfo()**
  Procedure to display client & server versions.

### Sample hoc code

The following procedure works only if there is a pre-existing database named "test"
To create it do: `Query_mysql("create database test")`

```hoc
objref lv,myv[2],lvres
proc TestInsert(){
  Init_mysql("your_host","your_user_name","your_password")
  SelectDB_mysql("test")
  Query_mysql("create table junk (d1 double,d2 double)")
  lv=new List()
  myv[0]=new Vector(10)
  myv[0].indgen(0,10)
  myv[1]=new Vector(10)
  myv[1].indgen(10,20)
  lv.append(myv[0])
  lv.append(myv[1])
  Insert_mysql("junk",lv)
  Query_mysql("select * from junk")
}
TestInsert()
```

Insert can take a Vector or List of Vectors

```hoc
objref myv
proc TestInsert2(){
  Init_mysql("your_host","your_user_name","your_password")
  Query_mysql("use test")
  Query_mysql("create table jnk (d1 double,d2 double)")
  myv=new Vector(2)
  myv.x(0)=0
  myv.x(1)=1
  Insert_mysql("jnk",myv)
  Query_mysql("select * from jnk")
}
```

There are some hoc utility functions in `mysql_utils.hoc` (don't add `_mysql` to call them):

- `CreateTable()`
  Creates a table in db currently connected to
  - `$s1` = table name
  - `$o2` = list of column names (as String objects or strdefs)
  - `$3` = whether to create index for each column

- `SelectedColNames()`
  Returns List containing column names from last Select call

- `sql2nqs()`
  Converts the results of a SQL select into an NQS db & returns it
  - `$s1` = SQL query

- `nqs2sql()`
  Converts NQS database to SQL format, automatically creates indices
  - `$o1` = NQS
  - `$s2` = name of table in MySQL db
  - `$3` = whether to create column `nqs_row_id` storing original NQS row index
  - `$4` = whether to create MySQL index on each column
  - NB: table must not have 'index' as a col name (MySQL reserved word)

### Example usage:

```hoc
objref ls
objref cols[5]
proc TestCreateTable(){ local ii,makeindex
  ls=new List()
  for ii=0,4{
    cols[ii]=new String()
    sprint(cols[ii].s,"col%d",ii+1)
    ls.append(cols[ii])
  }
  makeindex = 1
  if(CreateTable("hoc_table",ls,makeindex)){
    Query_mysql("show tables")
    Query_mysql("describe hoc_table")
  } else {
    Query_mysql("show tables")
  }
}
TestCreateTable()
```

This will create a table named "hoc_table" in current database with indices
on each column

### References:

1. Data mining of time-domain features from neural extracellular field data
   chapter in book
   *Applications of Computational Intelligence in Bioinformatics and Biomedicine: Current Trends and Open Problems*
   Series: Studies in Computational Intelligence (peer-reviewed), 151:119-140, 2008, Springer.
   S Neymotin, DJ Uhlrich, KA Manning, WW Lytton

2. Neural Query System: Data-mining from within the NEURON simulator.
   *Neuroinformatics*. 2006;4(2):163-76.
   WW Lytton

---

2022-05: Updated MOD files to contain valid C++ and be compatible with the upcoming versions 8.2 and 9.0 of NEURON.

---

2025-05-30: Standardized to Markdown.