# Investigation Notes — Wazuh Cluster Health and Node Consistency

## 1. Investigation Summary

This investigation evaluated the operational state of a Wazuh Manager deployment and examined whether its cluster configuration could be used to demonstrate cluster health and node consistency.

The environment consisted of a single Rocky Linux 9.8 virtual machine running Wazuh Manager, Indexer, and Dashboard version 4.13.1. The original Manager configuration had clustering disabled.

The investigation included cluster configuration inspection, cluster-control commands, Manager service checks, network-listener inspection, and log analysis. A single-node master configuration was attempted, but the Manager failed to restart after the change. The original configuration was restored successfully.

The final confirmed state after recovery was a running standalone Manager.

## 2. Environment Baseline

### Operating System and Network

| Attribute | Observed Value |
|---|---|
| Hostname | localhost |
| Operating System | Rocky Linux 9.8 (Blue Onyx) |
| Kernel | 5.14.0-687.31.1.el9_8.x86_64 |
| Virtualization | VMware |
| Architecture | x86-64 |
| Network Interface | ens160 |
| IPv4 Address | 192.168.203.129/24 |

### Wazuh Components

The RPM package query returned:

| Component | Installed Version |
|---|---|
| Wazuh Manager | 4.13.1-1 |
| Wazuh Indexer | 4.13.1-1 |
| Wazuh Dashboard | 4.13.1-1 |

The Wazuh control utility reported:

- `WAZUH_VERSION="v4.13.1"`
- `WAZUH_REVISION="rc1"`
- `WAZUH_TYPE="server"`

The three central Wazuh components were aligned at package version 4.13.1.

## 3. Original Cluster Configuration

The cluster section in `/var/ossec/etc/ossec.conf` was inspected using Python XML parsing to avoid exposing the shared key.

The captured configuration contained:

| Setting | Observed Value |
|---|---|
| Cluster name | `wazuh` |
| Node name | `node01` |
| Node type | `master` |
| Cluster port | `1516` |
| Bind address | `0.0.0.0` |
| Hidden | `no` |
| Disabled | `yes` |
| Configured node entry | `NODE_IP` |

The shared key was deliberately redacted.

The important finding was that the cluster section existed and identified the node as a master, but clustering was disabled.

The configured node entry appeared as `NODE_IP` in the captured output. Its intended address was not established from that value alone.

## 4. Initial Cluster-Control Result

The initial command:

```bash
sudo /var/ossec/bin/cluster_control -l
```

returned:

```text
ERROR: Cluster is not running.
```

This was consistent with the original `disabled: yes` setting.

The result indicated that cluster-control could not query a running cluster. It did not establish that the Wazuh Manager service itself was stopped or unhealthy.

A separate captured cluster-control result during the exercise showed:

```text
NAME    TYPE    VERSION ADDRESS
node01  master  4.13.1  192.168.203.129
```

The corresponding `cluster_control -i` output showed:

```text
Cluster name: wazuh

Last completed synchronization for connected nodes (0):
```

This demonstrated that a single master entry was visible at that captured point. It did not demonstrate worker connectivity or master-worker synchronization.

The cluster-control captures and the later post-rollback state represent different points in the exercise. The final state after the rollback was confirmed separately.

## 5. Network Listener Review

The `ss -lntp` output contained the following relevant listeners:

| Port | Observed Process | Purpose |
|---|---|---|
| 443 | node | Wazuh Dashboard HTTPS |
| 55000 | python3 | Wazuh API |
| 1514 | wazuh-remoted | Agent communication |
| 1515 | wazuh-authd | Agent enrollment |
| 1516 | python3 | Cluster communication listener observed in the captured state |
| 9200 | java | Indexer API, loopback-bound |
| 9300 | java | Indexer transport, loopback-bound |

The cluster port was also specifically queried:

```bash
sudo ss -lntp | grep ':1516'
```

The captured output showed a listener on `0.0.0.0:1516`.

This is evidence that the port was listening at the time of that inspection. It is not proof that a worker was connected or that synchronization was taking place.

The Indexer listeners on ports 9200 and 9300 were bound to loopback addresses in the captured output.

## 6. Manager Service Health

The Manager service was inspected using:

```bash
sudo systemctl status wazuh-manager --no-pager
sudo systemctl is-active wazuh-manager
sudo systemctl is-enabled wazuh-manager
```

The supplied systemd capture showed:

- Service state: `active (running)`
- Service enabled: `enabled`
- Startup completed successfully.
- The captured service start time was `2026-10-31 07:01:47 IST`.

The Wazuh control utility was also used to inspect individual daemon states.

