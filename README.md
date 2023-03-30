
Multicorn2


### MRR HELP

This scenario is specifically designed to work with firebase FHIR database.

## Docker

Im using the supabase/postgres:13.3.0 image because I had trouble finding out how to setup multicorn and postgres so it works.
I think there is some problem with postgres versions higher then version 13 and multicorn2. While trying to use the wrapper the server always crashes without printing
any error message. With the docker image and python 13.3 I could receive error message and fix my issues.

```
# Run docker image for python 13.3
docker run --rm -p 5431:5432 -e POSTGRES_PASSWORD=password supabase/postgres:13.3.0
docker exec -it <dockerid> /bin/bash
```

# Configure docker container
```
# psql --version
psql (PostgreSQL) 13.3 (Debian 13.3-1.pgdg100+1)
```
```
#config docker container
apt-get update
apt-get install wget
apt install vim
apt install unzip

# prepare multicorn install
apt install -y build-essential libreadline-dev zlib1g-dev flex bison libxml2-dev libxslt-dev libssl-dev libxml2-utils xsltproc postgresql-server-dev-13
apt install -y python3-dev python3-setuptools python3-pip

# install python packages or server creation will run into an error
pip3 install sqlalchemy
pip3 install psycopg2

#in case there is an issue selection from foreign table:
pip3 install --force-reinstall 'sqlalchemy<2.0.0'
# -------> https://github.com/dagster-io/dagster/issues/11880
```

## Install multicorn

```
# install multicorn 2.4
wget https://github.com/pgsql-io/multicorn2/archive/refs/tags/v2.4.zip
unzip v2.4.zip
cd multicorn2-2.4/

#Change Makefile (pip3 version)
vim Makefile

...
python_code: setup.py
	pip3 install .
...

make
make install
```

--> already change inside the forked repo

# Change multicorn wrapper to handle jsonb columns

The multicorn data wrapper be default is not able to proper import json columns. While import the json columns multicorn is treating special characters like python does with dictionaries.

Therefore you will receive the error:

```
ERROR: invalid input syntax for type json
DETAIL: Token "=" is invalid.
CONTEXT: JSON data

```

```
vim /usr/local/lib/python3.7/dist-packages/multicorn/sqlalchemyfdw.py
```

```
# Change import statments

...

from sqlalchemy.schema import Table, Column, MetaData
from sqlalchemy.dialects.mssql import base as mssql_dialect
from sqlalchemy.dialects.oracle import base as oracle_dialect
from sqlalchemy.dialects.postgresql.base import (
    ischema_names, PGDialect, NUMERIC, SMALLINT, VARCHAR, TIMESTAMP, BYTEA,
    BOOLEAN, TEXT)
import re
import operator
from sqlalchemy.dialects.postgresql import JSONB
from sqlalchemy.dialects.postgresql import JSON
import json

...

# Change mapping

CONVERSION_MAP = {
    oracle_dialect.NUMBER: basic_converter(NUMERIC),

    mssql_dialect.TINYINT: basic_converter(SMALLINT),
    mssql_dialect.NVARCHAR: basic_converter(VARCHAR),
    mssql_dialect.DATETIME: basic_converter(TIMESTAMP),
    mssql_dialect.VARBINARY: basic_converter(BYTEA),
    mssql_dialect.IMAGE: basic_converter(BYTEA),
    mssql_dialect.BIT: basic_converter(BOOLEAN),
    mssql_dialect.TEXT: length_stripper(TEXT),

    JSON: basic_converter(JSON),
    JSONB: basic_converter(JSONB)

}

...

# Change execution function to handle jsonb

def execute(self, quals, columns, sortkeys=None):
        """
        The quals are turned into an and'ed where clause.
        """
        sortkeys = sortkeys or []
        statement = self._build_statement(quals, columns, sortkeys)
        log_to_postgres(str(statement), DEBUG)
        rs = (self.connection
              .execution_options(stream_results=True)
              .execute(statement))
        # Workaround pymssql "trash old results on new query"
        # behaviour (See issue #100)
        if self.engine.driver == 'pymssql' and self.transaction is not None:
            rs = list(rs)

        for item in rs:
            row = dict(item)
            for column_name in row:
                if isinstance(row[column_name], dict) or isinstance(row[column_name], list):
                    row[column_name] = json.dumps(row[column_name])
            yield row


        for item in rs:
            yield dict(item)


...

```

Most important and added code is:
```
for item in rs:
            row = dict(item)
            for column_name in row:
                if isinstance(row[column_name], dict) or isinstance(row[column_name], list):
                    row[column_name] = json.dumps(row[column_name])
            yield row
```

--> already change inside the forked repo

# Start postgresql configuration 
```
su postgres
psql
```

