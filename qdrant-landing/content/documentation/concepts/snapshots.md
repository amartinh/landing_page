---
title: Snapshots
weight: 110
aliases:
  - ../snapshots
---

# Snapshots

*Available as of v0.8.4*

Snapshots are created individually for each node in a collection, resulting in a `tar` archive file that holds the data needed to restore that collection at the snapshot time. In a distributed setup, when you have multiple nodes in your cluster, you must create snapshots for each node separately when dealing with a single collection. 
Therefore, in a distributed deployment, a snapshot of a collection consists of multiple `tar` archive files, with one file for each collection node.

This feature can be used to archive data or easily replicate an existing deployment. For a step-by-step guide on how to use snapshots, see our [tutorial](/documentation/tutorials/create-snapshot/).

## Store snapshots

The target directory used to store generated snapshots is controlled through the [configuration](../../guides/configuration) or using the ENV variable: `QDRANT__STORAGE__SNAPSHOTS_PATH=./snapshots`.

You can set the snapshots storage directory from the [config.yaml](https://github.com/qdrant/qdrant/blob/master/config/config.yaml) file. If no value is given, default is `./snapshots`.

```yaml
storage:
  # Specify where you want to store snapshots.
  snapshots_path: ./snapshots
```

*Available as of v1.3.0*

While a snapshot is being created, temporary files are by default placed in the configured storage directory. 
This location may have limited capacity or be on a slow network-attached disk. You may specify a separate location for temporary files:

```yaml
storage:
  # Where to store temporary files
  temp_path: /tmp
```

## Create snapshot

<aside role="status">If you work with a distributed deployment, you have to create snapshots for each node separately. A single snapshot will contain only the data coming from the node on which the snapshot was created.</aside>

To create a new snapshot for an existing collection:

```http
POST /collections/{collection_name}/snapshots
```

```python
from qdrant_client import QdrantClient

client = QdrantClient("localhost", port=6333)

client.create_snapshot(collection_name="{collection_name}")
```

```typescript
import { QdrantClient } from "@qdrant/js-client-rest";

const client = new QdrantClient({ host: "localhost", port: 6333 });

client.createSnapshot("{collection_name}");
```

```rust
use qdrant_client::client::QdrantClient;

let client = QdrantClient::from_url("http://localhost:6334").build()?;

client.create_snapshot("{collection_name}").await?;
```

```java
import io.qdrant.client.QdrantClient;
import io.qdrant.client.QdrantGrpcClient;

QdrantClient client =
      new QdrantClient(QdrantGrpcClient.newBuilder("localhost", 6334, false).build());

client.createSnapshotAsync("{collection_name}").get();
```

```csharp
using Qdrant.Client;

var client = new QdrantClient("localhost", 6334);

await client.CreateSnapshotAsync("{collection_name}");
```

This is a synchronous operation for which a `tar` archive file will be generated into the `snapshot_path`.

### Delete snapshot

*Available as of v1.0.0*

```http
DELETE /collections/{collection_name}/snapshots/{snapshot_name}
```

```python
from qdrant_client import QdrantClient

client = QdrantClient("localhost", port=6333)

client.delete_snapshot(
    collection_name="{collection_name}", snapshot_name="{snapshot_name}"
)
```

```typescript
import { QdrantClient } from "@qdrant/js-client-rest";

const client = new QdrantClient({ host: "localhost", port: 6333 });

client.deleteSnapshot("{collection_name}", "{snapshot_name}");
```

```rust
use qdrant_client::client::QdrantClient;

let client = QdrantClient::from_url("http://localhost:6334").build()?;

client.delete_snapshot("{collection_name}", "{snapshot_name}").await?;
```

```java
import io.qdrant.client.QdrantClient;
import io.qdrant.client.QdrantGrpcClient;

QdrantClient client =
    new QdrantClient(QdrantGrpcClient.newBuilder("localhost", 6334, false).build());

client.deleteSnapshotAsync("{collection_name}", "{snapshot_name}").get();
```

```csharp
using Qdrant.Client;

var client = new QdrantClient("localhost", 6334);

await client.DeleteSnapshotAsync(collectionName: "{collection_name}", snapshotName: "{snapshot_name}");
```

## List snapshot

List of snapshots for a collection:

```http
GET /collections/{collection_name}/snapshots
```

```python
from qdrant_client import QdrantClient

client = QdrantClient("localhost", port=6333)

client.list_snapshots(collection_name="{collection_name}")
```

```typescript
import { QdrantClient } from "@qdrant/js-client-rest";

const client = new QdrantClient({ host: "localhost", port: 6333 });

client.listSnapshots("{collection_name}");
```

```rust
use qdrant_client::client::QdrantClient;

let client = QdrantClient::from_url("http://localhost:6334").build()?;

client.list_snapshots("{collection_name}").await?;
```

```java
import io.qdrant.client.QdrantClient;
import io.qdrant.client.QdrantGrpcClient;

QdrantClient client =
    new QdrantClient(QdrantGrpcClient.newBuilder("localhost", 6334, false).build());

client.listSnapshotAsync("{collection_name}").get();
```

```csharp
using Qdrant.Client;

var client = new QdrantClient("localhost", 6334);

await client.ListSnapshotsAsync("{collection_name}");
```

## Retrieve snapshot

<aside role="status">Only available through the REST API for the time being.</aside>

To download a specified snapshot from a collection as a file:

```http
GET /collections/{collection_name}/snapshots/{snapshot_name}
```

## Restore snapshot

<aside role="status">Restoration of snapshots is limited to Qdrant clusters sharing the same minor version. For instance, a snapshot captured in v1.4.1 is exclusive to restoration in clusters of version v1.4.x, where x is equal to or greater than 1.</aside>

There is a difference in recovering snapshots in single-deployment node and distributed deployment mode.

### Recover during start-up

<aside role="alert">This method cannot be used in a cluster deployment.</aside>

Single deployment is simpler, you can recover any collection on the start-up and it will be immediately available in the service.
Restoring snapshots is done through the Qdrant CLI at startup time.

The main entry point is the `--snapshot` argument which accepts a list of pairs `<snapshot_file_path>:<target_collection_name>`

For example:

```bash
./qdrant --snapshot /snapshots/test-collection-archive.snapshot:test-collection --snapshot /snapshots/test-collection-archive.snapshot:test-copy-collection 
```

The target collection **must** be absent otherwise the program will exit with an error.

If you wish instead to overwrite an existing collection, use the `--force_snapshot` flag with caution.

### Recover via API

*Available as of v0.11.3*

<aside role="status">You can use this method for both single-node and cluster setups.</aside>

Recovering in cluster mode is more sophisticated, as Qdrant should maintain consistency across peers even during the recovery process.
As the information about created collections is stored in the consensus, even a newly attached cluster node will automatically create collections.
Recovering non-existing collections with snapshots won't make this collection known to the consensus.

<aside role="status">It is recommended to explicitly set a <a href="#snapshot-priority">snapshot priority</a> during recovery to prevent unexpected results.</aside>

To recover snapshot via API one can use snapshot recovery endpoint:

```http
PUT /collections/{collection_name}/snapshots/recover
{
  "location": "http://qdrant-node-1:6333/collections/{collection_name}/snapshots/snapshot-2022-10-10.shapshot"
}
```

```python
from qdrant_client import QdrantClient

client = QdrantClient("qdrant-node-2", port=6333)

client.recover_snapshot(
    "{collection_name}",
    "http://qdrant-node-1:6333/collections/collection_name/snapshots/snapshot-2022-10-10.shapshot",
)
```

```typescript
import { QdrantClient } from "@qdrant/js-client-rest";

const client = new QdrantClient({ host: "localhost", port: 6333 });

client.recoverSnapshot("{collection_name}", {
  location: "http://qdrant-node-1:6333/collections/collection_name/snapshots/snapshot-2022-10-10.shapshot",
});
```

The recovery snapshot can also be uploaded as a file to the Qdrant server:

```bash
curl -X POST 'http://qdrant-node-1:6333/collections/collection_name/snapshots/upload' \
    -H 'Content-Type:multipart/form-data' \
    -F 'snapshot=@/path/to/snapshot-2022-10-10.shapshot'
```

Qdrant will extract shard data from the snapshot and properly register shards in the cluster.
If there are other active replicas of the recovered shards in the cluster, Qdrant will replicate them to the newly recovered node by default to maintain data consistency.

### Snapshot priority

When recovering a snapshot, you can specify what source of data is prioritized
during recovery. It is important because different priorities can give very
different end results. The default priority is probably not what you expect.

The available snapshot recovery priorities are:

- `replica`: _(default)_ prefer existing data over the snapshot.
- `snapshot`: prefer snapshot data over existing data.
- `no_sync`: restore snapshot without any additional synchronization.

To recover a new collection from a snapshot on a Qdrant cluster, you need to set
the `snapshot` priority. With `snapshot` priority, all data from the snapshot
will be recovered onto the cluster. With `replica` priority _(default)_, you'd
end up with an empty collection because the collection on the cluster did not
contain any points and that source was preferred.

`no_sync` is for specialized use cases and is not commonly used. It allows
managing shards and transferring shards between clusters manually without any
additional synchronization. Using it incorrectly will leave your cluster in a
broken state.

To recover from an URL you specify a request parameter:

```http
PUT /collections/{collection_name}/snapshots/recover
{
  "location": "http://qdrant-node-1:6333/collections/{collection_name}/snapshots/snapshot-2022-10-10.shapshot",
  "priority": "snapshot"
}
```

```python
from qdrant_client import QdrantClient, models

client = QdrantClient("qdrant-node-2", port=6333)

client.recover_snapshot(
    "{collection_name}",
    "http://qdrant-node-1:6333/collections/collection_name/snapshots/snapshot-2022-10-10.shapshot",
    priority=models.SnapshotPriority.SNAPSHOT,
)
```

```typescript
import { QdrantClient } from "@qdrant/js-client-rest";

const client = new QdrantClient({ host: "localhost", port: 6333 });

client.recoverSnapshot("{collection_name}", {
  location: "http://qdrant-node-1:6333/collections/collection_name/snapshots/snapshot-2022-10-10.shapshot",
  priority: "snapshot"
});
```

To upload a multipart file you specify it as URL parameter:

```bash
curl -X POST 'http://qdrant-node-1:6333/collections/collection_name/snapshots/upload?priority=snapshot' \
    -H 'Content-Type:multipart/form-data' \
    -F 'snapshot=@/path/to/snapshot-2022-10-10.shapshot'
```

## Snapshots for the whole storage

*Available as of v0.8.5*

Sometimes it might be handy to create snapshot not just for a single collection, but for the whole storage, including collection aliases.
Qdrant provides a dedicated API for that as well. It is similar to collection-level snapshots, but does not require `collecton_name`:

### Create full storage snapshot

```http
POST /snapshots
```

```python
from qdrant_client import QdrantClient

client = QdrantClient("localhost", port=6333)

client.create_full_snapshot()
```

```typescript
import { QdrantClient } from "@qdrant/js-client-rest";

const client = new QdrantClient({ host: "localhost", port: 6333 });

client.createFullSnapshot();
```

```rust
use qdrant_client::client::QdrantClient;

let client = QdrantClient::from_url("http://localhost:6334").build()?;

client.create_full_snapshot().await?;
```

```java
import io.qdrant.client.QdrantClient;
import io.qdrant.client.QdrantGrpcClient;

QdrantClient client =
    new QdrantClient(QdrantGrpcClient.newBuilder("localhost", 6334, false).build());

client.createFullSnapshotAsync().get();
```

```csharp
using Qdrant.Client;

var client = new QdrantClient("localhost", 6334);

await client.CreateFullSnapshotAsync();
```

### Delete full storage snapshot

*Available as of v1.0.0*

```http
DELETE /snapshots/{snapshot_name}
```

```python
from qdrant_client import QdrantClient

client = QdrantClient("localhost", port=6333)

client.delete_full_snapshot(snapshot_name="{snapshot_name}")
```

```typescript
import { QdrantClient } from "@qdrant/js-client-rest";

const client = new QdrantClient({ host: "localhost", port: 6333 });

client.deleteFullSnapshot("{snapshot_name}");
```

```rust
use qdrant_client::client::QdrantClient;

let client = QdrantClient::from_url("http://localhost:6334").build()?;

client.delete_full_snapshot("{snapshot_name}").await?;
```

```java
import io.qdrant.client.QdrantClient;
import io.qdrant.client.QdrantGrpcClient;

QdrantClient client =
    new QdrantClient(QdrantGrpcClient.newBuilder("localhost", 6334, false).build());

client.deleteFullSnapshotAsync("{snapshot_name}").get();
```

```csharp
using Qdrant.Client;

var client = new QdrantClient("localhost", 6334);

await client.DeleteFullSnapshotAsync("{snapshot_name}");
```

### List full storage snapshots

```http
GET /snapshots
```

```python
from qdrant_client import QdrantClient

client = QdrantClient("localhost", port=6333)

client.list_full_snapshots()
```

```typescript
import { QdrantClient } from "@qdrant/js-client-rest";

const client = new QdrantClient({ host: "localhost", port: 6333 });

client.listFullSnapshots();
```

```rust
use qdrant_client::client::QdrantClient;

let client = QdrantClient::from_url("http://localhost:6334").build()?;

client.list_full_snapshots().await?;
```

```java
import io.qdrant.client.QdrantClient;
import io.qdrant.client.QdrantGrpcClient;

QdrantClient client =
    new QdrantClient(QdrantGrpcClient.newBuilder("localhost", 6334, false).build());

client.listFullSnapshotAsync().get();
```

```csharp
using Qdrant.Client;

var client = new QdrantClient("localhost", 6334);

await client.ListFullSnapshotsAsync();
```

### Download full storage snapshot

<aside role="status">Only available through the REST API for the time being.</aside>

```http
GET /snapshots/{snapshot_name}
```

## Restore full storage snapshot

Restoring snapshots is done through the Qdrant CLI at startup time.

For example:

```bash
./qdrant --storage-snapshot /snapshots/full-snapshot-2022-07-18-11-20-51.snapshot 
```