In the final post-rollback check, the output included:

**Running:**
- `wazuh-modulesd`
- `wazuh-monitord`
- `wazuh-logcollector`
- `wazuh-remoted`
- `wazuh-syscheckd`
- `wazuh-analysisd`
- `wazuh-execd`
- `wazuh-db`
- `wazuh-authd`
- `wazuh-apid`

**Not running:**
- `wazuh-clusterd`
- `wazuh-maild`
- `wazuh-agentlessd`
- `wazuh-integratord`
- `wazuh-dbd`
- `wazuh-csyslogd`

The stopped optional or feature-dependent daemons should not automatically be treated as a Manager failure. Their relevance depends on the features configured for the deployment.

The stopped `wazuh-clusterd` process was expected after restoring the standalone configuration.

## 7. IndexerConnector Log Findings

Manager logs contained repeated IndexerConnector initialization warnings.

Examples included failures to initialize:

- `wazuh-states-inventory-system-localhost.localdomain`
- `wazuh-states-inventory-processes-localhost.localdomain`
- `wazuh-states-inventory-ports-localhost.localdomain`
- `wazuh-states-inventory-hotfixes-localhost.localdomain`
- `wazuh-states-inventory-hardware-localhost.localdomain`
- `wazuh-states-inventory-protocols-localhost.localdomain`
- `wazuh-states-inventory-interfaces-localhost.localdomain`
- `wazuh-states-inventory-networks-localhost.localdomain`

The log also contained:

```text
Failed to sync agent '001' with the indexer.
```

Additional errors included:

```text
Server returned nothing (no headers, no data), status code: -1.
```

and:

```text
Could not connect to server, status code: -1.
```

These messages indicate IndexerConnector initialization and communication problems at the recorded times.

They are separate from the cluster-control result. The available evidence does not establish that the cluster configuration attempt caused the IndexerConnector failures.

## 8. Cluster-Enablement Attempt

Because the deployment contained only one Wazuh VM, a single-node master configuration was attempted for basic cluster-mode exploration.

The cluster-enabled setting was changed from:

```xml
<disabled>yes</disabled>
```

to:

```xml
<disabled>no</disabled>
```

The modified configuration was checked using:

```bash
sudo xmllint --noout /var/ossec/etc/ossec.conf
```

The command returned no output, indicating that the XML was syntactically valid.

However, the subsequent Manager restart failed:

```text
Job for wazuh-manager.service failed because the control process exited with error code.
```

The exact startup cause was not established before the rollback. Therefore, the failure should not be attributed to a specific cluster parameter without further evidence.

## 9. Configuration Recovery

The modified configuration was preserved as:

```text
/var/ossec/etc/ossec.conf.cluster-failed
```

The original configuration was restored from:

```text
/var/ossec/etc/ossec.conf.pre-cluster
```

The Manager service was restarted after restoration.

The subsequent checks returned:

```text
active
```

and:

```text
wazuh-clusterd not running...
```

The Manager's core processes were running in the final post-rollback status output.

This confirmed recovery to the previous standalone operating state.

## 10. Findings and Assessment

| Area | Finding | Assessment |
|---|---|---|
| Wazuh component versions | Central components reported 4.13.1 | Verified |
| Operating system | Rocky Linux 9.8 | Verified |
| Original cluster setting | Disabled | Verified |
| Initial cluster-control query | Cluster not running | Verified |
| Single-node master listing | One master entry captured | Observed |
| Completed worker synchronization | Zero connected-node synchronizations reported | Verified for captured output |
| Cluster port | Port 1516 listener captured | Observed |
| Manager service after rollback | Active | Verified |
| Cluster daemon after rollback | Not running | Verified |
| IndexerConnector | Initialization and connection errors recorded | Verified |
| Worker connectivity | No second Manager available | Not testable |
| Cross-node consistency | No worker node available | Not testable |
| Cluster failover | Not performed | Not tested |

## 11. Conclusion

The investigation established the baseline configuration and operational state of the single-VM Wazuh deployment. It also demonstrated that XML syntax validation alone is insufficient to guarantee that a Wazuh Manager will start successfully after a configuration change.

The cluster-enablement attempt resulted in a Manager restart failure. Restoring the pre-change configuration returned the Manager service to an active state.

The final confirmed deployment state was standalone, with the cluster daemon not running.

The captured single-node master listing provided limited visibility into cluster-mode reporting, but no worker synchronization was demonstrated. A second Wazuh Manager is required to assess genuine master-worker communication and node consistency.

IndexerConnector initialization and connection errors remain separate findings that require further investigation.
