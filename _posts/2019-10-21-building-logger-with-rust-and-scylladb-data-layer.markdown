---
layout: post
title: 'Building logger with Rust and Scylla DB - Data Layer'
description: 'In this post we are starting building a logger using Rust as a programming language and Scylla DB as a data storage. This post is about data layer which is responsible for writing data to and reading it from a database.'
date: 2019-10-21 18:09:00 +0200
categories: rust
---

In this serie, we'll build a performant log-server using [Rust][rust] as a programming language. This post, the first in this serie, will describe the corner stone of our logger - the data layer.

Rust is designed to be fast and safe -- thanks to its ownership model it doesn't require a garabge collector yet providing safety guaranties for data. We will also use [ScyllaDB][scylla] as a database server to store logs data. Scylla is Cassandra compatible database written in C++. According to [benchmarks][scylla-samsung-benchmarks] made by Samsung it performs **10x** times better than Cassandra on high-end machines and up-to **3x** faster on smaller workstations.

![Scylla vs Cassandra benchmarks][scylla-vs-cassandra-benchmark]

<span style="font-size: 12px"><i><b>Figure 1. Scylla vs Cassandra performance benchmark.</b> Image is taken from Scylla official website. YCSB - The Yahoo! Cloud Serving Benchmark (YCSB) is an open-source specification and program suite for evaluating retrieval and maintenance capabilities of computer programs</i><span>

### Starting Scylla DB

In order to start Scylla instance let's use [official Docker][scylla-docker-docs] image provided by ScyllaDB team:

```bash
$ docker run \
  -p 9042:9042/tcp \
  --name some-scylla \
  --hostname some-scylla \
  -d scylladb/scylla
```

In order to persist logs we will create a data volume and mount it to a container.

```bash
$ mkdir -p /xx/yy/scylla/data \
  /xx/yy/scylla/commitlog \
  /xx/yy/scylla/hints \
  /xx/yy/scylla/view_hints

$ docker run --name some-scylla \
  --volume /xx/yy/scylla:/var/lib/scylla \
  -d scylladb/scylla
```

where `/xx/yy` is an actual path to a folder where we create `scylla` folder.

In order to access `cqlsh` and `nodetool` utility you can run

```bash
$ docker exec -it some-scylla cqlsh
```

and

```bash
$ docker exec -it some-scylla nodetool status
```

respectively.

### Data schema

Our log-server will have capabilities for storring and querying temperature
timeseries data. Each measurement entry will include:

- device ID that reported a temperature
- time of a temperature measurement
- temperature value itself.

Assuming that we will query the temperature reported by a certain device for a given time interval, we can use following table schema:

```sql
CREATE TABLE IF NOT EXISTS fast_logger.temperature (
  device UUID,
  time timestamp,
  temperature smallint,
  PRIMARY KEY(device, time)
);
```

This primary key has `device` as a partitioning key which means that all temperature records reported by a given device will be stored in the same partition (node). `time` is a clustering key that's why all records from the same device will be sorted by this field. Thus, this table schema will support an effective temperature querying:

```sql
SELECT * FROM fast_logger.temperature
  WHERE device = ?
  AND time > ?
  AND time < ?;
```

where `?` will be replaced with actual values -- device ID, time-from and time-to respectively.

In order to create `fast_logger` keyspace, first we need to decide which replication strategy to use and what would be the replication factor.

If it comes to replication strategy, there are two of them to choose:

- `SimpleStrategy` must be used for a single data center only.

- `NetworkTopologyStrategy` is a strategy you should consider when you plan to have few data centers.

As for the replication factor, it depends on how much replicas you want to have in your cluster.

In our case, we are going to build a logger that uses a single Scylla node. That's why `SimpleStrategy` with replication factor equal to 1 is absolutelly sufficient:

```sql
CREATE KEYSPACE IF NOT EXISTS fast_logger
  WITH REPLICATION = {
    'class': 'SimpleStrategy',
    'replication_factor': 1
  };
```

