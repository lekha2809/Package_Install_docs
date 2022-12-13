Link tham khảo: https://www.elastic.co/blog/managing-time-based-indices-efficiently

# Rollover Pattern:
```
- There is one alias which is used for indexing and which points to the active index.
- Another alias points to active and inactive indices, and is used for searching.
- The active index can have as many shards as you have hot nodes, to take advantage of the indexing resources of all your expensive hardware.
- When the active index is too full or too old, it is rolled over: a new index is created and the indexing alias switches atomically from the old index to the new.
- The old index is moved to a cold node and is shrunk down to one shard, which can also be force-merged and compressed.
```

Let’s assume we have a cluster with 10 hot nodes and a pool of cold nodes. Ideally, our active index (the one receiving all the writes) should have one shard on each hot node in order to split the indexing load over as many machines as possible.

We want to have one replica of each primary shard to ensure that we can tolerate the loss of a node without losing any data. This means that our active index should have 5 primary shards, giving us a total of 10 shards (or one per hot node). We could also use 10 primary shards (a total of 20 including replicas) and have two shards on each node.

First, we’ll create an index template for our active index:

```bash
PUT _template/active-logs
{
  "template": "active-logs-*",
  "settings": {
    "number_of_shards":   5,
    "number_of_replicas": 1,
    "routing.allocation.include.box_type": "hot",
    "routing.allocation.total_shards_per_node": 2
  },
  "aliases": {
    "search-logs": {}
  }
}
```

Indices created from this template will be allocated to nodes tagged with box_type: hot, and the total_shards_per_node setting will help to ensure that the shards are spread across as many hot nodes as possible. I’ve set it to 2 instead of 1, so that we can still allocate shards if a node fails.

We will use the active-logs alias to index into the current active index, and the search-logs alias to search across all log indices.

Here is the template which will be used by our inactive indices:

```bash
PUT _template/inactive-logs
{
  "template": "inactive-logs-*",
  "settings": {
    "number_of_shards":   1,
    "number_of_replicas": 0,
    "routing.allocation.include.box_type": "cold",
    "codec": "best_compression"
  }
}
```

Archived indices should be allocated to cold nodes and should use deflate compression to save space. I’ll explain why I’ve set replicas to 0 later.

Now we can create our first active index:

```bash
PUT active-logs-1
PUT active-logs-1/_alias/active-logs   
```

The -1 pattern in the name is recognised as a counter by the rollover API. We will use the active-logs alias to index into the current active index, and the search-logs alias to search across all log indices.

# Indexing events

When we created the active-logs-1 index, we also created the active-logs alias. From this point on, we should index using only the alias, and our documents will be sent to the current active index:

```bash
POST active-logs/log/_bulk
{ "create": {}}
{ "text": "Some log message", "@timestamp": "2016-07-01T01:00:00Z" }
{ "create": {}}
{ "text": "Some log message", "@timestamp": "2016-07-02T01:00:00Z" }
{ "create": {}}
{ "text": "Some log message", "@timestamp": "2016-07-03T01:00:00Z" }
{ "create": {}}
{ "text": "Some log message", "@timestamp": "2016-07-04T01:00:00Z" }
{ "create": {}}
{ "text": "Some log message", "@timestamp": "2016-07-05T01:00:00Z" }
```
 
# Rolling over the index

At some stage, the active index is going to become too big or too old and you will want to replace it with a new empty index. The rollover API allows you to specify just how big or how old an index is allowed to be before it is rolled over.

How big is too big? As always, it depends. It depends on the hardware you have, the types of searches you perform, the performance you expect to see, how long you are willing to wait for a shard to recover, etc etc. In practice, you can try out different shard sizes and see what works for you. To start, choose some arbitrary number like 100 million or 1 billion. You can adjust this number up or down depending on search performance, data retention periods, and available space.

There is a hard limit on the number of documents that a single shard can contain: 2,147,483,519. If you plan to shrink your active index down to a single shard, you must have fewer than 2.1 billion documents in your active index. If you have more documents than can fit into a single shard, you can shrink your index to more than one shard, as long as the target number of shards is a factor of the original, e.g. 6 → 3, or 6 → 2.

Rolling an index over by age may be convenient as it allows you to parcel up your logs by hour, day, week, etc, but it is usually more efficient to base the rollover decision on the number of documents in the index. One of the benefits of size-based rollover is that all of your shards are of approximately the same weight, which makes them easier to balance.

The rollover API would be called regularly by a cron job to check whether the max_docs or max_age constraints have been breached. As soon as at least one constraint has been breached, the index is rolled over. Since we’ve only indexed 5 documents in the example above, we’ll specify a max_docs value of 5, and (for completeness), a max_age of one week:

```bash
POST active-logs/_rollover
{
  "conditions": {
    "max_age":   "7d",
    "max_docs":  5
  }
}
```

This request tells Elasticsearch to rollover the index pointed to by the active-logs alias if that index either was created at least seven days ago, or contains at least 5 documents. The response looks like this:

```bash
{
  "old_index": "active-logs-1",
  "new_index": "active-logs-2",
  "rolled_over": true,
  "dry_run": false,
  "conditions": {
    "[max_docs: 5]": true,
    "[max_age: 7d]": false
  }
}
```

The active-logs-1 index has been rolled over to the active-logs-2 index because the max_docs: 5 constraint was met. This means that a new index called active-logs-2 has been created, based on the active-logs template, and the active-logs alias has been switched from active-logs-1 to active-logs-2.

By the way, if you want to override any values from the index template such as settings or mappings, you can just pass them in to the _rollover request body like you would with the create index API.

## Why don't we support a max_size constraint?

Given that the intention is to produce evenly sized shards, why don't we support a max_size constraint in addition to max_docs?  The answer is that shard size is a less reliable measure because ongoing merges can produce significant temporary growth in shard size which disappears as soon as a merge is completed.  Five primary shards, each in the process of merging to one 5GB shard, would temporarily increase the index size by 25GB!  The doc count, on the other hand grows predictably.

# Shrinking the index

Now that active-logs-1 is no longer being used for writes, we can move it off to the cold nodes and shrink it down to a single shard in a new index called `inactive-logs-1. Before shrinking, we have to:

- Make the index read-only.
- Move one copy of all shards to a single node. You can choose whichever node you like, probably whichever cold node has the most available space.

This can be achieved with the following request:

```bash
PUT active-logs-1/_settings
{
  "index.blocks.write": true,
  "index.routing.allocation.require._name": "some_node_name"
}
```

The allocation setting ensures that at least one copy of each shard will be moved to the node with name some_node_name. It won’t move ALL the shards — a replica shard can’t be allocated to the same node as its primary — but it will ensure that at least one primary or replica of each shard will move.

Once the index has finished relocating (use the cluster health API to check), issue the following request to shrink the index:

`POST active-logs-1/_shrink/inactive-logs-1`
        
This will kick off the shrink process. As long as your filesystem supports hard links, the shrink will be almost instantaneous. If your filesystem doesn’t support hard links, well, you’ll have to wait while all the segment files are copied from one index to another…

You can monitor the shrink process with the cat recovery API or with the cluster health API:

`GET _cluster/health/inactive-logs-1?wait_for_status=yellow`
        
As soon as it is done, you can remove the search-logs alias from the old index and add it to the new:

```bash
POST _aliases
{
  "actions": [
    {
      "remove": {
        "index": "active-logs-1",
        "alias": "search-logs"
      }
    },
    {
      "add": {
        "index": "inactive-logs-1",
        "alias": "search-logs"
      }
    }
  ]
}
```

# Saving space

Our index has been reduced to a single shard, but it still contains the same number of segment files as before, and the best_compression setting hasn’t kicked in yet because we haven’t made any writes. We can improve the situation by force-merging our single-shard index down to a single segment, as follows:

`POST inactive-logs-1/_forcemerge?max_num_segments=1`
        
This request will create a single new segment to replace the multiple segments that existed before. Also, because Elasticsearch has to write a new segment, the `best_compression` setting will kick in and the segment will be written with deflate compression.

There is no point in running a force-merge on both the primary and replica shard, which is why our template for inactive log indices set the number_of_replicas to 0. Now that force-merge has finished, we can increase the number of replicas to gain redundancy:

```bash
PUT inactive-logs-1/_settings
{ "number_of_replicas": 1 }
```
        
Once the replica has been allocated — use the cluster health API with ?wait_for_status=green to check — we can be sure that we have a redundant copy of our data and we can safely delete the active-logs-1 index:

`DELETE active-logs-1`

# Deleting old indices
With the old index-per-day pattern, it was easy to decide which old indices could be dropped. With the rollover pattern, it isn’t quite as obvious which index contains data from which time period.

Fortunately, the field stats API makes this easy to determine. We just need to look for all indices where the highest value in the @timestamp field is older than our cutoff:

```bash
GET search-logs/_field_stats?level=indices
{
  "fields": ["@timestamp"],
  "index_constraints": {
    "@timestamp": {
      "max_value": {
        "lt": "2016/07/03",
        "format": "yyyy/MM/dd"
      }
    }
  }
}
```