```
CREATE EXTENSION multicorn;
CREATE SERVER alchemy_srv foreign data wrapper multicorn OPTIONS ( wrapper 'multicorn.sqlalchemyfdw.SqlAlchemyFdw' , db_url 'postgresql://fdw_user:password@ip/db' );
IMPORT FOREIGN SCHEMA public LIMIT TO ( restricted_view ) FROM SERVER alchemy_srv INTO public;
SELECT * FROM restricted_view;
```

# Missing data types

In some cases it might be that the foreign database (source database) contains a user defined data type (like in fhirbase). Therefore we might first need to create that data types.

```
IF NOT EXISTS (
	SELECT 1 FROM pg_type WHERE typname = 'resource_status'
) THEN CREATE TYPE resource_status AS ENUM (
	'created', 'updated', 'deleted', 'recreated'
);

#or

IF NOT EXISTS (
	SELECT 1 FROM pg_type WHERE typname = 'resource_status'
) THEN CREATE TYPE resource_status AS ENUM (
	'created', 'updated', 'deleted', 'recreated'
);
```


====================================================================================================================================================================================

Multicorn Python3 Wrapper for Postgresql Foreign Data Wrapper.  Tested on Linux w/ Python 3.6+ & Postgres 10 thru 14.  Support for Postgres 14 is known to have some issues with predicate pushdown and updating.  We are starting to try and build w/ PG15 and look at what needs improving from PG14.

The Multicorn Foreign Data Wrapper allows you to fetch foreign data in Python in your PostgreSQL server.

Multicorn2 is distributed under the PostgreSQL license. See the LICENSE file for
details.

## Using in OSCG.IO

1.) Install OSCG.IO from the command line, in your home directory, with the curl command at the top of https://oscg.io/usage.html

2.) Change into the oscg directory and install pgXX
```bash
cd oscg
./io install pg13 --start
./io install multicorn2
```
      
3.) Use multicorn as you normally would AND you can install popular FDW's that use multicorn such as ElasticSerachFDW & BigQueryFDW
```bash
      ./io install esfdw
      ./io install bqfdw
```

## Building Multicorn2 against Postgres from Source

It is built the same way all standard postgres extensions are built with following dependcies needing to be installed:

### Install Dependencies for Building Postgres & the Multicorn2 extension
On Debian/Ubuntu systems:
```bash
sudo apt install -y build-essential libreadline-dev zlib1g-dev flex bison libxml2-dev libxslt-dev libssl-dev libxml2-utils xsltproc
sudo apt install -y python3 python3-dev python3-setuptools python3-pip
```

On CentOS/Rocky/Redhat systems:
```bash
sudo yum install -y bison-devel readline-devel libedit-devel zlib-devel openssl-devel bzip2-devel libmxl2 libxslt-devel wget
sudo yum groupinstall -y 'Development Tools'

sudo yum -y install git python3 python3-devel python3-pip
```

### Upgrade to latest PIP (recommended)
```bash
cd ~
wget https://bootstrap.pypa.io/get-pip.py
sudo python3 get-pip.py
rm get-pip.py
```

### Download & Compile Postgres 10+ source code
```bash
cd ~
wget https://ftp.postgresql.org/pub/source/v14.3/postgresql-14.3.tar.gz
tar -xvf postgresql-14.3.tar.gz
cd postgresql-14.3
./configure
make
sudo make install
```

### Download & Compile Multicorn2
```bash
set PATH=/usr/local/pgsql/bin:$PATH
cd ~/postgresql-14.3/contrib
wget https://github.com/pgsql-io/multicorn2/archive/refs/tags/v2.3.tar.gz
tar -xvf v2.3.tar.gz
cd multicorn2-2.3
make
sudo make install
```

### Create Multicorn2 Extension 
In your running instance of Postgres from the PSQL command line
```sql
CREATE EXTENSION multicorn;
```

## Building Multicorn2 against pre-built Postgres

When using a pre-built Postgres installed using your OS package manager, you will need the additional package that enables Postgres extensions to be built:

### Install Dependencies for Building the Multicorn2 extension
On Debian/Ubuntu systems:
```bash
sudo apt install -y build-essential ... postgresql-server-dev-13
sudo apt install -y python3 python3-dev python3-setuptools python3-pip
```

On CentOS/Rocky/Redhat systems:
```bash
sudo yum install -y ... <postgres server dev package>
sudo yum groupinstall -y 'Development Tools'
sudo yum -y install git python3 python3-devel python3-pip
```

### Download & Compile Multicorn2
```bash
wget https://github.com/pgsql-io/multicorn2/archive/refs/tags/v2.3.tar.gz
tar -xvf v2.3.tar.gz
cd multicorn2-2.3
make
sudo make install
```

Note that the last step installs both the extension into Postgres and also the Python part. The latter is done using "pip install", and so can be undone using "pip uninstall".

### Create Multicorn2 Extension 
In your running instance of Postgres from the PSQL command line
```sql
CREATE EXTENSION multicorn;
```
