# Using the ODBC Interface of SAP HANA

The ODBC standard (Open Database Connectivity) defines a SQL-interface to database systems in order to provide
application developers with a unified and stable communication channel to different backends. Many programming languages
provide ODBC client libraries which allow to develop the client application using the languages and frameworks most
suitable to the task at hand.

In order to use an ODBC client database-specific drivers are required.

### Linux-specific Setup (Ubuntu 19.10)

Install the [SAP Hana Tools](https://tools.hana.ondemand.com/#hanatools) suite which, among other things, contains the
database-specific ODBC driver in a file named `libodbcHDB.so`. Afterwards, install the unixODBC package:

```sh
## Using APT ..
sudo apt install unixodbc
sudo apt install unixodbc-dev odbcinst # these may be optional

# .. or, better, install the most recent version from source
# found at unixodbc.org. Dont' forget `./configure --sysconfig=/etc/`
```

As stated in `man unixodbc` ODBC connections are managed in central configuration files, namely `/etc/odbc.ini` and
`$HOME/.odbc.ini`. Edit one of these files to contain the connection information like so:

```ini
[hana_petet]

Driver=/home/stan/bin/sap/hdbclient/libodbcHDB.so
ServerNode=petet:30015
```

The section title *hana_petet* will be used later to refer to the connection defined following it. The *driver* field
shall contain the absolute path of the HANA ODBC driver file mentioned above and depends on where the SAP HANA Tools were
installed to. The *ServerNode* field shall contain the database's hostname and port.

Note that this is the minimal configuration required to use ODBC. More configuration options are listed in the official
documentation or [here](https://db.rstudio.com/best-practices/drivers/) or
[here](https://help.sap.com/viewer/0eec0d68141541d1b07893a39944924e/2.0.03/en-US/66a4169b84b2466892e1af9781049836.html).

#### Associate an ODBC DSN with credentials set up via `hdbuserstore`

`hdbuserstore` may be used to store one's credentials locally (encrypted) and to refer to them via a label.
`hdbuserstore SET PETET_JHN petet:30015@JHN PFISCHER MyPassword` will set up a record under the label of `PETET_JHN`
referring to its respective credentials. Henceforth any connection may be established via the label, e.g.
`hdbsql -U PETET_JHN ...` is equivalent to the verbose `hdbsql -u PFISCHER -p MyPassword -n petet:30015 -d JHN ...`.

In order to associate an ODBC DSN entry in `/etc/odbc.ini` with a label referecing credentials set up with
`hdbuserstore` one needs to prefix the label with an '@' and declare it the value to the *SERVERNODE* parameter of a
DSN section. Using the above example `PETET_JHN`:

```ini
[JHN_SECURE]

Driver=/home/stan/bin/sap/hdbclient/libodbcHDB.so
ServerNode=@PETET_JHN
```

This allows every application using ODBC to connect to the database using the stored credentials without the need to
explicitly put one's password into source code.

**Troubleshooting** 

+ Check the availability of the data base and correctness of the credentials via `hdbsql`.
+ Check the odbc-connectivity via `iusql -v <DSN> <USER> <PWD>`.
  + In case of the error *Data source name not found ...* it may be the case that unixODBC searches the
    /usr/local/etc/ directory for the .ini files, in contrast to what's stated in its manual. Check via
    `odbcinst -j`. This may be corrected at compile time using `./configure --sysconfig=/etc/` as explained
    [here](http://www.unixodbc.org/download.html).

### Usage examples

Given a properly configured setup the ODBC interface may be used from any application or programming language that
provides a ODBC client implementation. The simplest way to check whether everything is set up correctly is to use the
unixODBC command-line `iusql`. Given the above configuration `iusql hana_petet <USER> <PASSWORD>` will establish a
connection and open an interactive SQL shell which may execute any SQL query, e.g. `SELECT 'FOO' from dummy;`.

As stated above, many programming languages provide ODBC client libraries which implicitly make use of the same ODBC
configuration above. Here's an exemple for R.

```R
library(odbc)

# Establish connection. Note that the DSN field (data source name) refers to the
# system's ODBC configuration in the 'odbc.ini' file(s) mentioned above.
con <- dbConnect(odbc(),
                 DSN="hana_petet",
                 UID="PFISCHER",
                 PWD="MyPassword")

# Alternatively, establish the same connection using the stored credentials
# con <- dbConnect(odbc(), DSN="JHN_SECURE")

# Send a statement or query
res <- dbSendStatement(con, "CALL procfoo")

# Pull a few rows from the query
data <- dbFetch(res, n=10)
data

# "Clear" a used result
dbClearResult(res)

# Terminate connection
dbDisconnect(con)
```

### Arbitrary Statements and Limitations to ODBC

The scope of ODBC is limited and does not support arbitrary constructs [TODO: Is that true? What's the true extent of
ODBC?]. Some clients support dispatching arbitrary statements without semantic guarantees or limitations, however.

In order to use `iusql` to dispatch HANA SQL-Script as if using SAP's `hdbsql` empty lines have to be removed from the
SQL-file (e.g. `grep -v '^\s*$' | iusql -b`). An exemplary usage is provided in `man iusql`.

# SAP HANA R Client API for Machine Learning

Based on the previous setup one may develop machine learning applications using Python or R clients for SAP HANA. The
SAP HANA Client tools contain the R package `hana.ml.r` in an archive named 'hdbclient/hana.ml.r_2.5.20062600.tar.gz'
which can be installed locally via
`install.packages("hdbclient/hana.ml.r_2.5.20062600.tar.gz", repos = NULL, type = "source")`.

```R
# Establish a connection using saved credentials
con  <- hanaml.ConnectionContext(dsn = 'JHN_SECURE', username = '', password = '')
```
