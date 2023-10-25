# raft-java
The project described is a distributed Key-Value storage system with the following key features and functionalities:

1. **High Availability and Strong Consistency**: The system implements the Raft consensus algorithm to ensure strong data consistency and automatically select a new leader in case of node failures to maintain high availability.

2. **Raft Algorithm Implementation**: The project is built upon the Raft paper, implementing core Raft algorithm features, including leader election, log replication, and snapshot updates, ensuring reliable data replication and persistence.

3. **Multi-Raft Architecture**: Data is partitioned into multiple shards, managed using a Multi-Raft architecture that allows migration between multiple Raft groups to achieve dynamic data distribution and load balancing.

4. **Linear Consistency for Reads and Writes**: The system offers a Key-Value storage interface that supports linear consistency for reads and writes, ensuring clients access the latest data copy to meet strong consistency requirements.

5. **Asynchronous Optimization**: The system employs asynchronous techniques such as Apply, ReadIndex, FollowerRead, and Prevote to enhance performance. This means the system can provide higher throughput without sacrificing consistency.

6. **MVCC Multi-Version Concurrency Control**: The system implements Multi-Version Concurrency Control (MVCC) that allows multiple transactions to access data simultaneously without conflicts. This is based on the Precolator model, supporting high-performance distributed transactions to ensure data consistency and concurrency.

In summary, this project is a robust distributed Key-Value storage system with high availability, strong consistency, high performance, partition management, and linear consistency for reads and writes. It utilizes the Raft algorithm as its core, combines asynchronous optimizations, and employs MVCC multi-version control, providing a reliable and high-performance data storage solution for distributed applications.

# Supported Features

* Leader election
* Log replication
* Snapshot
* Dynamic membership changes

## Quick Start
To deploy a 3-instance Raft cluster on a local machine, run the following script:<br>
cd raft-java-example && sh deploy.sh <br>
This script will deploy three instances, example1, example2, and example3, in the raft-java-example/env directory;<br>
It will also create a client directory for testing Raft cluster read and write operations.<br>
After successful deployment, you can test write operations using the following script:
cd env/client <br>
./bin/run_client.sh "list://127.0.0.1:8051,127.0.0.1:8052,127.0.0.1:8053" hello world <br>
To test read operations, use the following command:<br>
./bin/run_client.sh "list://127.0.0.1:8051,127.0.0.1:8052,127.0.0.1:8053" hello


# Usage
Below are instructions on how to use the raft-java dependency library in your code to implement a distributed storage system.
## Configure Dependency (Not yet published to the Maven Central Repository, needs manual installation)
```
<dependency>
    <groupId>com.github.raftimpl.raft</groupId>
    <artifactId>raft-java-core</artifactId>
    <version>1.9.0</version>
</dependency>
```

## Define Data Write and Read Interfaces
```protobuf
message SetRequest {
    string key = 1;
    string value = 2;
}
message SetResponse {
    bool success = 1;
}
message GetRequest {
    string key = 1;
}
message GetResponse {
    string value = 1;
}
```
```java
public interface ExampleService {
    Example.SetResponse set(Example.SetRequest request);
    Example.GetResponse get(Example.GetRequest request);
}
```

## Server-Side Usage
1. Implement the StateMachine interface.
```java
// This interface is primarily used by Raft internally
public interface StateMachine {
    /**
     * Take a snapshot of the data in the state machine, called periodically by each node locally.
     * @param snapshotDir The output directory for snapshot data.
     */
    void writeSnapshot(String snapshotDir);

    /**
     * Read a snapshot into the state machine, called when a node starts.
     * @param snapshotDir The directory containing snapshot data.
     */
    void readSnapshot(String snapshotDir);

    /**
     * Apply data to the state machine.
     * @param dataBytes The binary data.
     */
    void apply(byte[] dataBytes);
}

```

2. Implement data write and read interfaces.
```
// The ExampleService implementation should include the following members
private RaftNode raftNode;
private ExampleStateMachine stateMachine;
```
```
// Main logic for data write
byte[] data = request.toByteArray();
// Synchronously replicate data to the Raft cluster
boolean success = raftNode.replicate(data, Raft.EntryType.ENTRY_TYPE_DATA);
Example.SetResponse response = Example.SetResponse.newBuilder().setSuccess(success).build();

```
```
// Main logic for data read, implemented by the specific application state machine
Example.GetResponse response = stateMachine.get(request);

```

3.Server startup logic.
```
// Initialize the RPCServer
RPCServer server = new RPCServer(localServer.getEndPoint().getPort());
// Apply the state machine
ExampleStateMachine stateMachine = new ExampleStateMachine();
// Set Raft options, for example:
RaftOptions.snapshotMinLogSize = 10 * 1024;
RaftOptions.snapshotPeriodSeconds = 30;
RaftOptions.maxSegmentFileSize = 1024 * 1024;
// Initialize the RaftNode
RaftNode raftNode = new RaftNode(serverList, localServer, stateMachine);
// Register services for Raft node-to-node communication
RaftConsensusService raftConsensusService = new RaftConsensusServiceImpl(raftNode);
server.registerService(raftConsensusService);
// Register Raft services for client calls
RaftClientService raftClientService = new RaftClientServiceImpl(raftNode);
server.registerService(raftClientService);
// Register services provided by your application
ExampleService exampleService = new ExampleServiceImpl(raftNode, stateMachine);
server.registerService(exampleService);
// Start the RPCServer and initialize the Raft node
server.start();
raftNode.init();

```
