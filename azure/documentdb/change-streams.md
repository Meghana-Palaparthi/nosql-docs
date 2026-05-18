---
title: Change streams
description: Learn how to configure and use change streams to track the real-time changes made on targeted collections or databases.
author: avijitgupta
ms.author: avijitgupta
ms.topic: concept-article
ms.date: 05/18/2026
ai-usage: ai-generated
---

# Change streams in Azure DocumentDB

Change streams are a real-time stream of database changes that flows from your database to your application. This feature enables you to build reactive applications by subscribing to database changes, eliminating the need for continuous polling to detect changes.

> [!NOTE]
> Change stream support for multi-shard clusters is currently in preview.

## Get started

The example code initiates a change stream on the `exampleCollection` collection, continuously monitoring for any changes. When a change is detected, it retrieves the change event and prints it in JSON format.

### [Python](#tab/python)

```python
from pymongo.errors import OperationFailure, PyMongoError
from mongo_utils import get_collection, load_mongo_config, load_resume_token, ping_server, save_resume_token

def main() -> None:
    client = None
    try:
        _, _, source_collection_name, destination_collection_name = load_mongo_config()
        client, database, collection = get_collection()
        destination_collection = None
        if destination_collection_name:
            destination_collection = database[destination_collection_name]
            print(
                f"Mirroring change events from '{source_collection_name}' to '{destination_collection_name}'."
            )
        else:
            print(
                f"Watching source collection '{source_collection_name}' with publish-only mode (no destination collection configured)."
            )
        ping_server(client)

        # Try to resume from last known token
        resume_token = load_resume_token()
        if resume_token:
            print(f"Resuming from last token: {resume_token.get('_data', '')[:20]}...")
            stream = collection.watch(resume_after=resume_token)
        else:
            print("Starting fresh change stream (no prior token found).")
            stream = collection.watch()

        print("Watching for changes...")
        with stream:
            for change in stream:
                print(f"Change: {change['operationType']} on {change['ns']['coll']}")
                print(f"  Full doc: {change.get('fullDocument', {})}")

                if destination_collection is not None:
                    mirrored_event = {
                        "source": change.get("ns", {}),
                        "operationType": change.get("operationType"),
                        "documentKey": change.get("documentKey", {}),
                        "fullDocument": change.get("fullDocument", {}),
                        "resumeToken": change.get("_id", {}),
                    }
                    destination_collection.insert_one(mirrored_event)

                # Save token after each successful change
                save_resume_token(change["_id"])
                print()
    except OperationFailure as error:
        print("The change stream could not be opened on this collection.")
        print(error)
        print(
            "Check whether your Azure database deployment supports change streams for the configured aggregation properties."
        )
    except PyMongoError as error:
        print("Azure DocumentDB connection or driver error while watching for changes.")
        print(error)
    except KeyboardInterrupt:
        print("\nConsumer stopped by user.")
    finally:
        if client is not None:
            client.close()

if __name__ == "__main__":
    main()
```

### [Java](#tab/java)

```java
package com.avijit.changestreams;

import com.mongodb.MongoException;
import com.mongodb.client.ChangeStreamIterable;
import com.mongodb.client.MongoChangeStreamCursor;
import com.mongodb.client.model.changestream.ChangeStreamDocument;
import org.bson.BsonDocument;
import org.bson.Document;

/**
 * Consumer: Listens for change-stream events and persists the resume token.
 */
public final class DataConsumerApp {
    private DataConsumerApp() {
    }

    public static void main(String[] args) {
        MongoUtils.MongoContext context = null;
        try {
            context = MongoUtils.getContext();
            MongoUtils.pingServer(context.client());

            BsonDocument resumeToken = MongoUtils.loadResumeToken();
            ChangeStreamIterable<Document> watch = context.collection().watch();
            if (resumeToken != null) {
                System.out.printf("Resuming from last token: %s...%n", abbreviate(resumeToken.toJson()));
                watch = watch.resumeAfter(resumeToken);
            } else {
                System.out.println("Starting fresh change stream (no prior token found).");
            }

            System.out.println("Watching for changes...");
            try (MongoChangeStreamCursor<ChangeStreamDocument<Document>> cursor = watch.cursor()) {
                while (cursor.hasNext()) {
                    ChangeStreamDocument<Document> change = cursor.next();
                    Document fullDocument = change.getFullDocument();
                    System.out.printf("Change: %s on %s%n", change.getOperationType().getValue(), change.getNamespace().getCollectionName());
                    System.out.printf("  Full doc: %s%n%n", fullDocument == null ? "{}" : fullDocument.toJson());
                    MongoUtils.saveResumeToken(change.getResumeToken());
                }
            }
        } catch (MongoException | IllegalStateException error) {
            System.out.println("Azure DocumentDB connection or driver error while watching for changes.");
            System.out.println(error.getMessage());
        } finally {
            if (context != null) {
                context.client().close();
            }
        }
    }

    private static String abbreviate(String tokenJson) {
        int maxLength = 24;
        if (tokenJson.length() <= maxLength) {
            return tokenJson;
        }
        return tokenJson.substring(0, maxLength);
    }
}
```

