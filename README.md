# ILM Limiter

Limit disk usage of Elasticsearch indexes in ILM phases

## Background

The [Index Lifecycle Management](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-lifecycle-management.html) (ILM) of Elasticsearch is only able to manage indexes individually. Any kind of grouping is not supported, and so it is not possible to prevent a cluster from running out of disk space, e.g. when the amount of logs ingested changes unexpectedly.

The predecessor of ILM, [Curator](https://www.elastic.co/guide/en/elasticsearch/client/curator/current/index.html), can manage indexes in groups, but is unable to move indexes between ILM phases.

## Running ilm-limiter

### Compatibility

_ilm-limiter_ is compatible with Elasticsearch 8.11 and newer versions

### Privileges

_ilm-limiter_ needs to run with credentials that have the `manage_ilm` privilege on cluster level and `manage` privilege on all indexes it should manage.

### Configuration

_ilm-limiter_ is configured by creating a specific object in the [Metadata](https://www.elastic.co/guide/en/elasticsearch/reference/current/ilm-put-lifecycle.html) parameter of the lifecycle policies that should be limited. It contains, for each phase, the maximum disk size that all indexes in that phase are allowed to use:
```json
{
  "_meta": {
    "ilm-limiter": {
      "phases": {
        "hot": {
          "max_size": "1800gb"
        },
        "cold": {
          "max_size": "7200gb"
        }
      }
    }
  }
}
```

### Functionality

_ilm-limiter_ should be run as a cronjob and will perform the following steps:

- identify all ILM policies that have a valid _ilm-limiter_ configuration
- for all phases in all limited policies:
  - identify which indexes are in the policy's phase
  - identify the oldest indexes that exceed the configured disk usage limit of the phase
  - move indexes exceeding the limit to the next phase in the ILM policy

Moving of indexes is done using the [Move to lifecycle step API](https://www.elastic.co/guide/en/elasticsearch/reference/current/ilm-move-to-step.html). Only indexes that have reached the final action and step (`completed`, `completed`) of their current phase get moved, to not interfere with ILM.

## Examples

Example 1:

- Rollover indexes when their primary shard size is over 50 GB or when they are 7 days old
- Delete indexes when they are 30 days old.
- Delete the oldest indexes whenever all indexes in the hot phase exceed 1500 GB

```json
{
  "policy": {
    "_meta": {
      "ilm-limiter": {
        "phases": {
          "hot": {
            "max_size": "1500gb"
          }
        }
      }
    },
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "set_priority": {
            "priority": 30
          },
          "rollover": {
            "max_age": "7d",
            "max_primary_shard_size": "50gb"
          },
          "forcemerge": {
            "max_num_segments": 1
          }
        }
      },
      "delete": {
        "min_age": "30d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

Example 2:

- Rollover indexes when their primary shard size is over 60 GB or when they are 5 days old.
- Move indexes to the cold phase when they are 7 days old.
- Move the oldest indexes to the cold phase whenever all indexes in the hot phase exceed 2 TB.
- Delete indexes when they are 30 days old.
- Delete the oldest indexes whenever all indexes in the cold phase exceed 6 TB


```json
{
  "policy": {
    "_meta": {
      "ilm-limiter": {
        "phases": {
          "hot": {
            "max_size": "2tb"
          },
          "cold": {
            "max_size": "6tb"
          }
        }
      }
    },
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "set_priority": {
            "priority": 100
          },
          "rollover": {
            "max_age": "5d",
            "max_primary_shard_size": "60gb"
          },
          "forcemerge": {
            "max_num_segments": 1
          }
        }
      },
      "cold": {
        "min_age": "7d",
        "actions": {
          "set_priority": {
            "priority": 0
          },
          "searchable_snapshot": {
            "snapshot_repository": "found-snapshots",
            "force_merge_index" : false
          }
        }
      },
      "delete": {
        "min_age": "30d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```
