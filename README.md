# YugaByte C# Driver for YugaByte DB's Cassandra-compatible YCQL API

A modern, [feature-rich][features] and highly tunable C# client library for YugaByte DB using exclusively Cassandra's binary protocol and Cassandra Query Language v3.

The driver supports .NET Framework 4.5+ and .NET Core 1+.

## Installation

[Get it on Nuget][nuget]

```bash
PM> Install-Package YugaByteCassandraCSharpDriver
```


[![Latest stable](https://img.shields.io/nuget/v/YugaByteCassandraCSharpDriver.svg)](https://www.nuget.org/packages/YugaByteCassandraCSharpDriver)

## Features

- Sync and [Async](#asynchronous-api) API
- Simple, [Prepared](#prepared-statements), and [Batch](#batching-statements) statements
- Asynchronous IO, parallel execution, request pipelining
- Connection pooling
- Auto node discovery
- Automatic reconnection
- Configurable [load balancing][policies] and [retry policies][policies]
- Works with any cluster size
- [Linq2Cql][linq] and Ado.Net support

## Documentation

- [Documentation index][docindex]
- [Getting started guide][getting-started]
- [API docs][apidocs]

## Getting Help

Please report any issues on github.

## Upgrading from previous versions



## Basic Usage

```csharp
var cluster = Cluster.Builder()
                     .AddContactPoints("host1")
                     .Build();
// Connect to the nodes using a keyspace
var session = cluster.Connect("sample_keyspace");
// Execute a query on a connection synchronously
var rs = session.Execute("SELECT * FROM sample_table");
// Iterate through the RowSet
foreach (var row in rs)
{
  var value = row.GetValue<int>("sample_int_column");
  // Do something with the value
}
```

### Prepared statements

Prepare your query **once** and bind different parameters to obtain best performance.

```csharp
// Prepare a statement once
var ps = session.Prepare("UPDATE user_profiles SET birth=? WHERE key=?");

// ...bind different parameters every time you need to execute
var statement = ps.Bind(new DateTime(1942, 11, 27), "hendrix");
// Execute the bound statement with the provided parameters
session.Execute(statement);
```

### Batching statements

You can execute multiple statements (prepared or unprepared) in a batch to update/insert several rows atomically even in different column families.

```csharp
// Prepare the statements involved in a profile update once
var profileStmt = session.Prepare("UPDATE user_profiles SET email=? WHERE key=?");
var userTrackStmt = session.Prepare("INSERT INTO user_track (key, text, date) VALUES (?, ?, ?)");
// ...you should reuse the prepared statement
// Bind the parameters and add the statement to the batch batch
var batch = new BatchStatement()
  .Add(profileStmt.Bind(emailAddress, "hendrix"))
  .Add(userTrackStmt.Bind("hendrix", "You changed your email", DateTime.Now));
// Execute the batch
session.Execute(batch);
```

### Asynchronous API

Session allows asynchronous execution of statements (for any type of statement: simple, bound or batch) by exposing the `ExecuteAsync` method.

```csharp
// Execute a statement asynchronously using await
var rs = await session.ExecuteAsync(statement);
```

### Avoid boilerplate mapping code

The driver features a built-in [Mapper][mapper] and [Linq][linq] components that can use to avoid boilerplate mapping code between cql rows and your application entities.

```csharp
User user = mapper.Single<User>("SELECT name, email FROM users WHERE id = ?", userId);
```

See the [driver components documentation][components] for more information.

### Automatic pagination of results

You can iterate indefinitely over the `RowSet`, having the rows fetched block by block until the rows available on the client side are exhausted.

```csharp
var statement = new SimpleStatement("SELECT * from large_table");
// Set the page size, in this case the RowSet will not contain more than 1000 at any time
statement.SetPageSize(1000);
var rs = session.Execute(statement);
foreach (var row in rs)
{
  // The enumerator will yield all the rows from Cassandra
  // Retrieving them in the back in blocks of 1000.
}
```

### User defined types mapping

You can map your [Cassandra User Defined Types][udt] to your application entities.

For a given udt

```cql
CREATE TYPE address (
  street text,
  city text,
  zip_code int,
  phones set<text>
);
```

For a given class

```csharp
public class Address
{
  public string Street { get; set; }
  public string City { get; set; }
  public int ZipCode { get; set; }
  public IEnumerable<string> Phones { get; set;}
}
```

You can either map the properties by name

```csharp
// Map the properties by name automatically
session.UserDefinedTypes.Define(
  UdtMap.For<Address>()
);
```

Or you can define the properties manually

```csharp
session.UserDefinedTypes.Define(
  UdtMap.For<Address>()
    .Map(a => a.Street, "street")
    .Map(a => a.City, "city")
    .Map(a => a.ZipCode, "zip_code")
    .Map(a => a.Phones, "phones")
);
```

You should **map your [UDT][udt] to your entity once** and you will be able to use that mapping during all your application lifetime.

```csharp
var rs = session.Execute("SELECT id, name, address FROM users where id = x");
var row = rs.First();
// You can retrieve the field as a value of type Address
var userAddress = row.GetValue<Address>("address");
Console.WriteLine("user lives on {0} Street", userAddress.Street);
```


### Setting cluster and statement execution options

You can set the options on how the driver connects to the nodes and the execution options.

```csharp
// Example at cluster level
var cluster = Cluster
  .Builder()
  .AddContactPoints(hosts)
  .WithCompression(CompressionType.LZ4)
  .WithLoadBalancingPolicy(new DCAwareRoundRobinPolicy("west"));

// Example at statement (simple, bound, batch) level
var statement = new SimpleStatement(query)
  .SetConsistencyLevel(ConsistencyLevel.Quorum)
  .SetRetryPolicy(DowngradingConsistencyRetryPolicy.Instance)
  .SetPageSize(1000);
```

## Compatibility

- YugaByte DB versions 1.1 and above.
- .NET Framework versions 4.5 and above and .NET Core versions 1.0 and above.

Note: This driver does not support big-endian systems.

## Feedback Requested

Please provide the feedback by opening a github issue.

## Building and running the tests

You can use Visual Studio or msbuild to build the solution. 

[Check the documentation for building the driver from source and running the tests](https://github.com/datastax/csharp-driver/wiki/Building-and-running-tests).

## Running YugaByte specific tests

Since YugaByte cannot be started at Windows, YugaByte cluster should be launched separately and be accessible from testing machine.
Also additional flag should be specified for this cluster:

`--cql_proxy_webserver_port 8888`

CQL proxy metrics is used by tests to verify that corresponding operations were handled locally, i.e. partition awary policy works correctly.

After cluster is created, his endpoint should be set in `YB_CLUSTER_ADDRESS` environment variable.

YugaByte specific tests could be found in project Cassandra.YugaByte.Tests/Cassandra.YugaByte.Tests.csproj

## License

Copyright 2018, YugaByte, Inc.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.

  [apidocs]: https://docs.datastax.com/en/latest-csharp-driver-api/
  [docindex]: https://docs.datastax.com/en/developer/csharp-driver/latest/
  [features]: https://docs.datastax.com/en/developer/csharp-driver/latest/features/
  [getting-started]: https://docs.yugabyte.com/latest/develop/client-drivers/csharp/
  [nuget]: https://nuget.org/packages/YugaByteCassandraCSharpDriver/
  [mailinglist]: https://groups.google.com/a/lists.datastax.com/forum/#!forum/csharp-driver-user
  [jira]: https://datastax-oss.atlassian.net/projects/CSHARP/issues
  [udt]: https://docs.datastax.com/en/dse/6.0/cql/cql/cql_using/useInsertUDT.html
  [poco]: http://en.wikipedia.org/wiki/Plain_Old_CLR_Object
  [linq]: https://docs.datastax.com/en/developer/csharp-driver/latest/features/components/linq/
  [mapper]: https://docs.datastax.com/en/developer/csharp-driver/latest/features/components/mapper/
  [components]: https://docs.datastax.com/en/developer/csharp-driver/latest/features/components/
  [policies]: https://docs.datastax.com/en/developer/csharp-driver/latest/features/tuning-policies/
  [upgrade-guide]: https://docs.datastax.com/en/developer/csharp-driver/latest/upgrade-guide/
  [survey]: https://goo.gl/forms/3BxcP8nKs6
  [dse-driver]: https://docs.datastax.com/en/developer/csharp-driver-dse/latest/
  [slack]: https://academy.datastax.com/slack
  [dse]: https://www.datastax.com/products/datastax-enterprise
