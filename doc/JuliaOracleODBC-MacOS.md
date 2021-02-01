# Julia Oracle ODBC on MacOS

GitHub link: https://github.com/bryanso/JuliaOracleODBC

Troubleshooting Guide: https://github.com/bryanso/JuliaOracleODBC/blob/main/doc/JuliaOracleODBC-TroubleShooting.md

This instruction has been tested on MacOS 10.15.7 Catalina.


## Prerequisites

You should have a running Oracle database accessible on your
network.  To access the DB over the network, you need to know:

* the DB hostname
* the DB port
* the DB service name
* a Db user and its password to login

Examples used in this document:

|Info         |Example Value| 
|-------------|-------------|
|hostname     | orcl.local  |
|port         | 1521        |
|service name | pdb1        |
|user         | system      |
|password     | secret      |

Obviously your firewall rule needs to allow access between
the Julia computer and the Oracle server.  You can test 
the network connection this way (MacOS Terminal supports
this command):

    nc -vz <DB host> <DB port>

For example, my installation shows:

    $ nc -vz orcl.local 1521
    Connection to orcl.local port 1521 [tcp/ncube-lm] succeeded!

I assume you have Julia installed as well.  I have tested
with Julia 1.4 and 1.5.

Your MacOS needs to have the development tools to compile
binaries:

1. Install XCode from the Mac App Store

2. Install the command line tools by type this in Terminal:

    xcode-select --install 

## Install unixOBDC

unixODBC also works on MacOS but it needs to be compiled.

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

    drwxr-xr-x   2 root     admin    64 Jan 31 15:08 ODBCDataSources/
    -rw-r--r--   1 root     admin     0 Jan 31 15:08 odbc.ini
    -rw-r--r--   1 root     admin     0 Jan 31 15:08 odbcinst.ini

## Install Oracle Instant Client

On your Mac, download Oracle Instant Client for MacOS (Intel x86) from their website:

https://www.oracle.com/database/technologies/instant-client/downloads.html

This doc uses Oracle Instant Client version 19.8 as an example.  But almost any version would work.  The simpliest way to install is to download the Basic Package zip file and unzip to a directory of your choice.  Alternatively download the Basic Package dmg file which will expand to a folder containing the same files.

    $ sudo mkdir -p /opt/oracle
    $ cd /opt/oracle
    $ sudo unzip <download directory>/instantclient-basic-macos.x64-19.8.0.0.0dbru.zip

This will create an instantclient_19_8/ subdirectory under /opt/oracle/ and unzip the files there.

## Install Oracle ODBC Package

Ref: https://www.oracle.com/database/technologies/releasenote-odbc-ic.html

Download Oracle ODBC library from the same Oracle Instant Client website.  
It is a simple zip file named instantclient-odbc-macos.x64-19.8.0.0.0dbru.zip.
Again, using the DMG file gives you the same content.

    $ cd /opt/oracle
    $ sudo unzip <download directory/instantclient-odbc-macos.x64-19.8.0.0.0dbru.zip

I find Oracle's instruction a little cryptic:

    Execute odbc_update_ini.sh from the Instant Client directory.

    Usage: odbc_update_ini.sh <ODBCDM_Home> [<Install_Location> <Driver_Name> <DSN> <ODBCINI>]

What they meant by <ODBCDM_Home> is simply the /usr/local directory if you have installed unixODBC as instructed above.

So do:

    $ cd /opt/oracle/instantclient_19_8
    $ sudo ./odbc_update_ini.sh /usr/local

It will give you this message, it's OK:

    *** ODBCINI environment variable not set,defaulting it to HOME directory!

You need to use sudo because they are going to add driver
info to /usr/local/etc/odbcinst.ini.  Previously the file
has zero bytes.  Now it is populated with:

    $ cat /usr/local/etc/odbcinst.ini

    [Oracle 19 ODBC driver]
    Description     = Oracle ODBC driver for Oracle 19
    Driver          = /opt/oracle/instantclient_19_8/libsqora.dylib.19.1
    Setup           =
    FileUsage       =
    CPTimeout       =
    CPReuse         = 

Essentially, that's enough for Julia's ODBC package to
access the database.

Note down the Driver path: /opt/oracle/instantclient_19_8/libsqora.dylib.19.1

## Set Up tnsnames.ora

Setting up TNS entries makes it easy for your users to 
connect to the database.

The Instant Client installation already provides a TNS directory:

    /opt/oracle/instantclient_19_8/network/admin

Your Oracle database instance should have a tnsnames.ora file
that you can copy from:

    $ORACLE_HOME/network/admin/tnsnames.ora

You may simply "scp" that file and place it in the Instant Client network/admin directory. 

For my example, I will simply create a new tnsnames.ora file instead of copying.  Sometimes it's easier this way!  The following is the TNS entry for my database, using Oracle's EZ Connect format.  It is just one line:

    $ sudo vi /opt/oracle/instantclient_19_8/network/admin/tnsnames.ora
    PDB1 = orcl.local:1521/pdb1

Make it readable by all:

    $ sudo chmod 644 /opt/oracle/instantclient_19_8/network/admin/tnsnames.ora

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