Since it's not recommended to disable durabl writes when using `SimpleStrategy` we keep default value for `DURABLE_WRITES` which is `true` (we don't specify this parameter in the query at all).

### Rust and connection to the DB

In order to create a skeleton for an executable application such as our logger server in Rust, we have to run `cargo new fast_logger` command in a terminal.

It's assumed that you already have Rust and [Cargo][cargo] installed. If you don't, we recommend you to use [rustup.rs toolchain][rustup] that will help you to install Rust compiler as well as other commonly used tools and crates. Each rustup.rs profile contains rust compiler (`rustc`), standard library (`rust-std`) and Cargo package manager.

As a database driver we are going to use [CDRS][cdrs-git] Rust crate. [UUID][uuid-repo] crate will be used to represed UUID of a device. Apart of CDRS itself we will use `cdrs_helpers_derive` crate that provide procedural macroses that derive methods which help with converting Rust structures into Cassandra/Scylla values. [`time`][time-docs] crate is used to work with time primitives. Here is how dependency section should look like in _Cargo.toml_ file:

```toml
[dependencies]
cdrs = "2"
cdrs_helpers_derive = "0.1.0"
uuid = "0.7"
time = "0.1"
```

Do not forget to add

```rust
extern crate cdrs;
#[macro_use]
extern crate cdrs_helpers_derive;
extern crate uuid;
extern crate time;
```

to _main.rs_ so that you could you declared dependencies in your code.

To be able to connect to a DB server CDRS needs following information:

- authenticator -- any structure that has [`Authenticator`](https://docs.rs/cdrs/2.2.0/cdrs/authenticators/trait.Authenticator.html) implementation and which matches an authentication strategy used by DB Server. We're going to disable authentication so `NoneAuthenticator` should be used:

```rust
use cdrs::authenticators::NoneAuthenticator;

let auth = NoneAuthenticator{};
```

- cluster configuration -- it consists of connection configuration provided for each node in a cluster which we want to connect to:

```rust
let node = NodeTcpConfigBuilder::new("127.0.0.1:9042", auth).build();
let cluster_config = ClusterTcpConfig(vec![node]);
```

Since we're going to have just a single Scylla node only one node config is provided to a cluster. [`NodeTcpConfigBuilder`](https://docs.rs/cdrs/2.2.0/cdrs/cluster/struct.NodeTcpConfigBuilder.html) is a builder for [`NodeTcpConfig`](https://docs.rs/cdrs/2.2.0/cdrs/cluster/struct.NodeTcpConfig.html). When we will pass `cluster_config` to session creator, the creator will creat a pool of re-usable DB connectoin for each node in a cluster config. `NodeTcpConfigBuilder` accepts the same parameters as Rust r2d2 `Pool` [builder](https://docs.rs/r2d2/0.8.6/r2d2/struct.Builder.html).

Having an authenticator and a cluster configurtion we can create a new CDRS session (_db.rs_ file - it's a module that will hold the logic for working with Scylla instance):

```rust
use cdrs::{
  authenticators::NoneAuthenticator,
  cluster::{
    session::{
      new as new_session,
      // other option: new_lz4 as new_lz4_session,
      // other option: new_snappy as new_snappy_session
      Session,
    },
    ClusterTcpConfig, NodeTcpConfigBuilder, TcpConnectionPool,
  },
  load_balancing::SingleNode,
  query::*,
  Result as CDRSResult,
};

pub type CurrentSession = Session<SingleNode<TcpConnectionPool<NoneAuthenticator>>>;

pub fn create_db_session() -> CDRSResult<CurrentSession> {
  let auth = NoneAuthenticator;
  let node = NodeTcpConfigBuilder::new("127.0.0.1:9042", auth).build();
  let cluster_config = ClusterTcpConfig(vec![node]);
  new_session(&cluster_config, SingleNode::new())
}
```

Apart of `SingleNode` load balancing strategy used here CDRS provides `Random`, `RoundRobin` and `RoundRobinSync` strategies. Being defined during session intialization load balancing strategy cannot be changed for a given session instance which is not the case for compression. Compression can be changed at any moment of time, for example `session.compression = cdrs::compression::Compression::Lz4;`.

Since we are going write temperature measurement one-by-one, also taking into account that each measurement is a fairly small amount of data, session without any compression looks most reasonable for our logger.

Now, let us add functions for creating keyspace and a table where we will store temperature measurements (add to _db.rs_):

```rust
// ...

static CREATE_KEYSPACE_QUERY: &'static str = r#"
  CREATE KEYSPACE IF NOT EXISTS fast_logger
    WITH REPLICATION = {
      'class': 'SimpleStrategy',
      'replication_factor': 1
    };
"#;

static CREATE_TEMPERATURE_TABLE_QUERY: &'static str = r#"
  CREATE TABLE IF NOT EXISTS fast_logger.temperature (
    device UUID,
    time timestamp,
    temperature smallint,
    PRIMARY KEY(device, time)
  );
"#;

pub fn create_keyspace(session: &mut CurrentSession) -> CDRSResult<()> {
  session.query(CREATE_KEYSPACE).map(|_| (()))
}

pub fn create_temperature_table(session: &mut CurrentSession) -> CDRSResult<()> {
  session.query(CREATE_TEMPERATURE_TABLE_QUERY).map(|_| (()))
}
```

Here, we use queries for creating a keyspace and a table which we have previously agreed on.

#### Write temperature measurements

Let us now create a structure that will represent a single temperature measurement that will be stored in `fast_logger.temperature` table (_temperature_measurement.rs_ file):

```rust
use time::Timespec;
use cdrs::types::prelude::*;
use cdrs::types::from_cdrs::FromCDRSByName;
use uuid::Uuid;

#[derive(Debug, TryFromRow)]
struct TemperatureMeasurement {
  pub device: Uuid,
  pub time: Timespec,
  pub temperature: i16
}

impl TemperatureMeasurement {
  pub fn into_query_values(self) -> QueryValues {
    query_values!(
      "device" => self.device,
      "time" => self.time,
      "temperature" => self.temperature
    )
  }
}
```

Apart of standard `Debug` we've derived one extra trait - `TryFromRow` provided by CDRS.

```rust
pub trait TryFromRow: Sized {
    fn try_from_row(row: Row) -> error::Result<Self>;
}
```

The proc macro from `cdrs_helpers_derive` brings `try_from_row` method that will convert Scylla row received from a DB server into `TemperatureMeasurement` structure.

As a part of `TemperatureMeasurement` implementation we've provided `into_query_values` method that consumes a measurement structure and returns Cassandra values that further will be used by Scylla for substituting `?` in `ADD_MEASUREMENT_QUERY` (add to _db.rs_ file):

```rust
// ...

static ADD_MEASUREMENT_QUERY: &'static str = r#"
  INSERT INTO fast_logger.temperature (device, time, temperature)
    VALUES (?, ?, ?);
"#;

//...

pub fn add_measurement(
  session: &mut CurrentSession,
  measurement: TemperatureMeasurement,
) -> CDRSResult<()> {
  session
    .query_with_values(ADD_MEASUREMENT_QUERY, measurement.into_query_values())
    .map(|_| (()))
}

```

### Reading measurements

As you remember, during the data scheme design phase we assumed that our logger will be cappable to return measurements made by a selected device(-s) during a given timeframe. Taking into account this scenario, for `fast_logger.temperature` table, we've created such primary key that guarantees efficient read operations.

Now we can create a select-query and a corresponded function in _db.rs_ module:

```rust
use time::Timespec;
use uuid::Uuid;

// ...

static SELECT_MEASUREMENTS_QUERY: &'static str = r#"
  SELECT * FROM fast_logger.temperature
    WHERE device = ?
      AND time > ?
      AND time < ?;
"#;

//...

pub fn select_measurements(
  session: &mut CurrentSession,
  devices: Uuid,
  time_from: Timespec,
  time_to: Timespec,
) -> CDRSResult<Vec<TemperatureMeasurement>> {
  let values = query_values!(devices, time_from, time_to);
  session
    .query_with_values(SELECT_MEASUREMENTS_QUERY, values)
    .and_then(|res| res.get_body())
    .and_then(|body| {
      body
        .into_rows()
        .ok_or(CDRSError::from("cannot get rows from a response body"))
    })
    .and_then(|rows| {
      let mut measurements: Vec<TemperatureMeasurement> = Vec::with_capacity(rows.len());

      for row in rows {
        measurements.push(TemperatureMeasurement::try_from_row(row)?);
      }

      Ok(measurements)
    })
}
```

Since the `SELECT` query has all filters (`device`, `time_from` and `time_to`) as parameters we need to provide actual values to CDRS. This is done with help of `query_values!` macro provided by CDRS.

`query_values!` accepts arguments in two different forms - named and non-named - and depending on that it returns either named query values or non-named ones. As you may guess, eventough named values can be provided in any order they will be used by Scylla as a values for columns that have matched names. Unlike that, non-named query values should be provided in the same order as `?`-s appear in a corresponded query.

### Further improvements - abstract session

So far we've created all functions necessary for logging and querying measurements data - `create_db_session` connects to a DB cluster (though it consists of just one node), `create_keyspace` creates `fast_logger` keyspace if not exits, `create_temperature_table` creates `fast_logger.temperature` table if not exist, `add_measurement` inserts temperature measurement data into the table and, finally, `select_measurements` gets all measurements made by given devices in a given timeframe.

Eventough these functions are sufficient for our needs we can try to make them more testable and refactor the process of saving measurements more efficient.

Let us analyse code of `create_keyspace`, `create_temperature_table`, `add_measurement` and `select_measurements`. From one hand side we can notice that all of them receive `&mut CurrentSession`. It is an alias that we've created for `Session<SingleNode<TcpConnectionPool<NoneAuthenticator>>>`. This long generic type means following `Session` that uses `SingleNode` as a load balancing strategy where each node has a pool of TCP connections that use `NoneAuthenticator` as an authentication strategy. From other hand side, why we need session is to run queries via `session.query(...)` or `session.query_with_values`. Both of these methods are the part of [`QueryExecutor`][cdrs-query-executor] implementation. It means that `session: &mut CurrentSession` in an argument can be efficiently replaced with following:

```rust
pub fn create_keyspace<T, M>(session: &mut impl QueryExecutor<T, M>) -> CDRSResult<()>
where
  T: CDRSTransport + 'static,
  M: r2d2::ManageConnection<Connection = RefCell<T>, Error = CDRSError> + Sized,
{
  /// ...
}

pub fn create_temperature_table<T, M>(session: &mut impl QueryExecutor<T, M>) -> CDRSResult<()>
where
  T: CDRSTransport + 'static,
  M: r2d2::ManageConnection<Connection = RefCell<T>, Error = CDRSError> + Sized,
{
  // ...
}

pub fn add_measurement<T, M>(
  session: &mut impl QueryExecutor<T, M>,
  measurement: TemperatureMeasurement,
) -> CDRSResult<()>
where
  T: CDRSTransport + 'static,
  M: r2d2::ManageConnection<Connection = RefCell<T>, Error = CDRSError> + Sized,
{
  // ...
}

pub fn select_measurements<T, M>(
  session: &mut impl QueryExecutor<T, M>,
  devices: Vec<Uuid>,
  time_from: Timespec,
  time_to: Timespec,
) -> CDRSResult<Vec<TemperatureMeasurement>>
where
  T: CDRSTransport + 'static,
  M: r2d2::ManageConnection<Connection = RefCell<T>, Error = CDRSError> + Sized,
{
  // ...
}
```

Despite function signatures now look more complex it's beneficial for the code to have them in this form. Thanks to that we've made our code more decoupled and more testable. We've rid of `CurrentSession` that creates a real connection to the DB, now these functions accepts **any** structure that implements `QueryExecutor<T, M>` trait. So, now we could create a mock query executor that doesn't need a real connection to a DB cluster. This is very beneficial for unit testing of our data layer. Apart of that, instead of TCP connection now `QueryExecutor` relies on general `CDRSTransport` trait, so the same functions will work with sessions based both on TCP and on TLS.

Generally speaking, CDRS `Session` implements following traits related to querying:

- [`BatchExecutor`][cdrs-batch-executor] that contains methods for batching querying, in other words making few queries in a single request to a DB.

- [`ExecExecutor`][cdrs-exec-executor] that contains methods for executing prepeared queries.

- [`PrepareExecutor`][cdrs-prepare-executor] that contains methods for preparing queries.

- [`QueryExecutor`][cdrs-query-executor] that contains methods for running queries.

### Further improvements - prepare once, execute many times

Another kind of improvement is related to the way our logger is going to communicate with a database. What I mean is because of the fact that the logger will always send measurements of the same shape the only one this which will differ is actuall values -- query will be the same. In terms of bytes, query string is comparable with actuall values. So, instead of doubling the size of a data being sent from a logger to Scylla we can try to optimize it by leveraging query preparation and query execution concepts that are supported by Scylla (as well as Cassandra).

So, we can request Scylla to prepare itself for `INSERT` queries. This is going to be happen just once - during logger initialization phase. Then we can just execute prepeared query by sending only values that should be inserted but not the query itself (add to _db.rs_):

```rust
pub fn prepare_add_measurement<T, M>(
  session: &mut impl PrepareExecutor<T, M>,
) -> CDRSResult<PreparedQuery>
where
  T: CDRSTransport + 'static,
  M: r2d2::ManageConnection<Connection = RefCell<T>, Error = CDRSError> + Sized,
{
  session.prepare(ADD_MEASUREMENT_QUERY)
}

pub fn execute_add_measurement<T, M>(
  session: &mut impl ExecExecutor<T, M>,
  prepared_query: &PreparedQuery,
  measurement: TemperatureMeasurement,
) -> CDRSResult<()>
where
  T: CDRSTransport + 'static,
  M: r2d2::ManageConnection<Connection = RefCell<T>, Error = CDRSError> + Sized,
{
  session
    .exec_with_values(prepared_query, measurement.into_query_values())
    .map(|_| (()))
}
```

### Conclusion

Thus in this chapter we've build a data layer of the logger. It's cappable for establishing Scylla session, creating a keyspace and a table for measurement entries, adding measurements and retrieving them. The full code of a data layer can be found [https://github.com/AlexPikalov/ps-logger](https://github.com/AlexPikalov/ps-logger) ([revision](https://github.com/AlexPikalov/ps-logger/tree/72b063c82484abfda41f3cf4eb95c999eb9e1e74)).

In next post we'll continue building our logger by adding a web interface to our service.

[rust]: https://www.rust-lang.org/
[scylla]: https://www.scylladb.com/
[scylla-samsung-benchmarks]: https://www.scylladb.com/product/benchmarks/samsung-benchmark/
[scylla-vs-cassandra-benchmark]: https://1bpezptkft73xxds029zrs59-wpengine.netdna-ssl.com/wp-content/uploads/samsung-1.png
[scylla-docker-docs]: https://hub.docker.com/r/scylladb/scylla/
[cdrs-git]: https://github.com/AlexPikalov/cdrs/
[cdrs-query-executor]: https://docs.rs/cdrs/2.2.0/cdrs/query/trait.QueryExecutor.html
[cdrs-batch-executor]: https://docs.rs/cdrs/2.2.0/cdrs/query/trait.BatchExecutor.html
[cdrs-exec-executor]: https://docs.rs/cdrs/2.2.0/cdrs/query/trait.ExecExecutor.html
[cdrs-prepare-executor]: https://docs.rs/cdrs/2.2.0/cdrs/query/trait.PrepareExecutor.html
[cargo]: https://doc.rust-lang.org/cargo/
[rustup]: https://github.com/rust-lang/rustup.rs
[uuid-repo]: https://github.com/uuid-rs/uuid
[time-docs]: https://docs.rs/time/0.1.42/time/
