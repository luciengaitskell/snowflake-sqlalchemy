# Snowflake SQLAlchemy

[![Build and Test](https://github.com/snowflakedb/snowflake-sqlalchemy/actions/workflows/build_test.yml/badge.svg)](https://github.com/snowflakedb/snowflake-sqlalchemy/actions/workflows/build_test.yml)
[![codecov](https://codecov.io/gh/snowflakedb/snowflake-sqlalchemy/branch/main/graph/badge.svg)](https://codecov.io/gh/snowflakedb/snowflake-sqlalchemy)
[![PyPi](https://img.shields.io/pypi/v/snowflake-sqlalchemy.svg)](https://pypi.python.org/pypi/snowflake-sqlalchemy/)
[![License Apache-2.0](https://img.shields.io/:license-Apache%202-brightgreen.svg)](http://www.apache.org/licenses/LICENSE-2.0.txt)
[![Codestyle Black](https://img.shields.io/badge/code%20style-black-000000.svg)](https://github.com/psf/black)

Snowflake SQLAlchemy runs on the top of the Snowflake Connector for Python as a [dialect](http://docs.sqlalchemy.org/en/latest/dialects/) to bridge a Snowflake database and SQLAlchemy applications.

## Prerequisites

### Snowflake Connector for Python

The only requirement for Snowflake SQLAlchemy is the Snowflake Connector for Python; however, the connector does not need to be installed because installing Snowflake SQLAlchemy automatically installs the connector.

### Data Analytics and Web Application Frameworks (Optional)

Snowflake SQLAlchemy can be used with [Pandas](http://pandas.pydata.org/), [Jupyter](http://jupyter.org/) and [Pyramid](http://www.pylonsproject.org/), which provide higher levels of application frameworks for data analytics and web applications. However, building a working environment from scratch is not a trivial task, particularly for novice users. Installing the frameworks requires C compilers and tools, and choosing the right tools and versions is a hurdle that might deter users from using Python applications.

An easier way to build an environment is through [Anaconda](https://www.continuum.io/why-anaconda), which provides a complete, precompiled technology stack for all users, including non-Python experts such as data analysts and students. For Anaconda installation instructions, see the [Anaconda install documentation](https://docs.continuum.io/anaconda/install). The Snowflake SQLAlchemy package can then be installed on top of Anaconda using [pip](https://pypi.python.org/pypi/pip).

## Installing Snowflake SQLAlchemy

The Snowflake SQLAlchemy package can be installed from the public PyPI repository using `pip`:

```shell
pip install --upgrade snowflake-sqlalchemy
```

`pip` automatically installs all required modules, including the Snowflake Connector for Python.

## Verifying Your Installation

1. Create a file (e.g. `validate.py`) that contains the following Python sample code,
   which connects to Snowflake and displays the Snowflake version:

    ```python
    from sqlalchemy import create_engine

    engine = create_engine(
        'snowflake://{user}:{password}@{account}/'.format(
            user='<your_user_login_name>',
            password='<your_password>',
            account='<your_account_name>',
        )
    )
    try:
        connection = engine.connect()
        results = connection.execute('select current_version()').fetchone()
        print(results[0])
    finally:
        connection.close()
        engine.dispose()
    ```

2. Replace `<your_user_login_name>`, `<your_password>`, and `<your_account_name>` with the appropriate values for your Snowflake account and user.

    For more details, see [Connection Parameters](#connection-parameters).

3. Execute the sample code. For example, if you created a file named `validate.py`:

    ```shell
    python validate.py
    ```

    The Snowflake version (e.g. `1.48.0`) should be displayed.

## Parameters and Behavior

As much as possible, Snowflake SQLAlchemy provides compatible functionality for SQLAlchemy applications. For information on using SQLAlchemy, see the [SQLAlchemy documentation](http://docs.sqlalchemy.org/en/latest/).

However, Snowflake SQLAlchemy also provides Snowflake-specific parameters and behavior, which are described in the following sections.

### Connection Parameters

Snowflake SQLAlchemy uses the following syntax for the connection string used to connect to Snowflake and initiate a session:

```python
'snowflake://<user_login_name>:<password>@<account_name>'
```

Where:

- `<user_login_name>` is the login name for your Snowflake user.
- `<password>` is the password for your Snowflake user.
- `<account_name>` is the name of your Snowflake account.

Include the region in the `<account_name>` if applicable, more info is available [here](https://docs.snowflake.com/en/user-guide/connecting.html#your-snowflake-account-name).

You can optionally specify the initial database and schema for the Snowflake session by including them at the end of the connection string, separated by `/`. You can also specify the initial warehouse and role for the session as a parameter string at the end of the connection string:

```python
'snowflake://<user_login_name>:<password>@<account_name>/<database_name>/<schema_name>?warehouse=<warehouse_name>&role=<role_name>'
```

**Note**:
After login, the initial database, schema, warehouse and role specified in the connection string can always be changed for the session.

The following example calls the `create_engine` method with the user name `testuser1`, password `0123456`, account name `abc123`, database `testdb`, schema `public`, warehouse `testwh`, and role `myrole`:

```python
from sqlalchemy import create_engine
engine = create_engine(
    'snowflake://testuser1:0123456@abc123/testdb/public?warehouse=testwh&role=myrole'
)
```

Other parameters, such as `timezone`, can also be specified as a URI parameter or in `connect_args` parameters. For example:

```python
from sqlalchemy import create_engine
engine = create_engine(
    'snowflake://testuser1:0123456@abc123/testdb/public?warehouse=testwh&role=myrole',
    connect_args={
        'timezone': 'America/Los_Angeles',
    }
)
```

For convenience, you can use the `snowflake.sqlalchemy.URL` method to construct the connection string and connect to the database. The following example constructs the same connection string from the previous example:

```python
from snowflake.sqlalchemy import URL
from sqlalchemy import create_engine

engine = create_engine(URL(
    account = 'abc123',
    user = 'testuser1',
    password = '0123456',
    database = 'testdb',
    schema = 'public',
    warehouse = 'testwh',
    role='myrole',
    timezone = 'America/Los_Angeles',
))
```

#### Using a proxy server

Use the supported environment variables, `HTTPS_PROXY`, `HTTP_PROXY` and `NO_PROXY` to configure a proxy server.

### Opening and Closing Connection

Open a connection by executing `engine.connect()`; avoid using `engine.execute()`. Make certain to close the connection by executing `connection.close()` before
`engine.dispose()`; otherwise, the Python Garbage collector removes the resources required to communicate with Snowflake, preventing the Python connector from closing the session properly.

```python
# Avoid this.
engine = create_engine(...)
engine.execute(<SQL>)
engine.dispose()

# Do this.
engine = create_engine(...)
connection = engine.connect()
try:
    connection.execute(<SQL>)
finally:
    connection.close()
    engine.dispose()
```

### Auto-increment Behavior

Auto-incrementing a value requires the `Sequence` object. Include the `Sequence` object in the primary key column to automatically increment the value as each new record is inserted. For example:

```python
t = Table('mytable', metadata,
    Column('id', Integer, Sequence('id_seq'), primary_key=True),
    Column(...), ...
)
```

### Object Name Case Handling

Snowflake stores all case-insensitive object names in uppercase text. In contrast, SQLAlchemy considers all lowercase object names to be case-insensitive. Snowflake SQLAlchemy converts the object name case during schema-level communication, i.e. during table and index reflection. If you use uppercase object names, SQLAlchemy assumes they are case-sensitive and encloses the names with quotes. This behavior will cause mismatches agaisnt data dictionary data received from Snowflake, so unless identifier names have been truly created as case sensitive using quotes, e.g., `"TestDb"`, all lowercase names should be used on the SQLAlchemy side.

### Index Support

Snowflake does not utilize indexes, so neither does Snowflake SQLAlchemy.

### Numpy Data Type Support

Snowflake SQLAlchemy supports binding and fetching `NumPy` data types. Binding is always supported. To enable fetching `NumPy` data types, add `numpy=True` to the connection parameters.

The following example shows the round trip of `numpy.datetime64` data:

```python
import numpy as np
import pandas as pd
engine = create_engine(URL(
    account = 'abc123',
    user = 'testuser1',
    password = 'pass',
    database = 'db',
    schema = 'public',
    warehouse = 'testwh',
    role='myrole',
    numpy=True,
))

specific_date = np.datetime64('2016-03-04T12:03:05.123456789Z')

connection = engine.connect()
connection.execute(
    "CREATE OR REPLACE TABLE ts_tbl(c1 TIMESTAMP_NTZ)")
connection.execute(
    "INSERT INTO ts_tbl(c1) values(%s)", (specific_date,)
)
df = pd.read_sql_query("SELECT * FROM ts_tbl", engine)
assert df.c1.values[0] == specific_date
```

The following `NumPy` data types are supported:

- numpy.int64
- numpy.float64
- numpy.datatime64

### Cache Column Metadata

SQLAlchemy provides [the runtime inspection API](http://docs.sqlalchemy.org/en/latest/core/inspection.html) to get the runtime information about the various objects. One of the common use case is get all tables and their column metadata in a schema in order to construct a schema catalog. For example, [alembic](http://alembic.zzzcomputing.com/) on top of SQLAlchemy manages database schema migrations. A pseudo code flow is as follows:

```python
inspector = inspect(engine)
schema = inspector.default_schema_name
for table_name in inspector.get_table_names(schema):
    column_metadata = inspector.get_columns(table_name, schema)
    primary_keys = inspector.get_pk_constraint(table_name, schema)
    foreign_keys = inspector.get_foreign_keys(table_name, schema)
    ...
```

In this flow, a potential problem is it may take quite a while as queries run on each table. The results are cached but getting column metadata is expensive.

To mitigate the problem, Snowflake SQLAlchemy takes a flag `cache_column_metadata=True` such that all of column metadata for all tables are cached when `get_table_names` is called and the rest of `get_columns`, `get_primary_keys` and `get_foreign_keys` can take advantage of the cache.

```python
engine = create_engine(URL(
    account = 'abc123',
    user = 'testuser1',
    password = 'pass',
    database = 'db',
    schema = 'public',
    warehouse = 'testwh',
    role='myrole',
    cache_column_metadata=True,
))
```

Note that this flag has been deprecated, as our caching now uses the built-in SQLAlchemy reflection cache, the flag has been removed, but caching has been improved and if possible extra data will be fetched and cached.

### VARIANT, ARRAY and OBJECT Support

Snowflake SQLAlchemy supports fetching `VARIANT`, `ARRAY` and `OBJECT` data types. All types are converted into `str` in Python so that you can convert them to native data types using `json.loads`.

This example shows how to create a table including `VARIANT`, `ARRAY`, and `OBJECT` data type columns.

```python
from snowflake.sqlalchemy import (VARIANT, ARRAY, OBJECT)

t = Table('my_semi_strucutred_datatype_table', metadata,
    Column('va', VARIANT),
    Column('ob', OBJECT),
    Column('ar', ARRAY))
metdata.create_all(engine)
```

In order to retrieve `VARIANT`, `ARRAY`, and `OBJECT` data type columns and convert them to the native Python data types, fetch data and call the `json.loads` method as follows:

```python
import json
connection = engine.connect()
results = connection.execute(select([t])
row = results.fetchone()
data_variant = json.loads(row[0])
data_object  = json.loads(row[1])
data_array   = json.loads(row[2])
```

### CLUSTER BY Support

Snowflake SQLAchemy supports the `CLUSTER BY` parameter for tables. For information about the parameter, see :doc:`/sql-reference/sql/create-table`.

This example shows how to create a table with two columns, `id` and `name`, as the clustering keys:

```python
t = Table('myuser', metadata,
    Column('id', Integer, primary_key=True),
    Column('name', String),
    snowflake_clusterby=['id', 'name'], ...
)
metadata.create_all(engine)
```

### Alembic Support

[Alembic](http://alembic.zzzcomputing.com) is a database migration tool on top of `SQLAlchemy`. Snowflake SQLAlchemy works by adding the following code to `alembic/env.py` so that Alembic can recognize Snowflake SQLAlchemy.

```python
from alembic.ddl.impl import DefaultImpl

class SnowflakeImpl(DefaultImpl):
    __dialect__ = 'snowflake'
```

See [Alembic Documentation](http://alembic.zzzcomputing.com/) for general usage.

### Key Pair Authentication Support

Snowflake SQLAlchemy supports key pair authentication by leveraging its Snowflake Connector for Python underpinnings. See [Using Key Pair Authentication](https://docs.snowflake.net/manuals/user-guide/python-connector-example.html#using-key-pair-authentication) for steps to create the private and public keys.

The private key parameter is passed through `connect_args` as follows:

```python
...
from snowflake.sqlalchemy import URL
from sqlalchemy import create_engine

from cryptography.hazmat.backends import default_backend
from cryptography.hazmat.primitives.asymmetric import rsa
from cryptography.hazmat.primitives.asymmetric import dsa
from cryptography.hazmat.primitives import serialization

with open("rsa_key.p8", "rb") as key:
    p_key= serialization.load_pem_private_key(
        key.read(),
        password=os.environ['PRIVATE_KEY_PASSPHRASE'].encode(),
        backend=default_backend()
    )

pkb = p_key.private_bytes(
    encoding=serialization.Encoding.DER,
    format=serialization.PrivateFormat.PKCS8,
    encryption_algorithm=serialization.NoEncryption())

engine = create_engine(URL(
    account='abc123',
    user='testuser1',
    ),
    connect_args={
        'private_key': pkb,
        },
    )
```

Where `PRIVATE_KEY_PASSPHRASE` is a passphrase to decrypt the private key file, `rsa_key.p8`.

Currently a private key parameter is not accepted by the `snowflake.sqlalchemy.URL` method.

### Merge Command Support

Snowflake SQLAlchemy supports upserting with its `MergeInto` custom expression.
See [Merge](https://docs.snowflake.net/manuals/sql-reference/sql/merge.html)  for full documentation.

Use it as follows:

```python
from sqlalchemy.orm import sessionmaker
from sqlalchemy import MetaData, create_engine
from snowflake.sqlalchemy import MergeInto

engine = create_engine(db.url, echo=False)
session = sessionmaker(bind=engine)()
connection = engine.connect()

meta = MetaData()
meta.reflect(bind=session.bind)
t1 = meta.tables['t1']
t2 = meta.tables['t2']

merge = MergeInto(target=t1, source=t2, on=t1.c.t1key == t2.c.t2key)
merge.when_matched_then_delete().where(t2.c.marked == 1)
merge.when_matched_then_update().where(t2.c.isnewstatus == 1).values(val = t2.c.newval, status=t2.c.newstatus)
merge.when_matched_then_update().values(val=t2.c.newval)
merge.when_not_matched_then_insert().values(val=t2.c.newval, status=t2.c.newstatus)
connection.execute(merge)
```

### CopyIntoStorage Support

Snowflake SQLAlchemy supports saving tables/query results into different stages, as well as into Azure Containers and
AWS buckets with its custom `CopyIntoStorage` expression. See [Copy into](https://docs.snowflake.net/manuals/sql-reference/sql/copy-into-location.html)
for full documentation.

Use it as follows:

```python
from sqlalchemy.orm import sessionmaker
from sqlalchemy import MetaData, create_engine
from snowflake.sqlalchemy import CopyIntoStorage, AWSBucket, CSVFormatter

engine = create_engine(db.url, echo=False)
session = sessionmaker(bind=engine)()
connection = engine.connect()

meta = MetaData()
meta.reflect(bind=session.bind)
users = meta.tables['users']

copy_into = CopyIntoStorage(from_=users,
                            into=AWSBucket.from_uri('s3://my_private_backup').encryption_aws_sse_kms('1234abcd-12ab-34cd-56ef-1234567890ab'),
                            formatter=CSVFormatter().null_if(['null', 'Null']))
connection.execute(copy_into)
```

## Support

Feel free to file an issue or submit a PR here for general cases. For official support, contact Snowflake support at:
<https://community.snowflake.com/s/article/How-To-Submit-a-Support-Case-in-Snowflake-Lodge>