### [JavaScript](#tab/javascript)

```javascript
// Open a change stream
const changeStream = db.exampleCollection.watch();

// Listen for changes
while (changeStream.hasNext()) 
    {
        const change = changeStream.next();
        printjson(change);
    }
```

### [C#](#tab/csharp)

```csharp
using MongoDB.Bson;
using MongoDB.Driver;

namespace ChangeStreamsDriver;

public static class DataConsumerApp
{
    public static async Task RunAsync(CancellationToken cancellationToken = default)
    {
        var context = MongoUtils.GetContext();
        await MongoUtils.PingServerAsync(context.Client, cancellationToken);

        var resumeToken = await MongoUtils.LoadResumeTokenAsync();

        var options = new ChangeStreamOptions
        {
            FullDocument = ChangeStreamFullDocumentOption.UpdateLookup,
            MaxAwaitTime = TimeSpan.FromSeconds(1)
        };

        if (resumeToken is not null)
        {
            options.ResumeAfter = resumeToken;
            Console.WriteLine($"Resuming from last token: {Abbreviate(resumeToken.ToJson())}...");
        }
        else
        {
            Console.WriteLine("Starting fresh change stream (no prior token found).");
        }

        Console.WriteLine("Watching for changes...");

        using var cursor = await context.Collection.WatchAsync(options, cancellationToken);

        while (!cancellationToken.IsCancellationRequested && await cursor.MoveNextAsync(cancellationToken))
        {
            foreach (var change in cursor.Current)
            {
                var operationType = change.OperationType;

                var collectionName = change.CollectionNamespace?.CollectionName ?? context.Collection.CollectionNamespace.CollectionName;
                var fullDoc = change.FullDocument?.ToJson() ?? "{}";

                Console.WriteLine($"Change: {operationType.ToString().ToLowerInvariant()} on {collectionName}");
                Console.WriteLine($"  Full doc: {fullDoc}\n");

                await MongoUtils.SaveResumeTokenAsync(change.ResumeToken);
            }
        }
    }

    private static string Abbreviate(string tokenJson)
    {
        const int maxLength = 24;
        return tokenJson.Length <= maxLength ? tokenJson : tokenJson[..maxLength];
    }
}
```

### [Ruby](#tab/ruby)

```ruby

require_relative 'mongo_utils'
require 'json'

# MongoDB Data Consumer - Watches for change stream events
class DataConsumerApp
  def self.run
    config = MongoUtils.load_mongo_config
    client = MongoUtils.create_client(config[:uri])

    begin
      collection = MongoUtils.get_collection(client, config[:db], config[:collection])

      # Load resume token if available
      resume_token = MongoUtils.load_resume_token
      
      if resume_token
        puts "Resuming from last token: #{resume_token['_data'][0..20]}..."
        stream = collection.watch([], resume_after: resume_token)
      else
        puts "Starting fresh change stream (no prior token found)."
        stream = collection.watch([])
      end

      puts "Watching for changes..."

      # Watch for change events
      stream.each do |change|
        operation_type = change['operationType']
        ns = change['ns']
        collection_name = ns['coll'] if ns
        full_document = change['fullDocument']

        puts "Event: #{operation_type} on #{collection_name}"
        puts "  Document: #{full_document.inspect}"

        # Save the resume token after processing each event
        if change['_id']
          MongoUtils.save_resume_token(change['_id'])
        end
      end

      stream.close
    rescue Interrupt
      puts "\nConsumer stopped by user."
    ensure
      client.close
    end
  end
end

DataConsumerApp.run if __FILE__ == $0
```

### [Nodejs](#tab/nodejs)

