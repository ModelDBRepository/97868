 $Id: readme.txt,v 1.2 2008/10/06 16:58:43 samn Exp $ 

Table of contents:

  1. SPUD feature extraction algorithm
  2. NEURON interface to MySQL




1. SPUD

SPUD feature extraction algorithm implementation as appearing in [1](see bottom of readme)

spud.mod - main implementation
spud.hoc - hoc utilities for ease of use
mosinit.hoc - sets up a GUI showing figure 2 from chapter - run this to see SPUD in action
rat_strobe_1.vec - single trace of electrocorticographic recordings from rat

//sample routine to demonstrate SPUD feature extraction algorithm (in spud.hoc)
//to use: testspud(vector,num_threshold_slices,log_spacing,[user-specified-thresholds])
//on return, output will store the extracted "bumps" as an NQS database
//$o1 = input data vector
//$2 = num threshold lines
//$3 = threshold spacing, 0=linear,1=log (optional)
//$o4 = user-specified thresholds to pass in to SPUD (optional)
proc testspud()




2. NEURON interface to MySQL readme - S Neymotin , WW Lytton - 4/2007

(for questions/comments contact samn at neurosim dot downstate dot edu)

This is an interface that allows access to a MySQL server from directly
within the NEURON simulation environment [1]. It allows for performing SQL
queries and returning the results into NEURON data structures. It also
works with the Neural Query System (NQS)[2] and can convert between NQS databases
and MySQL tables.

The MySQL C API is required. The version used was mysql-5.0.37. Some modifications
were made to it in order to compile it as a NEURON module. The main change was mysql/my_list.h
had #undef LIST so it wouldn't conflict with NEURON's list type. The C header files for the
modified version are available in this package. You'll also need to compile the API to a lib file and
link to it. The MySQL server must be running when using this interface.


NEURON mod files: 

  MySQL.mod - main interface to MySQL
  vecst.mod - used by NQS

HOC files:

  mosinit.hoc - demo file
  declist.hoc - used by NQS
  decnqs.hoc - used by NQS
  decvec.hoc - used by NQS
  grvec.hoc - graphics utils.
  drline.hoc - graphics utils.
  mysql_utils.hoc - MySQL interface utilities
  nqs.hoc - NQS
  setup.hoc - setup simulation utils.


mysql directory: MySQL API header files for version 5.0.37

For help with compilation/usage, contact samn at neurosim dot downstate dot edu . 

The interface has only been tested on Linux machines with version 5.0.37 of MySQL. If you are using
a different version of MySQL , this is not guaranteed to work/compile, and you may need to make
some small changes to get it to compile.

* to build:

make sure you are in the directory containing mod files, and have the mysql header
files in a subdir named mysql (or a symbolic link will be fine).

then:

nrnivmodl -loadflags "-L/usr/local/src/mysql-5.0.37-linux-x86_64-glibc23/lib -lmysqlclient -lz"

-L should have the full path to the mysql lib files (that you already compiled).

-lz is for zlib

mysql include dir must be in mod subdir (with a link is fine)

note: build can only be done once MySQL has been built on the system.


* sample usage

load_file("grvec.hoc")
load_file("nqs.hoc")
load_file("mysql_utils.hoc")

Init_mysql("localhost","username","password") //connect
Query_mysql("show databases") //perform a sql query
ListDBs_mysql()

see below for more example code and function descriptions

* MySQL.mod function descriptions:

all functions described below should have _mysql suffix added
to them when running from NEURON.

there is one main MYSQL object : MYSQL g_mysql;
since only one connection allowed

: closes any open connection to MySQL server
FUNCTION Close()


: initialize MySQL engine & connect to MySQL server
: user must supply host-name , user-name, password
: to connect to MySQL server
: returns 1.0 iff successful
: Init(host,user,pass)
FUNCTION Init()


: SelectDB(dbname)
: returns 1.0 iff successful
FUNCTION SelectDB()


: frees results of Select, responsibility of hoc user
FUNCTION FreeResults()


: check # of columns from previous Select call
FUNCTION NumCols()


