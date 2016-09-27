# Cache

## Overview
The cache filter is capable of caching the result of SELECTs, so that subsequent identical
SELECTs are served directly by MaxScale, without being routed to any server.

## Configuration

The cache filter is straightforward to configure and simple to add to any
existing service.

```
[Cache]
type=filter
module=cache
ttl=5
storage=...
storage_options=...
rules=...
debug=...

[Cached Routing Service]
type=service
...
filters=Cache
```

Each configured cache filter uses a storage of its own. That is, if there
are two services, each configured with a specific cache filter, then,
even if queries target the very same servers the cached data will not
be shared.

Two services can use the same cache filter, but then either the services
should use the very same servers _or_ a completely different set of servers,
where the used table names are different. Otherwise there can be unintended
sharing.


### Filter Parameters

The cache filter has one mandatory parameter - `storage` - and a few
optional ones.

#### `storage`

The name of the module that provides the storage for the cache. That
module will be loaded and provided with the value of `storage_options` as
argument. For instance:
```
storage=storage_rocksdb
```

#### `storage_options`

A comma separated list of arguments to be provided to the storage module,
specified in `storage`, when it is loaded. Note that the needed arguments
depend upon the specific module. For instance,
```
storage_options=storage_specific_option1=value1,storage_specific_option2=value2
```

#### `max_resultset_rows`

Specifies the maximum number of rows a resultset can have in order to be
stored in the cache. A resultset larger than this, will not be stored.
```
max_resultset_rows=1000
```
Zero or a negative value is interpreted as no limitation.

The default value is `-1`.

#### `max_resultset_size`

Specifies the maximum size a resultset can have, measured in kibibytes,
in order to be stored in the cache. A resultset larger than this, will
not be stored.
```
max_resultset_size=128
```
The default value is 64.

#### `ttl`

_Time to live_; the amount of time - in seconds - the cached result is used
before it is refreshed from the server.

If nothing is specified, the default _ttl_ value is 10.

```
ttl=60
```

#### `rules`

Specifies the path of the file where the caching rules are stored. A relative
path is interpreted relative to the _data directory_ of MariaDB MaxScale.

```
rules=/path/to/rules-file
```

#### `debug`

An integer value, using which the level of debug logging made by the cache
can be controlled. The value is actually a bitfield with different bits
denoting different logging.

   * `0` (`0b0000`) No logging is made.
   * `1` (`0b0001`) A matching rule is logged.
   * `2` (`0b0010`) A non-matching rule is logged.
   * `4` (`0b0100`) A decision to use data from the cache is logged.
   * `8` (`0b1000`) A decision not to use data from the cache is logged.

Default is `0`. To log everything, give `debug` a value of `15`.

```
debug=2
```

# Rules

The caching rules are expressed as a JSON object.

There are two decisions to be made regarding the caching; in what circumstances
should data be stored to the cache and in what circumstances should the data in
the cache be used.

In the JSON object this is visible as follows:

```
{
    store: [ ... ],
    use: [ ... ]
}
```

The `store` field specifies in what circumstances data should be stored to
the cache and the `use` field specifies in what circumstances the data in
the cache should be used. In both cases, the value is a JSON array containg
objects.

## When to Store

By default, if no rules file have been provided or if the `store` field is
missing from the object, the results of all queries will be stored to the
cache, subject to `max_resultset_rows` and `max_resultset_size` cache filter
parameters.

By providing a `store` field in the JSON object, the decision whether to
store the result of a particular query to the cache can be controlled in
a more detailed manner. The decision to cache the results of a query can
depend upon

   * the database,
   * the table,
   * the column, or
   * the query itself.

Each entry in the `store` array is an object containing three fields,

```
{
    "attribute": <string>,
    "op": <string>
    "value": <string>
}
```

where,
   * the _attribute_ can be `database`, `table`, `column` or `query`,
   * the _op_ can be `=`, `!=`, `like` or `unlike`, and
   * the _value_ a string.

If _op_ is `=` or `!=` then _value_ is used verbatim; if it is `like`
or `unlike`, then _value_ is interpreted as a _pcre2_ regular expression.

The objects in the `store` array are processed in order. If the result
of a comparison is _true_, no further processing will be made and the
result of the query in question will be stored to the cache.

If the result of the comparison is _false_, then the next object is
processed. The process continues until the array is exhausted. If there
is no match, then the result of the query is not stored to the cache.

Note that as the query itself is used as the key, although the following
queries
```
select * from db1.tbl
```
and
```
use db1;
select * from tbl
```
target the same table and produce the same results, they will be cached
separately. The same holds for queries like
```
select * from tbl where a = 2 and b = 3;
```
and
```
select * from tbl where b = 3 and a = 2;
```
as well. Although they conceptually are identical, there will be two
cache entries.

### Examples

Cache all queries targeting a particular database.
```
{
    "store": [
        {
            "attribute": "database",
            "op": "=",
            "value": "db1"
        }
    ]
}
```

Cache all queries _not_ targeting a particular table
```
{
    "store": [
        {
            "attribute": "table",
            "op": "!=",
            "value": "tbl1"
        }
    ]
}
```

That will exclude queries targeting table _tbl1_ irrespective of which
database it is in. To exclude a table in a particular database, specify
the table name using a qualified name.
```
{
    "store": [
        {
            "attribute": "table",
            "op": "!=",
            "value": "db1.tbl1"
        }
    ]
}
```

Cache all queries containing a WHERE clause
```
{
    "store": [
        {
            "attribute": "query",
            "op": "like",
            "value": ".*WHERE.*"
        }
    ]
}
```

Note that that will actually cause all queries that contain WHERE anywhere,
to be cached.

## When to Use

# Storage

## Storage RocksDB