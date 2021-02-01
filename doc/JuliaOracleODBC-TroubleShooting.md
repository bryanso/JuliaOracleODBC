# Julia Oracle ODBC Troubleshooting

GitHub link: https://github.com/bryanso/JuliaOracleODBC/blob/main/doc/JuliaOracleODBC-TroubleShooting.md

## ERROR: Can't open lib

    julia> db = ODBC.Connection("DSN=OracleODBC-21;Uid=system;Pwd=secret")
    ERROR: 01000: [unixODBC][Driver Manager]Can't open lib '/opt/oracle/instantclient_21_1/libsqora.so.21.1' : file not found

The above error is most likely caused by an unset or incorrectly
set LD_LIBRARY_PATH env variable. Make sure you set:

    $ export LD_LIBRARY_PATH=<Oracle Instant Client Path>

e.g.

    $ export LD_LIBRARY_PATH=/opt/oracle/instantclient_21_1

For csh/tcsh the syntax is:

    > setenv LD_LIBRARY_PATH /opt/oracle/instantclient_21_1

## ERROR: invalid username/password

    julia> db = ODBC.Connection("OracleODBC-21", user="system", password="secret")
    ERROR: 28000: [Oracle][ODBC][Ora]ORA-01017: invalid username/password; logon denied

This error may be caused by forgetting to set TNS_ADMIN env variable.  
See the Set Up tnsnames.ora discussion in https://github.com/bryanso/JuliaOracleODBC/blob/main/doc/JuliaOracleODBC.md.  Try:

    $ export TNS_ADMIN=<Path to tnsnames.ora>

e.g.

    $ export TNS_ADMIN=/opt/oracle/instantclient_21_1/network/admin

For csh/tcsh the syntax is:

    > setenv TNS_ADMIN /opt/oracle/instantclient_21_1/network/admin

## Use Sql*Plus to verify database connection

This was briefly covered in the JuliaOracleODBC.md doc.  
I'll explain a couple of variations.  First the original
information:

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

Another useful way to test the connection more directly is
to use another connection string, the EZ Connect string format:

    sqlplus <user>/<password>@<host>:<port>/<service name>

This will rule out TNS entry issue:

    $ export LD_LIBRARY_PATH=/opt/oracle/instantclient_21_1
    $ /opt/oracle/instantclient_21_1/sqlplus system/secret@orcl.local:1521/pdb1
    SQL*Plus: Release 21.0.0.0.0 - Production on Sat Jan 30 22:48:10 2021
    Version 21.1.0.0.0

    Copyright (c) 1982, 2020, Oracle.  All rights reserved.

    Last Successful login time: Sat Jan 30 2021 22:38:05 -08:00

    Connected to:
    Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
    Version 19.3.0.0.0

    SQL>
    
## Use isql To Verify ODBC

If your first attempt to set up Julia is not successful,
you should rule out Julia as the culprit.  You may use
the Linux isql utility to verify the ODBC setup.  If isql
is not able to connect also, then Julia is not the problem.

    $ sudo apt install unixodbc

The above should install isql:

    $ which isql
    /usr/local/bin/isql

Previously when we ran odbc_update_ini.sh it created a
default data source file in $HOME/.odbc.ini

For my installation I have the following:

    $ cat ~/.odbc.ini
    [OracleODBC-21]
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
    Driver = Oracle 21 ODBC driver
    DSN = OracleODBC-21
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

Notice the name in the first line: [OracleODBC-21].  This is our
DSN (data source name).  Now we can test isql this way:

    $ isql -v OracleODBC-21 system secret
    +---------------------------------------+
    | Connected!                            |
    |                                       |
    | sql-statement                         |
    | help [tablename]                      |
    | quit                                  |
    |                                       |
    +---------------------------------------+

    SQL> SELECT username FROM dba_users
    +-------------------------------------------------+
    | USERNAME                                        |
    +-------------------------------------------------+
    | SYS                                             |
    | SYSTEM                                          |
    | XS$NULL                                         |
    | LBACSYS                                         |
    | OUTLN                                           |
    | DBSNMP                                          |
    | APPQOSSYS                                       |                          
    ...
    | ORDSYS                                          |
    +-------------------------------------------------+
    SQLRowCount returns -1
    36 rows fetched

## Firewall Issue

If your Julia computer cannot "nc" to the Oracle server:

    nc -vz <DB host> <DB port>

then consider temporarily disabling the firewall from the
Oracle server and retest.  Oracle Linux 7 command to do so is:

    $ sudo systemctl stop firewalld

If this resolves your problem, consider re-enabling the
firewall but configure an allow list to allow port 1521
(or whatever your DB port is) to accept incoming traffic.

## MacOS: ERROR: Base.CodePointError{UInt32}

The following error is caused by iODBC in MacOS:

    julia> db = ODBC.Connection("Driver={Oracle 19 ODBC driver};Dbq=pdb1;Uid=system;Pwd=secret")
    ERROR: Base.CodePointError{UInt32}(0x00380030)
    Stacktrace:
     [1] code_point_err(::UInt32) at ./char.jl:86
     [2] Char at ./char.jl:160 [inlined]
     [3] transcode(::Type{UInt8}, ::Array{UInt32,1}) at ./c.jl:286
     [4] transcode at ./c.jl:292 [inlined]
     [5] str(::Array{UInt32,1}, ::Int64) at /Users/bryanso/.julia/packages/ODBC/qhwMX/src/API.jl:58
     [6] diagnostics(::ODBC.API.Handle) at /Users/bryanso/.julia/packages/ODBC/qhwMX/src/API.jl:554
     [7] driverconnect(::String) at /Users/bryanso/.julia/packages/ODBC/qhwMX/src/API.jl:114
     [8] connect at /Users/bryanso/.julia/packages/ODBC/qhwMX/src/API.jl:354 [inlined]
     [9] ODBC.Connection(::String; user::Nothing, password::Nothing, extraauth::Nothing) at /Users/bryanso/.julia/packages/ODBC/qhwMX/src/dbinterface.jl:57
     [10] ODBC.Connection(::String) at /Users/bryanso/.julia/packages/ODBC/qhwMX/src/dbinterface.jl:55
     [11] top-level scope at REPL[2]:1

Before running ODBC.Connection, change to unixODBC by

    julia> ODBC.setunixODBC()

    julia> db = ODBC.Connection("Driver={Oracle 19 ODBC driver};Dbq=pdb1;Uid=system;Pwd=secret")
    ODBC.Connection(DSN=OracleODBC-19;Uid=system;Pwd=secret)
