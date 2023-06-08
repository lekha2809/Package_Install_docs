PUT _ilm/policy/my_policy
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_age": "7d",
            "max_size": "50gb"
          }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "forcemerge": {
            "max_num_segments": 1
          },
          "shrink": {
            "number_of_shards": 1
          }
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
          "allocate": {
            "require": {
              "data": "cold_node"
            }
          }
        }
      },
      "delete": {
        "min_age": "90d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}

Let's break down the policy:

The hot phase has a rollover action defined, which will trigger a rollover when the index reaches either a maximum age of 7 days or a maximum size of 50GB. This allows the index to be split into smaller, more manageable indices.

The warm phase is triggered after 7 days (min_age: 7d). It performs a forcemerge action to optimize the search performance by reducing the number of segments in the index. Additionally, a shrink action is performed to reduce the number of shards to 1, optimizing storage.

The cold phase is triggered after 30 days (min_age: 30d). It allocates the index to nodes with the "cold_node" attribute, which could be a specific set of nodes designated for long-term storage or colder data.

The delete phase is triggered after 90 days (min_age: 90d). It simply deletes the index, assuming it has reached the end of its lifecycle.

You can adjust the phase durations and actions based on your specific requirements and environment. After creating the policy, you can associate it with indices using the ilm API or by specifying it when creating the index template.