```javascript
import {
  getContext,
  pingServer,
  saveResumeToken,
  loadResumeToken,
  closeConnection,
} from '../utils/MongoUtils.js';

/**
 * Consumer: Listens for change-stream events and persists the resume token
 */
async function main() {
  try {
    const { model } = await getContext();
    await pingServer();

    const resumeToken = await loadResumeToken();
    const watchOptions = resumeToken ? { resumeAfter: resumeToken } : {};

    if (resumeToken) {
      console.log(`Resuming from last token: ${abbreviate(JSON.stringify(resumeToken))}...`);
    } else {
      console.log('Starting fresh change stream (no prior token found).');
    }

    console.log('Watching for changes...');

    const changeStream = model.watch([], watchOptions);

    while (true) {
      const change = await changeStream.next();
      const operationType = change.operationType;

      if ('ns' in change && change.ns) {
        const ns = change.ns as { db: string; coll?: string };
        const collectionName = ns.coll ?? 'unknown';
        const fullDocument = 'fullDocument' in change ? change.fullDocument ?? {} : {};

        console.log(`Change: ${operationType} on ${collectionName}`);
        console.log(`  Full doc: ${JSON.stringify(fullDocument)}\n`);
      }

      if (change._id) {
        await saveResumeToken(change._id as Record<string, unknown>);
      }
    }
  } catch (error) {
    console.log('Azure DocumentDB connection or driver error while watching for changes.');
    console.log(error instanceof Error ? error.message : String(error));
    process.exit(1);
  } finally {
    await closeConnection();
  }
}

/**
 * Abbreviate token JSON for display
 */
function abbreviate(tokenJson: string): string {
  const maxLength = 24;
  if (tokenJson.length <= maxLength) {
    return tokenJson;
  }
  return tokenJson.substring(0, maxLength);
}

main().catch(console.error);
```

---

## Monitoring database changes with change streams

Let's understand the change stream output through the example.

### [Insert](#tab/insert)

In this change stream event, we see that a new record was `inserted` into the `exampleCollection` collection within the `cs` database, and the event details include the full content of the newly added document.

```json
{
  "_id": { "_data": "AeARBpQ/AAAA" }, // "resume_token"
  "operationType": "insert",
  "fullDocument": {
    "_id": { "$oid": "66e6f63e6f49ecaabf794958" },
    "employee_id": "17986",
    "name": "John",
    "position": "Software Engineer",
    "department": "IT",
    "rating": 4
  },
  "ns": { "db": "cs", "coll": "exampleCollection" },
  "documentKey": { "_id": { "$oid": "66e6f63e6f49ecaabf794958" } }
}
```

### [Update](#tab/update)

In this update event, the `position` & `rating` for John are modified. The change stream reflects an `update` in the `exampleCollection` collection with post update state of the document.

```json
{
  "_id": { "_data": "AWACAKM/AAAA" },
  "operationType": "update",
  "fullDocument": {
    "_id": { "$oid": "66e6f63e6f49ecaabf794958" },
    "employee_id": "17986",
    "name": "John",
    "position": "SSE",
    "department": "IT",
    "rating": 5
  },
  "ns": {
    "db": "cs", "coll": "exampleCollection"
  },
  "documentKey": {
    "_id": { "$oid": "66e6f63e6f49ecaabf794958" }
  }
}
```

- With `updateDescription` option

```json
{
  "operationType": "update",
  "updateDescription": {
    "updatedFields": {
      "position": "SSE",
      "rating": 5
    },
    "removedFields": [],
    "truncatedArrays": []
  },
  "fullDocument": { ... }
}
```

### [Delete](#tab/delete)

The change stream event indicates that a document was `deleted` from the `exampleCollection` collection in the `cs` database. The event captures the unique identifier of the removed document.

```json
{
  "_id": { "_data": "ASgBAJs/AAAA" },
  "operationType": "delete",
  "ns": { "db": "cs", "coll": "exampleCollection" },
  "documentKey": { "_id": { "$oid": "66e6f63e6f49ecaabf794958" } }
}
```

---

## Resume a change stream

Change streams are resumable by specifying a "resume token" to `resumeAfter` when opening the cursor. The `_id` field of each change event serves as the resume token. Store it durably and pass it back when reopening the stream.

### Resuming from Active Change Logs

```javascript
// Resume from the last stored\processed token
const stream = db.exampleCollection.watch([], {resumeAfter: savedResumeToken});

while (stream.hasNext()) {
  const event = stream.next();
  savedResumeToken = event._id;
  processEvent(event);
}
```

