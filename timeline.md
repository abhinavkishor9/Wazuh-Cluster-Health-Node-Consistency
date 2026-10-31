
# Lab Timeline — Wazuh Cluster Health and Node Consistency

## 2026-10-30

| Time (IST) | Activity / Observation | Result |
|---|---|---|
| 06:07:46 | Manager logs reported IndexerConnector initialization failures for multiple state inventory indices. | Warning recorded |
| 06:10:05 | Manager logs reported failure to synchronize agent `001` with the Indexer. | Warning recorded |
| 06:31:08 | Wazuh authentication service reported that it was accepting connections on port 1515. | Startup activity recorded |

## 2026-10-31

| Time (IST) | Activity / Observation | Result |
|---|---|---|
| 07:01:38 | Wazuh authentication service reported that it was accepting connections on port 1515. | Service activity recorded |
| 07:01:46 | Manager logs reported further IndexerConnector initialization failures and agent synchronization warnings. | Warnings recorded |
| 07:01:47 | Captured systemd status showed `wazuh-manager` active and running. | Manager active |
| 07:09:10 | Manager logs reported an IndexerConnector response error: no headers or data returned. | Error recorded |
| 07:09:13 | Manager logs reported that the Indexer server could not be reached. | Error recorded |

## Cluster Assessment Activities

The following activities were performed during the lab. Exact command execution times were not captured for each action.

| Activity | Observation / Result |
|---|---|
| Inspected the cluster configuration | Original configuration showed `disabled: yes` |
| Executed `cluster_control -l` | Initially returned `ERROR: Cluster is not running.` |
| Inspected cluster membership in a captured state | One master entry, `node01`, was listed |
| Inspected cluster synchronization information | Zero completed synchronizations for connected nodes were reported |
| Inspected network listeners | Port 1516 was observed listening in one captured state |
| Attempted single-node master configuration | XML syntax validation passed |
| Restarted Manager after the configuration change | Restart failed |
| Preserved the modified configuration | Saved as `ossec.conf.cluster-failed` |
| Restored the previous configuration | Restored from `ossec.conf.pre-cluster` |
| Restarted Manager after restoration | Restart completed successfully |
| Checked the final Manager service state | `active` |
| Checked final daemon status | Core Manager processes running; `wazuh-clusterd` not running |

## Final State

The original standalone configuration was restored successfully. The Wazuh Manager service was active, while the cluster daemon remained stopped as expected for the restored configuration.

Multi-node synchronization and cross-node consistency were not validated because the environment contained only one Wazuh VM.

IndexerConnector initialization and connection errors were documented as separate unresolved findings.
