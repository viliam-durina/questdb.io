---
id: programmatic
title: Programmatic Interfaces
sidebar_label: Programmatic Interfaces
---

## Overview
QuestDB exposes the following interfaces:
- **[Postgres wire protocol](intPROGRAM.md#postgres-wire-protocol)**
- **[Influx line protocol](intPROGRAM.md#influx-line-protocol)**
- **[Java](intPROGRAM.md#java)**
- **[HTTP REST](intPROGRAM.md#http-rest)**


## Postgres Wire Protocol

> This section is work in progress. 

### Overview

QuestDB implements the Postgres wire protocol. Users can connect to QuestDB the same way they would connect to Postgress with:
- Any language that already has a Postgres adapter (R, Python, etc.).
- Any tool in the Postgres ecosystem.
- QuestDB's time-series and performance capabilities.
- Await-free multi-user experience thanks to QuestDB's smart thread management.

## Influx Line Protocol

### Overview
QuestDB is able to ingest data over **Influx Line Protocol**. Existing systems already publishing line protocol
need not change at all. Although QuestDB uses relational model, line protocol parser will automatically create
tables and add missing columns.

For the time being QuestDB can listen for UDP packets, either multicast or unicast. TCP and HTTP support for line
protocol is on our road map.

> QuestDB listens for multicast on `232.1.2.3:9009`. The address change or switch to unicast can be done via configuration. If you run QuestDB via Docker
> start container as `run --rm -it -p 9000:9000 -p 8892:8892 -p 9009:9009 --net=host questdb/questdb`  and publish
> multicast packets with TTL of at least 2.

### Syntax
Influx Line Protocol follows this syntax

```shell script
table_name,tagset valueset  timestamp
```

where:
| Element               | Definition                                                                                 |
|-----------------------|--------------------------------------------------------------------------------------------|
| table_name            | Name of the table where QuestDB will write data.                                           |
| tagset                | Array of string key-value pairs  separated by commas that represent the reading's associated metadata|
| values                | Array of key-value pairs separated by commas that represent the readings. The keys are string, values can be numeric or boolean|
| timestamp             | UNIX timestamp. By default in microseconds. Can be changed in the configuration (see below) |

> When `table_name` does not correspond to an existing table, QuestDB will create the table on the fly using the name
>provided. Column types will be automatically recognized and assigned based on the data.
>
#### Examples
Let's assume the following data:
|timestamp             | city            | temperature           | humidity              | make              |
|----------------------|-----------------|-----------------------|-----------------------|-------------------|
|1465839830100400000   | London          | 23.5                  | 0.343                 | Omron             |
|1465839830100600000   | Bristol         | 23.2                  | 0.443                 | Honeywell         |
|1465839830100700000   | London          | 23.6                  | 0.358                 | Omron             |


Line protocol to insert this data in the `readings` table would look like this:
```shell script
readings,city=London,make=Omron temperature=23.5,humidity=0.343 1465839830100400000
readings,city=Bristol,make=Honeywell temperature=23.2,humidity=0.443 1465839830100600000
readings,city=London,make=Omron temperature=23.6,humidity=0.348 1465839830100700000
```

> Note there are only 2 spaces in each line. First between the `tagset` and `values`. Second between
> `values` and `timestamp`.

### Dealing with irregularly-structured data
>QuestDB can support on-the-fly data structure changes with minimal overhead. Should users decide to send
>varying quantities of readings or metadata tags for different entries, QuestDB will adapt on the fly.

Influx line protocol makes it possible to send data under different shapes. Each new entry may contain certain 
metadata tags or readings, and others not. Whilst the example just above
highlights structured data, it is possible for Influx line protocol users to send data as follows.

```shell script
readings,city=London temperature=23.2 1465839830100400000
readings,city=London temperature=23.6 1465839830100700000
readings,make=Honeywell temperature=23.2,humidity=0.443 1465839830100800000
```

Note that on the third line, 
- a new `tag` is added: "make"
- a new `field` is added: "humidity"

After writing two entries, the data would look like this:
|timestamp             | city            | temperature           |
|----------------------|-----------------|-----------------------|
|1465839830100400000   | London          | 23.5                  |
|1465839830100700000   | London          | 23.6                  | 

The third entry would result in the following table:
|timestamp             | city            | temperature           | humidity              | make              |
|----------------------|-----------------|-----------------------|-----------------------|-------------------|
|1465839830100400000   | London          | 23.5                  | NULL                  | NULL              |
|1465839830100700000   | London          | 23.6                  | NULL                  | NULL              |
|1465839830100800000   | NULL            | 23.2                  | 0.358                 | Honeywell         |

>Adding columns on the fly is no issue for QuestDB. New columns will be created in the affected partitions, and only
>populated if they contain values. Whilst we offer this function for flexibility, we recommend that users try to minimise 
>structural changes to maintain operational simplicity. 

### Configuration
The following configuration parameters can be added to the configuration file to customise the interaction using 
Influx line protocol. The configuration file is found under `/questdb/conf/questdb.conf`

| Property                       | Type           | Default value  | Description                                         |
|--------------------------------|----------------|----------------|-----------------------------------------------------|
|**line.udp.join**               | `string`       |*"232.1.2.3"*   | Multicast address receiver joins. This values is ignored when receiver is in "unicast" mode   |
|**line.udp.bind.to**            | `string`       |*"0.0.0.0:9009"* | IP address of the network interface to bind listener to and port. By default UDP receiver listens on all network interfaces|
|**line.udp.commit.rate**        | `long`         |*1000000*        | For packet bursts the number of continuously received messages after which receiver will force commit. Receiver will commit irrespective of this parameter when there are no messages.                 |
|**line.udp.msg.buffer.size**    | `long`         |*2048*          | Buffer used to receive single message. This value should be roughly equal to your MTU size.                |
|**line.udp.msg.count**          | `long`         |*10000*        | Only for Linux. On Linix QuestDB will use recvmmsg(). This is the max number of messages to receive at once.                                                    |
|**line.udp.receive.buffer.size**| `long`         |*8388608*       | UDP socket buffer size. Larger size of the buffer will help reduce message loss during bursts.                                                    |
|**line.udp.enabled**            | `boolean`      |*true*          | Flag to enable or disable UDP receiver                                                    |
|**line.udp.own.thread**         | `boolean`      |*false*         | When "true" UDP receiver will use its own thread and busy spin that for performance reasons. "false" makes receiver use worker threads that do everything else in QuestDB.                                                    |
|**line.udp.own.thread.affinity**         | `int`      |*-1*    |  -1 does not set thread affinity. OS will schedule thread and it will be liable to run on random cores and jump between the. 0 or higher pins thread to give core. This propertly is only valid when UDP receiver uses own thread. |
|**line.udp.unicast**            | `boolean`      |*false*         | When true, UDP will me unicast. Otherwise multicast |
|**line.udp.timestamp**          | `string` | "n" | Input timestamp resolution. Possible values are "n", "u", "ms", "s" and "h". |
|**line.udp.commit.mode**        | `string` | "nosync" | Commit durability. Available values are "nosync", "sync" and "async"   |


### Sending messages from Java

QuestDB library provides fast and efficient way of sending line protocol messages. Sender implementation entry point is
`io.questdb.cutlass.line.udp.LineProtoSender`, it is fully zero-GC and is latency in a region of 200ns per message.


#### Get started
**Step 1:** Create an instance of `LineProtoSender`.

| Arguments                    | Description                                                                          |
|------------------------------|--------------------------------------------------------------------------------------|
|interfaceIPv4Address          |  Network interface to use to send messages.                                                                                    |
|sendToIPv4Address             |  Destination IP address                                                                                    |
|bufferCapacity                |  Send buffer capacity to batch messages. Do not configure this buffer over the MTU size                                                                                    |
|ttl                           |  UDP packet TTL. Set this number appropriate to how many VLANs your messages have to traverse before reaching the destination                                                                                    |

Example:
```java

LineProtoSender sender = new LineProtoSender(0, Net.parseIPv4("232.1.2.3"), 9009, 110, 2);
```

**Step 2:** Create `entries` by building each entry's tagset and fieldset.
Syntax
```java

sender.metric("table_name").tag("key","value").field("key", value).$(timestamp);
```
where:
| Element               | Description                                                                   | Can be repeated     |
|-----------------------|-------------------------------------------------------------------------------|---------------------|
|metric(tableName)      | Specify which table the data is to be written into                            | no                  |
|tag("key","value")     | Use to add a new key-value entry as metadata                                  | yes                 |
|field("key","value")   | Use to add a new key-value entry as reading                                   | yes                 |
|$(timestamp)           | Specify the timestamp for the reading                                         | no                  |

> When an element can be repeated, you can concatenate them several times to add more.
> Example tag("a","x").tag("b","y").tag("c","z") etc

Example:
```java

    sender.metric("readings").tag("city", "London").tag("by", "quest").field("temp", 3400).field("humid", 0.434).$(Os.currentTimeNanos());
```


Sender will send message as soon as send buffer is full. To send messages before buffer fills up you can use `sender.flush()`

#### Example
```java

    LineProtoSender sender = new LineProtoSender(0, Net.parseIPv4("232.1.2.3"), 9009, 1024, 2);
    sender.metric("readings").tag("city", "London").tag("by", "quest").field("temp", 3400).$(Os.currentTimeNanos());
    sender.metric("readings").tag("city", "London").tag("by", "quest").field("temp", 3400).$(Os.currentTimeNanos());
    sender.metric("readings").tag("city", "London").tag("by", "quest").field("temp", 3400).$(Os.currentTimeNanos());
    sender.flush();
```

### Sending messages from Unix shell

It is very easy to send metrics from shell. `bash` has specific syntax to send udp packets, for example:
```shell script

echo "readings,city=London,make=Omron temperature=23.5,humidity=0.343 1465839830100400000" > /dev/udp/127.0.0.1/9009
```

You could also use net cat method, which works from all shells:

```shell script

echo "readings,city=London,make=Omron temperature=23.5,humidity=0.343 1465839830100400000" | nc -u 127.0.0.1 9009
```

## Java

### Compiling SQL in Java

#### Overview
JAVA users can use the `SqlCompiler` to run SQL queries like they would do in the web console for example.

> Note this can be used for any SQL query. This means you can use this with any supported SQL statement. For example 
> [INSERT](sqlINSERT.md) or [COPY](sqlCOPY.md) to write data, [CREATE TABLE](sqlCREATE.md) or [ALTER TABLE](sqlALTER.md)
>to administer tables, and [SELECT](sqlSELECT.md) to query data.

#### Syntax
```java
CairoConfiguration configuration = new DefaultCairoConfiguration("/tmp/my_database");
BindVariableService bindVariableService = new BindVariableService();
try (CairoEngine engine = new CairoEngine(configuration)) {
    try (SqlCompiler compiler = new SqlCompiler(engine, configuration)) {
        compiler.compile(
            "YOUR_SQL_HERE"
        );
    }
}
```

`configuration` holds various settings that can be overridden via a subclass. 
Most importantly configuration is bound to the database root - directory where table sub-directories will be created.