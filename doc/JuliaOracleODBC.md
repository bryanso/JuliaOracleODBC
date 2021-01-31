# Julia Oracle ODBC

GitHub link: https://github.com/bryanso/JuliaOracleODBC

## Prerequisites

You should have a running Oracle database accessible on your
network.  To access the DB over the network, you need to know:

* the DB hostname
* the DB port
* the DB service name
* a Db user and its password to login

Obviously your firewall rule needs to allow access between
the Julia computer and the Oracle server.  You can test 
the network connection this way:

    nc -vz <DB host> <DB port>

For example, my installation shows:

    $ nc -vz orcl.local 1521
    Connection to orcl.local 1521 port [tcp/*] succeeded!

I assume you have Julia installed as well.  I have tested
with Julia 1.4 and 1.5.

The steps in this document were tested on Oracle Linux 7 and
ElementaryOS 5.  It will almost certainly
work the same way on CentOS and Ubuntu as well.

## Install unixOBDC

Download unixODBC from http://www.unixodbc.org/download.html.
It is a unixODBC-2.3.9.tar.gz file that needs to be compiled.  

    $ cd <download directory>
    $ tar zxf unixODBC-2.3.9.tar.gz
    $ cd unixODBC-2.3.9
    $ ./configure
    $ make
    $ sudo make install

Verify that this has created a /usr/local/etc/ directory with
some default ODBC files

    $ ls -l /usr/local/etc

    drwxr-xr-x 2 root root 4096 Jan 30 18:42 ODBCDataSources/
    -rw-r--r-- 1 root root    0 Jan 30 18:42 odbc.ini
    -rw-r--r-- 1 root root    0 Jan 30 18:42 odbcinst.ini

### Where's the GUI?

Ref: http://www.unixodbc.org/doc/UserManual/

Reading up on unixODBC it was implied there will be some
GUI apps that one can use to configure data sources (DNS). 

They are:

* ODBCConfig
* DataManager

But they are not found after making. I examined the 
source code and didn't find them anywhere.  This is a
mystery to me.  Appreciate any feedback about why
they are gone.

Meanwhile, to connect to Oracle database you don't
need the GUI apps.

## Install Oracle Instant Client

On your computer where you run Julia, download Oracle 
Instant Client from their website:

https://www.oracle.com/database/technologies/instant-client/downloads.html

This doc uses Oracle Instant Client version 21.1 as an example.  But almost any version would work.  The simpliest way to install is to download the Basic Package zip file and unzip to a directory of your choice:

    $ sudo mkdir -p /opt/oracle
    $ cd /opt/oracle
    $ sudo unzip <download directory>/instantclient-basic-linux.x64-21.1.0.0.0.zip

This will create an instantclient_21_1/ subdirectory under /opt/oracle/ and unzip the files there.

## Install Oracle ODBC Package

Ref: https://www.oracle.com/database/technologies/releasenote-odbc-ic.html

On your computer where you run Julia, download Oracle 
ODBC library from the same Oracle Instant Client website.  
It is a simple zip file named instantclient-odbc-linux.x64-21.1.0.0.0.zip

    $ cd /opt/oracle
    $ sudo unzip <download directory/instantclient-odbc-linux.x64-21.1.0.0.0.zip

I find Oracle's instruction a little cryptic:

    Execute odbc_update_ini.sh from the Instant Client directory.

    Usage: odbc_update_ini.sh <ODBCDM_Home> [<Install_Location> <Driver_Name> <DSN> <ODBCINI>]

What they meant by <ODBCDM_Home> is simply the /usr/local directory if you have installed unixODBC as instructed above.

So do:

    $ cd /opt/oracle/instantclient_21_1
    $ sudo ./odbc_update_ini.sh /usr/local

It will give you this message, it's OK:

    *** ODBCINI environment variable not set,defaulting it to HOME directory!

You need to use sudo because they are going to add driver
info to /usr/local/etc/odbcinst.ini.  Previously the file
has zero bytes.  Now it is populated with:

    $ cat /usr/local/etc/odbcinst.ini

    [Oracle 21 ODBC driver]
    Description     = Oracle ODBC driver for Oracle 21
    Driver          = /opt/oracle/instantclient_21_1/libsqora.so.21.1
    Setup           =
    FileUsage       =
    CPTimeout       =
    CPReuse         =

Essentially, that's enough for Julia's ODBC package to
access the database.

Note down the Driver path: /opt/oracle/instantclient_21_1/libsqora.so.21.1

## Set Up tnsnames.ora

Setting up TNS entries makes it easy for your users to 
connect to the database.

The Instant Client installation already provides a TNS directory:

    /opt/oracle/instantclient_21_1/network/admin

Your Oracle database instance should have a tnsnames.ora file
that you can copy from:

    $ORACLE_HOME/network/admin/tnsnames.ora

You may simply "scp" that file and place it in the Instant Client network/admin directory. 

For my example, I will simply create a new tnsnames.ora file instead of copying.  Sometimes it's easier this way!  The following is the TNS entry for my database, using Oracle's EZ Connect format.  It is just one line:

    $ sudo vi /opt/oracle/instantclient_21_1/network/admin/tnsnames.ora
    PDB1 = orcl.local:1521/pdb1

Make it readable by all:

    $ sudo chmod 644 /opt/oracle/instantclient_21_1/network/admin/tnsnames.ora

Ref: https://www.oracle.com/technetwork/database/enterprise-edition/oraclenetservices-neteasyconnect-133058.pdf

An equivalent, fancier format is like so:
    
    PDB1 =
    (DESCRIPTION =
      (ADDRESS = (PROTOCOL = TCP)(HOST = orcl.local)(PORT = 1521))
      (CONNECT_DATA =
        (SERVER = DEDICATED)
        (SERVICE_NAME = pdb1)
      )
    )

## Install Oracle Sql*Plus Package

This is optional.  It is useful for verifying database connection
is working before using Julia.

On your computer where you run Julia, download Oracle Sql*Plus package 
from the same Oracle Instant Client website.  I have used the
simple zip file package:

    $ cd /opt/oracle
    $ sudo unzip <download directory>/instantclient-sqlplus-linux.x64-21.1.0.0.0.zip

Verify the Oracle database connection:

    $ cd /opt/oracle/instantclient_21_1
    $ export LD_LIBRARY_PATH=/opt/oracle/instantclient_21_1
    $ ./sqlplus <user>/<password>@<tns name>

For example, my host is orcl.local and service name is pdb1:

    $ export LD_LIBRARY_PATH=/opt/oracle/instantclient_21_1
    $ export TNS_ADMIN=/opt/oracle/instantclient_21_1/network/admin
    $ /opt/oracle/instantclient_21_1/sqlplus system/secret@pdb1

    SQL*Plus: Release 21.0.0.0.0 - Production on Sat Jan 30 19:02:11 2021
    Version 21.1.0.0.0

    Copyright (c) 1982, 2020, Oracle.  All rights reserved.

    Last Successful login time: Sat Jan 30 2021 19:01:45 -08:00

    Connected to:
    Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
    Version 19.3.0.0.0

    SQL> SELECT username FROM dba_users ORDER BY 1;

    USERNAME
    ----------------------------------------------------
    ANONYMOUS
    APPQOSSYS
    AUDSYS
    CTXSYS
    DBSFWUSER
    DBSNMP
    ...
    36 rows selected.

## Set Up Julia ODBC 

    From now on you will need to set these env variables, so
    you may consider putting them in your startup files:

    $ export LD_LIBRARY_PATH=/opt/oracle/instantclient_21_1
    $ export TNS_ADMIN=/opt/oracle/instantclient_21_1/network/admin
    $ julia

    julia> using Pkg
    julia> Pkg.add("ODBC")
    julia> Pkg.add("DBInterface")
    julia> Pkg.add("DataFrames")

    julia> using ODBC, DBInterface, DataFrames

    julia> ODBC.adddriver("Oracle 21 ODBC driver", "/opt/oracle/instantclient_21_1/libsqora.so.21.1")

In the above the first parameter is the string in the odbcinst.ini file:

    [Oracle 21 ODBC driver]

The second parameter is the driver full path you have noted down 
in the previous step (also in the odbcinst.ini file).

This "adddriver" registration is persistent.  You can exit Julia
and come back in to find it is remembered:

    julia> using ODBC

    julia> ODBC.drivers()
    Dict{String,String} with 2 entries:
      "Oracle 21 ODBC driver" => "Driver=/opt/oracle/instantclient_21_1/libsqora.so#
      "ODBC Drivers"          => ""

Now we will establish a connection and perform a simple query:

    julia> db = ODBC.Connection("Driver={Oracle 21 ODBC driver};Dbq=pdb1;Uid=system;Pwd=secret")

    julia> DBInterface.execute(db, "SELECT username FROM dba_users") |> DataFrame
    36×1 DataFrame
     Row | USERNAME               
         | String                 
    -----+------------------------
       1 | SYS
       2 | SYSTEM
       3 | XS$NULL
       4 | LBACSYS
       5 | OUTLN
       6 | DBSNMP
       7 | APPQOSSYS
       8 | DBSFWUSER
       9 | GGSYS
    ... 