### Recovering Streams from Historical checkpoints

Standard change streams are limited to events retained within the active 400-MB change log. Once rotated, older entries are archived and become inaccessible. Historical change streams remove this limitation by transparently retrieving archived logs, enabling stream replay or resumption from past timestamps.

```javascript
// Replay all events since 2026-05-01 00:00:00 UTC, at current timestamp 2026-06-01 00:00:00 UTC
const startTime = new Timestamp(Math.floor(new Date("2026-05-01").getTime() / 1000), 1);

const stream = db.exampleCollection.watch([], {
  startAtOperationTime: startTime
});

while (stream.hasNext()) {
  printjson(stream.next());
}
```

> [!IMPORTANT]
> Change streams support resumability through the `startAt`, `resumeAfter`, and `startAtOperationTime` parameters. With point-in-time restore (PITR) log integration, stream recovery can extend up to 35 days or the cluster initialization point, whichever is earlier.
>
> Processing historical checkpoints is recommended during low-traffic periods to minimize the effect of extra resource consumption. Keep more storage (for clusters with 128 GB or lower storage) depending on the volume of data to be processed from historical point.

## Pre-images in change streams (preview)

By default, a change event shows you the document after a change. Enabling pre-images tells the system to also record the complete document before the change, exposed as `fullDocumentBeforeChange` in each event.

| Option | Behavior | When to use |
| --- | --- | --- |
| `"off"` (default) | Pre-image never included | Standard streams |
| `"whenAvailable"` | Include if available; silently omit otherwise | Audit / change data capture (CDC) scenarios with graceful degradation |
| `"required"` | Include; throw an error if not available | Use when missing pre-images must be surfaced explicitly rather than skipped |

> [!IMPORTANT]
> Pre-images are only available for `update`, `delete`, and `replace` events.
>
> Enabling pre-image increases storage and processing overhead, as the previous version of modified documents must be captured and retained for each eligible operation, which results in higher write amplification, extra I/O consumption, and increased change stream processing latency. To minimize long-term resource effects, continuous enablement in production environments is discouraged unless explicitly required by the workload.
>

### Enable pre-images on the collection

```javascript
db.runCommand({
  collMod: "exampleCollection",
  changeStreamPreAndPostImages: { enabled: true }
});
```

Sample event with pre-image enabled

```json
{
  "_id": { "_data": "AWACAKM/AAAA" },
  "operationType": "update",
  "fullDocumentBeforeChange": {
    "_id": { "$oid": "66e6f63e6f49ecaabf794958" },
    "name": "John",
    "position": "Software Engineer",
    "rating": 4
  },
  "fullDocument": {
    "_id": { "$oid": "66e6f63e6f49ecaabf794958" },
    "name": "John",
    "position": "SSE",
    "rating": 5
  },
  "ns": { "db": "cs", "coll": "exampleCollection" },
  "documentKey": { "_id": { "$oid": "66e6f63e6f49ecaabf794958" } }
}
```

## Personalize data in change streams

Customize your change stream output by specifying an array of one or more pipeline stages during configuration. Supported operators include the following.

- `$addFields`
- `$match`
- `$project`
- `$set`
- `$unset`

Example allows filtering `insert` operations only related to "IT" department.

```javascript
const pipeline = [
  { $match: { operationType: "insert", "fullDocument.department": "IT" } },
  { $project: { "fullDocument.name": 1, "fullDocument.position": 1 } }
];

const stream = db.exampleCollection.watch(pipeline);
```

## Limitations

- Cluster topology transition from single-shard to multi-shard breaks change stream resumability.
- Multi-shard deployments don't preserve global event ordering.
- Change stream cursors must be reinitialized after failover events, while resume capability remains preserved.
- UpdateDescription isn't supported for `update` events within aggregation pipelines. However, update operators are supported.
- `$changestream` as a nested pipeline of another stage isn't supported.
- Pre-image isn't supported for historical processing of logs, since the additional pre-image information wasn't being tracked prior.
- Change streams don't support `showExpandedEvents`.

## Related content

- Integrate [change streams with Kafka (using Debezium connector)](https://github.com/Azure-Samples/Delivery-tracking-vCore-debezium).
- [Sample application demonstrating change stream usage](https://github.com/AzureCosmosDB/changestream-driver-compatibility).
- [Autoscale](autoscale.md)
