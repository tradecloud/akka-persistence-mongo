# An Akka Persistence Plugin for Mongo

[![Build Status](https://travis-ci.org/tradecloud/akka-persistence-mongo.svg?branch=master)](https://travis-ci.org/tradecloud/akka-persistence-mongo) [![Maven Central](https://maven-badges.herokuapp.com/maven-central/nl.tradecloud/akka-persistence-mongo_2.12/badge.svg)](https://maven-badges.herokuapp.com/maven-central/nl.tradecloud/akka-persistence-mongo_2.12) [![License](http://img.shields.io/:license-mit-blue.svg)](http://doge.mit-license.org)

A replicated [Akka Persistence](http://doc.akka.io/docs/akka/current/scala/persistence.html) journal backed by [MongoDB Casbah](http://mongodb.github.io/casbah/).

## Prerequisites

### Release

| Technology | Version                          |
| :--------: | -------------------------------- |
| Scala      | 2.11.7, 2.12.4 - Cross Compiled  |
| Akka       | 2.5.9 or higher                  |
| Mongo      | 2.6.x or higher                  |

## Installation

### SBT

<br/>It can then be included as dependency:

```scala
libraryDependencies += "nl.tradecloud" %% "akka-persistence-mongo" % "1.0.1"
```


### Build Locally

You can build and install the plugin to your local Ivy cache. This requires sbt 1.0.4 or above.

```scala
sbt publishLocal
```


## Mongo Specific Details

Both the [Journal](#journal-configuration) and [Snapshot](#snapshot-configuration) configurations use the following Mongo components for connection management and write guarantees.

### Mongo Connection String URI

The [Mongo Connection String URI Format](http://docs.mongodb.org/manual/reference/connection-string/) is used for establishing a connection to Mongo. Please **note** that while some of the components of the connection string are **[optional]** from a Mongo perspective, they are **[required]** for the Journal and Snapshot to function properly. Below are the required and optional components.

#### Standard URI connection scheme

```scala
mongodb://[username:password@]host1[:port1][,host2[:port2],...[,hostN[:portN]]][/[database][.collection][?options]]
```

#### Required

| Component     | Description                                                                                 |
| :------------ | ------------------------------------------------------------------------------------------- |
| `mongodb:// ` | The prefix to identify that this is a string in the standard connection format.             |
| `host1`       | A server address to connect to. It is either a hostname, IP address, or UNIX domain socket. |
| `/database  ` | The name of the database to use.                                                            |
| `.collection` | The name of the collection to use.                                                          |

#### Optional

| Component | Description |
| :-------- | ----------- |
| `username:password@` | If specified, the client will attempt to log in to the specific database using these credentials after connecting to the mongod instance. |
| `:port1` | The default value is :27017 if not specified.                                               |
| `hostN` | You can specify as many hosts as necessary. You would specify multiple hosts, for example, for connections to replica sets. |
| `:portN` | The default value is :27017 if not specified. |
| `?options` | Connection specific options. See [Connection String Options](http://docs.mongodb.org/manual/reference/connection-string/#connections-connection-options) for a full description of these options. |

### Mongo Write Concern

The [Write Concern Specification](https://docs.mongodb.org/manual/reference/write-concern/) describes the guarantee that MongoDB provides when reporting on the success of a write operation. The strength of the write concern determine the level of guarantee. When a `PersistentActor's` `persist` or `persistAsync` method completes successfully, a plugin must ensure the `message` or `snapshot` has persisted to the store. As a result, this plugin implementation enforces Mongo [Journaling](https://docs.mongodb.org/manual/core/journaling/) on all write concerns and requires all mongo instance(s) to enable journaling.

| Options | Description |
| :------------ | ----------- |
| `woption`     | The `woption` requests acknowledgment that the write operation has propagated to a specified number of mongod instances or to mongod instances with specified tags.  Mongo's `wOption` can be either an `Integer` or `String`, and this plugin implementation supports both with `woption`. <br/><br/>If `woption` is an `Integer`, then the write concern requests acknowledgment that the write operation has propagated to the specified number of mongod instances. Note: The `woption` cannot be set to zero. <br/><br/>If `woption` is a `String`, then the value can be either `"majority"` or a `tag set name`. If the value is `"majority"` then the write concern requests acknowledgment that write operations have propagated to the majority of voting nodes. If the value is a `tag set name`, the write concern requests acknowledgment that the write operations have propagated to a replica set member with the specified tag. The default value is an `Integer` value of `1`. |
| `wtimeout`    | This option specifies a time limit, in `milliseconds`, for the write concern. If you do not specify the `wtimeout` option, and the level of write concern is unachievable, the write operation will **block** indefinitely. Specifying a `wtimeout` value of `0` is equivalent to a write concern without the `wTimeout` option. The default value is `10000` (10 seconds). |

## Journal Configuration

### Activation

To activate the journal feature of the plugin, add the following line to your Akka `application.conf`. This will run the journal with its default settings.

```scala
akka.persistence.journal.plugin = "casbah-journal"
```

### Connection

The default `mongo-url` is a `string` with a value of:

```scala
casbah-journal.mongo-url = "mongodb://localhost:27017/store.messages"
```

<br/>This value can be changed in the `application.conf` with the following key:

```scala
casbah-journal.mongo-url

# Example
casbah-journal.mongo-url = "mongodb://localhost:27017/employee.events"
```

<br/>See the [Mongo Connection String URI](#mongo-connection-string-uri) section of this document for more information.

### Write Concern

The default `woption` is an `Integer` with a value of:

```scala
casbah-journal.woption = 1
```

<br/>This value can be changed in the `application.conf` with the following key:

```scala
casbah-journal.woption

# Example
casbah-journal.woption = "majority"
```

### Write Concern Timeout

The default `wtimeout` is an `Long` in milliseconds with a value of:

```scala
casbah-journal.wtimeout = 10000
```

<br/>This value can be changed in the `application.conf` with the following key:

```scala
casbah-journal.wtimeout

# Example
casbah-journal.wtimeout = 5000
```

<br/>See the [Mongo Write Concern](#mongo-write-concern) section of this document for more information.

### Reject Non-Serializable Objects

This plugin supports the rejection of `non-serializable` journal messages. If `reject-non-serializable-objects` is set to `true` and a message is received who's payload cannot be `serialized` then it is rejected.

If set to `false` (the default value) then the `non-serializable` payload of the message becomes `Array.empty[Byte]` and is persisted.

the default `reject-non-serializable-objects` is a `Boolean` with the value of:

```scala
casbah-journal.reject-non-serializable-objects = false
```

<br/>This value can be changed in the `application.conf` with the following key:

```scala
casbah-journal.reject-non-serializable-objects

# Example
casbah-journal.reject-non-serializable-objects = true
```

## Snapshot Configuration

### Activation

To activate the snapshot feature of the plugin, add the following line to your Akka `application.conf`. This will run the snapshot-store with its default settings.

```scala
akka.persistence.snapshot-store.plugin = "casbah-snapshot"
```

### Connection

The default `mongo-url` is a `string` with a value of:

```scala
casbah-snapshot.mongo-url = "mongodb://localhost:27017/store.snapshots"
```

<br/>This value can be changed in the `application.conf` with the following key:

```scala
casbah-snapshot.mongo-url

# Example
casbah-snapshot.mongo-url = "mongodb://localhost:27017/employee.snapshots"
```

<br/>See the [Mongo Connection String URI](#mongo-connection-string-uri) section of this document for more information.

### Write Concern

The default `woption` is an `Integer` with a value of:

```scala
casbah-snapshot.woption = 1
```

<br/>This value can be changed in the `application.conf` with the following key:

```scala
casbah-snapshot.woption

# Example
casbah-snapshot.woption = "majority"
```

### Write Concern Timeout

The default `wtimeout` is an `Long` in milliseconds with a value of:

```scala
casbah-snapshot.wtimeout = 10000
```

<br/>This value can be changed in the `application.conf` with the following key:

```scala
casbah-snapshot.wtimeout

# Example
casbah-snapshot.wtimeout = 5000
```

<br/>See the [Mongo Write Concern](#mongo-write-concern) section of this document for more information.

### Snapshot Load Attempts

The snapshot feature of the plugin allows for the selection of the youngest of `{n}` snapshots that match an upper bound specified by configuration. This helps where a snapshot may not have persisted correctly because of a JVM crash. As a result an attempt to load the snapshot may fail but an older may succeed.

The default `load-attempts` is an `Integer` with a value of:

```scala
casbah-snapshot.load-attempts = 3
```

<br/>This value can be changed in the `application.conf` with the following key:

```scala
casbah-snapshot.load-attempts

# Example
casbah-snapshot.load-attempts = 5
```

## Status

* All operations required by the Akka Persistence [journal plugin API](http://doc.akka.io/docs/akka/current/scala/persistence.html#Journal_plugin_API) are supported.
* All operations required by the Akka Persistence [Snapshot store plugin API](http://doc.akka.io/docs/akka/current/scala/persistence.html#Snapshot_store_plugin_API) are supported.
* Tested against [Plugin TCK](http://doc.akka.io/docs/akka/current/scala/persistence.html#Plugin_TCK).
* Plugin uses [Asynchronous Casbah Driver](http://mongodb.github.io/casbah/3.1/)
* Message writes are batched to optimize throughput.

## Performance

Minimal performance testing is included against a **native** instance. In general the journal will persist around 8,000 to 10,000 messages per second.

## Sample Applications

The [sample applications](https://github.com/ironfish/akka-persistence-mongo-samples) are now located in their own repository.

## Change Log

### 1.0.0-SNAPSHOT

* Upgrade `Akka` 2.4.1.
* Upgrade `Casbah` to `Async` driver 3.1.
* Supports latest [Plugin TCK](http://doc.akka.io/docs/akka/current/scala/persistence.html#Plugin_TCK).

### 0.7.6

* Upgrade `sbt` to 0.13.8.
* Upgrade `Scala` cross-compilation to 2.10.5 & 2.11.7.
* Upgrade `Akka` to 2.3.12.

### 0.7.5

* Upgrade `sbt` to 0.13.7.
* Upgrade `Scala` cross-compilation to 2.10.4 & 2.11.4.
* Upgrade `Akka` to 2.3.7.
* Examples moved to their own [repository](https://github.com/ironfish/akka-persistence-mongo-samples).
* Removed `logback.xml` in `akka-persistence-mongo-casbah` as it was not needed.
* Added `pomOnly()` resolution to `casbah` dependency, fixes #63.

### 0.7.4

* First release version to Maven Central Releases.
* Upgrade `Sbt` to 0.13.5.
* Upgrade `Scala` cross-compilation to 2.10.4 & 2.11.2.
* Upgrade `Akka` to 2.3.5.
* Added exception if `/database` or `.collection` are not accessible upon boot. Thanks @Fristi.
* Modified snapshot feature for custom serialization support. Thanks @remcobeckers.

### 0.7.3-SNAPSHOT

* Upgrade `Sbt` to 0.13.4.
* Upgrade `Scala` cross-compilation to 2.10.4 & 2.11.2.
* Upgrade `Akka` to 2.3.4.
* `@deprecated` write confirmations, `CasbahJournal.writeConfirmations`, in favor of `AtLeastOnceDelivery`.
* `@deprecated` delete messages, `CasbahJournal.deleteMessages`, per akka-persistence documentation.

## Author / Maintainer

* [Duncan DeVore (@ironfish)](https://github.com/ironfish/)

## Contributors

* [Heiko Seeberger (@hseeberger)](https://github.com/hseeberger)
* [Sean Walsh (@SeanWalshEsq)](https://github.com/sean-walsh/)
* [Al Iacovella](https://github.com/aiacovella/)
* [Remco Beckers](https://github.com/remcobeckers)
* [Mark Fristi](https://github.com/Fristi)
