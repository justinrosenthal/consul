---
layout: "docs"
page_title: "Bootstrapping"
sidebar_current: "docs-guides-bootstrapping"
description: |-
  An agent can run in both client and server mode. Server nodes are responsible for running the consensus protocol and storing the cluster state. Before a Consul cluster can begin to service requests, a server node must be elected leader. Thus, the first nodes that are started are generally the server nodes. Bootstrapping is the process of joining these server nodes into a cluster. 
---

# Bootstrapping a Datacenter

An agent can run in both client and server mode. Server nodes are responsible for running the
[consensus protocol](/docs/internals/consensus.html) and storing the cluster state.
The client nodes are mostly stateless and rely heavily on the server nodes.

Before a Consul cluster can begin to service requests, a server node must be elected leader.
Thus, the first nodes that are started are generally the server nodes. Bootstrapping is the process
of joining these server nodes into a cluster.

The recommended way to bootstrap is to use the [`-bootstrap-expect`](/docs/agent/options.html#_bootstrap_expect)
configuration option. This option informs Consul of the expected number of
server nodes and automatically bootstraps when that many servers are available. To prevent
inconsistencies and split-brain situations (that is, clusters where multiple servers consider
themselves leader), all servers should either specify the same value for
[`-bootstrap-expect`](/docs/agent/options.html#_bootstrap_expect)
or specify no value at all. Only servers that specify a value will attempt to bootstrap the cluster.

We recommend 3 or 5 total servers per datacenter. A single server deployment is _**highly**_ discouraged
as data loss is inevitable in a failure scenario. Please refer to the
[deployment table](/docs/internals/consensus.html#toc_4) for more detail.

Suppose we are starting a 3 server cluster. We can start `Node A`, `Node B`, and `Node C` with each
providing the `-bootstrap-expect 3` flag. Once the nodes are started, you should see a message like:

```text
[WARN] raft: EnableSingleNode disabled, and no known peers. Aborting election.
```

This indicates that the nodes are expecting 2 peers but none are known yet. To provent a split-brain
scenario, the servers will not elect themselves leader.

To trigger leader election, we must join these machines together. Since a join operation is symmetric,
it does not matter which node initiates it.

From any node, you can do the following:

```text
$ consul join <Node A Address> <Node B Address> <Node C Address>
Successfully joined cluster by contacting 3 nodes.
```

Once the join is successful, one of the nodes will output something like:

```text
[INFO] consul: adding server foo (Addr: 127.0.0.2:8300) (DC: dc1)
[INFO] consul: adding server bar (Addr: 127.0.0.1:8300) (DC: dc1)
[INFO] consul: Attempting bootstrap with nodes: [127.0.0.3:8300 127.0.0.2:8300 127.0.0.1:8300]
    ...
[INFO] consul: cluster leadership acquired
```

As a sanity check, the [`consul info`](/docs/commands/info.html) command is a useful tool. It can be used to
verify `raft.num_peers` is now 2, and you can view the latest log index under `raft.last_log_index`.
When running [`consul info`](/docs/commands/info.html) on the followers, you should see `raft.last_log_index`
converge to the same value once the leader begins replication. That value represents the last
log entry that has been stored on disk.

Now that the servers are all started and replicating to each other, all the remaining
clients can be joined. Clients are much easier as they can join against any existing node.
All nodes participate in a gossip protocol to perform basic discovery, so once joined to any
member of the cluster, new clients will automatically find the servers and register themselves.

Note: it is not strictly necessary to start the server nodes before the clients; however, most
operations will fail until the servers are available.

## Manual Bootstrapping

In versions of Consul prior to 0.4, bootstrapping was a more manual process. For details on
using the [`-bootstrap`](/docs/agent/options.html#_bootstrap) flag directly, see the
[manual bootstrapping guide](/docs/guides/manual-bootstrap.html).

Manual bootstrapping is not recommended as it is more error-prone than automatic bootstrapping
with [`-bootstrap-expect`](/docs/agent/options.html#_bootstrap_expect).