## Set Up Julia ODBC 

    From now on you will need to set these env variables, so
    you may consider putting them in your startup files:

    $ export LD_LIBRARY_PATH=/opt/oracle/instantclient_19_8
    $ export TNS_ADMIN=/opt/oracle/instantclient_19_8/network/admin
    $ julia

    julia> using Pkg
    julia> Pkg.add("ODBC")
    julia> Pkg.add("DBInterface")
    julia> Pkg.add("DataFrames")

    julia> using ODBC, DBInterface, DataFrames

    julia> ODBC.adddriver("Oracle 19 ODBC driver", "/opt/oracle/instantclient_19_8/libsqora.dylib.19.1")

In the above the first parameter is the string in the odbcinst.ini file:

    [Oracle 19 ODBC driver]

The second parameter is the driver full path you have noted down 
in the previous step (also in the odbcinst.ini file).

This "adddriver" registration is persistent.  You can exit Julia
and come back in to find it is remembered:

    julia> using ODBC

    julia> ODBC.drivers()
    Dict{String,String} with 1 entry:
      "Oracle 19 ODBC driver" => "Installed"

## Using Julia ODBC

Now we will establish a connection and perform a simple query:

    julia> using ODBC, DBInterface, DataFrames

The following is very important because by default Julia on MacOS
uses iODBC but we have chosen to use unixODBC:

    julia> ODBC.setunixODBC()

    julia> db = ODBC.Connection("Driver={Oracle 19 ODBC driver};Dbq=pdb1;Uid=system;Pwd=secret")

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

## Using DSN

Another common way to access the database is to set up DSNs (data source names).  Previously when we ran odbc_update_ini.sh it created a default data source file in $HOME/.odbc.ini

For my installation I have the following:

    $ cat ~/.odbc.ini
    [OracleODBC-19]
    AggregateSQLType = FLOAT
    Application Attributes = T
    Attributes = W
    BatchAutocommitMode = IfAllSuccessful
    BindAsFLOAT = F
    CacheBufferSize = 20
    CloseCursor = F
    DisableDPM = F
    DisableMTS = T
    DisableRULEHint = T
    Driver = Oracle 19 ODBC driver
    DSN = OracleODBC-19
    EXECSchemaOpt =
    EXECSyntax = T
    Failover = T
    FailoverDelay = 10
    FailoverRetryCount = 10
    FetchBufferSize = 64000
    ForceWCHAR = F
    LobPrefetchSize = 8192
    Lobs = T
    Longs = T
    MaxLargeData = 0
    MaxTokenSize = 8192
    MetadataIdDefault = F
    QueryTimeout = T
    ResultSets = T
    ServerName = 
    SQLGetData extensions = F
    SQLTranslateErrors = F
    StatementCache = F
    Translation DLL =
    Translation Option = 0
    UseOCIDescribeAny = F
    UserID = 

It's almost good to go.  You only need to update

    ServerName =

to the TNS name, for example:

    ServerName = PDB1

You may also want to change the file ownership (since it was created by sudo, not you):

    $ sudo chown $USER:$GROUP ~/.odbc.ini

Notice the name in the first line: [OracleODBC-19].  This is our
DSN (data source name). We need to copy all of the above into a special 
file -- Julia's ODBC "ini" file.

Julia's ODBC keeps DSN names in a special location.  First find it by:

    $ julia
    
    julia> using ODBC, DBInterface, DataFrames

    julia> realpath(joinpath(dirname(pathof(ODBC)), "../config/odbc.ini"))
    "/Users/bryanso/.julia/packages/ODBC/qhwMX/config/odbc.ini"
    
My location is /Users/bryanso/.julia/packages/ODBC/qhwMX/config/odbc.ini

Now edit the above file

    $ vi /Users/bryanso/.julia/packages/ODBC/qhwMX/config/odbc.ini

and copy all the info from $HOME/.odbc.ini to the bottom of that file.

Now we can connect using DSN:

    $ export LD_LIBRARY_PATH=/opt/oracle/instantclient_19_8
    $ export TNS_ADMIN=/opt/oracle/instantclient_19_8/network/admin
    $ julia
    julia> using ODBC, DBInterface, DataFrames

    julia> ODBC.setunixODBC()

The following shows the DSN entries from /Users/bryanso/.julia/packages/ODBC/qhwMX/config/odbc.ini
 
    julia> ODBC.dsns()
    Dict{String,String} with 2 entries:
      "OracleODBC-19" => "Oracle 19 ODBC driver"
      "ODBC"          => ""

    julia> db = ODBC.Connection("DSN=OracleODBC-19;Uid=system;Pwd=secret")
    ODBC.Connection(DSN=OracleODBC-19;Uid=system;Pwd=secret)

    julia> DBInterface.execute(db, "SELECT table_name from dba_tables") |> DataFrame
    2177×1 DataFrame
      Row | TABLE_NAME                 
          | String                     
    ------+----------------------------
        1 | IND$
        2 | CLU$
        3 | ICOL$
        4 | COL$
        5 | TAB$
        6 | LOB$
        7 | COLTYPE$
    ...

## Troubleshooting Guide

https://github.com/bryanso/JuliaOracleODBC/blob/main/doc/JuliaOracleODBC-TroubleShooting.md
