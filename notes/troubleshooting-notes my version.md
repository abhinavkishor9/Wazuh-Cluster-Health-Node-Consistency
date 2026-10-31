# Troubleshooting Notes 
## 1. Cluster Control Reports "Cluster is not running"

### Symptom

Running:

```bash
sudo /var/ossec/bin/cluster_control -l
```

returned:

```text
ERROR: Cluster is not running.
```

### Investigation

The cluster section in `/var/ossec/etc/ossec.conf` was inspected. The relevant setting was:

```xml
<disabled>yes</disabled>
```

The original configuration also identified the node type as `master`, but clustering was disabled.

### Explanation

A configured cluster section does not mean that clustering is active. With clustering disabled, the cluster daemon is not expected to run and cluster-control cannot retrieve an active cluster listing.

### Resolution

No correction was required for the original standalone deployment. The disabled state was documented as the existing configuration.

---

## 2. XML Validation Passed but Manager Restart Failed

### Symptom

After changing the cluster-enabled setting, the configuration was validated:

```bash
sudo xmllint --noout /var/ossec/etc/ossec.conf
```

The command returned no output.

However, restarting the Manager failed:

```text
Job for wazuh-manager.service failed because the control process exited with error code.
```

### Investigation

The XML file was syntactically valid, but the Manager did not start successfully with the modified configuration.

The exact startup cause was not confirmed before recovery.

### Explanation

XML validation checks document syntax. It does not validate every Wazuh-specific setting, the operational suitability of the cluster configuration, or whether the Manager can successfully initialize all required components.

### Resolution

The previous configuration was restored.

The modified configuration was first preserved:

```bash
sudo cp -a /var/ossec/etc/ossec.conf \
  /var/ossec/etc/ossec.conf.cluster-failed
```

The previous configuration was then restored:

```bash
sudo cp -a /var/ossec/etc/ossec.conf.pre-cluster \
  /var/ossec/etc/ossec.conf
```

The Manager was restarted:

```bash
sudo systemctl restart wazuh-manager
```

The service-state check returned:

```text
active
```

The rollback successfully recovered the Manager service.

---

## 3. Cluster Daemon Not Running After Recovery

### Symptom

After restoring the previous configuration:

```bash
sudo /var/ossec/bin/wazuh-control status
```

reported:

```text
wazuh-clusterd not running...
```

### Explanation

The restored configuration was the original standalone configuration, in which clustering was disabled.

Therefore, the cluster daemon not running was consistent with the restored deployment state. It did not indicate that the Manager recovery had failed.

### Resolution

No further change was made. The standalone configuration was retained.

---

## 4. Single Master Node Does Not Demonstrate Synchronization

### Observation

A captured cluster-control listing showed:

```text
NAME    TYPE    VERSION ADDRESS
node01  master  4.13.1  192.168.203.129
```

The cluster information output reported:

```text
Last completed synchronization for connected nodes (0):
```

### Explanation

A single master entry does not establish that master-worker synchronization has been tested.

The environment contained only one Wazuh VM. No second Manager was available to act as a worker.

### Resolution

Record the single-node master output as limited evidence only. Mark worker connectivity and cross-node consistency as not testable.

---

## 5. IndexerConnector Initialization and Connection Errors

### Symptoms

Manager logs contained repeated messages such as:

```text
IndexerConnector initialization failed
```

and:

```text
Failed to sync agent '001' with the indexer.
```

Additional messages included:

```text
Server returned nothing (no headers, no data), status code: -1.
```

and:

```text
Could not connect to server, status code: -1.
```

### Investigation

The captured network-listener output showed the Indexer listening on loopback addresses for ports 9200 and 9300.

The Manager API was also listening on port 55000.

### Explanation

These messages indicate IndexerConnector initialization or communication problems. They should not be automatically attributed to the cluster configuration attempt.

Cluster communication and Manager-to-Indexer communication are separate functions.

### Resolution Status

**Unresolved in this lab.**

No Indexer configuration change, credential reset, or connectivity repair was performed as part of the cluster assessment.

This issue should be investigated separately before relying on complete Indexer-backed state synchronization.

---

## 6. Interpreting Ports and Listeners

The captured listener inspection included:

| Port | Component | Interpretation |
|---|---|---|
| 1514 | Wazuh Remote | Agent communication |
| 1515 | Wazuh Auth | Agent enrollment |
| 1516 | Cluster service | Cluster communication listener observed in one captured state |
| 55000 | Wazuh API | API access |
| 443 | Dashboard | HTTPS interface |
| 9200 | Wazuh Indexer | Indexer API, loopback-bound |
| 9300 | Wazuh Indexer | Indexer transport, loopback-bound |

A listening port only confirms that a process was listening at the time of inspection. It does not prove successful application-level communication or synchronization.

After the standalone configuration was restored, the cluster daemon was reported as not running. Therefore, the earlier port 1516 observation should not be treated as the final deployment state.

---

## 7. Interpreting Non-Running Manager Daemons

The final Wazuh control output showed several processes running and others not running.

The following were reported as not running:

- `wazuh-clusterd`
- `wazuh-maild`
- `wazuh-agentlessd`
- `wazuh-integratord`
- `wazuh-dbd`
- `wazuh-csyslogd`

These states should be interpreted in the context of the deployment's enabled features.

For example, `wazuh-clusterd` was expected to be stopped after restoring the standalone configuration. The other stopped daemons should not automatically be treated as faults without confirming that their associated features are required.

---

## 8. Recovery Precautions

The following files were created during recovery:

| File | Purpose |
|---|---|
| `/var/ossec/etc/ossec.conf.pre-cluster` | Backup of the original configuration |
| `/var/ossec/etc/ossec.conf.cluster-failed` | Preserved copy of the modified configuration |
| `/var/ossec/etc/ossec.conf` | Restored active configuration |

The configuration files may contain sensitive settings, including cluster keys. They must not be committed to GitHub or shared without appropriate redaction.

The successful rollback demonstrated the value of preserving a configuration backup before attempting changes.

