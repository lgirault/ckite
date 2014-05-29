CKite [![Build Status](https://api.travis-ci.org/pablosmedina/ckite.png)](https://travis-ci.org/pablosmedina/ckite)
=====

## Overview

A __JVM__ implementation of the [Raft distributed consensus algorithm](http://raftconsensus.github.io/) written in Scala. CKite is a library with an easy to use api for both __Java and Scala__ to be used by distributed applications needing consensus agreement. 

## Status

CKite covers all the major topics of Raft including leader election, log replication, log compaction and cluster membership changes. Checkout the latest __Release 0.1.4__ following the instructions detailed below to start playing with it. 
Performance tests will be included soon.


## Features

* Leader Election
* Log Replication
* Cluster Membership Changes
* Log Compaction
* Finagle based RPC between members


## Getting started (Scala)

#### SBT settings

The latest release 0.1.3 is in Maven central. Add the following sbt dependency to your project settings:

```scala
libraryDependencies += "io.ckite" % "ckite" % "0.1.4"
```

## Getting started (Java)

#### Maven settings

Add the following maven dependency to your pom.xml:

```xml
<dependency>
	<groupId>io.ckite</groupId>
	<artifactId>ckite</artifactId>
	<version>0.1.4</version>
</dependency>
```


## Example

#### 1) Define a StateMachine and its commands
```scala
//KVStore is a distributed Map accepting Puts and Gets
class KVStore extends StateMachine {

  val map = new ConcurrentHashMap[String, String]()
  @volatile
  var lastIndex: Long = 0

  //called when a consensus has been reached for a WriteCommand
  //index associated to the write is provided to implement your own persistent semantics. see lastAppliedIndex
  def applyWrite = {
    case (index, Put(key: String, value: String)) => {
      map.put(key, value);
      lastIndex = index
      value
    }
  }
  
  //CKite needs to know the last applied write on restarts for log replay
  //If no persistence is needed then state machines can just return zero
  def lastAppliedIndex: Long = lastIndex

  //called when a read command has been received
  def applyRead = {
    case Get(key) => {
      map.get(key)
    }
  }

  //called during Log replay on startup and upon installSnapshot requests
  def deserialize(snapshotBytes: ByteBuffer) = {
	//some deserialization mechanism
  }
 
  //called when Log compaction is required
  def serialize: ByteBuffer = {
	//some serialization mechanism
  }

}

//WriteCommands are replicated under Raft rules
case class Put(key: String, value: String) extends WriteCommand

//ReadCommands are not replicated but forwarded to the Leader
case class Get(key: String) extends ReadCommand
```

#### 2) Create a CKite instance using the builder
```scala
val ckite = CKiteBuilder().listenAddress("localhost:9091")
                          .members(Seq("localhost:9092","localhost:9093"))
                          .minElectionTimeout(1000).withMaxElectionTimeout(1500) //optional
                          .heartbeatsPeriod(250) //optional
                          .dataDir("/home/ckite/data") //dataDir for persistent state (log, terms, snapshots, etc...)
                          .stateMachine(new KVStore()) //KVStore is an implementation of the StateMachine trait
                          .build
ckite.start()
```
#### 3) Send a write command
```scala
//this Put command is forwarded to the Leader and applied under Raft rules
ckite.write(Put("key1","value1")) 
```

#### 4) Send a consistent read command
```scala
//consistent read commands are forwarded to the Leader
val value = ckite.read(Get("key1")) 
```
#### 5) Add a new Member
```scala
//as write commands, cluster membership changes are forwarded to the Leader
ckite.addMember("someHost:9094")
```

#### 6) Remove a Member
```scala
//as write commands, cluster membership changes are forwarded to the Leader
ckite.removeMember("someHost:9094")
```

#### 7) Send a local read command
```scala
//alternatively you can read from its local state machine allowing possible stale values
val value = ckite.readLocal(Get("key1")) 
```

#### 8) Check leadership
```scala
//if necessary waits for elections to end
ckite.isLeader() 
```
#### 9) Stop ckite
```scala
ckite.stop()
```

## Rest admin console

CKite exposes an admin console showing its status and useful metrics. If the rpc port is 9091 then the admin console is exposed under http://localhost:10091/status by default.

#### Leader

```yaml
{
	cluster: {
		term: 1,
		state: "Leader",
		stateInfo: {
			leaderUptime: "13.hours+4.minutes+47.seconds+214.milliseconds",
			followers: {
				localhost:9093: {
					lastHeartbeatACK: "11.milliseconds",
					nextIndex: 10
				},
				localhost:9092: {
					lastHeartbeatACK: "8.milliseconds",
					nextIndex: 10
				}
				localhost:9095: {
					lastHeartbeatACK: "10.milliseconds",
					nextIndex: 10
				}
				localhost:9094: {
					lastHeartbeatACK: "12.milliseconds",
					nextIndex: 10
				}
			}
		}
	},
	log: {
		length: 9,
		commitIndex: 9,
		lastEntry: {
			term: 1,
			index: 9,
			command: {
				key: "foo",
				value: "bar"
			}
		}
	}
}
```

#### Follower
```yaml
{
	cluster: {
		term: 1,
		state: "Follower",
		stateInfo: {
			following: "Some(localhost:9091)"
		}
	},
	log: {
		length: 9,
		commitIndex: 9,
		lastEntry: {
			term: 1,
			index: 9,
			command: {
				key: "foo",
				value: "bar"
			}
		}
	}
}
```

## Running KVStore example (3 members)

#### Run Member 1
```bash
sbt run -Dport=9091 -Dmembers=localhost:9092,localhost:9093 -DdataDir=/tmp/ckite/member1
```
#### Run Member 2
```bash
sbt run -Dport=9092 -Dmembers=localhost:9091,localhost:9093 -DdataDir=/tmp/ckite/member2
```
#### Run Member 3
```bash
sbt run -Dport=9093 -Dmembers=localhost:9092,localhost:9091 -DdataDir=/tmp/ckite/member3
```
#### Put a key-value on the leader member (take a look at the logs for election result)
```bash
curl http://localhost:10091/put/key1/value1
```
#### Get the value of key1 replicated in member 2 
```bash
curl http://localhost:10092/get/key1
```
#### Checkout the admin console on any member to see the cluster status
```bash
curl http://localhost:10093/status
```
#### Add a new member (localhost:9094) to the Cluster
```bash
curl http://localhost:10091/changecluster/localhost:9091,localhost:9092,localhost:9093,localhost:9094
```
#### Run Member 4
```bash
sbt run -Dport=9094 -Dmembers=localhost:9092,localhost:9091,localhost:9093 -DdataDir=/tmp/ckite/member4
```

## Implementation details

  * Built in Scala.
  * [Twitter Finagle](http://twitter.github.io/finagle/).
  * [Thrift](http://thrift.apache.org/).
  * [Twitter Scrooge](http://twitter.github.io/scrooge/).


## Contributions

Feel free to contribute to CKite!. Any kind of help will be very welcome. We are happy to receive pull requests, issues, discuss implementation details, analyze the raft algorithm and whatever it makes CKite a better library. Checkout the issues. You can start from there!


## Importing the project into IntelliJ IDEA

To generate the necessary IDE config files first run the following command and then open the project as usual:

        sbt gen-idea
        
## Importing the project into Eclipse

To generate the necessary IDE config files first run the following command and then open the project as usual:

        sbt eclipse