: check # of rows from previous Select call
FUNCTION NumRows()


: get rows from previous Select
: into list of vectors (each vec is dimension/column)
: returns -1.0 on error, otherwise number of rows
FUNCTION GetRows()


: takes vector and returns # of times it exists as a row in table_name
: Find(table_name,Vector) also allows partial row match on first 
: min(vector.size,table.columns) columns stores results in g_result for later
: retrieval returns -1.0 on error, otherwise num_rows found matching vector
FUNCTION Find()



: updates a single col of
: a table. 
: UpdateCol(table_name,col_name,order_by_column_name,vector_of_values,start_idx)
: col_name is the column that will be updated
: order_by_column_name is the column that stores
: ids, if no column stores ids, updating a column
: does not make much sense because the storage
: order of column values may not be what the
: user is expecting. start_idx is starting value
: of order by column index. it is incremented
: for each row of a column.
: so it will be 
: update table set col_name = vec[0] where order_by_column_name=start_idx;
: update table set col_name = vec[1] where order_by_column_name=start_idx+1;
:   ...
: update table set col_name = vec[n] where order_by_column_name=start_idx+n;
FUNCTION UpdateCol()



: inserts data into existing table
: Insert(table_name,list_of_vectors or vector)
: Vector should have same size as # of columns in table , so Insert
: will add 1 row for Vector arg if arg is List, it should have
: num_cols vectors and vectors.size rows will be inserted into table
: returns 1.0 iff success
FUNCTION Insert()


: does a sql select and keeps results around for hoc user to retrieve
: hoc user must free results at a certain point
: returns -1.0 on error, otherwise # of rows found
FUNCTION Select()
** if you do the select from NEURON with Select_mysql, it doesn't display
all the rows onto screen. after that you can do GetRows_mysql to
get the rows or NumRows_mysql to see the # of rows returned from the
select



: gets col names from last sql select must pass in correct # of
: char* 's and they must have sufficient length to store col names
FUNCTION GetColNames()


: lists all available dbs
: returns -1 iff error, otherwise # of dbs
FUNCTION ListDBs()


: executes a sql command but doesnt store results for hoc user
: can execute any type of SQL command, i.e. create,select,insert,etc.
: displays results on screen
: returns -1.0 on error
: Query(query_string)
FUNCTION Query()


: display client & server versions
PROCEDURE VersionInfo()


* sample hoc code


the following proc works only if there is
a pre-existing database named "test"
to create it do: Query_mysql("create database test")

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


Insert can take a Vector or List of Vectors

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

there are some hoc utility functions in mysql_utils.hoc (don't add _mysql to call them):

//creates a table in db currently connected to
//$s1 = table name
//$o2 = list of column names (as String objects or strdefs)
//$3 = whether to create index for each col
func CreateTable () 

// SelectedColNames()
// returns List containing column names from last Select call
obfunc SelectedColNames()

//converts the results of a sql select
//into an nqs db & returns it
//$s1 = sql query
obfunc sql2nqs ()

//converts nqs database to sql format automatically creates indices
//$o1 = nqs
//$s2 = name of table in mysql db
//$3 = whether to create column nqs_row_id storing orig nqs row index
//$4 = whether to create mysql index on each col
// NB: table must not have 'index' as a col name: mysql reserved word
func nqs2sql () 


example usage:

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

this will create a table named "hoc_table" in current database with indices
on each column


references:

(1) Data mining of time-domain features from neural extracellular field data
    chapter in book
    Applications of Computational Intelligence in Bioinformatics and Biomedicine:
    Current Trends and Open Problems
    Series: Studies in Computational Intelligence (peer-reviewed), 151:119-140, 2008, Springer. 
    S Neymotin, DJ Uhlrich, KA Manning, WW Lytton

(2) Neural Query System: Data-mining from within the NEURON simulator.
    Neuroinformatics. 2006;4(2):163-76.
    WW Lytton

Changelog
---------
2022-05: Updated MOD files to contain valid C++ and be compatible with the
         upcoming versions 8.2 and 9.0 of NEURON.
