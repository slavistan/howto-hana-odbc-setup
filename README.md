# Linux ODBC Setup for SAP HANA

The ODBC standard (Open Database Connectivity) defines an SQL-interface to database systems in order to provide
application developers with a unified and stable communication channel to different backends. Many programming languages
provide ODBC client libraries which allow to develop the client application using the languages and frameworks most
suitable to the task at hand. In order to use an ODBC client database-specific drivers are required.

This guide assumes that valid login credentials to an SAP HANA database instance are available. The configuration herein
will use the following credentials:

| Key      |               |
| ---      | ---           |
| Host     | zion          |
| Port     | 30015         |
| Database | NEB           |
| User     | TANDERSON     |
| Password | WhiteRabbit99 |


## 1. Installation

Install, preferably from source, the [unixODBC](http://www.unixodbc.org/) package containing
+ the Linux ODBC libraries
+ `iusql`, an interactive SQL shell for ODBC connections
+ `odbcinst`, an ODBC configuration tool

Install the [SAP Hana Tools](https://tools.hana.ondemand.com/#hanatools) suite which encompasses
+ `libodbcHDB.so`, the database-specific ODBC driver
+ `hdbuserstore`, a tool to encrypt and store locally your HANA user credentials and connection details
+ `hdbsql`, an interactive HANA SQL shell


## 2. Configuration

### 2.1 Configure login credentials via `hdbuserstore`

Using `hdbuserstore` generate an encrypted entry in your local list of HANA credentials using your available login
details.

```sh
hsbuserstore set NEB_TANDERSON_DEV zion:30015@NEB TANDERSON WhiteRabbit99
```

Display all entries via `hdbuserstore list` and by launching an SQL shell ensure that henceforth you may connect to your
database using the stored credentials.

```sh
hdbsql -U NEB_TANDERSON_DEV
```

### 2.2 Locate ODBC configuration files

Depending on the unixODBC installation method the location of the configuration files may differ. Run `obdcinst -j` to
display your particular setup and should your paths differ from those used herein adjust them correspondingly.

```
unixODBC 2.3.10pre
DRIVERS............: /usr/local/etc/odbcinst.ini
SYSTEM DATA SOURCES: /usr/local/etc/odbc.ini
FILE DATA SOURCES..: /usr/local/etc/ODBCDataSources
USER DATA SOURCES..: /home/neo/.odbc.ini
SQLULEN Size.......: 8
SQLLEN Size........: 8
SQLSETPOSIROW Size.: 8
```

### 2.3 Register the SAP HANA driver

Locate your `libodbcHDB.so` and create an entry in the database-specific drivers configuration file
**/usr/local/etc/odbcinst.ini** like so:

```ini
[SAP HANA]
Driver=/home/neo/bin/sap/hdbclient/libodbcHDB.so
```

This file contains information about the location of the database-specific ODBC drivers on your system and will
reference one file per database architecture you intend to communication with via ODBC. Note that the driver path must
be absolute.


### 2.4 Register the data source

The files **/usr/local/etc/odbc.ini** and **/home/neo/.odbc.ini** store ODBC connection information used to connect to
databases which support ODBC and for which a driver has been made available. If you intend to make an ODBC connection
available to every user on a system modify the former file, for per-user connection management use the latter. Generate
an entry like so:

```ini
[HANA_ZION_NEB_TANDERSON]
DRIVER=SAP HANA
SERVERNODE=@NEB_TANDERSON_DEV
```

In order to establish communication the connection references the ODBC driver in its *DRIVER* field and the login
credentials set via `hdbuserstore` in its *SERVERNODE* field. Note that the value is prefaced by a '@' indicating that
this value is a reference. The label *HANA_ZION_NEB_TANDERSON* under which the configuration is stored is referred to
as the *Data Source Name (DSN)*.

## 2.5 Test & Troubleshooting

Check if your SAP HANA credentials are valid, your encrypted user credentials are functioning and the database is
responsive:

```sh
hdbsql -n zion:30015 -d NEB -u TANDERSON -p WhiteRabbit99
hdbsql -U NEB_TANDERSON_DEV
```

Check the ODBC connection using unixODBC's SQL `iusql -v HANA_ZION_NEB_TANDERSON`. Should this fail create two symlinks
```
/etc/odbc.ini -> /usr/local/etc/odbc.ini
/etc/odbcinst.ini -> /usr/local/etc/ocbcinst.ini
```
which will work around a bug in unixODBC which exists at the time of writing (2020/11/29) and causes unixODBC to search
for the ini-files under /etc/ instead of the custom directory configured prior to compilation via the --sysconfdir
parameter (the author has been informed). Creating the above symlinks will fix all problems related to this bug.

## 3. Usage

Given a properly configured setup the ODBC interface may be used from any application or programming language that
provides an ODBC client implementation. The simplest way to check whether everything is set up correctly is to use the
unixODBC command-line `iusql` as shown above which establishes a connection and will open an interactive SQL shell to
execute SQL queries.

Additionally, many programming languages provide ODBC client libraries which implicitly make use of the same underlying
ODBC configuration.

### 3.1 R Examples

```R
library(odbc)

# Establish connection. The DSN field (data source name) references to your
# ODBC configuration in the 'odbc.ini' file(s).
con <- dbConnect(odbc(), DSN="HANA_ZION_NEB_TANDERSON")

# Send a statement or query
res <- dbSendStatement(con, "SELECT 'Hello World!' FROM dummy;")

# Pull a few rows from the query
data <- dbFetch(res, n=10)
data

# "Clear" a used result
dbClearResult(res)

# Terminate connection
dbDisconnect(con)
```

Based on the previous setup one may develop machine learning applications using Python or R clients for SAP HANA's
Predictive Analysis Library (PAL). The SAP HANA Client tools contain the R package `hana.ml.r` in an archive named
'hdbclient/hana.ml.r_2.5.20062600.tar.gz' (or similar) which can be installed locally via
`install.packages("hdbclient/hana.ml.r_2.5.20062600.tar.gz", repos = NULL, type = "source")`. The package can be used
using the above ODBC connection setup.

```R
library(hana.ml.r)

# Establish a connection using saved credentials. Empty username and password fields are mandatory
con <- hanaml.ConnectionContext(dsn="HANA_ZION_NEB_TANDERSON", username="", password="")

# ...
```
