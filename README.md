# Postgrex

[![Build Status](https://travis-ci.org/elixir-ecto/postgrex.svg?branch=master)](https://travis-ci.org/elixir-ecto/postgrex)

PostgreSQL driver for Elixir.

Documentation: http://hexdocs.pm/postgrex/

## Example

```iex
iex> {:ok, pid} = Postgrex.start_link(hostname: "localhost", username: "postgres", password: "postgres", database: "postgres")
{:ok, #PID<0.69.0>}
iex> Postgrex.query!(pid, "SELECT user_id, text FROM comments", [])
%Postgrex.Result{command: :select, empty?: false, columns: ["user_id", "text"], rows: [[3,"hey"],[4,"there"]], size: 2}}
iex> Postgrex.query!(pid, "INSERT INTO comments (user_id, text) VALUES (10, 'heya')", [])
%Postgrex.Result{command: :insert, columns: nil, rows: nil, num_rows: 1}}
```

## Features

  * Automatic decoding and encoding of Elixir values to and from PostgreSQL's binary format
  * User defined extensions for encoding and decoding any PostgresSQL type
  * Supports transactions, prepared queries and multiple pools via [DBConnection](https://github.com/elixir-ecto/db_connection)
  * Supports PostgreSQL 8.4 and 9.0-9.6 (hstore is not supported on 8.4)

## Data representation

    PostgreSQL      Elixir
    ----------      ------
    NULL            nil
    bool            true | false
    char            "é"
    int             42
    float           42.0
    text            "eric"
    bytea           <<42>>
    numeric         #Decimal<42.0> *
    date            %Postgrex.Date{year: 2013, month: 10, day: 12}
    time(tz)        %Postgrex.Time{hour: 0, min: 37, sec: 14, usec: 0} **
    timestamp(tz)   %Postgrex.Timestamp{year: 2013 month: 10, day: 12, hour: 0, min: 37, sec: 14, usec: 0} **
    interval        %Postgrex.Interval{months: 14, days: 40, secs: 10920}
    array           [1, 2, 3]
    composite type  {42, "title", "content"}
    range           %Postgrex.Range{lower: 1, upper: 5}
    uuid            <<160,238,188,153,156,11,78,248,187,109,107,185,189,56,10,17>>
    hstore          %{"foo" => "bar"}
    oid types       42
    enum            "ok" ***
    bit             << 1::1, 0::1 >>
    varbit          << 1::1, 0::1 >>

\* [Decimal](http://github.com/ericmj/decimal)

\*\* Timezones will always be normalized to UTC or assumed to be UTC when no information is available, either by PostgreSQL or Postgrex

\*\*\* Enumerated types (enum) are custom named database types with strings as values.

Postgrex does not automatically cast between types. For example, you can't pass a string where a date is expected. To add type casting, support new types, or change how any of the types above are encoded/decoded, you can use extensions.

## Extensions

Extensions are used to extend Postgrex' built-in type encoding/decoding.

Here is a [JSON extension](https://github.com/elixir-ecto/postgrex/blob/master/lib/postgrex/extensions/json.ex) that supports encoding/decoding Elixir maps to the Postgres' JSON type.

To use the extension pass it to the connection as seen below:

```elixir
Postgrex.start_link(extensions: [{Postgrex.Extensions.JSON, library: Poison}], ...)
```

## OID type encoding

PostgreSQL's wire protocol supports encoding types either as text or as binary. Unlike most client libraries Postgrex uses the binary protocol, not the text protocol. This allows for efficient encoding of types (e.g. 4-byte integers are encoded as 4 bytes, not as a string of digits) and automatic support for arrays and composite types.

Unfortunately the PostgreSQL binary protocol transports [OID types](http://www.postgresql.org/docs/current/static/datatype-oid.html#DATATYPE-OID-TABLE) as integers while the text protocol transports them as string of their name, if one exists, and otherwise as integer.

This means you either need to supply oid types as integers or perform an explicit cast (which would be automatic when using the text protocol) in the query.

```elixir
# Fails since $1 is regclass not text.
query("select nextval($1)", ["some_sequence"])

# Perform an explicit cast, this would happen automatically when using a
# client library that uses the text protocol.
query("select nextval($1::text::regclass)", ["some_sequence"])

# Determine the oid once and store it for later usage. This is the most
# efficient way, since PostgreSQL only has to perform the lookup once. Client
# libraries using the text protocol do not support this.
%{rows: [{sequence_oid}]} = query("select $1::text::regclass", ["some_sequence"])
query("select nextval($1)", [sequence_oid])
```

## PgBouncer

When using PgBouncer with transaction or statement pooling named prepared
queries can not be used because the bouncer may route requests from the same
postgrex connection to different PostgreSQL backend processes and discards named
queries after the transactions closes. To force unnamed prepared queries:

```elixir
Postgrex.start_link(prepare: :unnamed)
```

## Contributing

To contribute you need to compile Postgrex from source and test it:

```
$ git clone https://github.com/elixir-ecto/postgrex.git
$ cd postgrex
$ mix test
```

The tests requires some modifications to your [hba file](http://www.postgresql.org/docs/9.3/static/auth-pg-hba-conf.html). The path to it can be found by running `$ psql -U postgres -c "SHOW hba_file"` in your shell. Put the following above all other configurations (so that they override):

```
host    all             postgrex_md5_pw         127.0.0.1/32    md5
host    all             postgrex_cleartext_pw   127.0.0.1/32    password
```

The server needs to be restarted for the changes to take effect. Additionally you need to setup a Postgres user with the same username as the local user and give it trust or ident in your hba file. Or you can export $PGUSER and $PGPASSWORD before running tests.

### Testing hstore on 9.0

Postgres versions 9.0 does not have the `CREATE EXTENSION` commands. This means we have to locate the postgres installation and run the `hstore.sql` in `contrib` to install `hstore`. Below is an example command to test 9.0 on OS X with homebrew installed postgres:

```
$ PGVERSION=9.0 PGPATH=/usr/local/share/postgresql9/ mix test
```

## License

Copyright 2013 Eric Meadows-Jönsson

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
